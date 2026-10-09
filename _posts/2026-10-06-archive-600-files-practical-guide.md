---
layout: post
title: "从混乱到有序：600+ 文件归档实战 — 工具、流程与避坑指南"
date: 2026-10-06 22:00:00 +0800
categories: [效率工具, 工程实践]
tags: [archive, rar, jekyll, file-management, organization, jupyter, obsidian, kb, automation, metadata]
---

## 600+ 文件从零开始整理是一种什么样的体验

当你拿到一份压成 RAR 的资料包，解压完发现里面 372 个条目、22 个解压失败、348 个文件散落在 76 个目录里，且文件名乱得像「创新小组.txt 实际是 JS 代码」的时候，你会发现——**单纯解压只是第一步**。

这篇文章记录了一次完整的「解压→分类→命名→索引」工作流，目标是把 600+ 文件变成一个**可搜索、可追溯、长期可维护**的知识库。每一步都有具体工具和可复用的脚本。

---

## 一、解压：选对工具比写脚本更重要

### 1.1 unrar 命令行 vs Python rarfile

Python `rarfile` 库依赖系统 unrar 工具，且对 RAR v5 格式兼容性差。**直接用命令行 unrar 更稳定**：

```bash
# 解压 RAR v5 标准格式
unrar x -y -r archive.rar ./output/
# -y 自动确认所有提示
# -r 递归子目录
```

如果系统没装 unrar 且没有 sudo：

```bash
# 下载 deb 包（Ubuntu/Debian）
mkdir -p ~/.local/bin
curl -L -o /tmp/unrar.deb http://archive.ubuntu.com/ubuntu/pool/universe/u/unrar-free/unrar-free_0.1.6-1_amd64.deb
dpkg -x /tmp/unrar.deb ~/.local/
ln -s ~/.local/usr/bin/unrar ~/.local/bin/unrar
export PATH=$HOME/.local/bin:$PATH
```

### 1.2 GBK 编码乱码的解决方案

中文文件名在 RAR 里经常是 GBK 编码，Linux 默认 UTF-8 会乱码。**Python rarfile 4.5 解压时 22 个文件失败**：

```
# 文件名含中文括号、双引号、中点的文件无法解压
FileNotFoundError: [Errno 2] No such file or directory
```

**接受限制，记录在 INDEX.md**，不强行修复。强行修复可能导致文件名错位，比乱码更难处理。

### 1.3 Silk 音频格式（如果遇到）

QQ/微信的 `.silk` 格式不在 ffmpeg 支持列表里，需要专用解码器：

```bash
git clone https://github.com/kn007/silk-v3-decoder
cd silk-v3-decoder/silk
make
# 解码 silk → pcm → wav
./decoder input.silk /tmp/output.pcm
ffmpeg -f s16le -ar 24000 -ac 1 -i /tmp/output.pcm /tmp/output.wav
```

---

## 二、分类：「场景驱动」而非「时间驱动」

### 2.1 双体系：个人 vs 工作 + 主题 vs 学院

大多数人会按学期或年份建文件夹：`2024-2025-1/`、`2024-2025-2/`。**这种分类的致命问题是：你找文件时不知道它是哪个学期的**。

更好的做法是**双体系**：

```
Archive/
├── 个人/                   # 我自己的东西
│   ├── 简历/
│   ├── 证件照/
│   ├── 专利申请/
│   └── 奖项_2025/
└── 工作/                   # 工作相关内容
    ├── 班务/              # 主题分类（按使用场景）
    │   ├── 01_花名册/
    │   ├── 02_值班请假/
    │   ├── 05_活动赛事/
    │   └── 10_班级证件照/
    ├── 学习笔记/
    ├── 项目/
    └── 湖北警官学院/      # 原始学院分类（保留）
        ├── 01_花名册/
        ├── 04_量化管理/
        └── 08_学院资料/
```

**关键原则**：
- **个人 vs 工作** = 顶层二分（所有权）
- **主题 vs 学院** = 二级分类（使用频率 vs 来源）
- 主题分类便于「找假条」「找活动」「找照片」
- 学院分类保留作为原始备份参考

### 2.2 主题目录的拆分粒度

不要一次性建 20 个分类，先看数据量：

```python
# 统计每个目录下文件数
for d in sorted(top_dirs):
    n = sum(1 for _, _, fs in os.walk(d) for _ in fs)
    print(f"{n:3d} {d}")
```

按文件数量动态调整主题边界——文件多的拆细，文件少的合并。

---

## 三、命名：「文件名是搜索引擎」

### 3.1 三种命名错误最常见

| 错误类型 | 案例 | 修复方式 |
|---------|------|----------|
| **rarfile 数字序号** | `1.jpg`、`2 (2).jpg` | 按内容重命名（如 `001_王世杰.jpg`）|
| **误导性原名** | `创新小组.txt`（实为 JS 代码）| 改名 `创新小组_实际为JS代码.txt` 或删除 |
| **缺关键信息** | `培训请假条.docx`（实为黄鹤实验室）| 改名 `黄鹤实验室培训请假说明_2025-01-08.docx` |

### 3.2 命名模板

按类型分类的命名模板（个人偏好）：

```
# 文档类：类别_事件_日期_版本.扩展名
2024级_网安_量化规则.xlsx
楚慧杯_L组_Darklightning_参赛名单_2024-12-30.docx

# 图片类：编号_姓名.扩展名（证件照）
001_王世杰.jpg
072_刘睿.jpg

# 资料类：主题_类型_日期.扩展名
度量衡数据库_deduction_rules_2024-11.xlsx
```

### 3.3 批量重命名脚本

```python
import os, re
from pathlib import Path

DIR = Path('/path/to/dir')
RENAME_MAP = {
    '创新小组.txt': '创新小组_实际为JS代码_删除.txt',
    '1.jpg': '001_王世杰.jpg',
    '培训请假条.docx': '黄鹤实验室培训请假说明_2025-01-08.docx',
}

for old, new in RENAME_MAP.items():
    src = DIR / old
    dst = DIR / new
    if src.exists() and not dst.exists():
        src.rename(dst)
        print(f'✓ {old} -> {new}')
```

### 3.4 删除空文件与冗余副本

```bash
# 找 0 字节文件
find . -type f -size 0

# 找相同 md5
md5sum * | sort | awk '{print $1}' | uniq -d
```

常见冗余：
- `.txt` 数据被 `.xlsx` 取代（早期版）
- 重复 doc（rarfile 解压 1 (2).jpg 这种）
- 损坏的空文件（0 字节）

---

## 四、识别：从「内容」而不是「名字」理解文件

### 4.1 docx 真假识别

很多 docx 文件名是「活动记录」，实际打开是「个人日记」。**用 zipfile + re 读 XML**：

```python
import zipfile, re

def read_docx_text(path):
    with zipfile.ZipFile(path) as z:
        with z.open('word/document.xml') as f:
            content = f.read().decode('utf-8')
    return re.findall(r'<w:t[^>]*>([^<]*)</w:t>', content)

texts = read_docx_text('some.docx')
print(' '.join(texts[:5]))  # 看前几段判断真实内容
```

### 4.2 图片 OCR（处理证件照胸牌号）

证件照整理时需要从胸牌读出学号，**vision_analyze 是最高效的方式**：

```python
# 给 vision_analyze 喂图片，问胸牌号
vision_analyze(
    image_url='/path/to/photo.jpg',
    question='照片右下角的胸牌编号？精确数字'
)
```

但 vision 一次只能看 1 张，**批量处理时**：
- 先按内容（rarfile 数字 / OCR 名字）分批
- 一次喂 10 张到 vision，问「所有人的胸牌号」
- 把结果批量重命名

### 4.3 学号 vs 胸牌号的对应关系

学生胸牌号和学号通常是**两个不同的编号系统**。本次遇到的：

```
胸牌号 = 24 + 06 + 学号末3位
即 胸牌号 №240695 = 学号末3位 095 = 学号 20240318095
```

**千万不要**用文件名「1.jpg」当作「1 号同学」。要从胸牌号反推学号末 3 位，再去花名册查真实姓名。

---

## 五、索引：每个目录都需要一份 INDEX.md

### 5.1 为什么 markdown 索引胜过树状结构

树状目录可以浏览，但**无法跨目录搜索**。**markdown 索引可以全文搜索、跨目录聚合**：

```markdown
---
tags: [archive, 24网安, 班级]
updated: 2026-10-06
---

# 📋 24网安全班花名册

## 一区（001-050）

| 学号末3位 | 姓名 | 备注 |
|----------|------|------|
| 001 | 王世杰 | 男 |
...
```

### 5.2 索引该写什么

每个 INDEX.md 至少包含：

```markdown
# 标题（清晰）
## 子目录说明
## 文件清单（含分类表）
## 关键统计（人数/日期/字节数）
## 备注（命名规则、缺失、版权）
```

### 5.3 Obsidian 双链集成

如果你用 Obsidian 当 KB：

```
KB/
└── 学习资料/
    └── 24网安全班花名册.md  ← 双向链接
```

Obsidian 的 `[[双链]]` 让「花名册」和具体人物之间形成网状结构，**比单纯的目录强大 10 倍**。

---

## 六、自动化：让归档变成「可重复的流程」

### 6.1 cron 任务保持活跃

归档完成后，**每天 08:00 自动跑一次「生日提醒」**就是典型的「自动化检查」：

```bash
#!/usr/bin/env bash
# /home/dynooob/.hermes/scripts/birthday_reminder.sh
PYTHON=/home/dynooob/.local/miniconda3/envs/dubin/bin/python
$PYTHON /home/dynooob/.hermes/scripts/birthday_reminder.py
```

```python
# birthday_reminder.py 核心
from datetime import date, timedelta
tomorrow = date.today() + timedelta(days=1)
target_mmdd = f"{tomorrow.month:02d}-{tomorrow.day:02d}"
matching = [b for b in birthdays if b['mmdd'] == target_mmdd]
if matching:
    print(f"🎂 明天 {tomorrow} 过生日:")
    for b in matching:
        print(f"  - {b['name']} ({b['sid'][-3:]})")
```

注册 cron（no_agent=true 模式）：每天 8:00 触发，提前一天提醒你。

### 6.2 元数据提取脚本

把重复操作变成可复用脚本：

```python
# ~/.hermes/scripts/birthday_extract.py
# 从花名册 xlsx 提取所有人生日 → JSON
import openpyxl, json
wb = openpyxl.load_workbook('花名册.xlsx', read_only=True, data_only=True)
birthdays = []
for row in wb.active.iter_rows(values_only=True):
    sid = str(row[1]) if row[1] else ''
    id_num = row[4] if row[4] else ''
    if id_num and len(id_num) >= 14:
        year = int(id_num[6:10])
        month = int(id_num[10:12])
        day = int(id_num[12:14])
        birthdays.append({'sid': sid, 'year': year, 'month': month, 'day': day,
                          'mmdd': f'{month:02d}-{day:02d}'})
json.dump(birthdays, open('birthdays.json', 'w'), ensure_ascii=False)
```

**数据驱动**比「硬编码提醒」更灵活——以后花名册更新，重新跑脚本即可。

---

## 七、避坑指南：常见 7 个雷区

1. **rarfile 数字序号陷阱**：`1.jpg`、`1 (2).jpg`、`1 (3).jpg` 不是 1/2/3 号同学，**是相机序列**。要从内容（OCR 胸牌号、文件元数据）反推真实身份。
2. **同名不同格式**：`第二章自测练习.docx` 和 `.pdf` 是同一内容的两种格式，**不要删其一**，命名加 `_v1`/`_v2` 区分。
3. **压缩包内的空格**：`李彦锐 优秀团员 .docx` 末尾空格会导致脚本处理失败。用 Python `zipfile` 重建压缩包修复。
4. **docx 里填的是银行卡号**：某些表单字段含义设计错误，同学把敏感信息填错了位置。**保留源文件，但意识到这可能是合规风险**。
5. **GBK 文件名解压失败**：接受限制，不强行修复。强行修复可能错位。
6. **保留原学期目录**：不要一次性删除原始 RAR 解压结构，**保留作为参考**。主题分类是导航层，原结构是备份层。
7. **批量操作前先备份**：重要的文件改动前 `git commit` 或 `cp` 备份。

---

## 八、总结：归档的核心是「可搜索」

| 维度 | 不好的做法 | 好的做法 |
|------|----------|----------|
| **分类** | 按时间（2024-2025-1）| 按使用场景（班务/01-13）|
| **命名** | `1.jpg`、`创新小组.txt` | `001_王世杰.jpg`、`24级网安_创新小组名单.txt` |
| **索引** | 没有 | 每个目录 INDEX.md |
| **数据** | 散落在 docx | 提取到 JSON |
| **自动化** | 手动提醒 | cron 每日检查 |

**核心目标**：未来找文件时，**3 次点击以内能找到**。如果不行，分类就有问题。

---

## 参考

- [unrar 命令行工具](https://www.rarlab.com/rar/unrar.html)
- [Jekyll 静态博客](https://jekyllrb.com/)
- [Obsidian 知识库](https://obsidian.md/)
- [rarfile Python 库](https://rarfile.readthedocs.io/)
