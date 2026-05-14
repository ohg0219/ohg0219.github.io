---
title: "GSC 에 블로그 등록하기"
description: "_config.yml 한 블록과 sitemap 제출까지"
date: 2026-05-13 17:24:31 +0900
categories: [tooling]
tags: [google-search-console, seo, sitemap, jekyll, chirpy]
image:
  path: /assets/img/posts/google-search-console-setup/cover.jpg
  alt: GitHub Pages 블로그를 Google Search Console 에 등록하고 사이트맵을 제출하는 흐름을 표현한 기술 커버 이미지
---

**"내 블로그를 구글 검색에 띄우고 싶었습니다."** 글을 몇 편 쌓다 보니 이 한 줄짜리 욕망이 점점 단단해졌는데, 정작 검색해보면 사이트가 안 나왔습니다. **Google Search Console (GSC) 에 등록하는 한 단계가 통째로 비어 있었기 때문입니다.**

이 글은 그 한 단계를 채운 기록입니다. GitHub Pages 호스팅이라는 환경 제약 안에서 Property type 선택부터 sitemap 제출까지, `_config.yml` 한 블록으로 끝나는 흐름.

## Property type — URL prefix 를 선택한 이유

GSC 에 사이트를 등록하면 처음 묻는 게 Property type 입니다. 두 가지 옵션이 있는데 GitHub Pages 환경에서는 선택지가 거의 좁혀집니다.

- **Domain property** — 도메인 전체(서브도메인 포함) 를 한 번에 등록. verification 은 **DNS TXT 레코드** 로 진행.
- **URL prefix property** — URL 단위로 등록. verification 옵션이 다양 (HTML tag · HTML 파일 · Google Analytics 등).

GitHub Pages 의 `<user>.github.io` 는 GitHub 의 서브도메인이라 **DNS 레코드를 직접 만질 수 없습니다.** Domain property 가 필요로 하는 TXT 레코드를 추가할 권한이 없어서, GH Pages 호스팅 사이트에서는 **URL prefix property + HTML tag verification** 조합이 현실적인 선택지입니다.

`https://ohg0219.github.io` 를 URL prefix property 로 등록하고 다음 단계로 넘어갑니다.

## 소유 확인 — HTML tag + `_config.yml` 한 블록

verification 옵션 중 **"HTML tag"** 를 선택하면 GSC 가 다음 형태의 코드를 발급합니다.

```html
<meta name="google-site-verification" content="<VERIFICATION_CONTENT>" />
```

이 meta tag 를 사이트의 `<head>` 에 넣는 게 verification 의 전부입니다. 직접 layout 을 override 해서 넣을 수도 있지만, **Chirpy 는 `_config.yml` 의 `webmaster_verifications` 블록을 자동으로 인식**해서 head 에 넣어줍니다.

```yaml
# _config.yml
webmaster_verifications:
  google: <VERIFICATION_CONTENT>   # ⚠️ GSC 의 HTML tag 옵션에서 받은 content 값
```

`<VERIFICATION_CONTENT>` 자리에 GSC 가 발급한 값만 채우고 commit · push 하면 됩니다. GitHub Actions 빌드(1~2분) 가 끝나면 사이트 head 에 meta tag 가 들어가 있고, GSC 콘솔의 **"확인"** 버튼을 누르면 verification 이 통과합니다.

> Chirpy 의 `webmaster_verifications` 는 google 외에 bing · alexa · yandex · baidu · facebook 등 다른 검색엔진 verification 도 같이 받습니다. 한 블록 안에 키만 추가하면 됩니다.

## Sitemap 제출

verification 이 끝나면 GSC 의 실질적인 시작점은 sitemap 제출입니다. 이 블로그는 `jekyll-sitemap` 플러그인이 켜져 있어서 빌드 시 `https://ohg0219.github.io/sitemap.xml` 이 자동으로 생성됩니다.

GSC 좌측 메뉴 → **Sitemaps** → `sitemap.xml` 입력 → 제출. 이후 새 글이 push 될 때마다 sitemap.xml 이 갱신되고, GSC 가 주기적으로 다시 크롤합니다.

## 남는 것

**GitHub Pages 호스팅 사이트의 GSC 등록은 선택지가 좁습니다.**
URL prefix property + HTML tag verification 조합이 DNS 접근 없이 가능한 현실적인 선택지이고, 그 안에서 Chirpy 의 `webmaster_verifications` 가 layout override 없이 `_config.yml` 한 블록으로 끝내줍니다. **호스팅 환경의 제약이 옵션을 한 곳으로 모아주는 경우** — 선택 비용이 0 에 가까워집니다.

**`jekyll-sitemap` 한 번 켜두면 평생 작동.**
새 글이 추가되거나 기존 글이 수정될 때마다 sitemap.xml 이 자동 갱신됩니다. GSC 에는 한 번만 제출해두면 이후엔 손댈 일이 없습니다 — Jekyll 의 정적 사이트 흐름이 가장 잘 맞는 부분입니다.

**검색 결과에 글이 잡히는 데는 시간이 걸립니다.**
verification · sitemap 제출이 끝나도 GSC 가 실제로 크롤 · 인덱싱하는 데는 며칠에서 한두 주가 걸립니다. 빠져 있던 단계가 채워졌으니, 다음은 기다림입니다.
