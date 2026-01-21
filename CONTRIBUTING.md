# Contributing to Warpspace Tech Blog

PR을 통해 블로그에 포스팅하는 방법을 안내합니다.

## 1. 저장소 Fork 및 Clone

```bash
# 1. GitHub에서 저장소 Fork
# https://github.com/warpspaceinc/blog 에서 Fork 버튼 클릭

# 2. Fork한 저장소 Clone (submodule 포함)
git clone --recursive https://github.com/<your-username>/blog.git
cd blog
```

## 2. 새 브랜치 생성

```bash
git checkout -b post/<post-slug>
# 예: git checkout -b post/kubernetes-deployment-guide
```

## 3. 포스트 파일 작성

`content/posts/` 폴더에 마크다운 파일을 생성합니다.

### 파일 명명 규칙

`<slug>.<lang>.md` 형식으로 작성합니다:

- 영어: `my-post.en.md`
- 한국어: `my-post.ko.md`
- 일본어: `my-post.ja.md`

### Front Matter 템플릿

```markdown
---
title: "포스트 제목"
date: 2026-01-21
draft: false
tags: ["tag1", "tag2"]
categories: ["Category"]
summary: "포스트 요약 (목록에 표시됨)"
author: "작성자 이름"
---

본문 내용...
```

### Front Matter 필드 설명

| 필드 | 필수 | 설명 |
|------|------|------|
| `title` | O | 포스트 제목 |
| `date` | O | 발행 날짜 (YYYY-MM-DD) |
| `draft` | O | `false`로 설정해야 발행됨 |
| `tags` | - | 태그 목록 |
| `categories` | - | 카테고리 |
| `summary` | - | 포스트 목록에 표시될 요약 |
| `author` | - | 작성자 이름 |

## 4. 이미지 추가

이미지는 `static/images/posts/<slug>/` 폴더에 저장합니다.

```bash
# 폴더 생성
mkdir -p static/images/posts/my-post

# 이미지 파일 복사
cp ~/my-image.png static/images/posts/my-post/
```

마크다운에서 참조:

```markdown
![이미지 설명](/blog/images/posts/my-post/my-image.png)
```

## 5. 로컬에서 미리보기

```bash
# Hugo 설치 필요 (https://gohugo.io/installation/)
hugo server -D

# 브라우저에서 확인
# http://localhost:1313/blog/
```

## 6. 커밋 및 푸시

```bash
git add content/posts/<your-post>.*
git add static/images/posts/<your-post>/  # 이미지가 있는 경우
git commit -m "Add post: <post-title>"
git push origin post/<post-slug>
```

## 7. Pull Request 생성

GitHub에서 PR 생성 시 다음 내용을 포함해주세요:

- 포스트 제목
- 간단한 설명
- 지원 언어 (en/ko/ja)

## 포스트 작성 팁

### 코드 블록

언어를 지정하면 구문 강조가 적용됩니다:

````markdown
```python
def hello():
    print("Hello, World!")
```
````

### 다국어 지원

같은 slug로 여러 언어 파일을 만들면 자동으로 언어 전환이 지원됩니다:

```
content/posts/
├── my-post.en.md  # 영어
├── my-post.ko.md  # 한국어
└── my-post.ja.md  # 일본어
```

### 목차 (Table of Contents)

블로그 설정에서 `ShowToc: true`가 활성화되어 있어 `##`, `###` 헤딩을 사용하면 자동으로 목차가 생성됩니다.

## PR 전 체크리스트

- [ ] `draft: false` 설정 확인
- [ ] `date` 필드가 올바른 날짜인지 확인
- [ ] `tags`와 `categories` 설정
- [ ] 다국어 지원 시 모든 언어 파일 작성
- [ ] 로컬에서 `hugo server`로 미리보기 확인
- [ ] 이미지가 올바르게 표시되는지 확인
