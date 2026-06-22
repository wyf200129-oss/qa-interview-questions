# SQL / 数据库面试题

---

## Q1. 左查询、右查询、内查询和子查询的语句格式是怎样的？

### 前置准备：测试中常用的几张表

为了方便理解，先假设我们有这几张表和示例数据：

```sql
-- 订单表
CREATE TABLE sale_order (
    id INT PRIMARY KEY,
    order_no VARCHAR(32),          -- 订单号
    user_id INT,                   -- 用户ID
    amount DECIMAL(10,2),          -- 订单金额
    status VARCHAR(16),            -- CREATED / PAID / CANCELED
    created_at DATETIME
);

-- 出库单表
CREATE TABLE wms_outbound (
    id INT PRIMARY KEY,
    order_id INT,                  -- 关联订单ID
    sku_code VARCHAR(32),          -- 商品编码
    quantity INT,                  -- 出库数量
    status VARCHAR(16),            -- PENDING / AUDITED / DONE
    created_at DATETIME
);

-- 示例数据
-- sale_order:
-- 1 | ORD001 | 101 | 100.00 | PAID    | 2026-06-21
-- 2 | ORD002 | 102 | 200.00 | PAID    | 2026-06-21
-- 3 | ORD003 | 103 | 300.00 | CANCELED| 2026-06-21
-- 4 | ORD004 | 101 | 400.00 | CREATED | 2026-06-21
--
-- wms_outbound:
-- 1 | 1 | SKU001 | 10 | DONE    | 2026-06-21
-- 2 | 1 | SKU002 | 20 | DONE    | 2026-06-21
-- 3 | 2 | SKU003 | 30 | AUDITED | 2026-06-21
```

---

### 一、LEFT JOIN（左连接）

**格式**：

```sql
SELECT 列
FROM 左表 A
LEFT JOIN 右表 B ON A.关键字段 = B.关键字段
```

**含义**：**以左表为主，左表的记录全部保留。右表有匹配的则显示数据，没有匹配的则显示 NULL。**

**测试场景**：查出所有订单及其出库信息，包括那些还没出库的订单。

```sql
SELECT
    o.id AS 订单ID,
    o.order_no AS 订单号,
    o.amount AS 金额,
    o.status AS 订单状态,
    w.sku_code AS 出库商品,
    w.quantity AS 出库数量
FROM sale_order o
LEFT JOIN wms_outbound w ON o.id = w.order_id;
```

**结果**：

| 订单ID | 订单号 | 金额 | 订单状态 | 出库商品 | 出库数量 |
|:---:|:---:|:---:|:---:|:---:|:---:|
| 1 | ORD001 | 100.00 | PAID | SKU001 | 10 |
| 1 | ORD001 | 100.00 | PAID | SKU002 | 20 |
| 2 | ORD002 | 200.00 | PAID | SKU003 | 30 |
| 3 | ORD003 | 300.00 | CANCELED | NULL | NULL |
| 4 | ORD004 | 400.00 | CREATED | NULL | NULL |

**经典场景：找"已付款但未出库"的异常数据**

```sql
SELECT o.id, o.order_no, o.amount, o.status
FROM sale_order o
LEFT JOIN wms_outbound w ON o.id = w.order_id
WHERE o.status = 'PAID'
  AND w.id IS NULL;
```

---

### 二、RIGHT JOIN（右连接）

**格式**：

```sql
SELECT 列
FROM 左表 A
RIGHT JOIN 右表 B ON A.关键字段 = B.关键字段
```

**含义**：**以右表为主，右表的记录全部保留。左表有匹配的则显示数据，没有匹配的则显示 NULL。**

**实际上 RIGHT JOIN 用得很少**，因为把表顺序换一下就变成了 LEFT JOIN。**面试建议**：

> **"RIGHT JOIN 实战中很少用，我一般用 LEFT JOIN 解决所有问题，把主表放左边就行了。"**

---

### 三、INNER JOIN（内连接 / 内查询）

**格式**：

```sql
SELECT 列
FROM 左表 A
INNER JOIN 右表 B ON A.关键字段 = B.关键字段
```

（简写也可以省略 `INNER`，直接写 `JOIN`）

**含义**：**只返回左右两表都有匹配的记录。有一方没有匹配的就不显示。**

> **LEFT JOIN 是"左表全部要"**，**INNER JOIN 是"两边都有的才要"**——这是本质区别。

**测试场景：查所有已生成出库单的订单**（不需要知道哪些订单没出库）

```sql
SELECT
    o.order_no,
    o.amount,
    w.sku_code,
    w.quantity,
    w.status
FROM sale_order o
INNER JOIN wms_outbound w ON o.id = w.order_id;
```

**结果**：

| order_no | amount | sku_code | quantity | status |
|:---:|:---:|:---:|:---:|:---:|
| ORD001 | 100.00 | SKU001 | 10 | DONE |
| ORD001 | 100.00 | SKU002 | 20 | DONE |
| ORD002 | 200.00 | SKU003 | 30 | AUDITED |

只返回了订单 1 和 2（有出库记录的），订单 3 和 4 因为没出库记录被过滤掉了。

**三种 JOIN 的对比（一目了然）**：

```sql
-- LEFT JOIN：左表全部
SELECT COUNT(*) FROM sale_order o LEFT JOIN wms_outbound w ON o.id = w.order_id;
-- 返回 5 行（订单1有2条出库，订单2有1条，订单3和4无出库）

-- INNER JOIN：两边都有
SELECT COUNT(*) FROM sale_order o INNER JOIN wms_outbound w ON o.id = w.order_id;
-- 返回 3 行（只有订单1和2有出库记录）

-- 本质区别图：
-- LEFT JOIN:
--   sale_order:   [1] [2] [3] [4]
--   wms_outbound: [1] [1] [2]          ← 订单3和4对应NULL
--   结果:  [1] [1] [2] [3-NULL] [4-NULL]

-- INNER JOIN:
--   sale_order:   [1] [2] [3] [4]
--   wms_outbound: [1] [1] [2]
--   结果:  [1] [1] [2]                ← 只有匹配的
```

**常用场景速查**：

| 场景 | 用什么 JOIN | 为什么 |
|:---:|:---|:---|
| 查所有订单及出库信息（含未出库的） | **LEFT JOIN** | 左表（订单）全部保留 |
| 查有出库记录的订单 | **INNER JOIN** | 只要两边都有的 |
| 查没有出库记录的订单 | **LEFT JOIN + WHERE IS NULL** | 左表保留，筛出右表为空的 |
| 订单金额 vs 出库金额对账 | **LEFT JOIN** | 发现两边不一致的异常 |

---

### 四、子查询（Subquery）

子查询就是**嵌套在另一个 SQL 语句中的 SELECT 查询**，可以放在 WHERE、FROM、SELECT 三个位置。

#### 4.1 WHERE 子查询

**格式**：

```sql
SELECT 列 FROM 表
WHERE 列 运算符 (SELECT 列 FROM 表 WHERE 条件);
```

**场景 1：查出金额高于平均值的订单**

```sql
SELECT order_no, amount
FROM sale_order
WHERE amount > (
    SELECT AVG(amount) FROM sale_order
);
```

**场景 2：查出有出库记录的订单（比 JOIN 更简洁）**

```sql
SELECT id, order_no, amount, status
FROM sale_order
WHERE id IN (
    SELECT DISTINCT order_id FROM wms_outbound
);
```

**场景 3：查出没有出库记录的订单（对账用）**

```sql
SELECT id, order_no, amount, status
FROM sale_order
WHERE id NOT IN (
    SELECT DISTINCT order_id FROM wms_outbound
    WHERE order_id IS NOT NULL   -- 防 NULL 陷阱
);
```

#### 4.2 FROM 子查询（派生表）

**格式**：

```sql
SELECT 列
FROM (SELECT 列 FROM 表 WHERE 条件) AS 别名
WHERE 条件;
```

**场景：查出"每个用户金额最高的订单"**

```sql
SELECT user_id, order_no, amount
FROM (
    SELECT
        user_id, order_no, amount,
        ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY amount DESC) AS rn
    FROM sale_order
) AS t
WHERE t.rn = 1;
```

#### 4.3 SELECT 子查询（标量子查询）

**格式**：

```sql
SELECT 列,
    (SELECT 聚合 FROM 表 WHERE 条件) AS 别名
FROM 表;
```

**场景：查出每个订单的金额 + 该用户的总消费金额**

```sql
SELECT
    order_no,
    amount,
    (SELECT SUM(s2.amount) FROM sale_order s2 WHERE s2.user_id = s1.user_id) AS user_total
FROM sale_order s1;
```

---

### 五、面试一句话总结

| 查询类型 | 一句话记法 |
|:---:|:---|
| **LEFT JOIN** | 左表全部要，右表有就填，没有就 NULL |
| **RIGHT JOIN** | 跟 LEFT JOIN 一回事，表换个方向就行，实战基本不用 |
| **INNER JOIN** | 两边都有的才要，一边没有就过滤掉 |
| **子查询** | 一个 SQL 里面再套一个 SQL，放 WHERE/FROM/SELECT 都行 |

---

**关键词**：LEFT JOIN / RIGHT JOIN / INNER JOIN / 子查询 / 对账 / 数据一致性
