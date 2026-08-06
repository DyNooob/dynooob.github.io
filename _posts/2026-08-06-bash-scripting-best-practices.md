---
layout: post
title: "Bash 脚本编写规范与调试实战指南"
date: 2026-08-06 09:00:00 +0800
categories: [开发]
tags: [bash, shell, 脚本, 调试, Linux, 运维]
---

## 前言

Bash 是 Linux 运维和开发中最常用的脚本语言。但是，绝大多数人写 Bash 脚本的方式是"写出来能跑就行"。这种心态在脚本只有三五行时没有问题，一旦脚本增长到几百行、需要处理异常、接受外部输入、作为 CI/CD 流水线的一环运行时，各种隐患就会暴露出来：变量未定义导致灾难、管道中间失败但继续执行、临时文件残留、退出码错误掩盖问题……

本文从实战出发，总结一套可落地的 Bash 脚本规范，覆盖错误处理、调试技术、安全编码和 CI 集成，帮助你把脚本从"能跑"提升到"可靠"。

## 一、脚本头与基础安全设置

### 1.1 Shebang

```bash
#!/usr/bin/env bash
```

推荐使用 `#!/usr/bin/env bash` 而非 `#!/bin/bash`，因为不同系统上 bash 的安装路径可能不同（如 FreeBSD 的 `/usr/local/bin/bash`）。`env` 会在 `PATH` 中查找，提高可移植性。

### 1.2 必备的 set 选项

脚本的第二行（实际有意义的第一行）应该是 `set` 命令：

```bash
set -euo pipefail
```

每个选项的含义：

| 选项 | 作用 |
|------|------|
| `set -e` | 任何命令返回非零退出码时立即退出（errexit） |
| `set -u` | 使用未定义变量时报错退出（nounset） |
| `set -o pipefail` | 管道中任一命令失败，整个管道返回失败 |
| `set -E` | 让 `trap` 能捕获 `set -e` 触发的错误（与 `trap ERR` 配合） |

**为什么这三个缺一不可**：

- 没有 `-e`：`rm -rf "$dir"` 失败后脚本继续执行，可能操作错误的数据
- 没有 `-u`：`rm -rf "$dir/$prefix"` 中 `$prefix` 拼写成了 `$prefx`，变成 `rm -rf /`（如果 `dir` 是根目录）
- 没有 `pipefail`：`python setup.py | grep Success` 即使 `python` 崩溃也返回 0

生产脚本推荐写法：

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
```

### 1.3 IFS 安全

```bash
IFS=$'\n\t'
```

将内部字段分隔符（Internal Field Separator）限制为换行和制表符，避免文件名等包含空格时被意外分割。这是许多安全漏洞的根源。

## 二、错误处理机制

### 2.1 退出陷阱

用 `trap` 确保脚本在意外退出时执行清理：

```bash
cleanup() {
    local exit_code=$?
    echo "[$(date +'%H:%M:%S')] 清理临时文件..." >&2
    rm -rf "$TMPDIR"
    exit "$exit_code"
}
trap cleanup EXIT
```

`trap cleanup EXIT` 无论脚本是正常结束还是异常退出都会触发。`trap cleanup ERR` 则只在错误时触发（需要 `set -E` 配合）。

### 2.2 自定义错误处理

```bash
err_handler() {
    local line=$1
    local cmd=$2
    echo "[ERROR] 第 ${line} 行执行失败: ${cmd}" >&2
}
trap 'err_handler ${LINENO} "$BASH_COMMAND"' ERR
```

这在调试时极其有用：能精确定位到哪一行、哪个命令失败。

### 2.3 退出码规范

```bash
# 成功
exit 0

# 通用错误
exit 1

# 参数错误
exit 2

# 文件不存在
exit 3

# 权限不足
exit 4
```

自定义退出码的含义应在脚本头部的注释中说明。调用方可以通过 `$?` 获取退出码。

## 三、变量处理与安全

### 3.1 变量展开

使用花括号避免歧义，并利用 Bash 的变量展开语法做默认值、错误检查：

```bash
# 默认值 — 如果变量未设置，使用默认值
BACKUP_DIR="${BACKUP_DIR:-/var/backups}"

# 必须设置的变量 — 未设置时报错退出
DB_HOST="${DB_HOST:?必须设置 DB_HOST 环境变量}"

# 替换模式
FILENAME="${FILEPATH##*/}"    # 去掉路径前缀，取文件名
DIRNAME="${FILEPATH%/*}"      # 去掉文件名，取目录
EXT="${FILENAME##*.}"         # 取扩展名
BASENAME="${FILENAME%.*}"     # 去扩展名
```

### 3.2 变量引用

**所有变量展开必须用双引号包裹**，除非你明确希望分词（极少情况）：

```bash
# 正确 — 即使 $file 包含空格也安全
cat "$file"

# 错误 — 如果 $file 是 "my file.txt"，会尝试 cat "my" "file.txt"
cat $file
```

### 3.3 数组处理

数组是 Bash 中常被忽视的利器：

```bash
# 声明数组
FILES=()
while IFS= read -r -d '' f; do
    FILES+=("$f")
done < <(find . -name '*.log' -print0)

# 安全遍历（支持空格和特殊字符）
for f in "${FILES[@]}"; do
    echo "处理: $f"
done
```

`-print0` 配合 `read -d ''` 是处理文件名空格和换行的黄金组合，比 `for f in $(find ...)` 安全得多。

### 3.4 输入验证

```bash
validate_input() {
    local input=$1
    local pattern=$2

    if [[ ! "$input" =~ $pattern ]]; then
        echo "输入格式无效: $input" >&2
        return 1
    fi
}

# 使用示例 — 只允许字母数字和下划线
validate_input "$USERNAME" '^[a-zA-Z0-9_]+$'
```

## 四、函数与模块化

### 4.1 函数结构

```bash
# 函数名用动词+名词，snake_case
install_dependencies() {
    local pkg_list=("$@")
    local failed=0

    for pkg in "${pkg_list[@]}"; do
        if ! dpkg -l "$pkg" &>/dev/null; then
            echo "安装: $pkg"
            apt-get install -y "$pkg" || ((failed++))
        fi
    done

    return "$failed"
}
```

### 4.2 局部变量

函数内部**必须**使用 `local` 声明变量，否则会污染全局作用域：

```bash
# 错误 — 会覆盖全局的 $count
count_items() {
    count=0
    for i in "$@"; do ((count++)); done
    echo "$count"
}

# 正确
count_items() {
    local count=0
    for i in "$@"; do ((count++)); done
    echo "$count"
}
```

### 4.3 返回结果

Bash 函数只能返回整数退出码（0-255）。需要返回字符串时，使用以下方式：

```bash
# 方法1：echo 捕获（推荐）
get_config() {
    local key=$1
    grep "^${key}=" config.ini | cut -d= -f2
}
VALUE=$(get_config "db_host")

# 方法2：全局变量（显式约定）
get_config() {
    local key=$1
    CONFIG_VAL=$(grep "^${key}=" config.ini | cut -d= -f2)
}
get_config "db_host"
echo "Host: $CONFIG_VAL"
```

方法1更清洁，符合 Unix 管道哲学。

## 五、调试技巧

### 5.1 最基础的调试

```bash
# 对整个脚本开启追踪
bash -x script.sh

# 对特定代码段开启追踪
set -x
# ... 需要调试的代码 ...
set +x
```

`set -x` 会在每行执行前打印该行（已展开变量）到 stderr。

### 5.2 高级调试

```bash
# 自定义 PS4 显示行号和函数名
export PS4='+${BASH_SOURCE}:${LINENO}:${FUNCNAME[0]}: '

# 或简化的版本，只显示行号
export PS4='+ [$LINENO] '

# 只有函数名
export PS4='+ [${FUNCNAME[0]:+${FUNCNAME[0]}:}] '
```

调试输出示例：
```
+script.sh:42:main: apt-get install -y nginx
+script.sh:43:main: systemctl start nginx
+script.sh:44:check_health: curl -sf http://localhost/health
```

### 5.3 DEBUG 陷阱

`trap DEBUG` 会在每一条命令执行前触发，适合做细粒度日志：

```bash
debug_log() {
    echo "[DEBUG] $(date +'%T') 执行: $BASH_COMMAND" >> /tmp/debug.log
}
trap debug_log DEBUG
```

### 5.4 语法检查

```bash
# 只检查语法，不执行
bash -n script.sh

# 使用 shellcheck 做静态分析（推荐）
shellcheck script.sh

# 检查所有 .sh 文件
find . -name '*.sh' -exec shellcheck {} +
```

ShellCheck 是 Bash 脚本的"lint"，能检测出变量引用缺失引号、未使用的变量、`cd` 后未检查路径等数百种问题。在 CI 中集成 ShellCheck 应该作为代码质量的最低门槛。

### 5.5 详细的错误上下文

组合前面的技巧，得到一个完整的错误报告框架：

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

# 记录脚本开始时间和位置
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
SCRIPT_NAME="$(basename "${BASH_SOURCE[0]}")"
START_TIME=$(date +%s)

# 错误处理函数
error_handler() {
    local line=$1
    local cmd=$2
    local code=$3
    echo "[FATAL] ${SCRIPT_NAME}:${line} — 命令失败" >&2
    echo "  命令: ${cmd}" >&2
    echo "  退出码: ${code}" >&2
    echo "  时间: $(date '+%Y-%m-%d %H:%M:%S')" >&2
}

trap 'error_handler ${LINENO} "$BASH_COMMAND" $?' ERR

cleanup() {
    local exit_code=$?
    local elapsed=$(( $(date +%s) - START_TIME ))
    echo "[INFO] 脚本退出 (${exit_code})，耗时 ${elapsed}s" >&2
    # 清理临时文件
    rm -rf "${TMPDIR:-/tmp/${SCRIPT_NAME}.$$}"
}

trap cleanup EXIT

# 主逻辑
TMPDIR=$(mktemp -d "/tmp/${SCRIPT_NAME}.XXXXXX")
echo "[INFO] 开始执行，临时目录: ${TMPDIR}"
```

## 六、临时文件管理

### 6.1 安全创建临时文件

```bash
# 使用 mktemp（推荐）
TMPFILE=$(mktemp)
TMPDIR=$(mktemp -d)

# 指定模板，避免冲突
TMPFILE=$(mktemp "/tmp/myapp.XXXXXX")
TMPDIR=$(mktemp -d "/tmp/myapp.XXXXXX")
```

永远不要使用硬编码的临时路径（如 `/tmp/myapp.log`），多个并发实例会互相覆盖，恶意用户可以创建符号链接指向受害文件。

### 6.2 自动清理

```bash
cleanup() {
    rm -rf "${TMPDIR:-/tmp/fallback}"
}
trap cleanup EXIT
```

## 七、日志与输出

### 7.1 统一日志函数

```bash
# 日志级别
LOG_LEVEL="${LOG_LEVEL:-INFO}"

log() {
    local level=$1
    local msg=$2
    local timestamp
    timestamp=$(date '+%Y-%m-%d %H:%M:%S')

    # 过滤低于当前级别的日志
    case $LOG_LEVEL in
        DEBUG) ;;
        INFO)  [[ $level == DEBUG ]] && return ;;
        WARN)  [[ $level =~ ^(DEBUG|INFO)$ ]] && return ;;
        ERROR) [[ $level != ERROR ]] && return ;;
    esac

    case $level in
        ERROR) echo "[${timestamp}] [ERROR] ${msg}" >&2 ;;
        WARN)  echo "[${timestamp}] [WARN]  ${msg}" >&2 ;;
        *)     echo "[${timestamp}] [${level}] ${msg}" ;;
    esac
}

# 使用
log INFO "开始处理文件: ${filename}"
log ERROR "文件不存在: ${filename}"
```

### 7.2 输出重定向原则

```bash
# 错误信息必须输出到 stderr
echo "错误: 参数无效" >&2

# 正常输出到 stdout
echo "处理完成: 3 条记录"

# 调试信息只输出到日志文件
exec 3>>/var/log/myapp/debug.log
echo "调试: 变量值=${VAR}" >&3
```

## 八、CI/CD 集成

### 8.1 GitHub Actions 中的 ShellCheck

```yaml
# .github/workflows/lint.yml
name: Shell Lint
on: [push, pull_request]
jobs:
  shellcheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run ShellCheck
        run: |
          sudo apt-get install -y shellcheck
          shellcheck --severity=style $(find . -name '*.sh')
```

### 8.2 单元测试 Bats

[Bats](https://github.com/bats-core/bats-core) 是 Bash 的测试框架：

```bash
#!/usr/bin/env bats

setup() {
    source "${BATS_TEST_DIRNAME}/../lib/utils.sh"
}

@test "is_valid_ip 验证合法 IP" {
    run is_valid_ip "192.168.1.1"
    [ "$status" -eq 0 ]
}

@test "is_valid_ip 拒绝非法 IP" {
    run is_valid_ip "999.999.999.999"
    [ "$status" -eq 1 ]
}

@test "trim 去除首尾空格" {
    result="$(trim '  hello world  ')"
    [ "$result" = "hello world" ]
}
```

运行：`bats tests/`

## 九、综合示例：完整的生产级脚本

下面是一个融合了上述所有规范的脚本示例：

```bash
#!/usr/bin/env bash
# backup.sh — 数据库备份工具
# 使用: ./backup.sh <db_name> [output_dir]
# 退出码: 0=成功, 1=通用错误, 2=参数错误, 3=备份失败
set -Eeuo pipefail

readonly SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
readonly SCRIPT_NAME="$(basename "${BASH_SOURCE[0]}")"
readonly START_TIME=$(date +%s)

# === 日志 ===
log() {
    local level=$1 msg=$2
    echo "[$(date '+%H:%M:%S')] [${level}] ${msg}"
}

# === 错误处理 ===
error_handler() {
    log "FATAL" "第 ${1} 行执行失败: ${2} (退出码: ${3})"
}
trap 'error_handler ${LINENO} "$BASH_COMMAND" $?' ERR

cleanup() {
    local rc=$?
    local elapsed=$(( $(date +%s) - START_TIME ))
    rm -rf "${TMPDIR:-}"
    log "INFO" "退出 (${rc})，耗时 ${elapsed}s"
}
trap cleanup EXIT

# === 参数处理 ===
DB_NAME="${1:?用法: ${SCRIPT_NAME} <db_name> [output_dir]}"
OUTPUT_DIR="${2:-/var/backups/mysql}"
readonly DB_NAME OUTPUT_DIR

# === 验证 ===
if ! command -v mysqldump &>/dev/null; then
    log "ERROR" "mysqldump 未安装"
    exit 1
fi

if [[ ! -d "$OUTPUT_DIR" ]]; then
    mkdir -p "$OUTPUT_DIR"
fi

# === 主流程 ===
TMPDIR=$(mktemp -d "/tmp/${SCRIPT_NAME}.XXXXXX")
BACKUP_FILE="${OUTPUT_DIR}/${DB_NAME}_$(date +%Y%m%d_%H%M%S).sql.gz"

log "INFO" "开始备份: ${DB_NAME} → ${BACKUP_FILE}"

mysqldump --single-transaction --routines --triggers "$DB_NAME" \
    | gzip > "${TMPDIR}/dump.sql.gz"

# 验证备份文件完整性
if ! gzip -t "${TMPDIR}/dump.sql.gz"; then
    log "ERROR" "备份文件损坏"
    exit 3
fi

mv "${TMPDIR}/dump.sql.gz" "$BACKUP_FILE"

log "INFO" "备份完成: $(du -h "$BACKUP_FILE" | cut -f1)"
```

## 十、常见陷阱

```bash
# 陷阱1: cd 后未检查
cd /nonexistent/dir
rm -rf *          # 在错误目录执行灾难性操作

# 正确做法
cd /nonexistent/dir || { echo "无法进入目录" >&2; exit 1; }

# 陷阱2: 忘记引号
file="my file.txt"
rm $file           # 实际删除 "my" 和 "file.txt"

# 陷阱3: 管道中 set -e 失效
set -e
false | true       # 返回 0，set -e 不触发
# 需要 set -o pipefail

# 陷阱4: 使用 ls 遍历文件
for f in $(ls *.txt); do ...   # 文件名含空格会分裂

# 正确做法
for f in *.txt; do ...

# 陷阱5: eval 执行用户输入
eval "echo $user_input"  # 命令注入

# 正确做法：避免 eval，用关联数组或 case 替代
```

## 总结

写好 Bash 脚本不是靠记命令，而是靠建立一套规范化的编写习惯。本文的核心原则可以总结为：

1. **每一行都假设会出错** — `set -Eeuo pipefail` 是基础
2. **所有变量加引号** — 除非你明确知道为什么不需要
3. **函数内部用 `local`** — 隔离作用域，避免副作用
4. **用 `trap` 清理** — 不留临时文件，不泄漏资源
5. **用 `mktemp` 创建临时文件** — 安全和并发的保障
6. **日志输出到 stderr** — 错误信息不混入正常输出
7. **CI 中跑 ShellCheck** — 自动化代码审查

把这些规则内化成肌肉记忆，你的 Bash 脚本就能从"碰运气"变成"可靠运行"。