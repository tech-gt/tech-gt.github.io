---
title: "MyBatisPlus 别只做 CRUD，试试自定义 SQL 注入器"
date: 2022-07-15T10:23:00+08:00
categories: ["Java", "MyBatisPlus"]
tags: ["MyBatisPlus"]
---

> 你还在用 MyBatisPlus 只做增删改查（CRUD）？其实它还能帮你写自定义 SQL

## 场景：CRUD 不够用怎么办？

日常开发中，MyBatisPlus 的 CRUD 方法确实很香，但总有些业务场景，标准的 `selectById`、`updateById` 根本搞不定。比如：

- 复杂的多表联查
- 批量更新某些字段
- 统计分析类 SQL

这时候，很多人会直接写 XML 或者在 Mapper 里加注解 SQL。但其实，MyBatisPlus 提供了更优雅的方式——自定义 SQL 注入器！

## 什么是 SQL 注入器？

简单说，就是你可以把自己的 SQL 方法“注入”到 MyBatisPlus 的基础 Mapper 里，像用内置方法一样用自定义 SQL。

## 实战：自定义一个批量逻辑删除方法

假设有个需求：批量逻辑删除用户（把 `deleted` 字段设为 1）。

### 1. 定义自定义方法接口

```java
public interface MyBaseMapper<T> extends BaseMapper<T> {
    int deleteBatchByIds(@Param("ids") List<Long> ids);
}
```

### 2. 编写 SQL 注入器

```java
public class DeleteBatchByIdsMethod extends AbstractMethod {
    @Override
    public MappedStatement injectMappedStatement(Class<?> mapperClass, Class<?> modelClass, TableInfo tableInfo) {
        String sql = "UPDATE " + tableInfo.getTableName() + " SET deleted=1 WHERE id IN (#{ids})";
        SqlSource sqlSource = languageDriver.createSqlSource(configuration, sql, modelClass);
        return this.addUpdateMappedStatement(mapperClass, modelClass, "deleteBatchByIds", sqlSource);
    }
}
```

### 3. 注册自定义注入器

```java
public class MySqlInjector extends DefaultSqlInjector {
    @Override
    public List<AbstractMethod> getMethodList(Class<?> mapperClass, TableInfo tableInfo) {
        List<AbstractMethod> methodList = super.getMethodList(mapperClass, tableInfo);
        methodList.add(new DeleteBatchByIdsMethod());
        return methodList;
    }
}
```

### 4. 配置到 MyBatisPlus

```java
@Configuration
public class MybatisPlusConfig {
    @Bean
    public ISqlInjector sqlInjector() {
        return new MySqlInjector();
    }
}
```

### 5. 直接用！

```java
@Autowired
private MyBaseMapper<User> userMapper;

userMapper.deleteBatchByIds(Arrays.asList(1L, 2L, 3L));
```

## 总结

- MyBatisPlus 不止 CRUD，自定义 SQL 注入器让你优雅扩展
- 代码复用高，团队协作更顺畅
- 以后遇到复杂 SQL，别再写一堆 XML 了，注入器安排上！

