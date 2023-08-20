---
title: Git
tags: 
 - git
aliases: git
date created: 2023-04-01
date modified: 2023-08-20
---

当我们在 git 版本库中发现一个问题后，如你在 git 上对它进行了在线修改，但是没有对本地库进行同步（做到 push 之前，都先 pull 下代码，就可以保证本地库和远程库代码一致）。这个时候你再次 push，想把本地库提交到远程 git 库中，就会出现 push 失败问题：failed to push some refs to。

解决方法：
只要把远程库同步到本地库即可，使用如下命令：
`git pull --rebase origin master`
指令意思就是把远程库中的跟新合并到本地库中（可能存在冲突需要解决），--rebase 的作用是取消本地库中刚刚提交的 commit，并把他们接到更新后的版本库中。

或者使用如下命令，将 commit 的代码撤回，然后再 git pull：
`git reset --soft HEAD^`

