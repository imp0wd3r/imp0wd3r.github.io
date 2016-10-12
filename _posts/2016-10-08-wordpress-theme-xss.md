---
layout: post
title: Wordpress <= 4.6.1 使用主题文件触发存储型XSS 漏洞分析
---

## 0x00 漏洞概述

### 1.漏洞简介

[WordPress](https://wordpress.org/)是一个以PHP和MySQL为平台的自由开源的博客软件和内容管理系统，近日研究者发现在其<=4.6.1版本中，通过上传恶意构造的主题文件可以触发一个后台存储型XSS漏洞。通过该漏洞，攻击者可以在能够上传主题文件的前提下执行获取管理员Cookie等敏感操作。

### 2.漏洞影响

在能够上传主题文件的前提下执行获取管理员Cookie等XSS可以进行的攻击，实际的攻击场景有以下两种：

- 攻击者诱导管理员上传恶意构造的主题文件，且管理员并没有对文件进行检查
- 攻击者拥有管理员权限可以直接上传主题文件，但既然已经有管理员权限再进行这样的攻击也就多此一举了

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

我们先随便下载一个主题：

```bash
wget https://downloads.wordpress.org/theme/illdy.1.0.29.zip
unzip -x illdy.1.0.29.zip
```

然后对`illdy/style.css`进行如下更改：

```css
/*
Theme Name: <svg onload=alert(1234)>
... DO NOT CHANGES HERE ...
*/
```

接着更改文件夹名字再打包：

```bash
mv illdy "<svg onload=alert(5678)>"
zip -r theme.zip "<svg onload=alert(5678)>"
```

构造好之后我们登录后台上传该主题文件，同时开始动态调试。

首先进入`wp-admin/includes/class-theme-installer-skin.php`中第55-82行：

```php?start_inline=1
$name = $theme_info->display('Name');
...

if ( current_user_can( 'edit_theme_options' ) && current_user_can( 'customize' ) ) {
	$install_actions['preview'] = '<a href="' . wp_customize_url( $stylesheet ) . '" class="hide-if-no-customize load-customize"><span aria-hidden="true">' . __( 'Live Preview' ) . '</span><span class="screen-reader-text">' . sprintf( __( 'Live Preview &#8220;%s&#8221;' ), $name ) . '</span></a>';
}
$install_actions['activate'] = '<a href="' . esc_url( $activate_link ) . '" class="activatelink"><span aria-hidden="true">' . __( 'Activate' ) . '</span><span class="screen-reader-text">' . sprintf( __( 'Activate &#8220;%s&#8221;' ), $name ) . '</span></a>';

```

其中`$theme_info`的值如下：

![Alt text](/images/2016-10-08-theme-info.png)

其中`stylesheet`和`template`的值为我们更改的文件夹名，`headers.Name`为更改的`style.css`中的`Name`。`$theme_info`中有我们可控的payload，其调用`display`函数后赋值给`$name`，`$name`直接与html拼接，所以关键点在`display`函数上，动态调试跟进到`wp-includes/class-wp-theme.php`中第630-646行：

```php?start_inline=1
public function display( $header, $markup = true, $translate = true ) {
	$value = $this->get( $header );
	if ( false === $value ) {
		return false;
	}

	if ( $translate && ( empty( $value ) || ! $this->load_textdomain() ) )
		$translate = false;

	if ( $translate )
		$value = $this->translate_header( $header, $value );

	if ( $markup )
		$value = $this->markup_header( $header, $value, $translate );

	return $value;
}
```

由之前的调用可知，这里的`$header`的值为`Name`。首先看`$this-get($header)`，在`wp-includes/class-wp-theme.php`中第594-617行：

```php?start_inline=1
public function get( $header ) {
		...
			$this->headers_sanitized[ $header ] = $this->sanitize_header( $header, $this->headers[ $header ] );
		...
		return $this->headers_sanitized[ $header ];
	}
```

这里省略了与漏洞无关的部分，程序进入了`$this->sanitize_header`，在`wp-includes/class-wp-theme.php`第661-705行：

```php?start_inline=1
private function sanitize_header( $header, $value ) {
	switch ( $header ) {
		...
		case 'Name' :
			static $header_tags = array(
				'abbr'    => array( 'title' => true ),
				'acronym' => array( 'title' => true ),
				'code'    => true,
				'em'      => true,
				'strong'  => true,
			);
			$value = wp_kses( $value, $header_tags );
			break;
		...
}
```

这里执行了`Name`这个分支，可以看到程序使用`wp_kses`对`$value`的值进行了过滤，仅允许`$header_tags`中的html符号，所以我们`headers.Name`的值`<svg onload=alert(1234)>`是不合法的，`$value`值被赋为空。

然后程序回到了`display`函数，根据动态调试可以知道程序执行了`$value = $this->markup_header( $header, $value, $translate );`这个条件分支，再跟进，在`wp-includes/class-wp-theme.php`中第720-748行：

```php?start_inline=1
private function markup_header( $header, $value, $translate ) {
	switch ( $header ) {
		case 'Name' :
			if ( empty( $value ) )
				$value = $this->get_stylesheet();
			break;
		...
	return $value;
}
```

这里我们看到由于`$value`在之前被赋为空，导致此处`$value`被重新赋值为了`$this->get_stylesheet()`，也就是值为`<svg onload=alert(5678)>`的`stylesheet`变量。最后返回的`$value`赋给了`$name`，`$name`再与html拼接返回给客户端，从而触发了漏洞：

![Alt text](/images/2016-10-08-xss-js.png)

![Alt text](/images/2016-10-08-xss-html.png)

这个漏洞有趣的地方在于`style.css`中的payload其实起到的是一个障眼法的作用，正是因为`<svg onload=alert(1234)>`被过滤了才使`$value`被赋值成了我们真正的payload`<svg onload=alert(5678)>`。所以在构造主题文件的时候`style.css`和文件夹名这两个地方都要更改。

### 3.补丁分析

可能是由于利用条件十分苛刻，目前Wordpress官方还没有发布补丁，最新版Wordpress仍存在该漏洞。

## 0x02 修复方案

在官方发布补丁前，管理员应提高安全意识，不要轻易使用来路不明的主题。

对于开发者来说建议对`$name`进行合法性检查，例如这样：

```php?start_inline=1
$allowed_html = array(
	'em'      => true,
	'strong'  => true,
);
$name = wp_kses($name, $allowed_html);
```

## 0x03 参考

[https://www.mehmetince.net/low-severity-wordpress-461-stored-xss-via-theme-file](https://www.mehmetince.net/low-severity-wordpress-461-stored-xss-via-theme-file)

[https://codex.wordpress.org/Function_Reference/wp_kses](https://codex.wordpress.org/Function_Reference/wp_kses)
