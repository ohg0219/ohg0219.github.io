---
title: "Spring Boot Actuator 헬스체크 경로가 root 로 떨어진 이유"
date: 2026-05-12 00:00:00 +0900
categories: [backend]
tags: [spring-boot, actuator, health-check, configuration, devops]
---

**404 도 아니고, 500 도 아니고 — 루트가 떴습니다.** 헬스체크 URL 을 호출했는데 Spring Boot 의 기본 환영 페이지가 떨어진 그날의 디버깅 기록입니다. 범인은 `exclude: "*"` 한 줄이었지만, 거기까지 가는 길에 두 번의 오진이 있었습니다.

## 환영 페이지가 떴다

운영 환경에서 헬스체크 경로를 **추측하기 어렵게** 만들고 싶었습니다.
`/actuator/health` 라는 뻔한 경로 대신, 랜덤 문자열을 붙인 URL 로 위장한 다음, 다른 endpoint 는 전부 닫는 것이 목표였죠.

설정을 적용하고 브라우저에서 새 경로를 열었습니다.

```
http://localhost:8080/health-x7k2m9p4
```

그런데 — health JSON 이 아니라 **Spring Boot 기본 welcome 페이지** 가 떴습니다.

```
404 도 아니고
500 도 아니고
그냥 root.
```

이건 매핑이 등록조차 되지 않았다는 신호입니다. 등록된 경로면 보안 차단이라도 4xx 가 떨어지지, root 로 가지는 않거든요.

문제 설정은 이거였습니다.

```yaml
management:
  endpoints:
    web:
      base-path: /
      path-mapping:
        health: health-x7k2m9p4
      exposure:
        include: health      # health 만 열고
        exclude: "*"         # 나머지는 닫는다
```

`include: health`, `exclude: "*"`. 문장으로 읽으면 의도가 명확합니다. *"health 만 열고 나머지는 전부 닫겠다."*

## 첫 번째 의심: base-path

`base-path: /` 와 동일 포트 조합이 의심스러웠습니다. "공식적으로 막혀 있다" 는 어렴풋한 기억으로 base-path 를 떼어보기로 했죠.

결과: **틀린 진단.** 동일 포트에 `/` 로 두면 경고는 날 수 있어도 동작 자체는 막히지 않습니다. "원래 잘 됐던 설정" 이라는 사용자 코멘트가 가설을 무너뜨렸습니다.

여기서 한 번 멈춰 생각해야 했습니다. "공식 문서가 그렇게 말한다" 와 "실제 런타임이 그렇게 동작한다" 는 다른 명제니까요.

## 진짜 범인은 한 줄이었다

문제는 `exclude: "*"` 였습니다.

Spring Boot actuator 의 노출 규칙은 직관과 반대로 작동합니다.

```
① include 로 후보 선정
② exclude 로 후보 제거  ← 우선 적용
③ 최종 노출 집합 산출
```

즉 `exclude` 가 `include` 를 이깁니다. 와일드카드 `*` 가 health 까지 같이 잡아먹어서, 결과적으로 **매핑이 등록되지 않습니다.**

그래서 `/health-x7k2m9p4` 를 호출하면 — Spring MVC 입장에서 그런 경로는 존재하지 않는 거고, fallback 으로 ViewResolver 가 동작해서 welcome 페이지가 떴던 겁니다. 404 가 아니라 root 가 뜬 이유까지 한 줄로 설명됩니다.

## Before / After

한눈에 비교하면 이렇습니다.

| | Before | After |
|---|---|---|
| **include** | `health` | `health` |
| **exclude** | `"*"` | *(삭제)* |
| **매핑 등록** | ❌ 안 됨 | ✅ 등록됨 |
| **`/health-x7k2m9p4` 응답** | root welcome | `{"status":"UP"}` |

수정된 설정:

```yaml
management:
  endpoints:
    web:
      base-path: /
      path-mapping:
        health: health-x7k2m9p4
      exposure:
        include: health      # 이것만 있으면 충분
  endpoint:
    health:
      show-details: never    # DB 종류 등 상세정보 차단
      probes:
        enabled: false       # K8s 미사용
  health:
    db:
      enabled: true
    redis:
      enabled: false
    diskspace:
      enabled: false
    ping:
      enabled: false
```

`include` 만 적으면 그 외는 자동으로 닫힙니다. `exclude` 는 "와일드카드로 다 열되 일부만 빼는" 좁은 용도에만 의미가 있고, "하나만 켠다" 시나리오에서는 오히려 함정입니다.

## 캐시가 가린 진실

`exclude` 를 떼고 재기동했는데도 404 가 잠깐 났습니다. 또 다른 원인인가 싶었지만 결국 **브라우저 캐시** 였습니다.

이때 가장 빠른 검증 도구는 추측이 아니라 부팅 로그입니다.

```
Exposing 1 endpoint(s) beneath base path ''
Mapped "{[/health-x7k2m9p4],...}" onto ... HealthEndpointWebExtension
```

- 이 로그가 **있는데** 404 → 시큐리티·인터셉터·캐시 의심
- 이 로그가 **없으면** → exposure·path-mapping 자체 문제

같은 404 라도 처방이 정반대라서, 이 분기점부터 잡고 들어가야 합니다.

## 가져갈 두 가지

**1. include / exclude 가 있는 설정은 우선순위부터 확인하라.**
Spring Security, logback filter, 각종 라우팅 — 대부분 `exclude` / `deny` 가 우선합니다. "둘 다 적어 두면 더 안전하겠지" 는 자주 정반대 결과를 냅니다. 화이트리스트 하나면 의도도 명확하고 부작용도 없습니다.

**2. "등록 안 됨" 과 "차단됨" 은 다른 문제다.**
같은 4xx·5xx 라도 원인 트리가 갈립니다. 매핑 로그 한 줄이 그 분기점을 가르는 가장 싼 진단 도구입니다.

그리고 사족 하나 — "공식적으로 안 된다" 라는 기억은 종종 부정확합니다. 단정 짓기 전에 매핑 로그로 확인하는 습관이 가장 안전합니다.
