---
title: "GoatCounter 로 블로그 조회수 추적하기"
description: "가입부터 사이드바 위젯까지"
date: 2026-05-13 13:31:21 +0900
categories: [tooling]
tags: [goatcounter, analytics, jekyll, sidebar-widget, pageviews]
image:
  path: /assets/img/posts/goatcounter-tracking/cover.jpg
  alt: GoatCounter 기반 블로그 조회수 추적과 사이드바 방문자 카운터를 표현한 기술 커버 이미지
---

**글마다 "몇 명이 봤지" 가 궁금하고, 사이드바엔 "오늘 누가 방문했지" 도 띄우고 싶었습니다.** 둘이 사실 같은 데이터인데, 분석 도구마다 라벨을 넣는 위치가 다르고 셋업 비용도 제각각입니다.

이 블로그는 그 두 가지를 **하나의 도구**로 해결합니다. 글 메타에 "조회수 N회" 가 자동으로 박히고, 사이드바 하단엔 "오늘 / 전체" 두 줄이 매번 갱신됩니다. 그 한 도구가 GoatCounter 입니다.

## 왜 GoatCounter 인가

비교 후보는 Google Analytics, Plausible, Umami 정도였는데, 결정타가 된 셋만 짚어 봅니다.

**Chirpy 7.x 의 pageviews provider 가 GoatCounter 만 지원합니다.** `_config.yml` 의 한 블록으로 글 메타에 "조회수 N회" 라벨이 자동으로 들어갑니다. 다른 도구를 쓰려면 post layout 을 override 해서 라벨 영역을 직접 만들어야 합니다 — 그 비용을 감수할 만큼 GA · Plausible · Umami 가 매력적이지 않았습니다.

**쿠키를 안 씁니다.** GoatCounter 는 IP + User-Agent 를 daily salt 와 함께 hash 해 unique 식별하고, 8시간 후 hash 가 만료됩니다. PII 저장이 없어서 개인 기술 블로그에 GDPR 쿠키 동의 배너를 안 띄워도 됩니다. GA 와의 가장 큰 차이입니다.

**공개 stats endpoint 가 있습니다.** `/counter/<path>.json`, `/counter/TOTAL.json` 같은 public JSON 으로 **인증 없이** 카운트를 받을 수 있습니다. Plausible · Umami 도 API 는 있지만 토큰 인증이 필요한 반면, GoatCounter 는 `fetch` 한 줄로 사이드바 위젯을 만들 수 있었습니다. 결과적으로 **글 메타 조회수 + 사이드바 위젯 두 가지를 한 도구로** 묶을 수 있었습니다.

부수로 트래킹 스크립트가 ~3KB 라 페이지에 부담이 거의 없고, 개인 사이트는 `<id>.goatcounter.com` 서브도메인이 무료로 제공된다는 점이 따라옵니다.

## 셋업 — 다섯 단계

가입부터 위젯 데이터까지, 다섯 줄로 정리합니다.

**1. 사이트 등록.** `goatcounter.com` 에서 무료 가입하고 `<id>.goatcounter.com` 서브도메인을 결정합니다. 이후 모든 트래킹 데이터가 이 서브도메인에 쌓입니다.

**2. read-only API 토큰 발급.** 대시보드 → User → API tokens 에서 토큰을 하나 만듭니다. 권한은 **`Count` (Read statistics) 한 줄만 체크**. 이 토큰은 곧 사이드바 위젯의 fetch 헤더에 들어가 빌드 결과물 JS 에 그대로 노출됩니다 — 그래서 권한을 최소화하는 게 핵심입니다.

> **빌드 결과물에 토큰 노출, 괜찮은가?**
> 권한이 통계 조회로 제한돼 있으면 노출되어도 다른 행위(쓰기·삭제·설정 변경)는 할 수 없습니다. 코드에 토큰을 넣을 때 가장 먼저 결정해야 할 것은 **"이 토큰이 read-only 인가, 그게 보장되는가"** 입니다. 이 보장이 있으면 평문 노출도 합리적입니다.

**3. Public counter 활성화.** GoatCounter Settings 에서 사이트 통계 공개 옵션을 켜면 `/counter/TOTAL.json` 같은 public endpoint 가 인증 없이 열립니다. 사이드바 위젯의 "전체" 숫자를 받기 위한 한 단계.

**4. `_config.yml` 두 블록.** 여기까지가 **글 메타에 "조회수 N회" 라벨이 자동으로 박히는 데 필요한 전부**입니다.

```yaml
analytics:
  goatcounter:
    id: <site_code>      # 1번에서 정한 서브도메인 앞부분

pageviews:
  provider: goatcounter
```

`analytics:` 가 `<head>` 에 트래킹 스크립트를 자동 삽입하고, `pageviews:` 가 글 메타에 라벨 영역을 만듭니다. **둘 다 켜야 동작**합니다. analytics 만 켜면 데이터는 쌓이지만 라벨이 안 보이고, pageviews 만 켜면 라벨은 켜지지만 보낼 데이터가 없어 영영 0입니다. 라벨이 0 인데 GoatCounter 대시보드엔 숫자가 잡힌다면 먼저 의심할 지점입니다.

**5. `_data/goatcounter.yml`.** 사이드바 위젯이 참조할 데이터 파일. site_code 와 read_token 두 값만 박습니다.

```yaml
site_code: <site_code>
read_token: <READ_TOKEN>   # ⚠️ 글에서는 마스킹. 실제 파일엔 평문 토큰값. read-only 권한이라 빌드 산출물 노출 OK.
```

이 파일은 git 에 그대로 커밋합니다. 2번에서 토큰 권한을 read-only 로 제한한 것이 그 보장이라, 환경변수로 빼서 빌드 시 주입하는 식의 복잡한 흐름이 필요 없습니다. 파일 헤더에 이 결정을 코멘트로 남겨 두면 미래의 자기와 동료가 안심합니다.

## 적용 — 두 카운터

### 글별 조회수는 셋업으로 이미 끝

위 4번 블록을 넣는 순간 글 메타에 "조회수 N회" 가 자동으로 들어갑니다. Chirpy 가 GoatCounter API 를 호출해 그 글 경로의 카운트를 가져와 라벨에 채우는 식. 추가로 만질 부분이 없습니다.

### 사이드바 위젯 — 본론

사이드바 하단에 두 줄을 붙입니다.

```html
<div id="visitor-counter"
     data-gc-site="{{ site.data.goatcounter.site_code }}"
     data-gc-token="{{ site.data.goatcounter.read_token }}">
  <div><span>오늘</span><span data-vc-today>—</span></div>
  <div><span>전체</span><span data-vc-total>—</span></div>
</div>
```

토큰을 **JS 안에 직접 넣지 않고** `data-*` attribute 로 빼는 게 작은 포인트입니다. 토큰을 다루는 위치가 한 군데(`_data/goatcounter.yml` + Liquid 주입) 로 모여서 디버깅·교체가 쉬워집니다.

**fetch 는 두 endpoint 로 갈립니다** — 각자의 강점에 맞춘 결정입니다.

| 항목 | endpoint | 인증 | 응답 |
|---|---|---|---|
| 오늘 | `/api/v0/stats/total?start=…&end=…` | Bearer token | ~800B |
| 전체 | `/counter/TOTAL.json` | **없음 (public)** | ~44B |

오늘은 날짜 범위 쿼리가 필요해 인증 API 가 자연스럽고, 전체는 누적 한 줄이라 public counter 가 훨씬 가볍습니다 (인증 헤더도 응답 크기도 절약). **같은 도구 안에서도 호출 목적에 따라 endpoint 의 무게가 달라진다**는 게 위젯 만들면서 처음 부딪힌 지점이었습니다.

마지막은 캐시. 페이지 로드마다 두 번씩 fetch 하는 건 GoatCounter 입장에서도 사용자 입장에서도 낭비입니다.

```javascript
var TTL_TODAY = 5 * 60 * 1000;       /* 5분 */
var TTL_TOTAL = 6 * 60 * 60 * 1000;  /* 6시간 */

function readCache(key, ttl) {
  var raw = localStorage.getItem(key);
  if (!raw) return null;
  var obj = JSON.parse(raw);
  if (Date.now() - obj.t > ttl) return null;
  return obj.v;
}
```

오늘은 5분, 전체는 6시간. 누적 숫자는 분 단위로 바뀌지 않으므로 길게 잡아도 무방합니다. 키 이름에 버전 접미사 (`gc:today-v2:<날짜>`, `gc:total-v2`) 를 붙여둔 게 작은 안전장치입니다 — 로직을 바꿀 때 **이전 캐시를 invalidate 하는 가장 단순한 방법은 키 버전을 올리는 것**입니다.

## 남는 것

**한 도구로 묶을 수 있는 것은 묶는다.**
글 메타 조회수와 사이드바 위젯이 같은 GoatCounter ID 안에 있다는 사실이, 데이터 일관성과 관리 지점을 한 군데로 모아줍니다. 두 도구를 같이 운영했다면 무엇이 실제 트래픽인지부터 매번 검증해야 했을 것입니다.

**토큰 노출 가능 여부는 권한 범위가 결정한다.**
"비밀이니까 항상 환경변수" 가 기본값은 아닙니다. read-only 라는 보장이 있으면 빌드 결과물에 평문이어도 합리적이고, 그 보장을 명시해 두는 지점(파일 헤더 코멘트, 토큰 생성 시 권한 체크) 이 더 중요합니다.

**fetch endpoint 의 무게는 응답 크기와 인증 필요로 가른다.**
같은 도구 안에서도 호출 목적마다 endpoint 가 갈릴 수 있습니다. 위젯의 응답성은 라이브러리가 아니라 endpoint 선택이 결정하는 경우가 의외로 많습니다.
