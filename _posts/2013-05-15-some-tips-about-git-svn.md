---
layout: post
title: "Some tips about Git svn"
description: ""
category: git
tags: [git]
---
{% include JB/setup %}
## 背景##
最近工作上有一个svn的库（简称A库），由于一些原因，我们组只有一个同事有读的权限，并且我们还有基于这个库来干活。并且我们还没有svn的admin权限。。。于是只能发挥革命主义精神，自己搞了。

大题解决方案就是我建立了一个svn库供我们组同事使用（B 库），而我每天去同步A库。由于我们公司不能用git，所以只能用 git-svn 来将就下了。

## work flow##
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
	
## 坑##

这里还有一些坑：
1. 首先win下和mac,linux的换行有所区别，rebase的时候会造成很多不必要的冲突。我们就把它忽略好了，在.git/config里增加：
	```
	    autocrlf = false
        safecrlf = false
	```	

2. 更久的一些提交，A库会有一些很久的提交，这里是提交不上来的。如果要提交的话，只能是生成patch，再`git am`进来。 如果不care的话，就直接先空的B 库merge，然后就如work flow里所讲的那样操作即可。

3. git format-patch A..B是 (A，B]的。

## CookBook##
- 首先需要clone 两个库在同一个project里。 这里由于以前的提交很多可以使用：

    Git svn clone -r [rxxx]:HEAD {svn:remote_repo}
             
- 然后再添加应外一个远端：

	Git config -add svn-remote.${brnch}.url https://xxxx
	
	Git config -add svn-remote/${brnch}.fetch :refs/remotes/${brnch}

	Git svn fetch –r ${brnch}

- 然后等人各种提交以后，我们过来再去分别更新两边的repo。

    Git svn fetch xxx.             {这个名字是.git/config里的名字}
	
- 然后拉好以后，就可以rebase到本地的分支。

	Git rebase remotes/xxx 

- 这里需要注意的是，需要checkout到对应的分支然后再rebase。（PS:其实可以使用—onto, 参数略多。。总是记不住）
然后需要checkout到深圳的分支，查看一下我们merge到哪里了，记下2个提交的range， 这里是(Start, End]的。
然后再checkout到我们的master，使用

	git cherry-pick (Start, End]

- 遇到冲突就:

	Git mergetool 召唤Beyond Compare
	
- 解决完了就

	Git cherry-pick –continue
