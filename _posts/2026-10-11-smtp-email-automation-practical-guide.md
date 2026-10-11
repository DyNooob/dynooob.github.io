---
layout: post
title: "SMTP 邮件自动化系统实战：cron + Python + 阿里企业邮箱"
date: 2026-10-11 14:30:00 +0800
categories: [技术开发, 自动化]
tags: [SMTP, Python, cron, 阿里企业邮箱, 邮件推送, HTML邮件]
---

# 背景

每天早上 6 点起床时，时政日报已经安静地躺在我的邮箱里——这背后是一套完整的自动化系统：cron 定时触发 → Python 抓取内容 → 渲染 HTML → SMTP 推送。

这套系统跑了两个月，期间踩过不少坑：SMTP 静默丢弃、IMAP 索引慢、base64 编码截断、HTML 渲染问题。今天把踩坑记录和最终方案整理出来。

# 系统架构

```
┌─────────────┐   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐
│ 06:00 cron  │ → │ 抓取/整理   │ → │ 渲染 HTML   │ → │ SMTP 推送   │
│ 06时政日报  │   │ Markdown    │   │ 看板邮件    │   │ 阿里邮箱    │
└─────────────┘   └─────────────┘   └─────────────┘   └─────────────┘
                                                    ↓
                                              dynooob@buno.dev
```

每日自动运行：
- **06:00** 抓取 8 个时政源，整理为 Markdown
- **06:10** 渲染 HTML 看板邮件，推送到邮箱

# 1. SMTP 基础配置

阿里企业邮箱的 SMTP 配置比较标准：

```python
import smtplib
from email.mime.text import MIMEText
from email.header import Header

env = {
    'host': 'smtp.qiye.aliyun.com',
    'port': 465,
    'user': 'ai@buno.dev',
    'pwd': 'your_password'
}

msg = MIMEText('<h1>Hello</h1>', 'html', 'utf-8')
msg['From'] = env['user']
msg['To'] = 'recipient@example.com'
msg['Subject'] = Header('测试邮件', 'utf-8')

server = smtplib.SMTP_SSL(env['host'], env['port'], timeout=30)
server.login(env['user'], env['pwd'])
server.sendmail(env['user'], ['recipient@example.com'], msg.as_string())
server.quit()
```

**关键点**：
- 端口 **465（SSL）** 比 25（明文）更稳定
- `timeout=30` 防止网络卡死
- 用 `MIMEText(html, 'html', 'utf-8')` 而不是 plain text

# 2. 环境变量管理

**不要把密码硬编码到代码里**——用 `.env` 文件 + `chmod 600`：

```python
# ~/.hermes/.env
EMAIL_ADDRESS=ai@buno.dev
EMAIL_PASSWORD=your_password
EMAIL_SMTP_HOST=smtp.qiye.aliyun.com
EMAIL_SMTP_PORT=465
```

```python
def load_env():
    env = {}
    with open('/home/dynooob/.hermes/.env') as f:
        for line in f:
            if '=' in line and not line.startswith('#'):
                k, v = line.split('=', 1)
                env[k.strip()] = v.strip().strip('"').strip("'")
    return env
```

**坑**：密码里如果有 `!` 历史字符，在 `.env` 里要用单引号包起来：`'password!123'`。

# 3. HTML 邮件渲染

Markdown → HTML 邮件比 Markdown → 网页**多 5 个坑**：

## 3.1 不要用 Markdown 表格

Markdown 表格在邮件客户端渲染**不一致**。改用 inline style div 网格：

```python
# ✗ 错误：Markdown 表格
"""
| 列1 | 列2 |
|---|---|
| a | b |
"""

# ✓ 正确：HTML 网格
table_html = """
<div style="display:grid; grid-template-columns:1fr 1fr; gap:8px;">
  <div>列1内容</div>
  <div>列2内容</div>
</div>
"""
```

## 3.2 inline style 不能用 `<style>` 块

邮件客户端会**过滤 `<style>` 标签**。所有样式必须 inline：

```python
# ✗ 错误
style_html = '<style>h1 {color: red;}</style><h1>标题</h1>'

# ✓ 正确
inline_html = '<h1 style="color: red;">标题</h1>'
```

## 3.3 base64 编码截断问题

HTML body 通过 base64 编码后传输。**SMTP 服务器可能截断大邮件**。我的解决方案：

```python
# 控制单封邮件大小 < 50KB
if len(html) > 50000:
    raise ValueError("HTML too large, split into multiple emails")

# 邮件标题加时间戳避免反垃圾误判
subject = f"时政日报 2026-10-10 【{datetime.now().strftime('%H%M%S')}]"
```

**坑**：阿里邮箱的 SMTP 会**静默接受然后丢弃**某些"重复主题"邮件（SMTP 返回 250 OK 但 Sent 文件夹看不到）。加时间戳解决。

## 3.4 段落 → blockquote 转换

颁奖辞、人物事例等应该在邮件里**突出显示**（blockquote 样式）：

```python
def md_to_html_simple(text):
    html = ''
    in_quote = False
    for line in text.split('\n'):
        line = line.strip()
        if not line:
            continue
        if line.startswith('> '):
            if not in_quote:
                html += '<blockquote style="border-left:3px solid #1976d2; padding:10px 14px; margin:8px 0; background:#f5f5f5; color:#1a1a1a; font-style:italic; border-radius:4px;">'
                in_quote = True
            html += escape_html(line[2:]) + '<br>'
        else:
            if in_quote:
                html += '</blockquote>'
                in_quote = False
            html += f'<div style="margin:6px 0; line-height:1.7; color:#1a1a1a;">{escape_html(line)}</div>'
    if in_quote:
        html += '</blockquote>'
    return html
```

# 4. 链接处理

邮件里的链接**必须显式构造 `<a href>`**，不能只放 URL 文本（很多客户端不识别）：

```python
def url_to_link(text):
    """将文本中的 URL 转成可点击的链接"""
    pattern = re.compile(r'(https?://[^\s\)\]]+)')
    return pattern.sub(
        lambda m: f'<a href="{m.group(1)}" target="_blank" style="color:#1976d2; text-decoration:underline; word-break:break-all;">{m.group(1)[:90]}</a>',
        text
    )
```

# 5. cron 定时任务

设计 cron 任务的关键是**幂等性 + 失败重试**：

```python
# send_email.py
def main():
    today = datetime.now().strftime('%Y-%m-%d')
    
    # 1. 读取 Markdown
    md_path = f'/home/dynooob/.hermes/data/daily_shizheng/{today}.md'
    if not os.path.exists(md_path):
        print(f"⚠️ 未找到 {md_path}")
        return  # 静默退出，不算失败
    
    # 2. 渲染 HTML
    html = build_html(today)
    
    # 3. 发送
    ok, err = send_email(f'时政日报 {today}', html)
    if ok:
        print(f"✅ 已发送")
    else:
        print(f"✗ 发送失败: {err}")
        sys.exit(1)  # cron 会收到非 0 exit code 并重试
```

`cron` 配置：

```cron
# 每天 06:00 抓取，06:10 发送邮件
0 6 * * * /usr/bin/python3 /home/dynooob/.hermes/scripts/fetch_news.py
10 6 * * * /usr/bin/python3 /home/dynooob/.hermes/scripts/send_email.py
```

# 6. 验证邮件到达

**SMTP 返回 250 OK 不等于邮件到达收件箱**。必须用 IMAP 主动验证：

```python
import imaplib
import time

def verify_email_sent(sent_folder_path, timeout=90):
    """等邮件进 Sent 文件夹"""
    M = imaplib.IMAP4_SSL('imap.qiye.aliyun.com', 993)
    M.login('ai@buno.dev', 'password')
    M.select(sent_folder_path)
    
    start = time.time()
    while time.time() - start < timeout:
        time.sleep(5)
        typ, data = M.search(None, 'ALL')
        nums = data[0].split()
        if nums:
            print(f"✅ 邮件进 Sent ({len(nums)} 封)")
            return True
    
    print(f"⚠️ 邮件没进 Sent（{timeout}秒超时）")
    return False
```

**坑**：阿里邮箱 IMAP 索引**有 30-90 秒延迟**——这是正常的，**不要因此判定失败**。

# 7. 时区问题

邮件日期用本地时区（CST +0800），不要用 UTC：

```python
from email.utils import format_datetime
from datetime import datetime, timezone, timedelta

tz = timezone(timedelta(hours=8))
msg['Date'] = format_datetime(datetime.now(tz))
```

# 8. 调试技巧

调试 SMTP 时启用 debug level 看真实交互：

```python
server = smtplib.SMTP_SSL(host, port, timeout=15)
server.set_debuglevel(1)  # 打印所有 SMTP 命令和响应
server.login(user, pwd)
server.sendmail(user, to_addrs, msg.as_string())
server.quit()
```

输出会显示：
- `mail from:<user@domain>` → `250 Mail Ok`
- `rcpt to:<recipient@domain>` → `250 Rcpt Ok` ⚠️ **这一步返回 250 不代表收件人有效！**
- `data` → `250 Data Ok: queued as freedom` ← **阿里邮箱入队**

# 9. 完整脚本

```python
#!/usr/bin/env python3
"""时政日报 HTML 邮件推送"""
import os
import smtplib
from email.mime.text import MIMEText
from email.header import Header
from datetime import datetime, timezone, timedelta

ENV_FILE = '/home/dynooob/.hermes/.env'

def load_env():
    env = {}
    with open(ENV_FILE) as f:
        for line in f:
            if '=' in line and not line.startswith('#'):
                k, v = line.split('=', 1)
                env[k.strip()] = v.strip().strip('"').strip("'")
    return env

def build_html(title, sections):
    """构建 HTML 看板邮件"""
    body = f'''
    <div style="max-width:680px; margin:0 auto; padding:24px 16px; font-family:-apple-system,sans-serif;">
      <div style="background:linear-gradient(135deg,#1a1a1a 0%,#2d2d2d 100%); color:#fff; padding:32px; border-radius:12px 12px 0 0;">
        <div style="font-size:32px; font-weight:700;">{title}</div>
      </div>
      <div style="background:#fff; padding:24px;">
        {sections}
      </div>
    </div>
    '''
    return body

def send_email(subject, html_body):
    env = load_env()
    addr = env['EMAIL_ADDRESS']
    pwd = env['EMAIL_PASSWORD']
    host = env['EMAIL_SMTP_HOST']
    port = int(env.get('EMAIL_SMTP_PORT', '465'))
    to_addr = env.get('EMAIL_RECIPIENT', addr)
    
    msg = MIMEText(html_body, 'html', 'utf-8')
    msg['From'] = addr
    msg['To'] = to_addr
    msg['Subject'] = Header(subject, 'utf-8')
    
    tz = timezone(timedelta(hours=8))
    msg['Date'] = format_datetime(datetime.now(tz))
    
    server = smtplib.SMTP_SSL(host, port, timeout=30)
    server.login(addr, pwd)
    server.sendmail(addr, [to_addr], msg.as_string())
    server.quit()
    
    return True

if __name__ == '__main__':
    today = datetime.now().strftime('%Y-%m-%d')
    html = build_html(f'日报 {today}', '<p>今日内容</p>')
    send_email(f'日报 {today}', html)
    print(f'✅ 已发送 {today}')
```

# 10. 常见错误

| 错误 | 原因 | 解决 |
|------|------|------|
| 535 Authentication failed | 密码错或 IMAP/SMTP 没授权 | 用专用密码，不是登录密码 |
| 554 DT:SPM 邮件被系统视为垃圾 | 主题/正文触发反垃圾 | 加时间戳 + 减少 emoji |
| 邮件发到 Sent 但收件人没收到 | 收件地址无效 | 用 IMAP 验证 `rcpt to` 真实有效 |
| HTML 渲染显示原文 | `<style>` 被过滤 | 全用 inline style |
| `UnicodeDecodeError` | Subject 含中文 | 用 `Header(subject, 'utf-8')` |

# 总结

SMTP + HTML 邮件自动化的关键点：

1. **环境变量管密码**（.env + chmod 600）
2. **inline style**（不要 `<style>` 块）
3. **base64 大小控制**（< 50KB）
4. **标题加时间戳**（避免反垃圾）
5. **IMAP 主动验证**（别只看 SMTP 返回）
6. **等 30-90 秒**（IMAP 慢是正常的）

整套系统现在每天自动工作：

```
06:00  fetch_news.py      抓取 → 整理 → Markdown
06:10  send_email.py      Markdown → HTML → SMTP
```

每天早上醒来，时政日报已安静躺在 dynooob 邮箱里。

---

**参考链接**：
- [Python smtplib 官方文档](https://docs.python.org/3/library/smtplib.html)
- [阿里企业邮箱 SMTP 设置](https://mail.qiye.aliyun.com/)
- [HTML Email Best Practices](https://www.htmlemailbestpractices.com/)