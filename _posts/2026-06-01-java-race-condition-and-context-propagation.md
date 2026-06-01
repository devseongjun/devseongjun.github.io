---
title: "Race Condition과 비동기 환경의 Context 전파 이해하기"
categories:
  - Backend
  - Java
tags:
  - Java
  - Thread
  - RaceCondition
  - Synchronization
  - Lock
  - Atomic
  - ConcurrentHashMap
  - ThreadLocal
  - MDC
  - SecurityContext
  - SpringSecurity
  - Async
---

멀티스레드 환경에서는 여러 Thread가 동시에 실행되면서 성능을 높일 수 있다.
하지만 여러 Thread가 같은 데이터를 함께 읽고 수정하면 예상하지 못한 결과가 발생할 수 있다.

대표적인 문제가 바로 `Race Condition`이다.

또한 Spring 애플리케이션에서 비동기 처리를 사용할 때는
`MDC(Logback Mapped Diagnostic Context)`나 `SecurityContext`처럼
현재 요청과 관련된 Context 정보가 다른 Thread로 자동 전달되지 않는 문제도 자주 만난다.

이 글에서는 다음 두 가지를 정리한다.

```text
1. Race Condition이 무엇이고 어떻게 해결할 수 있는지
2. 비동기 환경에서 MDC, SecurityContext 같은 Context를 어떻게 전달할 수 있는지
```

핵심은 하나다.

```text
Thread를 나누면 실행 흐름도 나뉘고,
ThreadLocal 기반 Context도 함께 자동 공유되지 않는다.
```

---

## 1. Race Condition이란?

`Race Condition`은 여러 Thread가 공유 자원에 동시에 접근하고,
실행 순서에 따라 결과가 달라지는 문제를 말한다.

예를 들어 여러 사용자가 동시에 같은 상품의 재고를 차감하는 상황을 생각해보자.

```java
public class Stock {

    private int quantity = 10;

    public void decrease() {
        if (quantity > 0) {
            quantity--;
        }
    }
}
```

코드만 보면 재고가 0보다 클 때만 감소하므로 안전해 보인다.
하지만 멀티스레드 환경에서는 다음과 같은 일이 발생할 수 있다.

```text
Thread A: quantity가 1인지 확인
Thread B: quantity가 1인지 확인
Thread A: quantity를 0으로 감소
Thread B: quantity를 다시 감소
```

두 Thread가 거의 동시에 `quantity > 0` 조건을 통과하면,
실제로는 재고가 하나뿐이었는데 두 번 차감될 수 있다.

이 문제의 원인은 `quantity--`가 하나의 작업처럼 보여도 실제로는 여러 단계로 나뉘기 때문이다.

```text
1. 현재 값을 읽는다.
2. 값을 1 감소시킨다.
3. 감소된 값을 다시 저장한다.
```

즉, 읽기와 수정 사이에 다른 Thread가 끼어들 수 있다.

---

## 2. Race Condition이 발생하는 조건

Race Condition은 보통 다음 조건이 함께 있을 때 발생한다.

- 여러 Thread가 동시에 실행된다.
- 둘 이상의 Thread가 같은 공유 자원에 접근한다.
- 공유 자원에 대한 읽기와 수정이 함께 일어난다.
- 실행 순서에 따라 결과가 달라질 수 있다.

단순히 여러 Thread를 사용한다고 무조건 문제가 생기는 것은 아니다.
문제는 **공유되는 변경 가능한 상태(Mutable Shared State)** 가 있을 때 발생한다.

```text
읽기만 하는 데이터 = 비교적 안전
여러 Thread가 수정하는 데이터 = 동기화 필요
```

---

## 3. synchronized로 임계 영역 보호하기

가장 기본적인 해결 방법은 `synchronized`를 사용해 한 번에 하나의 Thread만 공유 자원에 접근하도록 제한하는 것이다.

```java
public class Stock {

    private int quantity = 10;

    public synchronized void decrease() {
        if (quantity > 0) {
            quantity--;
        }
    }
}
```

`synchronized`가 붙은 메서드는 같은 객체 기준으로 한 Thread만 실행할 수 있다.
즉, `decrease()` 실행 중에는 다른 Thread가 같은 객체의 `decrease()`에 동시에 들어올 수 없다.

장점은 문법이 단순하다는 것이다.
하지만 임계 영역이 커질수록 대기 시간이 길어지고 성능이 떨어질 수 있다.

```text
임계 영역(Critical Section)
= 여러 Thread가 동시에 접근하면 안 되는 코드 영역
```

---

## 4. Lock 사용하기

`ReentrantLock`을 사용하면 `synchronized`보다 더 세밀하게 Lock을 제어할 수 있다.

```java
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class Stock {

    private final Lock lock = new ReentrantLock();
    private int quantity = 10;

    public void decrease() {
        lock.lock();
        try {
            if (quantity > 0) {
                quantity--;
            }
        } finally {
            lock.unlock();
        }
    }
}
```

`Lock`을 사용할 때는 반드시 `finally`에서 `unlock()`을 호출해야 한다.
중간에 예외가 발생했는데 Lock이 해제되지 않으면 다른 Thread가 계속 대기할 수 있다.

`ReentrantLock`은 다음과 같은 기능이 필요할 때 유용하다.

- Lock 획득 시도(`tryLock`)
- 대기 시간 제한
- 공정성 옵션
- 조건 변수(`Condition`) 사용

단, 단순한 동기화라면 `synchronized`가 더 읽기 쉽고 충분한 경우도 많다.

---

## 5. Atomic 클래스 사용하기

단순한 숫자 증가, 감소처럼 비교적 작은 연산은 `AtomicInteger` 같은 Atomic 클래스로 해결할 수 있다.

```java
import java.util.concurrent.atomic.AtomicInteger;

public class Counter {

    private final AtomicInteger count = new AtomicInteger(0);

    public int increment() {
        return count.incrementAndGet();
    }
}
```

`AtomicInteger`는 내부적으로 `CAS(Compare-And-Swap)` 기반의 원자적 연산을 제공한다.
즉, 여러 Thread가 동시에 값을 변경해도 중간에 값이 꼬이지 않도록 보장한다.

다만 Atomic 클래스가 모든 동시성 문제를 해결해주는 것은 아니다.

```java
if (stock.get() > 0) {
    stock.decrementAndGet();
}
```

위 코드는 `get()`과 `decrementAndGet()`이 각각은 원자적이어도,
두 작업 전체가 하나의 원자적 연산은 아니다.
조건 확인과 차감을 하나의 단위로 보호해야 한다면 `synchronized`, `Lock`, DB Lock 등을 고려해야 한다.

---

## 6. Concurrent Collection 사용하기

여러 Thread가 Collection을 함께 사용할 때는 일반 `HashMap`, `ArrayList` 대신
동시성 처리가 고려된 Collection을 사용할 수 있다.

대표적으로 `ConcurrentHashMap`이 있다.

```java
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ConcurrentMap;

public class ViewCounter {

    private final ConcurrentMap<Long, Integer> viewCounts = new ConcurrentHashMap<>();

    public void increase(Long postId) {
        viewCounts.merge(postId, 1, Integer::sum);
    }
}
```

`ConcurrentHashMap`은 멀티스레드 환경에서 일반 `HashMap`보다 안전하게 사용할 수 있다.
또한 `merge`, `compute`, `putIfAbsent` 같은 메서드를 사용하면
조회 후 수정하는 작업을 더 안전하게 표현할 수 있다.

주의할 점은 Collection 자체가 Thread-safe하다고 해서
그 안에 들어있는 객체의 모든 상태 변경까지 자동으로 안전해지는 것은 아니라는 점이다.

---

## 7. 불변 객체와 상태 분리

가장 좋은 해결책은 애초에 공유되는 변경 가능한 상태를 줄이는 것이다.

```java
public record Money(long amount) {

    public Money add(Money other) {
        return new Money(this.amount + other.amount);
    }
}
```

`record`나 불변 객체를 사용하면 객체 생성 후 상태가 바뀌지 않는다.
상태가 바뀌지 않으면 여러 Thread가 동시에 읽어도 Race Condition이 발생할 가능성이 줄어든다.

실무에서는 다음과 같은 설계가 도움이 된다.

- 가능한 한 지역 변수 사용
- 공유 필드 최소화
- DTO, Value Object는 불변으로 설계
- 상태 변경은 한 계층이나 한 트랜잭션 안으로 모으기

동시성 문제는 코드 몇 줄로만 해결하기보다,
공유 상태를 줄이는 설계가 함께 필요하다.

---

## 8. DB Lock과 Transaction으로 해결하기

Spring Boot 애플리케이션에서는 재고, 포인트, 쿠폰처럼 데이터 정합성이 중요한 작업을
메모리 Lock만으로 해결하면 부족할 수 있다.

서버가 여러 대라면 각 서버의 JVM Lock은 서로 공유되지 않기 때문이다.

```text
서버 A의 synchronized != 서버 B의 synchronized
```

이런 경우에는 DB 수준의 동시성 제어가 필요하다.

대표적인 방식은 다음과 같다.

- 비관적 Lock(Pessimistic Lock)
- 낙관적 Lock(Optimistic Lock)
- Unique Constraint
- Transaction Isolation Level 조정

예를 들어 JPA에서 비관적 Lock은 다음처럼 사용할 수 있다.

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("select s from Stock s where s.id = :id")
Optional<Stock> findByIdForUpdate(Long id);
```

비관적 Lock은 다른 트랜잭션이 해당 row를 동시에 수정하지 못하게 막는다.
정합성은 강해지지만 대기 시간이 길어질 수 있다.

낙관적 Lock은 `@Version`을 사용한다.

```java
@Entity
public class Stock {

    @Id
    private Long id;

    private int quantity;

    @Version
    private Long version;
}
```

낙관적 Lock은 먼저 막지 않고, 수정 시점에 version이 바뀌었는지 확인한다.
충돌이 적은 환경에서는 성능상 유리할 수 있지만,
충돌 발생 시 재시도 로직을 설계해야 한다.

---

## 9. Race Condition 해결 전략 정리

Race Condition을 해결하는 방법은 하나만 있는 것이 아니다.
문제의 범위와 데이터 정합성 요구사항에 따라 선택해야 한다.

| 전략 | 사용 상황 | 주의점 |
| --- | --- | --- |
| `synchronized` | 단일 JVM 안의 간단한 임계 영역 보호 | 서버가 여러 대면 JVM 밖 데이터는 보호하지 못함 |
| `ReentrantLock` | Lock 획득 시도, 대기 시간 제한 등 세밀한 제어 필요 | `unlock()` 누락 주의 |
| `AtomicInteger` | 단순 카운터, 증가/감소 연산 | 복합 조건 로직은 별도 보호 필요 |
| `ConcurrentHashMap` | 멀티스레드 환경의 Map 사용 | 내부 객체 상태 변경까지 자동 보호되지는 않음 |
| 불변 객체 | 공유 상태 자체를 줄이고 싶을 때 | 상태 변경이 필요한 도메인은 별도 설계 필요 |
| DB Lock | 여러 서버에서 같은 DB 데이터를 수정할 때 | 성능, Deadlock, 재시도 전략 고려 |
| Unique Constraint | 중복 생성 방지 | 예외 처리와 사용자 응답 설계 필요 |

면접에서는 다음처럼 답변할 수 있다.

```text
Race Condition은 여러 Thread가 공유 자원에 동시에 접근하고 수정할 때,
실행 순서에 따라 결과가 달라지는 문제입니다.
해결 방법으로는 synchronized, ReentrantLock, Atomic 클래스,
Concurrent Collection, 불변 객체 설계가 있고,
여러 서버가 같은 데이터를 수정하는 경우에는 DB Lock이나 Unique Constraint,
Transaction Isolation 같은 DB 수준의 제어가 필요합니다.
```

---

## 10. ThreadLocal과 Context 정보

Spring 애플리케이션에서는 요청 단위의 정보를 `ThreadLocal` 기반으로 관리하는 경우가 많다.

대표적인 예시는 다음과 같다.

- `MDC`: 로그에 requestId, traceId, userId 같은 값 출력
- `SecurityContext`: 현재 인증된 사용자 정보 저장
- `RequestContextHolder`: 현재 HTTP request 관련 정보 저장

`ThreadLocal`은 이름 그대로 Thread마다 독립적인 저장 공간을 제공한다.

```text
Thread A의 ThreadLocal 값과
Thread B의 ThreadLocal 값은 서로 다르다.
```

이 구조 덕분에 같은 서버에서 여러 요청이 동시에 처리되어도,
각 요청의 인증 정보나 로그 추적 정보를 분리해서 관리할 수 있다.

하지만 비동기 처리를 시작하면 문제가 생긴다.

```java
@Async
public void sendMail() {
    log.info("메일 발송");
}
```

`@Async` 메서드는 기존 요청 Thread가 아니라 별도의 Thread Pool에서 실행된다.
따라서 기존 Thread에 들어 있던 `MDC`나 `SecurityContext`가 자동으로 전달되지 않는다.

---

## 11. MDC가 비동기 환경에서 사라지는 이유

예를 들어 요청마다 `traceId`를 MDC에 넣고 로그를 남긴다고 가정해보자.

```java
MDC.put("traceId", traceId);
log.info("요청 시작");
```

동일 Thread 안에서는 로그 패턴에 `traceId`가 잘 출력된다.
하지만 비동기 작업이 다른 Thread에서 실행되면 MDC 값이 비어 있을 수 있다.

```text
request-thread-1
  - MDC traceId = abc-123
  - async 작업 실행 요청

async-thread-1
  - MDC traceId 없음
```

MDC는 보통 `ThreadLocal` 기반이기 때문에,
새 Thread나 Thread Pool의 다른 Thread로 자동 복사되지 않는다.

더 위험한 문제도 있다.
Thread Pool은 Thread를 재사용한다.
이전 작업의 MDC를 지우지 않으면 다음 작업 로그에 잘못된 traceId가 찍힐 수 있다.

그래서 비동기 환경에서는 Context를 전달하는 것뿐 아니라,
작업이 끝난 뒤 반드시 정리하는 것도 중요하다.

---

## 12. TaskDecorator로 MDC 전달하기

Spring에서는 `TaskDecorator`를 사용해 비동기 작업 실행 전후에 Context를 복사하고 정리할 수 있다.

```java
import java.util.Map;
import org.slf4j.MDC;
import org.springframework.core.task.TaskDecorator;

public class MdcTaskDecorator implements TaskDecorator {

    @Override
    public Runnable decorate(Runnable runnable) {
        Map<String, String> contextMap = MDC.getCopyOfContextMap();

        return () -> {
            try {
                if (contextMap != null) {
                    MDC.setContextMap(contextMap);
                }
                runnable.run();
            } finally {
                MDC.clear();
            }
        };
    }
}
```

그리고 `ThreadPoolTaskExecutor`에 등록한다.

```java
import java.util.concurrent.Executor;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.annotation.EnableAsync;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;

@EnableAsync
@Configuration
public class AsyncConfig {

    @Bean(name = "applicationTaskExecutor")
    public Executor applicationTaskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("app-async-");
        executor.setTaskDecorator(new MdcTaskDecorator());
        executor.initialize();
        return executor;
    }
}
```

이렇게 하면 비동기 작업을 제출한 Thread의 MDC 값을 복사해서
실제 비동기 작업을 실행하는 Thread에 설정할 수 있다.

중요한 점은 `finally`에서 `MDC.clear()`를 호출하는 것이다.
Thread Pool의 Thread는 재사용되기 때문에 Context 정리를 빼먹으면 로그가 섞일 수 있다.

---

## 13. SecurityContext 전달하기

Spring Security의 `SecurityContextHolder`도 기본적으로 ThreadLocal 기반으로 동작한다.
따라서 비동기 Thread에서는 인증 정보가 사라질 수 있다.

Spring Security는 이를 위해 Delegating 계열의 클래스를 제공한다.

대표적으로 다음 클래스를 사용할 수 있다.

- `DelegatingSecurityContextRunnable`
- `DelegatingSecurityContextCallable`
- `DelegatingSecurityContextExecutor`
- `DelegatingSecurityContextAsyncTaskExecutor`

`@Async`에서 사용할 Executor를 감쌀 때는 다음과 같이 구성할 수 있다.

```java
import java.util.concurrent.Executor;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.annotation.EnableAsync;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;
import org.springframework.security.task.DelegatingSecurityContextAsyncTaskExecutor;

@EnableAsync
@Configuration
public class AsyncSecurityConfig {

    @Bean(name = "securityTaskExecutor")
    public Executor securityTaskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("security-async-");
        executor.initialize();

        return new DelegatingSecurityContextAsyncTaskExecutor(executor);
    }
}
```

이 방식은 현재 Thread의 `SecurityContext`를 비동기 작업으로 전달해준다.

```java
@Async("securityTaskExecutor")
public void auditLoginHistory() {
    Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
    log.info("현재 사용자: {}", authentication.getName());
}
```

주의할 점은 비동기 작업이 요청보다 늦게 실행될 수 있다는 것이다.
따라서 정말 필요한 최소 정보만 전달하는 것이 좋다.
예를 들어 전체 인증 객체가 아니라 `userId`만 명시적으로 넘기는 방식이 더 단순하고 안전할 수 있다.

---

## 14. MDC와 SecurityContext를 함께 전달하기

실무에서는 로그 추적을 위한 MDC와 인증 정보를 함께 전달해야 할 때가 있다.
이 경우 다음 두 가지 방향을 고려할 수 있다.

첫 번째는 Executor를 감싸는 방식이다.

```java
ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
executor.setTaskDecorator(new MdcTaskDecorator());
executor.initialize();

return new DelegatingSecurityContextAsyncTaskExecutor(executor);
```

`TaskDecorator`로 MDC를 복사하고,
`DelegatingSecurityContextAsyncTaskExecutor`로 SecurityContext를 전달한다.

두 번째는 필요한 값을 명시적으로 파라미터로 전달하는 방식이다.

```java
public void request() {
    Long userId = currentUserId();
    String traceId = MDC.get("traceId");

    asyncService.process(userId, traceId);
}
```

```java
@Async
public void process(Long userId, String traceId) {
    MDC.put("traceId", traceId);
    try {
        log.info("userId={}", userId);
    } finally {
        MDC.clear();
    }
}
```

명시적 전달 방식은 코드가 조금 길어질 수 있지만,
비동기 작업이 어떤 값에 의존하는지 분명하게 드러난다.
특히 오래 걸리는 배치성 작업이나 메시지 큐 기반 작업에서는 명시적 전달이 더 적합한 경우가 많다.

---

## 15. InheritableThreadLocal은 조심해서 사용하기

`InheritableThreadLocal`을 사용하면 부모 Thread의 값을 자식 Thread로 상속할 수 있다.
그래서 Context 전파 문제를 쉽게 해결할 수 있어 보인다.

하지만 Thread Pool 환경에서는 주의해야 한다.

Thread Pool은 매번 새 Thread를 만드는 것이 아니라 기존 Thread를 재사용한다.
따라서 Thread 생성 시점의 값이 기대와 다를 수 있고,
이전 작업의 값이 남아 Context 오염이 발생할 수 있다.

```text
일반적으로 Spring @Async, ExecutorService 환경에서는
InheritableThreadLocal에 의존하기보다 TaskDecorator나 DelegatingSecurityContext 계열을 사용하는 편이 안전하다.
```

---

## 16. 실무에서의 선택 기준

비동기 환경에서 Context를 전달할 때는 다음 기준으로 선택하면 좋다.

| 상황 | 추천 방식 |
| --- | --- |
| 로그 traceId만 전달 | `TaskDecorator`로 MDC 복사 |
| 인증 정보가 필요한 짧은 비동기 작업 | `DelegatingSecurityContextAsyncTaskExecutor` |
| 오래 걸리는 작업 또는 메시지 큐 작업 | 필요한 값만 DTO나 파라미터로 명시적 전달 |
| 여러 Context를 공통으로 전파 | Executor 공통 설정 구성 |
| Thread Pool 사용 | 작업 종료 후 반드시 Context 정리 |

특히 보안 정보는 무조건 전달하기보다,
비동기 작업에 정말 필요한지 먼저 판단해야 한다.
필요한 값이 `userId`뿐이라면 전체 `Authentication` 객체를 넘기지 않아도 된다.

---

## 마무리

`Race Condition`은 여러 Thread가 공유 자원을 동시에 수정할 때 발생하는 대표적인 동시성 문제다.
이를 해결하기 위해서는 `synchronized`, `Lock`, `Atomic`, `Concurrent Collection`, 불변 객체, DB Lock 등
문제 범위에 맞는 전략을 선택해야 한다.

비동기 환경의 Context 전파 문제도 같은 맥락에서 이해할 수 있다.
`MDC`와 `SecurityContext`는 보통 `ThreadLocal` 기반이기 때문에,
Thread가 바뀌면 기존 요청의 Context가 자동으로 따라가지 않는다.

따라서 `TaskDecorator`, `DelegatingSecurityContextAsyncTaskExecutor`,
명시적 파라미터 전달 등을 사용해 필요한 Context를 전달하고,
Thread Pool 환경에서는 작업이 끝난 뒤 반드시 Context를 정리해야 한다.

면접에서는 다음 흐름으로 답변하면 좋다.

```text
Race Condition은 공유 자원에 대한 동시 접근으로 실행 순서에 따라 결과가 달라지는 문제입니다.
단일 JVM에서는 synchronized, Lock, Atomic, Concurrent Collection 등으로 제어할 수 있고,
분산 서버 환경에서는 DB Lock, Unique Constraint, Transaction Isolation 같은 저장소 수준의 제어가 필요합니다.

MDC나 SecurityContext는 ThreadLocal 기반이기 때문에 @Async처럼 Thread가 바뀌는 환경에서는 자동 전파되지 않습니다.
MDC는 TaskDecorator로 복사하고 finally에서 clear해야 하며,
SecurityContext는 DelegatingSecurityContextAsyncTaskExecutor 같은 Spring Security 지원 클래스를 사용할 수 있습니다.
다만 보안상 필요한 최소 정보만 명시적으로 전달하는 방식도 좋은 선택입니다.
```

---
