---
title: (Android/안드로이드) StateFlow + Coroutine으로 쓰레드 블로킹 없이 완료 조건을 기다리는 방법
tags: [Android,Kotlin]
style: fill
color: dark
description: (Android/안드로이드) StateFlow + Coroutine으로 쓰레드 블로킹 없이 완료 조건을 기다리는 방법
---

## ✨ 개요

안드로이드에서 다음과 같은 상황은 매우 흔하다.
- A/B/C 같은 여러 준비 작업(또는 이벤트)이 순서 없이 완료됨
- 모든 조건이 충족될 때까지 대기해야 함
- 하지만 CountDownLatch.await() 같은 방식으로 쓰레드를 블로킹하면 안 됨
- 조건이 만족되면 다음 로직을 “딱 1번만” 실행해야 함

이때 가장 깔끔한 해법은
- MutableStateFlow<Set<T>>로 완료 상태를 누적
- first { 조건 }로 suspend 대기 (쓰레드는 놀지 않고, 코루틴만 대기 상태로 둠)

---

## 1. 핵심 개념: “블로킹 대기”가 아니라 “서스펜드 대기”

아래 코드는 “기다린다”는 점에서 동일해 보이지만 완전히 다르게 동작한다.
- CountDownLatch.await() → 쓰레드 블로킹
- flow.first { 조건 } → 코루틴 서스펜드(Non-blocking wait)
> 즉, first {}는 조건이 만족될 때까지 현재 코루틴을 중단(suspend) 하지만, 쓰레드는 다른 작업을 계속 처리할 수 있다

---

## 2. StateFlow 기반 “완료 조건 코디네이터” 구현

### PopupCheckCoordinator (StateFlow + Set 누적)

```kotlin
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.first
import kotlinx.coroutines.flow.update

class PopupCheckCoordinator(
    required: Set<PopupType> = setOf(PopupType.A, PopupType.B, PopupType.C)
) {
    private val required: Set<PopupType> = required.toSet()

    private val checked = MutableStateFlow(emptySet<PopupType>())

    fun markChecked(type: PopupType) {
        checked.update { it + type } // Set 누적(중복 자동 제거)
    }

    suspend fun awaitAllChecked() {
        checked.first { acc ->
            acc.containsAll(required)
        }
    }
}
```

### 호출 방법

```kotlin
val coordinator = PopupCheckCoordinator()

lifecycleScope.launch {
    coordinator.awaitAllChecked()
    doNextLogic()
}
```

---

## 3. 조건 충족 후 딱 한번만 실행

```kotlin
coordinator.awaitAllCheckedAndRunOnce { doNextLogic() }
```

### 실행 딱 한번만 보장 (AtomicBoolean)

```kotlin
import java.util.concurrent.atomic.AtomicBoolean
import kotlinx.coroutines.flow.first
import kotlinx.coroutines.flow.update
import kotlinx.coroutines.flow.MutableStateFlow

class PopupCheckCoordinator(
    required: Set<PopupType> = setOf(PopupType.A, PopupType.B, PopupType.C)
) {
    private val required = required.toSet()
    private val checked = MutableStateFlow(emptySet<PopupType>())

    // 다음 로직은 한 번만 실행하도록 보장
    private val ran = AtomicBoolean(false)

    fun markChecked(type: PopupType) {
        checked.update { it + type }
    }

    suspend fun awaitAllCheckedAndRunOnce(block: suspend () -> Unit) {
        checked.first { it.containsAll(required) } // ✅ non-blocking wait

        // ✅ block은 1회만 실행
        if (ran.compareAndSet(false, true)) {
            block()
        }
    }
}

// 호출
val coordinator = PopupCheckCoordinator()

lifecycleScope.launch {
    coordinator.awaitAllCheckedAndRunOnce {
        doNextLogic()
    }
}
```

---

## 4. StateFlow vs SharedFlow: 왜 여기서는 StateFlow가 적합한가?

이 문제는 “이벤트를 받는다”라기보다 “현재 상태(누적된 완료 집합)를 유지”하는 문제다.
- StateFlow: “현재 상태를 항상 보유” → 완료 집합(Set) 유지에 최적
- SharedFlow: “이벤트 스트림”에 가깝고, 상태 보존은 직접 해야 함 
- 따라서 “A/B/C가 완료됐는지”처럼 상태 기반 대기에는 StateFlow가 더 자연스럽다.

---

## 5. 결론: 이 방식이 ‘쓰레드 블로킹 없이 기다리는 법’인 이유

awaitAllChecked()는 아래 특성을 갖는다.
- 조건이 만족될 때까지 코루틴만 대기(suspend)
- OS 쓰레드는 점유/블로킹되지 않음
- 조건이 만족되면 즉시 실행 재개(resume)
- 즉, 이 구조는 Non-blocking wait (Suspend without blocking)의 대표적인 예