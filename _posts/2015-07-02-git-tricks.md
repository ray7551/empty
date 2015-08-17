---
published: true
layout: default
title: Git Tricks
keywords: git
tags: git
---

###常用

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
git branch -b new-feature master
# push to new remote branch
git push origin new-featrue
# delete a local branch
git branch -d branchname
# delete a remote branch
git push origin --delete <branchName>
# same as
git push origin :<branchName>


```
 
###记住密码
　　每次提交时（HTTPS）都需要输入用户名和密码，网上提到的常见的解决方式是通过ssh提交，需要生成一个rsa key文件，关联到github，但是在某些网络环境下，不能通过ssh提交，怎么办呢？ 
　　只需要把以下内容写入~/.netrc，让linux记住你的用户名和密码。（https://gist.github.com/technoweenie/1072829） 
 
```
machine github.com
login username
password SECRET
 
machine api.github.com
login username
password SECRET
```
 
　　对windows环境，需要下载 `git-credential-winstore.exe` 到Git\libexec\git-core目录下,然后执行以下命令:
 
```
git config --global credential.helper winstore
```
 
###配置文件的处理
　　有部分文件，比如配置文件，需要在版本库中存在，但是需要针对每个本地环境做适当的修改，在提交时不希望提交的文件。对这类文件，应该如何处理呢？ 
　　gitignore只能忽略那些原来没有被track的文件，所以修改.gitignore是无效的。正确的做法是在每个clone下来的仓库中手动设置不要检查特定文件的更改情况。 

```
% git update-index --assume-unchanged /path/to/file
```
　　注意:这个方法要在每个本地环境运行一次。 
　　如果要重新track这个文件，用`--no-assume-unchanged`参数即可

```
git update-index --no-assume-unchanged /path/to/file
```
 
###对不需加入版本控制的文件的处理
　　已经commit的文件即使在.gitignore中也会被跟踪，需要使用如下命令来解除对这个文件版本控制： 

```
git rm --cached your_file
```
 
　　对于IDE生成的一些目录，或者类似vendor，node_modules这些不需要出现在git版本库中的文件，可以将他们添加进.git/info/exclude中

```
test.php            #匹配所有的test.php
/index.local.php    #匹配工作目录下的index.local.php
/res.local.php
/.idea/
/vendor/
 
```
 
 
###取消对某个文件的修改

```
git checkout -- <filename>
```
