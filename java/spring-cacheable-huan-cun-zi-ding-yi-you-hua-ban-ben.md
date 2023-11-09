# \[Spring @Cacheable缓存自定义]优化版本

> 早些前写了Cacheable的自定义缓存啥的, 这次优化了一下[Cacheable注解错误时缓存](https://roothk.top/index.php/archives/11/)

**需缓存的方法**

```
@Cacheable(key = "#cardNo + '_' + #unifTranId", cacheNames = "bill.usTrade")
public TradeSkinResDto<StmtTradeResDto> usTrade(String cardNo, String unifTranId) {
    return null;
}
```

**AOP**

```
import lombok.extern.slf4j.Slf4j;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.Signature;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Pointcut;
import org.aspectj.lang.reflect.MethodSignature;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.cache.Cache;
import org.springframework.cache.CacheManager;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.core.DefaultParameterNameDiscoverer;
import org.springframework.core.annotation.Order;
import org.springframework.data.redis.cache.RedisCacheConfiguration;
import org.springframework.data.redis.cache.RedisCacheManager;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.expression.EvaluationContext;
import org.springframework.expression.Expression;
import org.springframework.expression.spel.standard.SpelExpressionParser;
import org.springframework.expression.spel.support.StandardEvaluationContext;
import org.springframework.stereotype.Component;

import java.lang.reflect.Method;
import java.util.concurrent.TimeUnit;

/**
 * redis缓存调用切面
 * <p>
 * 由于服务调用时, 返回的可能是错误的信息, 因此需要将错误信息也缓存起来
 * <p>
 * 方法中使用这个 @Cacheable 注解, 就会处理返回的参数
 *
 * @author roothk
 * @date 2020/4/23 8:53
 */
@Slf4j
@Order(1)
@Aspect
@Component
public class CacheableHandlerAspect {

    @Autowired
    private CacheableUtil cacheableUtil;

    @Pointcut("@annotation(org.springframework.cache.annotation.Cacheable)")
    public void feignAspectPointCut() {
    }

    /**
     * 环绕通知 @Around  ， 当然也可以使用 @Before (前置通知)  @After (后置通知)
     *
     * @param point
     * @return
     * @throws Throwable
     */
    @Around("feignAspectPointCut()")
    public Object around(ProceedingJoinPoint point) throws Throwable {
        Cacheable cacheable = cacheableUtil.getCacheable(point);
        String cacheName = cacheable.cacheNames()[0];
        String key = cacheableUtil.getValueByKey(cacheable.key(), point);

        try {
            // 错误处理
            handlerException(key, cacheName);

            Object o = point.proceed();
            // 其他正常返回, 如空对象
            if (EsbThreadLocal.get()) {
                // 也依旧缓存, 防止缓存穿透
                cacheableUtil.saveCache(o, key, cacheName);
            }
            return o;
        } catch (EsbException e) {
            cacheableUtil.saveCache(e, key, cacheName);
            throw e;
        }
    }

    /**
     * 如果缓存报错的是Exception, 则直接抛出Exception
     *
     * @param key
     * @param cacheName
     */
    private void handlerException(Object key, String cacheName) {
        Object o = cacheableUtil.getCache(key, cacheName);
        if (o == null) {
            return;
        }
        // 如果是错误就直接输出
        if (o instanceof EsbException) {
            throw (EsbException) o;
        } else if (o instanceof EsbHystrixException) {
            throw (EsbHystrixException) o;
        } else if (o instanceof EsbServiceException) {
            throw (EsbServiceException) o;
        }
    }

}
```

**CacheableUtil工具类**

```
import lombok.extern.slf4j.Slf4j;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.Signature;
import org.aspectj.lang.reflect.MethodSignature;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.cache.Cache;
import org.springframework.cache.CacheManager;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.core.DefaultParameterNameDiscoverer;
import org.springframework.data.redis.cache.RedisCacheConfiguration;
import org.springframework.data.redis.cache.RedisCacheManager;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.expression.EvaluationContext;
import org.springframework.expression.Expression;
import org.springframework.expression.spel.standard.SpelExpressionParser;
import org.springframework.expression.spel.support.StandardEvaluationContext;
import org.springframework.stereotype.Component;

import java.lang.reflect.Method;
import java.util.concurrent.TimeUnit;

/**
 * @author RootHK
 */
@Slf4j
@Component
public class CacheableUtil {

    private final SpelExpressionParser parserSpel = new SpelExpressionParser();
    private final DefaultParameterNameDiscoverer parameterNameDiscoverer = new DefaultParameterNameDiscoverer();

    // 错误时的缓存时间
    @Value("${esb.cache.not.result.time}")
    private Long exceptionTimeout;

    @Autowired
    private CacheManager cacheManager;
    @Autowired
    private RedisCacheManager redisCacheManager;
    @Autowired
    private StringRedisTemplate stringRedisTemplate;

    /**
     * 获取方法上的 @Cacheable
     *
     * @param point
     * @return
     */
    public Cacheable getCacheable(ProceedingJoinPoint point) {
        Signature signature = point.getSignature();
        MethodSignature methodSignature = (MethodSignature) signature;
        Method targetMethod = methodSignature.getMethod();
        return targetMethod.getAnnotation(Cacheable.class);
    }

    /**
     * 获取Spring Sl语法的值
     * @param key
     * @param pjp
     * @return
     */
    public String getValueByKey(String key, ProceedingJoinPoint pjp) {
        MethodSignature methodSignature = (MethodSignature) pjp.getSignature();
        return getValueByKey(key, methodSignature.getMethod(), pjp.getArgs());
    }

    /**
     * 获取Spring Sl语法的值
     * @param key
     * @param m
     * @param args
     * @return
     */
    public String getValueByKey(String key, Method m, Object[] args) {
        Expression expression = parserSpel.parseExpression(key);
        EvaluationContext context = new StandardEvaluationContext();
        String[] paramNames = parameterNameDiscoverer.getParameterNames(m);
        for (int i = 0; i < args.length; i++) {
            if (paramNames == null) {
                continue;
            }
            context.setVariable(paramNames[i], args[i]);
        }
        Object o = expression.getValue(context);
        return o != null ? o.toString() : null;
    }

    public Object getCache(Object key, String cacheName) {
        // 获取指定命名空间的cache
        Cache cache = cacheManager.getCache(cacheName);
        if (cache == null) {
            if (log.isDebugEnabled()) {
                log.debug("------ 获取缓存: {} 失败", cacheName);
            }
            return null;
        }
        Cache.ValueWrapper wrapper = cache.get(key);
        if (wrapper == null) {
            // 空的
            return null;
        }
        return wrapper.get();
    }

    public Cache getCache(String cacheName) {
        return cacheManager.getCache(cacheName);
    }

    public void saveCache(Object o, Object key, String cacheName) {
        // 获取指定命名空间的cache
        Cache cache = this.getCache(cacheName);
        if (cache == null) {
            log.warn("获得cacheName失败, {}", cacheName);
            return;
        }
        // 加入缓存
        cache.put(key, o);
        try {
            // 获取redis的配置
            RedisCacheConfiguration configuration = redisCacheManager.getCacheConfigurations().get(cacheName);
            String redisKey = configuration.getKeyPrefixFor(cacheName).concat(key.toString());
            log.info("save exception|not result redis, key:{} timeout:{}", redisKey, exceptionTimeout);
            stringRedisTemplate.expire(redisKey, exceptionTimeout, TimeUnit.SECONDS);
        } catch (Exception e) {
            e.printStackTrace();
            log.info("获取RedisCacheConfiguration失败", e);
        }
    }
}
```

**缓存配置 CacheConfig**

```

import org.springframework.beans.factory.annotation.Value;
import org.springframework.cache.CacheManager;
import org.springframework.cache.annotation.CachingConfigurerSupport;
import org.springframework.cache.annotation.EnableCaching;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.cache.RedisCacheConfiguration;
import org.springframework.data.redis.cache.RedisCacheManager;
import org.springframework.data.redis.cache.RedisCacheWriter;
import org.springframework.data.redis.core.StringRedisTemplate;

import java.time.Duration;
import java.util.Objects;

/**
 * redis 缓存设置
 * @author RootHK
 */
@Configuration
@EnableCaching
public class CacheConfig extends CachingConfigurerSupport {

    // 正常缓存的时间
    @Value("${esb.cache.result.time}")
    private Long timeout;

    /**
     * 设置缓存策略
     *
     * @param redisTemplate
     * @return
     */
    @Bean
    public CacheManager cacheManager(StringRedisTemplate redisTemplate) {
        RedisCacheConfiguration redisCacheConfiguration = RedisCacheConfiguration.defaultCacheConfig()
                .prefixCacheNameWith(SystemConstant.REDIS_PREFIX)
                .entryTtl(Duration.ofSeconds(timeout));
        RedisCacheWriter redisCacheWriter = RedisCacheWriter.nonLockingRedisCacheWriter(Objects.requireNonNull(redisTemplate.getConnectionFactory()));
        return new RedisCacheManager(redisCacheWriter, redisCacheConfiguration);
    }

}
```
