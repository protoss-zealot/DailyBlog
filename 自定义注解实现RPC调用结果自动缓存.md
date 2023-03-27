# 自定义注解实现RPC调用结果自动缓存

标签（空格分隔）： 多线程 缓存

---

### 前言

- 在高并发请求的场景下，一些远程调用经常会有一些数据是不怎么变化的，如果反复请求则会增加系统开销，同时万一出现服务异常也会引起后续问题，例如用户User的信息，基本不怎么变化，但在很多业务服务中却需要频繁使用。因此我们经常采用将不变化，同时又需要反复使用的数据缓存起来。
    

- 直接在业务代码中进行缓存当然是最简单、直接的方式，然而这样会使得工具型的缓存代码和业务逻辑耦合在一起，不易维护。所以，最好的办法当然是使用切面进行处理。将缓存相关处理与业务代码解耦。

- 为此，我们可以自定义一个注解RpcThreadCache，来指定需要缓存相关数据的调用，代码如下。
    

### 实践

#### 1. 自定义注解结构如下
 
```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface RpcThreadCache {
}
```
 
#### 2. 定义缓存结构

```java

public final class ThreadLocalCache {

    // 一个线程对应一个 RemoteCache 实例
    private static ThreadLocal<LocalCache> instance = new ThreadLocal<>();

    public static LocalCache getInstance() {
        return instance.get();
    }

    public static void setInstance() {
        instance.set(new LocalCache());
    }

    public static void remove() {
        instance.remove();
    }
}
```



#### 3. 本地缓存结构LocalCache的定义

 - 其实就是一个map，map的key为接口返回数据的类型Class，value为需要存储的数据的k对。
 - 这里还使用了一个技巧，就是将调用的参数用逗号分隔，拼接起来作为这个调用的key，例如sellerId=111, venture = SG，拼接起来为"111,SG",value为调用查user的结果，这样当请求同样的user信息时，就可以直接从缓存中获取，而无需额外的调用请求了。


```java

public class LocalCache {
    private final Map<Class<?>, Map<Object, Object>> cache = Maps.newHashMap();
    public static final Null NULL = new Null();

    public LocalCache() {

    }

    public void put(Class cl, Object k, Object v) {
        Map<Object, Object> map = cache.get(cl);
        if (map == null) {
            map = Maps.newHashMap();
            cache.put(cl, map);
        }
        if (v == null) {
            map.put(k, NULL);
        } else {
            map.put(k, v);
        }
    }


    public <T> T get(Class cl, Object k) {
        Object obj = get0(cl, k);
        if (obj == NULL) {
            return null;
        }
        return (T) obj;
    }

    public Object get0(Class cl, Object k) {
        Map<Object, Object> map = cache.get(cl);
        if (map == null) {
            return null;
        }
        return map.get(k);
    }

    public <T> T get(Class cl, Object k, ServiceCallBack<T> scb) throws Exception {
        if (scb == null) {
            throw new Exception("callback is null");
        }
        Object obj = get0(cl, k);
        if (obj == null) {
            obj = scb.callBack();
            if (obj == null) {
                obj = NULL;
            }
            put(cl, k, obj);
        } else if (obj.equals(NULL)) {
            return null;
        }
        return (T) obj;
    }

}

```

#### 4. 在方法调用时创建切面拦截进行处理

 - 创建缓存的拦截器

```java


public class ThreadLocalCacheInterceptor implements MethodInterceptor {

    @Override
    public Object invoke(MethodInvocation methodInvocation) throws Throwable {
        try {
            //initialize thread local
            //当有AppThreadCache标记的才进行cache
            if (methodInvocation.getMethod().isAnnotationPresent(AppThreadCache.class)) {
                ThreadLocalCache.setInstance();
            }
            return methodInvocation.proceed();
        } finally {
            //clear thread local
            ThreadLocalCache.remove();
        }
    }
}

```


#### 5. 远程调用时拦截
 
 - joinArguments 用来拼接当前方法调用的所有参数作为缓存的key。
 
 - 当使用同样的key进行调用时，若缓存中已经存在，则直接返回对应结果，否则加入到缓存中。


```java


public class RpcRepoInterceptor implements MethodInterceptor {

    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        //如果不是经过appThreadCache标记的, 则不需要cache
        //或者, 不带RpcThreadCache标记的方法不进行cache
        if (ThreadLocalCache.getInstance() == null
                || !invocation.getMethod().isAnnotationPresent(RpcThreadCache.class)) {
            return invocation.proceed();
        } else {
            String arguments = joinArguments(invocation.getArguments());
            if (StringUtils.isNotBlank(arguments)) {
                Class<?> clazz = invocation.getMethod().getReturnType();
                Object cached = ThreadLocalCache.getInstance().get0(clazz, arguments);
                if (cached != null) {
                    //进行null的特别处理, 当为特定的NULL时, 表示上次的查询结果为null
                    if (cached == LocalCache.NULL) {
                        return null;
                    } else {
                        return cached;
                    }
                } else {
                    //进行rpc, 然后cache
                    Object result = invocation.proceed();
                    ThreadLocalCache.getInstance().put(clazz, arguments, result);
                    return result;
                }
            } else {
                return invocation.proceed();
            }
        }
    }

    private String joinArguments(Object[] arguments) {
        if (arguments == null || arguments.length == 0) {
            return null;
        }

        StringBuilder sb = new StringBuilder();
        for (Object obj : arguments) {
            sb.append(String.valueOf(obj)).append(",");
        }
        if (sb.length() > 0) {
            sb.deleteCharAt(sb.length() - 1);
        }
        return sb.toString();
    }
}

```


#### 6. 接下来是配置拦截器生效的接口

- org.springframework.aop.support.RegexpMethodPointcutAdvisor 是Spring-AOP的切面定义点，指定哪些匹配的方法需要通过切面。
    

```java
    <bean id="rpcRepoInterceptor" class="com.xxxx.RpcRepoInterceptor"/>
    <bean class="org.springframework.aop.support.RegexpMethodPointcutAdvisor">
      <property name="patterns">
        <list>
          <value>com.xxxxx.getSellerUserById*</value>
          <value>com.xxxx.getConfigOption*</value>
        </list>
      </property>
      <property name="advice" ref="rpcRepoInterceptor"/>
    </bean>
    
```


#### 7. 最后在需要使用的方法上加上注解即可

 - 在需要缓存的调用上加上注解@RpcThreadCache

```java
/**
 * 读取用户针对Option的设置
 *
 * @param userId
 * @return
 */
@RpcThreadCache
public String getUserConfigOption(Long userId) {
    return getConfig(userId, CONFIG_OPTIONS_KEY);
}
