---
title: 'hexo d 发布到github时，报错：fatal: The remote end hung up unexpectedly'
date: 2016-12-29 18:57:25
tags: 
- git
- hexo
categories: git
---
发布博客内容到github上时，报以下错误：fatal: The remote end hung up unexpectedly，现在也没明白是怎么回事.......
<!-- more -->
```
$ hexo d
INFO  Deploying: git
INFO  Clearing .deploy_git folder...
INFO  Copying files from public folder...
[master 045499c] Site updated: 2016-12-29 18:49:47
 1 file changed, 1 insertion(+), 1 deletion(-)
fatal: The remote end hung up unexpectedly
fatal: The remote end hung up unexpectedly
error: RPC failed; curl 52 Empty reply from server
Everything up-to-date
FATAL Something's wrong. Maybe you can find the solution here: http://hexo.io/docs/troubleshooting.html
Error: fatal: The remote end hung up unexpectedly
fatal: The remote end hung up unexpectedly
error: RPC failed; curl 52 Empty reply from server
Everything up-to-date

    at ChildProcess.<anonymous> (D:\blog\node_modules\hexo-util\lib\spawn.js:37:17)
    at emitTwo (events.js:106:13)
    at ChildProcess.emit (events.js:191:7)
    at ChildProcess.cp.emit (D:\blog\node_modules\cross-spawn\lib\enoent.js:40:29)
    at maybeClose (internal/child_process.js:850:16)
    at Process.ChildProcess._handle.onexit (internal/child_process.js:215:5)
FATAL fatal: The remote end hung up unexpectedly
fatal: The remote end hung up unexpectedly
error: RPC failed; curl 52 Empty reply from server
Everything up-to-date

Error: fatal: The remote end hung up unexpectedly
fatal: The remote end hung up unexpectedly
error: RPC failed; curl 52 Empty reply from server
Everything up-to-date

    at ChildProcess.<anonymous> (D:\blog\node_modules\hexo-util\lib\spawn.js:37:17)
    at emitTwo (events.js:106:13)
    at ChildProcess.emit (events.js:191:7)
    at ChildProcess.cp.emit (D:\blog\node_modules\cross-spawn\lib\enoent.js:40:29)
    at maybeClose (internal/child_process.js:850:16)
    at Process.ChildProcess._handle.onexit (internal/child_process.js:215:5)

```

按网上的童鞋的方法，配置了个参数

	git config http.postBuffer 524288000

或者打开`工程目录/.git/config`文件，在最后加一项

	[http]
	postBuffer = 524288000


`config`文件内容如下：

	[core]
		repositoryformatversion = 0
		filemode = false
		bare = false
		logallrefupdates = true
		symlinks = false
		ignorecase = true
	[remote "origin"]
		url = https://github.com/leileiyuan/hexoSource.git
		fetch = +refs/heads/*:refs/remotes/origin/*
	[branch "master"]
		remote = origin
		merge = refs/heads/master
	[gui]
		wmstate = normal
		geometry = 835x475+100+100 185 214
	[http]
		postBuffer = 524288000

再重新发布，就可以了。

**原因未知，一脸黑线.......**
**原因未知，一脸黑线.......**
**原因未知，一脸黑线.......**