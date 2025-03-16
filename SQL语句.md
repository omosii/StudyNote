### 查询某些列（所有列）并打印（user_profile是表名）
```sql
select 
id ,device_id ,gender, age ,university ,province // 或是 *
from user_profile;
```

### 取出某一列的去重信息并打印（user_profile是表名）
```sql
// 第一种方案
select distinct university from user_profile

// 第二种方案
SELECT
university
from user_profile
group by university
```
- 第一种方案：在列前面添加`distinct`关键字，该关键字实现去重功能。`distinct`必须放在所有列的前面，而不是单个列前，它的作用是对所有选中的列的组合进行去重。
  - `distinct`去重的底层原理可以是排序去重（先排序，在遍历，跳过重复的），也可以是哈希去重（使用哈希表记录已出现的哈希值）
- 第二种方案：通过 `GROUP BY university`对数据按大学分组，每组仅保留一行记录，最终结果与`SELECT DISTINCT university`等效。将所有行按 university 字段的值分组，相同大学的记录归为一组。从每组中选取一行作为输出（通常选取第一条，但未明确指定时结果可能不稳定）。输出所有分组的 university 值。

### 查询结果限制返回行数
```sql
select
device_id 
from user_profile
limit 0, 2  // 这里的0 2 分别是返回的起始行号 返回的行数   如果只填一个参数，就是返回的最大行数 e.g. limit 2
```
- 这里的`limit 0, 2`分别是返回的起始行号、返回的行数
- 如果只填一个参数，就是返回的最大行数 `e.g. limit 2`

### 将查询后的列重新命名
