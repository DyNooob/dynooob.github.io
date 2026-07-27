---
layout: post
title: "Linux Capabilities 实战：从全能 Root 到细粒度权限管理"
date: 2026-07-27 09:00:00 +0800
categories: [安全开发, 开发]
tags: [Linux, Capabilities, 权限管理, 安全, 容器, setuid, 系统管理]
---

## 为什么需要 Capabilities？

传统 Unix 权限模型只有两个层级：普通用户和 root。一个进程要么是全能的（UID 0），要么什么都干不了。这种"全有或全无"的设计带来了几个现实问题：

- 一个监听 80 端口的 Web 服务，只需要 `CAP_NET_BIND_SERVICE` 这一项能力，却必须以 root 运行——一旦被攻破，攻击者获得整个系统的控制权
- `ping` 需要打开原始套接字（`CAP_NET_RAW`），所以它被设置了 setuid root——意味着任何本地用户都能以 root 权限执行它
- 容器运行时想限制容器内的进程权限，但传统权限模型无法做到"部分放弃"

**Linux Capabilities** 就是为了解决这些问题而生的。它将 root 的超级权限拆分成 40+ 个独立的小单元，每个单元控制一个具体的操作。这样一来，你可以给一个进程"开 80 端口的能力"而不给它"修改内核的能力"。

## Capabilities 基础概念

### 什么是 Capability？

Capability 是一个二进制位，代表一项特权操作。例如：

- `CAP_NET_BIND_SERVICE` = 允许绑定小于 1024 的端口
- `CAP_SYS_ADMIN` = 允许执行系统管理操作（这个大而全的能力，被称为"新的 root"）
- `CAP_NET_RAW` = 允许使用原始套接字和 RAW 套接字
- `CAP_DAC_OVERRIDE` = 允许绕过文件权限检查

### 三套能力集

Linux 为每个线程维护了三套能力集：

```
Effective (E)    —— 当前正在生效的能力，内核据此做权限检查
Permitted (P)    —— 该线程允许拥有的能力上限（可以提升到 E 集）
Inheritable (I)  —— 执行 execve() 时传递给新程序的能力
```

此外还有两个特殊集合：

- **Bounding (B)** —— 整个系统范围内允许的最大能力范围
- **Ambient (A)** —— 非特权程序执行 execve() 后仍能保留的能力（Linux 4.3+）

理解这三者的关系很重要：一个进程要执行特权操作，对应的能力必须出现在 **Effective** 集中。而有效集可以从 Permitted 集中提升（通过 `cap_set_proc()` 或 `capng_apply()`），但不能超过 Permitted 集。

## 常用 Capabilities 分类

Linux 5.x 内核提供了约 40 种能力。我把最常用的按场景分类：

### 网络安全相关

| Capability | 作用 | 风险等级 |
|-----------|------|---------|
| `CAP_NET_BIND_SERVICE` | 绑定 <1024 端口 | 低 |
| `CAP_NET_RAW` | 使用 RAW/ PACKET 套接字 | 中 |
| `CAP_NET_ADMIN` | 路由、防火墙、接口配置 | 高 |
| `CAP_NET_BROADCAST` | 监听广播包 | 低 |

### 文件系统相关

| Capability | 作用 | 风险等级 |
|-----------|------|---------|
| `CAP_DAC_OVERRIDE` | 绕过文件读写权限检查 | 高 |
| `CAP_DAC_READ_SEARCH` | 绕过文件读/目录搜索权限 | 高 |
| `CAP_CHOWN` | 改变文件属主 | 低 |
| `CAP_FOWNER` | 绕过文件属主限制的操作 | 中 |
| `CAP_LEASE` | 建立文件租约 | 低 |

### 系统管理相关

| Capability | 作用 | 风险等级 |
|-----------|------|---------|
| `CAP_SYS_ADMIN` | 大量系统管理操作（几乎相当于 root） | 极高 |
| `CAP_SYS_PTRACE` | 跟踪任意进程 | 高 |
| `CAP_SYS_CHROOT` | 调用 chroot() | 中 |
| `CAP_SYS_BOOT` | 重启系统 | 中 |
| `CAP_KILL` | 发送信号给任意进程 | 中 |
| `CAP_SYS_NICE` | 调整进程优先级和 CPU 亲和性 | 中 |

### 进程相关

| Capability | 作用 | 风险等级 |
|-----------|------|---------|
| `CAP_SETUID` | 任意设置用户 ID | 高 |
| `CAP_SETGID` | 任意设置组 ID | 高 |
| `CAP_SETPCAP` | 转移/删除能力 | 极高 |
| `CAP_SYS_RESOURCE` | 突破资源限制 | 中 |

## 实战 1：用 Capabilities 替代 setuid

### 案例：安全的 ping

传统的 `ping` 需要 root 权限来创建原始套接字。大多数发行版用 setuid 解决：

```bash
$ ls -l /bin/ping
-rwsr-xr-x 1 root root 73K  4月 22 15:23 /bin/ping
```

这意味着任何用户运行 ping 时，都能以 root 权限执行它。如果 ping 存在漏洞，攻击者可以 escalate 到 root。

更好的做法：用 capabilities 替代 setuid：

```bash
# 去掉 setuid 位
sudo chmod u-s /bin/ping

# 设置 capabilities
sudo setcap cap_net_raw+ep /bin/ping

# 验证
$ getcap /bin/ping
/bin/ping cap_net_raw=ep
```

`cap_net_raw+ep` 表示：将 `CAP_NET_RAW` 同时设置到 Effective 和 Permitted 集。这样 ping 只需要原始套接字能力，即使被攻破，攻击者也无法获得 root 的其他权限。

### 案例：绑定低端口而不做 root

Web 开发中经常需要把服务绑定到 80/443 端口：

```bash
# 给一个二进制文件绑定低端口的能力
sudo setcap cap_net_bind_service+ep /usr/local/bin/my-server

# 然后就可以用普通用户启动
$ ./my-server  # 监听 80 端口，无需 root
```

这比 `authbind` 或 `iptables REDIRECT` 方案更直接、更安全。

### 案例：允许 tcpdump 抓包

让非 root 用户运行 tcpdump：

```bash
sudo setcap cap_net_raw,cap_net_admin+ep /usr/bin/tcpdump

# 验证
$ getcap /usr/bin/tcpdump
/usr/bin/tcpdump cap_net_admin,cap_net_raw=ep
```

注意：有些发行版（如 Ubuntu）的 AppArmor 或 SELinux 策略可能会阻止非 root 用户使用这些能力，需要同时调整 LSM 策略。

## 实战 2：运行时权限控制

### 给进程降权

启动一个服务时，先以 root 启动做必要的初始化，然后主动放弃不需要的能力：

```c
#include <sys/capability.h>
#include <stdio.h>
#include <unistd.h>

int main() {
    // 先以 root 绑定端口
    // bind(80)...

    // 定义最终需要保留的能力
    cap_t caps = cap_get_proc();
    cap_value_t cap_list[] = {CAP_NET_BIND_SERVICE};
    cap_set_flag(caps, CAP_EFFECTIVE, 1, cap_list, CAP_SET);
    cap_set_flag(caps, CAP_PERMITTED, 1, cap_list, CAP_SET);

    if (cap_set_proc(caps) == -1) {
        perror("cap_set_proc");
        return 1;
    }
    cap_free(caps);

    // 放弃所有其他权限后，降级到 nobody 用户
    if (setuid(65534) == -1) {
        perror("setuid");
        return 1;
    }

    // 现在只有 CAP_NET_BIND_SERVICE，且以 nobody 身份运行
    // 即使被攻破，攻击者也无法做危险操作

    // 进入主循环...
    return 0;
}
```

编译时需链接 `-lcap`：

```bash
gcc -o server server.c -lcap
```

### 使用 libcap-ng 简化操作

如果觉得 libcap 的 API 太啰嗦，libcap-ng 提供了更友好的封装：

```c
#include <cap-ng.h>
#include <stdio.h>
#include <unistd.h>

int main() {
    // 先初始化
    capng_clear(CAPNG_SELECT_BOTH);

    // 只保留绑定端口的能力
    capng_update(CAPNG_ADD, CAPNG_EFFECTIVE|CAPNG_PERMITTED,
                 CAP_NET_BIND_SERVICE);

    // 应用
    if (capng_apply(CAPNG_SELECT_BOTH) != 0) {
        perror("capng_apply");
        return 1;
    }

    // 降权
    setuid(65534);

    // 现在只有绑定端口的能力
    return 0;
}
```

```bash
gcc -o server server.c -lcap-ng
```

## 实战 3：容器环境中的 Capabilities

这是 Capabilities 在现实中应用最广泛的场景。Docker 和 Kubernetes 都深度依赖 Capabilities 来实现权限隔离。

### Docker 容器的默认能力

运行一个容器时，Docker 默认会授予一组能力，同时丢弃危险程度高的：

```bash
# 查看容器可用的能力
$ docker run --rm alpine cat /proc/1/status | grep Cap
CapInh:	0000000000000000
CapPrm:	00000000a80425fb
CapEff:	00000000a80425fb
CapBnd:	00000000a80425fb
CapAmb:	0000000000000000

# 用 capsh 解码
$ docker run --rm alpine capsh --decode=00000000a80425fb
0x00000000a80425fb=cap_chown,cap_dac_override,cap_fowner,cap_fsetid,
cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_net_bind_service,
cap_net_raw,cap_sys_chroot,cap_mknod,cap_setfcap
```

默认授予了 14 个能力——比普通用户多，但比 root 少得多（root 拥有全部 40+）。

### 最小权限原则：删除不需要的能力

```bash
# 运行一个不需要原始套接字的 Web 服务
docker run --rm \
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  --cap-add=NET_RAW \
  --cap-add=KILL \
  -p 80:80 \
  nginx:alpine

# 真正最小化——只保留绑定端口
docker run --rm \
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  -p 80:80 \
  nginx:alpine
```

### Kubernetes 中的 Pod Security Context

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-app
spec:
  containers:
  - name: app
    image: my-app:latest
    securityContext:
      capabilities:
        drop: ["ALL"]
        add: ["NET_BIND_SERVICE"]
      runAsNonRoot: true
      runAsUser: 1000
      allowPrivilegeEscalation: false
      seccompProfile:
        type: RuntimeDefault
```

这个配置几乎是最小权限的容器化部署模板了：放弃所有能力，只保留绑定端口的，以非 root 运行，禁止提权，启用 seccomp。

### 危险能力警示

有些能力看起来人畜无害，实际上威力巨大。以下是我认为最危险的几个：

**CAP_SYS_ADMIN** —— 这是"新 root"。挂载文件系统、访问 namespace 操作、设备控制等几十个系统调用都需要它。有人说"给了 CAP_SYS_ADMIN 就等于给了 root"，这话不夸张。

**CAP_DAC_OVERRIDE** —— 绕过所有文件权限检查。给了这个能力，进程可以读写任何文件，包括 `/etc/shadow`。

**CAP_SYS_PTRACE** —— 允许跟踪任意进程。攻击者可以用它从 sshd、sudo 等进程的内存中提取密钥。

**CAP_NET_ADMIN** —— 允许修改防火墙规则、IP 地址、路由表。给了这个能力，容器可以把自己从网络隔离中"逃"出来。

## 实战 4：调试与审计 Capabilities

### 查看文件能力

```bash
# 查看所有设置了能力的文件
$ getcap -r / 2>/dev/null
/usr/bin/tar cap_dac_read_search=ep
/usr/bin/ping cap_net_raw=ep
/usr/bin/tcpdump cap_net_admin,cap_net_raw=ep
/usr/bin/mtr cap_net_raw=ep
/usr/bin/arping cap_net_raw=ep
/usr/sbin/arping cap_net_raw=ep
```

### 查看进程能力

```bash
# 查看当前进程的能力
$ cat /proc/self/status | grep Cap
CapInh:	0000000000000000
CapPrm:	0000000000000000
CapEff:	0000000000000000
CapBnd:	000001ffffffffff
CapAmb:	0000000000000000

# 用 getpcaps 更直观
$ getpcaps $$
1: =

# 查看某个进程（如 PID 1234）
$ getpcaps 1234
1234: cap_net_admin,cap_net_raw=ep
```

### 解码能力位掩码

```bash
# 安装 libcap（已包含 capsh）
$ capsh --decode=000001ffffffffff
0x000001ffffffffff=cap_chown,cap_dac_override,cap_dac_read_search,
cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,
cap_linux_immutable,cap_net_bind_service,cap_net_broadcast,cap_net_admin,
cap_net_raw,cap_ipc_lock,cap_ipc_owner,cap_sys_module,cap_sys_rawio,
cap_sys_chroot,cap_sys_ptrace,cap_sys_pacct,cap_sys_admin,cap_sys_boot,
cap_sys_nice,cap_sys_resource,cap_sys_time,cap_sys_tty_config,
cap_mknod,cap_lease,cap_audit_write,cap_audit_control,cap_setfcap,
cap_mac_override,cap_mac_admin,cap_syslog,cap_wake_alarm,
cap_block_suspend,cap_audit_read,cap_perfmon,cap_bpf,cap_checkpoint_restore
```

### 运行时的能力审计

```bash
# 启动一个进程并跟踪它的能力使用
# 方法1：使用 strace 过滤能力相关系统调用
$ strace -e trace=capget,capset,capgetpid \
  -o /tmp/cap-trace.log \
  ./my-program

# 方法2：使用 auditd 监控能力使用
$ sudo auditctl -a always,exit -F arch=b64 \
  -S capset -S capget -k cap-monitor
$ sudo ausearch -k cap-monitor
```

## 实战 5：将 Capabilities 集成到 CI/CD

在 CI/CD 流水线中，你可以使用 Capabilities 来编写更安全的自动化测试。比如：

```yaml
# .github/workflows/secure-test.yml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: 运行需要特权能力的测试
        run: |
          # 不需要 sudo，只需要个别能力
          sudo setcap cap_net_admin+ep ./test-network
          ./test-network
          sudo setcap -r ./test-network
```

或者用 Docker 隔离测试环境：

```bash
# 在容器中运行测试，只给最小能力
docker run --rm \
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  --cap-add=NET_RAW \
  -v $PWD:/app \
  -w /app \
  alpine:latest \
  sh -c "apk add --no-cache gcc musl-dev libcap-dev && \
         gcc -o test-server test-server.c -lcap && \
         setcap cap_net_bind_service+ep test-server && \
         su -s /bin/sh nobody -c './test-server'"
```

## Capabilities 的局限性

尽管 Capabilities 是强大的工具，但也有一些局限性需要注意：

1. **粗粒度问题**：`CAP_SYS_ADMIN` 关联了太多系统调用，无法进一步细分。内核社区正在逐步拆分它，但目前仍是"万能钥匙"。

2. **文件系统不支持细粒度**：Capabilities 是进程级别的，不能对单个文件设置"读能力"或"写能力"——文件权限仍然由传统 DAC 控制。

3. **与 LSM 的交互复杂**：SELinux、AppArmor 的能力模型与 Linux Capabilities 不完全兼容，需要同时配置。

4. **Ambient 集的限制**：非 root 用户执行新程序时，除非使用 Ambient 集，否则能力会被丢失。这导致在脚本环境中使用 Capabilities 比较麻烦。

5. **不是银弹**：Capabilities 不能阻止利用内核漏洞的提权攻击。如果攻击者找到了内核 0day，能力再多也没用。

## 总结

Linux Capabilities 是从"全有或全无"的权限模型走向最小权限原则的关键工具。在实际使用中，记住几条原则：

- **能删则删**：默认启用的能力太多，先 `--cap-drop=ALL` 再按需添加
- **优先用能力替代 setuid**：setuid 是危险的粗粒度设计，尽可能用 `setcap` 替代
- **运行时降权**：先做初始化，然后主动放弃不需要的能力
- **警惕 CAP_SYS_ADMIN**：它几乎等于 root，不到万不得已不要给
- **审计不信任**：用 `getcap`、`getpcaps` 定期审计系统中的能力分配

从容器编排到嵌入式系统，从 Web 服务器到网络工具，Capabilities 已经成为 Linux 安全模型的基石之一。掌握它，不仅能写出更安全的代码，也能更深入地理解操作系统权限管理的设计哲学。