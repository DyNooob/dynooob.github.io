---
layout: post
title: "用 systemd 管理你的 AI 服务：自动重启与日志"
date: 2026-07-11 15:00:00 +0800
categories: [开发]
tags: [systemd, Linux, 运维, AI部署, 服务管理]
---

跑本地 LLM 服务的人越来越多——vLLM、ollama、ComfyUI、各种 API 网关。但很多人启动服务的方式还是 `nohup python server.py &`，进程挂了都不知道，重启全靠手动。

systemd 是现代 Linux 的服务管理器，用它管理 AI 服务比 nohup 靠谱得多。

## 一个标准的 service 文件

```ini
[Unit]
Description=vLLM Inference Service
After=network.target
Wants=network-online.target

[Service]
Type=simple
User=dynooob
WorkingDirectory=/home/dynooob/projects/vllm
Environment="CUDA_VISIBLE_DEVICES=0"
Environment="HF_HOME=/home/dynooob/.cache/huggingface"
ExecStart=/home/dynooob/.local/bin/vllm serve /models/Qwen2.5-7B \
    --port 8000 \
    --gpu-memory-utilization 0.9 \
    --max-model-len 8192
Restart=on-failure
RestartSec=10
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

核心参数解析：

- `Restart=on-failure`：进程非正常退出时自动重启
- `RestartSec=10`：重启前等 10 秒，防止频繁重启
- `StandardOutput=journal`：日志写入 systemd journal，用 `journalctl` 查看

## 常用操作

```bash
# 重载配置文件
sudo systemctl daemon-reload

# 启动服务
sudo systemctl start vllm

# 查看状态
sudo systemctl status vllm

# 查看实时日志
sudo journalctl -u vllm -f

# 开机自启
sudo systemctl enable vllm

# 停止
sudo systemctl stop vllm
```

## 处理 GPU 服务的特殊问题

### 显存泄漏

长时间运行的 AI 服务容易出现显存泄漏。可以加一个内存限制：

```ini
[Service]
# 限制物理内存 32GB，超限则重启
MemoryMax=32G
# 限制虚拟内存
MemorySwapMax=0
```

不过显存泄漏用 MemoryMax 管不到，更可靠的做法是周期性重启：

```ini
# 每天凌晨 4 点自动重启
[Unit]
Description=vLLM Service

[Service]
...

[Install]
WantedBy=multi-user.target

# 单独建一个 timer
```

创建 `/etc/systemd/system/vllm-restart.timer`：

```ini
[Unit]
Description=Daily restart of vLLM

[Timer]
OnCalendar=daily
Persistent=true

[Install]
WantedBy=timers.target
```

和对应的 service：

```ini
[Unit]
Description=Restart vLLM

[Service]
Type=oneshot
ExecStart=/usr/bin/systemctl restart vllm
```

### 多卡管理

多 GPU 可以通过环境变量控制：

```ini
Environment="CUDA_VISIBLE_DEVICES=0,1,2,3"
```

或者用 `NVIDIA_VISIBLE_DEVICES`（Docker 场景）。

## 日志管理

AI 服务的日志通常很大。journald 默认只保留 100MB，超过就删。可以调大：

```bash
# 查看当前日志限制
journalctl --vacuum-size=500M
```

永久配置（`/etc/systemd/journald.conf`）：

```
SystemMaxUse=2G
```

然后重启 journald：

```bash
sudo systemctl restart systemd-journald
```

## 一个完整的模板

```ini
[Unit]
Description=AI Inference Service
After=network.target
StartLimitIntervalSec=300

[Service]
Type=simple
User=dynooob
WorkingDirectory=/home/dynooob/projects
Environment="CUDA_VISIBLE_DEVICES=0"
Environment="HF_HOME=/home/dynooob/.cache/huggingface"
ExecStart=/path/to/start.sh
Restart=on-failure
RestartSec=15
StartLimitBurst=3
StandardOutput=journal
StandardError=journal
MemoryMax=32G

[Install]
WantedBy=multi-user.target
```

`StartLimitIntervalSec=300` + `StartLimitBurst=3` 的意思是：300 秒内最多重启 3 次，超过就不再尝试，避免 crash loop 把系统拖垮。

## 和 nohup 的对比

| 功能 | nohup | systemd |
|------|-------|---------|
| 自动重启 | ❌ | ✅ |
| 开机自启 | ❌ | ✅ |
| 日志管理 | 自己写文件 | journalctl |
| 资源限制 | ❌ | ✅ |
| 依赖管理 | ❌ | ✅ |
| 状态查询 | ps | systemctl status |

如果你的 AI 服务跑在 Linux 上，花 10 分钟写个 service 文件，比用 nohup 省心得多。