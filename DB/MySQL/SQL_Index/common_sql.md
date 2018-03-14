---
# 常见sql问题
---

## Ⅰ、范围问题
范围问题包括两种情况，求连续范围和缺失范围
```
测试用例：
(root@localhost) [test]> select * from t;
+-----+
| a   |
+-----+
|   1 |
|   2 |
|   3 |
|  45 |
|  46 |
|  47 |
|  48 |
|  49 |
|  99 |
| 100 |
| 101 |
+-----+
11 rows in set (0.00 sec)
```

### 1.1 连续范围
解决思路：给字段a加一个行号，计算行号与a的差，差值相同就是连续的，根据差值分组，取最大和最小即可
```
(root@localhost) [test]> SELECT 
    ->     MIN(a) start_range, MAX(a) end_range
    -> FROM
    ->     (SELECT 
    ->         a, rn, a - rn AS diff
    ->     FROM
    ->         (SELECT 
    ->         a, @a:=@a + 1 rn
    ->     FROM
    ->         t, (SELECT @a:=0) AS a) AS b) AS c
    -> GROUP BY diff;
+-------------+-----------+
| start_range | end_range |
+-------------+-----------+
|           1 |         3 |
|          45 |        49 |
|          99 |       101 |
+-------------+-----------+
3 rows in set (0.00 sec)
```

### 1.2 缺失范围
解决思路：做一个新列，将a列每条记录往前挪一位，新列-原列>1则说明不连续，此时原列+1即缺失范围的开始，新列-1则为缺失范围的结束
```
SELECT 
    cur + 1 start_range, next - 1 end_range
FROM
    (SELECT 
        a cur,
            (SELECT 
                    MIN(a)
                FROM
                    t b
                WHERE
                    b.a > a.a) next
    FROM
        t a) r
WHERE
    next - cur > 1;
```
