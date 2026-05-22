---
name: blog-crosspost
description: 발행된 Jekyll 포스트(`_posts/<category>/*.md`)를 velog·tistory 호환 마크다운으로 변환하고, Playwright MCP 로 두 플랫폼에 임시저장(velog '임시저장' / tistory '비공개 저장')까지 자동 입력한다. 공개 발행 버튼은 절대 누르지 않으며, 최종 발행은 사용자가 직접 한다.
---

# blog-crosspost

이미 GitHub Pages 에 배포된 블로그 글을 velog·tistory 에 크로스포스팅한다.
이 Skill 이 호출되면 다음 절차를 따른다.

**이 Skill 의 전제와 한계 (먼저 이해할 것)**

- velog 는 공식 발행 API 가 없고 tistory Open API 는 2024년 종료됐다. 그래서
  Playwright MCP 로 **브라우저를 직접 조작**해 입력한다.
- 로그인은 OAuth(카카오·구글·깃허브) + 캡차·2FA 라서 자동화할 수 없다.
  **최초 1회는 사용자가 브라우저 창에서 직접 로그인**해야 한다 (5절·6절 참고).
- 그래서 이 Skill 은 `git push` 자동 트리거가 아니라 **사용자가 수동 호출**한다.
  권장 시점: 글을 push 해서 GitHub Pages 배포가 끝나 원문 URL 이 살아난 뒤.
- 이 Skill 은 **임시저장까지만** 한다. velog '출간하기' / tistory '공개 발행'
  버튼은 **어떤 경우에도 누르지 않는다.** 변환 결과를 사용자가 각 플랫폼에서
  검토하고 직접 발행한다.

## 1. 대상 포스트·플랫폼 선택

호출 인자(`$ARGUMENTS`)를 공백으로 토큰화해 해석한다.

- 토큰이 `velog` 또는 `tistory` 면 → **대상 플랫폼 필터**. 둘 다 없으면 기본값은
  두 플랫폼 모두.
- 그 외 토큰은 → **대상 포스트 지정**(slug 또는 파일 경로). slug 는 파일명
  `YYYY-MM-DD-<slug>.md` 의 `<slug>` 부분으로 매칭한다.

| 호출 예 | 해석 |
|---|---|
| `/blog-crosspost` | 최신 포스트 자동 선택, velog+tistory |
| `/blog-crosspost docker-basics` | `docker-basics` slug 포스트, velog+tistory |
| `/blog-crosspost velog` | 최신 포스트, velog 만 |
| `/blog-crosspost docker-basics velog` | `docker-basics` 포스트, velog 만 |

**대상 포스트 결정**:

- 포스트 토큰이 없으면 → `_posts/**/*.md` 를 모두 훑어 frontmatter `date` 가
  가장 최근인 1개를 자동 선택한다.
- 포스트 토큰이 있으면 → slug/경로로 매칭한다. 여러 파일에 매칭되면 후보 목록을
  보여주고 사용자에게 하나 고르게 한다. 매칭이 없으면 그 사실을 알리고 종료.
- 결정한 파일을 한 줄로 알린다: `대상: _posts/infra/2026-05-12-docker-basics.md`.

선택한 `.md` 파일을 Read 로 읽는다.

## 2. 환경 설정 로드

`.claude/crosspost.yml` 을 읽는다. 이 파일은 tistory 블로그 주소 등 환경값을 담는다.

```yaml
tistory_blog: ohg0219.tistory.com     # tistory 글쓰기 URL 조립에 사용
canonical_base: https://ohg0219.github.io
default_targets: [velog, tistory]
```

- 파일이 **없고** tistory 가 대상에 포함되면 → 사용자에게 tistory 블로그 주소를
  **한 번만** 묻는다("tistory 블로그 주소가 무엇인가요? 예: `ohg0219.tistory.com`").
  답을 받아 위 형식으로 `.claude/crosspost.yml` 을 생성한다.
- `canonical_base` 가 없으면 `https://ohg0219.github.io` 를 기본값으로 쓴다.
- velog 만 대상이면 이 파일 없이도 진행할 수 있다(velog 는 고정 URL `/write` 사용).

## 3. 마크다운 변환

읽은 `.md` 의 frontmatter 와 본문을 velog·tistory 용 마크다운으로 변환한다.
변환은 Claude 가 메모리에서 수행한다(별도 스크립트 없음). 공통 마크다운 1벌을
만들되, **tistory 는 인용(`>`) 처리가 달라** 3.9 의 tistory 전용 규칙을 반드시
함께 적용한다.

### 3.1 frontmatter 매핑

frontmatter(`---` ~ `---`)는 본문에서 떼어내고 다음과 같이 쓴다.

| 필드 | 변환 |
|---|---|
| `title` | 플랫폼 글 **제목** 입력란에 그대로(`[N편]` 접미사 유지). |
| `tags` | 플랫폼 **태그** 입력란에 하나씩 입력(5절·6절). |
| `image.path` | 본문 맨 위에 커버 이미지로 삽입: `![<image.alt>](<canonical_base><image.path>)`. velog·tistory 는 frontmatter image 를 렌더하지 않으므로 본문에 직접 넣어야 한다. |
| `image.alt` | 위 이미지의 alt 텍스트. |
| `description` | 플랫폼에 부제 필드가 없으므로 본문에 넣지 않는다(본문 도입부가 이미 그 역할). |
| `date`, `categories` | 플랫폼 입력에 쓰지 않는다(타임스탬프·카테고리는 플랫폼 자체 개념). |
| `mermaid` | 본문 mermaid 코드블록 처리(3.6)의 신호로만 쓴다. |
| `series` | 3.5 참고. |

### 3.2 이미지·내부 링크 절대경로화

본문(및 3.1 의 커버 이미지)에서 사이트 루트 상대경로를 절대 URL 로 바꾼다.

- 마크다운 이미지 `![alt](/assets/...)`, HTML `<img src="/assets/...">`,
  링크 `[txt](/posts/...)` 등 `/` 로 시작하는 경로를 `<canonical_base>` + 경로 로 치환.
- 이미 `http://` `https://` 로 시작하면 그대로 둔다.
- 대상 prefix: `/assets/`, `/posts/`, `/categories/`, `/tags/` 등 내부 경로 전부.
  외부 플랫폼에서도 클릭·표시되도록 절대 URL 이어야 한다.
- 치환한 이미지 개수를 센다(4절 요약에 보고).

### 3.3 Liquid 태그 제거

본문의 Jekyll Liquid 구문(`{% ... %}`, `{{ ... }}`)을 제거한다.

- `{% include series-nav.html %}` → 줄 전체 삭제(시리즈 안내는 3.5 가 대체).
- 그 외 Liquid 구문도 제거하되, 무엇을 지웠는지 4절 요약에 보고한다.
- **단, 코드펜스(```` ``` ````) 안의 `{% %}` 는 예시 코드일 수 있으므로 절대
  건드리지 않는다.** 펜스 내부/외부를 구분해서 처리한다.
- 제거한 Liquid 태그 개수를 센다.

### 3.4 Chirpy prompt 박스 변환

Chirpy 전용 prompt 박스는 표준 마크다운이 아니므로 일반 blockquote 로 바꾼다.
형태: blockquote 다음 줄에 `{: .prompt-* }` IAL(인라인 속성) 이 붙어 있다.

- `{: .prompt-tip }` / `.prompt-info` / `.prompt-warning` / `.prompt-danger`
  속성 줄을 제거한다.
- 의미를 살리기 위해 해당 blockquote 첫 줄 맨 앞에 라벨을 붙인다:
  - `.prompt-tip` → `> 💡 **TIP** `
  - `.prompt-info` → `> ℹ️ **참고** `
  - `.prompt-warning` → `> ⚠️ **주의** `
  - `.prompt-danger` → `> 🚨 **경고** `
- 결과는 velog·tistory 모두 정상 렌더되는 표준 GFM blockquote 다.
- 변환한 박스 개수를 센다.

### 3.5 시리즈 안내

`series` frontmatter 가 있으면, 3.3 에서 제거된 `series-nav` 자리를 대신할
한 줄 안내를 본문에 넣는다.

- `_data/series.yml` 을 읽어 `series.name` 키의 `title` 을 조회한다(없으면
  `series.name` slug 를 그대로 쓴다).
- canonical 안내(3.7) 바로 아래에 한 줄 삽입(velog 는 `>` 인용, tistory 는
  3.9 에 따라 `>` 없는 일반 문단):
  > `> 이 글은 '<시리즈 제목>' 시리즈의 <part>번째 글입니다. 전체 시리즈는 원문 블로그에서 볼 수 있습니다.`

### 3.6 mermaid 코드블록 처리

velog·tistory 모두 mermaid 다이어그램을 렌더하지 못한다.

- ` ```mermaid ` 펜스를 ` ``` ` (언어 태그 없음)로 강등한다. velog 가 `mermaid`
  토큰을 만나면 레이아웃이 깨질 수 있으므로 반드시 강등한다. 다이어그램 정의
  자체는 코드로 보존한다.
- 그 코드블록 바로 위에 안내 한 줄을 넣는다:
  > `> 아래 다이어그램은 [원문 글](<canonical URL>)에서 그림으로 볼 수 있습니다.`
- mermaid 블록이 1개 이상이면 4절 요약에서 **⚠️ 경고**로 명시한다 — "외부
  플랫폼에서는 다이어그램이 코드로만 보임. 더 보기 좋게 하려면 원문 다이어그램을
  캡처해 이미지로 교체 권장."

### 3.7 canonical 원문 안내

검색엔진 중복 콘텐츠(duplicate content)를 피하기 위해 원문 출처를 본문에 명시한다.
velog·tistory 모두 canonical 메타태그를 설정하는 UI 가 없어, 본문 텍스트가
사실상 유일한 수단이다.

- **canonical URL 조립**: `<canonical_base>/posts/<slug>/`
  - `<slug>` 는 대상 파일명 `YYYY-MM-DD-<slug>.md` 의 `<slug>` 부분.
    (frontmatter 에 `slug:` 필드가 따로 있으면 그 값을 우선한다.)
  - 예: `2026-05-12-docker-basics.md` → `https://ohg0219.github.io/posts/docker-basics/`
  - 근거: `_config.yml` 의 `permalink: /posts/:title/` 이며 Jekyll `:title` 은
    파일명 slug 를 쓴다.
- **본문 최상단**(커버 이미지 바로 아래, 리드 단락 위)에 blockquote 삽입:
  > `> 📌 이 글은 [원문 블로그](<canonical URL>)에 처음 게시되었습니다.`
- **본문 최하단**에 구분선과 함께 한 번 더:
  > `---`
  > `> 원문: <canonical URL>`
- 위 두 안내는 **velog 기준**이다. tistory 는 3.9 에 따라 `>` 를 떼고 일반
  문단으로 넣는다.

### 3.8 민감 정보 마스킹

외부 발행 직전 한 번 더 스캔한다. API 키·토큰·비밀번호·사내 서버 주소/IP·
개인 이메일 패턴이 발견되면 **변환을 멈추고** 사용자에게 위치를 알린다. 발행된
원문에 이미 노출돼 있다면 그 사실도 함께 알린다(원문 수정이 우선).

### 3.9 플랫폼별 차이

공통 마크다운 1벌을 만들되, 플랫폼별로 다음을 다르게 적용한다.

**velog** — 3.1~3.8 을 그대로 적용한다. velog 의 blockquote(`>`)는 담백한
세로줄이라 canonical 안내·prompt 박스·인용을 모두 `>` 형태로 둔다.

**tistory** — ⚠️ **모든 `>` blockquote 를 제거하고 일반 문단으로 바꾼다.**
tistory 의 blockquote 는 큰 따옴표(❝) 장식 디자인이라, 안내문이든 인용문이든
큰 따옴표 박스로 렌더된다 — GitHub Pages 의 담백한 인용과 전혀 다르다. 구체적으로:
- canonical 안내(3.7): `> 📌 이 글은...` → `📌 이 글은...`
- 하단 원문(3.7): `> 원문: ...` → `원문: ...`
- prompt 박스(3.4): `> 💡 **TIP** ...` → `💡 **TIP** ...` (라벨은 굵게 유지)
- 시리즈 안내(3.5)·mermaid 안내(3.6)·본문 중 일반 인용도 `>` 만 떼고
  텍스트·줄바꿈은 유지한다.
- 그 외(`**굵게**`·`_기울임_`·`##` 제목·`-` 리스트·`---` 구분선·이미지·링크)는
  tistory 마크다운에서 정상 동작하므로 그대로 둔다. mermaid 강등(3.6)도 동일.

tistory 도 velog 처럼 **마크다운 모드**로 입력한다(HTML 모드는 쓰지 않는다 —
HTML 모드 에디터는 자동화 입력이 누적돼 다루기 어렵다).

## 4. 사전 점검 및 확인

자동화를 시작하기 전에 변환 결과를 요약해서 보여주고 진행 동의를 받는다.

```
🔁 크로스포스팅 준비 완료
   대상 글: <title>
   원문 URL: <canonical URL>
   대상 플랫폼: velog, tistory
   태그: <tags>

   변환 처리:
     - 이미지·링크 절대경로화: N개
     - Liquid 태그 제거: N개
     - prompt 박스 변환: N개
     - mermaid 다이어그램: N개   ⚠️ 외부 플랫폼에서는 코드로만 표시됨
```

- mermaid 가 있으면 ⚠️ 줄을 반드시 포함한다.
- canonical URL 이 아직 살아있지 않을 수 있다. 가볍게 `browser_navigate` 로
  원문 URL 에 접속해 정상 페이지인지 확인한다. 404 면 "원문이 아직 배포 안 된 것
  같습니다. push·배포를 먼저 끝내는 게 좋습니다. 그래도 진행할까요?" 라고 묻는다.
- 사용자가 진행에 동의하면 5절·6절로 넘어간다(대상 플랫폼만).

## 5. velog 자동화

Playwright MCP 로 velog 글쓰기 화면에 입력한다. 아래 도구 이름은 일반 명칭이며
실제로는 환경의 Playwright MCP 도구(`browser_navigate`, `browser_snapshot`,
`browser_click`, `browser_type`, `browser_press_key`, `browser_take_screenshot`,
`browser_wait_for`, `browser_evaluate` 등)를 사용한다.

**공통 원칙 (5절·6절 모두 적용)**

- **snapshot 우선.** CSS 셀렉터를 하드코딩하지 않는다. 각 단계에서
  `browser_snapshot` 으로 접근성 트리를 받아 role·name·placeholder·라벨로 요소를
  식별하고 그 ref 로 조작한다.
- **단계마다 검증.** 입력 후 snapshot 또는 screenshot 으로 상태를 확인한 뒤
  다음 단계로 간다.
- **불확실하면 멈춘다.** 마크다운 모드 토글·임시저장 버튼 같은 핵심 요소를 못
  찾으면 추측해서 클릭하지 말고, `browser_take_screenshot` 으로 화면을 사용자에게
  보여주고 "이 화면에서 어디를 눌러야 하나요?"라고 묻는다.
- **🚫 공개 발행 금지.** velog '출간하기', tistory '공개 발행' 버튼은 어떤
  경우에도 클릭하지 않는다. 출간 모달이 뜨면 닫는다.

### 5.1 로그인 확인

⚠️ velog 는 **미로그인 상태에서도 `/write` 에 진입할 수 있고 제목·본문·태그
입력란도 모두 보인다.** 따라서 입력란 존재 여부로 로그인을 판정하면 안 된다
(미로그인 상태로 입력하면 '임시저장'을 눌러도 저장되지 않는다). 반드시 아래
방법으로 판정한다.

1. `browser_navigate` → `https://velog.io/`
2. `browser_evaluate` 로 헤더의 로그인 버튼 존재를 확인한다:
   ```js
   () => [...document.querySelectorAll('button')]
           .some(b => b.textContent.trim() === '로그인')
   ```
   - 결과가 `true` (헤더에 '로그인' 버튼 있음) → **미로그인**
   - 결과가 `false` (헤더에 사용자 썸네일 아바타가 대신 있음) → 로그인됨
3. 미로그인이면 헤더의 '로그인' 버튼을 클릭해 로그인 모달을 띄운 뒤, 사용자에게
   요청하고 **대기**한다:
   > "지금 열린 브라우저 창의 velog 로그인 모달에서 직접 로그인해 주세요
   > (이메일·GitHub·Google·Facebook 중 택1, 2FA·캡차 포함). 끝나면 '완료'라고
   > 알려주세요."
4. 사용자가 완료를 알리면 `https://velog.io/` 를 다시 열어 2번 방법으로
   재확인한다. 여전히 미로그인이면 한 번 더 안내하고, 반복 실패 시 velog 는
   건너뛰고 tistory 로 넘어간다(부분 성공 허용).
   - 로그인 자격증명을 Skill 이 입력하거나 저장하지 않는다. 사용자가 직접 한다.
   - 한 번 로그인하면 Playwright MCP 의 persistent 프로필에 세션이 남아 다음
     실행부터는 이 단계가 자동 통과된다.
5. 로그인이 확인되면 `browser_navigate` → `https://velog.io/write` 로 글쓰기
   화면을 깨끗하게 연다(미로그인 상태에서 입력했던 내용은 버리고 새로 입력).

### 5.2 글 입력

velog 글쓰기는 기본이 마크다운 에디터다(좌측 입력·우측 프리뷰). 모드 전환 불필요.

1. **제목** — 제목 입력란을 클릭해 포커스하고 `title` 을 입력한다.
2. **본문** — 좌측 본문 영역(CodeMirror)을 클릭해 포커스하고 변환된 마크다운
   전체를 입력한다.
   - 먼저 `browser_type` 으로 시도한다.
   - 본문이 길어 입력이 누락되거나 CodeMirror 상태가 어긋나면, 클립보드 경유로
     바꾼다: `browser_evaluate` 로 `navigator.clipboard.writeText(<본문>)` 실행
     → 본문 영역 포커스 → `browser_press_key` 로 `Control+V`.
   - 입력 후 우측 프리뷰가 정상 렌더되는지 snapshot/screenshot 으로 확인한다.
3. **태그** — 하단 태그 입력창("태그를 입력하세요")에 `tags` 를 하나씩 입력하고
   각 태그마다 `browser_press_key` 로 `Enter` 를 눌러 확정한다.

### 5.3 임시저장

1. `browser_snapshot` 으로 '임시저장' 버튼을 찾는다(텍스트에 "임시저장" 포함).
2. 그 버튼을 클릭한다.
3. 🚫 '출간하기' 버튼은 누르지 않는다. 출간 설정 모달이 떴다면 닫는다.
4. **저장 검증** — velog 임시저장 토스트는 매우 짧게 떴다 사라져 스크린샷으로
   잡기 어렵다. 대신 `browser_navigate` → `https://velog.io/saves` (임시 글
   목록)로 이동해, 방금 글의 제목이 목록 최상단에 "방금 전" 으로 있는지
   확인한다. 목록에 없으면 저장 실패다 — 5.1 의 로그인 상태부터 다시 점검한다.

velog 임시 글은 이후에도 `https://velog.io/saves` 에서 볼 수 있다.

## 6. tistory 자동화

`.claude/crosspost.yml` 의 `tistory_blog` 로 글쓰기 URL 을 만든다:
`https://<tistory_blog>/manage/newpost/`. 5절의 공통 원칙을 그대로 적용한다.

### 6.1 로그인 확인

⚠️ **tistory 세션은 velog 와 달리 잘 유지되지 않는다.** 실측 결과, 카카오
로그인은 Playwright(브라우저)를 재시작하면 '로그인 상태 유지'를 체크해도 풀린다.
따라서 tistory 는 `/blog-crosspost` 를 실행할 때마다 — 특히 브라우저를 새로
띄울 때 — 로그인이 필요할 가능성이 높다(velog 는 세션이 유지되므로 불필요).
다만 브라우저를 계속 켜 두면 같은 세션 동안에는 유지되므로, 하루에 여러 글을
올릴 때는 한 번만 로그인하면 된다.

1. `browser_navigate` → `https://<tistory_blog>/manage/newpost/`
2. 페이지 URL 을 확인한다. `tistory.com/auth/login` (또는 `kauth.kakao.com`)으로
   리다이렉트됐으면 **미로그인**이다. 글쓰기 화면이 그대로 떠 있으면(페이지
   제목이 "글쓰기") 로그인된 상태다.
3. 미로그인이면 사용자에게 요청하고 **대기**한다:
   > "지금 열린 브라우저 창의 tistory 탭에서 '카카오계정으로 로그인'을 눌러 직접
   > 로그인해 주세요(2FA·캡차 포함). 끝나면 '완료'라고 알려주세요."
   완료 후 글쓰기 URL 로 다시 진입해 재확인한다. 반복 실패 시 tistory 는
   건너뛴다(부분 성공 허용).
4. ⚠️ **반드시 새 글로 시작한다.** "이어서 작성하시겠습니까?" confirm 다이얼로그가
   뜨면 `browser_handle_dialog` 로 **거절(accept=false)**한다. tistory 에디터는
   자동화 입력이 기존 내용에 누적되는 특성이 있어, 빈 새 글에서 시작하지 않으면
   본문이 중복된다.

### 6.2 마크다운 모드 전환

1. `browser_snapshot` 으로 에디터 툴바 우측의 모드 버튼을 찾는다(기본값은
   "기본모드"). 그 버튼을 클릭하면 '기본모드 / 마크다운 / HTML' 메뉴가 열린다.
2. 메뉴에서 '마크다운'을 클릭한다.
3. "작성 모드를 변경하시겠습니까?" confirm 다이얼로그가 뜨면
   `browser_handle_dialog` 로 accept 한다(본문이 비어 있어 손실 없음).
4. ⚠️ tistory 는 마크다운으로 저장한 뒤 기본모드로 되돌리면 서식이 깨진다.
   그러므로 **본문 입력 전에 반드시 마크다운 모드로 먼저 전환**하고, 처음부터
   마크다운으로 한 번에 입력한다.
5. 모드 전환 UI 를 못 찾으면 5절 공통 원칙대로 멈추고 사용자에게 묻는다.

### 6.3 글 입력

1. **제목** — 제목 입력란("제목" placeholder)에 `title` 을 입력한다.
2. **본문** — 마크다운 입력 영역에 변환된 마크다운(3.9 의 tistory 규칙 적용본,
   `>` 인용 없음)을 입력한다. 6.1 에서 새 글로 시작했으므로 에디터는 비어 있다 —
   기존 내용이 남아 있으면 누적되므로 입력 전 비어 있는지 확인한다.
   길면 5.2 와 같이 클립보드 경유로 붙여넣는다.
3. **카테고리** — 자동으로 선택하지 않는다. 저장 후 사용자가 직접 지정하도록
   7절 안내에 적는다.
4. **태그** — 태그 입력창에 `tags` 를 하나씩 입력하고 각 태그마다 `Enter` 로
   확정한다.

### 6.4 비공개 저장

1. 발행 모달을 거치지 않는 단순 **'저장'**(임시저장)을 1순위로 한다.
   `browser_snapshot` 으로 '저장' 버튼을 찾아 클릭한다.
2. 불가피하게 발행 설정 모달이 떠야만 저장된다면, 모달에서 공개범위가
   **'비공개'**로 선택돼 있는지 snapshot 으로 확인한 뒤 그 상태로만 저장한다.
3. 🚫 공개범위가 '공개'인 상태로는 저장하지 않는다. '공개 발행' 버튼은 누르지
   않는다. 확신이 안 서면 멈추고 스크린샷을 사용자에게 보여준다.
4. 저장 후 `browser_take_screenshot` 으로 결과를 확인한다.

tistory 비공개/임시 글은 블로그 관리 > '글 관리'에서 볼 수 있다.

## 7. 마무리 안내

작업이 끝나면 결과를 한 번에 정리해 보여준다.

```
✅ 크로스포스팅 완료 (임시저장 상태)

   velog    — 임시저장됨. https://velog.io/write 의 '임시 글'에서 확인
   tistory  — 비공개 저장됨. 블로그 관리 > 글 관리에서 확인

다음 단계 (사용자가 직접):
  1) 각 플랫폼에서 변환 결과를 검토 (특히 코드블록·이미지·표 렌더링)
  2) tistory 는 카테고리를 지정
  3) 이상 없으면 각 플랫폼에서 발행 버튼을 직접 클릭
```

- 한쪽 플랫폼만 성공했으면 성공/실패를 구분해 정직하게 보고한다.
- mermaid 가 있던 글이면 "원문 다이어그램을 캡처해 이미지로 교체하는 것을
  권장합니다" 를 한 줄 덧붙인다.

## 작성 원칙

- 변환은 **본문 톤을 바꾸지 않는다.** 사실·코드·명령어·URL·로그는 원문 그대로
  두고, 3절에 정의된 구조 변환(경로·Liquid·prompt·mermaid·canonical)만 한다.
- 셀렉터를 하드코딩하지 않는다. 항상 snapshot 기반으로 요소를 찾고, 못 찾으면
  멈추고 사용자에게 묻는다. 플랫폼 UI 는 언제든 바뀔 수 있다.
- 🚫 **공개 발행은 절대 자동으로 하지 않는다.** 이 Skill 의 종착점은 임시저장이다.
- 인자 파싱 시 따옴표·공백을 보존한다.
- `.claude/crosspost.yml` 이 없으면 만들되, tistory 가 대상일 때만 필요하다.

## 부수 동작

- 크로스포스팅에 성공했고 `.claude/blog-session.md` 가 존재하면, 그 frontmatter 에
  `crossposted_at: <YYYY-MM-DD HH:MM>` 을 추가한다(같은 세션 재실행 추적용).
  시각은 시스템 현재 시각을 쓴다.
- 같은 글을 다시 크로스포스팅하려는 것으로 보이면(예: `crossposted_at` 이 이미
  있음) 사용자에게 "이 글은 이미 크로스포스팅한 기록이 있습니다. 다시 진행할까요?"
  를 한 번 확인한다. velog/tistory 에 중복 임시저장이 쌓일 수 있기 때문이다.
