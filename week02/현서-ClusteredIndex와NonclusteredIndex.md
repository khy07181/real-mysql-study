# [MySQL] 테이블은 엑셀이 아니다: Clustered Index와 Secondary Index의 정체

우리는 흔히 데이터베이스의 테이블을 엑셀과 같은 '이쁜 표(Table)' 형태라고 생각한다. 데이터를 입력하면 차곡차곡 순서대로, 눈에 보이는 그대로 저장된다고 믿는 것이다.
<img width="484" height="335" alt="image" src="https://github.com/user-attachments/assets/025016cf-f878-404e-97e4-68f549789653" />


하지만 **실제로 MySQL(InnoDB)의 모든 데이터는 `B-Tree` 구조의 `Clustered Index` 형태로 저장된다.** 즉, 테이블 그 자체가 하나의 거대한 인덱스 덩어리다.

<img width="834" height="457" alt="image" src="https://github.com/user-attachments/assets/2d789d52-7384-471b-8ba0-d83a9871a917" />


이 구조를 이해해야 왜 PK 설정이 중요한지, 인덱스가 어떻게 동작하는지 제대로 알 수 있다.

<img width="895" height="220" alt="image" src="https://github.com/user-attachments/assets/6a8ff14c-b701-46a7-9f6d-9e1081ebafdf" />


---

## 1. Clustered Index: 테이블 그 자체

InnoDB 스토리지 엔진에서 테이블은 Primary Key(PK)를 기준으로 정렬된 `Clustered Index`로 구성된다.

- **구조:** PK값이 B-Tree의 키(Key)가 되고, 리프 노드(Leaf Node)에는 **실제 레코드의 모든 데이터**가 저장된다.
- **특징:**
    - 데이터 자체가 PK 순서대로 물리적으로 정렬되어 저장된다.
    - 따라서 테이블당 단 하나만 존재할 수 있다.
    - PK를 이용한 범위 검색(Range Scan)에서 압도적인 성능을 보인다.
    - 리프 노드(데이터 페이지)끼리는 Linked List(Double Linked List) 형태로 연결되어 있어 순차 스캔에 유리하다.

### 왜 PK를 `Auto Increment` & `Int(BigInt)`로 설정할까?

테이블을 설계할 때 관습적으로 PK를 숫자형(Int/BigInt)이자 자동 증가(Auto Increment) 값으로 설정한다. 그 이유는 Clustered Index의 특성 때문이다.

1. **쓰기 성능:** 데이터는 PK 순서대로 저장되어야 한다. 만약 랜덤한 문자열(UUID 등)을 PK로 쓰면, 데이터를 넣을 때마다 인덱스 페이지를 중간에 끼워 넣거나 분할(Page Split)하는 비용이 발생한다. 순차적으로 증가하는 숫자는 그냥 뒤에 붙이기만 하면 되므로 쓰기 성능이 좋다.
2. **공간 효율:** PK는 모든 보조 인덱스(Secondary Index)에도 포함된다. PK의 크기가 커지면 보조 인덱스의 크기도 덩달아 커져서 비효율적이다.

---

## 2. Non-Clustered Index (Secondary Index): 데이터로 가는 이정표

그렇다면 우리가 `CREATE INDEX` 문으로 직접 생성하는 인덱스는 무엇일까? 이를 **Non-Clustered Index** 혹은 Secondary Index(보조 인덱스)라고 부른다.

- **구조:** 별도의 B-Tree 구조를 가진다.
- **리프 노드의 정체:** Clustered Index와 달리, 리프 노드에 실제 데이터가 들어있지 않다. 대신 **해당 데이터의 PK 값**을 가지고 있다. (MyISAM 등 다른 엔진은 물리적 주소를 갖기도 하지만, InnoDB는 PK를 갖는다.)

### 검색 과정 (인덱스를 탈 때)

우리가 생성한 보조 인덱스를 통해 데이터를 조회하면 다음과 같은 '2단계' 과정을 거친다.

1. **보조 인덱스 탐색:** 검색 조건에 맞는 값을 보조 인덱스에서 찾는다. -> 여기서 **PK 값**을 얻는다.
2. **Clustered Index 탐색:** 얻어낸 PK 값을 가지고 다시 Clustered Index(테이블 본체)를 뒤져서 **최종 데이터**를 가져온다.

---
?근데 굳이 있어야 해?
실습을 통해서 인덱스가 있어야만 하는 이유를 보겠다

## 3. 실습환경

- **Environment:** Docker
- **Image:** `mysql:latest`

### 기본 세팅

```python
# MySQL 컨테이너 실행 (이미지가 없으면 Pull 받음)
docker run --name mysql-index-test -e MYSQL_ROOT_PASSWORD=1234 -d -p 3306:3306 mysql:latest

# 컨테이너 접속
docker exec -it mysql-index-test mysql -u root -p
# (패스워드 1234 입력)
```

### 데이터 생성 (10만 건)

```sql
-- 데이터베이스 및 테이블 생성
CREATE DATABASE my_test;
USE my_test;

CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(50),
    age INT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP 
);

-- 대량 데이터 삽입을 위한 프로시저 생성
CREATE PROCEDURE generate_data()
BEGIN
    DECLARE i INT DEFAULT 0;
    WHILE i < 100000 DO
        INSERT INTO users (username, age) 
        VALUES (CONCAT('user_', i), FLOOR(RAND() * 50) + 20);
        SET i = i + 1;
    END WHILE;
END$$

-- 프로시저 실행 (10만 건 삽입)
CALL generate_data();

-- 확인
SELECT count(*) FROM users; 
-- 결과: 100000
```

## 4. 실습 : 인덱스가 없을 때(Full Table Scan)

현재 `users` 테이블에는 PK(id)에만 인덱스가 걸려 있다. 만약 `username`으로 검색하면 어떻게 될까?

```sql
EXPLAIN SELECT * FROM users WHERE username = 'user_75000';
```

결과:

```sql
mysql> EXPLAIN SELECT * FROM users WHERE username = 'user_75000';
+----+-------------+-------+------------+------+---------------+------+---------+------+--------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows   | filtered | Extra       |
+----+-------------+-------+------------+------+---------------+------+---------+------+--------+----------+-------------+
|  1 | SIMPLE      | users | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 100184 |    10.00 | Using where |
+----+-------------+-------+------------+------+---------------+------+---------+------+--------+----------+-------------+
1 row in set, 1 warning (0.00 sec)

```

중요하지 않은 메타 데이터 격은 comment로 간단한 소개를 추가했다.

쿼리 튜닝 시 가장 눈여겨봐야 할 4가지 컬럼이다.

1. type(접근 방식)
    - 테이블에서 데이터를 어떻게 찾았는지 나타낸다.
    - 좋은 순서가 있다 : const > eq_ref > ref > range > index > ALL
        - **`const`**: PK나 Unique Key로 1건만 조회 (최고).
        - **`ref`**: Non-Unique 인덱스로 조회.
        - **`range`**: 인덱스 범위 검색 (Index Range Scan).
        - **`index`**: 인덱스 풀 스캔 (Index Full Scan - 데이터 파일 안 읽고 인덱스만 훑음).
        - **`ALL`**: Table Full Scan(최악). 인덱스를 전혀 타지 않고 데이터 전체를 읽었다.
2. key
    - optimizer가 실제로 선택한 인덱스
        - `NULL`: 인덱스를 타지 않음.
        - `PRIMARY`: Clustered Index 사용.
3. rows
    - 쿼리를 처리하기 위해 읽어야 할(접근해야 할) 데이터의 예상 개수. 작을수록 좋다.
4. Extra 
    - `Using index`: **아주 좋다.** 데이터 파일(Clustered Index)까지 안 가고, 인덱스만 읽어서 끝냈다(커버링 인덱스).
    - `Using filesort`: **나쁘다.** 인덱스로 정렬이 안 돼서 별도로 메모리에서 정렬 작업을 수행했다.
    - `Using where`: 스토리지 엔진에서 데이터를 가져온 뒤, MySQL 엔진에서 한 번 더 걸러냈다는 뜻이다. (일반적임)

### EXPLAIN 해석 ( index 생성 전)

위 결과에서 `type: ALL`, `rows: 100184`가 나왔다. 즉, user_75000이라는 사람 한 명을 찾기 위해 10만 명의 데이터를 처음부터 끝까지 다 뒤졌다(Full Scan)는 뜻이다. 데이터가 100억 개라면 100억 개를 다 뒤져야 한다. 시스템 마비의 주범이다…! 👿

## 5. 실습2 : 인덱스 추가 후(Index Range Scan)

`username` 컬럼에 보조 인덱스를 추가해 보자.

```sql
CREATE INDEX idx_username ON users(username);
```

인덱스를 생성하면 `username`을 기준으로 정렬된 B-Tree가 만들어지고, 리프 노드에는 `id`(PK)가 저장된다. 다시 똑같은 쿼리의 실행 계획을 확인해 보자.

```sql
explain select * from users where username = 'user_75000';
```

결과:

```sql
+----+-------------+-------+------------+------+---------------+--------------+---------+-------+------+----------+-------+
| id | select_type | table | partitions | type | possible_keys | key          | key_len | ref   | rows | filtered | Extra |
+----+-------------+-------+------------+------+---------------+--------------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | users | NULL       | ref  | idx_username  | idx_username | 203     | const |    1 |   100.00 | NULL  |
+----+-------------+-------+------------+------+---------------+--------------+---------+-------+------+----------+-------+
```

### 극적인 변화

1. **`type: ref`**: 더 이상 테이블 전체를 뒤지지 않는다. 인덱스를 통해 특정 값(ref)을 참조하여 데이터를 찾았다.
2. **`key: idx_username`**: 우리가 방금 만든 인덱스를 정상적으로 사용했다.
3. **`rows: 1`**: 10만 개를 뒤지던 것이 **1개**만 읽는 것으로 바뀌었다.

### 내부 동작 과정

이 쿼리는 내부적으로 다음과 같이 동작하여 `rows: 1`을 달성했다.

1. **Secondary Index 탐색:** `idx_username` B-Tree 루트에서 시작하여 `user_75000`을 찾는다. (매우 빠름, O(logN))
2. **PK 획득:** 리프 노드에서 `user_75000`에 매핑된 `id = 75001`을 얻는다.
3. **Clustered Index 탐색:** `id = 75001`을 가지고 테이블 본체(Clustered Index)로 가서 실제 데이터를 가져온다.

## 6. 그럼 모든 키에 대해서 인덱스를 생성하면 되겠다!

**그러면 안된다.**

### 1. 쓰기 성능 저하 (Insert/Update/Delete)

인덱스는 정렬된 상태를 유지해야 한다.

- 새로운 데이터가 추가되면, 테이블뿐만 아니라 **모든 인덱스에도 데이터를 집어넣고 다시 정렬**해야 한다.
- 인덱스가 많을수록 데이터 변경 시 작업량이 배로 늘어난다.
    - 인덱스가 100개의 컬럼에 대해서 다 있는 경우, 하나의 레코드를 추가하면, 100개의 인덱스에서도 재정렬이 이뤄진다.

### 2. 저장 공간 낭비

인덱스도 결국 데이터다. 디스크 공간을 차지하며, 불필요한 인덱스는 DB 용량을 낭비한다.

### 3. 옵티마이저의 혼란

인덱스가 너무 많으면 MySQL 옵티마이저가 어떤 인덱스를 탈지 계산하는 데 시간이 걸리거나, 잘못된 인덱스를 선택할 확률이 높아진다.

## **중요! 인덱스 힌트**

만약 옵티마이저가 멍청하게 이상한 인덱스를 탄다면, 개발자가 강제로 지정할 수 있다.

```sql
SELECT * FROM users USE INDEX (idx_username) WHERE username = 'user_75000';
```
