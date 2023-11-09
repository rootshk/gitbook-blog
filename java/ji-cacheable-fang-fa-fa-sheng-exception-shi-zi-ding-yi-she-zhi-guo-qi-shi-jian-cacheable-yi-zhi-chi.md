# \[记@Cacheable方法发生Exception时自定义设置过期时间]@Cacheable以支持Exception时的保存

> 事情是这样的, 有这么一个需求: 使用@Cacheable注解, 要求方法正常返回时保存x时间, 非正常返回时保存y时间 然后我就四处找解决方法啊, 什么@Cacheable.unless啊, RedisCacheManager啊, RedisCacheConfiguration啊找了一遍 最后还是用了AOP的方式解决了, 下面是实际代码, 大家可以参考一下

```
// 实际需要缓存的方法入口, 在方法类不使用@CacheConfig
@Cacheable(key = "#s", cacheNames = "account.getStr")
public String getStr(String s) {
    // ...省略方法具体实现, 这里可能会throw Exception
    return s;
}
```

> 这里需要说明为什么不使用@CacheConfig, 因为我懒, 你完全可以把cacheNames参数放到类上面 只是你在AOP的时候就要多操作一下

```
// redis配置
@Configuration
@EnableCaching
public class CacheConfig extends CachingConfigurerSupport {

    // 缓存的前缀
    private static final String REDIS_PREFIX = "cache:";
    // 默认的缓存有效期时长(秒)
    private static final Long TIME_OUT = 3600L;

    @Autowired
    Environment env;

    /**
     * 设置缓存策略
     *
     * @param redisTemplate
     * @return
     */
    @Bean
    public CacheManager cacheManager(StringRedisTemplate redisTemplate) {
        RedisCacheConfiguration redisCacheConfiguration = RedisCacheConfiguration.defaultCacheConfig()
                // 前缀设置
                .prefixCacheNameWith(REDIS_PREFIX)
                // 有效期
                .entryTtl(Duration.ofSeconds(TIME_OUT));
        RedisCacheWriter redisCacheWriter = 
            RedisCacheWriter.nonLockingRedisCacheWriter(
                Objects.requireNonNull(redisTemplate.getConnectionFactory())
            );
        return new RedisCacheManager(redisCacheWriter, redisCacheConfiguration);
    }
}
```

> 上面的有效期可以使用@Value来设置, 写死完全是因为懒

```
// AOP代码
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
 * @date 2020/8/23 8:53
 */
@Slf4j
@Aspect
@Component
public class CacheableHandlerAspect {

    // 错误时的缓存时间
    @Value("${system.cache.exception.timeout}")
    private Long exceptionTimeout;
    @Autowired
    private CacheManager cacheManager;
    @Autowired
    private RedisCacheManager redisCacheManager;
    @Autowired
    private StringRedisTemplate stringRedisTemplate;

    // 用于解析Sl语法
    private final SpelExpressionParser parserSpel = new SpelExpressionParser();
    private final DefaultParameterNameDiscoverer parameterNameDiscoverer = new DefaultParameterNameDiscoverer();

    @Pointcut("@annotation(org.springframework.cache.annotation.Cacheable)")
    public void feignAspectPointCut() {
    }

    /**
     * 环绕通知 @Around, 当然也可以使用 @Before (前置通知)  @After (后置通知)
     *
     * @param point
     * @return
     * @throws Throwable
     */
    @Around("feignAspectPointCut()")
    public Object around(ProceedingJoinPoint point) throws Throwable {
        // 获取方法Cacheable的相关can'shu
        Cacheable cacheable = getCacheable(point);
        String cacheName = cacheable.cacheNames()[0];
        String key = getValueByKey(cacheable.key(), point);
        
        try {
            // 程序直接执行
            return point.proceed();
        } catch (Exception e) {
            // 如果出错就进行报错的保存
            saveCache(e, key, cacheName);
            throw e;
        }
    }

    private void handlerException(Object key, String cacheName) {
        Object o = getCache(key, cacheName);
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

    private void saveCache(Object o, Object key, String cacheName) {
        // 获取指定命名空间的cache
        Cache cache = cacheManager.getCache(cacheName);
        // 为空就不操作, 程序设置有问题
        if (cache == null) {
            log.debug("------ 获取缓存: {} 失败", cacheName);
            return;
        }
        // 加入缓存
        cache.put(key, o);
        try {
            // 获取redis的配置
            log.debug("------ 准备获取RedisCacheConfiguration");
            RedisCacheConfiguration configuration = redisCacheManager.getCacheConfigurations().get(cacheName);
            log.debug("------ 准备拼接缓存key");
            String redisKey = configuration.getKeyPrefixFor(cacheName).concat(key.toString());
            log.debug("------ 拼接成功, 缓存key: {}, 准备设置reis有效时间: {}秒", redisKey, exceptionTimeout);
            stringRedisTemplate.expire(redisKey, exceptionTimeout, TimeUnit.SECONDS);
            log.debug("------ 设置失败时的缓存时间");
        } catch (Exception e) {
            e.printStackTrace();
            log.info("获取RedisCacheConfiguration失败", e);
        }
    }

    /**
      * 获取方法上的 @Cacheable
      *
      * @param point
      * @return
      */
     private Cacheable getCacheable(ProceedingJoinPoint point) {
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
    private String getValueByKey(String key, ProceedingJoinPoint pjp) {
        Expression expression = parserSpel.parseExpression(key);
        EvaluationContext context = new StandardEvaluationContext();
        MethodSignature methodSignature = (MethodSignature) pjp.getSignature();
        Object[] args = pjp.getArgs();
        String[] paramNames = parameterNameDiscoverer.getParameterNames(methodSignature.getMethod());
        for (int i = 0; i < args.length; i++) {
            context.setVariable(paramNames[i], args[i]);
        }
        return expression.getValue(context).toString();
    }

}
```
