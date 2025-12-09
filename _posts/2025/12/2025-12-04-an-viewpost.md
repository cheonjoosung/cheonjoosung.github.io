---
title: (Android/안드로이드) View.postDelayed()를 이용한 지연 실행
tags: [Android,Kotlin]
style: fill
color: dark
description: (Android/안드로이드) View.postDelayed() 완전 정리 — 가장 간단하고 안전한 지연 실행 방법
---

## ✨ 개요

- 안드로이드 개발에서는 “1초 뒤에 실행하기”, “딜레이 후 특정 UI 처리하기” 같은 작업이 정말 자주 필요하다. Handler, Coroutine, Timer 등 다양한 방법이 있지만, **UI 기반에서 가장 간단하고 안전한 방법은 View.postDelayed()**이다.
  + 동작 방식
  + 활용 패턴
  +  주의사항
  + 다른 delay 방식과의 비교

---

## 1.  요약

- View.postDelayed()는 해당 View가 UI 트리에 붙어 있을 때 main thread에서 안전하게 지연 실행한다.
- Handler를 따로 만들 필요 없고, Activity/Fragment 종료 시 자동 취소되어 메모리 누수 위험도 적다.
- UI 지연 실행만 필요하다면 가장 쉽고 권장되는 방식이다.

---

## 2. View.postDelayed() 기본 사용법

```kotlin
val time = 1000L

binding.myButton.postDelayed({
    // 1초 뒤 실행될 코드
}, time)
```
> 매우 간단하게 UI스레드에서 1초 뒤 실행된다.

---

## 3. post와 postDelayed의 차이

| 메서드                      | 의미                 |
| ------------------------ | ------------------ |
| `post {}`                | 다음 UI 프레임에서 실행     |
| `postDelayed({ }, 1000)` | n ms 뒤 UI 스레드에서 실행 |

> 둘 다 Main Looper 에서 실행되므로 UI작업에 적합하다.

---

## 4. View.postDelayed()가 UI 작업에 적합한 이유

- Main Thread에서 자동 실행
  + Handler를 만들 때 Looper.getMainLooper() 지정할 필요가 없다.
- View lifecycle에 따라 자동 취소
  + 만약 Activity/Fragment가 종료되면 View는 Detach됨 → Runnable은 자동 취소됨.
  + 이게 Handler보다 안전한 이유다.
- UI 컴포넌트와 자연스럽게 통합
  + 특정 버튼/레이아웃 관련 딜레이 작업에 가장 적합하다.
  + 별도의 Scope 관리가 필요 없음
- Coroutine처럼 scope cancel 관리가 필요 없다.

---

## 5. postDelayed 활용 예제

### 5-1. 버튼 클릭 후 1초 뒤 실행

 ```kotlin
binding.btnSend.setOnClickListener {
    binding.progressBar.isVisible = true

    binding.progressBar.postDelayed({
        binding.progressBar.isVisible = false
    }, 1000)
}
```

### 5-2. 애니메이션 끝난 후 다음 동작 실행

 ```kotlin
binding.imageView.animate()
    .alpha(1f)
    .setDuration(300)
    .withEndAction {
        binding.root.postDelayed({
            showToast("애니메이션 후 실행됨")
        }, 1000)
    }

```

---

## 6. postDelayed() 사용 시 주의할 점

- View가 detach되면 실행되지 않는다
  + Fragment 전환 중이라면 실행이 스킵될 수 있다.
  + 예: 화면 전환 직전에 실행 예약된 작업은 사라질 수 있음
- 백그라운드 작업에는 부적합
  + UI Thread에서 실행되므로 heavy 작업 금지
  + 백그라운드 작업은 Coroutine, HandlerThread 사용
- Null View 체크 필요할 수 있음
  + 뷰 바인딩을 사용하는 경우 Fragment Lifecycle에 따라 NPE 방지 필요.

---

## 7. postDelayed 취소하기

```kotlin
val runnable = Runnable {
    // 실행 코드
}

binding.root.postDelayed(runnable, 1000)

// 취소
binding.root.removeCallbacks(runnable)
```
> View 가 사라지면 자동 취소되지만 명시적으로 취소해야 하는 경우 이렇게 처리

---

## 8. Best Practice (실무 팁)

| 방식                                  | 실행 스레드         | UI 작업 적합성 | 라이프사이클 안전성 | 코드 난이도        | 비고                   |
| ----------------------------------- | -------------- | --------- | ---------- | ------------- | -------------------- |
| **View.postDelayed()**              | 메인(UI)         | ⭐⭐⭐⭐⭐     | ⭐⭐⭐⭐⭐      | ⭐⭐⭐⭐⭐ (가장 쉬움) | View가 detach되면 자동 취소 |
| **Handler(Looper.getMainLooper())** | 메인(UI)         | ⭐⭐⭐⭐      | ⭐⭐         | ⭐⭐⭐           | 직접 cancel 필요         |
| **Coroutine + delay**               | 메인/UI or 백그라운드 | ⭐⭐⭐⭐      | ⭐⭐⭐⭐⭐      | ⭐⭐⭐⭐          | scope cancel 필수      |
| **Timer/TimerTask**                 | 백그라운드          | ⭐         | ⭐          | ⭐⭐            | 오래된 방식, 메인 전환 필요     |
| **HandlerThread**                   | 별도 스레드         | ⭐         | ⭐⭐         | ⭐⭐            | 반복/백그라운드 작업에 유용      |
