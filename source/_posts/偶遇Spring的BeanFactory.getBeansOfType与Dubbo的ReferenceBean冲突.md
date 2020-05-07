# Spring的 BeanDefinitionxxPostProcessor与 dubbo的冲突


<br />

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
从报错看，我们可以讲断点放在 `CuratorZookeeperClient.java:83`， 然后会发现，此时的地址都是占位符的<br />![image.png](https://cdn.nlark.com/yuque/0/2020/png/289364/1588847505069-b8e3d43c-8a5e-475e-bf9d-fbcdbee14f18.png#align=left&display=inline&height=116&margin=%5Bobject%20Object%5D&name=image.png&originHeight=232&originWidth=1558&size=45480&status=done&style=none&width=779)<br />
<br />此时我们可以将我们自己写的代码注释掉，看看结果是什么样的，会发现其实是有值的，那么可以断定是代码的问题<br />![image.png](https://cdn.nlark.com/yuque/0/2020/png/289364/1588847659913-82bbb5ac-7754-4142-9bd5-acc3534de032.png#align=left&display=inline&height=170&margin=%5Bobject%20Object%5D&name=image.png&originHeight=340&originWidth=1652&size=58743&status=done&style=none&width=826)<br />
<br />
<br />
<br />经过排查，发现这一行代码有问题
```java
Map<String, IProxyBaseService> nameInstanceMap = beanFactory.getBeansOfType(IProxyBaseService.class);
```
该代码会去找IProxyBaseService类的bean，但是它是使用了缺省参数的方法:<br />org.springframework.beans.factory.support.DefaultListableBeanFactory#getBeansOfType<br />![image.png](https://cdn.nlark.com/yuque/0/2020/png/289364/1588847864017-878ee6fd-c842-4932-9c04-d3e72e3fb25a.png#align=left&display=inline&height=82&margin=%5Bobject%20Object%5D&name=image.png&originHeight=164&originWidth=1420&size=37848&status=done&style=none&width=710)<br />
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
							// 会走到这里	
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

<br />![image.png](https://cdn.nlark.com/yuque/0/2020/png/289364/1588848219378-9e7cf3a4-76bf-4329-a9f8-e0a4d241ac1e.png#align=left&display=inline&height=603&margin=%5Bobject%20Object%5D&name=image.png&originHeight=1206&originWidth=2742&size=457609&status=done&style=none&width=1371)<br />而org.apache.dubbo.config.spring.ReferenceBean 正好是一个FactoryBean
```java
public class ReferenceBean<T> extends ReferenceConfig<T> implements FactoryBean, ApplicationContextAware, InitializingBean, DisposableBean {
	...
}
```

<br />
<br />这样就会尝试去实例化ReferenceBean了，最后就会走到org.apache.dubbo.config.spring.ReferenceBean#afterPropertiesSet， 由于现在还只是在BeanDefinition处理阶段，还并没有到占位符的设置阶段，所以是读取不到占位符的值的，所以它还是原来的模样: ${zookeeper.address}, 并没有变形<br />![image.png](https://cdn.nlark.com/yuque/0/2020/png/289364/1588848509239-897fae83-12f2-4879-9de6-42f556f7f5d6.png#align=left&display=inline&height=600&margin=%5Bobject%20Object%5D&name=image.png&originHeight=1200&originWidth=2788&size=477519&status=done&style=none&width=1394)<br />
<br />

<a name="up6HB"></a>
### 解决
知道问题的原因了，改起来也很简单，就是 使用另外一个方法: `beanFactory.getBeansOfType(IProxyBaseService.class,false, false);`<br />最终代码如下：
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
