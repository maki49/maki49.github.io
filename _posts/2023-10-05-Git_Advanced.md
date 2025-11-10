---
layout: post
title: Git Advanced
categories: tech
---

### rebase family
A，B两分支都在共同的基上有各自的修改，如果希望把A分支（当前分支）的修改建立在B分支修改的基础上，即：共同的基 + B分支的修改 + A分支的修改：
```
git rebase <branch-B>
```
如果B分支在远程，可以拉取的同时rebase：
```
git pull --rebase origin <branch-B>
```
---
批量处理之前的n个commit：改名、多合一、删除
```
git rebase -i HEAD~n
```
pick保留，s/squash合并，r/reword改名，d/drop删除
---
### cherry-pick
将某个分支的连续某几个commit合并到当前分支
```
git cherry-pick <commit-id-begin>.. <commit-id-end> 
```
---