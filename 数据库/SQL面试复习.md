# SQL 3小时速通复习

> 从易到难，涵盖 SQL 操作 + 数据库常识，直击面试高频题。

---

## 3小时时间分配

| 时段 | 时长 | 内容 |
|------|------|------|
| 第1小时 | 30min | 基础语法 + 动手练 |
| 第1小时 | 30min | 多表查询 + 聚合 |
| 第2小时 | 30min | 子查询 + 窗口函数 |
| 第2小时 | 30min | 数据库常识（范式/索引/事务） |
| 第3小时 | 30min | 手撕高频 hard 题 |
| 第3小时 | 30min | 查漏补缺 + 面经对答案 |

---

## 前置准备

新建一个临时数据库用于练习（不会影响现有数据）：

```sql
CREATE DATABASE IF NOT EXISTS interview_practice;
USE interview_practice;

-- 建表
CREATE TABLE departments (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(50) NOT NULL
);

CREATE TABLE employees (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(50) NOT NULL,
    salary DECIMAL(10,2),
    dept_id INT,
    join_date DATE,
    FOREIGN KEY (dept_id) REFERENCES departments(id)
);

CREATE TABLE orders (
    id INT PRIMARY KEY AUTO_INCREMENT,
    employee_id INT,
    amount DECIMAL(10,2),
    order_date DATE,
    status VARCHAR(20),
    FOREIGN KEY (employee_id) REFERENCES employees(id)
);

-- 插入数据
INSERT INTO departments VALUES
(1, '技术部'), (2, '市场部'), (3, '财务部'), (4, '人事部');

INSERT INTO employees VALUES
(1, '张三', 25000, 1, '2020-03-15'),
(2, '李四', 18000, 1, '2021-06-01'),
(3, '王五', 22000, 2, '2019-11-20'),
(4, '赵六', 15000, 2, '2022-01-10'),
(5, '孙七', 28000, 3, '2018-07-08'),
(6, '周八', 12000, 4, '2023-02-28'),
(7, '吴九', NULL, 1, '2024-05-01');

INSERT INTO orders VALUES
(1, 1, 1500.00, '2024-01-15', '已完成'),
(2, 1, 2800.00, '2024-02-20', '已完成'),
(3, 2, 800.00, '2024-03-10', '处理中'),
(4, 3, 3200.00, '2024-01-05', '已完成'),
(5, 3, 450.00, '2024-04-18', '已取消'),
(6, 4, 1200.00, '2024-03-22', '处理中'),
(7, 5, 5000.00, '2024-02-14', '已完成'),
(8, 5, 600.00, '2024-05-01', '处理中'),
(9, 6, 900.00, '2024-04-10', '已完成'),
(10, 1, 3500.00, '2024-05-15', '处理中');
```

建议你打开 MySQL 客户端一边看题一边敲，**只看不写等于白复习**。

---

## 第一小时：基础语法 + 多表查询

### 第1题（入门）: 查询所有员工的名字和薪资

```sql
-- 你的答案：
```

**考察点**：最基本的 `SELECT` / `FROM`

---

### 第2题（入门）: 查询薪资大于 20000 的员工

```sql
-- 你的答案：
```

**考察点**：`WHERE` 条件过滤

---

### 第3题（基础）: 按薪资从高到低排序，取前 3 名

```sql
-- 你的答案：
```

**考察点**：`ORDER BY` / `LIMIT`

---

### 第4题（基础）: 查询每个部门的平均薪资

```sql
-- 你的答案：
```

**考察点**：`GROUP BY` + `AVG()` 聚合函数

---

### 第5题（中等）: 查询平均薪资大于 20000 的部门

```sql
-- 你的答案：
```

**考察点**：`HAVING`（对聚合结果过滤，不能用 WHERE）

---

### 第6题（中等）: 查询每个员工及其部门名称

```sql
-- 你的答案：
```

**考察点**：`INNER JOIN`

---

### 第7题（中等）: 查询所有部门及其员工数（包括没有员工的部门）

```sql
-- 你的答案：
```

**考察点**：`LEFT JOIN` + `COUNT()`，区分 JOIN 类型

---

### 第8题（中等）: 查询薪资高于本部门平均薪资的员工

```sql
-- 你的答案：
```

**考察点**：**相关子查询**，高频面试题

#### 第8题提示

```sql
-- 思路：外层查员工，内层算该部门的平均薪资
SELECT e1.name, e1.salary, e1.dept_id
FROM employees e1
WHERE e1.salary > (
    SELECT AVG(e2.salary)
    FROM employees e2
    WHERE e2.dept_id = e1.dept_id
);
```

---

## 第二小时：子查询 + 窗口函数

### 第9题（中高）: 查询每个部门薪资最高的员工

**两种写法都要会**：

写法一（子查询）：
```sql
SELECT e.name, e.salary, e.dept_id
FROM employees e
WHERE e.salary = (
    SELECT MAX(salary) FROM employees WHERE dept_id = e.dept_id
);
```

写法二（窗口函数）：
```sql
SELECT name, salary, dept_id
FROM (
    SELECT *,
           RANK() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS rk
    FROM employees
) t
WHERE rk = 1;
```

**考察点**：子查询 vs 窗口函数 `RANK() / DENSE_RANK() / ROW_NUMBER()` 的区别

---

### 第10题（中高）: 查询每个员工的薪资排名（部门内）

```sql
-- 你的答案：
```

**考察点**：`RANK() OVER (PARTITION BY ... ORDER BY ...)`

---

### 第11题（中高）: 查询累积订单金额最高的员工

```sql
-- 你的答案：
```

**考察点**：`SUM() OVER (ORDER BY ...)` 窗口聚合

---

### 第12题（中高）: 查询连续 3 个月都有订单的员工

```sql
-- 你的答案：
```

**考察点**：**连续区间问题**，互联网大厂高频 hard 题

#### 第12题提示

```sql
-- 核心技巧：行号与日期的月份差为恒定值
SELECT DISTINCT employee_id
FROM (
    SELECT employee_id,
           DATE_FORMAT(order_date, '%Y-%m') AS month,
           ROW_NUMBER() OVER (PARTITION BY employee_id ORDER BY order_date) AS rn
    FROM orders
    GROUP BY employee_id, DATE_FORMAT(order_date, '%Y-%m')
) t
GROUP BY employee_id, DATE_SUB(month, INTERVAL rn MONTH)
HAVING COUNT(*) >= 3;
```

---

## 第二小时下半段：数据库常识

### 第13题: 什么是三大范式？面试怎么答？

**回答模板**：

| 范式 | 核心要求 | 一句话 |
|------|----------|--------|
| 1NF | 列不可再分 | 每个字段都是原子值 |
| 2NF | 非主键列完全依赖于主键 | 联合主键下不能只依赖部分 |
| 3NF | 非主键列不传递依赖于主键 | 非主键列不能依赖于其他非主键列 |

**面试要点**：
- 一般做到 3NF 就够了
- 实际工作会**适当冗余**（反范式化）来换查询性能
- 例子：订单表冗余用户姓名，省去每次 JOIN

---

### 第14题: 索引的类型和选择原则

| 类型 | 说明 |
|------|------|
| **聚簇索引** | 数据行物理顺序与索引顺序一致（主键默认） |
| **辅助索引** | 非聚簇索引，叶子存主键值 |
| **联合索引** | 多列组合，遵循**最左前缀原则** |
| **唯一索引** | 列值唯一，允许 NULL |
| **全文索引** | 用于文本搜索（MyISAM / InnoDB 5.6+） |

**何时用索引**：
- WHERE / JOIN / ORDER BY / GROUP BY 的列
- 区分度高的列（性别就不适合）

**何时不该用**：
- 频繁更新
- 数据量小
- 区分度低

---

### 第15题: 事务的 ACID 和隔离级别

**ACID**：
- **A**tomicity（原子性）：要么全做，要么全不做
- **C**onsistency（一致性）：事务前后数据完整性一致
- **I**solation（隔离性）：并发事务互不干扰
- **D**urability（持久性）：提交后数据持久化

**隔离级别**（从低到高）：

| 级别 | 脏读 | 不可重复读 | 幻读 |
|------|------|------------|------|
| READ UNCOMMITTED | 可能 | 可能 | 可能 |
| READ COMMITTED | 解决 | 可能 | 可能 |
| REPEATABLE READ | 解决 | 解决 | 可能（InnoDB 通过间隙锁解决） |
| SERIALIZABLE | 解决 | 解决 | 解决 |

**MySQL 默认**：REPEATABLE READ

---

### 第16题: EXPLAIN 怎么看慢查询？

```sql
EXPLAIN SELECT * FROM employees WHERE salary > 20000;
```

关键字段：
- **type**：ALL（全表扫描）→ index → range → ref → const（越来越好）
- **key**：实际使用的索引
- **rows**：扫描行数（越少越好）
- **Extra**：Using filesort（需优化）、Using index（覆盖索引，好）

---

## 第三小时：手撕高频 Hard + 查漏补缺

### 第17题（Hard）: 部门薪资 Top 3（允许并列）

```sql
-- 用 DENSE_RANK 实现
SELECT name, salary, dept_id
FROM (
    SELECT *,
           DENSE_RANK() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS dr
    FROM employees
) t
WHERE dr <= 3;
```

**注意**：为什么用 `DENSE_RANK` 而不是 `RANK`？——如果有同薪，`RANK` 会跳号导致取不够 3 人。

---

### 第18题（Hard）: 查询有订单但从未完成过订单的员工

```sql
SELECT DISTINCT e.name
FROM employees e
WHERE EXISTS (
    SELECT 1 FROM orders o WHERE o.employee_id = e.id
)
AND NOT EXISTS (
    SELECT 1 FROM orders o WHERE o.employee_id = e.id AND o.status = '已完成'
);
```

也可用：
```sql
SELECT e.name
FROM employees e
JOIN orders o ON e.id = o.employee_id
GROUP BY e.id, e.name
HAVING SUM(CASE WHEN o.status = '已完成' THEN 1 ELSE 0 END) = 0;
```

---

### 第19题（Hard）: 求每个月的订单金额环比增长率

```sql
SELECT
    DATE_FORMAT(order_date, '%Y-%m') AS month,
    SUM(amount) AS curr_amount,
    LAG(SUM(amount)) OVER (ORDER BY DATE_FORMAT(order_date, '%Y-%m')) AS prev_amount,
    ROUND((SUM(amount) - LAG(SUM(amount)) OVER (ORDER BY DATE_FORMAT(order_date, '%Y-%m')))
          / LAG(SUM(amount)) OVER (ORDER BY DATE_FORMAT(order_date, '%Y-%m')) * 100, 2) AS growth_rate
FROM orders
GROUP BY DATE_FORMAT(order_date, '%Y-%m');
```

**考察点**：`LAG()` 窗口函数，环比/同比问题

---

### 第20题（Hard）: 删除重复数据（只保留一条）

```sql
-- 假设 employee 表有重复 name，保留 id 最小的
DELETE e1 FROM employees e1
JOIN employees e2
ON e1.name = e2.name AND e1.id > e2.id;
```

## 面试高频场景题

### 场景1: 分页查询优化

```sql
-- 最基础的（数据量大时越来越慢）
SELECT * FROM employees ORDER BY id LIMIT 100000, 20;

-- 优化方案1：子查询用覆盖索引
SELECT * FROM employees
WHERE id > (SELECT id FROM employees ORDER BY id LIMIT 100000, 1)
LIMIT 20;

-- 优化方案2：记录上一页最后一条的 id
SELECT * FROM employees WHERE id > 100020 ORDER BY id LIMIT 20;
```

### 场景2: JOIN 和子查询哪个快？

- 子查询在 **MySQL 5.6 之前**性能差（派生表无索引）
- **5.6+** 会自动优化派生表 JOIN，性能差距不大
- 能用 JOIN 优先用 JOIN，可读性更好

### 场景3: COUNT(*) 和 COUNT(1) 和 COUNT(列名)

- `COUNT(*)` 和 `COUNT(1)` 完全等价，统计行数
- `COUNT(列名)` 统计该列**非 NULL** 的行数

---

## 参考答案速查

| 题号 | 答案 |
|------|------|
| 1 | `SELECT name, salary FROM employees;` |
| 2 | `SELECT * FROM employees WHERE salary > 20000;` |
| 3 | `SELECT * FROM employees ORDER BY salary DESC LIMIT 3;` |
| 4 | `SELECT dept_id, AVG(salary) FROM employees GROUP BY dept_id;` |
| 5 | `SELECT dept_id, AVG(salary) FROM employees GROUP BY dept_id HAVING AVG(salary) > 20000;` |
| 6 | `SELECT e.name, d.name FROM employees e JOIN departments d ON e.dept_id = d.id;` |
| 7 | `SELECT d.name, COUNT(e.id) FROM departments d LEFT JOIN employees e ON d.id = e.dept_id GROUP BY d.id, d.name;` |
| 10 | `SELECT *, RANK() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS rk FROM employees;` |
| 11 | `SELECT employee_id, SUM(amount) AS total FROM orders GROUP BY employee_id ORDER BY total DESC LIMIT 1;` |

---

## 最后叮嘱

**面经最高频的 5 道题**：
1. 手写一个 JOIN 查询（INNER / LEFT / RIGHT）
2. GROUP BY + HAVING 用法，和 WHERE 的区别
3. 窗口函数 RANK / DENSE_RANK / ROW_NUMBER 的区别
4. 事务隔离级别及各自解决的问题
5. EXPLAIN 怎么看 type/key/rows

把这些题在 MySQL 里亲手敲一遍，面试 SQL 题基本稳了。
