---
title: "Git 常用命令"
date: 2019-01-16T23:43:52+08:00
categories:
    - Git
tags: 
    - Git
    - tips
---
## 基本设置
设置你自己的昵称与 email
```bash
git config --global user.name "zhaolion"
git config --global user.email "zhaoliangsyn@gmail.com"
git config --global push.default simple
```

配置你的编缉器
```bash
git config --global core.editor emacs
```
> Git 使用你的系统的缺省编辑器，这通常可能是 vi 或者 vim

配置你的比较工具
```bash
git config --global merge.tool vimdiff
```
> Git 可以使用 kdiff3, tkdiff, meld, xxdiff, emerge, vimdiff, gvimdiff, ecmerge, 和 opendiff 作为有效的合并工具。你也可以设置一个客户化的工具

检查你的配置
```bash
git config --list
```

编辑 Git 配置文件
```bash
git config -e [--global]
```

获取帮助
```bash
git help config
```

## 仓库相关命令
新建代码库
```bash
git init
```

检出仓库
```bash
git clone [repo]
```

查看远程仓库
```bash
git remote -v
```

添加远程仓库
```bash
git remote add [name] [url]
```

删除远程仓库
```bash
git remote rm [name]
```

修改远程仓库
```bash
git remote set-url --push [name] [newUrl]
```

拉取远程仓库
```bash
git pull [remoteName] [localBranchName]
```

推送远程仓库
```
git push [remoteName] [localBranchName]
```

下载远程仓库的所有变动
```bash
git fetch [remote]
```

显示某个远程仓库的信息
```bash
git remote show [remote]
```

增加一个新的远程仓库，并命名
```bash
git remote add [shortname] [url]
```

取回远程仓库的变化，并与本地分支合并
```bash
git pull [remote] [branch]
```

上传本地指定分支到远程仓库
```bash
git push [remote] [branch]
```

强行推送当前分支到远程仓库，即使有冲突
```bash
git push [remote] --force
```

推送所有分支到远程仓库
```bash
git push [remote] --all
```

## 分支 (branch) 操作相关命令
查看本地分支
```bash
git branch
```

查看远程分支
```bash
git branch -r
```

创建本地分支
```bash
git branch [name]
```
> 注意新分支创建后不会自动切换为当前分支

切换分支
```bash
git checkout [name]
```

创建新分支并立即切换到新分支
```bash
git checkout -b [name]
```

新建一个分支，指向指定 commit
```bash
git branch [branch] [commit]
```

新建一个分支，与指定的远程分支建立追踪关系
```bash
git branch --track [branch] [remote-branch]
```

切换到上一个分支
```bash
git checkout -
```

建立追踪关系，在现有分支与指定的远程分支之间
```bash
git branch --set-upstream [branch] [remote-branch]
```

选择一个 commit，合并进当前分支
```bash
git cherry-pick [commit]
```

删除分支
```bash
git branch -d [name]
```
> `-d` 选项只能删除已经参与了合并的分支，对于未有合并的分支是无法删除的。如果想强制删除一个分支，可以使用 `-D` 选项

合并分支
```bash
git merge [name]
```
> 将名称为 [name] 的分支与当前分支合并

创建远程分支 (本地分支 push 到远程)
```bash
git push origin [name]
```

删除远程分支
```bash
git push origin --delete [branch-name]
git branch -dr [remote/branch]
```

创建空的分支：(执行命令之前记得先提交你当前分支的修改，否则会被强制删干净没得后悔)
```bash
git symbolic-ref HEAD refs/heads/[name]
rm .git/index
git clean -fdx
```

## 增加 / 删除文件
添加指定文件到暂存区
```bash
git add [file1] [file2] ...
```

添加指定目录到暂存区，包括子目录
```bash
git add [dir]
```

添加当前目录的所有文件到暂存区
```bash
git add .
```

添加每个变化前，都会要求确认
```bash
git add -p
```

删除工作区文件，并且将这次删除放入暂存区
```bash
git rm [file1] [file2] ...
```

停止追踪指定文件，但该文件会保留在工作区
```bash
git rm --cached [file]
```

改名文件，并且将这个改名放入暂存区
```bash
git mv [file-original] [file-renamed]
```

## 代码提交 (commit)
提交暂存区到仓库区
```bash
git commit -m [message]
```

提交暂存区的指定文件到仓库区
```bash
git commit [file1] [file2] ... -m [message]
```

提交工作区自上次 commit 之后的变化，直接到仓库区
```bash
git commit -a
```

提交时显示所有 diff 信息
```bash
git commit -v
```

使用一次新的 commit，替代上一次提交
```
git commit --amend -m [message]
```
> 如果代码没有任何新变化，则用来改写上一次 commit 的提交信息
> 如果想直接使用上次提交信息 `git commit --amend --no-edit`

重做上一次 commit，并包括指定文件的新变化
```bash
git commit --amend [file1] [file2] ...
```

## 版本 (tag) 操作相关命令
查看版本
```bash
git tag
```

创建版本
```bash
git tag [name]
```

删除版本
```bash
git tag -d [name]
```

查看远程版本
```bash
git tag -r
```

查看 tag 信息
```bash
git show [tag]
```

创建远程版本 (本地版本 push 到远程)
```bash
git push origin [name]
```

删除远程版本
```bash
git push origin :refs/tags/[name]
```

合并远程仓库的 tag 到本地
```bash
git pull origin --tags
```

上传本地 tag 到远程仓库
```bash
git push origin --tags
```

创建带注释的 tag
```bash
git tag -a [name] -m 'message'
```

## 子模块 (submodule) 相关操作命令
添加子模块
```bash
git submodule add [url] [path]
```

初始化子模块
```bash
git submodule init
```
> 只在首次检出仓库时运行一次就行

更新子模块
```bash
git submodule update
```
> 每次更新或切换分支后都需要运行一下

删除子模块
```bash
git rm --cached [path]

编辑 ".gitmodules" 文件，将子模块的相关配置节点删除掉

编辑 ".git/config" 文件，将子模块的相关配置节点删除掉

手动删除子模块残留的目录
```

## 查看信息
显示有变更的文件
```bash
git status
```

显示当前分支的版本历史
```bash
git log
```

显示 commit 历史，以及每次 commit 发生变更的文件
```bash
git log --stat
```

搜索提交历史，根据关键词
```bash
git log -S [keyword]
```

显示某个 commit 之后的所有变动，每个 commit 占据一行
```bash
git log [tag] HEAD --pretty=format:%s
```

显示某个 commit 之后的所有变动，其 "提交说明" 必须符合搜索条件
```bash
git log [tag] HEAD --grep feature
```

显示某个文件的版本历史，包括文件改名
```bash
git log --follow [file]
git whatchanged [file]
```

显示指定文件相关的每一次 diff
```bash
git log -p [file]
```

显示过去 5 次提交
```bash
git log -5 --pretty --oneline
```

显示所有提交过的用户，按提交次数排序
```bash
git shortlog -sn
```

显示指定文件是什么人在什么时间修改过
```bash
git blame [file]
```

显示暂存区和工作区的差异
```bash
git diff
```

显示暂存区和上一个 commit 的差异
```bash
git diff --cached [file]
```

显示工作区与当前分支最新 commit 之间的差异
```bash
git diff HEAD
```

显示两次提交之间的差异
```bash
git diff [first-branch]...[second-branch]
```

显示今天你写了多少行代码
```bash
git diff --shortstat "@{0 day ago}"
```

显示某次提交的元数据和内容变化
```bash
git show [commit]
```

显示某次提交发生变化的文件
```bash
git show --name-only [commit]
```

显示某次提交时，某个文件的内容
```bash
git show [commit]:[filename]
```

显示当前分支的最近几次提交
```bash
git reflog
```
## 撤销
恢复暂存区的指定文件到工作区
```bash
git checkout [file]
```

恢复某个 commit 的指定文件到暂存区和工作区
```bash
git checkout [commit] [file]
```

恢复暂存区的所有文件到工作区
```bash
git checkout .
```

重置暂存区的指定文件，与上一次 commit 保持一致，但工作区不变
```bash
git reset [file]
```

重置暂存区与工作区，与上一次 commit 保持一致
```bash
git reset --hard
```

重置当前分支的指针为指定 commit，同时重置暂存区，但工作区不变

```bash
git reset [commit]
```

重置当前分支的 HEAD 为指定 commit，同时重置暂存区和工作区，与指定 commit 一致
```bash
git reset --hard [commit]
```

重置当前 HEAD 为指定 commit，但保持暂存区和工作区不变
```bash
git reset --keep [commit]
```

新建一个 commit，用来撤销指定 commit
```bash
git revert [commit]
```

暂时将未提交的变化移除，稍后再移入
```bash
git stash

git stash pop
```
