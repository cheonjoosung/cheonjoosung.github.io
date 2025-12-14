---
title: (Android/안드로이드) Thread 사용 시 꼭 알아야 할 유용한 메서드 정리
tags: [Android,Kotlin]
style: fill
color: dark
description: (Android/안드로이드) Thread 사용 시 꼭 알아야 할 유용한 메서드 정리
---

## ✨ 개요

안드로이드 개발을 하면 다음과 같은 로그를 보게된다.

```cpp
main
DefaultDispatcher-worker-1
pool-3-thread-1
OkHttp Dispatcher
```

이런 로그의 출처는 대부분 Thread.currentThread()다.
스레드 이름을 정확히 이해하고 활용하면 UI 스레드 문제, 비동기 버그, 크래시 원인을 훨씬 빠르게 추적할 수 있다.

---

## 1.  요약

- Thread.currentThread().name → 현재 실행 중인 스레드 이름 확인
- isMainThread 체크는 UI 크래시 방지의 기본
- id, priority, state는 디버깅에 매우 유용
- 스레드 이름 로그만 잘 찍어도 버그 원인의 50%는 잡힌다

---

## 2. Thread.currentThread()

```kotlin
val thread = Thread.currentThread()
```

- 현재 코드를 실행 중인 스레드 객체를 반환
- 어디서 호출하든 “지금 이 코드가 어느 스레드에서 실행 중인지” 알 수 있음
- 디버깅, 로깅, 스레드 분기 처리의 출발점


즉, Handler(Looper.getMainLooper())는:
> “메인 스레드(=UI 스레드)의 이벤트 큐에 작업을 넣어서, UI 스레드에서 실행되게 하겠다”

---

## 3. Thread.currentThread().name

```kotlin
Log.d("THREAD", Thread.currentThread().name)
```

| 스레드 이름                       | 의미                           |
| ---------------------------- | ---------------------------- |
| `main`                       | 메인(UI) 스레드                   |
| `DefaultDispatcher-worker-X` | Coroutine Default Dispatcher |
| `pool-1-thread-1`            | ExecutorService              |
| `OkHttp Dispatcher`          | OkHttp 네트워크 스레드              |
| `Binder:X_Y`                 | IPC Binder 스레드               |

---

## 4. 현재 코드가 메인(UI) 스레드인지 확인하기

```kotlin
val isMainThread = Thread.currentThread() == Looper.getMainLooper().thread

val isMainLooper = Looper.myLooper() == Looper.getMainLooper()

fun assertMainThread() {
    check(Looper.myLooper() == Looper.getMainLooper()) {
        "UI thread에서 실행되어야 합니다. current=${Thread.currentThread().name}"
    }
}
```

---

## 5. Thread.currentThread().id - Thread 고유 ID

```kotlin
val id = Thread.currentThread().id
```

- JVM이 부여하는 고유 식별자
- 같은 이름의 스레드라도 ID는 다를 수 있음
- 동시성 문제 추적 시 도움됨

---

## 6. Thread.currentThread().state - 스레드 상태 확인

```kotlin
val state = Thread.currentThread().state
```

| 상태            | 의미            |
| ------------- | ------------- |
| NEW           | 생성만 되고 시작 안 됨 |
| RUNNABLE      | 실행 중 또는 실행 가능 |
| BLOCKED       | lock 대기 중     |
| WAITING       | 무기한 대기        |
| TIMED_WAITING | 시간 제한 대기      |
| TERMINATED    | 종료            |

---

## 7. Thread.currentThread().priority - 우선순위

```kotlin
val priority = Thread.currentThread().priority
```

- 1 ~ 10 범위
- Android에서는 대부분 큰 의미는 없지만
- 커스텀 스레드/레거시 코드에서 참고용으로 사용

---

## 8. Thread.sleep() - 사용 주의

- 문제점
  - 현재 스레드를 완전히 블로킹
  - main thread에서 호출 시 → 즉시 ANR 위험

- 대체 수단
  - UI: Handler.postDelayed, View.postDelayed
  - 비동기: Coroutine delay()

---

## 9. 정리

| 메서드                      | 용도                |
| ------------------------ | ----------------- |
| `Thread.currentThread()` | 현재 스레드 객체         |
| `.name`                  | 스레드 이름 로그 (가장 중요) |
| `.id`                    | 고유 식별자            |
| `.state`                 | 상태 디버깅            |
| `.priority`              | 참고용               |
| `Looper.myLooper()`      | UI 스레드 체크         |