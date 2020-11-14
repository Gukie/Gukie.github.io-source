
---
title: Java运行时修改注解值
categories: 
    - [java]

tags: 
    - java

excerpt: 运行时动态修改注解的值，虽不常见但是在某些时候却是非常需要的好东西
---


<a name="pNBxM"></a>
### refer
- [https://blog.csdn.net/Crabime/article/details/54880411](https://blog.csdn.net/Crabime/article/details/54880411)
- [https://blog.csdn.net/wanson2015/article/details/83993618](https://blog.csdn.net/wanson2015/article/details/83993618)


<br />
<br />在业务开发中，我们可能希望动态地修改注解中的值，以达到不同的处理效果.<br />
<br />注解:
```java
@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface BitPoint {
    int length();
    int order();
}
```
```java
@Getter
@Setter
public class TestDataPoint {
    
    @BitPoint(order = 1, length = 1)
    private Integer type;

    @BitPoint(order = 2, length = 1)
    private Integer userId;

    @BitPoint(order = 3, length = 1)
    private Integer value;

}
```

<br />但是在业务发展过程中，中间那个userId字段一开始是1字节，后来1字节放不下了，需要2字节...那咋吧呢？其他业务逻辑都是一样的，就这个字段的长度变了。总不能再全写一套吧，虽然能实现，但是总感觉不优雅...<br />
<br />其实最好是在运行时进行annotation值的变更，网上找了资料之后，发现可以的。果然你遇到的问题别人都遇到过
```java
@Test
    public void testAnnotationChange() throws IllegalAccessException, NoSuchFieldException {

        final String byteStr = "01 00 01 01 02 03";
        System.out.println(byteStr);
        TestDataPoint dataPointRequest = DataPointUtils.fromRawValue(getBytes(byteStr), TestDataPoint.class);
        System.out.println(JSON.toJSONString(dataPointRequest));


        Field field = FieldUtils.getDeclaredField(TestDataPoint.class,"userId",true);
        BitPoint annotation = field.getAnnotation(BitPoint.class);
        if (annotation != null){
            InvocationHandler invocationHandler = Proxy.getInvocationHandler(annotation);
            // memberValues是固定的
            Field values = invocationHandler.getClass().getDeclaredField("memberValues");
            values.setAccessible(true);
            Map<String, Object> memberValues =(Map<String, Object>) values.get(invocationHandler);
            int val = (int) memberValues.get("length");
            System.out.println("before:"  + val);
            memberValues.put("length", 2);
            System.out.println("after:"  + annotation.length());
        }


        System.out.println("=========");
        TestDataPoint extDataPointRequest = DataPointUtils.fromRawValue(getBytes(byteStr), TestDataPoint.class);
        System.out.println(JSON.toJSONString(extDataPointRequest));

    }


    private byte[] getBytes(String byteStr) {
        String [] items = StringUtils.split(byteStr," ");
        List<Integer> integers = Arrays.stream(items).map(x -> Integer.valueOf(x, 16)).collect(Collectors.toList());
        byte[] actualBytes = new byte[items.length];
        for (int i = 0; i < integers.size(); i++) {
            actualBytes[i] = integers.get(i).byteValue();
        }
        return actualBytes;
    }
```


```
01 00 01 01 02 03

{"type":1,"userId":0,"value":1}
before:1
after:2
=========
{"type":1,"userId":1,"value":1}
```

<br />上面代码唯一需要注意的是 获取 注解的feild的时候，memberValues是固定的, 原因如下
```java
Field values = invocationHandler.getClass().getDeclaredField("memberValues");
```
![image.png](1.png)
