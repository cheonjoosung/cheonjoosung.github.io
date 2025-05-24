---
title: Android Handler를 이용한 지연 실행 - postDelayed() 원리와 실전 예제
tags: [Android]
style: fill
color: dark
description: Android에서 Handler를 사용하여 딜레이 처리하는 방법과 작동 원리를 예제와 함께 정리합니다.
---

## ✨ 개요


**Handler는 메시지(Message)나 실행(Runnable)을 스레드의 메시지 큐로 전달하는 역할**을 합니다.

Handler의 역할
- 메시지 전달 및 처리
- 일정 시간 후 작업 실행 (`postDelayed`)
 - 특정 스레드(UI 스레드 포함)의 `MessageQueue`에 접근 가능

---

## 1. ✅ 작동 원리

Handler는 내부적으로 `Looper`와 `MessageQueue`를 이용합니다.
1. `Handler`는 지정한 스레드의 `MessageQueue`에 작업을 등록합니다.
2. `Looper`는 해당 큐에서 메시지를 하나씩 꺼내 실행합니다.
3. `postDelayed()`는 특정 지연 시간(ms) 후 `Runnable`을 큐에 넣습니다.

> ✅ 기본적으로 MainThread(UI Thread)에 연결되어 있어 UI 작업도 안전하게 실행 가능합니다.

---

## 2. ✅ 기본 사용법 : postDelayed

```kotlin
val handler = Handler(Looper.getMainLooper())

handler.postDelayed({
    Log.d("TAG", "3초 후 실행됨!")
}, 3000) // 3000ms = 3초
```
- Looper.getMainLooper() : 메인(UI) 스레드의 메시지 루프에 연결
- postDelayed(Runnable, delayMillis) : 지정 시간 후에 실행될 작업을 큐에 등록

---

## 3. 💡 응용 예제

### 3-1. 버튼 클릭 후 2초 뒤 메시지 출력

```kotlin
val handler = Handler(Looper.getMainLooper())

binding.button.setOnClickListener {
    Toast.makeText(applicationContext, "2초뒤 메시지가 나옵니다!", Toast.LENGTH_SHORT).show()

    handler.postDelayed({
        Toast.makeText(applicationContext, "지금 실행 됨", Toast.LENGTH_SHORT).show()
    }, 2000)
}
```

### 3-2. 로딩 후 자동 종료

```kotlin
fun showLoadingAndDismiss() {
    loadingView.visibility = View.VISIBLE

    handler.postDelayed({
        loadingView.visibility = View.GONE
    }, 1500)
}
```

---

## 4.⚠️ **주의사항** && 기타사항

### 4-1 주의사항

| 항목         | 설명                                                                         |
| ---------- | -------------------------------------------------------------------------- |
| 메모리 누수 가능성 | Handler는 액티비티를 참조할 수 있기 때문에 `onDestroy()`에서 `removeCallbacks()`를 사용해 정리 필요 |
| 오래된 방식     | Jetpack Coroutine이나 Lifecycle-aware한 방식이 등장했지만, 간단한 UI 지연에는 여전히 유용         |
| 정확한 타이밍 아님 | Handler는 100% 정밀하지 않으며 시스템 상태에 따라 오차 발생 가능                                 |


### 4-2 Handler VS Coroutine Delay VS Timer

| 방식        | 간단성  | 정확도  | 메모리 안전성        | 사용 추천 상황         |
| --------- | ---- | ---- | -------------- | ---------------- |
| Handler   | ✅ 높음 | 보통   | ❌ 직접 정리 필요     | UI에서 간단한 지연 실행   |
| Coroutine | 보통   | ✅ 높음 | ✅ Lifecycle 대응 | 비동기 처리, API 대기 등 |
| Timer     | ❌ 낮음 | 보통   | ❌ 위험           | 거의 사용하지 않음       |


---

## 5.🧠 **결론**

- Handler는 Android에서 오래전부터 사용되던 지연 실행 방식으로, UI 작업을 안전하게 지연 실행할 수 있는 장점이 있습니다.
- postDelayed()를 통해 간단한 딜레이 로직을 구현할 수 있으며, 메시지 큐 기반으로 작동하는 구조를 이해하면 더욱 효과적으로 사용할 수 있습니다.
- 다만, 메모리 누수에 주의하고, 코루틴 등 대체 방식도 고려해보면 좋습니다.