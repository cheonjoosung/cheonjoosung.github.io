---
title: Markdown(.md) 완전 정리 - 기초 문법부터 실전 활용까지
tags: [ Etc ]
style: fill
color: dark
description: Markdown 파일의 기초 문법(제목, 목록, 표, 코드블록, 링크, 이미지)부터 GitHub, Jekyll 블로그 등 실전에서 자주 쓰는 패턴까지 한눈에 정리합니다.
---

## 개요

- Markdown은 일반 텍스트로 서식이 있는 문서를 작성할 수 있는 경량 마크업 언어입니다.
- `.md` 또는 `.markdown` 확장자를 사용합니다.
- GitHub README, 기술 블로그, 노션, 슬랙, Jira 등 개발 환경 곳곳에서 사용됩니다.
- 이 글에서는 기초 문법부터 실전 활용 패턴까지 전부 정리합니다.

---

## 1. 제목 (Heading)

`#` 개수로 제목 레벨을 표현합니다. (H1 ~ H6)

```markdown
# H1 제목
## H2 제목
### H3 제목
#### H4 제목
##### H5 제목
###### H6 제목
```

**렌더링 결과**

# H1 제목
## H2 제목
### H3 제목

> `#` 뒤에 반드시 공백을 하나 넣어야 합니다.

---

## 2. 강조 (Emphasis)

```markdown
**굵게 (Bold)**
*기울이기 (Italic)*
***굵게 + 기울이기***
~~취소선~~
```

**렌더링 결과**

**굵게 (Bold)**  
*기울이기 (Italic)*  
***굵게 + 기울이기***  
~~취소선~~

---

## 3. 목록 (List)

### 순서 없는 목록

```markdown
- 항목 1
- 항목 2
  - 하위 항목 2-1
  - 하위 항목 2-2
- 항목 3
```

**렌더링 결과**

- 항목 1
- 항목 2
  - 하위 항목 2-1
  - 하위 항목 2-2
- 항목 3

> `-`, `*`, `+` 모두 사용 가능합니다. 하위 항목은 스페이스 2칸 또는 탭으로 들여쓰기합니다.

### 순서 있는 목록

```markdown
1. 첫 번째
2. 두 번째
3. 세 번째
```

**렌더링 결과**

1. 첫 번째
2. 두 번째
3. 세 번째

> 숫자가 순서에 맞지 않아도 자동으로 올바른 번호가 매겨집니다.

### 체크리스트

```markdown
- [x] 완료된 항목
- [ ] 미완료 항목
- [x] 또 다른 완료 항목
```

**렌더링 결과**

- [x] 완료된 항목
- [ ] 미완료 항목
- [x] 또 다른 완료 항목

---

## 4. 링크 (Link)

```markdown
[링크 텍스트](URL)
[링크 텍스트](URL "툴팁 텍스트")

<!-- 참조 방식 -->
[링크 텍스트][id]
[id]: https://example.com "툴팁"

<!-- 자동 링크 -->
<https://example.com>
<email@example.com>
```

**예시**

```markdown
[Google](https://www.google.com)
[Google](https://www.google.com "구글로 이동")
<https://www.google.com>
```

---

## 5. 이미지 (Image)

```markdown
![대체 텍스트](이미지_URL)
![대체 텍스트](이미지_URL "툴팁")

<!-- 참조 방식 -->
![대체 텍스트][img-id]
[img-id]: /path/to/image.png "툴팁"
```

**예시**

```markdown
![프로필 이미지](https://example.com/profile.png)
![로고](/assets/logo.png "회사 로고")
```

> 링크와 문법이 동일하고 앞에 `!`만 붙입니다.

---

## 6. 코드 (Code)

### 인라인 코드

```markdown
`인라인 코드`는 백틱(`) 하나로 감쌉니다.
```

`val name = "Kotlin"` 이렇게 표시됩니다.

### 코드 블록

백틱 3개(` ``` `)로 감싸고 언어명을 지정하면 문법 강조가 적용됩니다.

````markdown
```kotlin
fun main() {
    println("Hello, World!")
}
```
````

```kotlin
fun main() {
    println("Hello, World!")
}
```

**지원 언어 예시**

| 언어 | 키워드 |
|------|--------|
| Kotlin | `kotlin` |
| Java | `java` |
| Swift | `swift` |
| Python | `python` |
| JavaScript | `javascript` / `js` |
| TypeScript | `typescript` / `ts` |
| Bash / Shell | `bash` / `shell` |
| XML | `xml` |
| JSON | `json` |
| Markdown | `markdown` |
| SQL | `sql` |

---

## 7. 인용문 (Blockquote)

```markdown
> 인용문입니다.
> 여러 줄도 가능합니다.

> 1단계 인용
>> 2단계 인용
>>> 3단계 인용
```

**렌더링 결과**

> 인용문입니다.
> 여러 줄도 가능합니다.

> 1단계 인용
>> 2단계 인용
>>> 3단계 인용

---

## 8. 구분선 (Horizontal Rule)

```markdown
---
***
___
```

세 가지 모두 아래처럼 렌더링됩니다.

---

## 9. 표 (Table)

```markdown
| 헤더1 | 헤더2 | 헤더3 |
|-------|-------|-------|
| 내용1 | 내용2 | 내용3 |
| 내용4 | 내용5 | 내용6 |
```

**렌더링 결과**

| 헤더1 | 헤더2 | 헤더3 |
|-------|-------|-------|
| 내용1 | 내용2 | 내용3 |
| 내용4 | 내용5 | 내용6 |

### 정렬 옵션

```markdown
| 좌측 정렬 | 가운데 정렬 | 우측 정렬 |
|:----------|:-----------:|----------:|
| Left      |   Center    |     Right |
| 왼쪽      |    가운데   |    오른쪽 |
```

**렌더링 결과**

| 좌측 정렬 | 가운데 정렬 | 우측 정렬 |
|:----------|:-----------:|----------:|
| Left      |   Center    |     Right |
| 왼쪽      |    가운데   |    오른쪽 |

> `:---` 좌측, `:---:` 가운데, `---:` 우측

---

## 10. 줄바꿈 & 단락

```markdown
줄 끝에 스페이스 2칸을 넣으면  
이렇게 줄바꿈이 됩니다.

빈 줄을 하나 넣으면 새로운 단락이 시작됩니다.
```

> 단순히 Enter 한 번만 누르면 줄바꿈이 되지 않습니다. 스페이스 2칸 또는 빈 줄을 사용하세요.

---

## 11. 이스케이프 (Escape)

Markdown 특수 문자를 그대로 표시하고 싶을 때 `\`를 앞에 붙입니다.

```markdown
\# 해시를 그대로 표시
\* 별표를 그대로 표시
\` 백틱을 그대로 표시
\[ 대괄호를 그대로 표시
```

이스케이프 가능한 문자:

```
\  `  *  _  {}  []  ()  #  +  -  .  !
```

---

## 12. HTML 직접 사용

Markdown 안에 HTML을 직접 쓸 수 있습니다.

```markdown
<br> <!-- 줄바꿈 -->
<b>굵게</b>
<i>기울이기</i>
<u>밑줄</u>
<mark>형광펜</mark>

<!-- 이미지 크기 조절 -->
<img src="image.png" width="200" height="100">

<!-- 접기/펼치기 -->
<details>
  <summary>클릭해서 펼치기</summary>
  숨겨진 내용입니다.
</details>
```

**렌더링 결과**

<details>
  <summary>클릭해서 펼치기</summary>
  숨겨진 내용입니다.
</details>

---

## 13. GitHub 전용 문법 (GFM)

GitHub Flavored Markdown(GFM)은 표준 Markdown에 기능을 추가합니다.

### 멘션 & 이슈 참조

```markdown
@username          <!-- 유저 멘션 -->
#123               <!-- 이슈/PR 번호 참조 -->
owner/repo#123     <!-- 다른 저장소 이슈 참조 -->
SHA: a5c3785       <!-- 커밋 해시 참조 -->
```

### 이모지

```markdown
:smile: :heart: :rocket: :white_check_mark: :warning:
```

:smile: :heart: :rocket:

---

## 14. 각주 (Footnote)

일부 환경(Jekyll, Notion 등)에서 지원됩니다.

```markdown
본문에 각주를 달고 싶을 때[^1] 이렇게 씁니다.
또 다른 각주[^2]도 가능합니다.

[^1]: 첫 번째 각주 내용입니다.
[^2]: 두 번째 각주 내용입니다.
```

---

## 15. 실전 패턴 — README.md 구성 예시

```markdown
# 프로젝트 이름

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Kotlin](https://img.shields.io/badge/kotlin-2.0-orange.svg)](https://kotlinlang.org)

> 한 줄 프로젝트 소개

## 📌 주요 기능

- 기능 1
- 기능 2
- 기능 3

## 🛠 기술 스택

| 분류 | 기술 |
|------|------|
| 언어 | Kotlin |
| UI | Jetpack Compose |
| 네트워크 | Retrofit2 + OkHttp |
| DI | Hilt |

## 🚀 시작하기

### 요구사항

- Android Studio Hedgehog 이상
- JDK 17

### 설치

```bash
git clone https://github.com/username/project.git
cd project
```

## 📁 프로젝트 구조

```
project/
├── app/
│   ├── src/
│   │   ├── main/
│   │   └── test/
├── build.gradle.kts
└── README.md
```

## 📄 라이선스

MIT License © 2026 [Your Name](https://github.com/username)
```

---

## 16. 자주 쓰는 문법 요약표

| 문법 | 마크다운 | 결과 |
|------|----------|------|
| 제목 | `# 제목` | H1 제목 |
| 굵게 | `**텍스트**` | **텍스트** |
| 기울이기 | `*텍스트*` | *텍스트* |
| 취소선 | `~~텍스트~~` | ~~텍스트~~ |
| 인라인 코드 | `` `코드` `` | `코드` |
| 링크 | `[텍스트](URL)` | 링크 |
| 이미지 | `![대체](URL)` | 이미지 |
| 인용문 | `> 텍스트` | 인용 |
| 구분선 | `---` | 수평선 |
| 순서 없는 목록 | `- 항목` | • 항목 |
| 순서 있는 목록 | `1. 항목` | 1. 항목 |
| 체크리스트 | `- [x] 항목` | ✅ 항목 |
| 줄바꿈 | 스페이스 2칸 + Enter | 줄바꿈 |

---

## 17. 에디터 추천

| 에디터 | 특징 |
|--------|------|
| **VS Code** | 미리보기 내장, 확장 플러그인 풍부 |
| **Typora** | WYSIWYG 방식, 문법 직접 확인 |
| **Obsidian** | 지식 관리, 링크 기반 노트 |
| **Notion** | 웹/앱 지원, 협업 최적화 |
| **IntelliJ / Android Studio** | `.md` 미리보기 내장 |

VS Code에서 미리보기 단축키:
- macOS: `Cmd + Shift + V`
- Windows: `Ctrl + Shift + V`

---

## 참고

- [CommonMark Spec](https://spec.commonmark.org/)
- [GitHub Flavored Markdown](https://github.github.com/gfm/)
- [Markdown Guide](https://www.markdownguide.org/)
