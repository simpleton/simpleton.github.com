---
layout: post
title: "Some tips about Git svn"
description: ""
category: git
tags: [git]
---
{% include JB/setup %}
### 背景###
最近工作上有一个svn的库（简称A库），由于一些原因，我们组只有一个同事有读的权限，并且我们还有基于这个库来干活。并且我们还没有svn的admin权限。。。于是只能发挥革命主义精神，自己搞了。

大题解决方案就是我建立了一个svn库供我们组同事使用（B 库），而我每天去同步A库。由于我们公司不能用git，所以只能用 git-svn 来将就下了。

### work flow###
大题步骤如下：
	1. 我在本地首先`git svn clone http://svn-repo.A` ;
	2. 然后我再添加一个remotes库，指向B库。	
	```
		git config --add svn-remote.${B}.url http://svn-repo.B
		git config --add svn-remote.${B}.fetch :refs/remotes/${B}
	```
	3. 我再拉去A的代码`git svn fetch A‘sname`(具体名字可以在.git/config里查)
	4. checkout出A库的最新代码 `git checkout -b tmp remotes/A`
	5. rebase B进来 `git rebase remotes/B`
	6. 解决冲突，这里如果偷懒就可以直接使用`git checkout --theirs .`
	7. 搞完冲突就可以提交B的代码了，这里rebase以后 index就会指向远端B的库。rebase么，已经把基点改变了。如果不放心可以交之前再查看一下。
	`git svn dcommit --dry-run`
	
### 坑###

这里还有一些坑：
	1. 首先win下和mac,linux的换行有所区别，rebase的时候会造成很多不必要的冲突。我们就把它忽略好了，在.git/config里增加：
	```
	    autocrlf = false
        safecrlf = false
	```	
	2. 更久的一些提交，A库会有一些很久的提交，这里是提交不上来的。如果要提交的话，只能是生成patch，再`git am`进来。 如果不care的话，就直接先空的B 库merge，然后就如work flow里所讲的那样操作即可。
	3. git format-patch A..B是 (A，B]的。
