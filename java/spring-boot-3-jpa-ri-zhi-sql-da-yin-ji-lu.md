# \[Spring Boot 3] JPA 日志SQL打印记录

> 网上找的JPA SQL打印参数都是加一个BasicBinder

```yaml
logging:
  level:
    org.hibernate.SQL: debug
    org.hibernate.type.descriptor.jdbc.BasicBinder: trace
```

> 然而我在Spring Boot 3.0.1上测试是没效的。然后我们找到BasicBinder.class

![截屏2022-12-28 15.09.05.png](https://roothk.top/usr/uploads/2022/12/2389458025.png)

> 我们可以看到确实有binding parameter的字样内容，但是没有地方使用到。但是查看到bind()方法里有一行日志打印调用

![截屏2022-12-28 15.10.53.png](https://roothk.top/usr/uploads/2022/12/1430653885.png)

> 这里使用了JdbcBindingLogging.TRACE\_ENABLED作为判断，我们再找到这个JdbcBindingLogging类

![截屏2022-12-28 15.12.00.png](https://roothk.top/usr/uploads/2022/12/1228890676.png)

> 可以看到日志级别LOGGER.tracef() 以及@SubSystemLogging的name是org.hibernate.orm.jdbc.bind 那么我们可以这样配置日志输出

```yaml
spring:
  jpa:
    properties:
      hibernate:
        # 格式化SQL
        format_sql: true
logging:
  level:
    # 用于打印SQL
    org.hibernate.SQL: debug
    # 用于打印参数
    org.hibernate.orm.jdbc.bind: trace
```

> 以下是具体的日志打印输出

![截屏2022-12-28 15.16.11.png](https://roothk.top/usr/uploads/2022/12/3253811806.png)
