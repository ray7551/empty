---
published: true
layout: default
title: Git Tricks
keywords: git
tags: git
---

## 常用

```bash
# revert
# revert to previous commit, ignoring any changes:
git reset --hard HEAD
# revert a push
git reset --hard HEAD@{1}
git push -f
# revert a merge

# branch
# start a new branch based on master
git checkout -b new-feature master
# push to new remote branch
git push --set-upstream origin new-featrue

# merge current branch to master and push to remote
git checkout master
git merge new-feature
git push

# delete a local branch
git branch -d branchname
# delete a remote branch
git push origin --delete <branchName>
# same as
git push origin :<branchName>
```
 
## 记住密码
　　每次提交时（HTTPS）都需要输入用户名和密码，网上提到的常见的解决方式是通过ssh提交，需要生成一个rsa key文件，关联到github，但是在某些网络环境下，不能通过ssh提交，怎么办呢？  
　　只需要把以下内容写入~/.netrc，让linux记住你的用户名和密码。
```
machine github.com
login username
password SECRET
 
machine api.github.com
login username
password SECRET
```
 
## 配置文件的处理
　　有部分文件，比如配置文件，需要在版本库中存在，但是需要针对每个本地环境做适当的修改，在提交时不希望提交的文件。对这类文件，应该如何处理呢？   
　　gitignore 只能忽略那些原来没有被track的文件，所以修改 .gitignore 是无效的。正确的做法是在每个 clone 下来的仓库中手动设置不要检查特定文件的更改情况。 
```
% git update-index --assume-unchanged /path/to/file
```
　　注意:这个方法要在每个本地环境运行一次。  
　　如果要重新track这个文件，用`--no-assume-unchanged`参数即可
```
git update-index --no-assume-unchanged /path/to/file
```

## 对空目录的处理
　　对于某些必须存在的目录，我们希望这个目录出现在版本库中，但是只作为一个空目录存在；程序运行起来之后将会往这个目录添加很多临时文件，我们不希望这些临时文件添加到版本库中。对于这种情况，在目录下下添加一个 gitignore 文件即可：  
```
# Ignore everything in this directory
*
# Except this file
!.gitignore
```
　　需要注意的是，这必须是在 `git add` 这个目录之前。

## 对不需加入版本控制的文件的处理
　　已经commit的文件即使在 .gitignore 中也会被跟踪，需要使用如下命令来解除对这个文件版本控制： 
```
git rm --cached your_file
```
　　对于 IDE 生成的一些目录，或者类似 vendor，node_modules 这些不需要出现在 git 版本库中的文件，可以将他们添加进.git/info/exclude中
```
test.php            #匹配所有的test.php
/index.local.php    #匹配工作目录下的index.local.php
/res.local.php
/.idea/
/vendor/
```

## 取消对某个文件的修改
```
git checkout -- <filename>
```