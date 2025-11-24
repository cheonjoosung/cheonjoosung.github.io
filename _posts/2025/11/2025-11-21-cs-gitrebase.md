---
title: (CS/컴퓨터공학) Git rebase vs merge 비교
tags: [CS, GIT]
style: fill
color: dark
description: (CS/컴퓨터공학) Git rebase vs merge 비교 히스토리 관리 전략과 실무 팁 총정리
---

## ✨ 개요

팀 협업에서 히스토리를 **깨끗하게 유지할 것인가** vs **사실을 있는 그대로 남길 것인가**는 늘 고민입니다.  
`rebase`와 `merge`의 차이, 선택 기준, 충돌 처리, 브랜치 보호 전략을 한 번에 정리합니다

---

## 1. 한 장 요약

| 상황 | 추천 | 이유 |
|---|---|---|
| 기능 브랜치를 메인에 깔끔하게 합치고 싶다 | **Rebase + FF merge** | 선형(linear) 히스토리, 리뷰/탐색 용이 |
| 큰 기능(여러 커밋)을 하나로 묶고 싶다 | **Interactive Rebase**(squash/fixup) | 잡음 제거, 의미 단위 커밋 유지 |
| 다수의 기능 브랜치를 동시에 통합 | **Merge** | 브랜치 관계 보존, 충돌 지점 명확화 |
| 공개 원격 브랜치, 이미 공유된 커밋 | **Merge** | 리라이트 금지(참조자 다수) |
| 릴리즈 브랜치에서 hotfix를 메인/릴리즈 모두 반영 | **Merge** 또는 **Cherry-pick** | 추적성/릴리즈 정책에 따라 선택 |

> 원칙: **로컬/개인 작업은 rebase OK**, **공개/공유 히스토리는 merge**가 안전.

---

## 2 개념 차이

### Merge (사실 기록)
- 두 브랜치의 **공통 조상**을 기준으로 **새로운 merge 커밋**을 만든다.
- 히스토리에 **분기와 병합 흔적**이 남는다.

```text
A---B---C (main)
\
D---E (feature)

M (merge commit)
```

### Rebase (히스토리 재작성)

- `feature`의 커밋 **기반(base)를 옮겨** 마치 **처음부터 main 위에서 작업한 것**처럼 만든다.
- 커밋 해시가 **새로 생성**된다(리라이트).

```text
A---B---C (main)

D'---E' (feature rebased)
```

---

## 3 핵심 장단점 비교

| 항목 | Rebase | Merge |
|---|---|---|
| 히스토리 | **선형**(읽기 쉬움) | **분기/병합 흔적 보존**(사실성↑) |
| 충돌 | 커밋 단위로 나눠 처리(세밀) | 한 번에 처리(맥락 넓음) |
| 커밋 해시 | **변경**(리라이트) | **변경 없음** |
| 협업 안전성 | **공유 브랜치에 금지** | 안전 |
| Bisect/추적 | 커밋 정제 유리 | 타임라인 보존 유리 |
| CI 캐시/빌드 | 변경 범위 작게 유지 가능 | merge 시 대규모 변경 감지 유리 |

---

## 4. 기본 워크플로우

### 4.1 기능 브랜치 선형화 (로컬)

```bash
# 최신 main 반영
git checkout feature/login
git fetch origin
git rebase origin/main   # 로컬에서만!

# 충돌 시
# 1) 파일 수정 → git add .
# 2) git rebase --continue
# 3) 포기: git rebase --abort

# 정리 후 푸시 (주의: 강제 푸시 필요)
git push -f origin feature/login
```

### 4.2 깔끔한 병합(Fast-forward 우선)

```bash
git checkout main
git fetch origin
git merge --ff-only feature/login   # 선형 유지 실패 시 에러
# 실패하면: git merge --no-ff feature/login  # merge 커밋을 명시적으로 남김
git push origin main
```

### 4.3 인터랙티브 리베이스로 커밋 정리

```bash
git rebase -i HEAD~6
# pick/squash/fixup/reword 선택
# fixup/squash 커밋은 --autosquash로 자동 정렬 가능
git rebase -i --autosquash HEAD~6
```

---

## 5. 충돌 처리 전략

- Rebase 충돌: 커밋마다 작은 단위로 해결 → 원인 커밋을 정확히 파악 가능
- Merge 충돌: 한 번에 해결하되, 충돌 파일이 많다면 기능 브랜치에서 rebase 후 다시 merge 시도
> 다툴만한 파일(예: 공용 상수/버전 파일)은 코드 오너/리뷰 규칙으로 선제 통제.

---

## 6. 보호 규칙(Branch Protection)과 정책

- 공유 브랜치(release/main)
  + force-push 금지(리라이트 방지)
  + 필수 리뷰/CI 통과 요구
  + `--ff-only` 머지 또는 `--no-ff` 규칙 팀 합의
- 기능 브랜치
  + 로컬에서 수시 rebase OK
  + PR 전 rebase origin/main + squash로 커밋 정리 권장

---

## 7. 실무 팁 모음

- 자동 스쿼시 태그: 커밋 메시지에 fixup! 대상커밋제목 → git rebase -i --autosquash로 자동 합치기
- 사고 복구: 리라이트 전 git reflog 스냅샷 확인 → 언제든 복원 가능
- 긴 살아있는 브랜치: 주기적으로 rebase main 하여 충돌을 빚처럼 쌓지 말기
- 릴리즈 노트: 머지 커밋 메시지에 ChangeLog 요약 → 추적성↑
- 모노레포/대규모 CI: 선형 히스토리(캐시 적중↑), 단 큰 기능은 merge 커밋으로 의미 덩어리 남기기

---

## 8. 안티패턴 방지

- 공유 브랜치 rebase: 절대 금지(히스토리 파괴, 다른 사람 작업 깨짐)
- 무분별한 squash: 디버깅에 필요한 단위까지 모두 합치면 원인 추적 어려움
- 머지 스팸: 소규모 변경마다 Merge branch 'main' 남기지 말고 rebase로 정리
- 무설명 merge: PR 머지 시 --no-ff를 쓰면 왜 합쳤는지 메시지로 근거 남기기

---

## 결론

- rebase는 읽기 쉬운 선형 히스토리를 만든다. 단, 공유 브랜치에선 금지.
- merge는 사실 관계를 보존하고 안전하다. 큰 단위 이벤트를 기록하기 좋다.
- 팀 규칙은 브랜치 보호 + 머지 정책 + 커밋 정리 규칙 세 트라이앵글로 문서화하라. 그렇게 하면 히스토리는 “아름답고 추적 가능한 아카이브”가 된다.