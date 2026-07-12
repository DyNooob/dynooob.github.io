---
layout: post
title: "Git 工作流：日常协作中常见的坑与解法"
date: 2026-07-11 19:00:00 +0800
categories: [开发]
tags: [Git, 工作流, 版本控制, 协作]
---

Git 的基本操作大多数人都会——add、commit、push、pull。但日常协作中真正让人头疼的不是这些，而是分支乱了、冲突不知道怎么解、合完发现少了东西。

这篇文章整理几个实际工作中经常遇到的问题和对应的处理方式。

## 场景一：提交信息写错了还没推送

```bash
# 修改最近一次 commit 的 message
git commit --amend -m "正确的提交信息"
```

如果 commit 已经推送了，不要 amend 后直接 push，会冲突。需要 force push：

```bash
git commit --amend -m "正确的提交信息"
git push --force-with-lease
```

`--force-with-lease` 比 `--force` 安全，它会检查远程分支有没有新提交，防止覆盖别人的工作。

## 场景二：提交后发现漏了文件

```bash
# 把漏掉的文件加进来，合并到上一个 commit
git add forgotten-file.py
git commit --amend --no-edit
```

`--no-edit` 表示沿用之前的提交信息不改。

## 场景三：想拆开一个 commit

已经 commit 了，但发现这个 commit 改了两件不相干的事：

```bash
# 撤销最近一次 commit，但保留工作区修改
git reset HEAD~1

# 或者用 soft reset（保留 staging area 的内容）
git reset --soft HEAD~1
```

然后重新分步 add 和 commit。

## 场景四：合并冲突

冲突标记长这样：

```
<<<<<<< HEAD
你本地的代码
=======
远程的代码
>>>>>>> branch-name
```

处理方式：

1. 手动编辑文件，删掉 `<<<<<<<`、`=======`、`>>>>>>>`，保留正确的代码
2. 保存文件
3. `git add 文件名`
4. `git commit`（或 `git merge --continue`）

如果合并到一半想放弃：

```bash
git merge --abort
```

## 场景五：不小心在 main 分支上改了代码

应该在 feature 分支上开发，但直接在 main 上写了代码，还没 push：

```bash
# 创建一个新分支，把改动带过去
git checkout -b feature/my-feature

# main 分支回到之前的状态
git branch -f main origin/main
```

如果已经 push 了，需要 force push main 回退，但可能影响其他人。更稳妥的做法是 rever：

```bash
# 在 main 上回退到某个 commit
git revert HEAD
```

## 场景六：rebase 还是 merge

这是 Git 工作流里最经典的争论。我的建议：

- 合并到主干（main）用 merge，保留历史
- 在自己的分支上同步主干的更新用 rebase，保持提交线干净

```bash
# 在 feature 分支上同步 main 的更新
git checkout feature/my-feature
git rebase main

# 如果有冲突，解完冲突后
git add 文件名
git rebase --continue

# 如果 rebase 到一半想放弃
git rebase --abort
```

rebase 的核心原则：**不要 rebase 已经推送到公共仓库的分支**。

## 场景七：.gitignore 不生效

`.gitignore` 只对 untracked 文件生效。如果某个文件已经被 Git 跟踪了，加 ignore 也没用：

```bash
# 先停止跟踪，但不删除本地文件
git rm --cached 文件名

# 然后 .gitignore 才会生效
```

批量操作：

```bash
# 停止跟踪所有 __pycache__ 目录
git rm -r --cached __pycache__/
echo "__pycache__/" >> .gitignore
```

## 场景八：找回误删的 commit

```bash
# 查看所有历史操作（包括已被删除的 commit）
git reflog

# 找到要恢复的 commit hash，然后
git checkout -b recovered-branch 哈希值
```

`reflog` 是 Git 的救命稻草。只要你的 commit 曾经存在过，reflog 里就有记录（默认保留 90 天）。

## 快速参考

| 需求 | 命令 |
|------|------|
| 修改最近 commit | `git commit --amend` |
| 撤销 staging | `git reset HEAD 文件名` |
| 撤销本地修改 | `git checkout -- 文件名` |
| 暂存当前工作 | `git stash` |
| 恢复暂存 | `git stash pop` |
| 查看历史操作 | `git reflog` |
| 安全 force push | `git push --force-with-lease` |
| 放弃合并 | `git merge --abort` |
| 放弃 rebase | `git rebase --abort` |

这些场景覆盖了日常协作中 90% 的"卡住"时刻。遇到问题先想想 git 有没有对应的命令，通常都有——不用删仓库重来。