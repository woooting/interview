# WHERE 条件子句

`WHERE` 用于从 `FROM` 的结果集中筛选满足条件的行。

## 1. 比较运算符

| 运算符 | 含义 | 示例 |
|--------|------|------|
| `=` | 等于 | `WHERE age = 18` |
| `<>` 或 `!=` | 不等于 | `WHERE age <> 18` |
| `>` | 大于 | `WHERE salary > 5000` |
| `<` | 小于 | `WHERE salary < 5000` |
| `>=` | 大于等于 | `WHERE price >= 100` |
| `<=` | 小于等于 | `WHERE price <= 100` |

## 2. 逻辑运算符

| 运算符 | 含义 | 示例 |
|--------|------|------|
| `AND` | 且（所有条件为真） | `WHERE age > 18 AND city = 'Beijing'` |
| `OR` | 或（任一条件为真） | `WHERE city = 'Beijing' OR city = 'Shanghai'` |
| `NOT` | 非（取反） | `WHERE NOT city = 'Beijing'` |

**注意**：`AND` 优先级高于 `OR`，建议用括号明确优先级。

## 3. 范围条件

| 条件 | 示例 | 说明 |
|------|------|------|
| `BETWEEN ... AND ...` | `WHERE age BETWEEN 18 AND 30` | 包含两端 |
| `IN (...)` | `WHERE city IN ('Beijing', 'Shanghai')` | 匹配列表中的任意值 |
| `NOT IN (...)` | `WHERE city NOT IN ('Beijing', 'Shanghai')` | 排除列表中的值 |

## 4. 模糊匹配（LIKE）

| 通配符 | 含义 | 示例 |
|--------|------|------|
| `%` | 匹配任意多个字符 | `WHERE name LIKE '张%'` → 张三、张伟 |
| `_` | 匹配单个字符 | `WHERE name LIKE '张_'` → 张三（不匹配张三丰） |
| `NOT LIKE` | 取反 | `WHERE name NOT LIKE '张%'` |

## 5. 空值判断

| 条件 | 示例 | 说明 |
|------|------|------|
| `IS NULL` | `WHERE email IS NULL` | 值为 `NULL` |
| `IS NOT NULL` | `WHERE email IS NOT NULL` | 值不为 `NULL` |

**注意**：`NULL` 不能用 `= NULL` 或 `<> NULL` 判断，必须用 `IS NULL` / `IS NOT NULL`。

## 6. 正则表达式（MySQL）

| 运算符 | 示例 | 说明 |
|--------|------|------|
| `REGEXP` / `RLIKE` | `WHERE name REGEXP '^张'` | 匹配正则 |

## 7. 子查询条件

| 条件 | 示例 | 说明 |
|------|------|------|
| `EXISTS` | `WHERE EXISTS (SELECT 1 FROM orders WHERE user_id = users.id)` | 子查询返回结果集则为真 |
| `NOT EXISTS` | `WHERE NOT EXISTS (...)` | 子查询无结果则为真 |
| `IN + 子查询` | `WHERE id IN (SELECT user_id FROM orders)` | 值在子查询结果中 |
| `比较运算符 + ANY` | `WHERE salary > ANY (SELECT salary FROM emp)` | 大于子查询中任意一个 |
| `比较运算符 + ALL` | `WHERE salary > ALL (SELECT salary FROM emp)` | 大于子查询中所有值 |

---

# SQL 子句执行顺序

```sql
SELECT    -- 5. 选择列
FROM      -- 1. 数据来源表
WHERE     -- 2. 行筛选条件
GROUP BY  -- 3. 分组
HAVING    -- 4. 分组后的筛选条件
ORDER BY  -- 6. 排序
LIMIT     -- 7. 限制返回行数（MySQL） / TOP / FETCH
```

**关键结论**：
- `WHERE` 在 `GROUP BY` 之前执行，所以不能对聚合结果用 `WHERE`，必须用 `HAVING`
- `SELECT` 在第 5 步才执行，意味着 `SELECT` 中取的别名不能在 `WHERE / GROUP BY` 中使用（但可以在 `ORDER BY` 中使用）
- 理解执行顺序才能写出正确的 SQL
