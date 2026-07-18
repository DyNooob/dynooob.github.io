---
layout: post
title: "Python 调试：从 print 到断点的渐进路线"
date: 2026-07-17 09:00:00 +0800
categories: [开发]
tags: [Python, 调试, 开发工具, 效率]
---

写 Python 的人，调试方式往往随着经验增长沿着一条清晰的路线演进。从最原始的 print，到断点调试，再到远程调试和日志分析。每种方式都有自己的适用场景。

## 第一级：print

最直接的方式。不需要任何工具，不需要配置，想打哪里打哪里。

```python
def process_data(items):
    print(f"收到 {len(items)} 条数据")
    result = []
    for i, item in enumerate(items):
        print(f"处理第 {i} 条: {item.get('id')}")
        # 处理逻辑...
    print(f"处理完成，返回 {len(result)} 条")
    return result
```

print 的缺点也很明显：代码里到处是调试输出，上线前要清理，清理不干净就留在生产环境里了。而且信息一多，根本分不清哪条输出是哪来的。

改进方案是用 logging 替代 print：

```python
import logging
logging.basicConfig(level=logging.DEBUG)
logger = logging.getLogger(__name__)

def process_data(items):
    logger.debug("收到 %d 条数据", len(items))
    # ...
```

logging 可以按级别控制输出，生产环境只开 WARNING 以上，调试时再开 DEBUG。

## 第二级：pdb

Python 内置的调试器，不需要安装任何东西。

```python
import pdb; pdb.set_trace()
```

程序执行到这行时会暂停，进入交互式调试界面。常用命令：

- `n` (next)：执行下一行
- `s` (step)：进入函数内部
- `c` (continue)：继续执行到下一个断点
- `p 变量名`：打印变量值
- `l`：显示当前行附近的代码
- `q`：退出调试

Python 3.7+ 可以用内置的 `breakpoint()` 函数，效果一样，但不需要 import：

```python
def calculate(a, b):
    result = a + b
    breakpoint()  # 停在这里
    return result * 2
```

## 第三级：IDE 断点调试

如果你用 VS Code 或 PyCharm，图形化断点调试比命令行 pdb 直观得多。

核心操作：
- 点击行号左侧设置断点
- 按 F5 启动调试
- 单步执行、查看变量、观察表达式
- 条件断点：只在满足条件时暂停

条件断点特别有用——比如循环 1000 次，只想知道第 500 次时变量是什么状态，不需要一帧一帧地按。

## 第四级：post-mortem 调试

程序崩溃了，但刚才没开调试器。post-mortem 调试可以事后分析：

```python
try:
    risky_operation()
except Exception:
    import pdb; pdb.post_mortem()
```

或者在命令行中：

```bash
python -m pdb script.py
```

程序崩溃后会自动进入调试器，可以查看当时的变量和调用栈。

## 第五级：远程调试

当代码运行在服务器、Docker 容器或嵌入式设备上时，本地调试器够不着，需要远程调试。

使用 debugpy（VS Code 的 Python 调试器）：

```python
import debugpy
debugpy.listen(("0.0.0.0", 5678))
print("等待调试器连接...")
debugpy.wait_for_client()
```

然后在本地 VS Code 中配置远程连接，attach 到远程进程。这个方案在调试 AI 推理服务、API 服务等场景中很实用。

## 第六级：日志驱动调试

生产环境不允许停服务，也不允许开调试器。这时候靠日志。

```python
import logging
import traceback

logger = logging.getLogger(__name__)

try:
    result = risky_operation()
except Exception:
    logger.error("操作失败", exc_info=True)
    # exc_info=True 会打印完整调用栈
```

关键是把日志打到足够的级别，遇到问题时能通过日志还原现场，而不是靠猜测。

## 实践中怎么选

| 场景 | 方式 |
|------|------|
| 快速验证想法 | print 或 logging.debug |
| 复现本地 bug | breakpoint() 或 IDE 断点 |
| 难复现的边界条件 | 条件断点 |
| 程序崩溃后分析 | post-mortem |
| 服务器/容器中 | 远程调试 (debugpy) |
| 生产环境 | 日志 + 监控 |

调试不是越高级越好。print 能解决的问题用 print，省时间。只有 print 搞不定的复杂问题时，才上断点或远程调试。