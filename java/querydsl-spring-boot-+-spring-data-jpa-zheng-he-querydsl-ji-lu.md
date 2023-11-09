# \[QueryDSL] Spring Boot + Spring Data Jpa整合QueryDSL记录

```
# 依赖版本
OpenJDK 17
Spring Boot 2.6.6
MySQL 8
QueryDSL 5.0
```

### Maven POM

```
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
        <scope>runtime</scope>
    </dependency>
    <dependency>
        <groupId>com.querydsl</groupId>
        <artifactId>querydsl-apt</artifactId>
        <scope>provided</scope>
    </dependency>
    <dependency>
        <groupId>com.querydsl</groupId>
        <artifactId>querydsl-jpa</artifactId>
    </dependency>
</dependencies>
<build>
    <plugins>
        <plugin>
            <groupId>com.mysema.maven</groupId>
            <artifactId>apt-maven-plugin</artifactId>
            <version>1.1.3</version>
            <executions>
                <execution>
                    <goals>
                        <goal>process</goal>
                    </goals>
                    <configuration>
                        <outputDirectory>target/generated-sources/java</outputDirectory>
                        <processor>com.querydsl.apt.jpa.JPAAnnotationProcessor</processor>
                    </configuration>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

### Config

```
import com.querydsl.jpa.impl.JPAQueryFactory;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import javax.persistence.EntityManager;

@Configuration
public class JpaConfig {

    @Bean
    public JPAQueryFactory jpaQuery(EntityManager entityManager) {
        return new JPAQueryFactory(entityManager);
    }

}
```

### BaseService.java

```
package top.roothk.mall.service;

import com.querydsl.core.types.OrderSpecifier;
import com.querydsl.core.types.Predicate;
import com.querydsl.core.types.QBean;
import com.querydsl.core.types.dsl.EntityPathBase;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.domain.Sort;
import top.roothk.mall.PageRequest;
import top.roothk.mall.entity.BaseEntity;

import java.io.Serializable;
import java.util.List;

/**
 * Service - 基类
 *
 * @param <T>  实体
 * @param <ID> id
 * @author RootHK
 */
public interface BaseService<T extends BaseEntity<ID>, ID extends Serializable> {

    EntityPathBase<T> getQueryDslEntity();

    /**
     * 查找实体对象
     *
     * @param id ID
     * @return 实体对象，若不存在则返回null
     */
    T find(ID id);

    T find(Predicate p);

    T find(List<Predicate> p);

    /**
     * 查找实体对象集合
     *
     * @param ids ID
     * @return 实体对象集合
     */
    List<T> findList(List<ID> ids);

    List<T> findList(Predicate p);

    List<T> findList(Predicate p, OrderSpecifier s);

    List<T> findList(List<Predicate> p, List<OrderSpecifier> s);

    /**
     * 查找实体对象集合
     */
    List<T> findList(Sort sort);

    /**
     * 查找所有实体对象集合
     *
     * @return 所有实体对象集合
     */
    List<T> findList();

    /**
     * 查找实体对象分页
     *
     * @param pageable 分页信息
     * @return 实体对象分页
     */
    Page<T> page(Pageable pageable);

    /**
     * 查找实体对象分页
     *
     * @param r 分页信息
     * @return 实体对象分页
     */
    top.roothk.mall.Page<T> page(PageRequest r);

    <Y> top.roothk.mall.Page<Y> page(QBean<Y> select, PageRequest r, Predicate p, OrderSpecifier s);

    top.roothk.mall.Page<T> page(PageRequest r, Predicate p, OrderSpecifier s);

    top.roothk.mall.Page<T> page(PageRequest r, List<Predicate> p, List<OrderSpecifier> s);

    /**
     * 查询实体对象总数
     *
     * @return 实体对象总数
     */
    long count();

    /**
     * 判断实体对象是否存在
     *
     * @param id ID
     * @return 实体对象是否存在
     */
    boolean exists(ID id);

    /**
     * 判断实体对象是否存在
     *
     * @param p 条件
     * @return 实体对象是否存在
     */
    boolean exists(Predicate p);

    /**
     * 保存实体对象
     *
     * @param entity 实体对象
     * @return 实体对象
     */
    T save(T entity);

    /**
     * 删除实体对象
     *
     * @param id ID
     */
    void delete(ID id);

    /**
     * 删除实体对象
     *
     * @param ids ID
     */
    void delete(List<ID> ids);

    /**
     * 删除实体对象
     *
     * @param entity 实体对象
     */
    void delete(T entity);

    /**
     * 删除实体对象
     *
     * @param entities 实体对象
     */
    void deleteEntity(List<T> entities);

    /**
     * 落库
     */
    void flush();

}
```

### BaseServiceImpl.java

```
package top.roothk.mall.service.impl;

import com.querydsl.core.types.OrderSpecifier;
import com.querydsl.core.types.Predicate;
import com.querydsl.core.types.QBean;
import com.querydsl.jpa.impl.JPAQuery;
import com.querydsl.jpa.impl.JPAQueryFactory;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.BeanUtils;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.domain.Sort;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.util.Assert;
import top.roothk.mall.PageRequest;
import top.roothk.mall.entity.BaseEntity;
import top.roothk.mall.service.BaseService;

import javax.annotation.Resource;
import java.io.Serializable;
import java.util.Collections;
import java.util.List;

/**
 * Service - 基类
 *
 * @author RootHK
 * @version 5.0
 */
@Slf4j
@Transactional(rollbackFor = Exception.class)
public abstract class BaseServiceImpl<T extends BaseEntity<ID>, ID extends Serializable> implements BaseService<T, ID> {

    @Resource
    protected JPAQueryFactory queryFactory;

    @Autowired
    protected JpaRepository<T, ID> repository;

    @Override
    public T find(ID id) {
        return repository.findById(id).orElse(null);
    }

    @Override
    public T find(Predicate p) {
        return find(p == null ? null : Collections.singletonList(p));
    }

    @Override
    public T find(List<Predicate> p) {
        JPAQuery<T> q = queryFactory.selectFrom(getQueryDslEntity());
        if (p != null && p.size() > 0) {
            q.where(p.toArray(new Predicate[0]));
        }
        return q.fetchFirst();
    }

    @Override
    public List<T> findList(List<ID> ids) {
        return repository.findAllById(ids);
    }

    @Override
    public List<T> findList(Predicate p) {
        return findList(p, null);
    }

    @Override
    public List<T> findList(Predicate p, OrderSpecifier s) {
        return findList(p == null ? null : Collections.singletonList(p),
                s == null ? null : Collections.singletonList(s));
    }

    @Override
    public List<T> findList(List<Predicate> p, List<OrderSpecifier> s) {
        JPAQuery<T> q = queryFactory.selectFrom(getQueryDslEntity());
        if (p != null && p.size() > 0) {
            q.where(p.toArray(new Predicate[0]));
        }
        if (s != null && s.size() > 0) {
            q.orderBy(s.toArray(new OrderSpecifier[0]));
        }
        return q.fetch();
    }

    @Override
    public List<T> findList(Sort sort) {
        return repository.findAll(sort);
    }

    @Override
    public List<T> findList() {
        return repository.findAll();
    }

    @Override
    public Page<T> page(Pageable pageable) {
        return repository.findAll(pageable);
    }

    @Override
    public top.roothk.mall.Page<T> page(PageRequest r) {
        return new top.roothk.mall.Page<>(page(r.pageRequest()));
    }

    @Override
    public <Y> top.roothk.mall.Page<Y> page(QBean<Y> select, PageRequest r, Predicate p, OrderSpecifier s) {
        Integer page = r.getPage();
        Integer size = r.getSize();
        JPAQuery<Y> q = queryFactory.select(select).from(getQueryDslEntity());
        JPAQuery<Long> qc = queryFactory.select(getQueryDslEntity().count());
        if (p != null) {
            q.where(p);
            qc.where(p);
        }
        List<Long> c = qc.from(getQueryDslEntity()).fetch();
        Long allCount = c.get(0);
        if (allCount <= 0) {
            return top.roothk.mall.Page.empty(r);
        }
        if (s != null) {
            q.orderBy(s);
        }
        List<Y> l;
        //分页查询
        if (page != null && size != null) {
            l = q.offset((long) (page) * size)
                    .limit(size)
                    .fetch();
        } else {
            l = q.fetch();
        }
        return new top.roothk.mall.Page<>(r, allCount, l);
    }

    @Override
    public top.roothk.mall.Page<T> page(PageRequest r, Predicate p, OrderSpecifier s) {
        return page(r,
                p == null ? null : Collections.singletonList(p),
                s == null ? null : Collections.singletonList(s));
    }

    @Override
    public top.roothk.mall.Page<T> page(PageRequest r, List<Predicate> p, List<OrderSpecifier> s) {
        Integer page = r.getPage();
        Integer size = r.getSize();
        JPAQuery<T> q = queryFactory.selectFrom(getQueryDslEntity());
        JPAQuery<Long> qc = queryFactory.select(getQueryDslEntity().count());
        if (p != null && p.size() > 0) {
            q.where(p.toArray(new Predicate[0]));
            qc.where(p.toArray(new Predicate[0]));
        }
        List<Long> c = qc.from(getQueryDslEntity()).fetch();
        Long allCount = c.get(0);
        if (allCount <= 0) {
            return top.roothk.mall.Page.empty(r);
        }
        if (s != null && s.size() > 0) {
            q.orderBy(s.toArray(new OrderSpecifier[0]));
        }
        List<T> l;
        //分页查询
        if (page != null && size != null) {
            l = q.offset((long) (page) * size)
                    .limit(size)
                    .fetch();
        } else {
            l = q.fetch();
        }
        return new top.roothk.mall.Page<>(r, allCount, l);
    }

    @Override
    public long count() {
        return repository.count();
    }

    @Override
    public boolean exists(ID id) {
        return repository.existsById(id);
    }

    @Override
    public boolean exists(Predicate p) {
        Integer exists = queryFactory.selectOne().from(getQueryDslEntity()).where(p).fetchFirst();
        return exists != null && exists > 0;
    }

    @Override
    public T save(T entity) {
        if (entity.getId() != null) {
            T t = find(entity.getId());
            BeanUtils.copyProperties(entity, t);
            return repository.save(t);
        }
        return repository.save(entity);
    }

    @Override
    public void delete(ID id) {
        repository.deleteById(id);
    }

    @Override
    public void delete(List<ID> ids) {
        repository.deleteAllById(ids);
    }

    @Override
    public void deleteEntity(List<T> entities) {
        Assert.notNull(entities, "删除对象为空");
        repository.deleteAllInBatch(entities);
    }

    @Override
    public void delete(T entity) {
        repository.delete(entity);
    }

    @Override
    public void flush() {
        repository.flush();
    }

}
```

### AccountService.class

```
package top.roothk.mall.service;

import top.roothk.mall.entity.Account;

public interface AccountService extends BaseService<Account, Long> {}
```

### AccountServiceImpl.class

```
package top.roothk.mall.service.impl;

import com.querydsl.core.types.dsl.EntityPathBase;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import top.roothk.mall.entity.Account;
import top.roothk.mall.repository.AccountRepository;
import top.roothk.mall.service.AccountService;

import javax.annotation.Resource;
import java.util.List;
import java.util.Map;

@Slf4j
@Service
@Transactional(rollbackFor = Exception.class)
public class AccountServiceImpl extends BaseServiceImpl<Account, Long> implements AccountService {

    @Override
    public EntityPathBase<Account> getQueryDslEntity() {
        return Q;
    }

}
```

### 使用

```
// 通过id查询
public String get(Long id) {
    return find(Q.accountId.eq(id)).getName();
}

```
