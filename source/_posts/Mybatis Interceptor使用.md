---
title: Mybatis Interceptor使用
categories: 
    - mybatis
    - web开发
tags: 
    - mybatis
---

### 为什么Interceptor只能是固定的几种类型

<br />首先，interceptor中的类型，目前只有以下四种才能生效:

- Executor (update, query, flushStatements, commit, rollback, getTransaction, close, isClosed)
- ParameterHandler (getParameterObject, setParameters)
- ResultSetHandler (handleResultSets, handleOutputParameters)
- StatementHandler (prepare, parameterize, batch, update, query)


<br />
<br />不能自定义，因为这样是找不到的，比如以下这种自定义的 LockPwdDAO.class 是找不到的
```java
@Intercepts(
        {@Signature(type = LockPwdDAO.class, method = "save", args = {BaseDO.class}),
                @Signature(type = LockPwdDAO.class, method = "add", args = {BaseDO.class}),
                @Signature(type = LockPwdDAO.class, method = "update", args = {BaseDO.class}),
                @Signature(type = LockPwdDAO.class, method = "add", args = {List.class}),
                @Signature(type = LockPwdDAO.class, method = "update", args = {List.class}),
                @Signature(type = LockPwdDAO.class, method = "updateByIds", args = {BaseDO.class})}
)

@Slf4j
public class TimeStampTransferInterceptor implements Interceptor {
    ...
}
```

<br />
<br />这是因为org.apache.ibatis.plugin.Plugin#wrap中会做校验，不是规范的类，是不会被处理的
```java
public static Object wrap(Object target, Interceptor interceptor) {
    Map<Class<?>, Set<Method>> signatureMap = getSignatureMap(interceptor);
    Class<?> type = target.getClass();
	// 这里出来是空的
    Class<?>[] interfaces = getAllInterfaces(type, signatureMap);
    if (interfaces.length > 0) {
      return Proxy.newProxyInstance(
          type.getClassLoader(),
          interfaces,
          new Plugin(target, interceptor, signatureMap));
    }
    return target;
  }


private static Class<?>[] getAllInterfaces(Class<?> type, Map<Class<?>, Set<Method>> signatureMap) {
    Set<Class<?>> interfaces = new HashSet<>();
    while (type != null) {
      for (Class<?> c : type.getInterfaces()) {
          // 因为我们自定义的类，在这里是不会为true的. 因为c是: Executor,ParameterHandler, ResultSetHandler,StatementHandler 中的一个
        if (signatureMap.containsKey(c)) {
          interfaces.add(c);
        }
      }
      type = type.getSuperclass();
    }
    return interfaces.toArray(new Class<?>[interfaces.size()]);
  }
```

<br />可以通过查看 org.apache.ibatis.plugin.Plugin#getSignatureMap 的调用链获取更详细的信息<br />![image.png](invoke-chain.png)<br />
<br />org.apache.ibatis.plugin.InterceptorChain#pluginAll的入参是传入的，也就是 Executor/ ParameterHandler/ResultSetHandler/ StatementHandler 中的一个，知道这个之后@Intercepts的@Signature中的method也就知道怎么配置就一目了然了
```java
public class InterceptorChain {

  private final List<Interceptor> interceptors = new ArrayList<>();

  public Object pluginAll(Object target) {
    for (Interceptor interceptor : interceptors) {
      target = interceptor.plugin(target);
    }
    return target;
  }

  public void addInterceptor(Interceptor interceptor) {
    interceptors.add(interceptor);
  }

  public List<Interceptor> getInterceptors() {
    return Collections.unmodifiableList(interceptors);
  }

}
```

<br />以下是 org.apache.ibatis.session.Configuration#newParameterHandler 的代码摘要:
```java
public ParameterHandler newParameterHandler(MappedStatement mappedStatement, Object parameterObject, BoundSql boundSql) {
    ParameterHandler parameterHandler = mappedStatement.getLang().createParameterHandler(mappedStatement, parameterObject, boundSql);
    parameterHandler = (ParameterHandler) interceptorChain.pluginAll(parameterHandler);
    return parameterHandler;
  }
```

<br />
<br />

<a name="hL9hH"></a>
### 最终可用代码示例

<br />以下的目的是: 对时间的统一处理，以免有的人用的时候传的13位(毫秒)，有的人使用的时候传的是10位，即秒
```java

@Intercepts( {
        @Signature(type = Executor.class, method = "update", args = {MappedStatement.class, Object.class})
})
@Slf4j
public class TimeStampTransferInterceptor implements Interceptor {

    private static final String targetStatementIdPrefix = "com.xxx.dao.yyy.impl.LockPwdDAO";
    private static final List<String> targetMethod = Arrays.asList("save", "add", "update", "batchUpdate", "updateByIds");

    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        Object[] args = invocation.getArgs();
        MappedStatement mappedStatement = (MappedStatement) args[0];
        Object param = args[1];
        try {
            handleEffectiveInvalidTimeSet(mappedStatement, param);
        } catch (Exception e) {
            log.error("[time-stamp-interceptor] ", e);
        }
        return invocation.proceed();
    }

    private void handleEffectiveInvalidTimeSet(MappedStatement mappedStatement, Object param) {
        // param
        String mappedStatementId = mappedStatement.getId();
        boolean isTargetMethod = isTargetStatement(mappedStatementId);
        if (!isTargetMethod) {
            return;
        }

        log.info("[time-stamp-interceptor] fix effective/invalid time" );
        if (param instanceof LockPwdDO) {
            LockPwdDO lockPwd = (LockPwdDO) param;
            updateEffectiveInvalidTime2Second(lockPwd);
        }
        // see: org.apache.ibatis.session.defaults.DefaultSqlSession.wrapCollection
        if (param instanceof DefaultSqlSession.StrictMap) {
            DefaultSqlSession.StrictMap strictMap= (DefaultSqlSession.StrictMap) param;
            List<LockPwdDO> lockPwds = (List<LockPwdDO>) strictMap.get("list");
            if (CollectionUtils.isEmpty(lockPwds)) {
                return;
            }
            for (LockPwdDO lockPwd : lockPwds) {
                updateEffectiveInvalidTime2Second(lockPwd);
            }
        }
    }

    private boolean isTargetStatement(String mappedStatementId) {

        boolean isLockPwdStatement = StringUtils.startsWith(mappedStatementId, targetStatementIdPrefix);
        if (!isLockPwdStatement) {
            return false;
        }
        int lastDotIndex = targetStatementIdPrefix.length() + 1;
        String method = mappedStatementId.substring(lastDotIndex);
        return targetMethod.contains(method);
    }

    private void updateEffectiveInvalidTime2Second(LockPwdDO lockPwd) {
        if (lockPwd == null) {
            return;
        }
        Long originalEffectiveTime = lockPwd.getEffectiveTime();
        Long originalInvalidTime = lockPwd.getInvalidTime();
        lockPwd.setEffectiveTime(format2FixedLength(originalEffectiveTime, 10));
        lockPwd.setInvalidTime(format2FixedLength(originalInvalidTime, 10));
    }

    private Long format2FixedLength(Long originalTime, int targetLength) {
        if (originalTime == null) {
            return originalTime;
        }
        int currLength = originalTime.toString().length();

        int diffLen = targetLength - currLength;
        if (diffLen == 0) {
            return originalTime;
        }
        double factor = Math.pow(10, Math.abs(diffLen));
        if (diffLen < 0) {
            Double tmp = originalTime / factor;
            return tmp.longValue();
        }
        if (diffLen > 0) {
            Double tmp = originalTime * factor;
            return tmp.longValue();

        }
        return originalTime;
    }
}

```

<br />定义好之后，需要将interceptor注入进 
```xml
<bean name="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
        <property name="dataSource" ref="myDataSource"/>
        <property name="typeAliasesPackage" value="com.xxx.entity"/>
        <property name="mapperLocations" value="classpath*:conf/mybatis/*.xml"/>
        <property name="plugins">
            <array>
                <bean id="lockPwdInterceptor" class="com.xxx.interceptor.TimeStampTransferInterceptor"/>
            </array>
        </property>
    </bean>
```

### 多个interceptor的执行顺序

<br />定义在后面的会先执行。 以下定义中，`interceptor1` 会在 `interceptor2`之前执行
```xml
<property name="plugins">
    <array>
        <bean id="interceptor2" class="com.tuya.hulk.dal.interceptor.Interceptor2"/>
        <bean id="interceptor1" class="com.tuya.hulk.dal.interceptor.Interceptor1"/>
    </array>
</property>

```

<br />执行的时候，是以以下顺序进行执行的![interceptor-invoke-sequence.jpg](interceptor-invoke-sequence.jpg)<br />
<br />所以在xml定义中的最后一个interceptor会被第一个执行，但是却最后一个执行完(因为要等其他interceptor执行完之后它才能结束)，所以可以在该Interceptor上做一些统一的处理(比如Exception的兜底处理，时间的统计等), 类似:
```java
public Object intercept(Invocation invocation) throws Throwable {
        try {
            return invocation.proceed();
        } catch (Throwable ex) {
            // 统一的异常处理
            ...
        }
    }
```

<br />那为什么执行顺序是这样的呢？<br />从 `org.apache.ibatis.plugin.InterceptorChain#pluginAll` 中可以找到答案:
```java
public class InterceptorChain {

  private final List<Interceptor> interceptors = new ArrayList<>();

  // 答案在这里
  public Object pluginAll(Object target) {
    for (Interceptor interceptor : interceptors) {
      target = interceptor.plugin(target);
    }
    return target;
  }

  public void addInterceptor(Interceptor interceptor) {
    interceptors.add(interceptor);
  }

  public List<Interceptor> getInterceptors() {
    return Collections.unmodifiableList(interceptors);
  }

}
```


1. 首先， 给 target 做plugin的时候，是按照 interceptors的顺序进行的，也就是按照interceptor在xml中的定义进行的
1. 最上面的，先进行 `target = interceptor.plugin(target);` 后面的后进行
1. 我们注意到  `interceptor.plugin(target); ` 执行完之后是会将结果覆盖掉 target的，即每做一次 拦截器的plugin，target都是会变更的. 所以
  1. 拦截器1 被插入到target
  1. 之后拦截器2再插进去
  1. 再之后是拦截器3
  1. 这样就形成像将羽毛球放入桶里一样，先进去的在最底下后进去的在最上面这样的队形:
```
拦截器3-->拦截器2-->拦截器1
```


4. 所以在执行的时候，是在最上面的先进行执行, 即 拦截器3
