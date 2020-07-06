---
title: xml的 context:property-placeholder 的SpringBoot方式实现
categories: 
    - [spring, springboot]

tags: 
    - spring
    - springboot

excerpt: 当使用惯了springboot之后，就不再喜欢代码里有配置文件了. 比如项目中存在 <context:property-placeholder>，就希望使用springboot的方式替换掉，然后将xml文件去除，保持整个项目结构的统一干净

---


<br />当我们从配置中心读取配置信息的时候，通常会将读取配置中心的代码封装成一个bean. 然后将该bean赋予给PropertyPlaceHolder. <br />


<a name="CGtVZ"></a>
### spring配置文件方式实现
```xml
 <bean id="hulkConfigBean" class="com.xxx.config.MousikaPropertiesFactoryBean" >
        <property name="appName" value="demo"></property>
 </bean>
 <context:property-placeholder properties-ref="hulkConfigBean"/>
```

<br />

```java

public class MousikaPropertiesFactoryBean implements FactoryBean<Properties>, EnvironmentAware {
    protected static Logger logger = LoggerFactory.getLogger(MousikaPropertiesFactoryBean.class);
    private final Properties deployProperties = new Properties();
    private boolean isInit = false;
    private Object lock = new Object();
    private String appName;
    private List<String> nameSpaces;
    private Environment environment;
    List<Config> configs = new ArrayList<Config>();

    public MousikaPropertiesFactoryBean() {
    }

    public Class<?> getObjectType() {
        return Properties.class;
    }

    public boolean isSingleton() {
        return true;
    }

    /** 这是 FactoryBean 获取bean时调用的方法，以下这种方式会调用更改犯法
    <bean id="hulkConfigBean" class="com.xxx.config.MousikaPropertiesFactoryBean" >
        <property name="appName" value="demo"></property>
 	</bean>
    */
    public Properties getObject() throws Exception {
        if (!this.isInit) {
            synchronized(this.lock) {
                if (!this.isInit) {
                    ...
                    Properties properties =    this.getAppConfigFromApollo();
                    this.deployProperties.putAll(properties);
                    ...
                    this.isInit = true;
                }
            }
        }

        return this.deployProperties;
    }

    public void setAppName(String appName) {
        this.appName = appName;
    }

    
    private Properties getAppConfigFromApollo() throws Exception {

        Properties properties = new Properties();
        // 从配置中心获取配置信息
        configs.add(getConfigFromRemote())
        if (configs.size() > 0) {
            for (Config config : configs) {
                properties.putAll(config.getAllProperties());
            }
        }
        return properties;
    }

    ...
}
```


<a name="Sqhpj"></a>
### 运行机制解析


<a name="C4jzH"></a>
#### xml文件设置过程

Spring在启动的时候，会解析xml文件，是

- 通过org.springframework.context.config.PropertyPlaceholderBeanDefinitionParser进行解析
- 然后将属性设置在org.springframework.context.support.PropertySourcesPlaceholderConfigurer中
<br />![image.png](1.png)<br />
<br />

1. 解析xml `<context:property-placeholder properties-ref="configBean"/>` 的代码是
代码位置: org.springframework.context.config.AbstractPropertyLoadingBeanDefinitionParser#doParse
```java
@Override
	protected void doParse(Element element, ParserContext parserContext, BeanDefinitionBuilder builder) {
		...
		String propertiesRef = element.getAttribute("properties-ref");
		if (StringUtils.hasLength(propertiesRef)) {
			builder.addPropertyReference("properties", propertiesRef);
		}
		...
	}
```


2. 将beanName以RuntimeBeanReference的形式包装起来:
代码位置： org.springframework.beans.factory.support.BeanDefinitionBuilder#addPropertyReference
```java
public BeanDefinitionBuilder addPropertyReference(String name, String beanName) {
    // 这里将beanName以 RuntimeBeanReference包装起来
    this.beanDefinition.getPropertyValues().add(name, new RuntimeBeanReference(beanName));
    return this;
}
```


<a name="HOAvi"></a>
#### propertyValue数据读取与设置

<br />在处理 org.springframework.context.config.PropertyPlaceholderBeanDefinitionParser 的时候，会通过以下代码进行处理<br />
1. org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#applyPropertyValues
```java
protected void applyPropertyValues(String beanName, BeanDefinition mbd, BeanWrapper bw, PropertyValues pvs) {
		...
		for (PropertyValue pv : original) {
			if (pv.isConverted()) {
				deepCopy.add(pv);
			}
			else {
				String propertyName = pv.getName();
				Object originalValue = pv.getValue();
                // 这里会去获取 hulkConfigBean 的bean对象. 通过上文可知获取到的是一个Properties
				Object resolvedValue = valueResolver.resolveValueIfNecessary(pv, originalValue);
				...
	}
```
![image.png](2.png)
2. org.springframework.beans.factory.support.BeanDefinitionValueResolver#resolveValueIfNecessary
```java
public Object resolveValueIfNecessary(Object argName, Object value) {
		// We must check each value to see whether it requires a runtime reference
		// to another bean to be resolved.
		if (value instanceof RuntimeBeanReference) {
			RuntimeBeanReference ref = (RuntimeBeanReference) value;
			return resolveReference(argName, ref);
		}
    ....
}

private Object resolveReference(Object argName, RuntimeBeanReference ref) {
		try {
			String refName = ref.getBeanName();
			refName = String.valueOf(doEvaluate(refName));
			if (ref.isToParent()) {
				if (this.beanFactory.getParentBeanFactory() == null) {
					throw new BeanCreationException(
							this.beanDefinition.getResourceDescription(), this.beanName,
							"Can't resolve reference to bean '" + refName +
							"' in parent factory: no parent factory available");
				}
				return this.beanFactory.getParentBeanFactory().getBean(refName);
			}
			else {
                // 会走到这里，然后获取hulkConfigBean的内容(一个properties)
				Object bean = this.beanFactory.getBean(refName);
				this.beanFactory.registerDependentBean(refName, this.beanName);
				return bean;
			}
		}
		catch (BeansException ex) {
			throw new BeanCreationException(
					this.beanDefinition.getResourceDescription(), this.beanName,
					"Cannot resolve reference to bean '" + ref.getBeanName() + "' while setting " + argName, ex);
		}
	}
```

<br />

<a name="g5zNs"></a>
### SpringBoot 等价写法
不采用xml的方式，使用编程的方式的写法如下:
```java
@Configuration
public class NonMemBizConfig {
    @Bean
    public MousikaPropertiesFactoryBean hulkConfigBean() {
        MousikaPropertiesFactoryBean item = new MousikaPropertiesFactoryBean();
        item.setAppName("demo");
        return item;
    }

    @Bean
    public PropertySourcesPlaceholderConfigurer propertySourcesPlaceholderConfigurer() {

        PropertySourcesPlaceholderConfigurer propertySourcesPlaceholderConfigurer = new PropertySourcesPlaceholderConfigurer();
        try {
            propertySourcesPlaceholderConfigurer.setProperties(hulkConfigBean().getObject());
        } catch (Exception e) {
            e.printStackTrace();
        }
        return propertySourcesPlaceholderConfigurer;
    }


}
```
