---
title: (CS/컴퓨터공학) git fetch vs git pull 차이 — 완벽 정리
tags: [CS, GIT]
style: fill
color: dark
description: (CS/컴퓨터공학) git fetch vs git pull 차이 — 완벽 정리
---

## ✨ 개요

- git fetch: 원격 저장소의 최신 이력을 다운로드만 함. 로컬 브랜치는 변경되지 않음.
- git pull: fetch + merge 또는 fetch + rebase를 한 번에 수행하며 로컬 브랜치를 즉시 업데이트함.
- 충돌 위험을 줄이고 안전하게 확인하려면 fetch,
- 빠르게 최신 상태를 맞추려면 pull.

---

## 1. git fetch

`git fetch`는 원격 저장소의 최신 커밋/브랜치 정보를 가져오지만, 로컬 브랜치를 변경하지 않는 명령어.

- 로컬 작업 내용에 영향 없음
- 원격 브랜치(origin/*)만 갱신
- 충돌 없음
- 안전함
- 이후 merge, rebase 여부를 직접 선택 가능

```bash
git fetch
git log origin/main
```

---

## 2. git pull이란?

`git pull`은 fetch 이후 자동으로 merge 또는 rebase까지 수행하는 명령어.

- 로컬 브랜치를 즉시 최신 상태로 업데이트
- merge/rebase 자동 실행
- 충돌 발생 가능성 존재
- 빠른 동기화에 유용

---

## 3. git fetch vs git pull 비교

| 구분    | git fetch       | git pull         |
| ----- | --------------- | ---------------- |
| 목적    | 원격 이력만 다운로드     | 다운로드 + 로컬 브랜치 갱신 |
| 로컬 영향 | 없음              | 있음               |
| 충돌 발생 | 없음              | 가능               |
| 안전성   | 매우 안전           | 상대적으로 위험         |
| 작업 방식 | 수동 merge/rebase | 자동 merge/rebase  |
| 사용 사례 | 안전 확인용          | 빠른 최신화           |

---

## 4. 언제 어떤 명령을 쓸까?

- git fetch를 사용할 때
  + 충돌이 걱정될 때
  + 팀원의 변경 사항을 먼저 확인하고 싶을 때
  + 히스토리를 보고 직접 merge/rebase 결정하고 싶을 때
  + 안전하게 작업하고 싶을 때
- git pull을 사용할 때
  + 최신 상태만 빠르게 반영하고 싶을 때
  + 충돌 가능성이 적은 상황
  + 자동화된 환경(CI 등)에서 최신 코드가 필요할 때

---

## 5. 실무 팁: pull을 rebase로 설정하기

- 불필요한 merge 커밋을 줄이기 위해 많이 사용하는 설정

```bash
git config --global pull.rebase true
```

- 이후 `git pull`은 자동으로

```bash
git fetch
git rebase
```

---

## 결론

- git fetch는 확인 + 안전
- git pull은 즉시 동기화
- 협업 환경에서는 fetch 후 merge/rebase 방식이 더 안전한 경우가 많음