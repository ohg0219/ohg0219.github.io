---
title: "giscus 로 댓글 기능 켜기"
description: "GitHub Discussions 를 댓글 저장소로"
date: 2026-05-13 14:37:08 +0900
categories: [tooling]
tags: [giscus, comments, github-discussions, jekyll, chirpy]
image:
  path: /assets/img/posts/giscus-comments-setup/cover.jpg
  alt: giscus 댓글 위젯과 GitHub Discussions 저장소 연결을 표현한 기술 커버 이미지
---

댓글 받을 자리를 켜고 싶었습니다. 외부 SaaS 위젯에 의존하지 않고, 저장소를 내가 통제할 수 있는 자리에 두고 싶었고요. **이 블로그는 giscus 로 그 두 가지를 한 번에 만족시켰습니다** — 댓글이 GitHub Discussions 에 그대로 저장되고, Chirpy 의 `_config.yml` 한 블록이면 댓글 자리가 자동으로 켜집니다.

이 글은 그 셋업을 처음부터 끝까지 정리합니다. 사전 작업 세 단계와 `_config.yml` 블록 하나, 그리고 운영 디테일 한 줌.

## 왜 giscus 인가

비교 후보는 Disqus, utterances, commento, GitHub 자체 솔루션 정도였습니다. 결정타가 된 셋만 짚어 봅니다.

- **댓글 저장소가 GitHub Discussions** 입니다. repo 의 Discussions 카테고리에 글타래가 매핑돼 댓글이 쌓입니다. repo 를 백업·이전·재구성할 때 **댓글이 자연스럽게 따라옵니다** — 외부 SaaS 락인이 없습니다.
- **쿠키·트래킹 없음.** 댓글 위젯이 방문자 식별을 위해 쿠키를 깔지 않습니다. GDPR 동의 배너 자리가 비어 있어도 됩니다.
- **광고 없음 / 무료.** repo 만 public 이면 비용도 광고도 없습니다.

부수로 Chirpy 7.x 가 댓글 자리(`comments:` 블록) 를 layout 에 미리 만들어 두어, `_config.yml` 한 블록만 채우면 post 끝에 위젯이 자동으로 박힙니다.

## 사전 작업 — 세 단계

**1. Repo Discussions 활성화.** repo Settings → Features → "Discussions" 체크. 댓글이 저장될 자리가 만들어집니다.

**2. giscus GitHub App 설치.** [github.com/apps/giscus](https://github.com/apps/giscus) 에서 해당 repo (또는 모든 repo) 에 install. 이 App 이 댓글 위젯과 repo Discussions 사이의 다리 역할을 합니다.

**3. giscus.app 에서 설정값 발급.** [giscus.app](https://giscus.app) 에 가서 repo 주소를 입력하면, 페이지 하단에 `data-repo-id`, `data-category-id` 같은 값들이 자동으로 채워집니다. 이 두 ID 가 다음 블록의 핵심 값입니다.

> Discussions 카테고리는 **"Announcements" 형식** 을 권장합니다 — 카테고리 owner 만 새 토론을 열 수 있어, 일반 토론 카테고리와 댓글 글타래가 섞이지 않습니다.

## `_config.yml` 의 `comments:` 블록

Chirpy 7.x 기준, 한 블록.

```yaml
comments:
  provider: giscus
  giscus:
    repo: <OWNER>/<REPO>
    repo_id: "<REPO_ID>"          # ⚠️ giscus.app 에서 발급받은 실제 값
    category: Announcements
    category_id: "<CATEGORY_ID>"  # ⚠️ giscus.app 에서 발급받은 실제 값
    mapping: pathname
    strict: "0"
    input_position: bottom
    lang: ko
    reactions_enabled: "1"
```

각 키의 의미를 한 줄씩.

- `provider: giscus` — Chirpy 가 post layout 의 댓글 자리에 giscus 위젯을 자동 박는 스위치
- `mapping: pathname` — URL 경로를 글타래 식별자로. 다른 옵션 (title, og:title, specific, number) 보다 **경로 안정성이 가장 높음** — title 을 다듬어도 글타래가 새로 안 생깁니다
- `strict: "0"` — 경로가 살짝 달라져도 같은 글타래로 매칭. `"1"` 이면 strict matching
- `category: Announcements` — 댓글 전용 카테고리. 일반 토론과 분리하는 자리
- `lang: ko` / `reactions_enabled: "1"` — 한국어 UI + 글 reaction 활성화
- `input_position: bottom` — 입력창을 글타래 하단에 배치

이 블록을 박고 `bundle exec jekyll serve` 로 띄우면, post 끝에 giscus 위젯 iframe 이 자동으로 박힙니다. 첫 댓글이 달리는 순간 repo Discussions 에 해당 글의 글타래가 새로 만들어지는 식.

## 남는 것

**댓글 저장소를 인프라 안에 두는 가치.**
giscus 의 결정적 매력은 디자인이나 무료가 아니라, **댓글이 이미 통제하고 있는 인프라 (GitHub repo) 안에 쌓인다** 는 점입니다. 글을 옮기든 블로그 엔진을 갈아엎든, 댓글이 따라옵니다. 외부 SaaS 의 데이터 export 기능을 신뢰해야 하는 자리 자체가 없어집니다.

**Chirpy 가 댓글 자리를 layout 에 미리 만들어 둔 덕.**
`_config.yml` 한 블록으로 끝나는 건 giscus 가 가벼워서가 아니라 Chirpy 의 `_includes/comments.html` 가 미리 자리를 잡아둔 결과입니다. 다른 테마였다면 layout 을 override 하거나 위젯 스크립트를 직접 박아야 했을 자리.

**한 줄로 추리면 — 셋업의 무게중심은 "사전 작업 3단계" 에 있다.**
실제 코드는 `_config.yml` 한 블록뿐이지만, Discussions 활성화 / GitHub App 설치 / giscus.app 설정값 발급 세 단계가 먼저 끝나야 합니다. 새 블로그에 댓글을 켤 때 가장 먼저 열어둘 탭 세 개가 정해집니다.
