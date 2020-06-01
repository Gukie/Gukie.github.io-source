---
title: getResourceAsStream 在class和classloader的异同
categories: 
    - java
    - 基础
tags: 
    - java
---


<a name="6kO5S"></a>
### Java中加载资源，有两种方式

- 通过classloader的getResourceAsStream
- 通过class的getResourceAsStream


<br />
<br />他们的相同点，都是去resources下找资源，不同点是class的getResourceAsStream在找资源之前会对参数"name"进行处理。以下为源码的简单分析<br />
<br />

<a name="C0wO2"></a>
#### java.lang.ClassLoader#getResourceAsStream

- name传什么，就用该name去找，不会做而外的处理
```java
public InputStream getResourceAsStream(String name) {
        URL url = getResource(name);
        try {
            return url != null ? url.openStream() : null;
        } catch (IOException e) {
            return null;
        }
    }
```


<a name="tyQpQ"></a>
#### java.lang.Class#getResourceAsStream
会先对传入的name做一层处理:

- 如果name是以"/"为开头，那么不做处理，即: 从resources下找
- 如果name不是以"/"为开头，则会将 调用该方法的类的package加上去. 即会从resources下找"packageName"+"name"的资源. 比如"com.test.test-1.txt"

![image.png](1.png)

- 之后通过classloader去找资源
```java
public InputStream getResourceAsStream(String name) {
        name = resolveName(name);
        ClassLoader cl = getClassLoader0();
        if (cl==null) {
            // A system class.
            return ClassLoader.getSystemResourceAsStream(name);
        }
        return cl.getResourceAsStream(name);
    }


/**
     * Add a package name prefix if the name is not absolute Remove leading "/"
     * if name is absolute
     */
    private String resolveName(String name) {
        if (name == null) {
            return name;
        }
        if (!name.startsWith("/")) {
            Class<?> c = this;
            while (c.isArray()) {
                c = c.getComponentType();
            }
            String baseName = c.getName();
            int index = baseName.lastIndexOf('.');
            if (index != -1) {
                name = baseName.substring(0, index).replace('.', '/')
                    +"/"+name;
            }
        } else {
            name = name.substring(1);
        }
        return name;
    }
```

<br />
<br />

<a name="RvqPr"></a>
### 一个例子


在找不到资源的时候，可以查看下maven打出来的target中是否包含资源, 如果找得到，那就基本都能找到，否则就要看看pom.xml中是否在builder时候将对应的资源给exclude掉了(默认都是会将resources下的资源加载进来的)<br />![image.png](2.png)

<a name="Oo89r"></a>
#### 在main下的示例
源码如下
```java
public class Test {

    public static void main(String[] args) {

        InputStream fis1 = Thread.currentThread().getContextClassLoader().getResourceAsStream("hello.txt");
        InputStream fis2 = Thread.currentThread().getContextClassLoader().getResourceAsStream("cloud_cert-daily.pfx");
        InputStream fis3 = Thread.currentThread().getContextClassLoader().getResourceAsStream("cloud_root-daily.crt");
        InputStream fis4 = Thread.currentThread().getContextClassLoader().getResourceAsStream("test.txt");
        System.out.println(false);
    }
}

```
![image.png](3.png)
<a name="lLGB4"></a>
#### 在test下进行的示例
源码如下:
```java
public class One {

    public static void main(String[] args) {

        InputStream fis1 = One.class.getClassLoader().getResourceAsStream("hello2.txt");
        InputStream fis2 = One.class.getResourceAsStream("test-1.txt");
        InputStream fis3 = Thread.currentThread().getContextClassLoader().getResourceAsStream("cert/pro/root-daily2.crt");
        InputStream fis4 = Thread.currentThread().getContextClassLoader().getResourceAsStream("test2.txt");
        InputStream fis5 = One.class.getResourceAsStream("/test3.txt");
        InputStream fis6 = One.class.getResourceAsStream("/cert/pro/cert-daily1.pfx");
        System.out.println(false);

    }
}
```
![image.png](4.png)<br />
<br />

