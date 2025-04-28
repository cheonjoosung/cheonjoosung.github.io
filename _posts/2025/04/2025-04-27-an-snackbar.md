---
title: Android Snackbar 사용 방법 (Toast와 비교)
tags: [Android]
style: fill
color: dark
description: Android Snackbar 사용 방법 (Toast와 비교)
---
---

## ✨ 개요

Android 앱에서 사용자의 액션 결과를 알려줄 때 가장 많이 쓰는 UI 컴포넌트 중 하나가 **Snackbar**입니다.  
Toast는 간단한 알림만 가능하지만, Snackbar는 **더 풍부한 피드백과 사용자 액션 제공**이 가능합니다.

이번 글에서는 Snackbar의 기능, 실전 사용 방법, 그리고 Toast 와의 비교까지 **Snackbar 중심**으로 상세히 알아봅니다.

---

## 1. ✅ Snackbar란?
```kotlin
adapter.notifyDataSetChanged()
```
- **Material Design**에 기반한 UI 피드백 컴포넌트입니다.
- 기본적으로 화면 하단에 짧게 표시됩니다.
- **Undo**, **Retry** 등 사용자가 즉시 반응할 수 있는 **액션 버튼**을 추가할 수 있습니다.
- 다른 화면 요소들과 자연스럽게 어우러지며 사용자 흐름을 방해하지 않습니다.

## 2. 🛠️ Snackbar 기본 사용법

```kotlin
Snackbar.make(view, "메시지가 전송되었습니다.", Snackbar.LENGTH_SHORT).show()
```
- view는 Snackbar를 띄울 기준이 되는 레이아웃(View)입니다.
- LENGTH_SHORT, LENGTH_LONG, LENGTH_INDEFINITE로 표시 시간을 조정할 수 있습니다.

---

## 3. 🔥 Snackbar 고급 사용법
### 🔥 액션 버튼 추가하기

```kotlin
Snackbar.make(view, "항목이 삭제되었습니다.", Snackbar.LENGTH_LONG)
    .setAction("취소") {
        // 복구 로직 작성
    }
    .show()
```
- setAction 메서드를 사용하여 사용자가 즉시 반응할 수 있는 버튼을 추가할 수 있습니다.
- 버튼 클릭 시 원하는 작업(취소, 복구 등)을 실행할 수 있습니다.

### 🎨 스타일 커스터마이징
```kotlin
val snackbar = Snackbar.make(view, "커스텀 메시지", Snackbar.LENGTH_LONG)
snackbar.setBackgroundTint(ContextCompat.getColor(context, R.color.black))
snackbar.setTextColor(ContextCompat.getColor(context, R.color.white))
snackbar.setActionTextColor(ContextCompat.getColor(context, R.color.red))
snackbar.show()
```

### 🧩 다양한 Snackbar 예시
- 상황	사용 예시 코드 
  + 저장 완료 알림	
    + Snackbar.make(view, "저장되었습니다.", Snackbar.LENGTH_SHORT).show()
  + 네트워크 실패 후 재시도
    + Snackbar.make(view, "네트워크 오류", Snackbar.LENGTH_INDEFINITE).setAction("다시 시도") { retry() }.show()
  + 삭제 후 Undo 제공
    + Snackbar.make(view, "삭제되었습니다.", Snackbar.LENGTH_LONG).setAction("복구") { undoDelete() }.show()

---

## 4. 📊 Toast와 Snackbar 간단 비교

| 항목                | Toast                         | Snackbar                        |
|---------------------|-------------------------------|---------------------------------|
| **표시 위치**         | 화면 위치 설정 가능             | View 기반, 보통 하단             |
| **사용자 액션**        | 불가능                         | 가능 (Undo, Retry 등)            |
| **디자인**            | 기본 Android 스타일             | Material Design 기반             |
| **사용 시점**          | 간단한 메시지 전달               | 중요한 작업 결과 알림, 복구 제공 |

---

## 5. 결론
Snackbar는 단순한 알림 그 이상입니다.
사용자의 흐름을 방해하지 않고 자연스럽게 피드백을 제공하며, 필요시 즉각적인 액션을 유도할 수 있습니다.

앞으로 Toast 대신 Snackbar를 적극 활용해 더 부드럽고 사용자 친화적인 앱을 만들어보세요! 🚀