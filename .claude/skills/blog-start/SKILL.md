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

**topic 길이 가이드**: title 로 그대로 들어가므로 **핵심 키워드만, 20자 이내**가 좋다. 부연 설명·증상은 분리해서 `subtitle` 로 잡는다 (다음 단계).

**subtitle 은 항상 함께 잡는다 (필수)**. /blog-draft 가 이걸 `description:` 필드로 매핑하고, description 은 모든 글에 채워져야 한다.

- 사용자가 한 문장으로 길게 던지면 (예: "Excel 자동 생성 파일이 매번 자체 복구를 띄운 이유 — POI freeze pane 과 활성 셀의 함정") 두 토막으로 쪼개서 확인한다:
  - `topic`: `POI freeze pane 과 활성 셀의 함정`
  - `subtitle`: `Excel 자동 생성 파일이 매번 자체 복구를 띄운 이유`
- 쪼개기 애매하면 사용자에게 한 번만 더 확인. ("이 두 토막으로 나눌까요? — topic: X / subtitle: Y")
- 사용자가 topic 만 짧게 던지고 끝나면 subtitle 을 한 줄 요약으로 직접 제안하고 확인받는다. ("부제는 'XYZ' 정도로 잡을까요?") 확정 안 되면 빈 값으로 두지 말고 `subtitle: <topic 보조 설명>` 형태로라도 채워 놓는다 — 이후 /blog-draft 단계에서 본문 요약으로 다듬을 수 있다.

## 2. slug 생성

주제에서 URL 친화적인 slug 를 만든다.

- 소문자, 하이픈(`-`) 구분, ASCII 권장
- 한글 주제면 핵심을 영문으로 의역 (예: "JPA N+1 해결기" → `jpa-n-plus-1-fix`)
- 30자 이내, 불용어/관사 제거

## 3. 카테고리·태그 추정

- `categories`: `backend`, `infra`, `frontend`, `tooling`, `language`, `etc` 중 1~2개를 주제 기반으로 추정
- `tags`: 구체적 키워드 3~5개 (예: `spring-boot`, `jpa`, `n+1`, `query-optimization`)
- 한글 태그도 허용하되 가능하면 영문 우선
- **중요**: `categories[0]` 이 곧 포스트 파일이 들어갈 폴더 이름이 된다 (`_posts/<categories[0]>/...`). 위 6개 카테고리 중 하나로 통일해서 폴더 일관성을 유지.

## 4. 메타데이터 파일 작성

`.claude/blog-session.md` 파일을 다음 형식으로 **덮어쓰기** 한다 (이 파일은 `.gitignore` 에 포함되어 커밋되지 않음):

```markdown
---
topic: <주제>                      # 그대로 title 로 들어가므로 짧게 (20자 내외)
subtitle: <부제>                   # 필수. /blog-draft 가 description: 필드로 매핑 (비울 수 없음)
slug: <slug>
started_at: <YYYY-MM-DD HH:MM>
categories: [<cat1>, <cat2>]
tags: [<tag1>, <tag2>, <tag3>]
# 시리즈 글이면 다음 블록을 추가한다. 단편이면 생략.
# series:
#   name: <series-slug>    # _data/series.yml 의 키 (예: docker)
#   part: <N>              # 1, 2, 3...   /blog-draft 가 title 끝에 [N편] 으로 자동 부착
---

# 대화 메모

(대화 중 핵심 포인트·결정·코드 스니펫 위치 등을 누적 메모해도 됨)
```

사용자가 "시리즈로 작성" 의향을 비치면 `series.name` (영문 slug) 과 `part` 를 함께 잡는다. 새 시리즈면 `_data/series.yml` 에 시리즈 이름·설명도 한 블록 추가하는 것이 깔끔하다 (이건 /blog-draft 단계에서 처리해도 됨).

`started_at` 의 날짜·시간은 시스템 현재 시각을 사용한다(Asia/Seoul 기준이면 그대로 표기).

## 5. 기존 세션 처리

`.claude/blog-session.md` 가 **이미 존재**하면 덮어쓰기 전에 사용자에게 한 번 확인한다:

> "기존 블로깅 세션('<기존 topic>') 이 있어요. 새 주제 '<새 topic>' 으로 덮어쓸까요? 아니면 기존 세션을 계속 쓸까요?"

사용자가 명시적으로 동의해야 덮어쓴다.

## 6. 확인 메시지

작성 후 사용자에게 한 줄로 요약한다 (subtitle 이 잡힌 경우만 두 번째 줄 추가):

```
🔖 블로깅 모드 시작 — 주제: <topic> · slug: <slug>
   부제: <subtitle>              # subtitle 이 있을 때만
   카테고리: <categories> · 태그: <tags>
   이제 자유롭게 대화하시면 됩니다. 마무리할 때 /blog-draft 실행.
```

## 작성 원칙

- 추정한 카테고리·태그가 모호하면 사용자에게 한 번 확인해도 좋다 (선택).
- `.claude/` 디렉토리가 없으면 만든다.
- 인자 파싱 시 따옴표·공백을 그대로 보존.
