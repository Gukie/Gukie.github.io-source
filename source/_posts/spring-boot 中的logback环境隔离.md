
---
title: spring-boot 中的logback环境隔离
categories: 
    - springboot
    - logback

tags: 
    - springboot
    - logback

excerpt: 日常开发中，不同环境需要有不同的日志输出非常常见，这里简单记录下一个实践过程
---


<a name="b2tSi"></a>
### refer
- [https://www.cnblogs.com/easonjim/p/7801549.html](https://www.cnblogs.com/easonjim/p/7801549.html)
- [http://logback.qos.ch/manual/filters.html](http://logback.qos.ch/manual/filters.html)（自定义logback filter）
- [https://docs.spring.io/spring-boot/docs/1.5.7.RELEASE/reference/htmlsingle/#boot-features-custom-log-configuration](https://docs.spring.io/spring-boot/docs/1.5.7.RELEASE/reference/htmlsingle/#boot-features-custom-log-configuration) （spring关于日志的环境隔离的官网文档，你想要的这里都有）


<br />
<br />不同环境需要不同的日志输出，比如dev环境希望在启动的时候输出一些内容到console，提示是否启动成功了. 但是在其他环境上就没有必要输出到console了。所以这里就需要按环境区分对待了<br />
<br />
<br />由于各个服务采用的框架不同，不一定都是springboot. 所以在运维部署时并没有统一使用 spring.profiles.active进行环境隔离，而是使用自定义的环境变量进行隔离，比如 `-Denv=dev`<br />
<br />所以一开始想到的是自定义logback的filter进行处理
```java

public class MyCustomLogFilter extends Filter<ILoggingEvent> {

    private String targetEnv;
    /**
     * 以逗号分割
     */
    private String targetClasses;

    public void setTargetEnv(String targetEnv) {
        this.targetEnv = targetEnv;
    }

    public void setTargetClasses(String targetClasses) {
        this.targetClasses = targetClasses;
    }


    @Override
    public FilterReply decide(ILoggingEvent event) {
        String currEnv = System.getProperty("env");
        String currClassName = event.getLoggerName();

        boolean classNameMatched = isClassNameMatched(currClassName);
        boolean envMatched = StringUtils.isBlank(targetEnv) || StringUtils.equals(targetEnv, currEnv);
        if (envMatched && classNameMatched) {
            return FilterReply.ACCEPT;
        }
        return FilterReply.DENY;
    }

    private boolean isClassNameMatched(String currClassName) {
        String[] targetClasses = StringUtils.split(this.targetClasses, ",");
        if (targetClasses == null || targetClasses.length < 1) {
            return true;
        }
        return Arrays.asList(targetClasses).contains(currClassName);
    }


}
```

<br />
<br />后来同事给的建议说是使用springProfile可以达到此目的. 仔细想了下，在开发环境上如果想要将一些信息输出到console，则开发人员必须设置spring.profiles.active=dev，因为springboot中spring.profiles.active的值默认是default的。 所以觉得该方案挺不错的，即在logback的配置文件中加入springProfile

一开始是这样的:
```xml
<springProfile name="dev">
        <root>
            <level value="INFO"/>
            <appender-ref ref="CONSOLE"/>
            <appender-ref ref="InfoFile" />
            <appender-ref ref="WarnFile" />
            <appender-ref ref="ErrorFile" />
        </root>
    </springProfile>

    <springProfile name="!dev">
        <root>
            <level value="INFO"/>
            <appender-ref ref="InfoFile" />
            <appender-ref ref="WarnFile" />
            <appender-ref ref="ErrorFile" />
        </root>
    </springProfile>
```
 后来再调整成了以下形式，更加简洁了
```xml
    <root>
        <level value="INFO"/>
        <springProfile name="dev">
            <appender-ref ref="CONSOLE"/>
        </springProfile>
        <appender-ref ref="InfoFile" />
        <appender-ref ref="WarnFile" />
        <appender-ref ref="ErrorFile" />
    </root>
```

<br />最终的logback-spring.xml如下:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration debug="false">
    <springProperty scope="context" name="loggingRoot" source="logging.path"/>

    <contextName>demo</contextName>

    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern><![CDATA[
				%d{yyyy-MM-dd HH:mm:ss.SSS}, %L, %logger{300} %M %m%n
            ]]></pattern>
        </encoder>
        <filter class="com.xxx.MyCustomLogFilter">
            <targetEnv>dev</targetEnv>
            <targetClasses>com.xxx.Startup</targetClasses>
        </filter>
    </appender>


	<appender name="InfoFile"
		class="ch.qos.logback.core.rolling.RollingFileAppender">
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>INFO</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
        <file>${loggingRoot}/info.log</file>
		<rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
			<FileNamePattern>${loggingRoot}/info_%d{yyyy-MM-dd.HH}.log
			</FileNamePattern>
			<MaxHistory>7</MaxHistory>
		</rollingPolicy>
	</appender>

    <root>
        <level value="INFO"/>
        <springProfile name="dev">
            <appender-ref ref="CONSOLE"/>
        </springProfile>
        <appender-ref ref="InfoFile" />
        <appender-ref ref="WarnFile" />
        <appender-ref ref="ErrorFile" />
    </root>
</configuration>

```

