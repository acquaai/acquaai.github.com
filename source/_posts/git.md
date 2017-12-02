---
title: Daily use of git command
date: 2017-11-23 09:16:08
categories: DevOps
---
git 命令 | 说明
------- | -----
git config --global user.name "xxx" |
git config --global user.email "xxx@xxx.com" |
git config user.name |
git config user.email |
git init |
git status [-s] |
git add . |
git commit -m "xxx" |
git commit -am "xxx" | 已在管理库中的文件
git log [--oneline] [--stat] |
git diff [--cached] |
git diff HEAD |
git commit --amend --no-edit | 合并到上一次提交
git reset file |
git reset --hard HEAD | 移动指针
git reset --hard HEAD^^ [=HEAD~2] |
git reset --hard `id` |
git reflog [--hard `id`] |
git reset --hard HEAD@{3} |
git checkout `id` -- file |
git log --oneline --graph |
git branch dev |
git branch [-a][-r] |
git checkout dev |
git checkout -b dev |
git branch master |
git branch -d[D] dev |
git branch --merged | 显示所有已合并到当前分支的分支
git branch --no-merged | 显示所有未合并到当前分支的分支
git branch -m master master_copy | 本地分支改名
git merge --no-ff -m "keep merge info" dev |
git rebase dev |
git rebase [--continue][ ][ ] |
git stash |
git stash pop |
git show master@{yesterday} | 显示master分支昨天的状态
git stash | 暂存当前修改，将所有至为HEAD状态
git stash list | 查看所有暂存
git stash show -p stash@{0} | 参考第一次暂存
git stash apply stash@{0} | 应用第一次暂存

[Reference `git`](https://git-scm.com/docs)

