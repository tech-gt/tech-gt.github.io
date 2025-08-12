---
title: "RabbitMQ如何实现事务消息？"
date: 2024-01-18T16:20:00+08:00
categories: ["分布式系统", "消息队列"]
tags: ["RabbitMQ", "事务消息", "分布式事务", "最终一致性"]
---

在构建分布式系统时，我们经常面临一个棘手的问题：如何保证跨服务的操作要么都成功，要么都失败？举个最经典的例子：电商系统中的订单服务创建了订单后，需要通知库存服务扣减库存。如果订单创建成功，但扣减库存的通知消息丢失了，就会导致超卖，这在业务上是不可接受的。

能不能让发送消息这个动作也加入到本地数据库事务里？ 这就是事务消息。
它的核心目标是：**如果你的数据库操作成功了，那么这条消息就必须成功发送出去；如果你的数据库操作因为任何原因回滚了，那么这条消息就绝对不能被发送出去**。

## 事务消息的实现原理 (两阶段提交)
事务消息通常采用一种“两阶段提交”(2PC - Two-Phase Commit) 的思想来实现。以支持事务消息的 **RocketMQ** 为例，其流程如下：

#### **阶段一：发送“半消息”（Half Message / Pre-committed Message）**

1.  生产者（例如订单服务）先将一条“半消息”发送到 MQ 服务器。
2.  这条“半消息”对下游的消费者是**完全不可见**的。它的作用是向 MQ 服务器“预定”一个消息位，告诉 MQ：“我准备要发一条消息了，请你先占个位置，但别让消费者看到。”
3.  MQ 服务器收到半消息后，会将其持久化，并向生产者返回一个“半消息发送成功”的确认。

#### **阶段二：执行本地事务并提交最终状态**

1.  生产者收到了“半消息发送成功”的确认后，开始执行自己的**本地数据库事务**（即创建订单的操作）。
2.  根据本地事务的执行结果，生产者会向 MQ 服务器发送一个**最终确认 (Commit / Rollback)**：
      * **如果本地事务成功提交 (Commit)：** 生产者就向 MQ 服务器发送一个 **Commit** 请求。MQ 服务器收到后，会将之前的“半消息”标记为**可投递状态**，此时消费者才能真正地拉取到这条消息。
      * **如果本地事务失败回滚 (Rollback)：** 生产者就向 MQ 服务器发送一个 **Rollback** 请求。MQ 服务器收到后，会直接**删除**之前的“半消息”。这条消息仿佛从未存在过，消费者自然也永远不会收到它。

#### **异常情况处理：事务状态回查机制**

一个棘手的问题是：如果在第二阶段，生产者执行完本地事务后，在发送最终 `Commit` 或 `Rollback` 请求时宕机了怎么办？

此时，MQ 服务器上的那条“半消息”会一直处于中间状态，既不被投递也不被删除。

为了解决这个问题，支持事务消息的 MQ 系统通常带有一个**状态回查机制**：

1.  MQ 服务器会定期（或在超时后）向消息的生产者**主动发起一个“回查”请求**。
2.  生产者的应用程序需要提供一个**回查接口**。在这个接口里，生产者需要去检查对应那条消息的本地事务的最终状态（比如去数据库里查一下那个订单号是否存在且状态是“已支付”）。
3.  根据查询到的本地事务状态，生产者在回查接口中再次告诉 MQ 服务器应该 `Commit` 还是 `Rollback` 这条半消息。




## 一、RabbitMQ原生事务的局限
RabbitMQ本身提供了事务机制。但在实际生产中，这套原生事务机制却很少被使用。

RabbitMQ通过三个核心命令提供了事务功能：
- `tx.Select`: 告诉服务器，当前Channel要开启一个事务。
- `tx.Commit`: 提交事务。
- `tx.Rollback`: 回滚事务。

使用起来大概是这样（伪代码）：
```java
try {
    // 1. 开启事务
    channel.txSelect();

    // 2. 发送消息
    channel.basicPublish("exchange", "routingKey", null, message.getBytes());

    // 模拟业务异常
    if (someCondition) {
        throw new RuntimeException("业务异常");
    }

    // 3. 提交事务
    channel.txCommit();
} catch (Exception e) {
    // 4. 回滚事务
    channel.txRollback();
}
```

这看起来似乎完美地解决了问题。如果在`txCommit()`之前发生任何异常，我们可以通过`txRollback()`来撤销所有已发送但未提交的消息。

**但它为什么会被冷落呢？**

根本原因在于**性能**。RabbitMQ的事务机制是**同步阻塞**的。当一个Channel调用`txSelect()`后，这个Channel上发布的所有消息都会在Broker端被缓存，直到`txCommit()`被调用，这些消息才会被真正地投递到目标队列。在这个过程中，Publisher会一直等待Broker的响应，这极大地降低了消息发送的吞吐量。官方文档也明确指出，使用事务会让消息吞吐量下降**2到10倍**。

对于需要高吞吐量的互联网应用来说，这种性能损失是致命的。因此，我们必须寻找替代方案。

## 二、发送方确认机制 (Publisher Confirms) 的局限

为了解决原生事务的性能问题，RabbitMQ引入了`Publisher Confirms`机制。这是一种轻量级的、异步的确认方式，用来保证消息被成功发送到了Broker。

它的核心思想是：Producer将Channel设置为Confirm模式后，后续在该Channel上发布的消息都会被分配一个唯一的ID。一旦消息被Broker正确接收，Broker就会发送一个确认（Ack）给Producer。如果消息因为Broker内部错误无法被处理，Broker则会发送一个否定确认（Nack）。

因为是异步的，Producer在发送消息后可以不必等待，继续发送下一条消息，大大提高了性能。

```java
// 开启发送方确认模式
channel.confirmSelect();

// 添加确认监听器
channel.addConfirmListener(new ConfirmListener() {
    @Override
    public void handleAck(long deliveryTag, boolean multiple) throws IOException {
        // deliveryTag: 消息的唯一ID
        // multiple: 是否是批量确认
        System.out.println("消息 " + deliveryTag + " 已被确认");
        // 在这里可以安全地认为消息已到达Broker
    }

    @Override
    public void handleNack(long deliveryTag, boolean multiple) throws IOException {
        System.out.println("消息 " + deliveryTag + " 未被确认，需要重试或记录日志");
        // 进行消息重发、记录日志等操作
    }
});

// 发送消息
channel.basicPublish("my-exchange", "routing-key", null, "Hello".getBytes());
```

`Publisher Confirms`机制解决了消息从Producer到Broker的可靠性问题，但它依然无法解决我们开篇提到的分布式事务问题——业务操作和消息发送，这两个动作如何原子化？

## 三、推荐方案：基于“本地消息表”的最终一致性

这是业界解决分布式事务最经典、最通用的模式之一，其核心思想是**将业务操作和“发送消息”这个动作捆绑在同一个本地事务中**。

{{< mermaid >}}
sequenceDiagram
    participant User as 用户
    participant OrderSvc as 订单服务
    participant OrderDB as 订单库(本地消息表)
    participant Scheduler as 定时任务
    participant RabbitMQ
    participant InventorySvc as 库存服务

    User->>+OrderSvc: 创建订单请求
    OrderSvc->>OrderDB: 开启数据库事务
    Note right of OrderSvc: 1. 在本地事务中执行业务
    OrderSvc->>OrderDB: 1.1 INSERT INTO orders ...
    OrderSvc->>OrderDB: 1.2 INSERT INTO message_log (status='PENDING') ...
    OrderSvc->>OrderDB: 提交数据库事务
    OrderSvc-->>-User: 下单成功

    loop 定时扫描
        Scheduler->>+OrderDB: 2. SELECT * FROM message_log WHERE status='PENDING'
        OrderDB-->>-Scheduler: 返回待发送消息列表
    end

    Scheduler->>+RabbitMQ: 3. 发送消息
    RabbitMQ-->>-Scheduler: 4. 返回Publisher Confirm (Ack)

    alt 收到Ack
        Scheduler->>+OrderDB: 5. UPDATE message_log SET status='SENT'
        OrderDB-->>-Scheduler: 更新成功
    else 收到Nack或超时
        Scheduler->>Scheduler: 重试或记录错误
    end

    RabbitMQ->>+InventorySvc: 6. 投递消息
    Note right of InventorySvc: 7. 消费者处理业务
    InventorySvc->>InventorySvc: 扣减库存等操作
    InventorySvc->>-RabbitMQ: 8. 发送ack (消费者确认)
{{< /mermaid >}}

整个流程分解如下：

1.  **执行本地事务**：当订单服务要创建订单时，它会开启一个本地数据库事务。在这个事务里，它不仅会向`orders`业务表插入数据，还会向一张`local_message`（本地消息表）里插入一条记录。这条消息记录了需要发送给下游的消息内容，其初始状态为“发送中”（Pending）。
2.  **事务提交**：由于业务操作和写消息日志在同一个事务里，因此它们要么都成功，要么都失败。这就保证了只要订单创建成功，要发送的消息就一定被存下来了。
3.  **定时任务投递消息**：一个独立的后台任务（或者线程池）会定期扫描`local_message`表，拉取状态为“发送中”的消息。
4.  **可靠发送**：定时任务将消息发送到RabbitMQ，并使用`Publisher Confirms`机制等待Broker的确认。
5.  **更新消息状态**：如果收到Broker的Ack，就将本地消息表中的对应记录状态更新为“已发送”（Sent）。如果收到Nack或者超时未收到确认，则可以根据策略进行重试。
6.  **消费者处理**：库存服务作为消费者，接收到消息后处理自己的业务（扣减库存）。为了防止消息处理失败导致消息丢失，消费者必须使用**手动ack模式**。只有当业务逻辑成功处理完毕后，才向RabbitMQ发送ack，此时RabbitMQ才会真正地将消息从队列中删除。

这个方案通过一张额外的日志表，巧妙地将分布式事务问题转化为了一个本地事务问题，并通过定时任务和重试机制，保证了消息最终一定能被成功发送和消费，实现了系统的**最终一致性**。

## 结语

技术选型没有绝对的好坏，只有适不适合。RabbitMQ的原生事务（`txSelect`）因其性能问题，在大部分场景下都不是一个好的选择。相比之下，`Publisher Confirms` + `手动ack` + `本地消息表`的组合拳，虽然实现上更复杂一些，但它提供了一种健壮的、高性能的、可扩展的分布式事务解决方案，是构建可靠微服务系统的事实标准之一。