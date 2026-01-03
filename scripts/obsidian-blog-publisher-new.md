---
name: obsidian-blog-publisher-new
description: Obsidian에서 Jekyll GitHub 블로그로 글을 게시하는 스킬. 사용자가 "게시해줘", "publish", "블로그에 올려줘" 등을 말하면 _posts_wip에서 published:true인 글을 찾아 이미지 경로를 변환하고 _posts로 이동. 블로그 관련 대화 시 지정된 경로에서 노트를 검색.
---

# Obsidian Blog Publisher

Obsidian 노트를 Jekyll GitHub 블로그로 게시하는 워크플로우.

## 블로그 경로

```
BLOG_PATH: /Users/jinwoong-kim/Library/Mobile Documents/iCloud~md~obsidian/Documents/Blog/blog/blog
IMAGE_SOURCE_PATH: /Users/jinwoong-kim/Library/Mobile Documents/iCloud~md~obsidian/Documents/Blog/blog
```

## 디렉토리 구조

| 경로 | 용도 |
|-----|------|
| `BLOG_PATH/_posts_wip/` | 작성 중인 글 |
| `BLOG_PATH/_posts/` | 게시된 글 |
| `BLOG_PATH/images/글폴더명/` | 게시된 글의 이미지 |
| `IMAGE_SOURCE_PATH` | Obsidian 붙여넣기 이미지 위치 |

## 게시 워크플로우 (Filesystem 도구 사용)

### Step 1: 게시 대상 글 찾기
**"게시해줘"라고 하면 `_posts_wip/`에서 `published: true`인 파일을 찾는다.**

```
1. Filesystem:list_directory - _posts_wip/ 파일 목록 조회
2. 각 파일의 front matter에서 published: true 확인
3. published: true인 파일만 게시 대상으로 선정
```

### Step 2: 글 읽고 이미지 추출
```
Filesystem:read_text_file - 글 내용 읽기
정규식: !\[.*?\]\(([^)]+\.(png|jpg|jpeg|gif|webp|heic))\)
또는: !\[\[([^\]]+\.(png|jpg|jpeg|gif|webp|heic))\]\]
```

### Step 3: 이미지 처리
각 이미지에 대해:
1. `Filesystem:search_files` - **IMAGE_SOURCE_PATH**에서 이미지 찾기
2. `Filesystem:create_directory` - `BLOG_PATH/images/글폴더명/` 생성
3. 이미지 이름 변경: `Pasted image YYYYMMDDHHMMSS.png` → `YYYYMMDDHHMMSS.png`
4. `Filesystem:move_file` - 이미지를 새 이름으로 폴더에 이동

### Step 4: 이미지 참조 변환 (두 줄 추가)
**원본 이미지 참조는 삭제하고, 아래 두 줄을 추가:**
```
변환 전: ![](Pasted%20image%2020260101120320.png) 또는 ![[Pasted image 20260101.png]]

변환 후 (두 줄 추가):
![](../images/YYYY-MM-DD-글제목/20260101120320.png)
![](/images/YYYY-MM-DD-글제목/20260101120320.png)
```

- 첫 번째 줄 (상대 경로): Obsidian에서 이미지 표시용
- 두 번째 줄 (절대 경로): Jekyll 블로그에서 이미지 표시용

**이미지 이름 규칙:**
- "Pasted image " 접두사 제거
- 날짜+시간 숫자만 남김 (예: `20260101120320.png`)

### Step 5: 파일명 날짜 처리
**파일명에 날짜가 없거나 불완전하면 오늘 날짜(YYYY-MM-DD)를 자동으로 붙인다.**

예시:
- `my-post.md` → `2026-01-03-my-post.md`
- `2024-01-my-post.md` → `2026-01-03-my-post.md` (오늘 날짜로 교체)
- `2024-01-15-my-post.md` → 그대로 유지 (이미 완전한 날짜가 있음)

### Step 6: 태그 자동 설정
**front matter의 tags가 비어있거나 부족하면, 글 내용을 기반으로 적절한 태그를 자동 생성한다.**

규칙:
- 최소 1개, 최대 3개
- 글의 주제, 카테고리, 핵심 키워드를 기반으로 선정
- 한글 또는 영어 모두 가능
- 기존 태그가 있으면 유지하되, 3개 초과 시 중요한 3개만 남김

### Step 7: 글 내용 수정 및 이동
1. 태그 정리 (Step 6 규칙 적용)
2. `Filesystem:write_file` - 수정된 내용으로 `_posts/`에 저장 (Step 5의 파일명 사용)
3. 원본 `_posts_wip/` 파일은 `_trash/`로 이동

## 이미지 폴더명 규칙

파일명에서 `.md` 제거:
- `2026-01-01-한해를-돌아보며.md` → `images/2026-01-01-한해를-돌아보며/`

## 블로그 탐색

블로그 관련 질문 시 Filesystem 도구로 BLOG_PATH 탐색.

## 요약

| 트리거 | 동작 |
|--------|------|
| "게시해줘" | `_posts_wip/`에서 `published: true`인 파일 찾아서 게시 |
| 파일명 날짜 없음 | 오늘 날짜(YYYY-MM-DD) 자동 추가 |
| 태그 부족 | 글 내용 기반 태그 1~3개 자동 생성 |
