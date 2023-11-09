# \[记录]Weblogic 12c关于@WebFilter的问题

> 最近项目在迁移weblogic(12c), 就遇到了多多少少的问题

> 这次遇到: @WebFilter不生效: 是整个不生效

```
// 源代码
@Slf4j
@ServletComponentScan
@Component
@WebFilter(filterName = "jwtFilter", urlPatterns = {"/register"})
public class JwtFilter implements Filter {

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {
        // 业务逻辑省略
        log.info("--- 日志打印 ---");
    }

}
```

> 这样启动之后, 就发现控制台完全没有相关日志打印

**踩坑过程**

**1. 搜索出来的结果主要是说需要配置web.xml: 但是@WebFilter本来就是web.xml的升级版??**

**2. 再搜索说的是urlPatterns不生效: 但我这个是压根没反应**

> 下面是不明所以的解决方法

```
    // 加init即可
    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
        
    }
```

> 有明白为什么的欢迎留言告诉我= =
