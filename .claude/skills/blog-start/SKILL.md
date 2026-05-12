---
name: blog-start
description: 현재 Claude Code 대화를 블로그 글 작성 대상으로 마킹한다. 인자로 주제를 넘기면 그 주제를, 안 넘기면 사용자에게 한 번 묻고 받아서 `.claude/blog-session.md` 에 주제·slug·카테고리·태그 메타데이터를 기록한다. 이후 /blog-draft 가 이 파일을 읽어 초안을 작성한다.
---

# blog-start

이 Skill 이 호출되면 다음 절차를 따른다.

## 1. 주제 확보

- 호출 시 인자(`$ARGUMENTS`)가 있으면 그것을 그대로 주제(`topic`)로 사용한다.
- 없으면 사용자에게 **단 한 번** 묻는다: "어떤 주제로 정리할까요? (예: 'JPA N+1 해결기', 'Spring Boot Redis 캐시 설정')"
- 사용자가 답하면 그 답을 주제로 사용한다.

## 2. slug 생성

주제에서 URL 친화적인 slug 를 만든다.

- 소문자, 하이픈(`-`) 구분, ASCII 권장
- 한글 주제면 핵심을 영문으로 의역 (예: "JPA N+1 해결기" → `jpa-n-plus-1-fix`)
- 30자 이내, 불용어/관사 제거

## 3. 카테고리·태그 추정

- `categories`: `backend`, `infra`, `frontend`, `tooling`, `language`, `etc` 중 1~2개를 주제 기반으로 추정
- `tags`: 구체적 키워드 3~5개 (예: `spring-boot`, `jpa`, `n+1`, `query-optimization`)
- 한글 태그도 허용하되 가능하면 영문 우선

## 4. 메타데이터 파일 작성

`.claude/blog-session.md` 파일을 다음 형식으로 **덮어쓰기** 한다 (이 파일은 `.gitignore` 에 포함되어 커밋되지 않음):

```markdown
---
topic: <주제>
slug: <slug>
started_at: <YYYY-MM-DD HH:MM>
categories: [<cat1>, <cat2>]
tags: [<tag1>, <tag2>, <tag3>]
---

# 대화 메모

(대화 중 핵심 포인트·결정·코드 스니펫 위치 등을 누적 메모해도 됨)
```

`started_at` 의 날짜·시간은 시스템 현재 시각을 사용한다(Asia/Seoul 기준이면 그대로 표기).

## 5. 기존 세션 처리

`.claude/blog-session.md` 가 **이미 존재**하면 덮어쓰기 전에 사용자에게 한 번 확인한다:

> "기존 블로깅 세션('<기존 topic>') 이 있어요. 새 주제 '<새 topic>' 으로 덮어쓸까요? 아니면 기존 세션을 계속 쓸까요?"

사용자가 명시적으로 동의해야 덮어쓴다.

## 6. 확인 메시지

작성 후 사용자에게 한 줄로 요약한다:

```
🔖 블로깅 모드 시작 — 주제: <topic> · slug: <slug>
   카테고리: <categories> · 태그: <tags>
   이제 자유롭게 대화하시면 됩니다. 마무리할 때 /blog-draft 실행.
```

## 작성 원칙

- 추정한 카테고리·태그가 모호하면 사용자에게 한 번 확인해도 좋다 (선택).
- `.claude/` 디렉토리가 없으면 만든다.
- 인자 파싱 시 따옴표·공백을 그대로 보존.
