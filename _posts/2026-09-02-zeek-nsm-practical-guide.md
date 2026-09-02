---
layout: post
title: "Zeek 网络流量监控与安全分析实战指南"
date: 2026-09-02 19:00:00 +0800
categories: [网络安全]
tags: [Zeek, NSM, 流量分析, 入侵检测, 网络安全, 威胁监控]
---

## 引言

网络流量监控是安全运营的基石。很多团队依赖 Snort/Suricata 做签名检测，但面对加密流量、未知攻击和复杂协议时，签名规则往往力不从心。Zeek（原名 Bro）走了另一条路：它不追求实时拦截，而是把流量还原成结构化日志，让分析师和安全工具自己去判断。

Zeek 的优势在于：

- **协议无关的日志体系**：200+ 个日志类型，覆盖 HTTP、DNS、TLS、SMTP、SSH、FTP 等主流协议
- **可编程的事件引擎**：用 Zeek 脚本（一种领域特定语言）编写自定义检测逻辑
- **文件提取**：自动从流量中提取 HTTP 传输的文件、邮件附件等
- **被动指纹识别**：识别操作系统、服务版本、浏览器指纹等
- **与现有工具链无缝集成**：输出 JSON 格式日志，可直接对接 Elasticsearch、Kafka、Splunk

本文从零搭建一个 Zeek 监控节点，配置核心检测策略，并通过一个实际攻击场景演示如何利用 Zeek 日志发现威胁。

## 一、Zeek 架构速览

Zeek 的架构分为两个层面：

**事件引擎（Event Engine）**：解析网络流量，将原始包转换为语义事件。例如，一个 TCP 三次握手完成后，Zeek 会触发 `tcp_connection_established` 事件；一个 HTTP 请求被解析后，会触发 `http_request` 事件。

**策略层（Policy Layer）**：用户编写的 Zeek 脚本在这些事件上挂载处理函数，执行检测、告警、记录等操作。Zeek 自带大量标准策略脚本，位于 `{prefix}/share/zeek/policy/` 目录下。

理解这个两层架构很重要：事件引擎负责"看得见"，策略层负责"想得通"。你不需要修改引擎代码，只需要写脚本就能扩展检测能力。

## 二、安装与基础配置

### 2.1 安装 Zeek

Zeek 支持主流 Linux 发行版。这里以 Ubuntu 22.04/24.04 为例：

```bash
# 添加官方仓库
echo 'deb http://download.opensuse.org/repositories/security:/zeek/xUbuntu_22.04/ /' | \
  sudo tee /etc/apt/sources.list.d/security:zeek.list
curl -fsSL https://download.opensuse.org/repositories/security:zeek/xUbuntu_22.04/Release.key | \
  gpg --dearmor | sudo tee /etc/apt/trusted.gpg.d/security_zeek.gpg > /dev/null

sudo apt update
sudo apt install zeek-lts
```

或者从源码编译（推荐生产环境，可以获得更多优化选项）：

```bash
sudo apt install -y cmake make gcc g++ flex bison libpcap-dev \
  libssl-dev python3 python3-dev swig zlib1g-dev

git clone --recursive https://github.com/zeek/zeek
cd zeek
./configure --prefix=/opt/zeek
make -j$(nproc)
sudo make install
```

安装完成后，将 Zeek 加入 PATH：

```bash
echo 'export PATH=/opt/zeek/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```

### 2.2 验证安装

```bash
zeek --version
# zeek version 6.x.x

zeek --help
```

### 2.3 基础配置

Zeek 的主配置文件是 `{prefix}/etc/zeekctl.cfg`。核心参数：

```ini
# /opt/zeek/etc/zeekctl.cfg
SitePolicyScripts = @LOAD local.zeek
LibDir = /opt/zeek/share/zeek
PrefixDir = /opt/zeek
LogDir = /var/log/zeek
SpoolDir = /var/spool/zeek
```

节点配置在 `{prefix}/etc/node.cfg` 中定义，指定 Zeek 监控哪个网卡：

```ini
# /opt/zeek/etc/node.cfg
[zeek]
type=standalone
host=localhost
interface=eth0
lb_method=pf_ring
lb_procs=1
```

如果你有多个网卡或需要负载均衡，可以定义多个节点。

## 三、启动 Zeek 并采集流量

### 3.1 使用 zeekctl 管理

Zeek 自带的 `zeekctl` 工具可以管理启动、停止、查看状态：

```bash
# 首次启动前需要部署配置
sudo zeekctl deploy

# 检查状态
sudo zeekctl status
# 输出类似：
# Name         Type       Host          Status      Pid    Started
# zeek         standalone localhost     running     12345  02 Sep 19:00:01

# 查看统计信息
sudo zeekctl netstats
# 输出类似：
#                 recv    drop   %drop     mem   ksent
# eth0         1234567    1234    0.10   123MB   67890
```

### 3.2 手动启动（调试用）

```bash
# 从 pcap 文件读取（离线分析）
zeek -r suspicious.pcap

# 实时监控（直接输出到终端）
sudo zeek -i eth0

# 加载自定义脚本
sudo zeek -i eth0 local.zeek
```

### 3.3 日志输出

启动后，Zeek 会在 `LogDir` 生成日志文件。最常用的日志包括：

| 日志文件 | 说明 |
|---------|------|
| `conn.log` | 所有网络连接记录 |
| `http.log` | HTTP 请求/响应 |
| `dns.log` | DNS 查询 |
| `ssl.log` | TLS 握手信息 |
| `ftp.log` | FTP 会话 |
| `smtp.log` | SMTP 邮件通信 |
| `notice.log` | Zeek 生成的告警 |
| `files.log` | 从流量中提取的文件信息 |
| `weird.log` | 协议异常事件 |

每个日志文件都是 tab-separated 格式（默认），也可以配置为 JSON 格式。

## 四、日志分析实战

### 4.1 连接日志（conn.log）—— 网络流量的基础视图

`conn.log` 是 Zeek 最核心的日志，记录了每条网络连接的五元组、时长、流量大小、状态等信息。

```bash
# 查看 conn.log 结构
head -5 /var/log/zeek/current/conn.log
```

输出示例：

```
#separator \x09
#fields	ts	uid	id.orig_h	id.orig_p	id.resp_h	id.resp_p	proto	service	duration	orig_bytes	resp_bytes	conn_state	local_orig	local_resp	missed_bytes	history	orig_pkts	orig_ip_bytes	resp_pkts	resp_ip_bytes	tunnel_parents
#types	time	string	addr	port	addr	port	enum	string	interval	count	count	string	bool	bool	count	string	count	count	count	count	set[string]
1234567890.123456	C1abcDEFghijklm 10.0.0.1 54321 93.184.216.34 80 tcp http 0.123456 350 1024 SF - - 0 ShADadfF 5 700 7 1500 (empty)
```

解读关键字段：

- **uid**：连接唯一标识，可在不同日志间关联
- **id.orig_h / id.orig_p**：源 IP 和端口
- **id.resp_h / id.resp_p**：目标 IP 和端口
- **proto**：协议类型（tcp, udp, icmp）
- **service**：Zeek 识别的应用层协议（http, dns, ssl, ssh 等）
- **duration**：连接持续时间
- **orig_bytes / resp_bytes**：发送/接收的字节数
- **conn_state**：连接状态（SF=正常关闭, S0=仅 SYN, REJ=拒绝, RSTO=重置等）

**实用分析命令：**

```bash
# 统计各服务协议的连接数
zeek-cut service < /var/log/zeek/current/conn.log | sort | uniq -c | sort -rn

# 找出传输数据量最大的连接
zeek-cut id.orig_h id.resp_h service orig_bytes resp_bytes < /var/log/zeek/current/conn.log \
  | sort -t$'\t' -k4,4rn | head -10

# 查找所有被拒绝的连接（扫描行为）
zeek-cut id.orig_h id.resp_h id.resp_p conn_state < /var/log/zeek/current/conn.log \
  | grep REJ | sort | uniq -c | sort -rn | head -10
```

### 4.2 HTTP 日志 —— Web 流量分析

```bash
# 查看 HTTP 请求
zeek-cut ts method host uri status_code user_agent < /var/log/zeek/current/http.log \
  | head -20
```

**检测场景：找出可疑的 User-Agent**

```bash
# 列出所有 User-Agent 及其出现次数
zeek-cut user_agent < /var/log/zeek/current/http.log \
  | sort | uniq -c | sort -rn | head -20
```

异常的 User-Agent（如空字符串、`curl/`、`python-requests/` 在非预期场景下大量出现）往往是自动化攻击的迹象。

### 4.3 DNS 日志 —— 检测 C2 通信

```bash
# 查看 DNS 查询
zeek-cut ts query qtype answers < /var/log/zeek/current/dns.log \
  | head -20
```

**检测场景：DGA（域名生成算法）或隧道流量**

```bash
# 查找高频率的 NXDOMAIN 响应（失败的 DNS 查询）
zeek-cut query rcode < /var/log/zeek/current/dns.log \
  | grep NXDOMAIN | cut -f1 | sort | uniq -c | sort -rn | head -20
```

大量的 NXDOMAIN 响应通常意味着 DGA 恶意软件在探测 C2 域名。

### 4.4 SSL 日志 —— 加密流量分析

```bash
# 查看 TLS 握手信息
zeek-cut ts server_name cipher version validated < /var/log/zeek/current/ssl.log \
  | head -20
```

**检测场景：使用自签名证书或异常证书的 TLS 连接**

```bash
# 找出证书验证失败的连接
zeek-cut ts server_name subject issuer_hash validation_status < /var/log/zeek/current/ssl.log \
  | grep -v "self-signed" | grep -v "ok"
```

## 五、加载检测策略

Zeek 自带丰富的检测脚本，位于 `{prefix}/share/zeek/policy/`。通过编辑 `local.zeek` 启用：

```bash
# /opt/zeek/share/zeek/site/local.zeek
```

启用常用检测策略：

```zeek
# 加载标准协议分析
@load protocols/http/detect-sqli
@load protocols/http/header-analysis
@load protocols/ssh/interesting-hostnames
@load protocols/ssl/validate-certs
@load protocols/ssl/known-certs

# 扫描检测
@load policy/misc/scan
@load policy/misc/bruteforce
@load policy/misc/detect-traceroute

# 文件分析
@load policy/frameworks/files/extract-all-files
@load policy/frameworks/files/hash-all-files

# 威胁情报
@load policy/frameworks/intel/seen
@load policy/frameworks/intel/do_notice

# 异常检测
@load policy/protocols/conn/weird-logging
@load policy/protocols/modbus/tracking
```

加载后重新部署：

```bash
sudo zeekctl deploy
```

### 扫描检测参数调优

扫描检测脚本默认值可能过于敏感。在 `local.zeek` 中调优：

```zeek
# 调整端口扫描阈值（默认 20 个端口/分钟）
redef Site::local_nets = { 192.168.0.0/16, 10.0.0.0/8 };

# 扫描检测阈值
redef Scan::port_scan_threshold = 15;
redef Scan::addr_scan_threshold = 30;

# 暴力破解检测阈值
redef BruteForce::password_guesses_threshold = 5;
redef BruteForce::summary_interval = 10mins;
```

## 六、编写自定义检测脚本

Zeek 脚本语言虽然看起来不太像主流语言，但其事件驱动模型非常直观。下面写一个检测 SSH 暴力破解的脚本：

```zeek
# /opt/zeek/share/zeek/site/detect-ssh-brute.zeek

module SSHBruteForce;

export {
    redef enum Notice::Type += {
        SSH_BruteForce_Detected,
        SSH_Successful_After_BruteForce,
    };

    # 阈值：同一源 IP 在窗口期内失败的认证次数
    const brute_threshold: count = 10;
    const brute_window: interval = 5mins;
}

# 记录每个源 IP 的失败尝试
global failed_attempts: table[addr] of count &create_expire=brute_window;

event ssh_auth_failed(c: connection, auth_method: string)
{
    local orig = c$id$orig_h;
    
    if (orig !in failed_attempts)
        failed_attempts[orig] = 0;
    
    failed_attempts[orig] += 1;
    
    if (failed_attempts[orig] == brute_threshold)
    {
        NOTICE([
            $note=SSH_BruteForce_Detected,
            $msg=fmt("SSH brute force detected from %s (%d failed attempts)", 
                     orig, failed_attempts[orig]),
            $src=orig,
            $identifier=cat(orig)
        ]);
    }
}

# 检测暴力破解后的成功登录
event ssh_auth_successful(c: connection, auth_method: string)
{
    local orig = c$id$orig_h;
    
    if (orig in failed_attempts && failed_attempts[orig] >= brute_threshold)
    {
        NOTICE([
            $note=SSH_Successful_After_BruteForce,
            $msg=fmt("Successful SSH login from %s after brute force (%d attempts)", 
                     orig, failed_attempts[orig]),
            $src=orig,
            $identifier=cat(orig)
        ]);
    }
}
```

在 `local.zeek` 中加载：

```zeek
@load ./detect-ssh-brute
```

重新部署后，Zeek 会在检测到 SSH 暴力破解时生成 `notice.log` 告警，而不再需要人工翻看 SSH 日志。

## 七、集成威胁情报

Zeek 的 Intel 框架可以加载外部威胁情报（IP、域名、URL、哈希等），自动标记匹配到的流量。

### 7.1 加载情报源

```bash
# 创建情报数据目录
sudo mkdir -p /opt/zeek/share/zeek/site/intel

# 下载 AlienVault OTX 情报（示例）
curl -o /opt/zeek/share/zeek/site/intel/alienvault.txt \
  https://reputation.alienvault.com/reputation.data
```

### 7.2 格式化情报

Intel 框架要求 tab-separated 格式：

```
#fields	indicator	indicator_type	meta.source	meta.url	meta.do_notice
10.0.0.5	Intel::ADDR	AlienVault	https://otx.alienvault.com	T
evil.example.com	Intel::DOMAIN	AlienVault	https://otx.alienvault.com	T
d41d8cd98f00b204e9800998ecf8427e	Intel::FILE_HASH	AlienVault	https://otx.alienvault.com	T
```

### 7.3 在 local.zeek 中加载

```zeek
@load frameworks/intel/seen
@load frameworks/intel/do_notice

redef Intel::read_files += {
    "/opt/zeek/share/zeek/site/intel/alienvault.txt",
};
```

### 7.4 关联情报日志

当流量匹配到情报条目时，Zeek 会在 `intel.log` 中记录，并触发 `notice.log` 告警。你可以通过 `zeek-cut` 快速查看哪些内部 IP 连接了已知恶意目标。

## 八、日志可视化与告警

### 8.1 JSON 输出

Zeek 默认输出 tab-separated 格式。切换到 JSON 格式可以更方便地对接日志系统：

```bash
# 在 local.zeek 中启用
@load policy/tuning/json-logs
```

### 8.2 对接 Elasticsearch

使用 Filebeat 或 Logstash 将 Zeek 日志送入 Elasticsearch：

```yaml
# filebeat.yml
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /var/log/zeek/current/*.log
  json.keys_under_root: true
  json.overwrite_keys: true
  
output.elasticsearch:
  hosts: ["http://localhost:9200"]
  index: "zeek-%{+yyyy.MM.dd}"
```

配合 Kibana 的 Zeek 预构建仪表板，可以快速搭建流量可视化看板。

### 8.3 实时告警

Zeek 本身不内置告警推送能力，但可以通过以下方式实现：

1. **Logstash + Alerting**：Logstash 读取 notice.log，匹配规则后通过 Email/Slack/Webhook 告警
2. **ElastAlert**：基于 Elasticsearch 的告警框架
3. **自定义脚本**：用 cron 定期扫描 notice.log 并推送

```bash
# 简单的告警脚本示例
#!/bin/bash
# /usr/local/bin/zeek-alert.sh
tail -n 0 -f /var/log/zeek/current/notice.log | while read line; do
    alert_severity=$(echo "$line" | zeek-cut note)
    alert_msg=$(echo "$line" | zeek-cut msg)
    
    if [ -n "$alert_msg" ]; then
        # 发送到 Slack
        curl -X POST -H "Content-type: application/json" \
          --data "{\"text\": \"Zeek Alert: $alert_severity - $alert_msg\"}" \
          https://hooks.slack.com/services/YOUR/WEBHOOK/URL
    fi
done
```

## 九、实战场景：检测 Cobalt Strike 通信

Cobalt Strike 是红队和勒索软件团伙常用的 C2 框架。其网络流量有一些特征可以通过 Zeek 检测。

### 9.1 HTTPS 证书特征

Cobalt Strike 默认使用自签名证书，且证书序列号往往为 0：

```bash
zeek-cut ts server_name subject issuer serial < /var/log/zeek/current/ssl.log \
  | grep "0$" | head -20
```

### 9.2 HTTP 请求特征

Cobalt Strike 的 HTTP Beacon 有特定的 URI 路径模式：

```zeek
# detect-cobaltstrike.zeek
event http_request(c: connection, method: string, original_uri: string,
                   unescaped_uri: string, version: string)
{
    # CS 默认的 URI 路径特征
    local cs_patterns = /\.php$/;
    
    if (cs_patterns in original_uri)
    {
        # 检查请求体大小
        local body_len = |c$http$body|;
        if (body_len > 0 && body_len < 1024)
        {
            NOTICE([
                $note=Notice::ACTION_LOG,
                $msg=fmt("Possible Cobalt Strike Beacon: %s %s from %s", 
                         method, original_uri, c$id$orig_h),
                $conn=c
            ]);
        }
    }
}
```

### 9.3 JA3 指纹识别

JA3 是 TLS 客户端指纹，Cobalt Strike 的默认 JA3 指纹是已知的。通过 Zeek 计算 JA3 并与已知恶意指纹对比：

```zeek
@load policy/protocols/ssl/ja3

event ja3_fingerprint(c: connection, ja3: string, ja3s: string)
{
    local malicious_ja3s: set[string] = {
        "72a589da5860247d6b3e94e4e9ea0c3b",  # CS 默认
        "f5e0a3b5c0e4e0f1a2b3c4d5e6f7a8b9",  # 其他已知恶意 JA3
    };
    
    if (ja3s in malicious_ja3s)
    {
        NOTICE([
            $note=Notice::ACTION_LOG,
            $msg=fmt("Known malicious JA3S fingerprint: %s from %s", 
                     ja3s, c$id$orig_h),
            $conn=c
        ]);
    }
}
```

## 十、性能优化建议

### 10.1 硬件选型

| 流量规模 | 推荐配置 |
|---------|---------|
| 100 Mbps 以下 | 4 vCPU, 8 GB RAM |
| 1 Gbps | 8 vCPU, 16 GB RAM |
| 10 Gbps | 16+ vCPU, 32 GB RAM + PF_RING/ZC |

### 10.2 软件优化

```ini
# zeekctl.cfg 性能相关参数
PinCPUs = 1,2,3    # 绑定 CPU 核心
lb_method = pf_ring
lb_procs = 4        # 负载均衡进程数
```

### 10.3 日志轮转

```bash
# 默认日志保留 7 天
sudo zeekctl cron
# 会生成 crontab 条目，自动执行日志轮转
```

调整日志保留策略：

```ini
# zeekctl.cfg
LogRotationInterval = 60    # 日志轮转间隔（分钟）
LogRetention = 30           # 日志保留天数
```

## 总结

Zeek 不是传统意义上的入侵检测系统，它更像一个网络流量取证平台。它不会替你做出"这是攻击"的判断，而是给你足够精确的数据让你自己下结论。这种哲学决定了它的优势：误报率低、可扩展性强、适合深度分析。

本文的实战要点：

1. 用 `conn.log` 理解网络流量全貌，关注异常的连接状态
2. 用 `dns.log` 发现 C2 通信的 DNS 行为
3. 用 `ssl.log` 识别恶意证书和加密流量特征
4. 编写 Zeek 脚本实现自定义检测逻辑
5. 集成威胁情报，自动标记已知恶意地址
6. 对接 Elasticsearch + Kibana 构建可视化平台

对于安全团队来说，Zeek 和 Suricata 不是二选一的关系——Suricata 做实时拦截，Zeek 做深度分析，两者互补才能构建完整的网络监控体系。