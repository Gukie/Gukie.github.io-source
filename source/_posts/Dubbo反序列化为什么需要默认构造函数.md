
---
title: Dubbo反序列化为什么需要默认构造函数
categories: 
    - 中间件
    - Dubbo
tags: 
    - Dubbo

excerpt: 无参构造函数在很多地方都会有使用到，比如Spring的bean的初始化，以及dubbo的序列化时。所以对外提供的时候尽量将它提供出去，以免创造一些诡异的bug

---


<br />最近发现一个问题，在提供的dubbo服务中，如果对参数校验失败抛出异常对时候，在consumer端进行反序列化时会失败. 错误如下:<br />

```
com.alibaba.com.caucho.hessian.io.HessianProtocolException: 'com.xxx.client.exception.BizException' could not be instantiated
	at com.alibaba.com.caucho.hessian.io.JavaDeserializer.instantiate(JavaDeserializer.java:316)
	at com.alibaba.com.caucho.hessian.io.JavaDeserializer.readObject(JavaDeserializer.java:201)
	at com.alibaba.com.caucho.hessian.io.SerializerFactory.readObject(SerializerFactory.java:536)
	at com.alibaba.com.caucho.hessian.io.Hessian2Input.readObjectInstance(Hessian2Input.java:2820)
	at com.alibaba.com.caucho.hessian.io.Hessian2Input.readObject(Hessian2Input.java:2743)
	at com.alibaba.com.caucho.hessian.io.Hessian2Input.readObject(Hessian2Input.java:2278)
	at com.alibaba.com.caucho.hessian.io.Hessian2Input.readObject(Hessian2Input.java:2717)
	at com.alibaba.com.caucho.hessian.io.Hessian2Input.readObject(Hessian2Input.java:2278)
	at org.apache.dubbo.common.serialize.hessian2.Hessian2ObjectInput.readObject(Hessian2ObjectInput.java:85)
	at org.apache.dubbo.rpc.protocol.dubbo.DecodeableRpcResult.handleException(DecodeableRpcResult.java:144)
	at org.apache.dubbo.rpc.protocol.dubbo.DecodeableRpcResult.decode(DecodeableRpcResult.java:96)
	at org.apache.dubbo.rpc.protocol.dubbo.DecodeableRpcResult.decode(DecodeableRpcResult.java:112)
	at org.apache.dubbo.rpc.protocol.dubbo.DubboCodec.decodeBody(DubboCodec.java:92)
	at org.apache.dubbo.remoting.exchange.codec.ExchangeCodec.decode(ExchangeCodec.java:122)
	at org.apache.dubbo.remoting.exchange.codec.ExchangeCodec.decode(ExchangeCodec.java:82)
	at org.apache.dubbo.rpc.protocol.dubbo.DubboCountCodec.decode(DubboCountCodec.java:48)
	at org.apache.dubbo.remoting.transport.netty4.NettyCodecAdapter$InternalDecoder.decode(NettyCodecAdapter.java:90)
	at io.netty.handler.codec.ByteToMessageDecoder.decodeRemovalReentryProtection(ByteToMessageDecoder.java:502)
	at io.netty.handler.codec.ByteToMessageDecoder.callDecode(ByteToMessageDecoder.java:441)
	at io.netty.handler.codec.ByteToMessageDecoder.channelRead(ByteToMessageDecoder.java:278)
	at io.netty.channel.AbstractChannelHandlerContext.invokeChannelRead(AbstractChannelHandlerContext.java:362)
	at io.netty.channel.AbstractChannelHandlerContext.invokeChannelRead(AbstractChannelHandlerContext.java:348)
	at io.netty.channel.AbstractChannelHandlerContext.fireChannelRead(AbstractChannelHandlerContext.java:340)
	at io.netty.channel.DefaultChannelPipeline$HeadContext.channelRead(DefaultChannelPipeline.java:1434)
	at io.netty.channel.AbstractChannelHandlerContext.invokeChannelRead(AbstractChannelHandlerContext.java:362)
	at io.netty.channel.AbstractChannelHandlerContext.invokeChannelRead(AbstractChannelHandlerContext.java:348)
	at io.netty.channel.DefaultChannelPipeline.fireChannelRead(DefaultChannelPipeline.java:965)
	at io.netty.channel.nio.AbstractNioByteChannel$NioByteUnsafe.read(AbstractNioByteChannel.java:163)
	at io.netty.channel.nio.NioEventLoop.processSelectedKey(NioEventLoop.java:644)
	at io.netty.channel.nio.NioEventLoop.processSelectedKeysOptimized(NioEventLoop.java:579)
	at io.netty.channel.nio.NioEventLoop.processSelectedKeys(NioEventLoop.java:496)
	at io.netty.channel.nio.NioEventLoop.run(NioEventLoop.java:458)
	at io.netty.util.concurrent.SingleThreadEventExecutor$5.run(SingleThreadEventExecutor.java:897)
	at io.netty.util.concurrent.FastThreadLocalRunnable.run(FastThreadLocalRunnable.java:30)
	at java.lang.Thread.run(Thread.java:748)
Caused by: java.lang.reflect.InvocationTargetException
	at sun.reflect.NativeConstructorAccessorImpl.newInstance0(Native Method)
	at sun.reflect.NativeConstructorAccessorImpl.newInstance(NativeConstructorAccessorImpl.java:62)
	at sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45)
	at java.lang.reflect.Constructor.newInstance(Constructor.java:423)
	at com.alibaba.com.caucho.hessian.io.JavaDeserializer.instantiate(JavaDeserializer.java:312)
	... 34 more
Caused by: java.lang.NullPointerException
	at com.xxx.client.exception.BizException.<init>(BizException.java:24)
	... 39 more

	at org.apache.dubbo.rpc.cluster.support.FailoverClusterInvoker.doInvoke(FailoverClusterInvoker.java:117)
	at org.apache.dubbo.rpc.cluster.support.AbstractClusterInvoker.invoke(AbstractClusterInvoker.java:248)
	at org.apache.dubbo.rpc.cluster.support.wrapper.MockClusterInvoker.invoke(MockClusterInvoker.java:78)
	at org.apache.dubbo.rpc.proxy.InvokerInvocationHandler.invoke(InvokerInvocationHandler.java:55)
	at org.apache.dubbo.common.bytecode.proxy31.add(proxy31.java)
	... 12 more
```

<br />
<br />BizException的代码如下:
```java
public class BizException extends BaseException {
    private Object[] params;
 
    public BizException(String message, Throwable cause) {
        super(message, cause);
    }
 
    public BizException(LangIEnum enums) {
        super(enums.getKey(), enums.getCnDesc());
    }
 
    public BizException(BizExceptionCode bizExceptionCode) {
        super(bizExceptionCode.getCode(),bizExceptionCode.getMessage());
    }
 
    public BizException(String code, String message) {
        super(code,message);
    }
 
    public Object[] getParams() {
        return params;
    }
 
    public void setParams(Object[] params) {
        this.params = params;
    }
}
```

<br />
<br />
<br />业务代码如下:
```java
public Long add(UserDevicePropertyDTO userDeviceProperty) {
    if (userDeviceProperty == null) {
        throw new BizException(BizExceptionCode.PARAM_INVALID);
    }
    ....
}
```

<br />

<a name="JbYu7"></a>
### 代码简析
从业务代码角度看，没有任何毛病。但从报错信息看是在构造BizException的时候，入参为null, 从而导致NPE了. 可是传的明明是 BizExceptionCode.PARAM_INVALID，见鬼了吗? 仔细分析之后会发现，BizExceptionCode的构造方法的入参确实是null, 具体过程如下<br />
<br />
<br />1.代码位置: com.alibaba.com.caucho.hessian.io.JavaDeserializer#JavaDeserializer<br />

```java
public JavaDeserializer(Class cl) {
        ...
        Constructor[] constructors = cl.getDeclaredConstructors();
        long bestCost = Long.MAX_VALUE;
		// 根据构造函数的参数计算cost，并取最小的那个. 如果有无参构造函数，则其cost最小.
        for (int i = 0; i < constructors.length; i++) {
            Class[] param = constructors[i].getParameterTypes();
            long cost = 0;

            for (int j = 0; j < param.length; j++) {
                cost = 4 * cost;

                if (Object.class.equals(param[j]))
                    cost += 1;
                else if (String.class.equals(param[j]))
                    cost += 2;
                else if (int.class.equals(param[j]))
                    cost += 3;
                else if (long.class.equals(param[j]))
                    cost += 4;
                else if (param[j].isPrimitive())
                    cost += 5;
                else
                    cost += 6;
            }

            if (cost < 0 || cost > (1 << 48))
                cost = 1 << 48;

            cost += (long) param.length << 48;

            if (cost < bestCost) {
                _constructor = constructors[i];
                bestCost = cost;
            }
        }

        if (_constructor != null) {
            _constructor.setAccessible(true);
            Class[] params = _constructor.getParameterTypes();
            _constructorArgs = new Object[params.length];
            for (int i = 0; i < params.length; i++) {
                // 获取构造函数的入参
                _constructorArgs[i] = getParamArg(params[i]);
            }
        }
    }

    /**
     * Creates a map of the classes fields.
     */
    protected static Object getParamArg(Class cl) {
        // 如果当前类不是基础类型，则会返回null
        if (!cl.isPrimitive())
            return null;
        else if (boolean.class.equals(cl))
            return Boolean.FALSE;
        else if (byte.class.equals(cl))
            return new Byte((byte) 0);
        else if (short.class.equals(cl))
            return new Short((short) 0);
        else if (char.class.equals(cl))
            return new Character((char) 0);
        else if (int.class.equals(cl))
            return Integer.valueOf(0);
        else if (long.class.equals(cl))
            return Long.valueOf(0);
        else if (float.class.equals(cl))
            return Float.valueOf(0);
        else if (double.class.equals(cl))
            return Double.valueOf(0);
        else
            throw new UnsupportedOperationException();
    }
```

<br />
<br />从以上代码可以看出

- 由于BizException没有无参构造函数，所以cost最小的是只有一个入参的。从Class.getDeclaredConstructors()返回的列表看，第一个只有一个入参的构造函数是BizException(BizExceptionCode bizExceptionCode)

![image.png](1.png)

- 由于BizExceptionCode不是基础类型，所以在getParamArg时返回了null


<br />
<br />
<br />所以在构造时候,  使用传入的参数去获取数据时，就NPE。<br />源码如下:com.alibaba.com.caucho.hessian.io.JavaDeserializer#instantiate
```java
protected Object instantiate()
        throws Exception {
    try {
        if (_constructor != null)
            return _constructor.newInstance(_constructorArgs);
        else
            return _type.newInstance();
    } catch (Exception e) {
        throw new HessianProtocolException("'" + _type.getName() + "' could not be instantiated", e);
    }
}


public BizException(BizExceptionCode bizExceptionCode) {
	// 构造的时候， bizExceptionCode为null， 所以这里会报NPE
    super(bizExceptionCode.getCode(),bizExceptionCode.getMessage());
}
```

<br />
<br />
<br />
<br />

<a name="2Dkaf"></a>
### 解决
从反序列化时，选择构造函数的算法中可以知道，如果我们有一个默认构造器, 那么它将会被选中. 从而不会出现上述问题. 
```java
public class BizException extends BaseException {
	...
	
    // 默认构造函数
    public BizException(){}
	...
}
```

<br />有了默认构造器之后，后面反序列化器会将数据设置到相应的field，从而完成反序列化
```java
    @Override
    public Object readObject(AbstractHessianInput in, String[] fieldNames)
            throws IOException {
        try {
            // 实例化
            Object obj = instantiate();
			// 从input中读取数据，其中fieldNames则是类中的所有fieldNames
            return readObject(in, obj, fieldNames);
        } catch (IOException e) {
            throw e;
        } catch (RuntimeException e) {
            throw e;
        } catch (Exception e) {
            throw new IOExceptionWrapper(_type.getName() + ":" + e.getMessage(), e);
        }
    }
```


<a name="ibk7Q"></a>
### 总结
无参构造函数在很多地方都会有使用到，比如Spring的bean的初始化。 这里则是dubbo的序列化上。所以在编写代码的时候尽量提供无参构造函数，以免增加不必要的烦恼

