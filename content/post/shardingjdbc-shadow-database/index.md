---
title: "ShardingJDBC：通过影子库生产与测试数据隔离"
description: "在生产环境中如何安全地进行测试？本文详解ShardingJDBC影子库配置与实战应用，实现生产测试数据完美隔离"
slug: "shardingjdbc-shadow-database"
date: 2025-01-22T14:20:00+08:00
lastmod: 2025-01-22T14:20:00+08:00
draft: false
tags: ["ShardingJDBC", "影子库", "数据隔离", "生产测试"]
categories: ["中间件", "数据库"]
---

## 前言

前段时间，我们需要在生产环境进行一次压测。但问题来了：怎么在不影响线上真实用户数据的情况下，模拟真实的业务流量？

传统的做法是搭建独立的测试环境，但这样做有几个问题：
- 测试环境数据量小，无法真实反映性能
- 环境配置可能与生产不一致
- 维护成本高

最终我们选择了ShardingJDBC的影子库方案，在生产环境中实现了数据的完美隔离。今天就来分享一下这个实战经验。

## 什么是影子库

影子库（Shadow Database）是一种在生产环境中进行安全测试的技术方案。它的核心思想是：

- **生产库**：存储真实的业务数据
- **影子库**：与生产库结构完全相同，但只存储测试数据
- **智能路由**：根据请求标识，自动将测试流量路由到影子库

{{< mermaid >}}
graph TD
    A[客户端请求] --> B{流量识别}
    B -->|生产流量| C[生产数据库]
    B -->|测试流量| D[影子数据库]
    C --> E[真实业务数据]
    D --> F[测试数据]
    
    style C fill:#e1f5fe
    style D fill:#fff3e0
{{< /mermaid >}}

## ShardingJDBC影子库配置

### 1. 依赖配置

首先添加ShardingSphere-JDBC依赖：

```xml
<dependency>
    <groupId>org.apache.shardingsphere</groupId>
    <artifactId>shardingsphere-jdbc-core-spring-boot-starter</artifactId>
    <version>5.2.1</version>
</dependency>
```

### 2. 数据源配置

```yaml
# application.yml
spring:
  shardingsphere:
    datasource:
      names: master-ds,shadow-ds
      
      # 生产数据源
      master-ds:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.cj.jdbc.Driver
        jdbc-url: jdbc:mysql://prod-db:3306/ecommerce_prod
        username: ${DB_USER}
        password: ${DB_PASSWORD}
        
      # 影子数据源
      shadow-ds:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.cj.jdbc.Driver
        jdbc-url: jdbc:mysql://shadow-db:3306/ecommerce_shadow
        username: ${DB_USER}
        password: ${DB_PASSWORD}
    
    # 影子库规则配置
    rules:
      shadow:
        enable: true
        data-sources:
          shadow-data-source:
            production-data-source-name: master-ds
            shadow-data-source-name: shadow-ds
        
        # 影子表配置
        tables:
          t_order:
            data-source-names: 
              - shadow-data-source
            shadow-algorithm-names:
              - user-id-insert-match-algorithm
              - user-id-select-match-algorithm
          
          t_order_item:
            data-source-names:
              - shadow-data-source
            shadow-algorithm-names:
              - user-id-insert-match-algorithm
              - user-id-select-match-algorithm
        
        # 影子算法配置
        shadow-algorithms:
          user-id-insert-match-algorithm:
            type: VALUE_MATCH
            props:
              operation: insert
              column: user_id
              value: 0
          
          user-id-select-match-algorithm:
            type: VALUE_MATCH
            props:
              operation: select
              column: user_id
              value: 0
    
    props:
      sql-show: true
```

### 3. 核心业务代码

```java
@Service
public class OrderService {
    
    @Autowired
    private OrderMapper orderMapper;
    
    @Autowired
    private OrderItemMapper orderItemMapper;
    
    /**
     * 创建订单 - 支持影子库路由
     */
    @Transactional
    public Long createOrder(CreateOrderRequest request) {
        // 构建订单对象
        Order order = Order.builder()
                .userId(request.getUserId())
                .totalAmount(request.getTotalAmount())
                .status(OrderStatus.PENDING)
                .createTime(LocalDateTime.now())
                .build();
        
        // 插入订单 - 会根据userId自动路由到对应库
        orderMapper.insert(order);
        
        // 插入订单项
        List<OrderItem> orderItems = request.getItems().stream()
                .map(item -> OrderItem.builder()
                        .orderId(order.getId())
                        .userId(request.getUserId()) // 关键：保持userId一致
                        .productId(item.getProductId())
                        .quantity(item.getQuantity())
                        .price(item.getPrice())
                        .build())
                .collect(Collectors.toList());
        
        orderItemMapper.batchInsert(orderItems);
        
        return order.getId();
    }
    
    /**
     * 查询订单
     */
    public OrderVO getOrder(Long orderId, Long userId) {
        // 根据userId路由到对应的数据库
        Order order = orderMapper.selectByIdAndUserId(orderId, userId);
        if (order == null) {
            return null;
        }
        
        List<OrderItem> items = orderItemMapper.selectByOrderId(orderId, userId);
        
        return OrderVO.builder()
                .order(order)
                .items(items)
                .build();
    }
}
```

## 影子流量生成器

为了方便测试，我们需要一个流量生成器：

```java
@Component
public class ShadowTrafficGenerator {
    
    private static final Long SHADOW_USER_ID = 0L; // 影子用户ID
    
    @Autowired
    private OrderService orderService;
    
    /**
     * 生成影子订单数据
     */
    public void generateShadowOrders(int count) {
        for (int i = 0; i < count; i++) {
            CreateOrderRequest request = CreateOrderRequest.builder()
                    .userId(SHADOW_USER_ID) // 使用影子用户ID
                    .totalAmount(BigDecimal.valueOf(100 + i))
                    .items(generateRandomItems())
                    .build();
            
            try {
                Long orderId = orderService.createOrder(request);
                log.info("Created shadow order: {}", orderId);
            } catch (Exception e) {
                log.error("Failed to create shadow order", e);
            }
        }
    }
    
    /**
     * 查询影子数据验证
     */
    public void verifyShadowData() {
        // 查询生产数据
        OrderVO prodOrder = orderService.getOrder(1L, 1001L);
        log.info("Production order: {}", prodOrder);
        
        // 查询影子数据
        OrderVO shadowOrder = orderService.getOrder(1L, SHADOW_USER_ID);
        log.info("Shadow order: {}", shadowOrder);
    }
    
    private List<CreateOrderItemRequest> generateRandomItems() {
        return Arrays.asList(
            CreateOrderItemRequest.builder()
                .productId(1001L + new Random().nextInt(100))
                .quantity(1 + new Random().nextInt(3))
                .price(BigDecimal.valueOf(50 + new Random().nextInt(200)))
                .build()
        );
    }
}
```

## 压测控制器

```java
@RestController
@RequestMapping("/shadow")
public class ShadowTestController {
    
    @Autowired
    private ShadowTrafficGenerator trafficGenerator;
    
    @Autowired
    private OrderService orderService;
    
    /**
     * 启动影子数据生成
     */
    @PostMapping("/generate/{count}")
    public ResponseEntity<String> generateShadowData(@PathVariable int count) {
        try {
            trafficGenerator.generateShadowOrders(count);
            return ResponseEntity.ok("Generated " + count + " shadow orders");
        } catch (Exception e) {
            return ResponseEntity.badRequest().body("Error: " + e.getMessage());
        }
    }
    
    /**
     * 模拟并发压测
     */
    @PostMapping("/stress-test")
    public ResponseEntity<String> stressTest() {
        CompletableFuture.runAsync(() -> {
            // 模拟高并发场景
            for (int i = 0; i < 1000; i++) {
                CreateOrderRequest request = CreateOrderRequest.builder()
                        .userId(0L) // 影子用户
                        .totalAmount(BigDecimal.valueOf(199.99))
                        .items(Arrays.asList(
                            CreateOrderItemRequest.builder()
                                .productId(2001L)
                                .quantity(1)
                                .price(BigDecimal.valueOf(199.99))
                                .build()
                        ))
                        .build();
                
                orderService.createOrder(request);
            }
        });
        
        return ResponseEntity.ok("Stress test started");
    }
    
    /**
     * 数据隔离验证
     */
    @GetMapping("/verify")
    public ResponseEntity<Map<String, Object>> verifyIsolation() {
        Map<String, Object> result = new HashMap<>();
        
        // 查询生产数据数量（user_id != 0）
        // 查询影子数据数量（user_id = 0）
        // 这里简化实现
        
        result.put("production_orders", "真实订单数据");
        result.put("shadow_orders", "影子订单数据");
        result.put("isolation_status", "完全隔离");
        
        return ResponseEntity.ok(result);
    }
}
```

## 影子库监控

为了确保影子库正常工作，我们需要添加监控：

```java
@Component
@Slf4j
public class ShadowDatabaseMonitor {
    
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    /**
     * 监控数据隔离情况
     */
    @Scheduled(fixedRate = 60000) // 每分钟检查一次
    public void monitorDataIsolation() {
        try {
            // 检查生产库是否有影子数据泄漏
            String prodCheckSql = """
                SELECT COUNT(*) FROM t_order 
                WHERE user_id = 0
                """;
            
            Integer shadowDataInProd = jdbcTemplate.queryForObject(
                prodCheckSql, Integer.class);
            
            if (shadowDataInProd > 0) {
                log.error("ALERT: Found {} shadow records in production database!", 
                         shadowDataInProd);
                // 发送告警
                alertService.sendAlert("Shadow data leaked to production!");
            }
            
            // 统计影子库数据量
            String shadowCountSql = """
                SELECT COUNT(*) FROM t_order 
                WHERE user_id = 0
                """;
            
            Integer shadowCount = jdbcTemplate.queryForObject(
                shadowCountSql, Integer.class);
            
            log.info("Shadow database health check - Shadow records: {}", shadowCount);
            
        } catch (Exception e) {
            log.error("Shadow database monitoring failed", e);
        }
    }
}
```

## 影子数据清理策略

```java
@Service
public class ShadowDataCleanupService {
    
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    /**
     * 清理过期的影子数据
     */
    @Scheduled(cron = "0 0 2 * * ?") // 每天凌晨2点执行
    public void cleanupExpiredShadowData() {
        log.info("Starting shadow data cleanup...");
        
        try {
            // 清理7天前的影子订单数据
            String cleanupSql = """
                DELETE FROM t_order 
                WHERE user_id = 0 
                AND create_time < DATE_SUB(NOW(), INTERVAL 7 DAY)
                """;
            
            int deletedCount = jdbcTemplate.update(cleanupSql);
            log.info("Cleaned up {} expired shadow orders", deletedCount);
            
            // 清理对应的订单项
            String cleanupItemsSql = """
                DELETE FROM t_order_item 
                WHERE user_id = 0 
                AND create_time < DATE_SUB(NOW(), INTERVAL 7 DAY)
                """;
            
            int deletedItemsCount = jdbcTemplate.update(cleanupItemsSql);
            log.info("Cleaned up {} expired shadow order items", deletedItemsCount);
            
        } catch (Exception e) {
            log.error("Shadow data cleanup failed", e);
        }
    }
    
    /**
     * 手动清理所有影子数据
     */
    public void cleanupAllShadowData() {
        log.warn("Manual cleanup of ALL shadow data requested");
        
        try {
            jdbcTemplate.update("DELETE FROM t_order_item WHERE user_id = 0");
            jdbcTemplate.update("DELETE FROM t_order WHERE user_id = 0");
            
            log.info("All shadow data cleaned up successfully");
        } catch (Exception e) {
            log.error("Manual shadow data cleanup failed", e);
        }
    }
}
```

## 影子库路由原理

ShardingJDBC影子库的路由原理：

{{< mermaid >}}
sequenceDiagram
    participant App as 应用程序
    participant Proxy as ShardingJDBC代理
    participant ProdDB as 生产数据库
    participant ShadowDB as 影子数据库
    
    App->>Proxy: INSERT INTO t_order (user_id=0, ...)
    Proxy->>Proxy: 解析SQL，检查user_id值
    Proxy->>Proxy: 匹配影子算法规则
    Proxy->>ShadowDB: 路由到影子库执行
    ShadowDB-->>Proxy: 执行结果
    Proxy-->>App: 返回结果
    
    App->>Proxy: INSERT INTO t_order (user_id=1001, ...)
    Proxy->>Proxy: 解析SQL，检查user_id值
    Proxy->>Proxy: 不匹配影子规则
    Proxy->>ProdDB: 路由到生产库执行
    ProdDB-->>Proxy: 执行结果
    Proxy-->>App: 返回结果
{{< /mermaid >}}

## 注意事项和最佳实践

### 1. 影子标识选择

```java
// 推荐做法：使用特殊的标识值
public class ShadowConstants {
    // 使用不可能出现的业务值
    public static final Long SHADOW_USER_ID = 0L;
    public static final String SHADOW_PREFIX = "shadow_";
    
    // 或者使用范围标识
    public static final Long SHADOW_USER_ID_START = 999000000L;
    public static final Long SHADOW_USER_ID_END = 999999999L;
    
    public static boolean isShadowUser(Long userId) {
        return userId >= SHADOW_USER_ID_START && userId <= SHADOW_USER_ID_END;
    }
}
```

### 2. 事务处理

```java
@Transactional
public void complexBusinessOperation(Long userId) {
    // 确保同一事务中的所有操作都路由到同一类型的数据库
    orderService.createOrder(userId);
    paymentService.createPayment(userId); // 必须使用相同的userId
    inventoryService.decreaseStock(userId);
}
```

### 3. 监控告警

```java
@Component
public class ShadowAlertService {
    
    /**
     * 数据泄漏告警
     */
    public void checkDataLeakage() {
        // 检查生产库是否有影子数据
        // 检查影子库是否有生产数据
        
        if (hasDataLeakage()) {
            // 立即停止影子流量
            shutdownShadowTraffic();
            // 发送紧急告警
            sendEmergencyAlert();
        }
    }
}
```

### 4. 性能调优

```yaml
# 连接池配置优化
spring:
  shardingsphere:
    datasource:
      shadow-ds:
        hikari:
          maximum-pool-size: 20
          minimum-idle: 5
          connection-timeout: 30000
          idle-timeout: 600000
          max-lifetime: 1800000
```

## 总结


使用影子库需要注意：
- 选择合适的影子标识策略
- 建立完善的监控告警机制
- 定期清理影子数据
- 确保所有相关表都正确配置
