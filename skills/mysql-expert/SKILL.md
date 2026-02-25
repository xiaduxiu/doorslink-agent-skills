---
name: mysql-expert
description: MySQL数据库专家，专注于查询优化、架构设计、安全性和性能。在编写SQL、创建迁移、设计架构或排查数据库性能问题时使用。融合MySQL最佳实践。
---

# MySQL 数据库专家

适用于 Java Spring Boot + MyBatis + MySQL 项目的数据库开发。

## 索引设计

### 索引基本原则
- 为主键和唯一约束自动创建索引
- 为 WHERE 条件、JOIN 条件、ORDER BY 字段创建索引
- 避免过多索引（写性能开销）
- 使用复合索引遵循最左前缀原则

### 复合索引示例
```sql
-- 常用查询: WHERE status = 'active' AND created_at > '2024-01-01'
CREATE INDEX idx_status_created ON orders(status, created_at);
```

## 查询优化

### 避免全表扫描
```sql
-- 避免
SELECT * FROM orders WHERE YEAR(created_at) = 2024;

-- 使用范围查询
SELECT * FROM orders WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01';
```

### 分页优化
```sql
-- 避免 OFFSET 过大（深度分页）
-- 低效
SELECT * FROM orders LIMIT 100000, 20;

-- 使用游标分页
SELECT * FROM orders WHERE id > #{lastId} ORDER BY id LIMIT 20;
```

### JOIN 优化
```sql
-- 小表驱动大表
SELECT /*+ STRAIGHT_JOIN */ ...
FROM small_table s
INNER JOIN large_table l ON s.id = l.small_id;
```

## MyBatis 最佳实践

### Mapper XML
```xml
<select id="findByCondition" resultMap="orderMap">
  SELECT o.*, u.name as user_name
  FROM orders o
  LEFT JOIN users u ON o.user_id = u.id
  <where>
    <if test="status != null">
      AND o.status = #{status}
    </if>
    <if test="startDate != null">
      AND o.created_at >= #{startDate}
    </if>
  </where>
  ORDER BY o.created_at DESC
  <if test="offset != null">
    OFFSET #{offset}
  </if>
  LIMIT #{limit}
</select>
```

### 批量操作
```java
@Insert("<script>" +
  "INSERT INTO order_items (order_id, product_id, quantity) VALUES " +
  "<foreach collection='items' item='item' separator=','>" +
  "(#{item.orderId}, #{item.productId}, #{item.quantity})" +
  "</foreach>" +
  "</script>")
void batchInsert(@Param("items") List<OrderItem> items);
```

## 事务管理

### 传播行为
```java
@Transactional(propagation = Propagation.REQUIRED)  // 默认，有则加入，无则创建
@Transactional(propagation = Propagation.REQUIRES_NEW)  // 总是新建事务
@Transactional(propagation = Propagation.SUPPORTS)  // 有则加入，无则非事务
@Transactional(propagation = Propagation.NOT_SUPPORTED)  // 非事务执行
```

### 隔离级别
```java
@Transactional(isolation = Isolation.READ_COMMITTED)  // 避免脏读
@Transactional(isolation = Isolation.REPEATABLE_READ)  // 默认，避免幻读
```

## 数据库安全

### 防止 SQL 注入
- 使用参数化查询，不拼接 SQL
- 严格限制用户输入

### 敏感数据
- 密码加密存储（BCrypt）
- 敏感字段加密（AES）

## 性能监控

### 慢查询日志
```sql
-- 查看慢查询
SHOW VARIABLES LIKE 'slow_query_log%';
SHOW VARIABLES LIKE 'long_query_time';

-- 分析执行计划
EXPLAIN FORMAT=JSON SELECT * FROM orders WHERE status = 'pending';
```

### 常用查询
```sql
-- 查看表大小
SELECT table_name, ROUND(data_length/1024/1024, 2) AS 'MB'
FROM information_schema.tables
WHERE table_schema = DATABASE();

-- 查看索引
SHOW INDEX FROM orders;

-- 查看锁等待
SELECT * FROM information_schema.INNODB_LOCK_WAITS;
```

## 项目数据库表结构

根据项目模块，典型表结构：

```sql
-- 用户表
CREATE TABLE user (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  username VARCHAR(50) NOT NULL UNIQUE,
  password VARCHAR(255) NOT NULL,
  phone VARCHAR(20),
  status TINYINT DEFAULT 1,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- 订单表
CREATE TABLE orders (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  order_no VARCHAR(64) NOT NULL UNIQUE,
  user_id BIGINT NOT NULL,
  total_amount DECIMAL(10,2),
  status VARCHAR(20),
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_user_id (user_id),
  INDEX idx_status (status),
  INDEX idx_created_at (created_at)
);
```
