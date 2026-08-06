---
layout: post
title: "Git 从入门到实战：核心概念、命令与工作流"
date: 2026-08-05 16:00:00 +0800
categories: [开发]
tags: [Git, 版本控制, 实战, 工作流, 精选]
---

Git 是现代开发的必备技能。本文从概念到实战，覆盖日常开发中最常用的操作。

## 核心概念

### 三个区域

Git 的工作区分为三层：

```
工作目录 (Working Directory)
    ↓ git add
暂存区 (Staging Area / Index)
    ↓ git commit
本地仓库 (Local Repository)
    ↓ git push
远程仓库 (Remote Repository)
```

- **工作目录**：你实际编辑文件的地方
- **暂存区**：下次要提交的文件清单
- **本地仓库**：已经提交到本地的历史记录
- **远程仓库**：服务器上的代码仓库（如 GitHub）

### 四类对象

Git 底层用四种对象存储一切：

- **Blob**：文件内容
- **Tree**：目录结构
- **Commit**：一次提交的快照
- **Tag**：指向某个 commit 的标签

每个对象用 SHA-1 哈希值唯一标识。

## 基础命令

### 配置

```bash
git config --global user.name "你的名字"
git config --global user.email "你的邮箱"
git config --global core.editor vim
```

### 创建仓库

```bash
# 从零开始
git init

# 克隆已有仓库
git clone https://github.com/用户名/仓库名.git

# 克隆指定分支
git clone -b 分支名 仓库地址
```

### 日常操作

```bash
# 查看状态
git status

# 添加文件到暂存区
git add 文件名        # 添加单个文件
git add .            # 添加所有改动
git add -p           # 交互式添加（逐块确认）

# 提交
git commit -m "提交信息"
git commit -am "提交信息"  # 跳过暂存区，直接提交已跟踪的文件

# 推送
git push origin main

# 拉取
git pull origin main

# 拉取但不合并
git fetch origin
```

### 查看历史

```bash
git log                  # 完整历史
git log --oneline        # 单行显示
git log --graph          # 图形化显示分支
git log -p               # 显示每次提交的 diff
git log --author="名字"  # 按作者过滤
git log --since="2026-01-01"  # 按时间过滤
git log --grep="关键字"  # 按提交信息搜索

# 查看某行代码的修改历史
git blame 文件名
```

## 分支管理

```bash
# 创建分支
git branch 分支名

# 切换分支
git checkout 分支名
git switch 分支名       # 新版 Git 推荐

# 创建并切换
git checkout -b 分支名

# 查看分支
git branch              # 本地分支
git branch -a           # 所有分支（含远程）

# 合并分支
git checkout main
git merge 功能分支名

# 删除分支
git branch -d 分支名    # 已合并的分支
git branch -D 分支名    # 强制删除（未合并）
```

## 撤销操作

```bash
# 撤销工作目录的修改
git restore 文件名

# 撤销暂存区的文件
git restore --staged 文件名

# 修改最近一次 commit 的信息
git commit --amend -m "新信息"

# 撤销最近一次 commit（保留修改）
git reset --soft HEAD~1

# 撤销最近一次 commit（丢弃修改）
git reset --hard HEAD~1

# 撤销某个历史 commit（创建新 commit 来撤销）
git revert 提交哈希
```

## 远程仓库

```bash
# 查看远程仓库
git remote -v

# 添加远程仓库
git remote add origin 仓库地址

# 修改远程仓库地址
git remote set-url origin 新地址

# 推送新分支到远程
git push -u origin 分支名

# 删除远程分支
git push origin --delete 分支名
```

## 暂存工作

```bash
# 暂存当前修改
git stash

# 暂存并加备注
git stash save "备注信息"

# 查看暂存列表
git stash list

# 恢复最近一次暂存
git stash pop

# 恢复但不删除暂存记录
git stash apply

# 删除暂存记录
git stash drop
```

## 实战场景

### 场景一：修复紧急 bug

```bash
# 当前在 feature 分支开发到一半
git stash                 # 暂存当前工作
git checkout main
git checkout -b hotfix    # 从 main 创建修复分支
# 修复 bug...
git add .
git commit -m "fix: 修复登录崩溃"
git push origin hotfix
# 提 PR 合并到 main
git checkout feature      # 回到原分支
git stash pop             # 恢复工作
```

### 场景二：合并冲突

冲突标记长这样：

```
<<<<<<< HEAD
你本地的代码
=======
远程的代码
>>>>>>> branch-name
```

处理步骤：

1. 打开文件，删除 `<<<<<<<`、`=======`、`>>>>>>>`
2. 保留正确的代码
3. 保存文件
4. `git add 文件名`
5. `git commit`

### 场景三：不小心提交到 main

```bash
# 在 main 上提交了不该提交的内容
git branch feature       # 先创建分支保住提交
git reset --hard HEAD~1  # main 回退
git checkout feature     # 切换到新分支继续开发
```

### 场景四：找回误删的 commit

```bash
git reflog               # 查看所有历史操作
# 找到要恢复的 commit 哈希
git checkout -b 恢复的分支 哈希值
```

## 常用配置别名

```bash
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.lg "log --oneline --graph --all"
```

配置后可以用 `git st` 代替 `git status`。

## 快速参考

| 需求 | 命令 |
|------|------|
| 查看状态 | `git status` |
| 查看历史 | `git log --oneline --graph` |
| 撤销本地修改 | `git restore 文件` |
| 撤销暂存 | `git restore --staged 文件` |
| 暂存工作 | `git stash` |
| 恢复暂存 | `git stash pop` |
| 修改最近 commit | `git commit --amend` |
| 撤销历史 commit | `git revert 哈希` |
| 查看所有操作 | `git reflog` |