---
layout: post
title: Drupal 8 配置文件下载漏洞
---

## 0x00 漏洞概述

### 1.漏洞简介

[Drupal](https://www.drupal.org)是一个自由开源的內容管理系统，近期研究者发现在其8.x < 8.1.10的版本中发现了三个安全漏洞，其中一个漏洞攻击者可以在未授权的情况下下载管理员之前导出的配置文件压缩包`config.tar.gz`。Drupal官方在9月21日发布了[升级公告](https://www.drupal.org/SA-CORE-2016-004)。

### 2.漏洞影响

未授权状态下下载管理员之前导出的配置文件

### 3.影响版本

8.x < 8.1.10

## 0x01 漏洞复现

### 1. 环境搭建

Dockerfile（来自Docker Hub）

```dockerfile
# from https://www.drupal.org/requirements/php#drupalversions
FROM php:7.0-apache

RUN a2enmod rewrite

# install the PHP extensions we need
RUN apt-get update && apt-get install -y libpng12-dev libjpeg-dev libpq-dev \
	&& rm -rf /var/lib/apt/lists/* \
	&& docker-php-ext-configure gd --with-png-dir=/usr --with-jpeg-dir=/usr \
	&& docker-php-ext-install gd mbstring opcache pdo pdo_mysql pdo_pgsql zip

# set recommended PHP.ini settings
# see https://secure.php.net/manual/en/opcache.installation.php
RUN { \
		echo 'opcache.memory_consumption=128'; \
		echo 'opcache.interned_strings_buffer=8'; \
		echo 'opcache.max_accelerated_files=4000'; \
		echo 'opcache.revalidate_freq=60'; \
		echo 'opcache.fast_shutdown=1'; \
		echo 'opcache.enable_cli=1'; \
	} > /usr/local/etc/php/conf.d/opcache-recommended.ini

WORKDIR /var/www/html

# https://www.drupal.org/node/3060/release
ENV DRUPAL_VERSION 8.1.9
ENV DRUPAL_MD5 4de7c001ecbd5c27e5837c97e40facc2

RUN curl -fSL "https://ftp.drupal.org/files/projects/drupal-${DRUPAL_VERSION}.tar.gz" -o drupal.tar.gz \
	&& echo "${DRUPAL_MD5} *drupal.tar.gz" | md5sum -c - \
	&& tar -xz --strip-components=1 -f drupal.tar.gz \
	&& rm drupal.tar.gz \
	&& chown -R www-data:www-data sites modules themes
```

```bash
docker run --name dp -p 8080:80 -d drupal
```

### 2.漏洞分析

首先我们进入后台把配置文件导出，默认导出到了`/tmp/config.tar.gz`。

![Alt text](/images/2016-09-22-export.png)


然后看代码，我们在`core/modules/system/system.routing.yml`可以看到这样一个路由项：

![Alt text](/images/2016-09-22-admin.png)

这是访问管理员页面时的路由，可以看到`requirements._permission`指定了需要管理员权限。

然后我们再看这一项：

![Alt text](/images/2016-09-22-router.png)

可以看到并没有设置`_permission`，并且`_access=TRUE`，也就是说在未授权的情况下是可以访问这个功能的。

接下来我们跟进它的controller，在`core/modules/system/src/FileDownloadController.php`第41-68行的`download`函数：

```php?start_inline=1
public function download(Request $request, $scheme = 'private') {
    $target = $request->query->get('file');
    // Merge remaining path arguments into relative file path.
    $uri = $scheme . '://' . $target;

    if (file_stream_wrapper_valid_scheme($scheme) && file_exists($uri)) {
      // Let other modules provide headers and controls access to the file.
      $headers = $this->moduleHandler()->invokeAll('file_download', array($uri));

      foreach ($headers as $result) {
        if ($result == -1) {
          throw new AccessDeniedHttpException();
        }
      }

      if (count($headers)) {
        return new BinaryFileResponse($uri, 200, $headers, $scheme !== 'private');
      }

      throw new AccessDeniedHttpException();
    }

    throw new NotFoundHttpException();
  }
```

函数获取了我们传入的`file`参数，与`$scheme`拼接后检测文件是否存在。我们访问 http://xxx/system/temporary/?file=config.tar.gz 然后下断点动态调试，执行到该函数时各变量的值如下：

![Alt text](/images/2016-09-22-uri.png)

也就是说`$scheme`的值是`temporary`。然后程序进入了`file_stream_wrapper_valid_scheme`函数检查协议有效性，动态跟进直到`core/lib/Drupal/Core/StreamWrapper/LocalStream.php`中第495-506行的`url_stat`函数：

![Alt text](/images/2016-09-22-path.png)

可以看到`temporary：//config.tar.gz`被映射到了`/tmp/config.tar.gz`，也就是我们刚备份到的位置，所以也就通过了`file_exists`。

通过检查后回到`download`函数中，接下来执行了如下语句：

```php?start_inline=1
$headers = $this->moduleHandler()->invokeAll('file_download', array($uri));
```

跟进`invokeAll`函数，在`core/lib/Drupal/Core/Extension/ModuleHandler.php`中第397-409行：

```php?start_inline=1
public function invokeAll($hook, array $args = array()) {
    $return = array();
    $implementations = $this->getImplementations($hook);
    foreach ($implementations as $module) {
      $function = $module . '_' . $hook;
      $result = call_user_func_array($function, $args);
      if (isset($result) && is_array($result)) {
        $return = NestedArray::mergeDeep($return, $result);
      }
      elseif (isset($result)) {
        $return[] = $result;
      }
    }
  }
```

动态调试情况如下图：

![Alt text](/images/2016-09-22-funcs.png)

`implementations`的值有`config`, `file`, `image`，遍历这三个值并调用相应函数：

- `config_file_download`
- `file_file_download`
- `image_file_download`

首先调用的是`config_file_download`，位于`core/modules/config/config.module`第64-78行：

```php?start_inline=1
function config_file_download($uri) {
  $scheme = file_uri_scheme($uri);
  $target = file_uri_target($uri);
  if ($scheme == 'temporary' && $target == 'config.tar.gz') {
    $request = \Drupal::request();
    $date = DateTime::createFromFormat('U', $request->server->get('REQUEST_TIME'));
    $date_string = $date->format('Y-m-d-H-i');
    $hostname = str_replace('.', '-', $request->getHttpHost());
    $filename = 'config' . '-' . $hostname . '-' . $date_string . '.tar.gz';
    $disposition = 'attachment; filename="' . $filename . '"';
    return array(
      'Content-disposition' => $disposition,
    );
  }
}
```

可以看到当且仅当文件是`config.tar.gz`时设置响应头以供下载，最后当返回到`download`函数时，执行如下语句：

```php?start_inline=1
return new BinaryFileResponse($uri, 200, $headers, $scheme !== 'private');
```

将最终的响应返回给了用户，从而触发了下载漏洞。

![Alt text](/images/2016-09-22-download.png)

到这里有一个想法，我们可不可以传入`../../etc/passwd\x00config.tar.gz`这样的参数来截断并且跳到别的目录呢？

我们看一下在`core/lib/Drupal/Core/StreamWrapper/LocalStream.php`中第120到144行的`getLocalPath()`函数，它在上面提到的`url_stat`函数中被调用：

```php?start_inline=1
protected function getLocalPath($uri = NULL) {
    if (!isset($uri)) {
      $uri = $this->uri;
    }
    $path = $this->getDirectoryPath() . '/' . $this->getTarget($uri);
    
    if (strpos($path, 'vfs://') === 0) {
      return $path;
    }

    $realpath = realpath($path);
    if (!$realpath) {
      // This file does not yet exist.
      $realpath = realpath(dirname($path)) . '/' . drupal_basename($path);
    }
    $directory = realpath($this->getDirectoryPath());
    if (!$realpath || !$directory || strpos($realpath, $directory) !== 0) {
      return FALSE;
    }
    return $realpath;
  }
```

可以看到路径中的`../`被`realpath`过滤，跳出`/tmp`的目的也就不能达到了。

另外还有一个方面，`config_file_download`函数比较苛刻，只允许下载`config.tar.gz`，而`image_file_download`是为了下载图片，那么`file_file_download`函数能否供我们利用以下载系统的敏感文件呢？

`file_file_download`函数在`core/modules/file/file.module`中第582-633行：

```php?start_inline=1
function file_file_download($uri) {
  // Get the file record based on the URI. If not in the database just return.
  /** @var \Drupal\file\FileInterface[] $files */
  $files = entity_load_multiple_by_properties('file', array('uri' => $uri));
  if (count($files)) {
    foreach ($files as $item) {
      if ($item->getFileUri() === $uri) {
        $file = $item;
        break;
      }
    }
  }
  if (!isset($file)) {
    return;
  }
...
}
```

有以下三点：

- 根据注释可以看到该函数是根据`uri`来查询数据库中的文件记录再进行下载
-  我们请求中的`$scheme`为`temporary`，在`file_stream_wrapper_valid_scheme($scheme) && file_exists($uri)`限制了我们只能下载`/tmp`目录下存在的文件
-  默认`/tmp`下除了`config.tar.gz`只有`.htaccess`

综合这三点来看`file_file_download`函数是不存在下载漏洞的。



所以总的来说该漏洞只能在管理员导出备份的情况下下载`/tmp/config.tar.gz`。

### 3.补丁分析

![Alt text](/images/2016-09-22-patch.png)

增加了权限验证，使未授权用户不能下载`cnofig.tar.gz`。

## 0x02 修复方案

升级Drupal到8.1.10

## 0x03 参考

[https://www.drupal.org/SA-CORE-2016-004](https://www.drupal.org/SA-CORE-2016-004)
