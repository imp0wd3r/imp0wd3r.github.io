---
layout: post
title: Wordpress <= 4.6.1 使用语言文件任意代码执行 漏洞分析
---

## 0x00 漏洞概述

### 1.漏洞简介

[WordPress](https://wordpress.org/)是一个以PHP和MySQL为平台的自由开源的博客软件和内容管理系统，近日在github （[https://gist.github.com/anonymous/908a087b95035d9fc9ca46cef4984e97](https://gist.github.com/anonymous/908a087b95035d9fc9ca46cef4984e97)）上爆出这样一个漏洞，在其<=4.6.1版本中，如果网站使用攻击者提前构造好的语言文件来对网站、主题、插件等等来进行翻译的话，就可以执行任意代码。

### 2.漏洞影响

任意代码执行，但有以下两个前提：

1. 攻击者可以上传自己构造的语言文件，或者含有该语言文件的主题、插件等文件夹
2. 网站使用攻击者构造好的语言文件来对网站、主题、插件等进行翻译

这里举一个真实场景中的例子：攻击者更改了某个插件中的语言文件，并更改了插件代码使插件初始化时使用恶意语言文件对插件进行翻译，然后攻击者通过诱导管理员安装此插件来触发漏洞。

### 3.影响版本

<= 4.6.1

## 0x01 漏洞复现

### 1. 环境搭建

```bash
docker pull wordpress:4.6.1
docker pull mysql
docker run --name wp-mysql -e MYSQL_ROOT_PASSWORD=hellowp -e MYSQL_DATABASE=wp -d mysql
docker run --name wp --link wp-mysql:mysql -d wordpress
```

### 2.漏洞分析

首先我们来看这样一个场景：

![Alt text](/images/2016-10-09-create-function-bug.png)

在调用`create_function`时，我们通过`}`将原函数闭合，添加我们想要执行的内容后再使用`/*`将后面不必要的部分注释掉，最后即使我们没有调用创建好的函数，我们添加的新内容也依然被执行了。之所以如此，是因为`create_function`内部使用了`eval`来执行代码，我们看PHP手册上的说明：

![Alt text](/images/2016-10-09-eval.png)

所以由于这个特性，如果我们可以控制`create_function`的`$code`参数，那就有了任意代码执行的可能。这里要说一下，`create_function`这个漏洞最早由80sec在08年提出，这里提供几个链接作为参考：

- [https://www.exploit-db.com/exploits/32416/](https://www.exploit-db.com/exploits/32416/)
- [https://bugs.php.net/bug.php?id=48231](https://bugs.php.net/bug.php?id=48231)
- [http://www.2cto.com/Article/201212/177146.html](http://www.2cto.com/Article/201212/177146.html)

接下来我们看Wordpress中一处用到`create_function`的地方，在`wp-includes/pomo/translations.php`第203-209行：

```php?start_inline=1
/**
 * Makes a function, which will return the right translation index, according to the
 * plural forms header
 * @param int    $nplurals
 * @param string $expression
 */
function make_plural_form_function($nplurals, $expression) {
	$expression = str_replace('n', '$n', $expression);
	$func_body = "
		\$index = (int)($expression);
		return (\$index < $nplurals)? \$index : $nplurals - 1;";
	return create_function('$n', $func_body);
}
```

根据注释可以看到该函数的作用是根据字体文件中的`plural forms`这个header来创建函数并返回，其中`$expression`用于组成`$func_body`，而`$func_body`作为`$code`参数传入了`create_function`，所以关键是控制`$expresstion`的值。

我们看一下正常的字体文件`zh_CN.mo`，其中有这么一段：

![Alt text](/images/2016-10-09-common.png)

`Plural-Froms`这个header就是上面的函数所需要处理的，其中`nplurals`的值即为`$nplurals`的值，而`plural`的值正是我们需要的`$expression`的值。所以我们将字体文件进行如下改动：

![Alt text](/images/2016-10-09-payload.png)

然后我们在后台重新加载这个字体文件，同时进行动态调试，可以看到如下情景：

![Alt text](/images/2016-10-09-debug.png)

我们payload中的`)`首先闭合了前面的`(`，然后`；`结束前面的语句，接着是我们的一句话木马，然后用`/*`将后面不必要的部分注释掉，通过这样，我们就将payload完整的传入了`create_function`，在其创建函数时我们的payload就会被执行，由于访问每个文件时都要用这个对字体文件解析的结果对文件进行翻译，所以我们访问任何文件都可以触发这个payload：

![Alt text](/images/2016-10-09-index.png)

![Alt text](/images/2016-10-09-tools.png)

其中访问`index.php?c=phpinfo();`的函数调用栈如下：

![Alt text](/images/2016-10-09-call.png)


### 3.补丁分析

目前官方还没有发布补丁，最新版仍存在该漏洞。

## 0x02 修复方案

在官方发布补丁前建议管理员增强安全意识，不要使用来路不明的字体文件、插件、主题等等。

对于开发者来说，建议对`$expression`中的特殊符号进行过滤，例如：

```php?start_inline=1
$not_allowed = array(";", ")", "}");
$experssion = str_replace($not_allowed, "", $expression);
```

## 0x03 参考

[https://gist.github.com/anonymous/908a087b95035d9fc9ca46cef4984e97](https://gist.github.com/anonymous/908a087b95035d9fc9ca46cef4984e97)

[http://php.net/manual/zh/function.create-function.php](http://php.net/manual/zh/function.create-function.php)

[https://www.exploit-db.com/exploits/32416/](https://www.exploit-db.com/exploits/32416/)

[https://bugs.php.net/bug.php?id=48231](https://bugs.php.net/bug.php?id=48231)

[http://www.2cto.com/Article/201212/177146.html](http://www.2cto.com/Article/201212/177146.html)

[https://codex.wordpress.org/Installing_WordPress_in_Your_Language](https://codex.wordpress.org/Installing_WordPress_in_Your_Language)
