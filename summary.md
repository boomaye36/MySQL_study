# summary

# 3장.

## MySQL 엔진 구조

![Untitled](summary%2074ff2959a0a342ee8b4d4afd854cffee/Untitled.png)

- 클라이언트로부터의 접속 및 쿼리 요청을 처리하는 커넥션 핸들러
- 요청받은 쿼리문을 실행하기 위해 파싱이라는 과정 진행 (쿼리 캐시에서 이전 실행 쿼리 체크)
- 캐시에 없을 경우 최적화된 쿼리 실행을 위한 옵티마이저
- 스토리지 엔진에서는 실행엔진에서 받은 요청에 따라 데이터를 디스크로 저장하고 디스크로부터 읽어온다.
    
    자주 사용하는 데이터는 버퍼 풀에 저장한다. 
    

### MySQL 서버는 프로세스 기반이 아니라 스레드 기반

![Untitled](summary%2074ff2959a0a342ee8b4d4afd854cffee/Untitled%201.png)

- 포그라운드 스레드 (클라이언트)
    - 최소한 서버에 접속된 클라이언트의 수만큼 존재, 각 클라이언트 사용자가 요청하는 쿼리 문장 처리
    - 일정 숫자만 캐시 존재
- 백그라운드 스레드
    - 인서트 버퍼를 병합하는 스레드
    - 로그를 디스크로 기록하는 스레드
    - InnoDB 버퍼 풀의 데이터를 디스크에 기록하는 스레드
    - 데이터를 버퍼로 읽어들이는 스레드
    - 여러가지 잠금이나 데드락을 모니터링하는 스레드
    
- 얼마나 많은 스레드가 공유하느냐에 따라 글로벌 / 로컬 메모리 영역으로 나뉨

## InnoDB

레코드 기반의 잠금 - 높은 동시성 처리, 안정적, 성능이 뛰어남

- 프라이머리 키에 의한 클러스터링
- 잠금이 필요 없는 일관된 읽기 - MVCC 이용(락을 걸지 않고 읽기 작업 수행) → 필요에 따라 보여지는 데이터가 달라짐
- 자동 데드락 감지
- InnoDB 버퍼 풀 : 디스크의 데이터 파일이나 인덱스 정보를 메모리에 캐시, 쓰기 작업을 지연시켜 일괄 작업으로 처리할 수 있게 해주는 버퍼 역할도 수행

### 언두로그와 리두로그

- 언두로그 : 데이터를 변경했을 때 변경되기 이전의 데이터를 보관하는 곳
- 리두로그 : 변경된 내용을 순차적으로 디스크에 기록하는 로그 파일

# 4장

## 트랜잭션

하나의 논리적인 작업 셋에 쿼리의 개수에 상관 없이 논리적인 작업 셋 자체가 모두 적용되거나 적용되지 않아야 함을 보장해 주는 것

- 트랜잭션을 처리 할 때 네트워크를 통해 원격 서버와 통신하는 등과 같은 작업은 트랜잭션 내에서 제거하는 것이 좋다.
- 처리 절차에서 묶어야 하는 작업을 한 트랜잭션으로 만든다.

- 트랜잭션 특징
    - 원자성 : 트랜잭션이 데이터베이스에 모두 반영되던가 아니면 전혀 반영되지 않아야 한다.
    - 일관성 : 트랜잭션의 작업 처리 결과가 항상 일관성이 있어야 한다.
    - 독립성 : 둘 이상의 트랜잭션이 동시에 실행되고 있을 경우 어떤 하나의 트랜잭션이라도 다른 트랜잭션의 연산에 끼어들 수 없다.
    - 지속성 : 트랜잭션이 성공적으로 완료됐을 경우 결과는 영구적으로 반영돼야 한다.

```sql
-- 트랜잭션 시작
BEGIN;

-- 이체를 수행하는 SQL 문
UPDATE accounts
SET balance = balance - 500
WHERE account_number = '123456'; -- 송금하는 계좌

UPDATE accounts
SET balance = balance + 500
WHERE account_number = '789012'; -- 수신하는 계좌

-- 트랜잭션 커밋(데이터 변경을 영구적으로 반영)
COMMIT;

-- 또는 트랜잭션 롤백(트랜잭션을 취소하고 이전 상태로 되돌림)
-- ROLLBACK;
```

## MySQL의 격리 수준

격리 수준을 설정함으로써 트랜잭션의 격리 수준과 동시성 제어를 조정할 수 있지만, 높은 격리 수준을 선택할수록 성능 저하와 병목 현상에 대해 고려해야 한다. 

### READ UNCOMMITTED

각 트랜잭션에서의 변경 내용이 COMMIT이나 ROLLBACK 여부에 상관 없이 다른 트랜잭션에서 보여진다. 

- 어떤 트랜잭션에서 처리한 작업이 완료되지 않았는데 다른 트랜잭션에서 볼 수 있게 되는 현상을 더티리드라한다.

```sql
-- 트랜잭션 1
BEGIN;
-- 학생 정보 조회
SELECT * FROM students WHERE student_id = '123';
-- 결과: { student_id: '123', name: 'John', age: 20 }

-- 트랜잭션 2에서 동일한 학생 정보를 변경한다고 가정
```

```sql
-- 트랜잭션 2
BEGIN;
-- 학생 정보 변경
UPDATE students SET age = 21 WHERE student_id = '123';
```

```sql
-- 이제 사용자 A가 다시 학생 정보 조회를 수행
SELECT * FROM students WHERE student_id = '123';
-- 결과: { student_id: '123', name: 'John', age: 21 }

-- 사용자 A의 트랜잭션이 롤백되면, 나이는 다시 20으로 돌아감
```

### READ COMMITTED

- 더티 리드 같은 현상은 발생하지 않는다.
- 트랜잭션에서 변경한 내용이 커밋되기 전까지는 다른 트랜잭션에서 그러한 변경 내역을 조회할 수 없다.
- NON-REAPEATABLE READ가 발생가능하다. → 하나의 트랜잭션에서 같은 값을 조회할 때 다른 값이 조회됨

```sql
BEGIN;
-- 상품의 재고 조회
SELECT stock FROM products WHERE product_id = '1001';
-- 결과: stock = 50 (예시로 재고가 50개라고 가정)

-- 트랜잭션 2에서 동일한 상품 재고를 변경한다고 가정
```

```sql
BEGIN;
-- 상품 재고 변경
UPDATE products SET stock = 30 WHERE product_id = '1001';
```

```sql
-- 이제 사용자 A가 다시 상품 재고 조회를 수행
SELECT stock FROM products WHERE product_id = '1001';
-- 결과: stock = 30 (Non-Repeatable Read가 발생하여 재고가 변경된 30개로 조회됨)

-- 사용자 A의 트랜잭션이 롤백되면, 재고는 다시 50개로 돌아갈 수 있음
```

### REAPEATABLE READ

- MVCC를 위해 언두 영역에 백업된 이전 데이터를 이용해 동일 트랜잭션 내에서는 동일한 결과를 보여줄 수 있도록 보장한다.
- 자신의 트랜잭션보다 작은 번호에서 변경한 것만 적용된다.
- 다른 트랜잭션에서 수행한 변경 작업에 의해 레코드가 보였다 안보였다 하는 현상인 PHANTOM READ가 발생할 수 있다.  → 값 자체는 같게 나오지만 레코드의 수가 다르게 나옴

```sql
BEGIN;
-- "Electronics" 카테고리에 속한 상품 개수 조회
SELECT COUNT(*) FROM products WHERE category = 'Electronics';
-- 결과: 10 (예시로 "Electronics" 카테고리에 10개의 상품이 있다고 가정)

-- 트랜잭션 2에서 동일한 카테고리에 새로운 상품을 추가한다고 가정
```

```sql
BEGIN;
-- "Electronics" 카테고리에 새로운 상품 추가
INSERT INTO products (product_id, category) VALUES ('101', 'Electronics');
```

```sql
-- 이제 사용자 A가 다시 "Electronics" 카테고리에 속한 상품 개수 조회를 수행
SELECT COUNT(*) FROM products WHERE category = 'Electronics';
-- 결과: 11 (Phantom Read가 발생하여 상품 개수가 10에서 11로 증가함)

-- 사용자 A의 트랜잭션이 커밋되거나 롤백되면, 상품 개수는 다시 10개로 돌아갈 수 있음
```

### SERIALIZABLE

가장 높은 격리 수준 - 동시성이 떨어짐

## MySQL 엔진의 잠금

MySQL 잠금은 스토리지 엔진 레벨과 MySQL 엔진 레벨로 나뉜다. MySQL 엔진 레벨의 잠금은 모든 스토리지 엔진에 영향을 미치게 되지만 스토리지 엔진 레벨의 잠금은 스토리지 엔진 간 상호 영향을 미치지는 않는다. 

### 테이블 락

개별 테이블 단위로 설정되는 잠금이다. 

### 유저 락

사용자가 지정한 문자열에 대해 반납, 해제 하는 잠금이다. 

### 네임 락

데이터베이스 객체의 이름을 변경하는 경우 획득하는 락

### 잠금 튜닝

잠금 대기 쿼리 비율이 높으면 테이블 잠금 때문에 경합이 많이 발생하고 있으면 처리 성능이 영향을 받고 있음을 의미하므로 테이블을 분리하거나 InnoDB 스토리지 엔진으로 변환하는 방법을 고려하는 것이 좋다.

### InnoDB 스토리지 엔진의 잠금

MySQL에서 제공하는 잠금과는 별개로 스토리지 엔진 내부에서 레코드 기반의 잠금 방식을 탑재하고 있다. 

- InnoDB는 레코드 자체를 잠그는 것이 아니라 인덱스의 레코드를 잠근다.
- Gap lock : 레코드와 레코드 사이의 간격에 새로운 레코드가 생성 되는 것을 제어하는 것.
- 넥스트 키 락 : 레코드 락과 갭 락을 합쳐놓은 형태의 잠금
- 자동 증가 락 : AUTO_INCREMENT시 테이블 수준의 락

# 5장

## B-Tree

칼럼의 원래 값을 변형시키지 않고 인덱스 구조체 내에서는 항상 정렬된 상태로 유지 → 키 변경하는 데 비용이 많이 들지만 빠른 탐색이 가능

![Untitled](summary%2074ff2959a0a342ee8b4d4afd854cffee/Untitled%202.png)

- 인덱스를 통해 읽어야 할 레코드의 건수가 전체 테이블 레코드의 20 ~ 25% 를 넘어서면 인덱스를 이용하지 않는다.
- 인덱스의 키 값은 정렬되어 있지만 데이터 파일의 레코드는 임의의 순서대로 저장돼 있다.

### 인덱스 레인지 스캔

- 검색해야할 인덱스의 범위가 주어졌을 때 사용.
- 가장 빠른 방식

### 인덱스 풀 스캔

- 인덱스의 처음부터 끝까지 모두 읽는 방식
- 쿼리가 인덱스에 명시된 컬럼만으로 조건을 처리할 수 있는경우

### 루스 인덱스 스캔

![Untitled](summary%2074ff2959a0a342ee8b4d4afd854cffee/Untitled%203.png)

- 중간마다 필요치 않은 인덱스 키값은 무시하고
    
     다음으로 넘어간다. 
    
- GROUP BY, MAX, MIN 사용

인덱스가 다중 칼럼으로 구성되어 있다면 한 칼럼 없이는 다른 칼럼을 정렬할 수 없다.

```sql
select dept_no, MIN(emp_no)
from dept_emp
where dept_no between 'doo2' and 'd004'
group by dept_no;
```

### 다중 칼럼 인덱스

![Untitled](summary%2074ff2959a0a342ee8b4d4afd854cffee/Untitled%204.png)

- 인덱스의 두 번째 칼럼은 첫 번째 칼럼에 의존해서 정렬되어 있다. → 두 번째 칼럼은 첫 번쨰 칼럼이 똑같은 레코드에서만 의미가 있다.
- 칼럼의 순서가 신중하게 결정되어야 한다. → 개수가 적은 데이터를 조회하는 칼럼을 인덱스 앞 쪽에 설정하고 개수가 많은 데이터를 조회하는 칼럼을 뒤쪽에 설정해야 한다.

```sql
SELECT * FROM dept_emp
WHERE dept_no = 'd002' 
AND emp_no >= 10114;
```

⇒ 데이터를 조회할 때 단일 인덱스를 여러 개 사용해야 하는 경우가 많을 때 사용

## 인덱스를 효율적으로 쓰지 못하는 상황

- NOT-EQUAL 로 비교된 경우
- LIKE ‘%??’ 형태로 문자열 패턴이 비교된 경우
- 스토어드 함수나 다른 연산자로 인덱스 칼럼이 변형된 후 비교된 경우
- NOT-DETERMINISTIC 속성의 스토어드 함수가 비교 조건에 사용된 경우
- 데이터 타입이 서로 다른 비교
- 문자열 데이터 타입의 콜레이션이 다른 경우
- 작업 범위 결정 조건으로 인덱스를 사용하지 못하는 경우
- 작업 범위 결정 조건으로 인덱스를 사용하는 경우

# 6장

### select_type 칼럼

- simple : UNION 이나 서브 쿼리를 사용하지 않는 단순한 SELECT 쿼리인 경우
- primary : UNION 이나 서브 쿼리가 포함된 SELECT 쿼리의 실행 계획에서 가장 바깥쪽에 있는 단위 쿼리
- UNION : UNION으로 결합하는 단위 SELECT 쿼리 가운데 첫 번째를 제외한 두 번째 이후 단위 SELECT 쿼리의 select_type
    
    ```sql
    EXPLAIN
    SELECT * FROM (
    (SELECT emp_no FROM employees e1 LIMIT 10)
    UNION ALL
    (SELECT emp_no FROM employees e2 LIMIT 10)
    UNION ALL
    (SELECT emp_no FROM employees e3 LIMIT 10)
    ) tb;
    ```
    
- DEPENDENT UNION : 내부 쿼리가 외부의 값을 참조해서 처리될 때 → 외부 쿼리에 의존적이므로 비효율적인 경우가 많다.
- UNION RESULT : UNION 결과를 담아두는 테이블  - UNION RESULT <union 2, 3, 4> : 2, 3, 4 단위의 쿼리가 UNION 되었다.
- SUBQUERY : FROM 절 이외에서 사용되는 서브쿼리
- DEPENDENT SUBQUERY : 서브 쿼리가 바깥쪽 SELECT 쿼리에서 정의된 칼럼을 사용하는 경우
- DERIVED : 서브 쿼리가 FROM 절에 사용된 경우, 파생 테이블에는 인덱스가 없으므로 다른 테이블과 조인할 때 성능상 불리하다. → 가능하다면 조인으로 바꿔주는 것이 좋다.
- UNCACHEABLE SUBQUERY
    - subquery는 바깥쪽의 영향을 받지 않으므로 처음 한 번만 실행해서 그 결과를 캐시하고 필요할 때 캐시된 결과를 이용한다.
    - DEPENDENT SUBQUERY는 의존하는 바깥쪽 쿼리의 칼럼의 값 단위로 캐시하고 사용한다.
- UNCACHEABLE UNION : UNION과 UNCACHEABLE 혼합

![Untitled](summary%2074ff2959a0a342ee8b4d4afd854cffee/Untitled%205.png)

### 

## table 칼럼

MySQL 의 실행 계획은 단위 SELECT 쿼리 기준이 아니라 테이블 기준으로 표시된다. 

- Table 칼럼에 <>로 둘러싸인 이름이 명시 되는 경우가 있는데 이 테이블은 임시 테이블을 의미한다.
- MySQL은 Derived는 반드시 별칭을 가져야 한다.

## type 칼럼

type 이후의 칼럼은 MySQL 서버가 각 테이블의 레코드를 어떤 방식으로 읽었는지를 의미한다. 이는 인덱스를 사용해 레코드를 읽었는지 처음부터 끝까지 읽는 풀 테이블 스캔으로 읽었는지를 의미한다.

system

레코드가 1건만 존재하는 테이블 또는 한 건도 존재하지 않는 테이블을 참조하는 형태의 접근 방법을 system이라고한다. 

## const

테이블의 레코드의 건수에 관계없이 쿼리가 프라이머리 키나 유니크 키 칼럼을 이용하는 WHERE 조건절을 가지고 있으며 반드시 1건을 반환하는 쿼리의 처리 방식이다. 

프라이머리 키의 일부만 조건으로 사용할 때는 ref으로 표시된다. 

### eq_ref

여러 테이블이 조인되는 쿼리의 실행 계획에서 표시.

조인에서 처음 읽은 테이블의 칼럼 값을, 그 다음 읽어야 할 테이블의 프라이머리 키나 유니크 키 칼럼의 검색 조건에 서용 할 때. 

조인에서 두 번째 이후에 읽는 테이블에서 반드시 1건만 존재한다는 보장이 있어야 한다.

```sql
EXPLAIN
SELECT *. FROM dept_emp de, employee e
WHERE e.emp_no = de.emp_no AND de.dept_no = 'd005';
```

### ref

 조인의 순서와 상관없이 사용, 프라이머리 키나 유니크 키 등의 제약 조건도 없다. 

인덱스의 종류와 관계없이 동등 조건으로 검색할 때 사용 (검색 속도 빠른편)

### full text

MySQL 전문검색 인덱스를 사용해 레코드를 읽는 접근 방법. 

- 전문 검색은 MATCH … AGAINST … 구문을 사용하는데 반드시 해당 테이블에 전문 검색용 인덱스가 준비돼 있어야한다.
- 전문 인덱스보다 일반 인덱스를 이용하는 range 방식이 접근이 빠른 편

### unique_subquery

WHERE 조건절에서 사용할 수 있는 IN 형태의 접근 방식이다. 

서브쿼리에서 중복되지 않은 유니크한 값만 반환한다. 

### index_subquery

IN 형태의 조건에서 subquery의 반환 값에 중복된 값이 있을 수 있지만 인덱스를 이용해 중복된 값을 제거할 수 있음

### Range

인덱스를 하나의 값이 아니라 범위로 검색

인덱스 레인지 스캔은 const, ref, range 세 가지 접근 방식을 모두 묶어서 칭하는 것이다. 

### index merge

2개 이상의 인덱스를 이용해 각각의 검색 결과를 만들어 낸 후 그 결과를 병합 처리

- 여러 인덱스를 읽어야 하므로 일반적으로 range 접근 방식보다 효율성이 떨어진다.
- AND와 OR 연산이 복잡하게 연결된 쿼리에서는 제대로 최적화되지 못할 때가 많다.
- 전문 검색 인덱스를 사용하는 쿼리에서는 index_merge가 적용되지 않는다.
- index_merge 접근 방식으로 처리된 결과는 항상 2개 이상의 집합이 되기 때문에 그 두 집합의 교집합이나 합집합 또는 중복 제거와 같은 부가적인 작업이 더 필요하다.

### index

index접근 방식은 인덱스를 처음부터 끝까지 읽는 인덱스 풀 스캔 방식이다. 

- 인덱스는 일반적으로 데이터 파일 전체보다는 크기가 작고 인덱스는 정렬되어 있으므로 풀 테이블 스캔보다는 빠르게 처리된다.
- 사용 조건
    - range나 const 또는 ref와 같은 접근 방식으로 인덱스를 사용하지 못하는 경우
    - 인덱스에 포함된 칼럼만으로 처리할 수 있는 쿼리인 경우
    - 인덱스를 이용해 정렬이나 그루핑 작업이 가능한 경우
- limit과 같이 추가 제약조건이 들어가면 효율적이다.

### All

풀 테이블 스캔을 의미 테이블을 처음부터 끝까지 전부 읽어서 불필요한 레코드를 제거하고 반환한다. 

가장 비효율적인 방법 

쿼리를 튜닝한다는 것이 무조건 INDEX, ALL을 사용하지 못하게 하는 것은 아니다.  

## Key

최종 선택된 실행 계획에서 사용하는 인덱스

index_merge 가 아닌 경우에는 반드시 테이블 하나당 하나의 인덱스만 이용할 수 있다. 

## Key_len

인덱스의 각 레코드에서 몇 바이트까지 사용했는지 알려줌

```sql
SELECT * FROM dept_emp WHERE dept_no = 'd005' AND emp_no = 10001;
```

emp_no (4바이트) + dept_no (4바이트 * 3) = 16

## ref

접근 방법이 ref 방식이면 참조 조건으로 어떤 값이 제공 됐는지 보여 준다. 

## rows

옵티마이저가 대상 테이블에 얼마나 많은 레코드가 포함되어있는지 예측한 값

## Extra

### DIstinct

```sql
EXPLAIN
SELECT DISTINCT d.dept_no
FROM departments d, dept_emp de WHERE de.dept_no = d.dept_no;
```

![Untitled](summary%2074ff2959a0a342ee8b4d4afd854cffee/Untitled%206.png)

조인하지 않아도 되는 항목은 모두 무시하고 꼭 필요한 것만 조인했으며 dept_emp 테이블에서는 꼭 필요한 레코드만 읽음.

### Using filesort

ORDER BY 를 처리할 때 인덱스를 사용하지 못하는 경우 조회된 레코드를 정렬용 메모리 버퍼에 복사하여 퀵소트 알고리즘을 수행할 때 나오는 코멘트. 

### Using index

데이터 파일 을 전혀 읽지 않고 인덱스만 읽어서 쿼리를 모두 처리할 수 있을 때 

### Using index for group-by

GROUP BY 처리에 인덱스를 이용하면 레코드의 정렬이 필요하지 않고 인덱스의 필요한 부분만 읽으면 된다. 

루스 인덱스 스캔을 사용한다.  

- WHERE 절이 존재하지 않거나 WHERE 절에서 사용할 수 있는 인덱스가 존재하여야 한다.

### Using where

```sql
SELECT * FROM employees WHERE department = 'IT' AND salary > 50000;
```

다음 쿼리에서 salary > 50000 은 작업 범위 조건으로 스토리지 엔진에서, department = ‘IT’는 체크 조건으로 MySQL 엔진에서 처리하는데 where 조건을 사용하여 절의 조건을 평가하고 해당 조건에 맞는 행들만 필터링 했다는 것을 뜻함

⇒ EXPLAIN EXTENDED 의 Filtered 칼럼을 통해 인덱스 수립 계획을 잡아야 함 (where 절을 통해 몇개가 남았는지 알 수 있음)

## DISTINCT 처리

특정 칼럼의 유니크한 값만을 조회하려면 SELECT 쿼리에 DISTINCT를 사용한다. 

집합 함수가 사용되는 경우와 사용되지 않는 경우에 따라 미치는 범위가 달라진다. 

### SELECT DISTINCT

단순히 SELECT 되는 레코드 중에서 유니크한 레코드만 가져오고자 하면 SELECT DISTINCT 형태의 쿼리 문장을 사용한다. 

- 정렬을 보장하지 않는다. (GROUP BY 와 다른점)
- 인덱스 이용 - 정렬 필요치 않음
- DISTINCT는 SELECT하는 레코드를 유니크하게 SELECT한느 것이지 칼럼을 유니크하게 조회하는 것이 아니다.
- SELECT 절에 사용된 DISTINCT 키워드는 조회되는 모든 칼럼에 영향을 미친다.

### 집합 함수와 함께 사용된 DISTINCT

집합 함수가 없는 SELECT 쿼리에서 DISTINCT는 조회하는 모든 칼럼의 조합이 유니크한 것만 가져온다.

→ 집합함수 내에서 사용되면 그 집합 함수의 인자로 전달된 칼럼 값이 유니크한 것들을 가져온다. 

- 인덱싱된 칼럼에 대해 DISTINCT 처리를 수행할 때는 인덱스를 풀 스캔하거나 레인지 스캔하면서 임시 테이블 없이 최적화된 처리를 수행할 수 있다.

## 임시 테이블

MySQL 엔진이 스토리지 엔진으로부터 받아온 레코드를 정렬하거나 그루핑할 때 내부적인 테이블을 사용한다. 

### 임시 테이블이 필요한 쿼리

- ORDER BY 와 GROUP BY 에 명시된 칼럼이 다른 쿼리 → 유니크
- ORDER BY 나 GROUP BY 에 명시된 칼럼이 조인의 순서상 첫 번째 테이블이 아닌 쿼리 →유니크
- DISTINCT 와 ORDER BY 가 동시에 존재하는 경우 또는 DISTINCT가 인덱스로 처리되지 못하는 쿼리 → 유니크
- UNION이나 UNION DISTINCT 가 사용된 쿼리 → 유니크
- UNION ALL이 사용된 쿼리 → 유니크 인덱스 X
- 쿼리의 실행 계획에서 select_type이 DERIVED인 쿼리 → 유니크 인덱스 X

→ Extra 칼럼에 “Using temporary” 라는 키워드가 있으면 사용중

유니크 인덱스가 존재하는 것이 더 느리다.

- 대용량 칼럼이 있거나 메모리의 저장공간을 넘어서면 임시테이블이 디스크에 생성된다.

---

## 조인의 종류

### JOIN(INNER JOIN)

일반적으로 조인이라 함은 INNER JOIN을 지칭한다. 

조인은 두개의 반복 루프로 두 개의 테이블을 조건에 맞게 연결해주는 작업이다. 

- 중첩된 반복 루프에서 최종적으로 선택될 레코드가 안쪽 반복 루프에 의해 결정되는 경우를 INNER JOIN 이라고 한다. → 조건을 만족하는 레코드만 조인의 결과로 가져온다.

### OUTER JOIN

조건을 만족하지 않는 레코드도 NULL로 처리하여 가져온다. 

- INNER 테이블이 조인의 결과에 영향을 미치지 않고, OUTER 테이블의 내용에 따라 조인의 결과가 결정된다.

- LEFT OUTER JOIN : 조인의 결과를 결정하는 아우터 테이블이 왼쪽
- RIGHT OUTER JOIN : 조인의 결과를 결정하는 아우터 테이블이 오른쪽

 OUTER JOIN에서 레코드가 없을 수도 있는 쪽의 테이블에 대한 조건은 반드시 LEFT JOIN ON절에 명시한다. 

### NATURAL JOIN

서로 이름이 같은 칼럼을 모두 조인 조건으로 사용. 

### Single-sweep multi join

반복 루프를 돌면서 레코드 단위로 모든 조인 대상 테이블을 차례대로 읽는 방식

### 조인 버퍼를 이용한 조인

드리븐 테이블의 풀 테이블 스캔이나 인덱스 풀 스캔을 피할 수 없다면  옵티마이저는 드라이빙 테이블에서 읽은 레코드를 메모리에 캐시한 후 드리븐 테이블과 이 메모리 캐시를 조인한다. 이 때 사용되는 메모리 캐시를 조인 버퍼라 한다 

- 조인 버퍼가 사용되는 조인에서는 결과의 정렬 순서가 흐트러질 수 있다.

### INNER JOIN 과 OUTER JOIN의 선택

INNER JOIN과 OUTER JOIN은 성능을 고려해서 선택할 것이 아니라 업무 요건에 따라 선택해야한다. 

# 7장

## 리터럴 표기법

### 문자열

SQL 표준에서 문자열은 항상 ‘를 사용해서 표현한다. 그렇지만 쌍따옴표를 사용할 수 있다. 

- 문자열 값에 홑따옴표가 포함돼 있을 때 홑따옴표를 두 번 연속해서 입력하면 된다.
- SQL에서 사용되는 식별자가 키워드와 충돌할 때 쌍따옴표나 대괄호로 감싸서 충돌을 피한다.

## 숫자

- 따옴표 없이 입력
- 문자열 형태로 따옴표를 사용하더라도 비교 대상이 숫자 값 이거나 숫자 타입의 칼럼이면 숫자값으로 자동 타입변환한다.

### 날짜

- MYSQL 에서는 정해진 형태의 날짜 포맷으로 표기하면 MySQL 서버가 자동으로 DATE나 DATETIME 값으로 변환한다.

### 불리언

- BOOL 이나 BOOLEAN 이라는 타입이 있지만 이것은 TINYINT 타입에 대한 동의어이다.
- true / false 를 1과 0으로 나타낸다. → 모든 숫자가 아닌 0과 1로만 나타낼 수 있으므로 enum으로 지정하는 것이 좋다

## MySQL 연산자

### 동등 비교

<=> 연산자는 NULL값 비교까지한다. 

### 부정 비교

같지 않다 비교를 위한 연산자는 <>를 많이 사용한다. != 도 사용 가능하다. 

부정 연산자는 !을 사용한다. 

### AND, OR

AND - &&, OR - || 

오라클에서 || 가 표현식의 결합 연산자가 아니라 문자열을 결합하는 연산자로 사용한다. 

### REGEXP 연산자

RLIKE는 정규 표현식을 비교하는 연산자이다. 

REGEXP 연산자를 문자열 칼럼 비교에 사용할 때 REGEXP 조건의 비교는 인덱스 레인지 스캔을 사용할 수 없다. 

### LIKE 연산자

REGEXP 연산자와 다르게 인덱스를 이용해 처리할 수 있다. 

비교 대상의 처음부터 끝까지 일치하는 경우에만 TRUE를 반환한다. 

- 와일드카드 문자인 %, _ 가 검색어의 뒤쪽에 있다면 인덱스 레인지 스캔으로 사용할 수 있지만 와일드카드가 검색어의 앞쪽에 있다면 인덱스 레인지 스캔을 사용할 수 없다.

### BETWEEN 연산자

- IN과의 차이점 : IN은 동등비교, BETWEEN 은 크다와 작다 비교 → BETWEEN이 선형으로 인덱스를 검색해야 하는 것과는 달리 IN은 동등 비교를 여러 번 수행하는 것과 같은 효과가 있기 때문에 테이블의 인덱스를 최적으로 사용할 수 있다.

### IN 연산자

IN은 여러 개의 값에 대해 동등 비교 연산을 수행하는 연산자다. 

- IN 연산자의 입력이 상수가 아니라 서브 쿼리인 경우에는 상당히 느려질 수 있다.
- NULL 값을 검색할 수는 없다.
- NOT IN의 실행 계획은 부정형 비교라 인덱스를 이용해 처리 범위를 줄이는 조건으로 사용할 수 없어 인덱스 풀 스캔으로 표시된다.

### 현재 시각 조회 (NOW, SYSDATE)

두 함수 모드 현재의 시간을 반환하는 함수로서 같은 기능을 수행한다. 

- NOW 함수는 같은 값을 반환하지만 SYSDATE() 함수는 호출 시점에 따라 값이 달라진다.

### 날짜와 시간의 포맷(DATE_FORMAT, STR_TO_DATE)

### 날짜와 시간의 연산 (DATE ADD, DATE_SUB)

특정 날짜에서 년도나 월일 또는 시간등을 더하거나 뺄 때 사용

### 타임 스탬프 연산 (UNIX_TIMESTAMP, FROM_UNIXTIME)

### 문자열 처리 (RPAD, LPAD / RTRIM, LTRIM, TRIM)

- RPAD() 와 LPAD() 함수는 문자열의 좌측 또는 우측에 문자를 덧붙여서 지정된 길이의 문자열로 만드는 함수다.
- LTRIM() 함수와 RTRIM() 함수는 문자열의 우측 또는 좌측에 연속된 공백문자를 제거하는 함수다.

### 문자열 접합 CONCAT

여러 개의 문자열을 연결해서 하나의 문자열로 반환하는 함수로 인자의 개수는 제한이 없다. 

### GROUP BY 문자열 접합(GROUP_CONCAT)

COUNT()나 MAX(), MIN(), AVG() 등과 같은 그룹함수 중 하나다. 

- GROUP_CONCAT() 함수는 지정한 칼럼의 값들을 연결하기 위해 제한적인 메모리 버퍼 공간을 사용한다.

### 값의 비교와 대체 (CASE WHEN .. THEN .. END)

### 타입의 변환(CAST, CONVERT)

### 처리 대기 (SLEEP)

- SQL 의 개발이나 디버깅 용도로 잠깐 대기하거나 일부러 쿼리의 실행을 유지할 때 유용한 함수

### VALUES

해당 칼럼에 INSERT 하려고 했던 값을 참조하는 것이 가능하다. 

### COUNT()

COUNT함수는 칼럼이나 표현식을 인자로 받는다. 

- where 조건이 없으면 세지 않고 바로 값을 반환한다.
- where 조건이 있으면 세어보고 값을 반환해 속도가 느리다.
- 칼럼명이나 표현식이 인자로 사용되면 그 칼럼이나 표현식의 결과가 NULL이 아닌 레코드 건수만 반환된다.

## 인덱스

### 인덱스를 사용하기 위한 기본 규칙

- WHERE 절이나 ORDER BY 또는 GROUP BY가 인덱스를 사용하려면 기본적으로 인덱스된 칼럼의 값 자체를 변환하지 않고 그래도 사용한다는 조건을 만족해야 한다.
- 칼럼값 여러 개를 곱하거나 더해서 비교해야 하는 복잡한 연산이 필요할 떄는 미리 계산된 값을 저장할 별도의 칼럼을 추가하고 그 칼럼에 인덱스를 생성해야한다.
- WHERE 절에 사용되는 비교 조건에서 연산자 양쪽의 두 비교 대상 값은 데이터 타입이 일치해야 한다. → 일치 하지 않는다면 ref, range 대신에 풀 인덱스 스캔을 하는 index가 된다.

## ORDER BY 절의 인덱스 사용

ORDER BY 는 GROUP BY 와 비슷하지만 다른 점이 하나 더 있는데 정렬되는 각 칼럼의 오름차순 및 내림차순 옵션이 인덱스와 같거나 또는 정반대의 경우에만 사용할 수 있다. 

### WHERE 조건과 ORDER BY 절의 인덱스 사용

- WHERE 절과 ORDER BY 절이 동시에 같은 인덱스를 이용 : 빠른 성능
- WHERE 절만 인덱스를 이용 : WHERE 절의 조건에 일치하는 레코드의 건수가 많지 않을 때
- ORDER BY 절만 인덱스를 이용 : 아주 많은 레코드를 조회해서 정렬 할 때

### GROUP BY 절과 ORDER BY 절의 인덱스 사용

- GROUP BY 절에 명시된 칼럼과 ORDER BY 에 명시된 칼럼이 순서와 내용이 모두 같아야 한다.  → 둘 다 인덱스 사용 가능해야함

## JOIN

- 조인 작업에서 드라이빙 테이블을 읽을 때는 인덱스 탐색 작업을 단 한번만 수행하고 이후부터는 스캔만 수행한다.

- 두 칼럼 모두 인덱스가 있는 경우 : 어느 테이블을 드라이빙으로 선택하든 인덱스를 이용해 드리븐 테이블의 검색 작업을 빠르게 수행할 수 있다.
- 한쪽에만 인덱스가 있는 경우 : 항상 인덱스가 없는 쪽을 드라이빙 테이블로 선택하고 인덱스가 있는 쪽을 드라이븐 테이블로 선택한다.
- 두 칼럼 모두 인덱스가 없는 경우 : 레코드 건수가 적은 테이블을 드리븐 테이블로 선택하는 것이 효율적이다.

### JOIN 칼럼의 데이터 타입

조인 칼럼 간의 비교에서 각 칼럼의 데이터 타입이 일치하여야한다. 

### OUTER JOIN의 주의사항

OUTER JOIN 에서 OUTER로 조인되는 테이블의 칼럼에 대한 조건은 모두 ON절에 명시해야 한다.  그렇지 않으면 옵티마이저가 INNER JOIN과 같은 방법으로 처리한다. 

### OUTER JOIN 을 이용한 ANTI JOIN

두 개의 테이블에서 한쪽 테이블에는 있지만 다른 한쪽 테이블에는 없는 레코드를 검색할 때 ANTI JOIN을 이용한다. 

- 레코드가 적다면 NOT IN 이나 NOT EXIST를 이용하면 되지만 많다면 ANTI JOIN을 이용하는 것이 좋다.
- NOT IN → ANTI JOIN : WHERE 절의 조건에 NOT NULL인 칼럼을 선택해야 한다.

### FULL OUTER JOIN

양쪽 테이블의 모든 레코드를 조회할 수 있다. 

UNION을 사용하여 구현가능하다. 

```sql
SELECT e.yearmonth, e.event_name, n.news_title FROM tab_event e
LEFT JOIN tab_news n ON n.yearmonth = e.yearmonth
UNION
SELECT n.yearmonth, e.event_name, n.news_title FROM tab_news n
LEFT JOIN tab_event e ON e.yearmonth = n.yearmonth;
```

### JOIN과 FOREIGN KEY

FOREIGN KEY는 조인과 아무런 관련이 없다. 

### 지연된 조인(DELAYED JOIN)

조인의 결과를 GROUP BY 나 ORDER BY 하면 더 많은 레코드를 처리 해야한다. 

```sql
SELECT e.*
FROM salaries s, employees e
WHERE e.emp_no = s.emp_no
	AND s.emp_no BETWEEN 1001 AND 13000
GROUP BY s.emp_no
ORDER BY SUM(s.salary) DESC
LIMIT 10;
```

- employees 테이블을 드라이빙 테이블로 선택해서 조건을 만족하는 레코드를 얻는다
- salaries 테이블을 조인
- 조인의 결과 레코드를 임시 테이블에 저장하고 GROUP BY 처리를 통해 레코드를 줄였다.
- ORDER BY 로 10건만 순서대로 가져온다

```sql
SELECT e.*
FROM 
(
SELECT s.emp_no
FROM salaries s
WHERE s.emp_no BETWEEN 1001 AND 13000
GROUP BY s.emp_no
ORDER BY SUM(s.salary) DESC
LIMIT 10
) x,
employees e
WHERE e.emp_no = x.emp_no;
```

임시 테이블을 한 번 더 사용하기 때문에 느리다고 생각 할 수 있지만 저장할 레코드가 10건밖에 되지 않으므로 메모리를 빠르게 처리한다. 

조인 횟수가 적다.

- 지연된 조인은 조인의 개수를 줄이는 것 뿐만 아니라 GROUP BY 나 ORDER BY 처리가 필요한 레코드의 전체 크기를 줄이는 역할도 한다.

## 서브쿼리

쿼리를 작성 할 때 서브 쿼리를 사용하면 단위 처리별로 쿼리를 독립시킬 수 있다. 조인처럼 여러 테이블을 섞어두는 형태가 아니라서 뭐리의 가독성도 높아지며 복잡한 쿼리도 손쉽게 작성할 수 있다. 

### 서브 쿼리의 제약사항

- 대부분의 쿼리 문장에서 사용할 수 있지만 LIMIT 절과 LOAD DATA INFILE 파일명에는 사용 불가
- IN 연산자와 함께 사용될 때는 비효율적
- ORDER BY 와 LIMIT를 동시에 사용 불가
- FROM 절에 사용하는 서브 쿼리는 상관 서브 쿼리 형태로 사용 불가.
- 서브 쿼리를 이용해 하나의 테이블에 대해 읽고 쓰기를 동시에 할 수 없다.  → 임시 테이블을 만들어서 처리 가능

### SELECT 절에 사용된 서브쿼리

일반적으로 칼럼과 레코드가 하나인 결과를 반환해야한다. 

- 조인으로 처리하는 것이 효율적이다.

### WHERE 절에 단순 비교를 위해 사용된 서브 쿼리

독립 서브 쿼리일 때 서브 쿼리를 먼저 실행한 후 상수로 변환하고 그 조건을 범위 제한 조건으로 사용한다. 

### WHERE 절에 IN과 함께 사용된 서브 쿼리

IN 안에 있는 독립된 서브쿼리가 EXIST로 변환되어 상관 서브쿼리로 사용 되기 때문에 외부 쿼리 type 값이 ALL로 된다. 

- 바깥쪽 테이블과 서브 쿼리 테이블 관계가 1 : 1이거나 M : 1인 경우
    - 바깥 쪽 쿼리와 서브 쿼리를 조인으로 풀어서 작성해도 같은 결과가 보장되기 때문에 다음과 같이 조인으로 풀어서 작성하면 쉽게 성능을 개선할 수 있다.

```sql
SELECT de.*
FROM dept_emp de INNER JOIN departments d
	ON d.dept_name = 'Finance' AND d.dept_no = de.dept_no;
```

- 바깥 쪽 테이블 (dept_emp) 과 서브 쿼리 테이블 (departments)의 관계가 1 : M 인 경우
    - 바깥쪽 쿼리와 서브 쿼리를 조인으로 풀어서 작성하면 최종 결과의 건수가 달라질 수 있기 때문에 단순히 서브 쿼리를 조인으로 변경할 수 없다.
    1. 조인 후 조인 칼럼으로 그룹핑해서 결과를 가져오는 방법
    
    ```sql
    SELECT de.*
    FROM dept_emp de INNER JOIN departments d
    	ON d.dept_name = 'Finance' AND d.dept_no = de.dept_no
    GROUP BY de.dept_no;
    ```
    
    이와 같이 GROUP BY 를 추가해서 조인 때문에 발생한 중복 레코드를 강제로 제거한다. 
    
    ![Untitled](summary%2074ff2959a0a342ee8b4d4afd854cffee/Untitled%207.png)
    
    ![Untitled](summary%2074ff2959a0a342ee8b4d4afd854cffee/Untitled%208.png)
    
    # 쿼리캐시
    
    ```sql
    SELECT u.*
    FROM `user` u
    JOIN `friend` f ON u.`id` = f.`user_sendid`
    WHERE f.`user_receiveid` = #{id}
    AND f.`confirm` = '승인중';
    GROUP BY f.`user_sendid`
    ```
    
    1. 원본 쿼리에서 서브 쿼리를 분리시켜서 2개의 쿼리를 실행

```sql
ResultSet rs = statement.executeQuery("SELECT d.dept_no FROM departments d " 
	+ "WHERE d.dept_name = 'Finance'");

ResultSet rs1 = null;
StringBuffer inEnumBuffer = new StringBuffer();
while(rs.next()){
	inEnumBuffer.append(",':").append(rs.getString("dept_no")).append("'");
}
rs1 = statement.executeQuery("SELECT * FROM dept_emp " + 
	"WHERE dept_no IN ('" + inEnumBuffer.toString() + "')");
```

## WHERE 절에 NOT IN과 함께 사용된 서브 쿼리

만약 NOT IN 조건에서 칼럼이 NULL이 될 수 있으면 NOT EXISTS 형태로 변환할 수 없게된다. 

- 서브 쿼리가 결과가 레코드를 한 건이라도 가진다면 : NULL IN (레코드를 가지는 결과) ⇒ NULL
- 서브 쿼리가 결과 레코드를 한 건도 가지지 않는다면 : NULL IN ( 빈 결과) ⇒ FALSE
- 왼쪽의 값이 NULL인지 여부에 따라 NOT EXISTS로 최적화를 적용할지 여부가 결정

![Untitled](summary%2074ff2959a0a342ee8b4d4afd854cffee/Untitled%209.png)

```sql
SELECT u.*
FROM `user` u
LEFT JOIN (
    SELECT `user_receiveid`
    FROM `friend`
    WHERE `user_sendid` = {id} OR `user_receiveid` = {id}
) AS f ON u.`id` = f.`user_receiveid`
LEFT JOIN (
    SELECT `user_receiveid`
    FROM `block`
    WHERE `user_sendid` = #{id}
) AS b1 ON u.`id` = b1.`user_receiveid`
LEFT JOIN (
    SELECT `user_sendid`
    FROM `block`
    WHERE `user_receiveid` = #{id}
) AS b2 ON u.`id` = b2.`user_sendid`
WHERE u.`id` <> {id} AND f.`user_receiveid` IS NULL AND b1.`user_receiveid` IS NULL AND b2.`user_sendid` IS NULL
ORDER BY RAND()
LIMIT 5;
```

## FROM 절에 사용된 서브 쿼리

FROM 절에 사용된 서브 쿼리는 항상 임시 테이블을 사용하므로 제대로 최적화되지 못하고 비효율적이다. 

# 10장

# 파티션

파티션이란 MySQL 서버의 입장에서 데이터를 별도의 테이블로 분리해서 저장하지만 사용자 입장에서는 여전히 하나의 테이블로 분리해서 읽기와 쓰기를 할 수 있게 해주는 솔루션이다. 

### 파티션을 사용하는 경우

1. 단일 INSERT와 단일 또는 범위 SELECT의 빠른 처리
- 데이터와 인덱스를 조각화해서 물리적 메모리를 효율적으로 사용할 수 있게한다.
1. 데이터의 물리적인 저장소를 분리
2. 로그 데이터의 효율적인 관리
- 로그 데이터를 파티션 테이블로 관리한다면 불필요한 데이터 삭제 작업을 파티션을 추가하거나 삭제하는 방식으로 처리할 수 있다.

### 파티션 테이블의 검색

파티션 테이블에서 검색할 때 영향을 미치는 요소

- WHERE 절의 조건으로 검색해야 할 파티션을 선택할 수 있는가
- WHERE 절의 조건이 인덱스를 효율적으로 사용할 수 있는가

1. 파티션 선택 가능 + 인덱스 효율적 사용 가능

쿼리가 가장 효율적으로 처리될 수 있다. 파티션의 개수에 관계없이 검색을 위해 꼭 필요한 파티션의 인덱스만 레인지 스캔한다. 

1. 파티션 선택 불가 + 인덱스 효율적 사용 가능

WHERE 조건에 일치하는 레코드가 저장된 파티션을 걸러낼 수 없기 때문에 우선 테이블의 모든 파티션을 대상으로 검색해야 한다. 하지만 각 파티션에 대해서는 인덱스 레인지 스캔을 사용할 수 있기 때문에 최종적으로 테이블에 존재 하는 모든 파티션의 개수만큼 인덱스 레인지 스캔을 수행해서 검색하게 된다. 

1. 파티션 선택 가능 + 인덱스 효율적 사용 불가

검색하려는 레코드가 저장된 파티션을 선별할 수 있기 때문에 파티션 개수에 관계없이 검색을 위해 필요한 파티션만 읽으면 된다. 

1. 모두 사용 불가 

풀 테이블 스캔 수행

### 파티션에 대한 결론

어떤 파티션 종류를 사용하든 모든 파티션을 골고루 읽고 써야 하는 테이블이라면 파티션을 사용해 SELECT 기능을 향상시키기 어렵다. 그렇지만 레코드 건수가 느려져 작업이 느려지면 파티션을 고려해보자. 

- 날짜 칼럼을 이용해 레인지 파티션을 사용할 수 있고, 읽기나 쓰기 작업을 일부 파티션으로 모을 수 있다면 테이블의 크기에 상관없이 항상 파티션을 적용하는 것이 도움될 것이다.

```sql
CREATE TABLE orders (
    id INT AUTO_INCREMENT PRIMARY KEY,
    order_date DATE,
    order_amount DECIMAL(10, 2),
    INDEX idx_order_date (order_date)
)
PARTITION BY RANGE (TO_DAYS(order_date)) (
    PARTITION p0 VALUES LESS THAN (TO_DAYS('2023-01-01')),
    PARTITION p1 VALUES LESS THAN (TO_DAYS('2023-02-01')),
    PARTITION p2 VALUES LESS THAN (TO_DAYS('2023-03-01')),
    PARTITION p3 VALUES LESS THAN (TO_DAYS('2023-04-01')),
    PARTITION p4 VALUES LESS THAN (MAXVALUE)
);
```