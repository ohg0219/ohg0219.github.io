---
title: "Jekyll 블로그 자동화 셋업"
date: 2026-05-12 12:58:00 +0900
categories: [tooling]
tags: [jekyll, blog, automation, claude-code, github-pages]
redirect_from:
  - /2026/05/12/jekyll-blog-automation-setup/
---

## 문제 상황

Claude Code 와 나눈 Q&A, 에러 해결 과정, 기술 탐구 같은 휘발성 높은 학습 메모를 어딘가에 차곡차곡 쌓고 싶었습니다. 요구 사항은 단순했습니다.

- 마크다운으로 쓰고 싶다 (에디터 자유, 버전 관리 가능)
- 별도 서버·DB 운영은 절대 사절
- "대화 → 글 초안" 사이의 마찰을 최대한 줄이고 싶다 — 매번 새 파일 만들고 frontmatter 채우는 게 귀찮으면 안 씀
- 배포 파이프라인은 `git push` 한 번으로 끝나야 함

환경은 Windows 11 + PowerShell, 글쓰기 도구로는 Claude Code (CLI) 를 이미 일상적으로 쓰고 있는 상태였습니다.

## 시도와 발견

가장 먼저 고민한 건 호스팅·빌드 방식의 조합이었습니다.

**1) "어디서 빌드할 것인가" 결정.** Jekyll 은 정적 사이트 생성기이므로 빌드 산출물(`_site/`) 을 어딘가에서 만들어 어딘가에 올려야 합니다. 옵션은 대략 세 가지였습니다.

- 로컬에서 빌드 → 산출물을 푸시 (구식, 산출물이 레포를 더럽힘)
- GitHub Actions 로 빌드 → `gh-pages` 브랜치로 배포 (유연하지만 워크플로 YAML 유지보수 필요)
- **GitHub Pages 의 내장 Jekyll 빌더에 맡기기** (소스만 푸시하면 됨)

플러그인 자유도가 굳이 필요 없다면 마지막이 압도적으로 편합니다. 레포 이름을 `<username>.github.io` 로 만들면 GH Pages 가 user site 로 자동 인식하고, main 브랜치 루트를 곧바로 사이트 소스로 본다는 점이 결정타였습니다.

**2) 최소 스켈레톤이 어디까지 줄어드는지 확인.** Jekyll 의 기본 테마 `minima` 가 GH Pages 화이트리스트에 포함돼 있어서, `_layouts`·`_includes` 도 직접 만들 필요가 없습니다. 결과적으로 레포에 들어간 파일은 다음이 전부:

```
.
├── _config.yml      # 사이트 메타·테마·플러그인·permalink
├── Gemfile          # github-pages gem (로컬 = GH Pages 버전 일치)
├── _posts/          # 글이 들어갈 디렉터리
├── index.md         # 홈 페이지 (layout: home)
└── .gitignore       # _site/, Gemfile.lock 등 제외
```

**3) "대화 → 초안" 의 마찰을 어떻게 줄일지.** 여기서 Claude Code 의 Skill 기능을 활용하기로 했습니다. 글 한 편당 두 단계만 기억하면 되도록 슬래시 커맨드 두 개를 만들었습니다.

- `/blog-start <주제>` — 대화 시작 시 `.claude/blog-session.md` 에 topic / slug / categories / tags 를 기록
- `/blog-draft` — 대화 마무리에 호출하면 위 메타데이터 + 대화 전체를 읽어 `_posts/YYYY-MM-DD-<slug>.md` 초안 생성

이 글 자체가 그 두 Skill 로 작성된 결과물입니다.

## 해결책

### `_config.yml`

```yaml
title: Hayden's Dev Notes
description: AI 와 함께 정리하는 개발 학습 기록
author: Hayden
url: https://ohg0219.github.io
theme: minima
plugins:
  - jekyll-feed
permalink: /:year/:month/:day/:title/
timezone: Asia/Seoul

defaults:
  - scope:
      path: ""
      type: posts
    values:
      layout: post

exclude:
  - Gemfile
  - Gemfile.lock
  - vendor
  - .bundle
  - README.md
  - .claude
```

`exclude` 에 `.claude` 를 넣어둔 게 작은 포인트입니다. Skill 이 쓰는 임시 세션 파일이 빌드에 끌려 들어가지 않게 막아줍니다.

### `Gemfile`

```ruby
source "https://rubygems.org"

gem "github-pages", group: :jekyll_plugins
gem "webrick", "~> 1.8"

group :jekyll_plugins do
  gem "jekyll-feed"
end
```

`github-pages` gem 한 줄이 GH Pages 가 실제 서버에서 쓰는 Jekyll 버전·플러그인 세트를 그대로 끌어옵니다. 로컬과 운영의 버전이 어긋날 일이 없어집니다. `webrick` 은 Ruby 3.x 부터 표준 라이브러리에서 빠졌으므로 명시.

### `.gitignore` 의 핵심 라인

```
_site/
.jekyll-cache/
Gemfile.lock

# 대화 단위 임시 메타데이터 (Skill 이 매번 새로 씀)
.claude/blog-session.md
```

빌드 산출물과 Skill 의 휘발성 메타데이터를 같이 막아둡니다.

### 로컬 미리보기

Windows 에서는 RubyInstaller 의 **Ruby+Devkit** 버전을 받아 설치하고, MSYS2 development toolchain 까지 함께 설치하는 것이 가장 잡음이 적습니다. 그 다음은 두 줄로 끝납니다.

```powershell
bundle install
bundle exec jekyll serve --livereload
```

`http://127.0.0.1:4000` 으로 접속하면 마크다운 저장 즉시 브라우저가 자동 리프레시됩니다. 단, `_config.yml` 만은 서버 시작 시 한 번만 읽히므로 변경 후엔 `Ctrl+C` 로 재시작해야 합니다.

### 배포

```powershell
git add _posts/YYYY-MM-DD-<slug>.md
git commit -m "post: <주제>"
git push
```

이것이 전부입니다. GH Pages 가 푸시를 감지해 Jekyll 빌드를 돌리고 약 1분 내에 사이트에 반영합니다. 별도 워크플로 YAML 도, 배포 키도 필요 없습니다.

### 글쓰기 워크플로

```
대화 시작 → /blog-start "주제"
   ↓
자유롭게 Claude Code 와 Q&A
   ↓
/blog-draft        ← _posts/ 에 초안 마크다운 생성
   ↓
에디터에서 검토·수정
   ↓
git push           ← GH Pages 자동 빌드·배포
```

## 배운 점

**1) 배포 파이프라인은 가장 단순한 옵션부터 시도하라.** GitHub Actions 로 직접 빌드하는 구성은 자유도가 높지만, 워크플로 YAML 자체가 또 하나의 유지보수 대상이 됩니다. 이번처럼 화이트리스트 외 플러그인이 필요 없으면 GH Pages 의 내장 빌더가 압도적으로 편합니다. "자유도가 필요해질 때 마이그레이션" 을 default 로 두고 시작하는 게 낫습니다.

**2) 로컬 환경과 운영 환경의 버전을 한 gem 으로 묶기.** `github-pages` gem 하나로 Jekyll · minima · 허용 플러그인 버전을 통째로 핀 고정하는 패턴은 "로컬에선 잘 되는데" 류의 버전 차이 디버깅을 통째로 없애줍니다.

**3) "마찰을 줄이는 자동화" 가 글 쓸 동기를 결정한다.** 글쓰기는 환경이 1분만 거추장스러워도 안 쓰게 됩니다. 새 포스트마다 파일명·frontmatter·날짜를 직접 채우는 작업조차도 충분히 큰 마찰입니다. 도구 위에 얇은 자동화 레이어(이 경우 Skill 두 개) 를 얹어두면, 의지력이 아니라 관성으로 글을 쓸 수 있게 됩니다.

다음에 비슷한 "기록용 사이트" 를 세팅할 때 가장 먼저 확인할 두 가지는:
- 호스팅 제공자에 **내장 빌더**가 있는가? (있으면 거기 먼저 얹어본다)
- 글 한 편당 **새로 만드는 파일 수** 가 몇 개인가? (둘 이상이면 스크립트화 후보)

---

## 참고

- [RubyInstaller for Windows](https://rubyinstaller.org/downloads/)
- [GitHub Pages 공식 문서 — Jekyll](https://docs.github.com/en/pages/setting-up-a-github-pages-site-with-jekyll)
- [Jekyll 공식 사이트](https://jekyllrb.com/)
