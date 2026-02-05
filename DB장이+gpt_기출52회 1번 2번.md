
```yml
52회 실기 기출 문제 유형 정리
실기 1번 : 3장 3절(인덱스) + 3정 4절(NL조인),"정렬 제거, 부분범위 처리, 스칼라 서브쿼리"
실기 2번 : 3장 4절(해시조인) + 3장 5절 3항(쿼리변환),"파티션 Pruning, Build Input 제어, 집계 뷰(Group By)"
```
##
##

```yml
1. 테이블 ddl과 기존 SQL을 참고하여 수정 후의 실행계획이 똑같이 나오도록 쿼리를 수정하시오.
- 인덱스 수정이 필요하면 정확한 구문을 작성하시오. (drop index, create index)
- 힌트를 사용하여 정확하게 의도한 실행계획대로 표현되도록 하시오.
- 10건 추출 시 부분범위처리가 가능하게 하시오.
```

```sql
-- 1. 주문 테이블 및 인덱스
create table 주문(
  주문번호 number not null,
  고객번호 varchar2(11) not null,
  주문일시 date,
  배송번호 number,
  constraint 주문_pk primary key (주문번호)
);
create index 주문_x1 on 주문(고객번호, 주문일시);

-- 2. 상품이력 테이블 및 인덱스
create table 상품이력 (
  상품번호 number not null,
  시작일시 date not null,
  종료일시 date not null,
  이벤트명 varchar2(10) not null,
  상품금액 number,
  constraint 상품이력_pk primary key (상품번호)
);
create index 상품이력_x1 on 상품이력(상품번호, 시작일시);

-- 3. 주문상품 테이블
create table 주문상품(
  주문번호 number not null,
  상품번호 number not null,
  주문수량 number,
  constraint 주문상품_pk primary key (주문번호, 상품번호)
);

-- 4. 배송 테이블
create table 배송 (
  배송번호 number not null,
  배송상태코드 varchar2(5),
  constraint 배송_pk primary key (배송번호)
);

-- [수정 전 SQL]
select a.주문번호, a.주문일시, a.주문금액, a.배송상태코드
from (
  select a.주문번호, a.주문일시, (b.주문수량 * c.상품금액) as 주문금액, 
         (select 배송상태코드 from 배송 where 배송번호 = a.배송번호) as 배송상태코드,
         dense_rank() over (order by a.주문일시 desc) rnum
  from 주문 a, 주문상품 b, 상품이력 c
  where a.고객번호 ='00000000042'
    and b.주문번호 = a.주문번호
    and c.상품번호(+) = b.상품번호
    and a.주문일시 between c.시작일시(+) and c.종료일시(+)
    and c.이벤트명(+) = 'SQLP'
) a
where rnum <= 10;
```

```sql
-- 수정 후의 실행 계획
-----------------------------------------------------------
   0      SELECT STATEMENT Optimizer=ALL_ROWS
   1    0   TABLE ACCESS (BY INDEX ROWID) OF '배송'
   2    1     INDEX (UNIQUE SCAN) OF '배송_PK' (UNIQUE)
   3    0   NESTED LOOPS (OUTER)
   4    3     NESTED LOOPS
   5    4       VIEW
   6    5         COUNT (STOPKEY)
   7    6           VIEW
   8    7             TABLE ACCESS (BY INDEX ROWID) OF '주문'
   9    8               INDEX (RANGE SCAN DESCENDING) OF '주문_X1'
  10    4       TABLE ACCESS (BY INDEX ROWID BATCHED) OF '주문상품'
  11   10         INDEX (RANGE SCAN) OF '주문상품_PK'
  12    3     TABLE ACCESS (BY INDEX ROWID BATCHED) OF '상품이력'
  13   12       INDEX (RANGE SCAN) OF '상품이력_X2'
-----------------------------------------------------------
Predicate information:
   2 - access("배송번호"=:B1)
   6 - filter(ROWNUM<=10)
   9 - access("고객번호"='00000000042')
  11 - access("A"."주문번호"="B"."주문번호")
  13 - access("B"."상품번호"="C"."상품번호"(+) AND "A"."주문일시"<="C"."종료일시"(+) 
              AND "A"."주문일시">="C"."시작일시"(+))
```

```yml
--
/* 풀이 작성 */
--
```

##
##

```yml
2. 테이블 DDL과 수정 전 SQL을 참고하여, 수정 후 실행계획과 똑같이 나오도록 쿼리를 수정하시오.
- FROM 절의 테이블 순서는 a, b, c, d, e, f 순서를 유지하시오.
- 힌트를 사용하여 정확하게 의도한 실행계획(Hash Join 순서 및 Partition Iterator)이 나오도록 하시오.
```

```sql
-- 1. 주문 (Partitioned)
CREATE TABLE 주문 (
    주문번호 VARCHAR2(16) NOT NULL,
    고객번호 VARCHAR2(11) NOT NULL,
    주문일시 DATE,
    CONSTRAINT 주문_PK PRIMARY KEY (주문번호)
) PARTITION BY RANGE (주문번호) (
    PARTITION P202501 VALUES LESS THAN ('202502'),
    PARTITION P202502 VALUES LESS THAN ('202503'),
    -- ... (이하 생략)
);

-- 2. 주문상세 (Partitioned)
CREATE TABLE 주문상세 (
    주문번호 VARCHAR2(16) NOT NULL,
    상품번호 VARCHAR2(10) NOT NULL,
    CONSTRAINT 주문상세_PK PRIMARY KEY (주문번호)
) PARTITION BY RANGE (주문번호) (
    PARTITION P202501 VALUES LESS THAN ('202502'),
    -- ... (이하 생략)
);

-- 3. 고객, 상품, 코드상세, 주문통계 (일반 Table)
CREATE TABLE 고객 ( 고객번호 VARCHAR2(11) PRIMARY KEY, 고객명 VARCHAR2(20), 고객유형 VARCHAR2(5) );
CREATE TABLE 상품 ( 상품번호 VARCHAR2(10) PRIMARY KEY, 상품유형 VARCHAR2(5) );
CREATE TABLE 코드상세 ( 코드상세유형 VARCHAR2(5), 코드구분 VARCHAR2(5) );
CREATE TABLE 주문통계 ( 주문일시 DATE, 고객유형 VARCHAR2(5), 고객유형명 VARCHAR2(5), 상품유형 VARCHAR2(5), 상품유형명 VARCHAR2(5), 주문수량 NUMBER );

-- query(수정 전 SQL)
INSERT INTO 주문통계
SELECT TRUNC(a.주문일시) AS 주문일시, b.고객유형, e.코드상세유형 AS 고객유형명, 
       d.상품유형, f.코드상세유형 AS 상품유형명, COUNT(*) 주문수량
FROM   주문 a, 고객 b, 주문상세 c, 상품 d, 코드상세 e, 코드상세 f
WHERE  SUBSTR(a.주문번호, 1, 6) IN ('202501', '202502')
AND    b.고객번호 = a.고객번호
AND    c.주문번호 = a.주문번호
AND    d.상품번호 = c.상품번호
AND    e.코드구분(+) = 'A01'
AND    e.코드상세유형(+) = b.고객유형
AND    f.코드구분(+) = 'A02'
AND    f.코드상세유형(+) = d.상품유형
GROUP BY TRUNC(a.주문일시), b.고객유형, e.코드상세유형, d.상품유형, f.코드상세유형;

-- Execution Plan(수정 후 실행계획)
-----------------------------------------------------------
   0      SELECT STATEMENT
   1    0   HASH JOIN (RIGHT OUTER) 
   2    1     TABLE ACCESS (FULL) OF '코드상세' (F)
   3    1     HASH JOIN (RIGHT OUTER)
   4    3       TABLE ACCESS (FULL) OF '코드상세' (E)
   5    3       VIEW
   6    5         HASH (GROUP BY)
   7    6           HASH JOIN
   8    7             TABLE ACCESS (FULL) OF '고객' (B)
   9    7             HASH JOIN
  10    9               TABLE ACCESS (FULL) OF '상품' (D)
  11    9               PARTITION RANGE (ITERATOR)
  12   11                 HASH JOIN
  13   12                   TABLE ACCESS (FULL) OF '주문' (A)
  14   12                   TABLE ACCESS (FULL) OF '주문상세' (C)
-----------------------------------------------------------
-- Predicate: A.주문번호 >= '202501' (Range Iterator 조건)
-- Hash Join 순서: 코드상세가 Build Input (SWAP_JOIN_INPUTS 필요)
```

```yml
--
/* 풀이 작성 */
--
```

##
##

### 52회 실기 1번 정답 (예상안)
```sql
-- Index
CREATE INDEX 상품이력_x2 ON 상품이력(상품번호, 시작일시, 종료일시);

-- query
SELECT /*+ LEADING(A B C) USE_NL(B) USE_NL(C) INDEX(B 주문상품_PK) INDEX(C 상품이력_X2) */
       A.주문번호, 
       A.주문일시, 
       (B.주문수량 * C.상품금액) AS 주문금액,
       (SELECT /*+ NO_UNNEST */ 배송상태코드 
        FROM   배송 
        WHERE  배송번호 = A.배송번호) AS 배송상태코드
FROM (
    SELECT *
    FROM (
        SELECT /*+ INDEX_DESC(주문 주문_x1) NO_MERGE */ 
               주문번호, 주문일시, 배송번호
        FROM   주문
        WHERE  고객번호 = '00000000042'
        ORDER BY 주문일시 DESC
    )
    WHERE ROWNUM <= 10
) A, 주문상품 B, 상품이력 C
WHERE  B.주문번호 = A.주문번호
AND    C.상품번호(+) = B.상품번호
AND    A.주문일시 >= C.시작일시(+)
AND    A.주문일시 <= C.종료일시(+)
AND    C.이벤트명(+) = 'SQLP';
```


## 52회 실기 2번 정답 (예상안)
```
INSERT INTO 주문통계
SELECT /*+ LEADING(A E F) USE_HASH(E F) SWAP_JOIN_INPUTS(E) SWAP_JOIN_INPUTS(F) */
       A.주문일시, A.고객유형, E.코드상세유형, A.상품유형, F.코드상세유형, A.주문수량
FROM (
    SELECT /*+ NO_MERGE LEADING(A C D B) USE_HASH(C D B) SWAP_JOIN_INPUTS(D) SWAP_JOIN_INPUTS(B) */
           TRUNC(A.주문일시) AS 주문일시, B.고객유형, D.상품유형, COUNT(*) AS 주문수량
    FROM   주문 A, 고객 B, 주문상세 C, 상품 D
    WHERE  A.주문번호 >= '202501' AND A.주문번호 < '202503'
    AND    A.주문번호 = C.주문번호
    AND    C.상품번호 = D.상품번호
    AND    A.고객번호 = B.고객번호
    GROUP BY TRUNC(A.주문일시), B.고객유형, D.상품유형
) A, 코드상세 E, 코드상세 F
WHERE A.고객유형 = E.코드상세유형(+)
AND   E.코드구분(+) = 'A01'
AND   A.상품유형 = F.코드상세유형(+)
AND   F.코드구분(+) = 'A02';
```

