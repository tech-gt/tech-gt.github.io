---
title: "MySQL索引一定要遵循最左匹配原则吗？"
date: 2023-08-12T10:00:00+08:00
categories: ["数据库", "MySQL"]
tags: ["MySQL", "索引", "最左匹配", "性能优化", "Index Skip Scan"]
description: "最左匹配原则是MySQL联合索引的经典说法，但实际应用中有很多细节和误区。本文结合实际案例，深入解析MySQL索引的最左匹配原则及其例外，特别是MySQL 8的Index Skip Scan。"
---

> “最左匹配原则”是MySQL联合索引的经典说法，但它真的绝对吗？实际开发中有哪些容易被忽略的细节？本文结合实际案例，带你深入理解MySQL索引的最左匹配原则，并关注MySQL 8的新特性——Index Skip Scan。

## 什么是最左匹配原则？

假设有如下联合索引：

```sql
index(A, B, C)
```

所谓“最左匹配原则”，指的是：**查询条件必须从索引的最左前缀开始连续匹配，索引才能被有效利用**。比如：

- `where A=1` 可以用到索引
- `where A=1 and B=2` 可以用到索引
- `where A=1 and B=2 and C=3` 可以用到索引
- 但 `where B=2 and C=3` 通常**不能**用到索引

## 常见误区：最左匹配不是“必须”！

网上流行的说法是“必须最左”，但实际情况要复杂得多。来看下面的例子：

### 误区一：跳过最左字段就一定不能用索引？

假设有如下SQL：

```sql
select * from student where B='GT' and C='男'
```

很多人会说：因为没有A，所以不满足最左匹配，**不能用索引**。但实际上，MySQL的优化器有时可以“跳跃”使用索引，尤其是在**某些字段被等值查询**且索引统计信息允许的情况下。

### 例外情况：索引下推与条件重写

如果你的查询条件中，虽然没有A，但B和C的选择性很高，MySQL可能会选择用索引扫描（Index Condition Pushdown, ICP），但通常效率不如从最左字段开始。

更常见的做法是**重写SQL**，让索引能被利用。例如：

```sql
select * from student where (A=1 and B='GT' and C='男')
   or (A=2 and B='GT' and C='男')
   or (A=3 and B='GT' and C='男')
```

这样，虽然没有直接写A=？，但通过“拆分”条件，依然可以让MySQL用到索引。

## MySQL 8 新特性：Index Skip Scan 打破最左匹配原则

MySQL 8.0 引入了 **Index Skip Scan（索引跳跃扫描）**，它让“最左匹配原则”不再是绝对的。

### Index Skip Scan 原理

当你有联合索引 `index(A, B, C)`，但查询条件没有包含最左的A字段时，MySQL 8 可以自动遍历A的所有可能值，对每个A值分别用B、C做索引查找，相当于自动帮你做了“条件拆分”。

**举例：**

```sql
select * from student where B='GT' and C='男'
```

在MySQL 8.0之前，这种写法通常不会用到索引。但开启 Index Skip Scan 后，MySQL 会自动遍历A的所有取值，分别查找B和C。

### 适用场景

- 联合索引的最左字段基数（distinct值）较少时效果较好（如A只有1、2、3）
- 查询条件跳过了最左字段，但后续字段有高选择性
- 适合数据量不是特别大的表，或最左字段枚举值有限的场景

### 注意事项

- Index Skip Scan 并不总是最优，尤其是最左字段基数很大时，遍历代价高，可能还不如全表扫描
- 是否启用可通过 `optimizer_switch='index_skip_scan=on'` 控制
- 实际是否用到，可以通过 `EXPLAIN` 查看执行计划，type 字段会显示 `index_skip_scan`

### EXPLAIN 示例

```sql
EXPLAIN SELECT * FROM student WHERE B='GT' AND C='男';
```

输出示例：

| id | select_type | table   | type            | key           | key_len | ref  | rows | Extra           |
|----|-------------|---------|-----------------|---------------|---------|------|------|----------------|
| 1  | SIMPLE      | student | index_skip_scan | index_a_b_c   | ...     | NULL | ...  | Using where     |

可以看到 type 字段为 `index_skip_scan`，说明 MySQL 自动使用了索引跳跃扫描。

## 实际案例分析

假设有如下表结构：

| 字段 | 含义   |
|------|--------|
| A    | 年级   |
| B    | 学生名 |
| C    | 性别   |

联合索引：`index(A, B, C)`

### 查询1

```sql
select * from student where b='GT' and c='男'
```
**MySQL 8 之前**：不满足最左匹配，不能用索引。

**MySQL 8 及以后**：可能会用到 index_skip_scan。

### 查询2

```sql
select * from student where (A=1 and b='GT' and c='男')
   or (A=2 and b='GT' and c='男')
   or (A=3 and b='GT' and c='男')
```
**优化后**：通过枚举A的所有可能值，满足了最左匹配，索引可以被利用。

## 总结与建议

- **最左匹配原则是MySQL索引的基础，但在MySQL 8以后已不是绝对规则**。
- **Index Skip Scan** 让跳过最左字段也有机会用到索引，但要关注最左字段基数和实际执行计划。
- 如果查询条件中跳过了最左字段，建议用 `EXPLAIN` 检查是否用到了 index_skip_scan。
- 实际开发中，优先让查询条件覆盖索引的最左字段，或通过IN/OR等方式补齐。
- 使用`EXPLAIN`分析SQL执行计划，判断索引是否被正确利用。