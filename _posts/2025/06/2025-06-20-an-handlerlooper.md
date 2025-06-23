---
title: Android Handler, Looper, MessageQueue 완벽 정리
tags: [ Android ]
style: fill
color: dark
description: Android의 스레드 통신 메커니즘에서 핵심 역할을 하는 Handler, Looper, MessageQueue의 역할과 차이점, 그리고 실전 코드 예제를 정리합니다.
---

## ✨ 개요

Android에서 메인 스레드(UI Thread) 또는 워커 스레드 간의 작업을 안전하게 처리하기 위해 Handler, Looper, MessageQueue가 긴밀히 협력합니다. 
이들은 메시지 기반의 비동기 작업 처리를 가능하게 하는 핵심 컴포넌트입니다.

---

## 1. 📦 각 구성요소의 역할

- Handler
  + 메시지를 생성하거나 전송하고 이를 수신하여 처리하는 도구
- Looper
  + 스레드당 1개 존재, 메시지 큐를 돌면서 메시지를 계속 처리
- MessageQueue
  + 실제 메시지 객체를 저장하고 처리 순서를 유지하는 큐

```kotlin
// 예: 메인스레드에서 실행
val handler = Handler(Looper.getMainLooper())
handler.post {
    Log.d("TAG", "이 코드는 메인 스레드에서 실행됩니다")
}
```

---

## 2. ✅ 공통점과 차이점

- 공통점
  + 모두 Android 스레드 메시징 시스템의 핵심 구성요소
  + 비동기 메시지 또는 Runnable 작업을 처리하기 위해 함께 사용됨
- 차이점
  + | 항목        | Handler                          | Looper                             | MessageQueue                         |
    |-------------|-----------------------------------|-------------------------------------|--------------------------------------|
    | 소속        | 개발자가 명시적으로 생성         | 스레드에 1개만 존재 (명시적 시작 필요) | Looper 내부에 포함됨                 |
    | 책임        | 메시지 전송 및 처리               | 메시지 루프 실행                     | 메시지 저장/순서 관리                |
    | 생성 방식   | 직접 생성 + Looper 필요           | `Looper.prepare()` + `Looper.loop()` | 시스템 또는 Looper가 생성 관리      |

---

## 3. 🧪 샘플 코드: 워커 스레드에서 Looper 사용

```kotlin
class WorkerThread : Thread() {
    lateinit var handler: Handler

    override fun run() {
        Looper.prepare()

        handler = Handler(Looper.myLooper()!!) {
            Log.d("WorkerThread", "Message received on worker thread")
            true
        }

        Looper.loop()
    }
}

val thread = WorkerThread()
thread.start()

// 메인스레드에서 메시지 전송
thread.handler.post {
    Log.d("Main", "메시지 전송 완료")
}
```

---

## 4. 🧾 결론

- Handler, Looper, MessageQueue는 Android의 스레드 간 작업 처리 시스템을 구성하는 필수 요소입니다.
- 이 구조를 이해하면 비동기 처리, 백그라운드 작업 분리, UI 스레드 보호에 효과적으로 대응할 수 있습니다.
- 특히 커스텀 워커 스레드 구성 시 Looper와 Handler를 함께 사용하면 더욱 정교한 스레드 관리를 할 수 있습니다.