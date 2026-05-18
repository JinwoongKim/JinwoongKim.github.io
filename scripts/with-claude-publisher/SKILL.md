---
name: with-claude-publisher
description: Claude와 정리한 자료를 진웅님 블로그의 archives/with-claude/ 아래에 HTML로 퍼블리시하는 스킬. 사용자가 "퍼블리시해줘", "with-claude에 올려줘", "아카이브에 정리해줘", "publish" 등을 말하면 발동. 통일된 디자인 템플릿(Fraunces/Inter 폰트, 아줄레주 블루 + 테라코타 색상)으로 HTML 작성하고 허브 index.html에 카드 자동 추가. 결과는 https://jinwoongkim.net/archives/with-claude/ 에 노출.
---

# With-Claude Publisher

Claude와의 대화에서 정리한 자료를 진웅님(jinwoongkim.net)의 GitHub Pages 블로그 아래
`archives/with-claude/` 섹션에 통일된 디자인의 HTML로 게시하는 워크플로우.

## 핵심 경로

```
BLOG_PATH:        /Users/jinwoong-kim/Library/Mobile Documents/iCloud~md~obsidian/Documents/Blog/blog/blog
ARCHIVES_PATH:    BLOG_PATH/archives/with-claude
HUB_INDEX:        ARCHIVES_PATH/index.html
TEMPLATE_PATH:    BLOG_PATH/scripts/with-claude-publisher/template.html
LIVE_URL_BASE:    https://jinwoongkim.net/archives/with-claude/
```

## 트리거 문구 (한국어/영어)

- "퍼블리시해줘", "publish해줘"
- "with-claude에 올려줘", "아카이브에 올려줘"
- "내 블로그에 정리해줘"
- "archives에 추가해줘"

## 디렉토리 구조 규칙

```
archives/with-claude/
├── index.html                 ← 허브 페이지 (수정 시 카드 추가)
└── {topic-slug}/              ← 주제별 폴더 (소문자-하이픈)
    ├── master.html            ← 메인 문서 (대표)
    └── {detail-page}.html     ← 상세 페이지들
```

**주제 슬러그 규칙:**
- 소문자, 하이픈 구분, 영문/숫자 (한글 슬러그는 URL에서 깨질 수 있어 피함)
- 연도/시즌 포함하면 좋음: `portugal-2026`, `book-2026-h1`, `team-offsite-2026q2`
- 사용자에게 슬러그 확인받기 권장

## 퍼블리시 워크플로우

### Step 1: 사용자 입력 파악

질문하지 않고도 추론 가능하면 바로 진행. 모호하면 `ask_user_input_v0`로 확인:
- **주제 슬러그**: 새 주제인지, 기존 폴더 추가인지
- **파일명**: `master.html`인지 추가 상세 페이지인지
- **콘텐츠**: 어떤 내용을 담을지

### Step 2: 폴더 / 파일 확인

```
Filesystem:list_directory - ARCHIVES_PATH/{topic-slug}/ 존재 확인
없으면: Filesystem:create_directory
있으면: 파일명 충돌 확인 후 덮어쓸지 사용자 확인
```

### Step 3: HTML 작성

**디자인 규칙 (반드시 준수):**
- 폰트: `Fraunces` (제목, serif) + `Inter` (본문) + `JetBrains Mono` (코드/메타)
- 색상 변수 (CSS `:root`):
  ```css
  --bg: #f7f1e3; --paper: #fffaf0; --paper-2: #fbf6ea;
  --ink: #1d1d1b; --ink-soft: #4a4742; --ink-faint: #6f6a62;
  --rule: #d9cfb8; --rule-strong: #b9ad90;
  --azul: #1f3a5f; --azul-deep: #122644; --azul-soft: #4a6b8a;
  --terra: #c0552d; --terra-light: #e89968;
  --olive: #7a8b5c; --gold: #b8893d;
  --critical: #a52a2a; --warn-bg: #fef3e3;
  ```
- 상단에 home-link: `<a href="../index.html" class="home-link">← with-claude 허브로</a>`
- breadcrumb 포함
- `h1`에 `<em>` 부분 강조 (이탤릭 + 색상)
- `eyebrow` 클래스로 카테고리 표시 (JetBrains Mono, letter-spacing 4px)
- 푸터에 `✦` ornament + 작성일

**템플릿 활용:**
- `TEMPLATE_PATH`의 `template.html`을 베이스로 시작
- 콘텐츠만 `<section>` 내부에 교체

**파일 작성:**
```
Filesystem:write_file - ARCHIVES_PATH/{topic-slug}/{filename}.html
```

### Step 4: 허브 index.html 업데이트

새 주제이거나 첫 파일이면 허브에 카드 추가:

```
Filesystem:read_file - HUB_INDEX
```

추가할 카드 (해당 `<section>`의 `.collection` 안에):
```html
<a href="{topic-slug}/{filename}.html" class="item">
  <div class="item-header">
    <span class="item-title">{문서 제목}</span>
    <span class="item-meta">{DETAIL or MAIN}</span>
  </div>
  <p class="item-desc">{1-2줄 설명}</p>
  <div class="item-tags">
    <span class="item-tag">{TAG1}</span>
    <span class="item-tag">{TAG2}</span>
  </div>
</a>
```

기존 주제에 추가하는 경우엔 해당 주제 섹션 안의 `.collection`에만 카드 추가.

새 주제 섹션을 만들어야 하면 다음 형식:
```html
<section>
  <h2><span class="num">0X /</span> {주제 이름}</h2>
  <p class="section-desc">{주제 한 줄 설명}</p>
  <div class="collection">
    {카드들}
  </div>
</section>
```

LAST UPDATED 날짜와 COLLECTIONS 수도 업데이트:
```html
<div class="meta">
  LAST UPDATED · {YYYY.MM.DD} · COLLECTIONS · {N}
</div>
```

```
Filesystem:str_replace - HUB_INDEX 수정
```

### Step 5: 사용자에게 알림

작성 완료 후 다음 메시지 형식으로 알림:

```
✅ 퍼블리시 완료.

자동 sync로 1~2분 후 확인:
- {LIVE_URL_BASE}{topic-slug}/{filename}.html
- 허브: {LIVE_URL_BASE}

새 카드는 허브의 "{주제 이름}" 섹션에 추가됨.
```

## 디자인 일관성 체크리스트

작성 후 자체 검증:
- [ ] 폰트 3종 모두 Google Fonts에서 link
- [ ] CSS `:root` 색상 변수 정의됨
- [ ] home-link 상단에 있음
- [ ] breadcrumb 있음
- [ ] h1에 `<em>` 강조 부분 있음
- [ ] eyebrow 클래스로 카테고리 표시
- [ ] 푸터에 ornament + 작성일
- [ ] 모바일 미디어 쿼리 (`@media (max-width: 700px)`) 있음
- [ ] 외부 리소스(폰트) 외 dependency 없음

## 콘텐츠 유형별 가이드

**Type 1: Master 문서** (한 주제의 종합 정리)
- 파일명: `master.html`
- 섹션 다수 + sticky TOC
- 허브에서 `MAIN` 메타 태그

**Type 2: Detail 페이지** (특정 측면 깊이 다룸)
- 파일명: 영문 슬러그 (예: `packing.html`, `itinerary-plans.html`)
- master에서 링크 걸어둠
- 허브에서 `DETAIL` 메타 태그

**Type 3: 비교/분석** (옵션 여러 개 비교)
- 비교 테이블 활용 (`.compare-table`)
- 추천 한 줄 명시 (`callout.recommend`)

## 사용자 선호 반영

진웅님은:
- 한국어로 답변
- 냉정/냉철한 의견 (무조건 동조 금지)
- 출처 표기
- 추천 시 한 줄 이유
- 3년 이내 정보 우선, 그 이상은 명시

스킬 발동 시에도 이 톤 유지. HTML 콘텐츠 안 본문에서도 마찬가지.

## 주의사항

- **민감 정보 금지**: 호텔 예약번호, 결제 정보, 토큰, 비밀번호 등은 HTML에 절대 넣지 않음
- **저작권**: 외부 콘텐츠 인용 시 짧게 + 출처 표기
- **자동 sync**: 파일 작성하면 iCloud → GitHub 자동 push되므로 즉시 라이브
- **덮어쓰기 확인**: 같은 경로에 파일 있으면 사용자에게 덮어쓸지 물어보기

## 디버깅

문제 발생 시:
1. Filesystem MCP 살아있는지 확인 (`list_allowed_directories`)
2. ARCHIVES_PATH 존재 확인
3. 작성 후 GitHub Actions 빌드 1~2분 대기
4. 404 뜨면 자동 sync 지연일 가능성 → 사용자에게 직접 push 안내
