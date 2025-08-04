---
title: "从8秒到3秒：一次SQL批量更新的性能优化"
date: 2022-09-18T15:20:00+08:00
categories:
  - 数据库
  - 性能优化
tags:
  - MySQL
  - 慢查询
  - explain
  - 索引优化
  - 批量更新
description: "通过一个出库系统慢查询优化案例，深入分析MySQL执行计划的type类型，以及如何通过索引优化将批量更新性能提升60%以上。"
---

## 问题背景

我们有一个仓库管理系统，其中有个出库的业务场景。数据库中存储着一条条的库存明细记录，当商品出库后，需要批量更新这些记录的状态为"已出库"。

### 数据表结构

```sql
CREATE TABLE `inventory_detail` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  `product_id` varchar(50) NOT NULL COMMENT '商品ID',
  `warehouse_id` varchar(50) NOT NULL COMMENT '仓库ID',
  `quantity` int(11) NOT NULL COMMENT '数量',
  `status` tinyint(4) NOT NULL DEFAULT '0' COMMENT '状态：0-在库，1-已出库',
  `batch_no` varchar(50) DEFAULT NULL COMMENT '批次号',
  `create_time` datetime NOT NULL COMMENT '创建时间',
  `update_time` datetime NOT NULL COMMENT '更新时间',
  PRIMARY KEY (`id`),
  KEY `idx_product_warehouse` (`product_id`, `warehouse_id`),
  KEY `idx_status` (`status`),
  KEY `idx_create_time` (`create_time`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### 原始的慢查询

最初的实现是这样的：

```sql
-- 1. 先查询出需要出库的记录ID
SELECT id FROM inventory_detail 
WHERE product_id = 'PROD001' 
  AND warehouse_id = 'WH001' 
  AND status = 0 
LIMIT 10000;

-- 2. 批量更新状态
UPDATE inventory_detail 
SET status = 1, update_time = NOW() 
WHERE id IN (1001, 1002, 1003, ..., 11000);
```

当需要批量出库1万条记录时，这个UPDATE语句竟然耗时8秒！

## 执行计划分析

遇到慢查询，第一步就是用`EXPLAIN`来分析执行计划：

```sql
EXPLAIN UPDATE inventory_detail 
SET status = 1, update_time = NOW() 
WHERE id IN (1001, 1002, 1003, ..., 11000);
```

执行结果：

```
+----+-------------+------------------+-------+---------------+---------+---------+------+------+-------------+
| id | select_type | table            | type  | possible_keys | key     | key_len | ref  | rows | Extra       |
+----+-------------+------------------+-------+---------------+---------+---------+------+------+-------------+
|  1 | UPDATE      | inventory_detail | range | PRIMARY       | PRIMARY | 8       | NULL | 5000 | Using where |
+----+-------------+------------------+-------+---------------+---------+---------+------+------+-------------+
```

### 理解执行计划的type字段

这里的关键是`type`字段，它表示MySQL访问表的方式。性能从好到差的顺序是：

1. **system** - 表只有一行记录（系统表），这是const类型的特例
2. **const** - 通过索引一次就找到了，用于比较primary key或unique索引
3. **eq_ref** - 唯一性索引扫描，对于每个索引键，表中只有一条记录与之匹配
4. **ref** - 非唯一性索引扫描，返回匹配某个单独值的所有行
5. **range** - 只检索给定范围的行，使用一个索引来选择行
6. **index** - 全索引扫描，只遍历索引树
7. **ALL** - 全表扫描，性能最差

我们的查询type是`range`，虽然不是最差的，但在处理1万条IN查询时，性能还是不够理想。
EXPLAIN 结果中的 type 字段显示为 `range`。`range` 意味着MySQL正在使用索引进行范围扫描。虽然 `range` 级别不算差，但为什么在这里会这么慢？
IN 操作的本质：当 IN 列表包含大量值时，MySQL会将其视为一系列的等值查找。对于我们的10000个ID，数据库需要进行10000次独立的索引定位（Index Seek）。
这就像让一位图书管理员去书库里找10000本编码完全不连续的书，虽然他每次找书都很快（走了索引），但这来来回回的奔波和查找次数累加起来，总耗时就变得非常可观。

## 优化思路

分析了执行计划后，我开始思考优化方案。既然IN查询的性能不够好，能不能换个思路？

观察数据特点，我发现：
1. 表使用自增主键ID
2. 出库时通常是按时间顺序，也就是ID较小的记录先出库
3. 1万条记录的ID基本是连续的

基于这个特点，我想到了一个优化方案：

```sql
-- 1. 查询出需要出库的最大ID
SELECT MAX(id) as max_id FROM (
    SELECT id FROM inventory_detail 
    WHERE product_id = 'PROD001' 
      AND warehouse_id = 'WH001' 
      AND status = 0 
    ORDER BY id 
    LIMIT 10000
) t;

-- 2. 直接更新ID小于等于max_id的记录
UPDATE inventory_detail 
SET status = 1, update_time = NOW() 
WHERE product_id = 'PROD001' 
  AND warehouse_id = 'WH001' 
  AND status = 0 
  AND id <= @max_id
LIMIT 10000;
```

## 优化后的执行计划

让我们看看优化后的执行计划：

```sql
EXPLAIN UPDATE inventory_detail 
SET status = 1, update_time = NOW() 
WHERE product_id = 'PROD001' 
  AND warehouse_id = 'WH001' 
  AND status = 0 
  AND id <= 11000
LIMIT 10000;
```

执行结果：

```
+----+-------------+------------------+-------------+---------------------------+---------------------------+---------+------+------+------------------------------------+
| id | select_type | table            | type        | possible_keys             | key                       | key_len | ref  | rows | Extra                              |
+----+-------------+------------------+-------------+---------------------------+---------------------------+---------+------+------+------------------------------------+
|  1 | UPDATE      | inventory_detail | index_merge | PRIMARY,idx_product_warehouse,idx_status | idx_product_warehouse,idx_status | 206,1   | NULL | 3200 | Using intersect(idx_product_warehouse,idx_status); Using where |
+----+-------------+------------------+-------------+---------------------------+---------------------------+---------+------+------+------------------------------------+
```

可以看到，type变成了`index_merge`，这意味着MySQL使用了索引合并优化，同时使用了`idx_product_warehouse`和`idx_status`两个索引，然后取交集。

## 代码实现

在Java代码中，优化后的实现是这样的：

```java
@Service
public class InventoryService {
    
    @Autowired
    private InventoryDetailMapper inventoryDetailMapper;
    
    /**
     * 批量出库优化版本
     */
    @Transactional
    public int batchOutbound(String productId, String warehouseId, int quantity) {
        // 1. 查询需要出库的最大ID
        Long maxId = inventoryDetailMapper.findMaxIdForOutbound(productId, warehouseId, quantity);
        
        if (maxId == null) {
            return 0;
        }
        
        // 2. 批量更新状态
        return inventoryDetailMapper.batchUpdateStatusOptimized(productId, warehouseId, maxId, quantity);
    }
}
```

Mapper接口：

```java
@Mapper
public interface InventoryDetailMapper {
    
    @Select("SELECT MAX(id) FROM (" +
            "SELECT id FROM inventory_detail " +
            "WHERE product_id = #{productId} " +
            "AND warehouse_id = #{warehouseId} " +
            "AND status = 0 " +
            "ORDER BY id LIMIT #{quantity}" +
            ") t")
    Long findMaxIdForOutbound(@Param("productId") String productId, 
                             @Param("warehouseId") String warehouseId, 
                             @Param("quantity") int quantity);
    
    @Update("UPDATE inventory_detail " +
            "SET status = 1, update_time = NOW() " +
            "WHERE product_id = #{productId} " +
            "AND warehouse_id = #{warehouseId} " +
            "AND status = 0 " +
            "AND id <= #{maxId} " +
            "LIMIT #{quantity}")
    int batchUpdateStatusOptimized(@Param("productId") String productId,
                                  @Param("warehouseId") String warehouseId,
                                  @Param("maxId") Long maxId,
                                  @Param("quantity") int quantity);
}
```

## 性能对比

优化前后的性能对比：

| 方案 | 执行时间 | type类型 | 使用索引 | 性能提升 |
|------|----------|----------|----------|----------|
| 原始IN查询 | 8秒 | range | PRIMARY | - |
| 优化后范围查询 | 3.2秒 | index_merge | idx_product_warehouse + idx_status | 60% |

## 优化原理分析

为什么优化后性能提升这么明显？主要原因有几个：

### 1. 索引使用更高效

原始方案使用IN查询，MySQL需要对每个ID值进行查找，虽然走的是主键索引，但1万个值的查找开销还是很大。

优化后使用范围查询配合复合条件，MySQL可以同时利用多个索引进行index_merge优化。

### 2. 减少了回表操作

```sql
-- 原始方案需要先查询ID，再用IN更新
SELECT id FROM inventory_detail WHERE ...; -- 第一次查询
UPDATE inventory_detail WHERE id IN (...); -- 第二次查询

-- 优化方案只需要一次更新操作
UPDATE inventory_detail WHERE ... AND id <= max_id;
```

### 3. 利用了数据的局部性

由于ID是自增的，需要出库的记录ID基本连续，范围查询比IN查询更适合这种场景。

## 进一步优化思考

虽然性能已经提升了60%，但还可以考虑进一步优化：

### 1. 分批处理

对于超大批量的更新，可以分批进行：

```java
public int batchOutboundInChunks(String productId, String warehouseId, int totalQuantity) {
    int chunkSize = 5000; // 每批5000条
    int totalUpdated = 0;
    
    while (totalUpdated < totalQuantity) {
        int currentChunk = Math.min(chunkSize, totalQuantity - totalUpdated);
        int updated = batchOutbound(productId, warehouseId, currentChunk);
        
        if (updated == 0) {
            break; // 没有更多记录可更新
        }
        
        totalUpdated += updated;
    }
    
    return totalUpdated;
}
```

### 2. 异步处理

对于非实时要求的场景，可以考虑异步处理：

```java
@Async
public CompletableFuture<Integer> asyncBatchOutbound(String productId, String warehouseId, int quantity) {
    int result = batchOutbound(productId, warehouseId, quantity);
    return CompletableFuture.completedFuture(result);
}
```

## 总结

这次慢查询优化的关键点：

1. **善用EXPLAIN**：执行计划是定位性能问题的第一步
2. **理解type类型**：不同的访问类型性能差异巨大
3. **结合业务特点**：利用数据的自然特性进行优化
4. **索引合并**：MySQL的index_merge可以同时利用多个索引
5. **减少查询次数**：能一次完成的操作不要分两次
