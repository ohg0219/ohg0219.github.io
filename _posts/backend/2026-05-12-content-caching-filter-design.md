---
title: "ContentCachingFilter 설계"
description: "request/response 본문을 한 곳에서 일관되게 로깅하기"
date: 2026-05-12 00:00:00 +0900
categories: [backend]
tags: [spring-boot, servlet-filter, request-logging, debugging, http]
---

> "방금 그 요청 본문이 뭐였어요? 우리가 받은 게 맞아요?"

PG 입금통보, 카드사 콜백 같은 **서버 대 서버 API** 를 다루다 보면 거의 한 달에 한 번은 듣는 질문입니다. 대부분 진실은 로그 어딘가에 있습니다. 다만 컨트롤러까지 도달하기 전에 필터·인터셉터 어디선가 body 스트림이 한 번 소진되면, 이후엔 같은 본문을 다시 읽을 수 없습니다. `HttpServletRequest` 의 InputStream 은 **단방향 1회성** 이라 두 번 읽으면 `IllegalStateException` 이나 빈 문자열만 돌아옵니다.

이 글은 그래서 글로벌 필터로 박아둔 `ContentCachingFilter` 의 설계 의도, 빠지기 쉬운 함정, 그리고 운영 디버깅에서 실제로 어떻게 써먹고 있는지를 정리한 기록입니다. 목적은 단순합니다 — **모든 API 요청의 request/response 를 한 곳에서 일관되게 로깅**.

## 왜 래퍼가 필요한가

`@RequestBody` 로 매핑하는 순간 Spring 은 body 스트림을 읽어버립니다. 컨트롤러 안에서 같은 본문을 다시 보려고 `request.getReader()` 를 호출하면 빈 문자열이 나옵니다.

로그를 남기려면 어디선가 한 번은 body 를 읽어야 하는데, 그 한 번이 컨트롤러에서 일어나면 로깅 단에서는 이미 빈 스트림이고, 로깅 단에서 일어나면 컨트롤러가 빈 본문을 받습니다. 어느 한 쪽이 손해보는 구조입니다.

Spring 이 이미 정답을 갖고 있습니다. `ContentCachingRequestWrapper` 와 `ContentCachingResponseWrapper`. 이 래퍼는 한 번 흘러간 바이트를 내부 버퍼에 보관해 두기 때문에 `getContentAsByteArray()` 로 몇 번이고 다시 꺼낼 수 있습니다.

문제는 "어디서, 언제 감싸느냐"입니다. 컨트롤러에서 감싸봐야 이미 늦었습니다. **필터 체인의 가장 앞쪽** 에서 한 번 감싸 두고, 그 뒤의 모든 필터·컨트롤러가 같은 래퍼를 보게 만들어야 의미가 있습니다.

## 필터 본문 — 핵심만

골격은 단순합니다.

```java
public class ContentCachingFilter extends OncePerRequestFilter {

    private final AntPathMatcher pathMatcher = new AntPathMatcher();

    @Override
    protected boolean shouldNotFilter(HttpServletRequest request) {
        String url = request.getRequestURI();
        return Stream.of(EXCLUDE_PATTERNS)
                     .anyMatch(p -> pathMatcher.match(p, url));
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain) throws ServletException, IOException {
        String traceId = ThreadContext.get("traceId");
        if (traceId == null) {
            traceId = generateTraceId();
            ThreadContext.put("traceId", traceId);
            ThreadContext.put("requestUri", request.getMethod() + " " + request.getRequestURI());
        }

        ContentCachingRequestWrapper  requestWrapper  = new ContentCachingRequestWrapper(request);
        ContentCachingResponseWrapper responseWrapper = new ContentCachingResponseWrapper(response);
        try {
            filterChain.doFilter(requestWrapper, responseWrapper);
        } finally {
            printLog(requestWrapper, responseWrapper);
            responseWrapper.copyBodyToResponse(); // 필수!
        }
    }
}
```

설계 의도를 정리하면 다음 다섯 가지입니다.

### OncePerRequestFilter 로 중복 호출 차단

서블릿 컨테이너는 forward·include·async dispatch 마다 필터 체인을 다시 태웁니다. 그때마다 본문을 또 캐싱하고 또 로깅하면 트래픽이 두 배가 됩니다. `OncePerRequestFilter` 는 요청 1회당 한 번만 동작하도록 보장합니다. body 캐싱처럼 **부작용이 큰 필터** 는 거의 무조건 이 상위 클래스를 상속합니다.

### 필터 체인 최상단 부근에 위치

Spring Security 의 필터 체인에 등록할 때는 인증 필터보다 **앞쪽** 에 둡니다. 예시는 `SecurityContextPersistenceFilter` 직후.

```java
http.addFilterAfter(new ContentCachingFilter(), SecurityContextPersistenceFilter.class);
```

이 순서가 핵심입니다. 이후 어떤 필터·컨트롤러로 흘러가도 **모두 같은 래퍼** 를 받게 됩니다. 그래서 컨트롤러가 `@RequestBody` 로 파싱하고 나서도, 필터가 다시 `getContentAsByteArray()` 로 동일한 원문 바이트를 꺼낼 수 있습니다.

### shouldNotFilter 로 정적 리소스 제외

`/css/**`, `/js/**`, 이미지 경로 같은 정적 자원까지 캐싱하면 메모리만 낭비됩니다. `AntPathMatcher` 기반으로 화이트리스트 패턴을 두고 짧게 끊습니다.

```java
private static final String[] EXCLUDE_PATTERNS = {
    "/css/**", "/js/**", "/images/**", "/favicon.ico"
};
```

### copyBodyToResponse() 가 진짜 필수

`ContentCachingResponseWrapper` 는 내부 버퍼에만 응답을 쌓고, 원래 응답 스트림으로는 **아무것도 흘리지 않습니다.** 마지막에 `copyBodyToResponse()` 를 호출해야 비로소 실제 클라이언트로 본문이 전달됩니다.

이걸 빼먹으면 클라이언트는 `Content-Length: 123` 헤더만 받고 본문은 비어 있는 응답을 받습니다. 브라우저에서는 "흰 화면", 외부 API 호출 측에서는 "빈 JSON" 으로 보고됩니다. 한 번 데어 보면 안 잊습니다.

`finally` 블록에 둔 이유는 컨트롤러에서 예외가 터져도 응답을 흘리긴 해야 하기 때문입니다. 예외 → ExceptionHandler → 응답 본문 작성까지 모두 래퍼에 쌓이기 때문에, 마지막에 copy 만 해주면 정상·예외 경로 모두 깔끔하게 빠집니다.

### ThreadContext 에 traceId 주입

짧은 traceId (`K7M2X9R` 같은 7자리 코드) 를 MDC (Log4j2 의 `ThreadContext`) 에 박아둡니다. 이후 어떤 서비스·DAO 에서 찍는 로그든 동일한 traceId 가 prefix 로 붙기 때문에, 운영에서 "이 요청 하나만 골라 보기" 가 grep 한 방으로 끝납니다.

```java
private String generateTraceId() {
    String alphabet = "23456789ABCDEFGHJKLMNPQRSTUVWXYZ"; // 혼동 문자 제외
    StringBuilder sb = new StringBuilder();
    Random random = new Random();
    for (int i = 0; i < 7; i++) {
        sb.append(alphabet.charAt(random.nextInt(alphabet.length())));
    }
    return sb.toString(); // e.g. K7M2X9R
}
```

혼동 문자를 뺀 32진 알파벳에서 7자리. 짧고, 시각적으로 안 헷갈리고, 짧은 시간 안에는 충돌 없음 — UUID 풀까지 안 가도 되는 영역이라 일부러 가독성 우선으로 잡았습니다.

## printLog — multipart 와 reader 폴백

`printLog` 안쪽에서 굳이 신경 쓴 두 가지.

**multipart 는 본문을 안 찍습니다.**

```java
String reqContentType = request.getContentType();
if (StringUtils.contains(reqContentType, "multipart")) {
    log.info("do not print multipart request body");
} else {
    byte[] body = request.getContentAsByteArray();
    if (body.length > 0) {
        log.info("Request Body: {}", new String(body, StandardCharsets.UTF_8));
    }
}
```

파일 업로드 본문을 통째로 로그에 박으면 디스크가 며칠을 못 버팁니다. content-type 만 보고 빠르게 스킵합니다.

**`getContentAsByteArray()` 가 비어 있으면 `getReader()` 로 한 번 더 읽습니다.**

래퍼는 "읽힌 만큼만" 캐싱합니다. 컨트롤러에서 `@RequestBody` 로 읽힌 본문은 캐시에 잡히지만, 아무도 안 읽고 통과한 본문은 byte array 가 0바이트인 경우가 있습니다. 이때를 위해 reader 폴백을 한 번 더 태웁니다.

```java
BufferedReader reader = request.getReader();
StringBuilder requestBody = new StringBuilder();
String line;
while ((line = reader.readLine()) != null) {
    requestBody.append(line).append('\n');
}
if (requestBody.length() > 0) {
    log.info("Request Body (reader): {}", requestBody);
}
```

약간 못생긴 방어 코드지만, 외부 시스템 콜백처럼 "들어왔는데 컨트롤러 매핑이 잘못돼서 안 읽힌" 상황까지 잡아줍니다.

응답 쪽도 같은 패턴입니다. JSON 만 본문을 찍고, 그 외 content-type (HTML, 바이너리) 은 헤더만 찍습니다.

```java
String resContentType = response.getContentType();
if (StringUtils.startsWith(resContentType, MediaType.APPLICATION_JSON_VALUE)) {
    byte[] body = response.getContentAsByteArray();
    log.info("Response Body: {}", new String(body, StandardCharsets.UTF_8));
}
```

## 운영에서 실제로 어떻게 써먹는가

디버깅 패턴은 거의 정해져 있습니다.

**시나리오 A. "외부에서 콜백 보냈는데 처리가 안 됐다" 는 문의**

비즈니스 테이블에 흔적이 없으면 두 가지 가능성입니다.

1. 콜백 자체가 안 들어왔다 (상대측 문제)
2. 들어왔는데 컨트롤러 진입 전·후 어딘가에서 떨어졌다 (우리 문제)

ContentCachingFilter 가 무조건 본문을 찍어 두기 때문에, **요청이 우리 서버에 도달은 했는지** 는 traceId 한 줄 grep 으로 즉시 판가름 납니다. 도달했다면 원문이 그대로 로그에 있고, 못 했다면 로그에 아무것도 없습니다 — 책임 소재가 1분 안에 갈립니다.

**시나리오 B. "응답으로 뭘 줬는지 알고 싶다"**

상대측에서 "응답 형식이 우리 스펙과 다르다" 고 클레임이 들어오면, 우리가 실제로 흘려보낸 응답 본문을 보여줄 수 있어야 합니다. response wrapper 가 응답 본문도 같이 캐싱해 두기 때문에 같은 traceId 묶음에 request 와 response 가 짝으로 따라옵니다. 스펙 분쟁이 굳이 길게 갈 일이 없습니다.

**시나리오 C. "예외가 났는데 어떤 입력 때문이었지?"**

ExceptionHandler 가 깔끔하게 500 으로 떨어뜨리고 끝낸 케이스. 원인 추적을 위해선 그 요청의 본문이 필요합니다. 필터가 finally 블록에서 출력하기 때문에, 컨트롤러에서 예외가 터지든 정상 응답이 가든 본문은 무조건 남습니다. 비즈니스 코드에서 별도로 본문을 찍으려고 애쓰지 않아도 된다는 게 가장 큰 장점입니다.

## 어디에 쓰면 좋은가

- **서버-서버 API 게이트웨이 성격** 의 엔드포인트가 있는 서비스: PG, 카드사, 보험사 연동 등
- **외부 콜백을 받는 서비스**: 본문을 한 번 읽고 끝나면 디버깅이 막힌다
- **감사·추적 요구가 있는 서비스**: 모든 트래픽의 원문이 로그에 남아 있어야 할 때

반대로 트래픽이 매우 크고 본문 사이즈가 큰 서비스 (파일 업로드 위주, 스트리밍 등) 에서는 비용이 큽니다. `shouldNotFilter` 화이트리스트를 세심하게 잡거나, 필요 경로에만 좁게 거는 식으로 운용해야 합니다.

## 가져갈 한 줄

> **`copyBodyToResponse()` 를 빼먹지 말 것.** 운영에서 흰 화면을 만나는 가장 빠른 방법이다.
