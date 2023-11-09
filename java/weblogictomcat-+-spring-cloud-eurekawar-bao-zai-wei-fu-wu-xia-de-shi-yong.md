# \[Weblogic/Tomcat + Spring Cloud Eureka]war包在微服务下的使用

> 按道理，微服务应该jar包+docker/k8s的方式运行，但总有这么个操蛋时候需要通过war包运行 那这时候小葵花妈妈课堂开课啦～ 踩过的坑就要为后面的铺平道路

**Eureka Client配置更改**

```
# Tomcat/Weblogic的服务端口
server.port=8080
# 注册的服务名，这个要和你的war包名称一致
# 比如文件是demo.war,那么这里就是demo
wl.name=demo
# 应用名称
spring.application.name=${wl.name}
# 指定服务的请求路径前缀
server.servlet.context-path=/${wl.name}
eureka.instance.prefer-ip-address=true
# 设置eureka server的注册地址
# 需要注意的是，后面的‘/eureka/eureka/’，
# 这里前面的/eureka/是固定写法，后面的‘eureka/’是eureka server部署在容器里面的context-path
eureka.client.service-url.defaultZone=http://10.1.24.227:${server.port}/eureka/eureka/
eureka.instance.hostname=${wl.name}
# 微服务路径前缀
eureka.instance.home-page-url-path=/${wl.name}
# 设置eureka client的地址
eureka.instance.instance-id=${spring.cloud.client.ip-address}:${server.port}/${wl.name}
#心跳时间/服务续约间隔时间(默认30s)
eureka.instance.lease-renewal-interval-in-seconds=10
#发呆（无心跳淘汰时间）时间/服务续约到期时间(默认90s)
eureka.instance.lease-expiration-duration-in-seconds=20
#健康检查
eureka.client.healthcheck.enabled=true
```

**Feign 配置**

```
@FeignClient(
    # 服务名 spring.application.name
    name = "demo",
    # 服务路径前缀 server.servlet.context-path
    path = "/demo")
public interface DemoFeign {
    // xxxx实际业务
}
```

> 重要提醒，如果你的war包没有设置context-root，那么你的war包名称就要和spring.application.name保持一致 这样就可以正常被访问了 下课～
