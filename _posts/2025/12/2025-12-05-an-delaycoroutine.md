---
title: (Android/안드로이드) Coroutine으로 1초 지연 실행하기
tags: [Android,Kotlin]
style: fill
color: dark
description: (Android/안드로이드) Coroutine으로 1초 지연 실행하기 — delay() 완전 정리
---

## ✨ 개요

- Android/Kotlin 개발에서 **“1초 뒤 실행하기”**는 매우 자주 필요한 기능이다.
- 과거에는 Handler, Timer 등을 사용했지만, **Modern Android Development(MAD)**에서는 **Coroutine + delay()**가 가장 권장된다.
- Coroutine은 구조화된 동시성(Structured Concurrency)을 갖고 있기 때문에 LifecycleScope, ViewModelScope 등과 함께 사용하면 취소·메모리 관리까지 자연스럽게 해결된다.

---

## 1.  요약

- `delay(1000)` → 스레드를 블로킹하지 않고 1초 뒤 실행
- Activity/Fragment에서는 lifecycleScope.launch
- ViewModel에서는 viewModelScope.launch
- 반복 delay도 쉽게 구현 가능
- Coroutine은 cancel이 편하고 lifecycle에 안전

---

## 2. Coroutine delay 기본 사용법

```kotlin
GlobalScope.launch {
    delay(1000)
    println("1초 뒤 실행됨")
}
```
> 하지만 GlobalScope는 절대 사용하면 안 된다. (수명 관리가 되지 않아 메모리 누수 위험)

---

## 3. Activity/Fragment에서 delay 실행 (lifecycleScope)

UI에서 delay를 실행하려면 lifecycleScope를 사용한다.

```kotlin
lifecycleScope.launch {
    delay(1000)
    // 1초 뒤 실행
    binding.textView.text = "1초 뒤 업데이트!"
}
```

- Activity/Fragment lifecycle에 맞춰 자동으로 cancel
- UI 업데이트에 적합
- 매우 간단하고 안전함

---

## 4. ViewModel에서 delay 실행 (viewModelScope)

ViewModel에서는 viewModelScope로 실행한다.

```kotlin
viewModelScope.launch {
    delay(1000)
    _uiState.value = _uiState.value.copy(showMessage = true)
}
```

- 화면이 회전되거나 Fragment가 recreate 되어도 취소되지 않음
- UIState 조작에 적합

---

## 5. Dispatchers 지정 (Main / IO / Default)

Coroutine delay는 어떤 Dispatcher에서도 사용할 수 있다.

### 5-1. Main에서 delay 후 UI 갱신

 ```kotlin
lifecycleScope.launch(Dispatchers.Main) {
    delay(1000)
    updateUI()
}

```

### 5-2. 백그라운드(IO)에서 delay 후 작업 수행

 ```kotlin
lifecycleScope.launch(Dispatchers.IO) {
    delay(1000)
    doBackgroundTask()
}
```

- delay는 스레드를 블로킹하지 않기 때문에 IO에서 사용해도 안전하다.

---

## 6. try/catch로 delay 취소 처리하기

- Coroutine이 cancel되면 CancellationException이 발생한다.
- delay도 예외를 던지기 때문에 try/catch로 처리할 수 있다.

```kotlin
lifecycleScope.launch {
    try {
        delay(1000)
        println("실행됨")
    } catch (e: CancellationException) {
        println("코루틴 취소됨")
    }
}
```

---

## 7. delay VS Thread.sleep 차이

| 구분            | delay            | Thread.sleep       |
| ------------- | ---------------- | ------------------ |
| 스레드 블로킹 여부    | ❌ 블로킹하지 않음       | ⭕ 해당 스레드를 블로킹      |
| 사용 위치         | Coroutine 내에서 사용 | 어디서나 가능            |
| 권장 여부         | ⭐⭐⭐⭐⭐ 매우 권장      | ❌ 안드로이드에서 거의 사용 금지 |
| lifecycle 안전성 | 있음               | 없음                 |

---

## 8. binding.progressBar.isVisible = true

```kotlin
lifecycleScope.launch {
    delay(1000)
    binding.progressBar.isVisible = false
}
```
---

## 9. View.postDelayed() vs Coroutine delay

| 항목              | Coroutine delay                          | View.postDelayed     |
| --------------- | ---------------------------------------- | -------------------- |
| 실행 위치           | CoroutineScope 내부                        | View가 attach된 상태     |
| 스레드 블로킹         | ❌ 없음                                     | ❌ 없음                 |
| lifecycle 자동 취소 | Activity/Fragment/ViewModelScope 사용 시 가능 | View가 detach되면 자동 취소 |
| 코드 간결성          | ⭐⭐                                       | ⭐⭐⭐⭐⭐                |
| 취소 편의성          | 매우 좋음                                    | removeCallbacks 필요   |
| UI 관련 작업        | 매우 적합                                    | View 기반 작업에 최적       |
| 백그라운드 작업        | 매우 적합                                    | 부적합                  |