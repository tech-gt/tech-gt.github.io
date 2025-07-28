---
title: "手把手教你写一个Mybatis插件，实现多租户无感切换"
date: 2023-05-20T11:00:00+08:00
categories: ["Mybatis", "后端架构"]
tags: ["Mybatis插件", "多租户", "SaaS", "数据库设计"]
description: "通过开发一个自定义的Mybatis插件，优雅地实现SaaS应用中的多租户数据隔离，让业务代码无感知。"
---

在SaaS（软件即服务）应用的开发中，多租户是一个绕不开的核心概念。简单来说，就是一套系统要服务于多个不同的客户（租户），并且要保证他们之间的数据是严格隔离的。最常见的数据隔离方案就是在业务表中增加一个`tenant_id`字段。

但问题随之而来：我们总不能在每个SQL查询后面都手动加上`WHERE tenant_id = ?`吧？这不仅工作量巨大，容易遗漏，而且对业务代码有很强的侵入性。

今天，我们就来探讨如何利用Mybatis强大的插件（Interceptor）机制，开发一个通用的多租户插件，实现对业务代码的“无感”数据隔离。

## Mybatis插件机制简介

Mybatis允许我们在SQL执行过程的特定时机点进行拦截和干预。它提供了四个可以拦截的核心对象：

1.  `Executor`: SQL执行器，负责SQL的动态拼装和执行。
2.  `StatementHandler`: 数据库会话处理器，负责处理JDBC语句。
3.  `ParameterHandler`: 参数处理器，负责将用户参数映射到JDBC的PreparedStatement。
4.  `ResultSetHandler`: 结果集处理器，负责将JDBC查询结果映射成Java对象。

对于多租户的场景，我们需要在SQL执行前对它进行改写，`StatementHandler`是最佳的拦截点。

## 多租户插件实现步骤

我们的目标是自动为SQL语句（SELECT, UPDATE, DELETE, INSERT）添加`tenant_id`的条件。

### 1. 引入JSqlParser

直接用字符串替换或拼接的方式改写SQL非常危险，容易出错。我们引入一个专业的SQL解析库——JSqlParser。

```xml
<dependency>
    <groupId>com.github.jsqlparser</groupId>
    <artifactId>jsqlparser</artifactId>
    <version>4.6</version>
</dependency>
```

### 2. 租户ID上下文

我们需要一个地方能暂存当前请求的`tenant_id`。`ThreadLocal`是处理这种场景的利器。

```java
public class TenantContextHolder {
    private static final ThreadLocal<String> CONTEXT = new ThreadLocal<>();

    public static void setTenantId(String tenantId) {
        CONTEXT.set(tenantId);
    }

    public static String getTenantId() {
        return CONTEXT.get();
    }

    public static void clear() {
        CONTEXT.remove();
    }
}
```

通常，我们会通过一个Web Filter或Interceptor，在请求开始时从Session、Token或Header中获取租户ID并设置，在请求结束时清除。

### 3. 核心拦截器TenantInterceptor

这是我们插件的核心，代码虽然有点长，但逻辑很清晰。

```java
@Intercepts({@Signature(
    type = StatementHandler.class,
    method = "prepare",
    args = {Connection.class, Integer.class}
)})
public class TenantInterceptor implements Interceptor {

    private final List<String> ignoreTables = Arrays.asList("sys_config", "tenant_info");

    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        StatementHandler statementHandler = (StatementHandler) invocation.getTarget();
        BoundSql boundSql = statementHandler.getBoundSql();
        String originalSql = boundSql.getSql();
        String tenantId = TenantContextHolder.getTenantId();

        // 如果没有获取到租户ID，则不处理
        if (tenantId == null || tenantId.trim().isEmpty()) {
            return invocation.proceed();
        }

        try {
            // 使用JSqlParser解析SQL
            Statement statement = CCJSqlParserUtil.parse(originalSql);

            if (statement instanceof Select) {
                processSelect((Select) statement, tenantId);
            } else if (statement instanceof Update) {
                processUpdate((Update) statement, tenantId);
            } else if (statement instanceof Delete) {
                processDelete((Delete) statement, tenantId);
            } else if (statement instanceof Insert) {
                 processInsert((Insert) statement, tenantId);
            }
            
            // 将改写后的SQL重新设置回去
            MetaObject metaObject = SystemMetaObject.forObject(statementHandler);
            metaObject.setValue("delegate.boundSql.sql", statement.toString());

        } catch (Exception e) {
            // 解析失败或处理异常，可以选择直接放行或抛出异常
            // 这里选择放行，避免影响正常业务
        }

        return invocation.proceed();
    }
    
    // 省略 processSelect, processUpdate, processDelete, processInsert 的具体实现
    // 核心思想是获取到WHERE条件，并用AND拼接 tenant_id = 'xxx'
    // 对于INSERT，则是在列中增加 tenant_id，在VALUES中增加对应的租户ID
}
```

**处理SELECT的示例**

```java
private void processSelect(Select select, String tenantId) {
    SelectBody selectBody = select.getSelectBody();
    if (selectBody instanceof PlainSelect) {
        PlainSelect plainSelect = (PlainSelect) selectBody;
        // 忽略特定表
        if (ignoreTables.contains(plainSelect.getFromItem().toString().toLowerCase())) {
            return;
        }
        addWhereCondition(plainSelect, tenantId);
    }
}

private void addWhereCondition(PlainSelect plainSelect, String tenantId) {
    Expression where = plainSelect.getWhere();
    String tenantFilter = "tenant_id = '" + tenantId + "'";
    
    if (where == null) {
        try {
            plainSelect.setWhere(CCJSqlParserUtil.parseCondExpression(tenantFilter));
        } catch (JSQLParserException e) {
            // handle exception
        }
    } else {
        AndExpression and = new AndExpression(where, new StringValue(tenantFilter));
        plainSelect.setWhere(and);
    }
}
```

### 4. 注册插件

最后一步，在Mybatis的配置中注册我们的拦截器。

```java
@Configuration
public class MybatisPlusConfig {
    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
        // 添加我们自定义的租户拦截器
        interceptor.addInnerInterceptor(new TenantInterceptor()); 
        // 还可以添加分页插件等
        interceptor.addInnerInterceptor(new PaginationInnerInterceptor(DbType.MYSQL));
        return interceptor;
    }
}
```
*注意：Mybatis-Plus的`TenantLineInnerInterceptor`提供了更完善的官方实现，这里是为了演示插件开发原理。若使用Mybatis-Plus，建议优先使用官方方案。*

## 总结

通过自定义Mybatis插件，我们成功地将多租户的数据隔离逻辑从业务代码中剥离出来，实现了对开发人员透明的无感切换。这种方式不仅优雅，而且易于维护和扩展。

当然，这种方案也有一些需要注意的地方：
- **SQL解析开销**：JSqlParser会带来微小的性能开销，但通常可以忽略不计。
- **复杂SQL兼容性**：对于极其复杂的SQL（如多层嵌套、JOIN等），需要充分测试以保证解析的正确性。
- **忽略表配置**：需要维护一个不需要加租户ID的白名单表。