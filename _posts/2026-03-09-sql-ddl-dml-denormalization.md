---
title: "SQL의 DDL과 DML 차이, 그리고 데이터베이스 역정규화 이해하기"
categories:
  - Backend
  - Database
tags:
  - SQL
  - Database
  - DDL
  - DML
  - Normalization
  - Denormalization
---

데이터베이스를 설계하고 운영하기 위해선 데이터 구조와 조작 방식을 이해해야 한다.
`SQL`에서는 데이터베이스 구조를 정의하는 명령어와 실제 데이터를 다루는 명령어가 구분되어 있으며,
이를 각각 `DDL(Data Definition Language)`과 `DML(Data Manipulation Language)`라고 한다.

데이터베이스 설계 과정에서는 데이터의 중복을 줄이기 위해 `정규화(Normalization)`를 수행하고,
또는 성능 최적화를 위해 구조를 단순화하는 `역정규화(Denormalization)`를 사용하기도 한다.

이 글에서는 `SQL`의 `DDL`, `DML`의 차이를 정리해보고,
DB 설계에서 사용되는 `역정규화`의 개념과 적용 시의 고려사항도 함께 살펴본다.

---

## 1. SQL에서의 DDL vs DML

`SQL` 명령어는 크게 **DB 구조를 정의하는 명령어**와 **데이터를 조작하는 명령어**로 나눌 수 있다.

| 구분 | DDL | DML |
|---|---|---|
| 의미 | Data Definition Language | Data Manipulation Language |
| 목적 | DB 구조 정의 | Data 조회 및 수정 |
| 대상 | 테이블, 스키마, 인덱스 등 | 테이블 내부 데이터 |
| 트랜잭션 | 대부분 자동 커밋 | 적용 가능 |

---

### 1.1) DDL(Data Definition Language)

`DDL`은 데이터베이스의 구조를 정의하거나 변경하는 명령어이다.
테이블, 스키마, 인덱스와 같은 **DB 객체를 생성하거나 수정하는데 사용**된다.

### 대표적 DDL 명령어

`CREATE` : 새로운 데이터베이스 객체를 생성한다
```sql
CREATE TABLE users (
    id INT PRIMARY KEY,
    name VARCHAR(50),
    email VARCHAR(100)
);
```
`ALTER` : 기존 테이블 구조를 수정한다.
```sql
ALTER TABLE users ADD COLUMN age INT;
```
`DROP` : 데이터베이스 객체를 삭제한다.
```sql
DROP TABLE users;
```
`TRUNCATE` : 테이블의 모든 데이터를 삭제하되 구조는 유지한다.
```sql
TRUNCATE TABLE users;
```
위와 같은 `DDL` 명령어는 대부분 실행 즉시 데이터베이스 구조에 반영되며 **자동으로 커밋**되는 특징을 가진다.

---

### 1.2) DML(Data Manipulation Language)

`DML`은 데이터베이스에 저장된 **실제 데이터를 조회하거나 변경하는 명령어**이다.

### 대표적 DML 명령어
 
`SELECT` : 데이터를 조회한다.
```sql
SELECT * FROM users;
```
`INSERT` : 새로운 데이터를 삽입한다.
```sql
INSERT INTO users (id, name, email)
VALUES (1, 'Seongjun', 'test@email.com');
```
`UPDATE` : 기존 데이터를 수정한다.
```sql
UPDATE users
SET name = 'Seongyoung'
WHERE id = 1;
```
`DELETE` : 데이터를 삭제한다.
```sql
DELETE FROM users WHERE id = 1;
```
`DML` 명령어는 트랜잭션을 통해 `COMMIT` 또는 `ROLLBACK`이 가능하다는 특징이 있다.

---

## 2. 역정규화(Denormalization)

데이터베이스 설계에서 정규화는 데이터 중복을 줄이고 무결성을 유지하기 위한 중요한 과정이다.
하지만 정규화가 지나치게 이루어지면 테이블이 여러 개로 분리되어 **JOIN 연산이 증가하고 성능이 저하**될 수 있다.

이러한 문제를 해결하기 위해 사용되는 방법이 `역정규화(Denormalization)`이다.

역정규화는 성능 개선을 위해 **의도적으로 데이터 중복을 허용하거나 테이블 구조를 단순화하는 설계 방식**이다.

### 역정규화가 필요한 상황

다음과 같은 상황에서 `역정규화`가 고려된다.
  - JOIN 연산이 과도하게 발생하는 경우
  - 조회 성능이 중요한 서비스
  - 읽기(Read) 작업이 많은 시스템
  - 집계 데이터가 자주 사용되는 경우

예를 들어 주문 시스템에서 사용자 정보와 주문 정보를 자주 함께 조회한다면, 일부 사용자 정보를 주문 테이블에 중복 저장할 수 있다.

### 적용 시 고려사항

`역정규화`를 적용할 때는 다음과 같은 사항을 고려해야 한다.
  - 데이터 중복으로 인한 데이터 불일치 가능성
  - 데이터 수정 시 동기화 문제
  - 데이터 무결성 유지 여부
  - 성능 개선 효과

즉, `역정규화`는 단순히 구조를 줄이기 위한 것이 아니라 **성능과 데이터 무결성 사이의 균형을 고려하여 적용**해야 한다.

### 역정규화의 장단점

#### 장점
  - JOIN 연산 감소
  - 조회 성능 향상
  - 쿼리 구조 단순화

#### 단점
  - 데이터 중복 발생
  - 데이터 수정 시 복잡성 증가
  - 데이터 무결성 관리 어려움

---

### 정리

`SQL`에서는 데이터베이스 구조를 정의하는 `DDL`과 데이터를 조작하는 `DML` 명령어가 구분된다.
또한 데이터베이스 설계에서는 데이터 중복을 최소화하기 위한 `정규화`가 기본 원칙이지만, 시스템 성능을 고려하여 `역정규화`를 적용하기도 한다.
결국 데이터베이스 설계에서는 **데이터 무결성과 시스템 성능 사이의 균형을 찾는 것이 중요**하다.
