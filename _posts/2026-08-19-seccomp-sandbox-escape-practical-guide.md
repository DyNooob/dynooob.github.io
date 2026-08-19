---
layout: post
title: "Linux seccomp 安全编程与沙箱逃逸分析实战"
date: 2026-08-19 09:00:00 +0800
categories: [安全]
tags: [seccomp, sandbox, linux-security, container-security, kernel]
---

## 引言

seccomp（Secure Computing Mode）是 Linux 内核提供的最重要的安全机制之一，它允许进程限制自己能发起的系统调用，从而大幅缩小攻击面。Docker、Chromium、OpenSSH、systemd 等主流项目都依赖 seccomp 实现沙箱隔离。

但 seccomp 不是银弹——错误的配置规则、与内核其他机制的交互漏洞，仍然可能被绕过。本文从实战出发，先讲 seccomp 的正确用法，再分析几种真实的沙箱逃逸手法，最后给出可落地的规则模板。

## 一、seccomp 基础

### 1.1 两种模式

seccomp 有两种工作模式：

- **strict 模式**：只允许 read、write、exit、sigreturn 四个系统调用。一旦调用其他 syscall，进程立即被 SIGKILL 杀死。过于严格，实际生产环境几乎不用。
- **filter 模式**：通过 BPF 规则自定义允许/禁止哪些系统调用，精细控制沙箱边界。

### 1.2 BPF 规则结构

seccomp 使用 BPF（Berkeley Packet Filter）语法定义规则，每条指令是一个 8 字节的 `sock_filter` 结构：

```c
struct sock_filter {
    __u16 code;    // 指令码
    __u8  jt;      // 条件为真时跳转偏移
    __u8  jf;      // 条件为假时跳转偏移
    __u32 k;       // 通用字段
};
```

内核通过 `prctl(PR_SET_SECCOMP, ...)` 或 `seccomp(SECCOMP_SET_MODE_FILTER, ...)` 系统调用加载规则。

### 1.3 最小化 seccomp 示例

直接写 BPF 汇编非常痛苦，推荐使用 libseccomp 库（C）或 `seccomp` 包（Go/Python/Rust）。

#### C 语言示例

```c
#include <seccomp.h>
#include <stdio.h>
#include <unistd.h>

int main() {
    scmp_filter_ctx ctx;
    ctx = seccomp_init(SCMP_ACT_LOG); // 默认记录，不阻止

    // 只允许 read, write, exit, exit_group
    seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(read), 0);
    seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(write), 0);
    seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(exit), 0);
    seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(exit_group), 0);

    // 禁止 execve——阻止执行新程序
    seccomp_rule_add(ctx, SCMP_ACT_KILL, SCMP_SYS(execve), 0);

    seccomp_load(ctx);
    seccomp_release(ctx);

    printf("seccomp 已加载，尝试执行 /bin/sh...\n");
    execl("/bin/sh", "sh", NULL);
    printf("execve 被拦截，这行不会执行到\n");
    return 0;
}
```

编译：
```bash
gcc -o seccomp_demo seccomp_demo.c -lseccomp
```

#### Python 示例

```python
import seccomp
import os

# 默认动作：记录并拒绝
filter = seccomp.SyscallFilter(seccomp.ACTION_LOG)

# 白名单允许的系统调用
filter.add_rule(seccomp.ACTION_ALLOW, "read")
filter.add_rule(seccomp.ACTION_ALLOW, "write")
filter.add_rule(seccomp.ACTION_ALLOW, "exit")
filter.add_rule(seccomp.ACTION_ALLOW, "exit_group")
filter.add_rule(seccomp.ACTION_ALLOW, "mmap")
filter.add_rule(seccomp.ACTION_ALLOW, "munmap")
filter.add_rule(seccomp.ACTION_ALLOW, "brk")

# 禁止 execve
filter.add_rule(seccomp.ACTION_KILL, "execve")

filter.load()

os.system("echo '这行不会执行到'")
```

## 二、seccomp 规则编写最佳实践

### 2.1 白名单优先，黑名单免谈

永远使用白名单模式（`SCMP_ACT_ALLOW` 逐条放行），而不是黑名单模式（`SCMP_ACT_KILL` 逐条禁止）。黑名单漏掉一个关键 syscall 就可能导致沙箱被绕过。

```python
# 正确做法：白名单
filter = seccomp.SyscallFilter(seccomp.ACTION_KILL)  # 默认拒绝所有
filter.add_rule(seccomp.ACTION_ALLOW, "read")
filter.add_rule(seccomp.ACTION_ALLOW, "write")

# 错误做法：黑名单
filter = seccomp.SyscallFilter(seccomp.ACTION_ALLOW)  # 默认允许所有
filter.add_rule(seccomp.ACTION_KILL, "execve")        # 只禁止 execve
```

### 2.2 参数过滤

seccomp 支持通过 `seccomp_rule_add_exact` 检查系统调用的参数值：

```c
// 只允许 open 只读模式
seccomp_rule_add_exact(ctx, SCMP_ACT_ALLOW, SCMP_SYS(open), 1,
                       SCMP_A1(SCMP_CMP_MASKED_EQ, O_RDONLY, O_RDONLY));
```

参数过滤能极大提升规则精度，但也要注意：并非所有 syscall 的参数结构都简单可预测。

### 2.3 兼容性处理

不同架构的 syscall 编号不同，x86_64 的 `execve` 是 59，i386 的 `execve` 是 11。如果进程可能切换到 32 位模式，必须同时处理两套编号：

```c
#define ARCH_AUDIT_ARCH_X86_64 AUDIT_ARCH_X86_64
#define ARCH_AUDIT_ARCH_I386   AUDIT_ARCH_I386

// 在 BPF 中检查架构
// 如果是 i386，使用 i386 的 syscall 编号
```

### 2.4 使用 SCMP_ACT_LOG 进行调试

在规则上线前，先使用 `SCMP_ACT_LOG` 模式运行一段时间，观察哪些 syscall 被触发：

```bash
# 内核日志中查看 seccomp 违规记录
dmesg | grep seccomp

# 或者用 auditd 捕获
ausearch -m SECCOMP --start today
```

## 三、沙箱逃逸攻击手法分析

理解攻击手法才能写出更坚固的规则。以下是几种绕过 seccomp 的经典路径。

### 3.1 通过未过滤的 syscall 提权

seccomp 只限制系统调用，不限制进程已有的能力。如果规则只禁止了 `execve` 和 `open`，但放行了 `ptrace`，攻击者可以：

```c
// 假设 seccomp 允许 ptrace
ptrace(PTRACE_TRACEME, 0, 0, 0);  // 将自己作为被调试进程
// 通过 ptrace 修改内存，执行任意代码
```

**防御方案**：白名单模式下不加入 `ptrace`、`process_vm_writev`、`process_vm_readv` 等危险 syscall。

### 3.2 seccomp 与用户命名空间的交互

在用户命名空间内，一些原本受限的 syscall 会被解禁。例如，`unshare(CLONE_NEWUSER)` 允许非特权用户创建命名空间，在某些内核版本中可以用 `bpf()` 系统调用加载新的 seccomp 规则覆盖原有规则。

```c
// 攻击思路：创建新用户命名空间
if (unshare(CLONE_NEWUSER) == 0) {
    // 在新命名空间中有更多权限
    // 可能加载新的 seccomp 规则
}
```

**防御方案**：
- 禁止 `unshare` 和 `clone` 的命名空间相关 flag
- 开启 `/proc/sys/kernel/unprivileged_userns_clone = 0`

### 3.3 利用内核漏洞绕过 seccomp

seccomp 是内核模块，但内核本身存在漏洞。最著名的案例是 CVE-2021-22555——Linux 内核 `net/can/bcm.c` 中的堆溢出漏洞，攻击者通过 CAN 总线协议触发漏洞，获取任意代码执行，然后加载新的 seccomp 规则。

```c
// 漏洞利用流程（概念性）
// 1. 触发内核漏洞获取任意读写
// 2. 修改当前 task_struct 的 seccomp 指针
// 3. 或直接修改 seccomp 过滤器链
// 4. 执行原本被禁止的 syscall
```

**防御方案**：保持内核版本更新，这是最根本的缓解措施。

### 3.4 时间侧信道绕过

在某些场景下，攻击者可以通过测量系统调用的执行时间判断 seccomp 规则是否存在，进而调整攻击策略。但严格来说这不属于"逃逸"，而是信息泄露。

## 四、Docker 的 seccomp 配置实战

Docker 默认使用一个 seccomp 配置文件，白名单放行了约 300 个系统调用，禁用了 44 个危险调用（如 `kexec_load`、`acct`、`ptrace` 等）。

### 4.1 查看默认配置

```bash
# 导出默认 seccomp 配置
docker run --rm -it --security-opt seccomp=unconfined alpine cat /etc/docker/seccomp.json
```

### 4.2 自定义 seccomp 配置

```json
{
    "defaultAction": "SCMP_ACT_ERRNO",
    "architectures": ["SCMP_ARCH_X86_64"],
    "syscalls": [
        {
            "names": ["read", "write", "exit", "exit_group", "mmap", "munmap", "brk", "openat", "fstat", "close", "nanosleep"],
            "action": "SCMP_ACT_ALLOW",
            "args": []
        }
    ]
}
```

运行容器时指定自定义配置：

```bash
docker run --rm --security-opt seccomp=custom.json alpine echo "hello"
```

### 4.3 调试容器中的 seccomp 违规

```bash
# 在宿主机上
sudo cat /sys/kernel/debug/seccomp/entries-* 2>/dev/null

# 或使用 strace 观察
docker run --rm --security-opt seccomp=profile.json alpine strace ls /tmp
```

## 五、生产环境 seccomp 规则模板

以下是一个适用于 Web 服务容器的通用 seccomp 规则模板：

```json
{
    "defaultAction": "SCMP_ACT_ERRNO",
    "architectures": ["SCMP_ARCH_X86_64"],
    "syscalls": [
        {
            "names": [
                "read", "write", "readv", "writev", "pread64", "pwrite64",
                "openat", "close", "fstat", "statx", "newfstatat",
                "mmap", "munmap", "mprotect", "brk", "madvise",
                "exit", "exit_group", "gettid", "getpid", "getppid",
                "clock_gettime", "nanosleep", "sched_yield",
                "recvfrom", "sendto", "recvmsg", "sendmsg",
                "accept", "accept4", "bind", "listen", "connect",
                "getsockname", "getpeername", "setsockopt", "getsockopt",
                "epoll_create1", "epoll_ctl", "epoll_wait",
                "futex", "clone3", "rt_sigaction", "rt_sigprocmask",
                "getrandom", "gettid", "tgkill"
            ],
            "action": "SCMP_ACT_ALLOW",
            "args": []
        }
    ]
}
```

**注意**：这是最小化模板，实际部署时需根据应用的具体 syscall 覆盖情况调整。建议先用 `SCMP_ACT_LOG` 模式收集应用在正常负载下的所有 syscall 调用，再构建白名单。

## 六、检测与监控

### 6.1 内核审计

```bash
# 启用 seccomp 审计
auditctl -a always,exit -F arch=b64 -S all -F seccomp=1

# 查看审计日志
ausearch -m SECCOMP --format text
```

### 6.2 Prometheus 指标

使用 `seccomp_notify` 机制（Linux 5.9+），用户态程序可以接收 seccomp 违规通知，将违规次数暴露为 Prometheus 指标：

```go
// 伪代码：seccomp 违规通知 --> 指标
notifyFd, _ := seccomp.NotifyInit(ctx, SCMP_ACT_NOTIFY, filter)
go func() {
    for {
        req, _ := seccomp.NotifyReceive(notifyFd)
        metricInc("seccomp_violation_total", req.Pid)
        seccomp.NotifyRespond(notifyFd, req.ID, -EPERM)
    }
}()
```

### 6.3 容器运行时验证

```bash
# 检查容器是否启用了 seccomp
docker inspect $(docker ps -q) --format '{{.HostConfig.SecurityOpt}}'

# 验证进程的 seccomp 模式
cat /proc/$(pidof nginx)/status | grep -i seccomp
```

输出值含义：
- `0`：未启用 seccomp
- `1`：strict 模式
- `2`：filter 模式

## 七、总结

seccomp 是 Linux 沙箱体系中最核心的组件之一，正确使用能有效阻止大量提权和代码执行攻击。关键要点：

1. **永远使用白名单**，默认拒绝再逐条放行
2. **参数过滤**能大幅提升精度，但需要充分测试
3. **注意跨架构兼容**，32 位 syscall 编号可能绕过 64 位规则
4. **seccomp 不是万能药**，需与 Capabilities、AppArmor/SELinux、用户命名空间控制配合使用
5. **保持内核更新**，修补已知漏洞是防止绕过的最根本手段

将 seccomp 纳入安全左移流程，在 CI/CD 中自动生成和验证 seccomp 规则，是生产环境安全架构的必修课。