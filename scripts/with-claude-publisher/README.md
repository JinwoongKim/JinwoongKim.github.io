# With-Claude Publisher

Claude와 정리한 자료를 `archives/with-claude/`에 통일된 디자인의 HTML로 게시하는 스킬.

## 사용법

데스크톱/웹 Claude에서 다음과 같이 말하면 발동:

- "퍼블리시해줘"
- "with-claude에 올려줘"
- "내 블로그에 정리해줘"
- "archives에 추가해줘"

발동 후 Claude가:
1. 주제 슬러그 확인 (예: `portugal-2026`)
2. 폴더 없으면 생성
3. 통일된 디자인 템플릿으로 HTML 작성
4. 허브 `index.html`에 카드 자동 추가

## 결과

작성 후 1~2분 내 (자동 sync):
- 페이지: https://jinwoongkim.net/archives/with-claude/{slug}/{filename}.html
- 허브: https://jinwoongkim.net/archives/with-claude/

## 파일 구조

```
scripts/with-claude-publisher/
├── SKILL.md          ← 스킬 정의 (Claude가 읽음)
├── template.html     ← HTML 템플릿 (디자인 기준)
└── README.md         ← 이 파일
```

## Anthropic 시스템에 등록

이 스킬을 Claude 컨텍스트에 자동 로드시키려면 `/mnt/skills/user/with-claude-publisher/`에도
복사 등록 필요. 등록 방법은 `obsidian-blog-publisher`와 동일 방식.

미등록 상태에서도 동작 가능 — 사용자가 매번 "scripts/with-claude-publisher/SKILL.md 참고해서
퍼블리시해줘"라고 지시하면 됨.

## 디자인 토큰

`template.html`의 `:root` 변수에서 색상 통일 관리. 페이지별 톤 변경은 `--accent` 변수만
바꾸면 됨:

| 톤 | --accent | 용도 |
|----|---------|-----|
| Terra (기본) | `var(--terra)` | 일반/감성 |
| Azul | `var(--azul)` | 정보/예약 |
| Olive | `var(--olive)` | 체크리스트/준비물 |
| Port | `var(--port)` | 와인/음식 |

폰트: `Fraunces` (제목) + `Inter` (본문) + `JetBrains Mono` (메타).
