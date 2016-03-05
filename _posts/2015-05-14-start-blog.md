---
layout: post 
title: 使用GitHub Pages搭建博客 
---

## Info

&emsp;&emsp;一直想拥有一个自己的博客，之前不知道有GitHub Pages，认为只能自己买主机然后在上面搭建，无奈各路云主机的价格实在经受不起。最近看到了GitHub Pages，正合我意。

### 优缺点

优点：

* 空间免费，无限流量
* 若无特殊需求，可直接使用username.github.io访问
* 利用Git，方便管理，同时提供了练习Git的机会
* 静态网站，高安全性
* 由GitHub托管，使自己专注于博客本身

缺点：

* 没有数据库，不适用于大型网站
* 评论功能需要用到第三方插件，例如Disqus

### 安装过程及注意事项

安装过程：

1. 安装并配置Git
2. GitHub上添加名为username.github.io的远程库
3. 安装[Jekyll](http://jekyll.bootcss.com/docs/installation/)
4. 获得Disqus的[执行代码](https://disqus.com/websites/)

注意事项：

* 安装Jekyll的过程中若出现如下情景，则需要安装ruby-dev

		ERROR: Failed to build gem native extension.
* 可先在GitHub中生成一个模板页，方便目录结构的构建

### 开发过程及注意事项

开发过程：

1. 远程库pull到本地，按照Jekyll的[目录结构](http://jekyll.bootcss.com/docs/structure/)进行开发。(PS:可先在GitHub上生成模板页，方便目录结构的构建)
2. 在本地的博客目录下使用jekyll s可在浏览器中对博客进行预览，方便测试
3. 准备就绪后，git add commit push到远程库

注意事项：

* [官方文档](http://jekyllrb.com/docs/home/)，[中文版](http://jekyll.bootcss.com/docs/home/)
* index及post的文件都要有[YAML头信息](http://jekyll.bootcss.com/docs/frontmatter/)，具体格式如下：

		---
		layout: default
		title: test
		---

	PS: key与value之间必须有一个空格，否则会出现类似错误：

		jekyll 2.5.3 | Error:  undefined method `default_proc=' for "layout:default title:test":String

* 每次在_config.yml修改之后，都必须重启jekyll s才能在本地测试时看到结果
* 若要使用独立域名，首先在根目录下创建CNAME文件，内容为域名，再对域名进行[解析](http://jingyan.baidu.com/album/dca1fa6fa1e403f1a5405262.html)，PS: 需要添加@和www两种主机记录
* 对需要分离的部分，首先在根目录下创建_include文件夹，建立相应html文件，再在需要引入的地方用include标签来引入
* [分页](http://jekyll.bootcss.com/docs/pagination/)
