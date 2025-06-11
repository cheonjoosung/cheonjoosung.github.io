---
title: Android에서 Strong Reference & Weak Reference 정리
tags: [ Android ]
style: fill
color: dark
description: Android Strong Reference와 Weak Reference는 메모리 관리와 관련된 중요한 개념입니다. Android에서의 차이점, 사용 예시, 주의사항을 간단한 예제와 함께 정리합니다.
---

## ✨ 개요

- Java 기반의 Android에서는 객체 간 참조가 메모리 관리에 큰 영향을 줍니다. 
- 특히 Strong Reference와 Weak Reference의 개념을 이해하면 메모리 누수 방지에 큰 도움이 됩니다.
- Garbage Collector(GC)는 참조 방식에 따라 객체를 메모리에서 해제할지 판단합니다.

---

## 1. 💪 Strong Reference란?

- 기본적으로 우리가 선언하는 대부분의 참조는 Strong Reference
- GC는 Strong Reference가 존재하는 한 해당 객체를 절대 수거하지 않음
> val activity = MainActivity() // Strong Reference

---

## 2. 🪶 Weak Reference란?

- GC가 객체를 참조하는 약한 참조
- 객체가 더 이상 Strong Reference에 의해 참조되지 않으면 GC는 이를 자유롭게 수거할 수 있음
```kotlin
val contextRef = WeakReference(context)
val context = contextRef.get()
```
- Handler, Listener, Callbacks에서 Activity 참조 시 메모리 누수 방지를 위해 사용
  + ```kotlin
    import java.lang.ref.WeakReference

    class MyHandler(activity: Activity) : Handler(Looper.getMainLooper()) {
    private val activityRef = WeakReference(activity)

        override fun handleMessage(msg: Message) {
            val activity = activityRef.get()
            activity?.let {
                // activity가 아직 살아있을 때만 실행
                it.updateUI()
            }
        }
    }
    ```

---

## 3. 📌 언제 사용해야 할까?

| 상황                          | 추천 참조 방식     | 이유                                        |
|-----------------------------|-------------------|---------------------------------------------|
| UI 객체, 뷰 직접 사용            | Strong Reference | 뷰 생명주기 내에서는 GC에 의해 수거되면 안 됨     |
| 오래 실행되는 백그라운드 작업에서 Context 참조 | Weak Reference  | 작업이 끝난 후 GC에 의해 해제되도록 하기 위함     |
| 콜백, 리스너 등록 시 Activity 참조 | Weak Reference  | 리스너가 Activity를 계속 붙잡고 있는 것을 방지함 |

---

## 4.🧠 결론

- Strong Reference는 일반적인 객체 참조이며, Weak Reference는 메모리 누수 방지용으로 유용하게 활용됩니다.
- 특히 Android에서는 Activity/Context를 잘못 참조하면 GC가 수거하지 못해 앱 성능에 영향을 줍니다.
- 상황에 맞게 두 참조 방식을 구분해서 사용하는 습관이 필요합니다.