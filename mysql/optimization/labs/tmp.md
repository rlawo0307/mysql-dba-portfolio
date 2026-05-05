# 목차
* Optimizer Trace + Cost Model 분석 실습
* 참고 자료
<br><br>

# Optimizer Trace + Cost Model 분석 실습
* optimizer trace를 통해 실행 계획 선택 과정을 확인해보자
* cost model이 어떤 기준으로 실행 계획을 선택하는지 분석해보자
* cardinality / selectivity / histogram이 cost 계산에 미치는 영향을 확인해보자
## 실험 준비
* `optimizer_trace_max_mem_size`의 기본값은 `16K`
    * trace 결과가 복잡하고 길면 잘릴 수 있음
    * trace 결과가 잘리는 현상을 방지하기 위해 1MB로 확장 필요
```sql
set session cte_max_recursion_depth = 500000;

create table t1(id int primary key, c1 int, c2 int) engine=InnoDB;
insert into t1 with recursive seq as
(
    select 1 as n
    union all
    select n+1 from seq where n < 500000
)
select
    n,
    case
        when n <= 450000 then 1
        else n
    end as c1,      -- c1의 90%는 1로 skewed data
    n % 100 as c2   -- c2는 0~99로 uniform data
from seq;

set optimizer_trace="enabled=on"; -- optimizer trace 활성화
set optimizer_trace_max_mem_size=1000000; -- 1MB로 저장 용량 확장
```
## 인덱스가 없는 경우(Full Table Scan)
```sql
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
            "expanded_query": "/* select#1 */ select `t1`.`id` AS `id`,`t1`.`c1` AS `c1`,`t1`.`c2` AS `c2` from `t1` where (`t1`.`c1` = 1)" 
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
            ]
          },
          {
            "rows_estimation": [
              {
                "table": "`t1`",
                "table_scan": {
                  "rows": 500001,
                  "cost": 0.75
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
                      "rows_to_scan": 500001,
                      "access_type": "scan",
                      "resulting_rows": 500001,
                      "cost": 50000.9,
                      "chosen": true
                    }
                  ]
                },
                "condition_filtering_pct": 100,
                "rows_for_plan": 500001,
                "cost_for_plan": 50000.9,
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
                "final_table_condition   ": "(`t1`.`c1` = 1)"
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
<details><summary>join_preparation 해석</summary>

```sql
"join_preparation": {
        "select#": 1, -- 1번 query block
        "steps": [
          {
            "expanded_query": "/* select#1 */ select `t1`.`id` AS `id`,`t1`.`c1` AS `c1`,`t1`.`c2` AS `c2` from `t1` where (`t1`.`c1` = 1)" 
            -- MySQL이 사용자가 작성한 SQL을 내부적으로 해석한 형태
            -- select * 가 실제 컬럼 목록으로 풀림
          }
        ]
}
```
</details>
<details><summary>condition_processing 해석</summary>

```sql
-- 조건을 내부적으로 분석하고 변환하는 과정
"condition_processing": {
              "condition": "WHERE", -- where 절 조건 분석 및 변환
              "original_condition": "(`t1`.`c1` = 1)", -- 유저가 작성한 쿼리
              "steps": [
                {
                    -- 동등(=) 관계를 내부 표현으로 변환하는 단계
                  "transformation": "equality_propagation",
                  "resulting_condition": "multiple equal(1, `t1`.`c1`)"
                },
                {
                    -- 상수 값을 다른 조건에 퍼뜨리는 단계
                    -- 현재 조건에서는 변화 없음
                  "transformation": "constant_propagation",
                  "resulting_condition": "multiple equal(1, `t1`.`c1`)"
                },
                {
                    -- 불필요한 조건 제거
                    -- 현재 조건에서는 변화 없음
                  "transformation": "trivial_condition_removal",
                  "resulting_condition": "multiple equal(1, `t1`.`c1`)"
                }
              ]
}
```
</details>
<details><summary>substitute_generated_columns 해석</summary>

```sql
-- expression을 포함한 조건을 generated column으로 치환해서 인덱스를 사용할 수 있도록 만드는 작업 
"substitute_generated_columns": { -- 현재 실습에서는 할 일 없음
}
```
</details>
<details><summary>table_dependencies 해석</summary>

```sql
-- 각 테이블의 조인 순서 및 의존 관계를 정의하는 단계
"table_dependencies": [
              {
                "table": "`t1`",
                "row_may_be_null": false, -- 이 테이블의 row가 null로 확장될 가능성이 있는지 여부
                "map_bit": 0, -- 옵티마이저 내부에서 테이블을 식별하는 bit
                "depends_on_map_bits": [  -- 이 테이블이 의존하는 다른 테이블의 bit 번호
                ]
              }
]
```
</details>
<details><summary>(★) ref_optimizer_key_uses 해석 </summary>

```sql
-- ref 방식(equality 조건 + index lookup)으로 사용할 수 있는 인덱스 후보 목록
"ref_optimizer_key_uses": [ -- 현재 실습에서는 활용 가능한 인덱스 후보 없음
]
```
</details>
<details><summary>(★) rows_estimation 해석 </summary>

```sql
-- 옵티마이저가 추정한 읽을 row 추정 값
"rows_estimation": [
              {
                "table": "`t1`",
                "table_scan": {
                  "rows": 500001, -- 옵티마이저가 예상한 scan row 수
                  "cost": 0.75 -- row 수 추정 과정에서 계산된 cost
                }              -- 초기 estimation 단계 cost
              }                -- 최종 실행 계획 비용이 아님
]
```
</details>
<details><summary>(★) considered_execution_plans 해석 </summary>

```sql
-- 옵티마이저가 후보 실행 계획을 비교하고 최종 선택하는 단계
"considered_execution_plans": [
              {
                "plan_prefix": [ -- 현재 테이블 앞에 이미 선택된 테이블 목록
                ],
                "table": "`t1`",
                "best_access_path": { -- 현재 테이블 기준, 선택된 가장 좋은 path
                  "considered_access_paths": [
                    {
                      "rows_to_scan": 500001, -- 읽어야 할 row 수
                      "access_type": "scan", -- full table scan
                      "resulting_rows": 500001, -- 이 access path 이후 남을 것으로 예상되는 row 수
                                                -- 현재 실습에서는 필터링이 반영 안됨
                      "cost": 50000.9, -- 이 access path의 cost
                      "chosen": true -- 최종적으로 선택됐는지 여부
                    }
                  ]
                },
                "condition_filtering_pct": 100, -- 해당 access path 이후 추가 조건 필터링으로 row 수가 얼마나 줄어들 것으로 예상하는지(%)
                                                -- 인덱스 없거나 사용되지 않을 경우, 100%는 필터링이 되지 않았음을 의미. 즉, 조건(c1=1)을 반영하지 못함
                "rows_for_plan": 500001, -- 이 실행 계획에서 최종적으로 처리해야 할 row 수
                "cost_for_plan": 50000.9, -- 이 실행 계획의 전체 cost
                "chosen": true -- 최종적으로 선택됐는지 여부
              }
]
```
</details>
<details><summary>attaching_conditions_to_tables 해석</summary>

```sql
-- 조건을 어느 테이블에 적용할지 결정해서 붙이는 단계
"attaching_conditions_to_tables": {
              "original_condition": "(`t1`.`c1` = 1)", -- 조건
              "attached_conditions_computation": [ -- 조건을 어디에 붙일지 계산 과정
                                                   -- 현재 실습에서는 조건이 단순해서 비어 있음 
              ],
              "attached_conditions_summary": [ -- 결과
                {
                  "table": "`t1`", -- 적용할 테이블
                  "attached": "(`t1`.`c1` = 1)" -- 적용할 조건
                }
              ]
}
```
</details>
<details><summary>finalizing_table_conditions 해석</summary>

```sql
-- 각 테이블에 붙은 조건을 최종 실행용으로 확정하는 단계
"finalizing_table_conditions": [
              {
                "table": "`t1`", -- 조건이 적용될 대상 테이블
                "original_table_condition": "(`t1`.`c1` = 1)", -- attach 단계에서 붙은 조건
                "final_table_condition   ": "(`t1`.`c1` = 1)" -- 최종적으로 실행 시 사용될 조건
              }
]
```
</details>

## 인덱스가 있는 경우
```sql
create index idx1 on t1(c1); -- 인덱스 생성
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
            "expanded_query": "/* select#1 */ select `t1`.`id` AS `id`,`t1`.`c1` AS `c1`,`t1`.`c2` AS `c2` from `t1` where (`t1`.`c1` = 1)"
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
                    "rows": 499352,
                    "cost": 50201.6
                  },
                  "potential_range_indexes": [
                    {
                      "index": "PRIMARY",
                      "usable": false,
                      "cause": "not_applicable"
                    },
                    {
                      "index": "idx1",
                      "usable": true,
                      "key_parts": [
                        "c1",
                        "id"
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
                        "in_memory": 0,
                        "rows": 249676,
                        "cost": 87386.9,
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
                      "rows": 249676,
                      "cost": 25760.4,
                      "chosen": true
                    },
                    {
                      "rows_to_scan": 499352,
                      "access_type": "scan",
                      "resulting_rows": 499352,
                      "cost": 50199.5,
                      "chosen": false
                    }
                  ]
                },
                "condition_filtering_pct": 100,
                "rows_for_plan": 249676,
                "cost_for_plan": 25760.4,
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
<details><summary>(★) ref_optimizer_key_uses 해석</summary>

```sql
-- ref 방식(equality 조건 + index lookup)으로 사용할 수 있는 인덱스 후보 목록
"ref_optimizer_key_uses": [
              {
                "table": "`t1`", -- 대상 테이블
                "field": "c1", -- 인덱스에 사용될 컬럼
                "equals": "1", -- 조건(c1 = 1)으로 lookup 가능
                "null_rejecting": true -- 해당 조건의 null 값 허용 여부
              }
]
```
</details>
<details><summary>(★) rows_estimation 해석</summary>

```sql
-- 현재 실습에서 옵티마이저가 full table scan과 index range scan을 비교한 결과
"rows_estimation": [
              {
                "table": "`t1`",
                "range_analysis": {
                  "table_scan": { -- full table scan
                    "rows": 499352, -- 읽기 예상 row 수
                    "cost": 50201.6 -- 읽기 예상 cost
                  },
                  "potential_range_indexes": [ -- index range scan 후보 목록
                    {
                      "index": "PRIMARY", -- 후보 1 : pk
                      "usable": false, -- 사용 불가
                      "cause": "not_applicable" -- 원인 : 조건이 pk(id)가 아니라 c1이기 때문에
                    },
                    {
                      "index": "idx1", -- 후보 2 : idx1
                      "usable": true, -- 사용 가능
                      "key_parts": [ -- 해당 인덱스를 구성하는 key(컬럼) 목록
                        "c1",
                        "id" -- idx1은 secondary index이므로 내부적으로 leaf node에 pk를 가지고 있음
                      ]
                    }
                  ],
                  "setup_range_conditions": [ -- range scan에서 where 조건을 인덱스 탐색용 범위 조건 변환하는 단계
                  ],
                  "group_index_range": { -- group index range scan을 사용할 수 있는지 확인
                    "chosen": false, -- 사용 불가
                    "cause": "not_group_by_or_distinct" -- 원인 : 현재 쿼리는 group by나 distinct가 없음
                  },
                  "skip_scan_range": { -- skip scan을 사용할 수 있는지 확인
                    "potential_skip_scan_indexes": [
                      {
                        "index": "idx1",
                        "usable": false, -- 사용 불가
                        "cause": "query_references_nonkey_column" -- 원인 : 인덱스에 없는 컬럼을 조회함
                      }
                    ]
                  },
                  "analyzing_range_alternatives": { -- 인덱스를 사용하는 여러 range scan 후보들을 실제로 비교하는 단계
                    "range_scan_alternatives": [ -- 인덱스를 사용한 range scan 후보 목록
                      {
                        "index": "idx1", -- 후보 1 : idx1
                        "ranges": [ -- 인덱스에 탐색할 범위
                          "c1 = 1" -- c1 = 1인 구간을 range scan 할 수 있음
                        ],
                        "index_dives_for_eq_ranges": true, -- row 수를 추정하기 위해 옵티마이저가 인덱스 통계를 더 자세하게 확인했는지 여부
                        "rowid_ordered": true, -- rowid(pk) 값이 정렬된 순서로 나오는지 여부
                        "using_mrr": false, -- Multi-Range Read 사용 여부
                        "index_only": false, -- covering index 여부. true면 covering index
                        "in_memory": 0, -- 인덱스 및 데이터가 buffer pool에 있을 것으로 예상되는 비율
                        "rows": 249676, -- 읽기 예상 row 수
                        "cost": 87386.9, -- 읽기 예상 cost
                        "chosen": false, -- 최종적으로 선택됐는지 여부
                        "cause": "cost" -- 선택되지 않은 원인 : 인덱스를 쓰는게 더 비싸다고 판단
                      }
                    ],
                    "analyzing_roworder_intersect": { -- index merge 같은 후보들을 검토하는 단계
                      "usable": false, -- 사용 불가
                      "cause": "too_few_roworder_scans" -- 원인 : row 순서를 활용해서 합칠 인덱스 후보가 부족함. (인덱스가 idx1 한개 밖에 없음)
                    }
                  }
                }
              }
]
```
</details>
<details><summary>(★) considered_execution_plans 해석</summary>

```sql
-- 옵티마이저가 후보 실행 계획을 비교하고 최종 선택하는 단계
"considered_execution_plans": [
              {
                "plan_prefix": [ -- 현재 테이블 앞에 이미 선택된 테이블 목록
                ],
                "table": "`t1`",
                "best_access_path": { -- 현재 테이블 기준, 선택된 가장 좋은 path
                  "considered_access_paths": [
                    {
                      "access_type": "ref", -- equality + index lookup
                      "index": "idx1",
                      "rows": 249676,
                      "cost": 25760.4,
                      "chosen": true -- 인덱스를 사용하는 방식이 선택됨
                    },
                    {
                      "rows_to_scan": 499352,
                      "access_type": "scan", -- full table scan
                      "resulting_rows": 499352,
                      "cost": 50199.5,
                      "chosen": false
                    }
                  ]
                },
                "condition_filtering_pct": 100, -- 해당 access path 이후 추가 조건 필터링으로 row 수가 얼마나 줄어들 것으로 예상하는지(%)
                                                -- 인덱스가 사용된 경우, 100%는 조건이 이미 인덱스 탐색 조건으로 사용되었기 때문에 추가 필터링이 남지 않음을 의미
                "rows_for_plan": 249676, -- 이 실행 계획에서 최종적으로 처리해야 할 row 수 (ref 실행 계획의 읽기 추정 row 수와 동일)
                "cost_for_plan": 25760.4, -- 이 실행 계획의 전체 cost (ref 실행 계획의 cost와 동일)
                "chosen": true
              }
]
```
</details>
<details><summary>finalizing_table_conditions 해석</summary>

```sql
-- 각 테이블에 붙은 조건을 최종 실행용으로 확정하는 단계
"finalizing_table_conditions": [
              {
                "table": "`t1`",
                "original_table_condition": "(`t1`.`c1` = 1)", -- attach 단계에서 붙은 조건
                "final_table_condition   ": null -- 최종적으로 실행 시 사용될 조건
                                                 -- 조건이 이미 인덱스 탐색 조건으로 사용되어서 별도로 테이블 필터로 적용할 필요 없음 
              }
]
```
</details>

## 결론