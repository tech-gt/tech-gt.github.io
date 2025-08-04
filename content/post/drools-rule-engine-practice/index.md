---
title: "Drools规则引擎在业务系统中的实战应用"
date: 2021-03-22T09:15:00+08:00
categories:
  - Java
  - 架构设计
tags:
  - Drools
  - 规则引擎
  - 业务规则
  - 决策引擎
description: "从实际项目出发，详细介绍Drools规则引擎的使用方法，包括基础概念、核心API、规则编写和性能优化等内容。"
---

## 引言

在开发业务系统时，我们经常会遇到复杂的业务规则判断。比如电商系统的促销规则、风控系统的风险评估、保险系统的理赔审核等。这些规则往往变化频繁，如果硬编码在代码里，每次修改都要重新发布，维护成本很高。

这时候，规则引擎就派上用场了。今天就来聊聊Drools规则引擎在实际项目中的应用。

## 什么是Drools

Drools是一个开源的业务规则管理系统（BRMS），它提供了一个核心的业务规则引擎（BRE）。简单来说，Drools可以让我们把复杂的业务逻辑从代码中抽离出来，用更直观的规则语言来表达。

### 核心概念

在使用之前，先了解几个核心概念：

- **Fact**：事实，即输入到规则引擎的数据对象
- **Rule**：规则，包含条件（when）和动作（then）
- **Working Memory**：工作内存，存放Fact的地方
- **Knowledge Base**：知识库，包含所有规则的集合
- **Session**：会话，规则执行的上下文

## 项目集成

首先在项目中引入Drools依赖：

```xml
<dependency>
    <groupId>org.drools</groupId>
    <artifactId>drools-core</artifactId>
    <version>7.74.1.Final</version>
</dependency>
<dependency>
    <groupId>org.drools</groupId>
    <artifactId>drools-compiler</artifactId>
    <version>7.74.1.Final</version>
</dependency>
<dependency>
    <groupId>org.drools</groupId>
    <artifactId>drools-decisiontables</artifactId>
    <version>7.74.1.Final</version>
</dependency>
```

## 实战案例：电商促销规则

我们以一个电商促销系统为例，来看看Drools的具体使用。

### 1. 定义业务对象

首先定义订单和用户对象：

```java
public class Order {
    private String orderId;
    private BigDecimal totalAmount;
    private String userId;
    private List<OrderItem> items;
    private BigDecimal discountAmount = BigDecimal.ZERO;
    
    // 构造函数、getter、setter省略
}

public class User {
    private String userId;
    private String level; // VIP, GOLD, SILVER, NORMAL
    private int orderCount; // 历史订单数
    private boolean isNewUser;
    
    // 构造函数、getter、setter省略
}

public class OrderItem {
    private String productId;
    private String category;
    private BigDecimal price;
    private int quantity;
    
    // 构造函数、getter、setter省略
}
```

### 2. 编写规则文件

在`src/main/resources/rules`目录下创建`promotion.drl`文件：

```drools
package com.example.promotion

import com.example.model.Order
import com.example.model.User
import com.example.model.OrderItem
import java.math.BigDecimal

// 新用户首单优惠
rule "新用户首单8折优惠"
    when
        $user: User(isNewUser == true)
        $order: Order(userId == $user.userId, totalAmount > 100)
    then
        BigDecimal discount = $order.getTotalAmount().multiply(new BigDecimal("0.2"));
        $order.setDiscountAmount(discount);
        System.out.println("应用新用户首单8折优惠，优惠金额：" + discount);
end

// VIP用户满500减100
rule "VIP用户满500减100"
    when
        $user: User(level == "VIP")
        $order: Order(userId == $user.userId, totalAmount >= 500)
    then
        $order.setDiscountAmount(new BigDecimal("100"));
        System.out.println("应用VIP满500减100优惠");
end

// 图书类商品满200减50
rule "图书满200减50"
    when
        $order: Order()
        $bookAmount: BigDecimal() from accumulate(
            OrderItem(category == "BOOK", $price: price, $qty: quantity) from $order.items,
            sum($price.multiply(new BigDecimal($qty)))
        )
        eval($bookAmount.compareTo(new BigDecimal("200")) >= 0)
    then
        BigDecimal currentDiscount = $order.getDiscountAmount();
        $order.setDiscountAmount(currentDiscount.add(new BigDecimal("50")));
        System.out.println("应用图书满200减50优惠");
end

// 老用户回购优惠
rule "老用户回购优惠"
    salience 10  // 优先级，数字越大优先级越高
    when
        $user: User(orderCount >= 10, level != "VIP")
        $order: Order(userId == $user.userId, totalAmount > 300)
    then
        BigDecimal discount = $order.getTotalAmount().multiply(new BigDecimal("0.05"));
        BigDecimal currentDiscount = $order.getDiscountAmount();
        $order.setDiscountAmount(currentDiscount.add(discount));
        System.out.println("应用老用户回购5%优惠，优惠金额：" + discount);
end
```

### 3. 规则引擎服务

创建一个规则引擎服务类：

```java
@Service
public class PromotionRuleService {
    
    private KieContainer kieContainer;
    
    @PostConstruct
    public void init() {
        KieServices kieServices = KieServices.Factory.get();
        KieFileSystem kieFileSystem = kieServices.newKieFileSystem();
        
        // 加载规则文件
        Resource resource = kieServices.getResources()
                .newClassPathResource("rules/promotion.drl");
        kieFileSystem.write(resource);
        
        // 构建知识库
        KieBuilder kieBuilder = kieServices.newKieBuilder(kieFileSystem);
        kieBuilder.buildAll();
        
        if (kieBuilder.getResults().hasMessages(Level.ERROR)) {
            throw new RuntimeException("规则编译失败：" + kieBuilder.getResults());
        }
        
        KieModule kieModule = kieBuilder.getKieModule();
        this.kieContainer = kieServices.newKieContainer(kieModule.getReleaseId());
    }
    
    public void applyPromotionRules(Order order, User user) {
        KieSession kieSession = kieContainer.newKieSession();
        
        try {
            // 插入事实
            kieSession.insert(order);
            kieSession.insert(user);
            
            // 执行规则
            int firedRules = kieSession.fireAllRules();
            System.out.println("执行了 " + firedRules + " 条规则");
            
        } finally {
            kieSession.dispose();
        }
    }
}
```

### 4. 控制器使用

在控制器中使用规则引擎：

```java
@RestController
@RequestMapping("/api/orders")
public class OrderController {
    
    @Autowired
    private PromotionRuleService promotionRuleService;
    
    @Autowired
    private UserService userService;
    
    @PostMapping("/calculate-discount")
    public ResponseEntity<OrderDiscountResponse> calculateDiscount(@RequestBody Order order) {
        // 获取用户信息
        User user = userService.getUserById(order.getUserId());
        
        // 应用促销规则
        promotionRuleService.applyPromotionRules(order, user);
        
        // 返回结果
        OrderDiscountResponse response = new OrderDiscountResponse();
        response.setOrderId(order.getOrderId());
        response.setOriginalAmount(order.getTotalAmount());
        response.setDiscountAmount(order.getDiscountAmount());
        response.setFinalAmount(order.getTotalAmount().subtract(order.getDiscountAmount()));
        
        return ResponseEntity.ok(response);
    }
}
```

## 高级特性

### 1. 规则优先级

使用`salience`属性可以控制规则执行顺序：

```drools
rule "高优先级规则"
    salience 100
    when
        // 条件
    then
        // 动作
end
```

### 2. 规则组

使用`agenda-group`可以将规则分组：

```drools
rule "VIP专享规则"
    agenda-group "vip-promotion"
    when
        // 条件
    then
        // 动作
end
```

在代码中激活特定组：

```java
kieSession.getAgenda().getAgendaGroup("vip-promotion").setFocus();
kieSession.fireAllRules();
```

### 3. 全局变量

可以在规则中使用全局变量：

```drools
global java.util.List promotionLogs;

rule "记录促销日志"
    when
        $order: Order(discountAmount > 0)
    then
        promotionLogs.add("订单 " + $order.getOrderId() + " 享受优惠 " + $order.getDiscountAmount());
end
```

在代码中设置全局变量：

```java
List<String> logs = new ArrayList<>();
kieSession.setGlobal("promotionLogs", logs);
```

## 性能优化

### 1. 会话复用

对于频繁调用的场景，可以考虑复用KieSession：

```java
@Component
public class RuleSessionPool {
    
    private final Queue<KieSession> sessionPool = new ConcurrentLinkedQueue<>();
    private final KieContainer kieContainer;
    
    public RuleSessionPool(KieContainer kieContainer) {
        this.kieContainer = kieContainer;
    }
    
    public KieSession borrowSession() {
        KieSession session = sessionPool.poll();
        if (session == null) {
            session = kieContainer.newKieSession();
        }
        return session;
    }
    
    public void returnSession(KieSession session) {
        // 清理会话状态
        session.getFactHandles().forEach(session::delete);
        sessionPool.offer(session);
    }
}
```

### 2. 规则编译缓存

将编译好的规则缓存起来，避免重复编译：

```java
@Configuration
public class DroolsConfig {
    
    @Bean
    @Scope("singleton")
    public KieContainer kieContainer() {
        KieServices kieServices = KieServices.Factory.get();
        KieContainer kieContainer = kieServices.getKieClasspathContainer();
        return kieContainer;
    }
}
```

## 规则管理

在生产环境中，我们通常需要动态管理规则。可以考虑以下方案：

### 1. 数据库存储规则

```java
@Entity
public class BusinessRule {
    @Id
    private String ruleId;
    private String ruleName;
    private String ruleContent;
    private boolean enabled;
    private Date createTime;
    private Date updateTime;
    
    // getter、setter省略
}
```

### 2. 动态加载规则

```java
public void reloadRules() {
    List<BusinessRule> rules = businessRuleRepository.findByEnabled(true);
    
    KieServices kieServices = KieServices.Factory.get();
    KieFileSystem kieFileSystem = kieServices.newKieFileSystem();
    
    for (BusinessRule rule : rules) {
        String resourcePath = "src/main/resources/rules/" + rule.getRuleId() + ".drl";
        kieFileSystem.write(resourcePath, rule.getRuleContent());
    }
    
    KieBuilder kieBuilder = kieServices.newKieBuilder(kieFileSystem);
    kieBuilder.buildAll();
    
    if (!kieBuilder.getResults().hasMessages(Level.ERROR)) {
        KieModule kieModule = kieBuilder.getKieModule();
        this.kieContainer = kieServices.newKieContainer(kieModule.getReleaseId());
    }
}
```

## 最佳实践

基于实际使用经验，总结几个最佳实践：

### 1. 规则设计原则
- **单一职责**：每个规则只处理一个业务逻辑
- **避免循环**：注意规则之间的依赖关系，避免无限循环
- **合理分组**：使用agenda-group对规则进行分类管理

### 2. 性能考虑
- **减少事实数量**：只插入必要的事实对象
- **优化规则条件**：将最容易失败的条件放在前面
- **使用索引**：对于大量数据，考虑在Fact对象上建立索引

### 3. 测试策略

```java
@Test
public void testPromotionRules() {
    // 准备测试数据
    User vipUser = new User("user1", "VIP", 5, false);
    Order order = new Order("order1", new BigDecimal("600"), "user1");
    
    // 执行规则
    promotionRuleService.applyPromotionRules(order, vipUser);
    
    // 验证结果
    assertEquals(new BigDecimal("100"), order.getDiscountAmount());
}
```

当然，Drools也不是万能的。对于简单的业务逻辑，直接用代码实现可能更简单。但对于复杂多变的业务规则，Drools确实是一个很好的选择。

在实际项目中，建议从简单的规则开始，逐步积累经验，然后再处理更复杂的场景。同时要注意规则的版本管理和测试，确保系统的稳定性。