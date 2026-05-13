---
name: blog-cover
description: 지정한 (혹은 최근 작성한) 블로그 포스트의 본문·메타데이터를 분석해 codex CLI 로 커버 이미지(960x540, JPEG 품질 82) 를 자동 생성한다. 결과는 `assets/img/posts/<slug>/cover.jpg` 로 저장되고, 필요하면 포스트 frontmatter 의 `image.path` 도 함께 갱신한다.
---

# blog-cover

`/blog-cover [post-file-or-slug]` 형태로 호출한다. 인자가 없으면 다음 우선순위로 대상 글을 결정한다.

1. `.claude/blog-session.md` 의 `slug` 가 가리키는 `_posts/<categories[0]>/YYYY-MM-DD-<slug>.md`
2. `_posts/` 하위 가장 최근 수정된 `.md` 파일
3. 둘 다 없거나 모호하면 사용자에게 한 번 묻고 종료.

## 1. 컨텍스트 추출

대상 포스트 파일을 Read 로 읽고 다음을 뽑는다.

| 자리표시자 | 추출 위치 |
|---|---|
| `<SLUG>` | 파일명 `YYYY-MM-DD-<slug>.md` 에서 |
| `<TITLE>` | frontmatter `title:` |
| `<DESCRIPTION>` | frontmatter `description:` |
| `<CATEGORY>` | frontmatter `categories[0]` |
| `<ALT>` | frontmatter `image.alt:` (없으면 빈 문자열) |
| `<LEAD_PARAGRAPHS>` | frontmatter 종료(`---`) 이후 첫 인용구(`>` 줄) 와 그 다음 1~2 단락. 시리즈 글이면 `{% include series-nav.html %}` 줄은 건너뛴다. |

`<LEAD_PARAGRAPHS>` 는 1500자 이내로 잘라 codex 컨텍스트 비용을 제한한다.

## 2. codex exec 호출

Bash 툴로 PowerShell here-string 을 그대로 넘긴다. 플레이스홀더는 1절에서 추출한 값으로 모두 치환한 뒤 실행한다.

```powershell
codex exec `
  --sandbox workspace-write `
  -a never `
  @'
다음 블로그 글의 커버 이미지를 1장 만들어 정확히 아래 경로에 저장해줘.

[저장 경로 — 반드시 이 경로에만 저장, 다른 위치 금지]
assets/img/posts/<SLUG>/cover.jpg

[이미지 사양]
- 해상도 960x540 (16:9)
- JPEG (.jpg), 품질 82
- 파일명 cover.jpg 고정
- 디렉터리가 없으면 생성
- 기존 파일이 있으면 덮어쓰기

[스타일 — 기존 블로그 커버와 일관성 필수]
- 톤: 기술 일러스트, 단정한 다이어그램 풍, 라인·박스·플로우 강조
- 색감: 글 내용·분위기에 맞춰 자유롭게 선택. 단, 차분한 톤 유지하고 화려한 네온·게이밍·캐릭터 톤 금지
- 인물·얼굴 없음
- 텍스트·문자·로고 일체 포함 금지 (한글·영문 모두)

[콘텐츠 컨텍스트]
- 제목: <TITLE>
- 부제: <DESCRIPTION>
- 카테고리: <CATEGORY>
- 본문 도입부:
<LEAD_PARAGRAPHS>

- 의도된 alt 텍스트: <ALT>

위 컨텍스트의 **핵심 구조·관계**를 시각화해줘. 단순 장식이 아니라, 글이 다루는 시스템·흐름·관계를 다이어그램스럽게 추상화한 결과여야 한다.

[금지]
- 저장 경로 임의 변경 금지 (반드시 assets/img/posts/<SLUG>/cover.jpg)
- 추가 파일 생성 금지 (cover 1장만)
- 본문 파일(.md) 수정 금지 — 이미지 파일만 만든다
- 종료 시 작업 요약 외 다른 부수 작업 금지
'@
```

호출 시 주의:

- **Bash 툴 timeout 을 600000 (10분)** 으로 늘려서 호출한다. 이미지 생성 모델 응답이 1~3분 걸릴 수 있다.
- `@'...'@` single-quoted here-string 이라 본문에 `$`, 백틱이 있어도 PowerShell 이 보간하지 않는다 — 그대로 안전.
- `<LEAD_PARAGRAPHS>` 안에 `'@` 시퀀스가 들어가지 않도록 치환 시 한 번 점검. (드물지만 인용 닫는 토큰 충돌 가능.)

## 3. 결과 검증

호출이 끝나면 다음을 확인:

1. `assets/img/posts/<SLUG>/cover.jpg` 가 실제로 존재하는지 Glob 으로 확인.
2. 파일이 없으면 codex stdout 의 마지막 30 줄을 사용자에게 보여주고 실패 보고. 재시도 여부를 묻는다.
3. 파일이 있으면 크기를 확인 (`Get-Item ... | Select-Object Length`). 0~10KB 사이로 비정상적으로 작으면 실패 의심 — 사용자에게 보여주고 확인.

## 4. frontmatter 갱신 (필요 시)

대상 포스트 파일의 frontmatter 를 점검:

- `image.path` 가 비어 있거나 다른 값이면 `/assets/img/posts/<SLUG>/cover.jpg` 로 갱신.
- `image.alt` 가 비어 있으면 글 핵심을 한 줄로 요약해서 채운다. 기존 블로그의 alt 톤은 *"~을(를) 표현한 기술 커버 이미지"* 로 끝나는 한국어 한 문장. 사용자에게 한 번 보여주고 확인받는다.
- `image:` 블록 자체가 없으면 `date:` 줄 바로 아래에 다음 블록을 삽입:

  ```yaml
  image:
    path: /assets/img/posts/<SLUG>/cover.jpg
    alt: <ALT_TEXT>
  ```

## 5. 사용자 안내

작성 후 한 번에 출력:

```
🖼️  커버 생성 완료
   파일: assets/img/posts/<SLUG>/cover.jpg
   크기: <bytes>
   적용된 alt: <alt>

다음 단계:
  1) 이미지 미리보기로 톤·구성 확인
  2) 마음에 안 들면 /blog-cover 다시 실행 (덮어쓰기)
  3) 발행 흐름은 그대로 (git add → commit → push)
```

## 6. 작성 원칙

- **경로 이탈 금지**: 프롬프트에 박힌 저장 경로가 codex 판단의 외부에 있도록 강제. 결과가 다른 곳에 저장됐으면 실패로 보고 사용자에게 알린다.
- **본문 수정 금지**: codex 호출은 이미지 파일 1장만 만든다. codex 실행 중 본문 .md 가 수정된 흔적이 보이면 즉시 사용자에게 보고하고 `git diff` 를 보여준다.
- **재실행 안전**: 같은 slug 로 다시 호출하면 `cover.jpg` 만 덮어쓴다. 다른 파일을 만들지 않는다.
- **codex 인증 누락**: codex 가 로그인되어 있지 않으면 codex 가 직접 에러 메시지를 띄운다. 그 출력을 그대로 사용자에게 전달하고 `codex login` 을 안내.
- **모델 지정**: 기본은 codex 의 기본 모델. 사용자가 인자에 `--model gpt-image-1` 같은 codex 옵션을 같이 주면 그대로 전달.
