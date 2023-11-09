# Spring Boot 2读取jar包地resource文件夹下面的资源



> 面向百度变成老是告诉我用ResourceUtils.getFile(), 打成jar包就会报错

&#x20;**用下面的就可以了**&#x20;

```
// 需要Apache Common包 
// 文件夹初始 public static final String BASE_PATH = "db/"; 
// 对应文件转String 
public static String getString(String name) { 
    ClassPathResource c = new ClassPathResource(BASE_PATH.concat(name)); 
    InputStream i = c.getInputStream(); return IOUtils.toString(i, CHAR); 
}
```

