---
title: (Android/안드로이드) Handler(Looper.getMainLooper()) 완전 정리
tags: [Android,Kotlin]
style: fill
color: dark
description: (Android/안드로이드) Handler(Looper.getMainLooper()) 완전 정리 — 메인(UI) 스레드에서 안전하게 작업 예약하기
---

## ✨ 개요

- Android 개발에서 “UI 스레드에서 실행 보장”, “n ms 후 실행”, “백그라운드 결과를 메인으로 전달” 같은 요구사항은 매우 흔합니다.
- 이때 가장 전통적이면서도 확실한 도구가 Handler이며, 특히 UI 스레드 전용 핸들러는 보통 아래 형태로 사용합니다.

---

## 1.  요약

- `Handler(Looper.getMainLooper())`는 메인(UI) 스레드의 MessageQueue에 작업을 넣는 도구
- post {}는 “가능한 빨리(다음 루프)” 실행, postDelayed {}는 “지정 시간 뒤” 실행
- 예약한 작업은 removeCallbacks()로 반드시 취소 가능하게 관리하는 것이 안전
- UI 작업은 메인 스레드에서만 해야 하므로, 백그라운드 결과를 메인으로 올릴 때 유용
- Modern Android에서는 Coroutine이 권장되지만, Handler는 여전히 필요한 케이스가 많음(레거시/브릿지/SDK/저수준 제어)

---

## 2. Handler와 Looper를 한 문장으로 설명하면?

- Looper: 한 스레드에 붙어 있는 “이벤트 루프(메시지 처리기)”
- MessageQueue: 해당 스레드가 처리할 작업(메시지/런너블)이 줄 서는 큐
- Handler: “어떤 Looper의 큐에 작업을 넣을지”를 지정해주는 전달자(Dispatcher)


즉, Handler(Looper.getMainLooper())는:
> “메인 스레드(=UI 스레드)의 이벤트 큐에 작업을 넣어서, UI 스레드에서 실행되게 하겠다”

---

## 3. Looper.getMainLooper() vs Looper.myLooper()

- Looper.getMainLooper()
  + 항상 메인(UI) 스레드의 Looper를 반환
  + 어디서 호출하든 결과는 동일(메인 Looper)
- Looper.myLooper()
  + 현재 실행 중인 스레드의 Looper 반환
  + 현재 스레드에 Looper가 없으면 null
- 따라서 UI 작업 보장은:
  + ```kotlin
    Handler(Looper.getMainLooper()).post { /* UI */ }
    ```

---

## 4. 가장 많이 쓰는 패턴 4가지

#### 4.1 메인 스레드로 전환(즉시 실행)

백그라운드 스레드에서 UI를 갱신해야 할 때:

```kotlin
val mainHandler = Handler(Looper.getMainLooper())

fun updateUiFromBg() {
    mainHandler.post {
        // UI 업데이트는 여기서
        binding.textView.text = "Updated"
    }
}

```

#### 4.2 1초 지연 실행

```kotlin
val mainHandler = Handler(Looper.getMainLooper())

mainHandler.postDelayed({
    // 1초 후 실행
}, 1000L)

```

#### 4.3 디바운스(Debounce) 연속 호출 중 마지막만 실행

검색창 입력, 다중 클릭 등:

```kotlin
private val mainHandler = Handler(Looper.getMainLooper())
private var debounceRunnable: Runnable? = null

fun debounceSearch(query: String) {
    debounceRunnable?.let { mainHandler.removeCallbacks(it) }

    debounceRunnable = Runnable {
        // 마지막 입력 이후 300ms 지나면 실행
        performSearch(query)
    }.also {
        mainHandler.postDelayed(it, 300L)
    }
}
```

#### 4.4 특정 Runnable을 취소(removeCallbacks)

```kotlin
val mainHandler = Handler(Looper.getMainLooper())
val r = Runnable { /* 실행 */ }

mainHandler.postDelayed(r, 1000L)

// 필요 시 취소
mainHandler.removeCallbacks(r)
```

---

## 5. 내부 동작

Coroutine delay는 어떤 Dispatcher에서도 사용할 수 있다.

### 5-1. Main에서 delay 후 UI 갱신

- Runnable을 Message로 래핑
- 메인 스레드 MessageQueue에 “현재 시간 + 1000ms”로 enqueue
- 메인 Looper가 루프를 돌면서 시간이 되면 dispatch
- 메인 스레드에서 Runnable 실행
> 즉, “작업을 큐에 예약해둔다”가 정확한 표현입니다.

---

## 6. 메모리 누수/크래시를 피하는 핵심 규칙

### 6.1 Activity/Fragment 참조를 오래 잡지 않기

Handler로 딜레이를 걸면, Runnable이 실행될 때까지 외부 참조가 유지될 수 있습니다.

안전한 습관:
- onDestroy() / onDestroyView()에서 예약된 콜백 제거
- ViewBinding은 Fragment에서 특히 중요

Fragment 예시:

```kotlin
private val mainHandler = Handler(Looper.getMainLooper())
private var pending: Runnable? = null

override fun onDestroyView() {
    pending?.let { mainHandler.removeCallbacks(it) }
    pending = null
    super.onDestroyView()
}
```

### 6-2. View가 이미 사라졌는데 UI 갱신하려는 문제

- 딜레이 후 실행되는 코드에서는 “뷰가 아직 유효한가”가 중요합니다.
- (특히 Fragment binding)

---

## 7. Coroutine / View.postDelayed와 비교(언제 Handler가 맞나)

- Coroutine이 이미 프로젝트 표준이면 lifecycleScope.launch { delay(1000) }가 더 깔끔할 때가 많습니다.
- 반면 Handler는 이런 경우에 여전히 강점이 있습니다:
  + 레거시 코드/SDK 연동
  + 브릿지/콜백 기반 구조(코루틴 스코프가 애매한 곳)
  + 디바운스/스로틀 등 “메시지 큐 기반 제어”가 필요할 때
  + UI 스레드에 “정확히 이 Runnable”을 등록하고 취소해야 할 때

---

## 8. 결론

Handler(Looper.getMainLooper())는 “메인 스레드 큐에 작업을 예약”하는 가장 전통적인 방식이며, 지금도 실무에서 충분히 유효합니다.
- 메인 Looper를 명시
- 취소(removeCallbacks)를 습관화
- Fragment/ViewBinding 수명에 주의