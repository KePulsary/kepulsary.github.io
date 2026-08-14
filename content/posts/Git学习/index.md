---
title: "Git学习"
date: 2022-03-02 00:00:00
url: /2022/03/02/Git学习/
updated: 2023-04-03 01:07:47
tags:
  - 基础知识
description: "Git学习笔记"
---

# Git学习

- [Git教程 - 廖雪峰的官方网站 (liaoxuefeng.com)](https://www.liaoxuefeng.com/wiki/896043488029600)
- [Git常用命令总结](https://www.jianshu.com/p/cdccfef91ae1)

## git命令前置

```shell
git init	把这个目录变成Git可以管理的仓库
git add readme.txt	把文件添加到仓库
git commit -m "wrote a readme file"	把文件提交到仓库
git status	仓库当前的状态
git diff readme.txt 查看difference
git reset --hard HEAD^	回退到上一个版本
git reset --hard commit id //指定版本
git log	查看历史记录
git reflog	回退到上一个版本
git checkout -- file	可以丢弃工作区的修改：
git reset HEAD <file>可以把暂存区的修改撤销掉
git rm	版本库中删除该文件
git checkout	其实是用版本库里的版本替换工作区的版本，无论工作区是修改还是删除，都可以“一键还原”。
git remote add origin git@github.com:L1aovo/LFI_exp.git	关联远程仓库
git push -u origin master	推送到远端
git remote rm <name>	删除远程库
git remote -v	查看远程库信息
git clone git@github.com:michaelliao/gitskills.git	克隆仓库
git checkout -b dev	创建dev分支，然后切换到dev分支
git branch dev	创建
git checkout dev	切换、
git merge dev	把dev分支的工作成果合并到master分支上
git branch -d dev	删除dev分支
git switch -c dev	创建并切换到新的dev分支，可以使用
git switch master	直接切换到已有的master分支
git branch	查看分支
git log --graph --pretty=oneline --abbrev-commit	看分支的合并情况
git log --graph命令可以看到分支合并图
git merge --no-ff -m "merge with no-ff" dev	强制禁用Fast forward模式
git stash	把当前工作现场“储藏”起来，等以后恢复现场后继续工作
git checkout -b issue-101	master创建临时分支
git stash list	刚才的工作现场
git stash apply	恢复
git stash pop	恢复的同时把stash内容也删了
git stash drop	来删除
git cherry-pick	我们就不需要在dev分支上手动再把修bug的过程重复一遍
git branch -D feature-vulcan	强行删除
git remote	查看远程库的信息
git push origin dev	推送其他分支
git rebase	把分叉的提交历史“整理”成一条直线
git tag <name>	就可以打一个新标签
git tag v0.9 f52c633	commit id
git tag	查看所有标签
git show <tagname>	查看标签信息
git tag -d v0.1	删除标签
git push origin v1.0 推到远端
git push origin --tags	一次性推送全部尚未推送到远程
git tag -d v0.9	本地删除
it push origin :refs/tags/v0.9	远程删除
git add -f App.class	强制添加到Git
git check-ignore -v App.class	检查规则
```

## 集中式和分布式版本控制系统的区别

先说集中式版本控制系统，版本库是集中存放在中央服务器的，而干活的时候，用的都是自己的电脑，所以要先从中央服务器取得最新的版本，然后开始干活，干完活了，再把自己的活推送给中央服务器。中央服务器就好比是一个图书馆，你要改一本书，必须先从图书馆借出来，然后回到家自己改，改完了，再放回图书馆。集中式版本控制系统最大的毛病就是必须联网才能工作

分布式版本控制系统根本没有“中央服务器”，每个人的电脑上都是一个完整的版本库，这样，你工作的时候，就不需要联网了，因为版本库就在你自己的电脑上。既然每个人电脑上都有一个完整的版本库，那多个人如何协作呢？比方说你在自己电脑上改了文件A，你的同事也在他的电脑上改了文件A，这时，你们俩之间只需把各自的修改推送给对方，就可以互相看到对方的修改了。分布式版本控制系统通常也有一台充当“中央服务器”的电脑，但这个服务器的作用仅仅是用来方便“交换”大家的修改，没有它大家也一样干活，只是交换修改不方便而已。

## 创建版本库

版本库又名仓库，英文名**repository**，你可以简单理解成一个目录，这个目录里面的所有文件都可以被Git管理起来，每个文件的修改、删除，Git都能跟踪，以便任何时刻都可以追踪历史，或者在将来某个时刻可以“还原”。

创建一个空目录

```shell
$ mkdir learngit
$ cd learngit
$ pwd
```

通过`git init`命令把这个目录变成Git可以管理的仓库

命令`git add`告诉Git，把文件添加到仓库：

```shell
$ git add readme.txt
```

命令`git commit`告诉Git，把文件提交到仓库：`-m`后面输入的是本次提交的说明，可以输入任意内容，当然最好是有意义的，这样你就能从历史记录里方便地找到改动记录。

```shell
$ git commit -m "wrote a readme file"
```

运行`git status`命令看看结果：`git status`命令可以让我们时刻掌握仓库当前的状态

```shell
$ git status
```

`git diff`这个命令看看：顾名思义就是查看difference，显示的格式正是Unix通用的diff格式

```shell
$ git diff readme.txt 
```

## 版本回退

在Git中，我们用`git log`命令查看历史记录：`git log`命令显示从最近到最远的提交日志

在Git中，用`HEAD`表示当前版本，也就是最新的提交`1094adb...`（注意我的提交ID和你的肯定不一样），上一个版本就是`HEAD^`，上上一个版本就是`HEAD^^`，当然往上100个版本写100个`^`比较容易数不过来，所以写成`HEAD~100`。

使用`git reset`命令：回退到上一个版本

```shell
$ git reset --hard HEAD^

$ git reset --hard commit id //指定版本
```

`git reflog`用来记录你的每一次命令：可以查看 `commit id`

把文件往Git版本库里添加的时候，是分两步执行的：

第一步是用`git add`把文件添加进去，实际上就是把文件修改添加到暂存区；

第二步是用`git commit`提交更改，实际上就是把暂存区的所有内容提交到当前分支。

因为创建Git版本库时，Git自动创建了唯一一个`master`分支，所以，现在，`git commit`就是往`master`分支上提交更改。

简单理解为，需要提交的文件修改通通放到暂存区，然后，一次性提交暂存区的所有修改。

为什么Git比其他版本控制系统设计得优秀?Git跟踪并管理的是修改，而非文件

`git checkout -- file`可以丢弃工作区的修改：

命令`git checkout -- readme.txt`意思就是，把`readme.txt`文件在工作区的修改全部撤销，这里有两种情况：

一种是`readme.txt`自修改后还没有被放到暂存区，现在，撤销修改就回到和版本库一模一样的状态；

一种是`readme.txt`已经添加到暂存区后，又作了修改，现在，撤销修改就回到添加到暂存区后的状态。

命令`git reset HEAD <file>`可以把暂存区的修改撤销掉（unstage），重新放回工作区

版本库中删除该文件，那就用命令`git rm`删掉，并且`git commit`：

把误删的文件恢复到最新版本：`git checkout`其实是用版本库里的版本替换工作区的版本，无论工作区是修改还是删除，都可以“一键还原”。

```shell
git remote add origin git@github.com:L1aovo/LFI_exp.git关联远程仓库

git push -u origin master
```

删除远程库，可以用`git remote rm <name>`命令。使用前，建议先用`git remote -v`查看远程库信息：根据名字删除，比如删除`origin`：

```
$ git remote rm origin
```

克隆仓库

```
$ git clone git@github.com:michaelliao/gitskills.git
```

创建`dev`分支，然后切换到`dev`分支：

```shell
$ git checkout -b dev
Switched to a new branch 'dev'
```

`git checkout`命令加上`-b`参数表示创建并切换，相当于以下两条命令：

```shell
$ git branch dev
$ git checkout dev //切换
Switched to branch 'dev'
```

把`dev`分支的工作成果合并到`master`分支上：

```shell
$ git merge dev
```

删除`dev`分支：

```shell
$ git branch -d dev
```

创建并切换到新的`dev`分支，可以使用：

```shell
$ git switch -c dev
```

直接切换到已有的`master`分支，可以使用：

```shell
$ git switch master
```

查看分支：`git branch`

创建分支：`git branch <name>`

切换分支：`git checkout <name>`或者`git switch <name>`

创建+切换分支：`git checkout -b <name>`或者`git switch -c <name>`

合并某分支到当前分支：`git merge <name>`

删除分支：`git branch -d <name>`

带参数的`git log`也可以看到分支的合并情况：

```shell
$ git log --graph --pretty=oneline --abbrev-commit
```

用`git log --graph`命令可以看到分支合并图。

Git会用`Fast forward`模式，但这种模式下，删除分支后，会丢掉分支信息。

如果要强制禁用`Fast forward`模式，Git就会在merge时生成一个新的commit，这样，从分支历史上就可以看出分支信息。

```shell
$ git merge --no-ff -m "merge with no-ff" dev
```

Git还提供了一个`stash`功能，可以把当前工作现场“储藏”起来，等以后恢复现场后继续工作：

```shell
$ git stash
```

`master`创建临时分支：

```shell
$ git checkout -b issue-101
```

用`git stash list`命令看刚才的工作现场

用`git stash apply`恢复，但是恢复后，stash内容并不删除，你需要用`git stash drop`来删除；

`git stash pop`，恢复的同时把stash内容也删了：

`cherry-pick`命令，让我们能复制一个特定的提交到当前分支：

用`git cherry-pick`，我们就不需要在dev分支上手动再把修bug的过程重复一遍。

```shell
$ git branch -d feature-vulcan
```

强行删除：

```shell
$ git branch -D feature-vulcan
```

查看远程库的信息，用`git remote`：

```shell
$ git remote
```

如果要推送其他分支，比如`dev`，就改成：

```shell
$ git push origin dev
```

当你的小伙伴从远程库clone时，默认情况下，你的小伙伴只能看到本地的`master`分支。不信可以用`git branch`命令看看：

命令`git rebase`把分叉的提交历史“整理”成一条直线，看上去更直观。缺点是本地的分叉提交已经被修改过了。

命令`git tag <name>`就可以打一个新标签：

```shell
$ git tag v1.0
```

命令`git tag`查看所有标签：

```shell
$ git tag
```

找到历史提交的commit id，然后打上

比方说要对`add merge`这次提交打标签，它对应的commit id是`f52c633`，敲入命令：

```shell
$ git tag v0.9 f52c633
```

`git show <tagname>`查看标签信息：

还可以创建带有说明的标签，用`-a`指定标签名，`-m`指定说明文字：

```shell
$ git tag -a v0.1 -m "version 0.1 released" 1094adb
```

标签打错了，也可以删除：

```shell
$ git tag -d v0.1
```

命令`git push origin <tagname>`：

```shell
$ git push origin v1.0
```

一次性推送全部尚未推送到远程的本地标签：

```shell
$ git push origin --tags
```

如果标签已经推送到远程，要删除远程标签就麻烦一点，先从本地删除：

```shell
$ git tag -d v0.9
```

然后，从远程删除。删除命令也是push，但是格式如下：

```shell
$ git push origin :refs/tags/v0.9
```

Git工作区的根目录下创建一个特殊的`.gitignore`文件，然后把要忽略的文件名填进去，Git就会自动忽略这些文件。

不需要从头写`.gitignore`文件，GitHub已经为我们准备了各种配置文件，只需要组合一下就可以使用了。所有配置文件可以直接在线浏览：[https://github.com/github/gitignore](https://github.com/github/gitignore)

有些时候，你想添加一个文件到Git，但发现添加不了，原因是这个文件被`.gitignore`忽略了.如果你确实想添加该文件，可以用`-f`强制添加到Git：

```shell
$ git add -f App.class
```

`.gitignore`写得有问题，需要找出来到底哪个规则写错了，可以用`git check-ignore`命令检查：

```shell
$ git check-ignore -v App.class
```
