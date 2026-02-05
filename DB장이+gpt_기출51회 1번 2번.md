```yml
51회 기출문제 배경 지식
실기 1번
TOP-N: ROW_NUMBER() OVER (ORDER BY 주문일시 DESC) → WINDOW SORT PUSHED RANK → 2-1
조건 필터: 주문일시 >= SYSDATE-1/24 → INDEX RANGE SCAN → 3-2
집계: COUNT(*) / GROUP BY b.상품번호, a.주문일시 / HAVING COUNT(*)>=2 → HASH GROUP BY → 3-3
JOIN: 주문→주문상품→상품 → NESTED LOOPS + INDEX SCAN → 3-2~3
정렬: ORDER BY 합계주문수량 DESC → SORT ORDER BY → 2-1
힌트: /*+ leading(a) index(a) use_nl(b) */ → 옵티마이저 경로 제어 → 3-2

----------------------------------

실기 2번
주문일자 변환: TO_CHAR(주문일시,'YYYYMMDD') → GROUP BY → 2-1
집계: COUNT(*) → HASH GROUP BY → 3-3
조인: 주문→주문상세→고객 → HASH JOIN → 4-3
조건 필터: 주문일시 >= '20240901' AND <= '20240930' → PARTITION RANGE SCAN → 3-2
INSERT: INSERT INTO 주문통계정보 → 결과 저장 → 2-1
힌트: /*+ use_hash */ → 옵티마이저 경로 제어 → 4-3
```

## 실기 1번
```yml
1. 테이블 및 인덱스를 참고하여 실행계획이 똑같이 나오게 쿼리를 작성하시오. (힌트는 실행계획이 똑같이 되게)
주문일시가 1시간 이내의 최근주문 1000건 중 2번이상 주문된 상품명과 주문합계수량,주문일시를
쿼리로 합계주문수량 내림차순으로 정렬하시오.
```

```sql
--query
1)
CREATE TABLE YSBAE."상품"
(
    "상품번호"  NUMBER,
    "상품명"    VARCHAR2(10),
    "고객번호"  NUMBER
)
TABLESPACE USERS
NOCOMPRESS;

ALTER TABLE YSBAE."상품"
ADD PRIMARY KEY ("상품번호");

2)
CREATE TABLE YSBAE."주문1"
(
    "주문번호"  NUMBER,
    "주문일시"  DATE
)
TABLESPACE USERS
NOCOMPRESS;

ALTER TABLE YSBAE."주문1"
ADD PRIMARY KEY ("주문번호");

create index 주문_x01 on 주문1(주문일시);

3)
CREATE TABLE YSBAE."주문상품"
(
    "주문번호"  NUMBER,
    "상품번호"  NUMBER
)
TABLESPACE USERS
STORAGE
(
    INITIAL 64K
    NEXT 1M
)
NOCOMPRESS;

ALTER TABLE YSBAE."주문상품"
ADD PRIMARY KEY ("주문번호");

-- 테스트 데이터 1000건씩 생성
insert into 상품
select abs(dbms_random.random) as 상품번호,dbms_random.string('A',10),mod(level,10) from dual connect by level <= 1000;
commit;

insert into 주문1
select abs(dbms_random.random) as 주문번호,(sysdate - level/3600) as tt from dual connect by level <= 1000;
commit;


insert into 주문상품
with tmp_상품 as(
select 상품번호,row_number() over(order by 상품번호) as rn
from 상품
where 상품번호 is not null
),
tmp_주문 as(
select 주문번호,row_number() over(order by 주문번호) as rn
from 주문1
where 주문번호 is not null
)
select a.주문번호,b.상품번호
from tmp_주문 a,tmp_상품 b
where a.rn = b.rn;

commit;


--Execution Plan
--------------------------------------------------------------------
| Id  | Operation                                   | Name         |
--------------------------------------------------------------------
|   0 | SELECT STATEMENT                            |              |
|   1 |  TABLE ACCESS BY INDEX ROWID                | 상품         |  
|*  2 |   INDEX UNIQUE SCAN                         | SYS_C0081814 |
|   3 |  SORT ORDER BY                              |              |
|   4 |   HASH GROUP BY                             |              |
|   5 |    NESTED LOOPS                             |              |
|   6 |     NESTED LOOPS                            |              |
|   7 |      VIEW                                   |              |
|   8 |       HASH GROUP BY                         |              |
|*  9 |        VIEW                                 |              |
|* 10 |         WINDOW SORT PUSHED RANK             |              |
|  11 |          TABLE ACCESS BY INDEX ROWID BATCHED| 주문1        |  
|* 12 |           INDEX RANGE SCAN                  | 주문_X01     |  
|* 13 |      INDEX RANGE SCAN                       | 주문상품_X1  |  
|  14 |     TABLE ACCESS BY INDEX ROWID             | 주문상품     |   
--------------------------------------------------------------------
Predicate Information (identified by operation id):
---------------------------------------------------
 
   2 - access("상품번호"=:B1)
   9 - filter("RN"<=1000)
  10 - filter(ROW_NUMBER() OVER ( ORDER BY INTERNAL_FUNCTION("주문일시") DESC )<=1000)
  12 - access("주문일시">=SYSDATE@!-.0416666666666666666666666666666666666667)
  13 - access("A"."주문번호"="B"."주문번호")
```

```
--
/* 풀이작성란 */
--
```

<br><br>

## 실기 2번
```yml
2. 테이블 ,인덱스를 참고하여 실행계획이 똑같이 나오게 쿼리를 작성하시오. (힌트는 실행계획이 똑같이 되게)
성별,주문일자,상품번호별 주문 수량을 구하여 주문통계정보 insert하기
- 주문상세의 주문수량은 문제에서 요구하는 주문수량과 관계없음
```

```sql
-- query
create table 주문
(주문번호 varchar2(20),
 순번 number,
 주문일시 date
)
partition by range(주문번호)
(
partition 주문_p1 values less than ('20240631'),
partition 주문_p2 values less than ('20240730'),
partition 주문_p3 values less than ('20240831'),
partition 주문_p4 values less than ('20240930'),
partition 주문_max values less than (maxvalue)
);

create index 주문_pk on 주문(주문번호) local;


create table 주문상세
(주문번호 varchar2(20),
 주문수량 number, --## 이건 사용 x
 상품번호 number,
 고객번호 number,
 주문일시 date
)
partition by range (주문번호)
(
partition 주문상세_p1 values less than ('20240631'),
partition 주문상세_p2 values less than ('20240730'),
partition 주문상세_p3 values less than ('20240831'),
partition 주문상세_p4 values less than ('20240930'),
partition 주문상세_max values less than (maxvalue)
);

create index 주문상세_pk on 주문상세(주문번호) local;

create table 고객
(고객번호 number,
성별 varchar2(10)
);


create table 주문통계정보
(성별 varchar2(10),
 주문일자 varchar2(8),
 상품번호 number,
 합계주문수량 varchar2(10)
 );
 

-- 각 테이블변 1000건씩 데이터 생성
 insert into 주문
select to_char(sysdate - 2/level,'YYYYMMDDHH24MISS')||level,level as 순번,sysdate - level/24 from dual connect by level <= 1000;

insert into 주문상세
select 주문번호,round(dbms_random.value(0,10)) 주문수량,abs(dbms_random.random) as 상품번호,abs(dbms_random.random) as 고객번호,주문일시 from 주문;

commit;

insert into 고객
select 고객번호,substr((case when mod(고객번호,2) = 0 then '남' else '여' end),0,1) from 주문상세;

commit;


-- Execution Plan
---------------------------------------------------
| Id  | Operation                       | Name    |
---------------------------------------------------
|   0 | SELECT STATEMENT                |         |
|   1 |  HASH GROUP BY                  |         |
|*  2 |   FILTER                        |         |
|*  3 |    HASH JOIN                    |         |
|   4 |     PART JOIN FILTER CREATE     | :BF0000 |
|   5 |      PARTITION RANGE ALL        |         |
|*  6 |       TABLE ACCESS FULL         | 주문    |  
|*  7 |     HASH JOIN                   |         |
|   8 |      TABLE ACCESS FULL          | 고객    |  
|   9 |      PARTITION RANGE JOIN-FILTER|         |
|* 10 |       TABLE ACCESS FULL         | 주문상세|    
---------------------------------------------------
Predicate Information (identified by operation id):
---------------------------------------------------
 
   2 - filter(TO_DATE('20240930')>=TO_DATE('20240901'))
   3 - access("A"."주문번호"="B"."주문번호")
   6 - filter(("A"."주문일시">='20240901' AND "A"."주문일시"<='20240930'))
   7 - access("B"."고객번호"="D"."고객번호")
  10 - filter(("B"."주문일시">='20240901' AND "B"."주문일시"<='20240930'))

```

```
--
/* 풀이작성란 */
--
```

### 1번 정답(예상안)
```sql
--query
SELECT
    p.상품명,
    COUNT(*) AS 주문합계수량,
    MAX(o.주문일시) AS 주문일시
FROM (
        SELECT *
        FROM (
            SELECT
                주문번호,
                주문일시,
                ROW_NUMBER() OVER (ORDER BY 주문일시 DESC) AS rn
            FROM 주문1
            WHERE 주문일시 >= SYSDATE - 1/24
        )
        WHERE rn <= 1000
     ) o
JOIN 주문상품 op
  ON o.주문번호 = op.주문번호
JOIN 상품 p
  ON op.상품번호 = p.상품번호
GROUP BY p.상품명
HAVING COUNT(*) >= 2
ORDER BY 주문합계수량 DESC;
```

### 2번 정답 (예상안)
```sql
-- query
INSERT INTO 주문통계정보
SELECT
    k.성별,
    TO_CHAR(o.주문일시,'YYYYMMDD') AS 주문일자,
    od.상품번호,
    COUNT(*) AS 합계주문수량
FROM 주문 o
JOIN 주문상세 od
  ON o.주문번호 = od.주문번호
JOIN 고객 k
  ON od.고객번호 = k.고객번호
WHERE o.주문일시 >= TO_DATE('20240901','YYYYMMDD')
  AND o.주문일시 <= TO_DATE('20240930','YYYYMMDD')
  AND od.주문일시 >= TO_DATE('20240901','YYYYMMDD')
  AND od.주문일시 <= TO_DATE('20240930','YYYYMMDD')
GROUP BY
    k.성별,
    TO_CHAR(o.주문일시,'YYYYMMDD'),
    od.상품번호;
```
