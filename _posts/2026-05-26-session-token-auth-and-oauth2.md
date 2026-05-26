---
title: "Session 기반 인증과 Token 기반 인증, OAuth 2.0 Authorization Code Grant 이해하기"
categories:
  - Backend
  - Security
tags:
  - Authentication
  - Authorization
  - Session
  - Token
  - JWT
  - OAuth2
  - SpringSecurity
---

웹 애플리케이션에서 인증(Authentication)은 사용자가 누구인지 확인하는 과정이다.
대표적인 방식으로는 서버가 로그인 상태를 관리하는 `Session` 기반 인증과,
클라이언트가 인증 정보를 담은 `Token`을 요청마다 전달하는 Token 기반 인증이 있다.

또한 다른 서비스의 리소스에 안전하게 접근하기 위해서는 `OAuth 2.0` 같은 권한 위임(Authorization Delegation) 프로토콜을 사용한다.
특히 `Authorization Code Grant`는 웹 애플리케이션에서 가장 많이 사용되는 OAuth 2.0 흐름이며,
보안 관점에서도 반드시 이해해야 하는 방식이다.

이 글에서는 먼저 **Session 기반 인증과 Token 기반 인증의 차이점과 보안 고려사항**을 정리하고,
이어서 **OAuth 2.0의 주요 컴포넌트와 Authorization Code Grant 흐름**을 살펴본다.

---

## 1. Session 기반 인증

### Session 기반 인증이란?

`Session` 기반 인증은 사용자의 로그인 상태를 서버가 관리하는 방식이다.
사용자가 로그인에 성공하면 서버는 Session 저장소에 로그인 정보를 저장하고,
클라이언트에게는 Session을 식별할 수 있는 `Session ID`를 Cookie로 전달한다.

이후 클라이언트는 요청마다 Cookie에 담긴 Session ID를 서버로 보내고,
서버는 해당 Session ID를 기준으로 사용자의 인증 상태를 확인한다.

```text
1. 사용자 로그인 요청
2. 서버가 사용자 검증
3. 서버가 Session 저장소에 인증 정보 저장
4. 서버가 Session ID를 Cookie로 응답
5. 이후 요청마다 Cookie로 Session ID 전달
6. 서버가 Session ID로 사용자 인증 상태 확인
```

즉, 핵심은 **인증 상태가 서버에 저장된다**는 점이다.

---

### Session 기반 인증의 장점

Session 기반 인증은 서버가 로그인 상태를 직접 관리하기 때문에 제어가 쉽다.
예를 들어 사용자가 로그아웃하면 서버의 Session을 삭제하면 되고,
관리자가 특정 사용자의 Session을 강제로 만료시키는 것도 가능하다.

또한 민감한 사용자 정보가 클라이언트에 직접 저장되지 않는다.
클라이언트는 Session ID만 가지고 있고, 실제 인증 정보는 서버 Session 저장소에 있기 때문이다.

---

### Session 기반 인증의 단점

Session 기반 인증은 서버가 상태를 저장하는 `Stateful` 방식이다.
따라서 사용자가 많아질수록 Session 저장소에 대한 부담이 커질 수 있다.

또한 서버를 여러 대로 확장하는 경우 문제가 생길 수 있다.
예를 들어 사용자가 A 서버에서 로그인했는데 다음 요청이 B 서버로 전달되면,
B 서버는 해당 Session 정보를 모를 수 있다.

이 문제를 해결하려면 다음과 같은 방식이 필요하다.

- Sticky Session 사용
- Redis 같은 중앙 Session 저장소 사용
- Session Replication 구성

즉, Session 기반 인증은 단일 서버에서는 단순하지만,
서버 확장 환경에서는 Session 공유 전략을 함께 설계해야 한다.

---

### Session 기반 인증의 보안 고려사항

Session 기반 인증에서는 Session ID가 탈취되면 공격자가 사용자처럼 요청할 수 있다.
따라서 Cookie와 Session 관리가 중요하다.

#### 1. Cookie 보안 설정

Session ID는 보통 Cookie에 저장되므로 다음 옵션을 설정해야 한다.

- `HttpOnly`: JavaScript에서 Cookie에 접근하지 못하게 하여 XSS 피해를 줄임
- `Secure`: HTTPS 연결에서만 Cookie가 전송되도록 제한
- `SameSite`: CSRF 공격 가능성을 줄임

```http
Set-Cookie: JSESSIONID=abc123; HttpOnly; Secure; SameSite=Lax
```

#### 2. CSRF 방어

Session 기반 인증은 Cookie가 요청에 자동 포함된다는 특징이 있다.
이 때문에 공격자가 사용자를 악성 페이지로 유도하면,
사용자의 Cookie가 포함된 요청이 의도치 않게 서버로 전송될 수 있다.

이를 `CSRF(Cross-Site Request Forgery)`라고 한다.
방어 방법으로는 `CSRF Token`, `SameSite Cookie`, 중요 요청에 대한 재인증 등이 있다.

#### 3. Session Fixation 방어

`Session Fixation`은 공격자가 미리 알고 있는 Session ID를 사용자가 로그인하도록 유도한 뒤,
로그인 후 같은 Session ID로 접근하는 공격이다.

이를 막기 위해 로그인 성공 시 Session ID를 새로 발급해야 한다.
`Spring Security`는 기본적으로 로그인 성공 시 Session ID를 변경해 Session Fixation 공격을 방어한다.

#### 4. Session 만료 정책

Session은 무기한 유지되면 안 된다.
서비스 성격에 맞게 Idle Timeout과 Absolute Timeout을 설정해야 한다.

- Idle Timeout: 일정 시간 요청이 없으면 만료
- Absolute Timeout: 활동 여부와 관계없이 일정 시간이 지나면 만료

---

## 2. Token 기반 인증

### Token 기반 인증이란?

Token 기반 인증은 로그인 성공 후 서버가 Token을 발급하고,
클라이언트가 이후 요청마다 Token을 함께 전달하는 방식이다.

Token은 보통 `Authorization` Header에 담아 전송한다.

```http
Authorization: Bearer access-token-value
```

대표적인 Token 형식으로는 `JWT(JSON Web Token)`가 있다.
JWT는 Header, Payload, Signature로 구성되며,
서버는 Signature를 검증해 Token이 위조되지 않았는지 확인할 수 있다.

```text
header.payload.signature
```

즉, 핵심은 **인증에 필요한 정보를 Token에 담고, 요청마다 Token을 검증한다**는 점이다.

---

### Token 기반 인증의 장점

Token 기반 인증은 서버가 Session 상태를 저장하지 않아도 되기 때문에 `Stateless` 구조를 만들기 쉽다.
서버 여러 대로 확장해도 각 서버가 Token 검증만 할 수 있으면 인증 처리가 가능하다.

또한 Web, Mobile App, 외부 API Client처럼 다양한 클라이언트 환경에서 사용하기 좋다.
Cookie에 강하게 의존하지 않고 Header 기반으로 인증 정보를 전달할 수 있기 때문이다.

---

### Token 기반 인증의 단점

Token 기반 인증은 발급된 Token을 즉시 무효화하기 어렵다.
특히 JWT처럼 서버가 상태를 저장하지 않고 검증만 하는 구조에서는,
Token이 만료되기 전까지 계속 유효할 수 있다.

이를 보완하기 위해 다음과 같은 전략을 사용한다.

- Access Token 만료 시간을 짧게 설정
- Refresh Token을 사용해 Access Token 재발급
- Refresh Token Rotation 적용
- 탈취된 Token을 차단하기 위한 Blacklist 또는 Token Version 관리

또한 JWT Payload는 암호화된 것이 아니라 Base64Url로 인코딩된 값이다.
따라서 비밀번호, 주민등록번호 같은 민감한 정보를 Payload에 넣으면 안 된다.

---

### Token 기반 인증의 보안 고려사항

#### 1. Token 저장 위치

Token을 어디에 저장하느냐에 따라 공격 표면이 달라진다.

`localStorage`에 저장하면 JavaScript로 접근하기 쉬워 XSS에 취약하다.
반면 Cookie에 저장하면 `HttpOnly` 설정으로 XSS 위험을 줄일 수 있지만,
Cookie 자동 전송 특성 때문에 CSRF를 함께 고려해야 한다.

따라서 실무에서는 서비스 구조에 따라 다음을 함께 검토해야 한다.

- XSS 방어를 위한 입력값 검증과 출력 인코딩
- `HttpOnly`, `Secure`, `SameSite` Cookie 설정
- CSRF Token 적용 여부
- Token 만료 시간과 재발급 정책

#### 2. HTTPS 필수 사용

Token은 탈취되면 곧바로 인증 수단으로 사용될 수 있다.
따라서 HTTP가 아닌 HTTPS를 사용해 전송 구간을 암호화해야 한다.

#### 3. Access Token과 Refresh Token 분리

Access Token은 API 접근에 사용하고, Refresh Token은 Access Token을 재발급받는 용도로 사용한다.
Access Token은 짧게, Refresh Token은 상대적으로 길게 가져가는 방식이 일반적이다.

단, Refresh Token은 더 오래 살아있기 때문에 탈취되면 위험이 크다.
그래서 Refresh Token Rotation, 재사용 감지, 서버 저장소 관리 같은 추가 방어가 필요하다.

#### 4. JWT Signature 검증

JWT를 사용할 때는 Signature 검증을 반드시 수행해야 한다.
또한 `alg` 값을 신뢰하지 않고 서버에서 허용한 알고리즘만 사용해야 한다.

만료 시간인 `exp`, 발급자인 `iss`, 대상자인 `aud` 같은 Claim도 검증하는 것이 안전하다.

---

## 3. Session 기반 인증과 Token 기반 인증 비교

| 항목 | Session 기반 인증 | Token 기반 인증 |
| --- | --- | --- |
| 상태 관리 | 서버가 Session 상태 저장 | 서버가 상태를 저장하지 않는 구조 가능 |
| 인증 정보 위치 | 서버 Session 저장소 | 클라이언트가 Token 보관 |
| 확장성 | Session 공유 전략 필요 | 서버 간 Token 검증만 가능하면 확장 쉬움 |
| 로그아웃 처리 | 서버 Session 삭제로 즉시 처리 가능 | Token 만료 전까지 유효할 수 있어 추가 전략 필요 |
| 주요 위험 | Session Hijacking, CSRF, Session Fixation | Token 탈취, XSS, 긴 만료 시간, 잘못된 JWT 검증 |
| 적합한 환경 | 서버 렌더링 웹, 전통적인 웹 애플리케이션 | SPA, Mobile App, API Server, MSA 환경 |

정리하면 Session 기반 인증은 **서버가 로그인 상태를 통제하기 쉽다**는 장점이 있고,
Token 기반 인증은 **Stateless 구조와 확장성에 유리하다**는 장점이 있다.

하지만 어느 방식이 무조건 더 안전한 것은 아니다.
Session 방식은 Cookie 보안과 CSRF 방어가 중요하고,
Token 방식은 Token 저장 위치, 만료 정책, 탈취 대응이 중요하다.

---

## 4. OAuth 2.0이란?

`OAuth 2.0`은 인증(Authentication) 프로토콜이 아니라 권한 위임(Authorization) 프로토콜이다.

예를 들어 사용자가 어떤 서비스에 가입할 때 “Google 계정으로 로그인”을 선택하는 상황을 생각해보자.
이때 애플리케이션은 사용자의 Google 비밀번호를 직접 받지 않는다.
대신 Google의 Authorization Server를 통해 사용자의 동의를 받고,
허용된 범위의 리소스에 접근할 수 있는 Access Token을 발급받는다.

즉, OAuth 2.0의 핵심은 다음과 같다.

> 사용자의 비밀번호를 제3자 애플리케이션에 넘기지 않고, 제한된 권한만 위임한다.

실무에서 “OAuth 로그인”이라고 부르는 기능은 OAuth 2.0만으로 완성되는 것이 아니라,
사용자 식별 정보를 얻기 위해 `OpenID Connect(OIDC)`가 함께 사용되는 경우가 많다.
OAuth 2.0은 권한 위임, OIDC는 인증까지 다룬다고 구분하면 이해하기 쉽다.

---

## 5. OAuth 2.0의 주요 컴포넌트

### Resource Owner

`Resource Owner`는 보호된 리소스의 소유자다.
일반적인 로그인 시나리오에서는 사용자가 Resource Owner에 해당한다.

예를 들어 사용자의 Google 프로필, 이메일 정보에 대한 소유자는 사용자 자신이다.

---

### Client

`Client`는 Resource Owner의 권한을 위임받아 Resource Server에 접근하려는 애플리케이션이다.

예를 들어 “Google 계정으로 로그인” 기능을 제공하는 우리 서비스가 Client다.
Client는 사용자의 비밀번호를 직접 받지 않고, Authorization Server로부터 Access Token을 발급받아 사용한다.

---

### Authorization Server

`Authorization Server`는 Resource Owner를 인증하고,
사용자의 동의를 받은 뒤 Client에게 Authorization Code 또는 Access Token을 발급하는 서버다.

Google, Kakao, Naver 같은 OAuth Provider의 인증 서버가 여기에 해당한다.

---

### Resource Server

`Resource Server`는 보호된 리소스를 가지고 있는 서버다.
Client는 Access Token을 사용해 Resource Server의 API에 접근한다.

예를 들어 Google 사용자 정보 API 서버가 Resource Server에 해당한다.

---

### Access Token

`Access Token`은 Resource Server에 접근하기 위한 권한 증명이다.
Client는 API 요청 시 Access Token을 전달하고,
Resource Server는 Token을 검증한 뒤 요청을 허용한다.

```http
Authorization: Bearer access-token-value
```

---

### Refresh Token

`Refresh Token`은 Access Token이 만료되었을 때 새로운 Access Token을 발급받기 위해 사용한다.
Access Token보다 오래 유지되는 경우가 많기 때문에 더 안전하게 보관해야 한다.

---

### Scope

`Scope`는 Client가 요청하는 권한의 범위다.

예를 들어 이메일 조회만 필요한 서비스라면 `profile`, `email` 정도의 Scope만 요청해야 한다.
불필요하게 넓은 Scope를 요청하면 보안 위험이 커지고 사용자 신뢰도 떨어질 수 있다.

---

## 6. Authorization Code Grant 흐름

`Authorization Code Grant`는 OAuth 2.0에서 가장 대표적인 흐름이다.
Client가 사용자를 Authorization Server로 리다이렉트하고,
사용자가 로그인과 동의를 완료하면 Authorization Code를 발급받는다.
이후 Client는 Authorization Code를 Access Token으로 교환한다.

중요한 점은 Access Token이 브라우저 URL에 직접 노출되지 않는다는 것이다.
먼저 짧은 수명의 Authorization Code를 받고,
서버 간 통신으로 Access Token을 교환하기 때문에 보안성이 높다.

---

### Authorization Code Grant 단계

#### 1. Client가 Authorization Server로 리다이렉트

사용자가 “Google로 로그인” 버튼을 누르면 Client는 사용자를 Authorization Server의 인증 페이지로 이동시킨다.

```http
GET /oauth2/authorize?
  response_type=code&
  client_id=client-id&
  redirect_uri=https://client.com/callback&
  scope=profile email&
  state=random-state-value
```

주요 파라미터는 다음과 같다.

- `response_type=code`: Authorization Code를 요청한다는 의미
- `client_id`: Client를 식별하는 값
- `redirect_uri`: 인증 완료 후 돌아올 Callback URL
- `scope`: 요청 권한 범위
- `state`: CSRF 방지를 위한 임의 문자열

---

#### 2. Resource Owner가 로그인하고 권한 위임에 동의

Authorization Server는 사용자에게 로그인 화면과 동의 화면을 보여준다.
사용자가 로그인하고 권한 위임에 동의하면 다음 단계로 넘어간다.

---

#### 3. Authorization Server가 Authorization Code를 발급

Authorization Server는 등록된 `redirect_uri`로 사용자를 다시 이동시키면서 Authorization Code를 전달한다.

```http
GET https://client.com/callback?code=authorization-code&state=random-state-value
```

Client는 이때 전달받은 `state`가 처음 요청한 값과 같은지 검증해야 한다.
검증에 실패하면 CSRF 공격 가능성이 있으므로 요청을 거부해야 한다.

---

#### 4. Client가 Authorization Code를 Access Token으로 교환

Client 서버는 Authorization Server의 Token Endpoint로 Authorization Code를 전달해 Access Token을 발급받는다.

```http
POST /oauth2/token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code&
code=authorization-code&
redirect_uri=https://client.com/callback&
client_id=client-id&
client_secret=client-secret
```

이 과정은 브라우저가 아니라 Client 서버와 Authorization Server 사이에서 이루어진다.
따라서 Access Token이 사용자 브라우저 URL에 노출되지 않는다.

---

#### 5. Client가 Access Token으로 Resource Server에 접근

Client는 발급받은 Access Token을 사용해 Resource Server의 API를 호출한다.

```http
GET /userinfo
Authorization: Bearer access-token-value
```

Resource Server는 Access Token을 검증하고,
Token에 허용된 Scope 범위 안에서 리소스를 응답한다.

---

## 7. Authorization Code Grant의 보안 고려사항

### 1. Redirect URI 검증

Authorization Server는 요청에 포함된 `redirect_uri`가 사전에 등록된 값과 정확히 일치하는지 검증해야 한다.
Redirect URI 검증이 느슨하면 Authorization Code가 공격자의 서버로 전달될 수 있다.

---

### 2. State 파라미터 검증

`state`는 OAuth 2.0 흐름에서 CSRF를 방어하기 위한 값이다.
Client는 Authorization 요청을 보낼 때 예측하기 어려운 state 값을 생성해 저장하고,
Callback으로 돌아온 state 값과 반드시 비교해야 한다.

---

### 3. Authorization Code는 짧게 유지

Authorization Code는 Access Token으로 교환되기 전의 임시 값이다.
따라서 매우 짧은 만료 시간을 가져야 하고, 한 번 사용된 Code는 재사용할 수 없어야 한다.

---

### 4. Client Secret 보호

서버 기반 Client는 `client_secret`을 안전하게 보관해야 한다.
GitHub 같은 공개 저장소에 client_secret을 올리면 안 되며,
환경 변수나 Secret Manager를 사용해 관리해야 한다.

반면 SPA나 Mobile App처럼 Secret을 안전하게 숨기기 어려운 Public Client에서는 `PKCE(Proof Key for Code Exchange)`를 함께 사용해야 한다.

---

### 5. PKCE 적용

`PKCE`는 Authorization Code가 중간에 탈취되더라도 공격자가 Access Token으로 교환하지 못하도록 방어하는 방식이다.
처음 Authorization 요청 시 `code_challenge`를 보내고,
Token 요청 시 원본 값인 `code_verifier`를 제출해 검증한다.

최근에는 Public Client뿐 아니라 Confidential Client에서도 PKCE를 사용하는 것이 권장된다.

---

### 6. Scope 최소화

Client는 필요한 권한만 요청해야 한다.
사용자 이메일만 필요하다면 과도한 Calendar, Drive, 결제 정보 Scope를 요청하면 안 된다.

Scope를 최소화하면 Token 탈취나 권한 오남용이 발생했을 때 피해 범위를 줄일 수 있다.

---

## 마무리

Session 기반 인증과 Token 기반 인증은 모두 사용자의 인증 상태를 처리하기 위한 방식이지만,
상태를 어디서 관리하느냐가 다르다.
Session 기반 인증은 서버가 상태를 관리하기 때문에 통제가 쉽고,
Token 기반 인증은 Stateless 구조를 만들기 쉬워 확장성에 유리하다.

OAuth 2.0은 사용자의 비밀번호를 직접 공유하지 않고 제한된 권한만 위임하기 위한 프로토콜이다.
그중 Authorization Code Grant는 Authorization Code를 먼저 발급받고,
서버 간 통신으로 Access Token을 교환하기 때문에 웹 애플리케이션에서 널리 사용된다.

실무에서는 단순히 인증이 동작하는지만 볼 것이 아니라,
Cookie 설정, CSRF 방어, Token 만료 정책, Redirect URI 검증, State 검증, PKCE 적용까지 함께 고려해야 한다.
결국 인증과 인가는 기능 구현보다 **탈취되었을 때 피해를 어떻게 줄이고, 잘못된 요청을 어떻게 차단할 것인가**가 핵심이다.

---
