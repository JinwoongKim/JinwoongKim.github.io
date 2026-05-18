# With-Claude 퍼블리시 가이드

Claude한테 이 파일을 던지고 "이 가이드 따라 퍼블리시해줘"라고 하면 됨.

---

## Claude가 할 일 (순서대로)

### 1단계: 현재 대화 내용 파악
지금까지 대화에서 **무엇을 정리할지** 스스로 판단한다. 사용자에게 묻지 말고 추론할 것.

- 주제가 명확하면 그대로
- 여러 주제 섞여 있으면 가장 비중 큰 것 하나
- 모호하면 그때만 한 줄 확인

### 2단계: 슬러그 + 파일명 자동 결정

**슬러그 규칙:**
- 소문자 + 하이픈 + 영문/숫자만
- 연도 포함하면 좋음
- 예: `portugal-2026`, `book-q1-2026`, `vllm-deep-dive`, `housing-loan-2026`

**파일명 규칙:**
- 한 주제의 첫 글이면 → `master.html`
- 기존 주제 추가면 → 내용에 맞는 영문명 (`packing.html`, `restaurants.html`, `comparison.html` 등)
- `archives/with-claude/{슬러그}/` 폴더 이미 있는지 확인 후 결정

**판단 후 사용자에게 한 줄 알림 (확인 안 받음, 그냥 통보):**
```
→ archives/with-claude/{슬러그}/{파일명}.html 로 퍼블리시 진행.
```

### 3단계: SKILL.md 워크플로우 따라 실행

`/Users/jinwoong-kim/Library/Mobile Documents/iCloud~md~obsidian/Documents/Blog/blog/blog/scripts/with-claude-publisher/SKILL.md` 의 워크플로우 그대로:

1. 폴더 없으면 생성 (`Filesystem:create_directory`)
2. `template.html` 베이스로 HTML 작성 (`Filesystem:write_file`)
3. 허브 `index.html`에 카드 추가 (`Filesystem:edit_file` 또는 `str_replace`)
4. 디자인 토큰(폰트, 색상) SKILL.md 규칙대로

### 4단계: 결과 통보

```
✅ 퍼블리시 완료.

1~2분 후 확인:
- https://jinwoongkim.net/archives/with-claude/{슬러그}/{파일명}.html
- 허브: https://jinwoongkim.net/archives/with-claude/
```

---

## 사용자 톤 / 스타일 유지

콘텐츠 작성 시 사용자 선호 반영:

- 한국어
- 냉정/냉철 (무조건 동조 금지)
- 출처 표기 필수
- 추천 시 한 줄 이유
- 3년 이내 정보 우선

---

## 금지사항

- ❌ "슬러그 뭐로 할까요?" 묻기 (스스로 정해라)
- ❌ "정말 이대로 퍼블리시할까요?" 확인 (그냥 해라)
- ❌ "어떤 파일명으로?" 묻기 (스스로 정해라)
- ❌ 호텔 예약번호, 결제 정보, 토큰 등 민감 정보 HTML에 포함
- ❌ 디자인 토큰 변경 (Fraunces/Inter 폰트 + SKILL.md 색상 변수 고정)

---

## 사용자가 명시하면 따를 것

진웅님이 같이 알려주면 그대로 사용:

- "포르투갈 자료에 추가해줘" → 슬러그 `portugal-2026` 고정
- "{XXX} 로 만들어줘" → 그 슬러그/파일명 사용
- "간단하게" → 짧고 핵심만
- "상세하게" → 풀버전 + 섹션 다수

---

## 트리거 발화 예시

진웅님이 보통 이렇게 말함:
- "이 가이드 따라 퍼블리시해줘"
- "지금까지 한 거 with-claude에 올려줘"
- "이거 정리해서 archives에 추가해줘"

모두 같은 작업으로 처리.
