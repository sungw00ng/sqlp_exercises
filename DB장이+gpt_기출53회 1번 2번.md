```yml
53회 실기 기출 문제 정리
1번 문제: 주어진 상황을 바탕으로 TO-BE 실행계획이 나오도록 쿼리를 작성하는 문제
2번 문제: 주어진 상황에 맞게 쿼리와 명령어를 작성하는 문제로, 핵심은 EXCHANGE PARTITION 활용
```

```yml
1. 실행계획을 보고 실행계획처럼 나오게 쿼리를 작성하시오.
수정후의 실행계획을 보고 실행계획처럼 나오도록 쿼리를 수정하기.
힌트를 사용하여 정확하게 의도한대로 표현되도록 하기
```

```sql
-- ddl
CREATE TABLE YSBAE.T1
(
    DT  DATE,
    ID  VARCHAR2(10),
    CD  VARCHAR2(10),
    V1  NUMBER
)
;

CREATE UNIQUE INDEX YSBAE.T1_PK
ON YSBAE.T1 (DT,ID) 
;

ALTER TABLE YSBAE.T1
ADD CONSTRAINT T1_PK PRIMARY KEY (DT,ID);


CREATE TABLE YSBAE.T2
(
    DT  DATE,
    ID  VARCHAR2(10),
    CD  VARCHAR2(10),
    V1  NUMBER
)
;

CREATE UNIQUE INDEX YSBAE.T2_PK
ON YSBAE.T2 (DT,ID) 
;

ALTER TABLE YSBAE.T2
ADD CONSTRAINT T2_PK PRIMARY KEY (DT,ID);

CREATE TABLE YSBAE.T3
(
    ID  VARCHAR2(10),
    NM  VARCHAR2(10)
)
;

CREATE UNIQUE INDEX YSBAE.T3_PK
ON YSBAE.T3 (ID) 
;

ALTER TABLE YSBAE.T3
ADD CONSTRAINT T3_PK PRIMARY KEY (ID);


CREATE TABLE YSBAE.T4
(
    ID  VARCHAR2(10),
    NM  VARCHAR2(10)
)
;

CREATE UNIQUE INDEX YSBAE.T4_PK
ON YSBAE.T4 (ID) 
;

ALTER TABLE YSBAE.T4
ADD CONSTRAINT T4_PK PRIMARY KEY (ID);

-- query(AS-IS)
select a.tp,a.nm
from
(
select *
from
(
select 1 as tp, a.id,b.nm
from t1 a, t3 b
where a.id = b.id
and a.dt = sysdate
union all
select 2 as tp, a.id,b.nm
from t2 a, t3 b
where a.id = b.id
and a.dt = sysdate
)
order by tp,nm ) a
where exists (select 1 from t4 where id = a.id) 
and rownum <= 10;

-- Execution Plan(TO-BE)
---------------------------------------------------------------------------
| Id  | Operation                   | Name  | E-Rows |E-Bytes| Cost (%CPU)|
---------------------------------------------------------------------------
|   0 | SELECT STATEMENT            |       |        |       |     1 (100)|
|   1 |  TABLE ACCESS BY INDEX ROWID| T3    |      1 |    14 |     0   (0)|
|*  2 |   INDEX UNIQUE SCAN         | T3_PK |      1 |       |     0   (0)|
|*  3 |  COUNT STOPKEY              |       |        |       |            |
|   4 |   VIEW                      |       |      2 |    20 |     0   (0)|
|   5 |    UNION-ALL                |       |        |       |            |
|*  6 |     COUNT STOPKEY           |       |        |       |            | <---
|   7 |      VIEW                   |       |      1 |    10 |     0   (0)|
|   8 |       NESTED LOOPS          |       |      1 |    23 |     0   (0)|
|*  9 |        INDEX RANGE SCAN     | T1_PK |      1 |    16 |     0   (0)|
|* 10 |        INDEX UNIQUE SCAN    | T4_PK |      1 |     7 |     0   (0)|
|* 11 |     COUNT STOPKEY           |       |        |       |            | <---
|  12 |      VIEW                   |       |      1 |    10 |     0   (0)|
|  13 |       NESTED LOOPS          |       |      1 |    23 |     0   (0)|
|* 14 |        INDEX RANGE SCAN     | T2_PK |      1 |    16 |     0   (0)|
|* 15 |        INDEX UNIQUE SCAN    | T4_PK |      1 |     7 |     0   (0)|
---------------------------------------------------------------------------
```

```yml
--
/* 풀이 작성 */
--
```

```yml
[문제 2]
다음은 파티션 테이블 T1에 대해 수행되는 배치 작업에 대한 설명이다.
해당 배치 작업은 특정 월(2025년 01월)에 해당하는 데이터에 대해
UPDATE 작업이 약 90%, DELETE 작업이 약 10% 비중으로 수행된다.
이로 인해 다량의 UNDO/REDO가 발생하며 성능 저하가 발생하고 있다.

성능 향상을 위해 파티션 EXCHANGE 방식을 사용하여
동일한 처리 결과를 얻을 수 있도록 쿼리 및 명령어를 작성하시오.

- 파티션 대상은 P202501 이다.
- 교환 대상 테이블명은 T1_P202501 로 작성하시오.
- 파티션 EXCHANGE를 이용하여 UPDATE/DELETE 작업을 대체하시오.
- 기존 데이터 처리 결과와 논리적으로 동일해야 한다.
- 필요한 경우 CREATE, INSERT, ALTER TABLE 문을 모두 작성하시오.
```

```sql
-- ddl
drop table t1;
create table t1
(
    DT  DATE,
    ID  VARCHAR2(10),
    CD  VARCHAR2(10),
    V1  NUMBER
)
partition by range(dt)
(
partition p202501 values less than (to_date('2025/02/01 00:00:00','YYYY/MM/DD HH24:MI:SS')),
partition p202502 values less than (to_date('2025/03/01 00:00:00','YYYY/MM/DD HH24:MI:SS')),
partition p202503 values less than (to_date('2025/04/01 00:00:00','YYYY/MM/DD HH24:MI:SS'))
);


create unique index t1_pk on t1(dt,id) local;

-- query(AS-IS)
merge into t1 a
using t2 b
on (a.id = b.id and a.dt = b.dt)
when matched then
update a.v1 = a.v1 + b.v1
delete cd <= 100
when not matches then
insert into (dt,id,v1)
values (b.dt,b.id,b.v1);

-- Execution Plan
-------------------------------------------------------------------------------------------------------------
| Id  | Operation                     | Name       | E-Rows |E-Bytes| Cost (%CPU)| E-Time   | Pstart| Pstop |
-------------------------------------------------------------------------------------------------------------
|   0 | INSERT STATEMENT              |            |        |       |     2 (100)|          |       |       |
|   1 |  LOAD TABLE CONVENTIONAL      | T1_P202501 |        |       |            |          |       |       |
|   2 |   NESTED LOOPS OUTER          |            |      1 |    65 |     2   (0)| 00:00:01 |       |       |
|   3 |    PARTITION RANGE ALL        |            |      1 |    36 |     2   (0)| 00:00:01 |     1 |     3 |
|*  4 |     TABLE ACCESS FULL         | T1         |      1 |    36 |     2   (0)| 00:00:01 |     1 |     3 |
|   5 |    TABLE ACCESS BY INDEX ROWID| T2         |      1 |    29 |     0   (0)|          |       |       |
|*  6 |     INDEX UNIQUE SCAN         | T2_PK      |      1 |       |     0   (0)|          |       |       |
-------------------------------------------------------------------------------------------------------------
```

```yml
--
/* 풀이 작성 *.
--
```

#

## 1번 정답
```sql
SELECT /*+ LEADING(A) */
       A.TP,
       (SELECT /*+ NO_UNNEST */
               NM
        FROM   T3
        WHERE  ID = A.ID) AS NM
FROM (
    SELECT *
    FROM (
        SELECT /*+ LEADING(A B)
                  USE_NL(B)
                  INDEX(A T1_PK)
                  INDEX(B T4_PK) */
               1 AS TP,
               A.ID
        FROM   T1 A,
               T4 B
        WHERE  A.ID = B.ID
        AND    A.DT = SYSDATE
    )
    WHERE ROWNUM <= 10   -- COUNT STOPKEY (Id 6)
    
    UNION ALL
    
    SELECT *
    FROM (
        SELECT /*+ LEADING(A B)
                  USE_NL(B)
                  INDEX(A T2_PK)
                  INDEX(B T4_PK) */
               2 AS TP,
               A.ID
        FROM   T2 A,
               T4 B
        WHERE  A.ID = B.ID
        AND    A.DT = SYSDATE
    )
    WHERE ROWNUM <= 10   -- COUNT STOPKEY (Id 11)
) A
WHERE ROWNUM <= 10;      -- 최종 STOPKEY (Id 3)

```

## 2번 정답
```sql
-- t1_p202501 생성
create table t1_p202501 
(
    DT  DATE,
    ID  VARCHAR2(10),
    CD  VARCHAR2(10),
    V1  NUMBER
);

-- index
create unique index t1_p202501_pk on t1_p202501(dt,id) ;

-- insert
insert into t1_p202501
select
(case when a.dt is not null and a.id is not null then a.dt else b.dt end) dt,
(case when a.dt is not null and a.id is not null then a.id else b.id end) id,
(case when a.dt is not null and a.id is not null then a.v1 + b.v1 else b.v1 end) v1
from t1 a,t2 b
where a.dt = b.dt(+)
and a.id = b.id(+)
and a.cd > 100; 

-- commit
commit;

-- partition exchange
alter table t1 exchange partition p202501 with table t1_p202501 including indexes without validation;

-- drop table
drop table t1_p202501;
```
