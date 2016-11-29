---
title: 使用Hexo构建个人博客
date: 2016-11-26 10:52:24
tags: 
 - hexo
categories: Hexo
---

hexo是一个快速、简单的博客框架。可能通过Markdown语言编写文章。hexo来帮助你生成各种主题风格的静态网站，支持多终端访问

> 安装hexo之前需要先安装node和git。


### 安装hexo

执行下面的命令安装hexo

	$ npm install -g hexo

查看是否安装成功

	$ hexo -v

可能显示这些信息，说明hexo安装成功

	hexo: 3.2.2
	hexo-cli: 1.0.2
	os: Windows_NT 6.1.7601 win32 x64
	http_parser: 2.7.0
	node: 6.2.0
	v8: 5.0.71.47
	uv: 1.9.1
	zlib: 1.2.8
	ares: 1.10.1-DEV
	icu: 57.1
	modules: 48
	openssl: 1.0.2h

### Hexo的使用

#### 初始化站点

	$ hexo init blog

`blog`是你的站点目录。hexo会在`blog`下构建必要的目录结构，创建必要的配置文件等
初始化好的目录结构：

	.
	├── _config.yml
	├── package.json
	├── scaffolds/
	├── scripts/
	├── source/
	|   ├── _drafts
	|   └── _posts
	└── themes/

各目录结构说明

`_config.yml`：站点的全局配置文件。
`package.json`：应用数据。从它可以看出Hexo的版本信息，依赖的组件。
`scaffolds/`：模板文件目录。创建新文章的时候，通过模板生成
`scripts/`：js脚本文件目录。
`source/`：文章目录。整个站点的数据内容，图片等。
`_drafts`：在`source/`目录下的子目录，草稿放在这里。
`_posts`：在`source/`目录下的子目录，文章放在这里。
`themes/`：主题目录。hexo默认主题是landscape，难看。通常会下载一些别的主题放在这个目录下来使用，主题一般也有自己的配置文件。

#### 创建文章

	$ hexo new "测试文章"
	INFO  Created: D:\blog\source\_posts\测试文章.md

在`source/_posts`目录下生成一篇文章

#### 编辑文章
用文本编辑器打开"测试文章.md"文件，文章内容如下：

	---
	title: 测试文章
	date: 2016-11-26 11:48:41
	tags:
	---

在文章结尾，输入一些内容：

	### 这是一篇测试文章
	hexo的基本使用。。。。。。

#### 构建网站

	$ hexo generate

构建网站会在`blog`站点目录下，生成一个目录`public`，用来存放生成的所有站点文件

#### 启动服务，预览博客

	$ hexo s
	INFO  Start processing
	INFO  Hexo is running at http://localhost:4000/. Press Ctrl+C to stop.

如果电脑上安装了`Foxit Reader`褔昕阅读器，端口可能被占用，需要手动指定端口

	$ hexo s -p 5000
	INFO  Start processing
	INFO  Hexo is running at http://localhost:5000/. Press Ctrl+C to stop.

>  Ctrl+C停止服务

浏览器中访问 http://localhost:5000/ 即可看到生成的主页。

### 清理
	
	$ hexo clean
	INFO  Deleted database.
	INFO  Deleted public folder.

主要是把`public`目录删除掉了。

> 一般在使用`hexo generate`命令构建博客的时候，会先清理下。


### Hexo 主题

hexo有一套默认的主题，不过太难看了。一般会安装一套别的主题使用

#### 安装next主题
我用一套主题是`Next`,命令安装：

	$ git clone https://github.com/iissnan/hexo-theme-next themes/next

安装成功后，会在`themes`目录里多出一个目录`next`就是该主题的文件

#### 启用主题
打开`blog/_config.yml`站点全局配置文件，设置

	theme: next

清理 构建博客 启动服务 浏览器中预览效果。

	hexo clean
	hexo g
	hexo s -p 5000

### 部署

如果自己的域名或空间，可以把博客直接部署上去

我是部署到github上，在站点全局配置文件中配置：

	deploy:
	  type: git
	  repository: https://github.com/leileiyuan/leileiyuan.github.io.git
	  branch: master

执行部署命令
	
	hexo deploy

可以将生成的部署命令组合使用 `hexo d -g` 或 `hexo g -d`，是指生成后立刻部署。


使用到的命令列表：

	$ hexo init -- 初始化网站

	$ hexo new "新文章名" -- 创建新文章

	$ hexo new page tags -- 创建标签，创始分类，创建其它结构的目录，都可以用

	$ hexo clean -- 清理
	$ hexo g  -- generate 构建 
	$ hexo s  -- server 启动本地服务 
	$ hexo d  -- deploy 部署到远程服务器 

	$ hexo s -p 5000 -- 指定启动本地服务的端口

	$ hexo g -d -- 生成后部署

### 参考
> * Hexo中文官网： https://hexo.io/zh-cn/
> * Next主题官网： http://theme-next.iissnan.com/
