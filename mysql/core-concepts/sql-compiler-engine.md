# MySQL SQL Compiler Engine
* MySQL에서 SQL이 실행되기 전까지의 처리 과정을 담당하는 컴포넌트 집합
```text
MySQL Server
 ├─ SQL Layer
 │    ├─ Parser
 │    ├─ Resolver
 │    ├─ Optimizer
 │    └─ Executor
 └─ Storage Engine
      └─ InnoDB
```
<br>

# Parser
* 구문 분석 단계
* 사용 언어 : C/C++, lex/yacc
* 결과물 : Parse Tree(`AST`)
* 역할
    * SQL 문자열을 토큰으로 분해한 후 문법 검사
    * 문법 오류 검사
    * 토큰 레벨 기본 구조 검증
* 테이블 접근 없음
## AST란?
* AST = Abstract Syntax Tree(추상 구문 트리)
* SQL 문자열을 파싱한 뒤 만들어지는 트리 구조의 내부 표현
* 단순히 구문 구조만 표현한 트리로, 논리적 구조 표현체
```sql
ex) select c1, c2 from t1 where c1 > 10 and c2 = 0;

SelectStatement
├── SelectList
│     ├── ColumnReference(c1)
│     └── ColumnReference(c2)
├── FromClause
│     └── TableReference(t1)
└── WhereClause
      └── LogicalAnd
            ├── GreaterThan
            │     ├── ColumnReference(c1)
            │     └── IntegerLiteral(10)
            └── Equals
                  ├── ColumnReference(c2)
                  └── IntegerLiteral(0)
```
<br>

# Resolver (=Name Resolution)
* parser에서 만든 AST를 수정/보강하는 단계
* 결과물 : `Name-resolved Query Tree` (의미가 확정된 AST)
* 역할
    * 테이블, 컬럼 등 객체 존재 여부 확인
    * 객체 간 논리적 연관성을 검사
    * alias 처리
    * view 확장
    * asterisk(*)를 실제 컬럼 목록으로 확장
    * 권한 검사
    * 표현식의 데이터 타입 결정 등
<br>

# Optimizer
* MySQL은 Cost-Based Optimizer(CBO)를 사용
* 결과물 : `Optimized Execution Plan` (=physical plan)
    * explain 명령어로 실행계획 확인 가능
* 역할
    * Access path 선택
        * 인덱스 사용 여부
        * 어떤 인덱스를 사용할지
        * scan 방식 선택
        * lookup 여부 등
    * join 순서 및 방식 결정
    * filesort 사용 여부
    * temp table 사용 여부 등
* [→ optimizer에 대한 내용 자세히 보기](../optimization/optimizer-statistics.md)
<br>

# Executor
* Optimizer가 만든 실행 계획을 실제로 수행하는 단계
* 결과물 : `Result Set` (클라이언트로 반환될 데이터)
* 역할
    * row 단위 처리
    * 인덱스 탐색 수행
    * buffer pool 접근
    * storage engine 호출
    * 결과 생성
