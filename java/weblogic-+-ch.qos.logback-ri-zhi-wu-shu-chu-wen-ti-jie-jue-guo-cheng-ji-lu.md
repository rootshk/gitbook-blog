# \[Weblogic + ch.qos.logback]日志无输出问题解决过程记录

> 最近出现一个问题，部署在weblogic的war包，启动后，除了Spring的标示日志，就没有其他日志。 百思不得其解，就开始排查
>
> 1. 开始排查了logback.xml，以为是配置问题 -> 使用可正常打印的war包的logback.xml配置：排除这个问题
> 2. 后面排查weblogic.xml，以为是包优先加载的问题 -> 加入对应包设置，依旧无效：排除问题
> 3. 开始怀疑是maven包引用问题，排查pom.xml 发现包里有一个

```
<dependency>
    <groupId>ch.qos.logback</groupId>
    <artifactId>logback-classic</artifactId>
    <version>1.2.3</version>
</dependency>
```

> 取消引入，解决
