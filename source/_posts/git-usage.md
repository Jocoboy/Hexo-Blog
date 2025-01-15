---
title: Git常用命令
date: 2024-08-11 23:09:40
categories:
- Version-Control
tags:
- Git
---

协作开发时一些常用的git命令。

<!--more-->

## Git 常用命令

### rebase

git变基到[branch]

`git rebase [branch]`

git变基后合并冲突

`git rebase --cotinue`

合并代码前先进行变基操作(避免多出一条和分支修改无关的commit)

`git pull --rebase`

git变基修改多次提交

`git rebase -i [commit-id]` 或 `git rebase -i HEAD~n`

git取消变基

`git rebase --abort`

### reset

git重置提交到[commit-id]，并强制推送

`git reset --hard [commit-id]]`

`git push -f`

### revert

git回退到某一版本并提交

`git revert [commit-id]`

`git commit -m [commit-id]`

### others

git获取远程分支内容

`git fetch -a  origin/[branch]`

本地remote分支和远程同步

`git remote prune origin`

将本次修改的内容合并到上一次提交中，并强制推送

`git commit -a --amend`

`git push -f`

## Git 批处理

### filter-branch

本地git中的用户名和邮箱配置错误，会导致远程仓库提交记录中无法正确显示作者信息(用户名、头像等)，还会导致用户贡献无法统计。

如果只是更正单条作者信息，使用rebase即可。

`git rebase -i -p [commit-id]`

`git commit --amend --author="New Name Value <correct@example.com>" --no-edit`

`git rebase --continue`

若要重写所有历史commit中的授权用户和提交用户，可以使用`git filter-branch`编写批处理脚本，

```sh
# If you see bash tips "A previous backup already exists in refs/original/",
# that means you are not doing this for the first time.
# Just use '-f' to force to rewrite.
git filter-branch --env-filter '
WRONG_EMAIL="wrong@example.com"
NEW_NAME="New Name Value"
NEW_EMAIL="correct@example.com"

if [ "$GIT_COMMITTER_EMAIL" = "$WRONG_EMAIL" ]
then
    export GIT_COMMITTER_NAME="$NEW_NAME"
    export GIT_COMMITTER_EMAIL="$NEW_EMAIL"
fi
if [ "$GIT_AUTHOR_EMAIL" = "$WRONG_EMAIL" ]
then
    export GIT_AUTHOR_NAME="$NEW_NAME"
    export GIT_AUTHOR_EMAIL="$NEW_EMAIL"
fi
' --tag-name-filter cat -- --branches --tags
```

在git仓库根目录下运行后，使用`git push -f`强制推送即可。

## Git配置问题

### 文件路径过长

git在拉取或提交代码时，文件路径名过长会报错"filename too long"。

输入`git config --get core.longpaths`，发现返回结果为false。

可通过`git config core.longpaths true`命令，在当前项目中启用长路径名。

若想全局生效，可输入`git config --global core.longpaths true`。

## 参考文档

[Git官方中文文档](https://git-scm.com/docs/git/zh_HANS-CN)