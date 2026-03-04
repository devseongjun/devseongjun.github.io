---
layout: post
title: "SOAP에서 REST로의 전환과 Spring Boot 요청 처리 흐름 이해하기"
categories: [Backend, Spring]
tags: [Spring, SpringBoot, REST, SOAP, HttpMessageConverter, MVC, RestController, Java]
---

웹 API는 시간이 지나면서 여러 방식으로 발전해왔다.
초기에는 `SOAP`(Simple Object Access Protocol) 기반의 웹 서비스가 널리 사용되었지만,
현대 웹 환경에서는 `REST`(Representational State Transfer) 방식이 사실상 표준처럼 사용되고 있다.

`REST API`는 HTTP 프로토콜을 기반으로 자원을 표현하고,
JSON과 같은 경량 데이터 형식을 활용하여 효율적인 통신을 가능하게 한다.
Spring Boot는 이러한 REST API를 쉽게 구현할 수 있도록 다양한 기능을 제공하는데,
그 핵심 중 하나가 `@RestController`와 `HTTP 메시지 컨버터`(HttpMessageConverter) 이다.

이 글에서는 먼저 SOAP에서 REST로 전환된 이유를 살펴보고,
이어서 Spring Boot에서 @RestController로 들어온 HTTP 요청이 
어떻게 처리되고 응답으로 변환되는지 전체 흐름을 정리해본다.

---

## 1. SOAP에서 REST로 전환된 이유

`SOAP`는 XML 기반 메시지 프로토콜로, 엄격한 표준과 다양한 확장 기능을 제공하는 웹 서비스 방식이다.
`SOAP`는 WS-Security, WS-Transaction 등 다양한 표준을 통해
높은 신뢰성과 보안성을 제공하며 기업 시스템 간 통신에서 널리 사용되었다.

하지만 `SOAP`는 아래와 같은 단점을 내포하고 있다.
  - XML 기반 메세지로 인한 높은 네트워크 비용
  - 메시지 구조의 복잡성
  - 구현 난이도의 증가
  - 웹 환경과의 낮은 친화도

이러한 문제를 해결하기 위해 등장한 것이 **REST 아키텍처 스타일**이다. 
`REST`는 HTTP 프로토콜의 특징을 그대로 활용하며, `URI`를 통해 자원을 표현하고 
HTTP 메서드(GET, POST, PUT, DELETE)를 통해 자원을 조작한다.

또한 `REST`는 JSON과 같은 경량 데이터 형식을 사용하기 때문에 
네트워크 효율이 높고, 웹 환경에 자연스럽게 통합될 수 있다.
이러한 이유로 `REST`는 모바일 환경과 마이크로서비스 아키텍처에서 널리 사용되는 API 방식이 되었다.

---

## 2. Spring Boot에서의 @RestController 요청 처리 흐름

`Spring Boot`에서 `REST API` 요청은 `Spring MVC` 구조를 기반으로 처리된다.

요청이 들어오면 다음과 같은 흐름으로 처리된다.
  1. Client → HTTP Request
  2. DispatcherServlet (Front Controller)
  3. HandlerMapping (컨트롤러 탐색)
  4. HandlerAdapter (메서드 실행)
  5. Controller (@RestController)
  6. HttpMessageConverter (객체 ↔ JSON 변환)
  7. HTTP Response 반환

### 2.1) DispatcherServlet

Spring MVC의 핵심 컴포넌트는 `DispatcherServlet`이다.

`DispatcherServlet`은 Front Controller 패턴을 구현한 서블릿으로,
모든 HTTP 요청을 가장 먼저 받아 전체 처리 과정을 제어한다.

클라이언트가 다음과 같은 요청을 보낸다고 가정해보자.
```
GET /api/users/1
```
이 요청은 먼저 `DispatcherServlet`이 수신하게 된다.

### 2.2) HandlerMapping

`DispatcherServlet`은 요청을 처리할 Controller 메서드를 찾기 위해 `HandlerMapping`을 사용한다.

예를 들어 다음과 같은 컨트롤러가 있다고 가정해보자.
```java

@RestController
@RequestMapping("/api/users")
public class UserController {

    @GetMapping("/{id}")
    public UserResponse getUser(@PathVariable Long id) {
        return new UserResponse(id, "Seongjun");
    }
}

```
위에서 `HandlerMapping`은 /api/users/1 요청을 `getUser()` 메서드와 매핑한다.

### 2.3) HandlerAdapter

Controller가 결정되면 `HandlerAdapter`가 실제로 해당 메서드를 실행할 수 있도록 호출을 수행한다.

이 과정에서 다음과 같은 파라미터들이 처리된다.
  - @PathVariable
  - @RequestParam
  - @RequestBody

특히 `@RequestBody`가 있을 경우 HTTP 메시지 컨버터가 동작하게 된다.

--- 

## 3. HTTP 메시지 컨버터의 역할

`HTTP 메시지 컨버터`는 HTTP 요청/응답 본문과 Java 객체 사이의 변환을 담당하는 컴포넌트이다.
Spring Boot에서는 기본적으로 Jackson 기반 JSON 컨버터가 사용된다.

대표적인 구현체는 다음과 같다.
```
MappingJackson2HttpMessageConverter
```

### 3.1) 요청 처리 시 동작 (역직렬화)

클라이언트가 `JSON` 데이터를 보내는 경우를 예로 들어보자.
```
POST /api/users
Content-Type: application/json
```
이때 요청 바디는 아래와 같다.
```json

{
  "name": "Seongjun"
}

```
이제 컨트롤러가 다음과 같이 정의되어 있다고 해보자.
```java

@PostMapping
public UserResponse createUser(@RequestBody UserRequest request) {
    return new UserResponse(1L, request.getName());
}

```
위에서 HTTP 메시지 컨버터가 `JSON` 데이터를 읽어서 `UserRequest` 객체로 변환하는 것을 볼 수 있다.

즉, `JSON` 데이터가 `Java` 객체로의 변환이 일어나는 것이다.

### 3.2) 응답 처리 시 동작 (직렬화)

컨트롤러가 아래와 같은 객체를 반환한다고 가정해보자.
```java

return new UserResponse(1L, "Seongjun");

```
HTTP 메시지 컨버터는 이 객체를 `JSON`으로 변환하게 되고,

응답은 다음과 같은 형태가 된다.
```json

{
  "id": 1,
  "name": "Seongjun"
}

```

---

#### 요청 처리 흐름 정리

Spring Boot에서 `@RestController` 요청 처리 과정은 다음과 같이 정리할 수 있다.
  1.  클라이언트가 HTTP 요청 전송
  2.  DispatcherServlet이 요청 수신
  3.  HandlerMapping이 컨트롤러 메서드 탐색
  4.  HandlerAdapter가 컨트롤러 실행
  5.  HTTP 메시지 컨버터가 요청 데이터를 객체로 변환
  6.  컨트롤러 로직 실행
  7.  HTTP 메시지 컨버터가 반환 객체를 JSON으로 변환
  8.  클라이언트에게 HTTP 응답 전달

`SOAP`에서 `REST`로의 전환은 웹 환경이 단순성, 확장성, 경량화를 요구하게 되면서 이루어진 변화라고 볼 수 있다.
`REST`는 `HTTP` 기반의 구조를 활용해 효율적인 `API` 설계를 가능하게 했고, `Spring Boot`는 이러한 
`REST API`를 쉽게 구현할 수 있도록 요청 처리 흐름과 데이터 변환을 자동화하는 기능을 제공한다.

특히 `HTTP 메시지 컨버터`는 REST API에서 매우 중요한 역할을 하며,
개발자가 **직접 JSON 변환 코드를 작성하지 않아도 객체와 HTTP 메시지 사이의 변환을 자동으로 처리**해준다.
