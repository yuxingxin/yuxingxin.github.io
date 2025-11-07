---
layout: post
title: Git使用过程中的一些常见场景问题总结
tags: Git
categories: Tools
date: 2017-12-11
---

之前在公司内部推Git，写了一份[git使用教程](https://devops.yuxingxin.com)，后来又在团队内部做了一次分享，内容是关于Git使用过程中经常会遇到的一些场景，并有了这份总结。

### git基础

基于feature的工作流

- 添加忽略文件   .gitignore (https://gitignore.io/)
- 基于develop分支开发：feature分支   bugfix分支   版本节点tag
- 问题排查: diff 、log  、reflog、blame
- 撤销操作: checkout  、reset、revert、commit --amend
- 删除操作: rm  clean
- 储藏操作: stash
- 分支操作：创建、删除（注意远程分支的删除）、切换、合并（--no-ff 、rebase）
- 标签操作

更多详细查看上面教程链接

### 场景

#### 1. 本地已经存在的项目/分支与如何远程仓库关联
```
git remote add origin <your-repo-git-url>
```
#### 2. 刚刚提交了的commit log发现错了，想修改
```
git commit --amend -m "your new log"
```
#### 3. 查看某次提交的日志和ID
```
git reflog
```
#### 4. 查看某次提交的内容
```
git show <commit_id>
```
#### 5. 只是修改了工作区的文件，想恢复到原来修改前的样子
```
git reset --hard HEAD
git checkout -- <file_name>
```
#### 6. 被修改的文件已经添加到了暂存区，想撤销添加
```
git reset --mixed HEAD
```
#### 7. 被修改的文件已经commit提交，想撤销提交
```
git reset --soft HEAD^
```
#### 8. 已经提交到远程主机的文件，想撤销
```
git revert <commit_id>
git revert HEAD
```
#### 9. 已经开发一半的功能，但是没有开发完，这时候有个bug要紧急处理，需要放下手头的功能，赶去修改BUG
```
// 保存现场
git stash  
// 恢复现场
git stash pop
```
#### 10. 加入过历史版本的文件，因某些原因被删除了想恢复
```
git checkout <commit_id> -- <file_name>
```
另外你也可以用reset命令来完成

#### 11. 需要单独把多次提交中的某一次提交从你的分支迁移到另外一个分支上，即跨分支应用commit
```
git cherry-pick <commit_id>
```
比如：我想把以下分支
```
A-B  master
   \
    C-D-E-F-G develop
```
中的D，F 两次提交移动到master分支，而保持其他commit不变，结果就像这样
```
A-B-D-F  master
       \
        C-E-G develop
```
那么，思路是将D，F 用cherry-pick应用到master分支上，然后将develop分支对master分支变基。
```
$ git checkout master  
$ git cherry-pick D  
$ git cherry-pick F  
$ git checkout develop  
$ git rebase master
```
注意有些情况下使用cherry-pick会存在冲突，解决方法和我们平时合并分支遇到冲突一样。

#### 12. 遇到文件冲突，可以手动解决，或者用你配置的工具解决，记得把文件标位resolved：add/rm
如：常见的拉取同事的代码合并引起冲突
```
1. 手动处理冲突
2. 文件标志位置为resolved：git add <file_name>
3. 继续合并  git merge --continue
当然也可以选择放弃合并：git merge --abort
```

#### 13. 让自己本地分支上面的每一次提交日志变得更有意义，有时候需要我们选择有意义的提交日志信息合并上去

比如我们在bugfix分支上面由于修改bug提交了很多次，修复好了之后，我们想把这些提交合并入我们的master分支
```
git checkout master
git merge --squash bugfix
git commit -m "bug fixed"
```

上面操作会将bugfix分支上的所有commit都合并为一个commit，并把它并入我们的master分支上去。这里还有一点需要注意的是：--squash含义代表的是本地内容与不使用该选项的合并结果相同，但是不提交，不移动HEAD指针，所以我们要另外多一条语句来移动我们的HEAD指针，即最后的commit。

#### 14. 有时候需要整理我们本地的commits，可以使用Squash

```
git rebase -i <commit>
```

举例：

```
git rebase -i HEAD~5

执行完后，Git会把所有commit列出来，让你进行一些修改，修改完成之后会根据你的修改来rebase。HEAD-5的意思是只修改最近的5个commit。

pick 033beb4 b1
pick b426a8a b2
pick c216era b3
pick d627c9a b4
pick e416c8b b5

# Rebase 033beb4..e416c8b onto 033beb4
#
# Commands:
#  p, pick = use commit
#  r, reword = use commit, but edit the commit message
#  e, edit = use commit, but stop for amending
#  s, squash = use commit, but meld into previous commit
#  f, fixup = like "squash", but discard this commit's log message
#  x, exec = run command (the rest of the line) using shell
#
# If you remove a line here THAT COMMIT WILL BE LOST.
# However, if you remove everything, the rebase will be aborted.
#
```

上面pick是要执行的commit指令，另外还有reword、edit、squash、fixup、exec这5个，具体的含义可以看上面的注释解释，比较简单，这里就不说了。
我们要合并就需要修改前面的pick指令：

```
pick 033beb4 b1
squash b426a8a b2
squash c216era b3
squash d627c9a b4
squash e416c8b b5
```

也就是下面这4个提交合并到最前面的那个提交里面，按esc，打上:wq提交保存离开。
接着是输入新的commit message

```
b
# This is a combination of 2 commits.
# The first commit's message is:
# b1
#
# This is the 2nd commit message:
#
# b2
#
# This is the 3rd commit message:
#
# b3
#
# This is the 4th commit message:
#
# b4
#
# This is the 5th commit message:
#
# b5
#
# Please enter the commit message for your changes. Lines starting
# with '#' will be ignored, and an empty message aborts the commit.
# Not currently on any branch.
# Changes to be committed:
# (use "git reset HEAD <file>..." to unstage)
#
# modified:   a.txt
#
```

其中第一行的b就是需要我们输入的新信息，同样编辑完保存，出现类似下面的信息：

```
Successfully rebased and updated refs/heads/develop.
```

最后可以用git log指令来验证commits是不是我们要变成的样子。

#### 15. 多人协作开发项目，想知道某个文件的当前改动情况
通常查问题时想知道某个文件的某部分代码是谁改动的，那么git blame 就派上用场了。
```
git blame <file_name>
```
你也可以具体指定到某一行或者某几行代码
```
git blame -L <start_line>,<end_line> <file_name>
```

#### 16. 执行push命令向多个仓库同时提交代码
有时候会做代码备份，将代码保存在几个不同的Git代码管理平台，这时候就需要用到了

```
修改本地仓库目录下.git/config文件

[core]
	repositoryformatversion = 0
	filemode = true
	bare = false
	logallrefupdates = true
	ignorecase = true
	precomposeunicode = true
[remote "origin"]
	url = git@github.com:yuxingxin/blog.git
    url = ……
    url = ……
	fetch = +refs/heads/*:refs/remotes/origin/*
```

如上 在remote处可以添加多个远程地址。

#### 17. 从多次提交中快速定位某一次提交的bug

```
# 开始 bisect
$ git bisect start

# 录入正确的 commit
$ git bisect good xxxxxx

# 录入出错的 commit
$ git bisect bad xxxxxx

# 然后 git 开始在出错的 commit 与正确的 commit 之间开始二分查找，这个过程中你需要不断的验证你的应用是否正常
$ git bisect bad
$ git bisect good
$ git bisect good
...

# 直到定位到出错的 commit，退出 bisect
$ git bisect reset
```

### 总结

当然了，git的一些常见场景，还远不止这些，限于本人能力有限，如果你在平时的工作中遇到一些很实用的命令，也欢迎反馈给我，我好一并学习。更多的详细可以参考之前总结的一系列文档: https://devops.yuxingxin.com。 学习git命令是一件很有意思的事情，我想它能帮助使用git命令的人更好的理解这一代码管理工具，从而不至于犯一些低级错误，MobDevGroup网站上面也分享过几个学习命令的网站，可以供参考：https://mobdevgroup.com/tools/assistant
