---
categories: spring
---


<a name="MojXv"></a>
### 背景及问题发现
有时候，我们会在spring的bean实例化之前做一些校验，比如校验某些package下的类的命名是否符合规范，此时就会去实现 BeanDefinitionRegistryPostProcessor 做一些事情<br />
<br />我们假设，我们需要校验的类都实现了一个interface，名叫: IProxyBaseService， 那么我们很自然的就会有以下思路

- 获取所有 IProxyBaseService的bean的定义
- 通过该bean找到对应的class，然后通过反射校验他们的方法是否合规


<br />代码如下
```java
public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {

        Map<String, IProxyBaseService> nameInstanceMap = beanFactory.getBeansOfType(IProxyBaseService.class);
        if (MapUtils.isEmpty(nameInstanceMap)) {
            return;
        }
        nameInstanceMap.forEach((beanName, beanInstance) -> {
            Set<Class<?>> interfaces = ClassUtils.getAllInterfacesAsSet(beanInstance);
            if (CollectionUtils.isEmpty(interfaces)) {
                return;
            }
            for (Class<?> interfaceCls : interfaces) {
                if (!ClassUtils.isAssignable(IProxyBaseService.class, interfaceCls)) {
                    continue;
                }
                checkInterfaceMethods(interfaceCls);
            }
        });

    }
```

<br />如果系统中使用了dubbo，那么坑就来了: 当果你的dubbo的配置是通过占位符来做的，那么你将获取不到对应占位符的值
```xml
 <dubbo:registry protocol="zookeeper" address="${zookeeper.address}"/>
```
最后报以下错误:
```
Caused by: java.lang.IllegalStateException: zookeeper not connected
    at org.apache.dubbo.remoting.zookeeper.curator.CuratorZookeeperClient.<init>(CuratorZookeeperClient.java:83)
    at org.apache.dubbo.remoting.zookeeper.curator.CuratorZookeeperTransporter.createZookeeperClient(CuratorZookeeperTransporter.java:26)
    at org.apache.dubbo.remoting.zookeeper.support.AbstractZookeeperTransporter.connect(AbstractZookeeperTransporter.java:68)
    at org.apache.dubbo.remoting.zookeeper.ZookeeperTransporter$Adaptive.connect(ZookeeperTransporter$Adaptive.java)
    at org.apache.dubbo.configcenter.support.zookeeper.ZookeeperDynamicConfiguration.<init>(ZookeeperDynamicConfiguration.java:62)
    at org.apache.dubbo.configcenter.support.zookeeper.ZookeeperDynamicConfigurationFactory.createDynamicConfiguration(ZookeeperDynamicConfigurationFactory.java:37)
```

<br />这时候如果没有认真debug过，会认为zk链接不上了，然后把问题指向zk，从而偏离了问题的本质<br />

<a name="MERTr"></a>
### 排查过程
从报错看，我们可以讲断点放在 `CuratorZookeeperClient.java:83`， 然后会发现，此时的地址都是占位符的<br />![image.png](spring-1.png)<br />
<br />此时我们可以将我们自己写的代码注释掉，看看结果是什么样的，会发现其实是有值的，那么可以断定是代码的问题<br />![image.png](spring-2.png)<br />
<br />
<br />
<br />经过排查，发现这一行代码有问题
```java
Map<String, IProxyBaseService> nameInstanceMap = beanFactory.getBeansOfType(IProxyBaseService.class);
```
该代码会去找IProxyBaseService类的bean，但是它是使用了缺省参数的方法:<br />org.springframework.beans.factory.support.DefaultListableBeanFactory#getBeansOfType<br />![image.png](spring-3.png)<br />
<br />再往下会发现<br />org.springframework.beans.factory.support.DefaultListableBeanFactory#doGetBeanNamesForType
```java
// 这里的 allowEagerInit 会让条件为true
if (!mbd.isAbstract() && (allowEagerInit ||
        (mbd.hasBeanClass() || !mbd.isLazyInit() || isAllowEagerClassLoading()) &&
                !requiresEagerInitForType(mbd.getFactoryBeanName()))) {
    // In case of FactoryBean, match object created by FactoryBean.
    boolean isFactoryBean = isFactoryBean(beanName, mbd);
    BeanDefinitionHolder dbd = mbd.getDecoratedDefinition();
    boolean matchFound =
            (allowEagerInit || !isFactoryBean ||
                    (dbd != null && !mbd.isLazyInit()) || containsSingleton(beanName)) &&
            (includeNonSingletons ||
                    (dbd != null ? mbd.isSingleton() : isSingleton(beanName))) &&
        // 会进入到这里去  
        isTypeMatch(beanName, type);
    if (!matchFound && isFactoryBean) {
        // In case of FactoryBean, try to match FactoryBean instance itself next.
        beanName = FACTORY_BEAN_PREFIX + beanName;
        matchFound = (includeNonSingletons || mbd.isSingleton()) && isTypeMatch(beanName, type);
    }
    if (matchFound) {
        result.add(beanName);
    }
}
```

<br />然后会进入到<br />org.springframework.beans.factory.support.AbstractBeanFactory#isTypeMatch的以下代码段
```java
// 这里会校验当前类是否是FactoryBean
if (FactoryBean.class.isAssignableFrom(beanType)) {
    if (!BeanFactoryUtils.isFactoryDereference(name)) {
        // If it's a FactoryBean, we want to look at what it creates, not the factory class.
        beanType = getTypeForFactoryBean(beanName, mbd);
        if (beanType == null) {
            return false;
        }
    }
}
```

<br />![image.png](spring-4.png)<br />而org.apache.dubbo.config.spring.ReferenceBean 正好是一个FactoryBean
```java
public class ReferenceBean<T> extends ReferenceConfig<T> implements FactoryBean, ApplicationContextAware, InitializingBean, DisposableBean {
    ...
}
```

<br />
<br />这样就会尝试去实例化ReferenceBean了，最后就会走到org.apache.dubbo.config.spring.ReferenceBean#afterPropertiesSet， 由于现在还只是在BeanDefinition处理阶段，还并没有到占位符的设置阶段，所以是读取不到占位符的值的，所以它还是原来的模样: ${zookeeper.address}, 并没有变形<br />
<br />

![image.png](spring-5.png)


<a name="up6HB"></a>
### 解决
知道问题的原因了: beanFactory.getBeansOfType的参数allowEagerInit=true时会将FactoryBean初始化掉。<br />所以改起来也很简单，就是使用另外一个方法:<br />`beanFactory.getBeansOfType(IProxyBaseService.class,false, false);`<br />
<br />最终代码如下：
```java
public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {

        Map<String, IProxyBaseService> nameInstanceMap = beanFactory.getBeansOfType(IProxyBaseService.class,false, false);
        if (MapUtils.isEmpty(nameInstanceMap)) {
            return;
        }
        nameInstanceMap.forEach((beanName, beanInstance) -> {
            Set<Class<?>> interfaces = ClassUtils.getAllInterfacesAsSet(beanInstance);
            if (CollectionUtils.isEmpty(interfaces)) {
                return;
            }
            for (Class<?> interfaceCls : interfaces) {
                if (!ClassUtils.isAssignable(IProxyBaseService.class, interfaceCls)) {
                    continue;
                }
                checkInterfaceMethods(interfaceCls);
            }
        });

    }
```

<br />

<a name="LfS6k"></a>
### 事情还在继续

<br />然而事情并没有那么简单，在后面的测试中，会发现，实现了IProxyBaseService的类，其field都是null.<br />![image.png](spring-6.png)<br />
<br />这很严重啊。既然是都为空，那么看看它是什么时候进行初始化的，然后找了一个类，在其后面加了个InitializingBean进行断点调试
```java
public class DeviceProxyServiceImpl implements IDeviceProxyService, InitializingBean{
    ...
    @Override
    public void afterPropertiesSet() throws Exception {
        log.info("....");
    }
}
```

<br />
<br />debug后，发现这是从 beanFactory.getBeansOfType(IProxyBaseService.class,false, false); 调用过来的。<br />代码如下: org.springframework.beans.factory.support.DefaultListableBeanFactory#getBeansOfType<br />

```java
public <T> Map<String, T> getBeansOfType(Class<T> type, boolean includeNonSingletons, boolean allowEagerInit)
            throws BeansException {
        // 1.这一行，是FactoryBean初始化的问题的根源
        String[] beanNames = getBeanNamesForType(type, includeNonSingletons, allowEagerInit);
        
        Map<String, T> result = new LinkedHashMap<String, T>(beanNames.length);
        // 2.可是事情并没有完，当找出了IProxyBaseService这个类的beanName之后，就到了这
        for (String beanName : beanNames) {
            try {
               // 这里有一个getBean，然后就会去创建bean. 而此时bean都还没有实例化出来，所以都是null 
                result.put(beanName, getBean(beanName, type));
            }
            catch (BeanCreationException ex) {
                ...
            }
        }
        return result;
    }
```

<br />解决方法是, 不使用beanFactory.getBeansOfType， 而是使用BeanDefinitionRegistry去获取对应的beanName，然后找到对应的class做自己的业务
```java
 @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
        String[] beanNames = registry.getBeanDefinitionNames();
        if (beanNames == null || beanNames.length < 1) {
            return;
        }
        for (String beanName : beanNames) {
            BeanDefinition beanDefinition = registry.getBeanDefinition(beanName);
            String beanClsName = beanDefinition.getBeanClassName();
            if (!StringUtils.contains(beanClsName, "com.xxx.impl")) {
                continue;
            }

            Class cls = ClassUtils.resolveClassName(beanClsName, Thread.currentThread().getContextClassLoader());
            Set<Class<?>> interfaces = ClassUtils.getAllInterfacesAsSet(cls);
            if (CollectionUtils.isEmpty(interfaces)) {
                continue;
            }
            for (Class<?> interfaceCls : interfaces) {
                if (!ClassUtils.isAssignable(IProxyBaseService.class, interfaceCls)) {
                    continue;
                }
                checkInterfaceMethods(interfaceCls);
            }
        }

    }
```

<br />

<a name="dyGDj"></a>
### 总结

- 在bean没有实例化好的时候，不能随便使用beanFactory.getBeansOfType这个方法，这个会坑死人的
- 在bean没有实例化时做一些校验，能减少系统资源的开销(链接创建/资源分配等)，但实现的时候需要千万小心


<br />

