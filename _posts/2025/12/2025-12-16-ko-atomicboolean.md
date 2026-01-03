---
title: (Kotlin/코틀린) AtomicBoolean 사용 방법
tags: [Kotlin]
style: fill
color: dark
description: (Kotlin/코틀린) AtomicBoolean 사용 방법
---

## ✨ 개요

안드로이드/코틀린 개발을 하다 보면 다음과 같은 요구사항이 자주 등장합니다.
- 한 번만 실행되도록 보장해야 하는 로직
- 동시에 여러 스레드에서 접근하지만 중복 실행을 막아야 하는 상태
- synchronized 없이 가볍게 스레드 안전한 Boolean 플래그를 쓰고 싶을 때

이때 가장 적합한 도구가 바로 **AtomicBoolean**입니다.

---

## 1. 요약

- `AtomicBoolean`은 lock 없이(lock-free) thread-safe한 Boolean
- `get()` / `set()`은 단순 조회/설정
- 핵심은 `compareAndSet()` (CAS)
- “한 번만 실행”, “중복 방지”, “동시 클릭 방어”에 매우 적합
- UI 상태 관리에는 AtomicBoolean보다 StateFlow/LiveData가 더 적합한 경우도 많음

---

## 2. AtomicBoolean 이 필요한 이유

```kotlin
var isRunning = false

if (!isRunning) {
    isRunning = true
    doWork()
}
```

멀티스레드 환경에서는:
- 스레드 A, B가 동시에 isRunning == false를 읽을 수 있음
- 결과적으로 doWork()가 두 번 실행
- ➡️ race condition 발생

```kotlin
val isRunning = AtomicBoolean(false)

if (isRunning.compareAndSet(false, true)) {
    doWork()
}
```
- compareAndSet(false, true)는 한 스레드만 성공
- 나머지는 실패 → 중복 실행 방지

---

## 3. AtomicBoolean 기본 API

### 3.1 get/set

```kotlin
val flag = AtomicBoolean(false)

flag.get()      // 현재 값 조회
flag.set(true) // 값 설정
```

### 3.2 compareAndSet (가장 중요 ⭐)

```kotlin
flag.compareAndSet(expect = false, update = true)
```
> “현재 값이 false라면 true로 바꾸고, 성공하면 true를 반환”
- 단일 원자 연산
- lock 없이 구현(CPU CAS 명령)

### 3.3 getAndSet

```kotlin
val previous = flag.getAndSet(true)
```
- 이전 값을 반환하면서 새 값으로 설정
- 토글/상태 스냅샷에 유용

---

## 4. 실무 패턴

### 4-1. 앱 최초 1회 초기화

```kotlin
private val initialized = AtomicBoolean(false)

fun initOnce() {
    if (initialized.compareAndSet(false, true)) {
        initSdk()
    }
}
```

### 4-2. 중복 클릭 방지

```kotlin
private val isProcessing = AtomicBoolean(false)

fun onButtonClick() {
    if (!isProcessing.compareAndSet(false, true)) return

    doAsyncWork {
        isProcessing.set(false)
    }
}
```

---

## 5. AtomicBoolean vs synchronized vs volatile

| 구분        | AtomicBoolean | synchronized | volatile |
| --------- | ------------- | ------------ | -------- |
| 원자성       | ⭕             | ⭕            | ❌        |
| 락 사용      | ❌             | ⭕            | ❌        |
| 성능        | 매우 좋음         | 상대적으로 느림     | 좋음       |
| 복합 연산 안전  | ⭕ (CAS)       | ⭕            | ❌        |
| 단순 가시성 보장 | ⭕             | ⭕            | ⭕        |
