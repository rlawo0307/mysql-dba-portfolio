# 목차
* Stale Statistics 상태에서 row estimate 동작 확인
* 참고 자료
  * [옵티마이저와 통계정보](../optimizer-statistics.md)
  * [explain 해석](../explain.md)
  * [optimizer trace란](../optimizer-trace.md)
  * [optimizer trace의 각 항목 분석](optimizer-trace-and-cost-model.md)
<br><br>

# Stale Statistics 상태에서 row estimate 동작 확인
## 초기(최신) 통계 상태에서 실행 계획 확인하기
### 실험 준비
```sql
set session cte_max_recursion_depth = 500000;

create table t1(c1 int, c2 int, index idx1(c1));
insert into t1 with recursive seq as
(
    select 1 as n
    union all
    select n+1 from seq where n < 500000
) select n, n*100 from seq; -- 초기 c1 데이터는 unique

analyze table t1; -- 통계 정보 갱신
show index from t1;
```
```sql
+-------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
| Table | Non_unique | Key_name | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment | Visible | Expression |
+-------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
| t1    |          1 | idx1     |            1 | c1          | A         |      486698 |     NULL |   NULL | YES  | BTREE      |         |               | YES     | NULL       |
+-------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
1 row in set (0.01 sec)
```
### 실행 계획 확인
```sql
explain select * from t1 where c1 = 1;
```
```sql
+----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+-------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref   | rows | filtered | Extra |
+----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | t1    | NULL       | ref  | idx1          | idx1 | 5       | const |    1 |   100.00 | NULL  |
+----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)
```
## stale 통계 상태에서 실행 계획 확인하기 
### 데이터 변경 + 통계 갱신 X
```sql
insert into t1
with recursive seq as
(
    select 1 as n
    union all
    select n+1 from seq where n < 500000
) select 1, 100 from seq;
```
### 실제 데이터 분포 확인
```sql
select count(*) from t1 where c1 = 1;
```
```sql
+----------+
| count(*) |
+----------+
|   500001 |
+----------+
1 row in set (0.18 sec)
```
### 실행 계획 확인 
```sql
explain select * from t1 where c1 = 1;
```
```sql
+----+-------------+-------+------------+------+---------------+------+---------+-------+--------+----------+-------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref   | rows   | filtered | Extra |
+----+-------------+-------+------------+------+---------------+------+---------+-------+--------+----------+-------+
|  1 | SIMPLE      | t1    | NULL       | ref  | idx1          | idx1 | 5       | const | 499206 |   100.00 | NULL  |
+----+-------------+-------+------------+------+---------------+------+---------+-------+--------+----------+-------+
1 row in set, 1 warning (0.00 sec)
```
### optimizer trace 확인
```sql
set optimizer_trace="enabled=on";
select * from t1 where c1 = 1;
select trace from information_schema.optimizer_trace\G
```
<details><summary>전체 optimizer trace 결과</summary>

```JSON
*************************** 1. row ***************************
trace: {
  "steps": [
    {
      "join_preparation": {
        "select#": 1,
        "steps": [
          {
            "expanded_query": "/* select#1 */ select `t1`.`c1` AS `c1`,`t1`.`c2` AS `c2` from `t1` where (`t1`.`c1` = 1)"
          }
        ]
      }
    },
    {
      "join_optimization": {
        "select#": 1,
        "steps": [
          {
            "condition_processing": {
              "condition": "WHERE",
              "original_condition": "(`t1`.`c1` = 1)",
              "steps": [
                {
                  "transformation": "equality_propagation",
                  "resulting_condition": "multiple equal(1, `t1`.`c1`)"
                },
                {
                  "transformation": "constant_propagation",
                  "resulting_condition": "multiple equal(1, `t1`.`c1`)"
                },
                {
                  "transformation": "trivial_condition_removal",
                  "resulting_condition": "multiple equal(1, `t1`.`c1`)"
                }
              ]
            }
          },
          {
            "substitute_generated_columns": {
            }
          },
          {
            "table_dependencies": [
              {
                "table": "`t1`",
                "row_may_be_null": false,
                "map_bit": 0,
                "depends_on_map_bits": [
                ]
              }
            ]
          },
          {
            "ref_optimizer_key_uses": [
              {
                "table": "`t1`",
                "field": "c1",
                "equals": "1",
                "null_rejecting": true
              }
            ]
          },
          {
            "rows_estimation": [
              {
                "table": "`t1`",
                "range_analysis": {
                  "table_scan": {
                    "rows": 998412,
                    "cost": 100542
                  },
                  "potential_range_indexes": [
                    {
                      "index": "idx1",
                      "usable": true,
                      "key_parts": [
                        "c1"
                      ]
                    }
                  ],
                  "setup_range_conditions": [
                  ],
                  "group_index_range": {
                    "chosen": false,
                    "cause": "not_group_by_or_distinct"
                  },
                  "skip_scan_range": {
                    "potential_skip_scan_indexes": [
                      {
                        "index": "idx1",
                        "usable": false,
                        "cause": "query_references_nonkey_column"
                      }
                    ]
                  },
                  "analyzing_range_alternatives": {
                    "range_scan_alternatives": [
                      {
                        "index": "idx1",
                        "ranges": [
                          "c1 = 1"
                        ],
                        "index_dives_for_eq_ranges": true,
                        "rowid_ordered": true,
                        "using_mrr": false,
                        "index_only": false,
                        "in_memory": 1,
                        "rows": 499206,
                        "cost": 207492,
                        "chosen": false,
                        "cause": "cost"
                      }
                    ],
                    "analyzing_roworder_intersect": {
                      "usable": false,
                      "cause": "too_few_roworder_scans"
                    }
                  }
                }
              }
            ]
          },
          {
            "considered_execution_plans": [
              {
                "plan_prefix": [
                ],
                "table": "`t1`",
                "best_access_path": {
                  "considered_access_paths": [
                    {
                      "access_type": "ref",
                      "index": "idx1",
                      "rows": 499206,
                      "cost": 52015.2,
                      "chosen": true
                    },
                    {
                      "rows_to_scan": 998412,
                      "access_type": "scan",
                      "resulting_rows": 998412,
                      "cost": 100539,
                      "chosen": false
                    }
                  ]
                },
                "condition_filtering_pct": 100,
                "rows_for_plan": 499206,
                "cost_for_plan": 52015.2,
                "chosen": true
              }
            ]
          },
          {
            "attaching_conditions_to_tables": {
              "original_condition": "(`t1`.`c1` = 1)",
              "attached_conditions_computation": [
              ],
              "attached_conditions_summary": [
                {
                  "table": "`t1`",
                  "attached": "(`t1`.`c1` = 1)"
                }
              ]
            }
          },
          {
            "finalizing_table_conditions": [
              {
                "table": "`t1`",
                "original_table_condition": "(`t1`.`c1` = 1)",
                "final_table_condition   ": null
              }
            ]
          },
          {
            "refine_plan": [
              {
                "table": "`t1`"
              }
            ]
          }
        ]
      }
    },
    {
      "join_execution": {
        "select#": 1,
        "steps": [
        ]
      }
    }
  ]
}
1 row in set (0.00 sec)
```
</details>
<br>

```sql
"analyzing_range_alternatives": {
                    "range_scan_alternatives": [
                      {
                        "index": "idx1",
                        "ranges": [
                          "c1 = 1"
                        ],
                        "index_dives_for_eq_ranges": true, -- equality 조건의 row 수 추정을 위해 옵티마이저가 실제 인덱스 값을 자세하게 확인했는지 여부
                        "rowid_ordered": true,
                        "using_mrr": false,
                        "index_only": false,
                        "in_memory": 1,
                        "rows": 499206,
                        "cost": 207492,
                        "chosen": false,
                        "cause": "cost"
                      }
                    ],
                    "analyzing_roworder_intersect": {
                      "usable": false,
                      "cause": "too_few_roworder_scans"
}
```
## 결과
* 초기 통계 상태에서의 실행 계획
    * rows : `1`
    * filtered : `100.00`
    * 최종 예상 결과 수 : 1 * 100 / 100 = `1`
    * 실제 결과 수 : 1
* 데이터 입력 후 stale 통계 상태에서의 실행 계획
    * rows : `499206`
    * filtered : `100.00`
    * 최종 예상 결과 수 : 499206 * 100 / 100 = `499206`
    * 실제 결과 수 : 500001
## 결론
실험 결과, 통계 정보가 stale 상태라고 해서 항상 row estimate가 크게 부정확해지는 것은 아님을 확인할 수 있다.

초기에는 `c1 = 1` 조건의 실제 결과가 1건이었고, 옵티마이저도 이를 정확하게 추정하였다. 이후 `c1 = 1` 데이터를 500,000건 추가하여 통계 정보를 stale 상태로 만들었지만, 실행 계획에서는 여전히 실제 결과 수(500,001건)에 가까운 값(499,206건)을 추정하였다.

이는 단순 equality 조건에서 인덱스가 존재할 경우, 옵티마이저가 항상 오래된 인덱스 통계만 참고하는 것이 아니라 index dive를 사용하여 equality 조건의 row estimate를 계산할 수 있기 때문이다. (optimizer trace의 `index_dives_for_eq_ranges = true`)

즉, row estimate 정확도는 단순히 통계 정보의 최신 여부만으로 결정되지 않으며, 조건 형태(equality/range), 인덱스 존재 여부, index dive 사용 여부에 따라 달라질 수 있다.