---
layout: post 
title: Ubuntu下nodejs的安装与配置
---

## Info

刚上手nodejs，在这里总结一下安装与配置的过程以及过程中出现的问题和解决方案。

### 安装

1. 在[官网](https://nodejs.org/download/)下载对应安装包
2. 解压
3. 设置环境变量（修改.zshrc或者/etc/profile）：

		export NODE_HOME=/opt/nodejs
		export PATH=$PATH:$NODE_HOME/bin
		export NODE_PATH=$NODE_HOME/lib/node_modules
4. 检查版本：
			
		node -v

### 配置

1. 更新版本：

		npm install -g n
		n stable
		n
2. 全局安装模块：

		npm install -g xxx
模块会安装在NODE_PATH的路径里面
3. 使用express和swig：
	1. 安装express和generator：

			npm install -g express express-generator
	2. 在工程目录下用生成express项目：

			express project

	3. 把package.json里面的jade的键值换成swig以及其相应版本
	4. 安装依赖模块：

			npm install
	5. 在项目里的app.js中进行如下改动：

			var swig = require('swig');
			
			app.engine('html', swig.renderFile);
			
			app.set('view engine', 'html');
			app.set('views', __dirname + '/views');
	6. 运行：

			DEBUG=xxx:* npm start

### 问题

* 全局安装模块后运行app时显示` Cannot find module 'xxx'`
原因：没有设置NODE_PATH
解决方案：见安装步骤3
* n下载了新版node但是node -v仍是旧版本
原因：用apt-get安装的node引起的问题
解决方案：purge掉apt-get的node
