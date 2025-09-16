## AddRequestHeader GatewayFilter Factory
**자주 쓰일 것 같은 것들 위주**

`AddRequestHeader` GatewayFilter Factory 는 `name`과 `value` 파라미터를 받는다.
아래 예시는 `AddRequestHeader GatewayFilter`를 설정하는 방법을 보여준다.

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: add_request_header_route
          uri: https://example.org
          filters:
            - AddRequestHeader=X-Request-red, blue

```
- 위 설정은 매칭되는 모든 요청에 대해 `X-Request-red: blue` 헤더를 추가하여 다운스트림 요청에 전달
---
`AddRequestHeader` 는 경로나 호스트를 매칭할 때 사용되는 **URI 변수** 도 인식한다.
즉, 헤더 값에 URI 변수를 사용할 수 있으며, 런타임 시점에 실제 값으로 치환
```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: add_request_header_route
          uri: https://example.org
          predicates:
            - Path=/red/{segment}
          filters:
            - AddRequestHeader=X-Request-Red, Blue-{segment}
```
- 이 설정은 `/red/{segment}` 경로와 매칭되는 요청이 들어오면,
  `X-Request-Red: Blue-{segment}` 형태의 헤더가 추가된다.
  예를 들어 `/red/123` 요청이 들어오면 헤더 값은 `X-Request-Red: Blue-123` 이 된다.
---

## AddResponseHeader GatewayFilter Factory
`AddResponseHeader` GatewayFilter Factory 는 3가지 파라미터를 받는다.
- `name`
- `value`
- `override` (기본값: `true`)
```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: add_response_header_route
          uri: https://example.org
          filters:
            - AddResponseHeader=X-Response-Red, Blue
            - AddResponseHeader=X-Response-Black, White, false
```
- 매칭되는 모든 요청의 응답 헤더에 `X-Response-Red: Blue` 를 추가
- 응답에 이미 `X-Response-Black` 헤더가 존재한다면 `X-Response-Black: White` 헤더는 추가 되지 않음
  (`override=false` 설정 때문)
---

`AddResponseHaeder` 도 마찬가지로 **경로나 호스트 매칭 시 사용되는 URI 변수를 인식한다.**
```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: add_response_header_route
          uri: https://example.org
          predicates:
            - Host: {segment}.myhost.org
          filters:
            - AddResponseHeader=foo, bar-{segment}

```
---

## RewritePath (경로 재작성)
**들어오는 요청 URL 의 경로(Path)를 바꿔서 다운스트림 서비스로 전달**
`RewritePath` 는 두가지 파라미터를 받음
- `path regexp`: 변경할 경로를 지정하는 정규식
- `replacement`: 변환 후 경로

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: rewritepath_route
          uri: https://example.org
          predicates:
            - Path=/red/**
          filters:
            - RewritePath=/red/?(?<segment>.*), /$\{segment}
```
- 요청 경로가 `/red/blue` 이면, 다운스트림 요청을 보내기 전에 경로가 `/blue` 로 변경
- 참고로 `YAML` 문법 때문에 `$` 기호는 `$\`로 작성해야 함

---

## RequestRateLimiter (요청 속도 제한)
**일정 시간 동안 허용되는 요청 횟수를 제한하여, 과도한 요청으로 시스템이 다운되는 걸 방지**

### 언제 쓰지?
- 외부 사용자/클라이언트가 API 를 남용할 가능성이 있을 때
- 서비스 안정성을 위해 트래픽을 제어하고 싶을 때
- 특히 공개 API, 모바일 앱 서버, 외부 서비스 연동에서 유용
  (허용되지 않는 경우, 기본적으로 `HTTP 429 - Too Many Requests` 상태 코드가 반환)

---
### 주요 파라미터

- `keyResolver(선택) :` 요청 제한 키를 결정하는 전략
- `RateLimiter 관련 파라미터` : 선택한 RateLimiter 구현체에 따라 다름
---

### KeyResolver
`KeyResolver` 는 요청 제한을 위한 키를 결정하는 전략 인터페이스
```java
public interface KeyResolver {
    Mono<String> resolve(ServerWebExchange exchange);
}
```
- 기본 구현체는 **PrincipalNameKeyResolver**로 `ServerWebExchange` 에서 `Principal.getName()` 을 가져옴
- 기본적으로 `KeyResolver` 가 키를 찾지 못하면 요청은 거부
- 동작을 변경하려면 아래 속석을 설정할 수 있다.
  - `spring.cloud.gateway.filter.request-rate-limiter.deny-empty-key (true/false)`
  - `spring.cloud.gateway.filter.request-rate-limiter.empty-key-status-code`

**예시**
```java
@Bean
KeyResolver userKeyResolver() {
    return exchange -> Mono.just(exchange.getRequest().getQueryParams().getFirst("user"));
}
```
---

### 설정 방법
- 단축(notation) 방식은 지원 X
- YAML 에서 `name` 과 `args` 방식으로 설정해야 함.
```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: limit
          uri: https://example.org
          filters:
            - name: RequestRateLimiter
              args:
                key-resolver: "#{@userKeyResolver}"

```
---
## Redis RateLimiter
- Stripe 에서 개발한 **Token Bucket** 알고리즘 기반
- 사용 시 `spring-boot-starter-data-redis-reactive` 필요

**주요 속성**
- `replenishRate`: **초당 허용 요청 수**
- `burstCapacity`: **한 번에 허용되는 최대 요청 수(버스트)**
- `requestedTokens`: **요청 하나가 소모하는 토큰 수(기본1)**

**예시: 초당 10회, 버스트 20회 허용**
```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: requestratelimiter_route
          uri: https://example.org
          filters:
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 10
                redis-rate-limiter.burstCapacity: 20
                redis-rate-limiter.requestedTokens: 1

```
- KeyResolver 는 예제처럼 간단히 사용자 쿼리 파라미터를 가져올 수 있음
- 프로덕션 환경에서는 단순 KeyResolver 사용을 권장하지 않음

