---
title: annotation定义的总结
date: 2020-05-16 11:12:25
categories: 
    - java 
    - 基础
---
# [Annotation] annotation定义的总结

<a name="tyC3q"></a>
### refer
- [https://juejin.im/post/5c63e2b0f265da2dd94c8e16](https://juejin.im/post/5c63e2b0f265da2dd94c8e16) （@Inherited 使用注意点）


<br />
<br />


| **元注解** | **作用** | **使用注意点** |
| --- | --- | --- |
| @Inherited | 是否可以继承. 即子类是否可以不用定义直接使用父类的注解 | 使用@Inherited注意点<br />- @Inherited 并不会从接口中继承，而只会从class中继承<br />- 方法并不从它所重载的方法继承annotation<br /> |
| @Retention | 表示在系统运行的哪个阶段出现 | RetentionPolicy.RUNTIME - 会被compiler记住并且会在JVM 运行时存留<br />RetentionPolicy._CLASS - 会被compiler记住但是不会在JVM中存留_ |
| @Documented | 是否在javadoc中出现 |  |
| @Target | 定义的注解可以使用在那些地方 | 具体可以查看 ElementType的定义<br />ElementType._METHOD - 可以使用在方法上_<br />ElementType._TYPE - 可以使用在class/interface上_ |


<br />一个完整的例子
```java
@Documented
@Inherited
@Retention(RetentionPolicy.RUNTIME)
@Target( {ElementType.METHOD, ElementType.TYPE})
public @interface TimeTraceLog {

    long value() default 200;
    
    Level level() default Level.WARN;
}

```

<br />
<br />

