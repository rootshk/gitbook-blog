# \[Java] \[Spring Boot] \[@Async] \[JPA]异步下数据刷新的问题复盘

#### 出现问题

> org.springframework.dao.InvalidDataAccessApiUsageException: Executing an update/delete query; nested exception is javax.persistence.TransactionRequiredException: Executing an update/delete query

**问题原因: 缺少@Transactional注解 (解决方案: 下面二选一即可)**

**1. 相对Repository加入**

```
@Repository
@Transactional
public interface MessagePluginSendParamRepository extends JpaRepository<Bean, Long> {
    // xxxx
}
```

**2. 相对应的Service加入**

```
@Service
@Transactional
public class Service {
    // xxxx
}
```

#### 出现问题

> 在父级线程设置了对象值

```
// Class1
// 查询
A a = repository.findById(1L).orEles(null);
// 设置了B
a.setB("B");
// 保存
repository.save(a);
// Async调用方法
async(a.getId());

// Class2
@Async
public void async(Long id) {
    A a = repository.findById(id).orEles(null);
    // 这里获取的不是B
    a.getB();
}
```

**问题原因: JPA缓存导致**

**1. 对象保存时使用.saveAndFlush(S s);**

```
repository.save(a); -> repository.saveAndFlush(a);
```

**2. 设置Transactional**

```
@Transactional -> @Transactional(readOnly = false)
```

**3. 修改Repository的Update方法**

```
@Modifying(clearAutomatically=true)
```

> 以上方法都会清除JPA的一级缓存
