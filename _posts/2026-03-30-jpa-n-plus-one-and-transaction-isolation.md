---
title: "JPA N+1 문제와 트랜잭션 격리 수준(Isolation) 이해하기"
categories:
  - Backend
  - Database
tags:
  - JPA
  - Hibernate
  - N+1
  - Transaction
  - ACID
  - Isolation
  - Spring
---

`JPA`를 사용한 데이터 접근에서는 편리함 이면에 성능 문제와 트랜잭션 관련 이슈가 함께 존재한다.
특히 `N+1`문제는 `JPA`를 처음 사용할 때 가장 많이 겪는 성능 문제 중 하나이며,
트랜잭션의 **격리성(Isolation)**은 데이터 정합성과 직결되는 중요한 개념이다.

이 글에서는 `JPA`에서 발생하는 `N+1` 문제의 원인과 해결 방법을 살펴보고,
트랜잭션의 `ACID` 속성 중 격리성이 보장되지 않을 때 발생하는 문제와
이를 해결하기 위한 격리 수준을 정리한다.

---

## 1. JPA에서 발생하는 N+1 문제

### N+1 문제란?
`N+1` 문제는 **연관된 엔티티를 조회할 때 발생하는 비효율적인 쿼리 실행 문제**이다.

예를 들어 게시글(Post)과 작성자(User)가 연관 관계일 때,
게시글 목록을 조회하는 상황을 가정해보자.

```java
List<Post> posts = postRepository.findAll();
```
이때 다음과 같은 쿼리가 실행될 수 있다.
```sql
SELECT * FROM post;
SELECT * FROM user WHERE id = ?;
```
처음 1번 조회 + 각 데이터마다 추가 조회 N번이 발생하며,
총 `N+1`번 쿼리가 발생하게 된다.

---

### N+1 문제 발생 원인
`N+1`문제는 `JPA`의 **지연 로딩(Lazy Loading)**과 연관 관계 처리 방식 때문에 발생한다.
- @ManyToOne(fetch = FetchType.LAZY)
- @OneToMany(fetch = FetchType.LAZY)

연관 엔티티는 실제로 접근하는 시점에 조회되기 때문에, 루프를 돌면서 접근하면 쿼리가 반복 실행된다.

또한 `EAGER` 전략을 사용하더라도 내부적으로 쿼리가 분리될 수 있어 `N+1` 문제가 발생할 수 있다.

---

### N+1 문제 해결 방법

#### 1. Fetch Join 사용
```java
@Query("SELECT p FROM Post p JOIN FETCH p.user")
List<Post> findAllWithUser();
```
- 한 번의 쿼리로 연관 엔티티까지 조회
- 가장 대표적 해결 방법

#### 2. EntityGraph 사용
```java
@EntityGraph(attributePaths = {"user"})
List<Post> findAll();
``` 
- `JPQL` 없이도 연관 엔티티 함께 조회 가능

#### 3. Batch Size 설정
```java
@BatchSize(size = 100)
```
또는
```yaml
spring:
  jpa:
    properties:
      hibernate.default_batch_fetch_size: 100
```
- IN 쿼리로 묶어서 조회
- 완전한 해결은 아니지만 성능 개선

### 정리
`N+1` 문제는 `JPA`의 지연 로딩으로 인해 발생하며, 위의 제시된 방법들을 통해 해결할 수 있다.

---

## 2. 트랜잭션 격리성(Isolation)과 격리 수준

### Isolation이란?
트랜잭션의 `ACID` 속성 중 **격리성(Isolation)**은 
여러 트랜잭션이 동시에 실행될 때 서로 영향을 주지 않도록 하는 성질이다.

즉, 하나의 트랜잭션이 수행 중일 때 다른 트랜잭션이 중간 상태를 볼 수 없게 보장하는 것이다.

### 격리성이 보장되지 않을 때 발생하는 문제

#### 1. Dirty Read
- 커밋되지 않은 데이터를 읽는 문제

#### 2. Non-Repeatable Read
- 같은 데이터를 두 번 조회했을 때 값이 달라지는 문제

#### 3. Phantom Read
- 같은 조건으로 조회했는데 결과 행이 달라지는 문제

---

### 트랜잭션 격리 수준(Level)

#### 1. READ UNCOMMITTED
- `Dirty Read` 허용
- 가장 낮은 수준

#### 2. READ COMMITTED
- 커밋된 데이터만 읽음
- `Dirty Read` 방지
- 대부분 `DB` 기본값

#### 3. REPEATABLE READ
- 같은 데이터는 항상 동일하게 조회됨
- `Non-Repeatable Read` 방지

#### 4. SERIALIZABLE
- 가장 높은 격리 수준
- 모든 트랜잭션을 순차적으로 실행한 것처럼 보장
- 성능 비용이 큼

### 정리
격리 수준이 높을수록 데이터 정합성은 높아지지만, 동시성 성능은 낮아진다.

---

## 마무리
`JPA`의 `N+1` 문제와 트랜잭션 격리성은 각각 **성능**과 **데이터 정합성**을 대표하는 개념이다.
효율적인 데이터 접근을 위해서는 쿼리 실행 방식(N+1)을 이해해야 하고,
안정적인 데이터 처리를 위해서는 트랜잭션 격리 수준을 적절히 선택해야 한다.

---