### DB장이님의 풀이, 근데 이제 gpt를 곁들인..
```
50회 기출문제 정리
실기 1번은 소문제로 5문제 ->옵티마이저 실행게획보고 튜닝 (1~4는 CH3, 5는 CH6)
실기 2번은 소문제로 3문제 ->로깅 (CH6)
```
## 1. 실행 계획을 보고 문제점과 개선점을 각각 적으시오. <br>
1-1) <br>
```sql
| Id | Operation         | Name | Rows | Bytes |
| -- | ----------------- | ---- | ---- | ----- |
| 0  | SELECT STATEMENT  |      | 14   | 100k  |
| 1  |  TABLE ACCESS FULL | EMP  | 14   | 100k  |
```

1-2) <br>
```sql
| Id | Operation                           | Name    | Rows | Bytes |
| -- | ----------------------------------- | ------- | ---- | ----- |
| 0  | SELECT STATEMENT                    |         | 100  | 532   |
| 1  |  TABLE ACCESS BY INDEX ROWID BATCHED | EMP     | 100  | 532   |
| *2 |   INDEX RANGE SCAN                    | EMP_X01 | 1000 |       |
```

1-3) <br>
```sql
| Id | Operation                           | Name    | Rows | Bytes |
| -- | ----------------------------------- | ------- | ---- | ----- |
| 0  | SELECT STATEMENT                    |         | 2k   | 1010  |
| 1  |  TABLE ACCESS BY INDEX ROWID BATCHED | EMP     | 2k   | 1010  |
| *2 |   INDEX RANGE SCAN                    | EMP_X03 | 2k   | 1000  |
```

1-4) <br>
```sql
| Id | Operation                           | Name    | Rows | Bytes |
| -- | ----------------------------------- | ------- | ---- | ----- |
| 0  | SELECT STATEMENT                    |         | 2k   | 1000  |
| 1  |  TABLE ACCESS BY INDEX ROWID BATCHED | EMP     | 2k   | 1000  |
| *2 |   INDEX RANGE SCAN                    | EMP_X03 | 2k   | 10    |
```

1-5) <br>
```sql
| Id | Operation                                 | Name        | Rows | Bytes | Pstart |
| -- | ----------------------------------------- | ----------- | ---- | ----- | ------ |
| 0  | SELECT STATEMENT                          |             | 1    | 87    |        |
| 1  |  PARTITION RANGE ALL                       |             | 1    | 87    | 1      |
| 2  |   TABLE ACCESS BY LOCAL INDEX ROWID BATCHED | EMP_PART    | 1    | 87    | 1      |
| *3 |    INDEX RANGE SCAN                          | EMP_PART_X2 | 1    |       | 1      |
```

<br>

## 2. 로깅을 최소화하도록 튜닝하라.

2-1) <br>
```sql
insert into t1 select * from t2
```

2-2) <br>
```sql
-- c1, c2 는 모두 NOT NULL
UPDATE t1
SET c2 = CASE
           WHEN c1 < TRUNC(ADD_MONTHS(SYSDATE, 2))
           THEN 'Y'
           ELSE c2
         END;
```

2-3) <br>
```sql
-- t1의 특정 파티션(202402)에서 c2='Y'인 데이터를 삭제
-- (전체의 약 95%)
DELETE FROM t1 PARTITION (202402)
WHERE c2 = 'Y';
```

## 풀이
1-1)
```
소량의 결과(14건)를 조회함에도 불구하고 EMP 테이블에 대해 TABLE ACCESS FULL이 발생하여 대량의 블록을 읽고 있다.
이는 조건 컬럼에 적절한 인덱스가 존재하지 않기 때문이다.
해당 컬럼에 인덱스를 생성하면 옵티마이저가 인덱스 스캔을 선택하여 I/O 비용과 Bytes를 감소시킬 수 있다.
```

1-2)
```
현재 실행계획은 인덱스를 사용하고 있으나, 인덱스에서 과도한 범위 스캔(1000건)이 발생한 후 일부 데이터만 테이블에서 조회되고 있다.
이는 인덱스 구성 컬럼만으로는 조건을 충분히 필터링하지 못하기 때문이다.
WHERE 절 조건 컬럼을 포함한 복합 인덱스를 생성하여 인덱스 단계에서 선택도를 높이면 랜덤 I/O를 감소시킬 수 있다.
```

1-3)
```
EMP_X03 인덱스를 사용하고 있으나, 조건에 의해 인덱스 스캔 건수(2k)가 결과 건수와 동일하여 선택도가 낮다.
이로 인해 테이블 접근이 다량 발생하므로, FULL TABLE SCAN이 더 효율적일 수 있다.
```

1-4)
```
EMP_X03 인덱스는 선택도가 높고 컬럼 구성도 적절하여 인덱스 스캔 자체는 효율적이다.
그러나 테이블 접근 시 다량의 랜덤 I/O가 발생하고 있으며, 이는 테이블과 인덱스 간 클러스터링 팩터가 좋지 않기 때문으로 판단된다.
테이블 재구성(리오그) 또는 인덱스 재생성을 통해 클러스터링 팩터를 개선할 수 있다.
```

1-5)
```
EMP_PART_X2는 로컬 파티션 인덱스이나, 실행계획에서 PARTITION RANGE ALL이 발생하여 모든 파티션을 탐색하고 있다.
이는 파티션 키 조건이 없거나, 파티션 Pruning이 불가능한 조건식으로 인해 로컬 인덱스의 장점을 활용하지 못한 경우로 판단된다.
해당 쿼리는 파티션 기반 접근이 어렵기 때문에, 글로벌 인덱스를 사용하는 것이 보다 효율적인 접근 방식이 될 수 있다.
```

2-1)
```
ALTER TABLE t1 NOLOGGING;

INSERT /*+ APPEND */
INTO t1
SELECT * FROM t2;
```

2-2)
```
UPDATE t1
SET    c2 = 'Y'
WHERE  c1 < TRUNC(ADD_MONTHS(SYSDATE, 2))
AND    c2 <> 'Y';
```

2-3)
```
-- 5%만 임시 저장
INSERT /*+ APPEND */ INTO tmp
SELECT * FROM t1 PARTITION (202402)
WHERE c2 <> 'Y';

-- 파티션 비우기
ALTER TABLE t1 TRUNCATE PARTITION (202402);

-- 다시 적재
INSERT /*+ APPEND */ INTO t1
SELECT * FROM tmp;
```
