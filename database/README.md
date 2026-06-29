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

---

## Q2. 多表查询在项目中用到哪些场景？结合ERP项目具体举例

### 问题描述

简历写了"熟悉MySQL多表联查"，面试官会追问"你在项目中具体哪里用到了？"——他要的不是SQL语法背诵，而是你知道**什么时候需要多表查询、为什么这样设计**。

> 这道题的含金量在于——能不能把"多表查询"和真实的业务场景对上号。干背语法 = 零分，说出业务场景 + 为什么这样查 = 满分。

---

### 一、ERP项目中多表查询的七大典型场景

| 场景 | 涉及的表 | 查询目的 |
|:---|:---|:---|
| 采购订单详情 | 采购订单 + 供应商 + 商品 | 显示订单时附带供应商名称和商品信息 |
| 销售出库追溯 | 销售出库 + 销售订单 + 客户 | 出库单追溯源订单和客户信息 |
| 库存调拨双向 | 调拨单 + 仓库(出) + 仓库(入) | 显示调出仓库和调入仓库名称 |
| 采购入库溯源 | 采购入库 + 采购订单 + 请购单 | 入库单追溯采购申请来源 |
| 销售退货溯源 | 销售退货 + 销售出库 + 销售订单 | 退货追溯到原始出库和订单 |
| 即时库存查询 | 库存表 + 商品 + 仓库 | 库存列表显示商品名和仓库名 |
| 月度报表汇总 | 出入库明细 + 商品 + 仓库 + 部门 | 按月/按仓库统计出入库汇总 |

---

### 二、具体场景 + SQL示例

#### 场景一：采购订单列表（内连接 + 左连接）

**业务需求**：采购订单列表页需要显示**订单号、供应商名称、制单人、订单日期**。供应商名称在供应商表里，订单里只有供应商ID。

**为什么用 INNER JOIN**：每张采购订单一定有对应的供应商，所以用内连接即可。

```sql
SELECT po.order_no, s.name AS supplier_name, po.creator, po.create_time
FROM purchase_order po
INNER JOIN supplier s ON po.supplier_id = s.id
WHERE po.status = '已审核'
ORDER BY po.create_time DESC;
```

**面试时怎么说**：
> "ERP里每张采购订单一定关联了供应商，所以用INNER JOIN。实际项目中，订单列表页还经常要关联商品表——因为订单行项目里存的是商品ID，显示的时候要翻译成商品名称和规格。"

---

#### 场景二：销售出库单追溯源订单（左连接）

**业务需求**：销售出库列表需要显示——**出库单号、源销售订单号、客户名称**。出库单关联销售订单，销售订单关联客户。三层关联。

**为什么用 LEFT JOIN**：不是每张出库单都有源订单——比如"其它出库"（盘点差异出库）就没有对应的销售订单。如果用INNER JOIN，这类出库单就查不到了。

```sql
SELECT so.out_no, so.create_time, 
       sal.order_no AS source_order,
       c.name AS customer_name
FROM stock_out so
LEFT JOIN sale_order sal ON so.source_order_id = sal.id
LEFT JOIN customer c ON sal.customer_id = c.id
WHERE so.out_type IN ('销售出库', '其它出库')
ORDER BY so.create_time DESC;
```

**面试时怎么说**：
> "这里必须用LEFT JOIN而不是INNER JOIN——因为仓库里除了销售出库，还有盘点差异出库、报废出库等，这些类型的出库单没有源销售订单。如果用了INNER JOIN，这些记录就查不到了，列表数据不完整。"

**这道题在面试里很加分**——你能说出"什么时候用LEFT JOIN而不是INNER JOIN"的业务理由，说明你是在真实项目里写过SQL的。

---

#### 场景三：库存调拨（同一张表JOIN两次）

**业务需求**：调拨单列表需要同时显示**调出仓库名称和调入仓库名称**。调出仓库和调入仓库都在同一张仓库表（warehouse）里。

**关键点**：要JOIN同一张表两次，分别给不同的别名。

```sql
SELECT t.transfer_no, 
       w_out.name AS from_warehouse,  -- 调出仓库
       w_in.name AS to_warehouse,     -- 调入仓库
       t.quantity, t.create_time
FROM transfer t
LEFT JOIN warehouse w_out ON t.from_warehouse_id = w_out.id
LEFT JOIN warehouse w_in ON t.to_warehouse_id = w_in.id
WHERE t.status = '已执行';
```

**面试时怎么说**：
> "调拨单里有两个仓库ID——`from_warehouse_id`和`to_warehouse_id`，都指向同一张warehouse表。这种情况必须给表起两个别名（w_out和w_in），分别JOIN。实际测试中我会专门验证——如果调出仓库和调入仓库是同一个仓库（自己调拨给自己），SQL不会出错。"

---

#### 场景四：采购入库溯源（三级联查）

**业务需求**：查询某张采购入库单的完整来源——从入库→采购订单→请购单，一次性查出**谁请购的、谁下的采购订单、谁入库的**。

```sql
SELECT pi.in_no, pi.create_time AS in_time,
       po.order_no, po.creator AS order_creator,
       pr.request_no, pr.creator AS request_creator,
       pr.request_date
FROM purchase_in pi
LEFT JOIN purchase_order po ON pi.source_order_id = po.id
LEFT JOIN purchase_request pr ON po.request_id = pr.id
WHERE pi.in_no = 'PI-20240601-001';
```

**面试时怎么说**：
> "这张请购→采购订单→入库的三级联查在测试里很有用——当我发现入库数量不对的时候，用这条SQL可以快速查到是哪个环节出了问题，是请购环节就填错了数量，还是采购订单改动了，还是入库时录入错误。"

---

#### 场景五：即时库存查询（展示层常用）

**业务需求**：库存列表页需要显示——**商品名称、规格、仓库名称、当前库存数量**。库存表只存了商品ID和仓库ID，需要JOIN两表翻译。

```sql
SELECT i.id, 
       p.name AS product_name, p.spec AS product_spec,
       w.name AS warehouse_name,
       i.stock_qty, i.locked_qty,
       (i.stock_qty - i.locked_qty) AS available_qty  -- 可用库存
FROM inventory i
INNER JOIN product p ON i.product_id = p.id
INNER JOIN warehouse w ON i.warehouse_id = w.id
WHERE i.stock_qty > 0 OR i.locked_qty > 0;
```

**面试时怎么说**：
> "库存表设计的时候只存了商品ID和仓库ID——这是范式设计，避免数据冗余。但前端展示的时候用户看到的是商品名称和仓库名称，所以必须JOIN。另外库存查询里还加了计算字段`stock_qty - locked_qty AS available_qty`——'可用库存'这个概念在前端是实时算出来的，不是存的。"

---

#### 场景六：月度出入库汇总报表（GROUP BY + 多表联查 + 聚合函数）

**业务需求**：按月、按仓库统计每种商品的入库总量和出库总量。

```sql
SELECT DATE_FORMAT(pi.create_time, '%Y-%m') AS month,
       w.name AS warehouse_name,
       p.name AS product_name,
       SUM(pi.quantity) AS total_in,
       COALESCE(so_total.total_out, 0) AS total_out
FROM purchase_in pi
INNER JOIN warehouse w ON pi.warehouse_id = w.id
INNER JOIN product p ON pi.product_id = p.id
LEFT JOIN (
    SELECT warehouse_id, product_id, 
           DATE_FORMAT(create_time, '%Y-%m') AS month,
           SUM(quantity) AS total_out
    FROM stock_out
    GROUP BY warehouse_id, product_id, month
) so_total ON pi.warehouse_id = so_total.warehouse_id 
          AND pi.product_id = so_total.product_id
          AND DATE_FORMAT(pi.create_time, '%Y-%m') = so_total.month
GROUP BY month, w.name, p.name
ORDER BY month DESC, w.name;
```

---

### 三、面试回答模板（约1.5分钟）

> "多表查询在ERP项目里几乎每天都在用，我最常用的有这么几个场景：
>
> 一是**单据关联翻译**——ERP里单据表为了范式设计只存ID，前端显示时要JOIN翻译。比如采购订单列表要显示供应商名称，库存列表要显示商品名和仓库名。
>
> 二是**业务追溯**——出库单追溯源销售订单、采购入库追溯源采购订单和请购单，这是三级联查。用LEFT JOIN而不是INNER JOIN，因为有些出库类型（盘点出库、报废出库）没有源订单，用内连接会丢数据。
>
> 三是**同一张表JOIN两次**——库存调拨单里调出仓库和调入仓库都指向同一张仓库表，必须起两个别名分别关联。
>
> 四是**报表统计**——月度出入库汇总，要跨出入库两张表做聚合查询。
>
> 作为测试，我用多表查询主要是两个用途：一是验证数据一致性——接口返回的数据和数据库里多表联查的结果是不是一致；二是辅助排查——发现Bug后写一条多表联查快速定位数据是哪个环节出的问题。"

---

### 四、测试视角——你是怎么验证多表查询结果的

| 测试场景 | SQL验证方式 |
|:---|:---|
| 接口返回的供应商名称是否正确 | `SELECT po.order_no, s.name FROM purchase_order po INNER JOIN supplier s...` 对比接口返回 |
| 库存数量接口数据一致性 | `SELECT SUM(stock_qty) FROM inventory WHERE...` 对比接口返回的库存总量 |
| 报表数据准确性 | 对报表SQL做反向校验——取一条明细数据手动算一遍，和报表聚合结果对比 |
| 数据追溯链路完整性 | 三级联查验证请购→采购订单→入库的关联字段是否有断裂 |

---

**关键词**：多表查询 / INNER JOIN / LEFT JOIN / 自关联 / 三级联查 / 数据一致性 / 库存调拨 / 报表聚合

