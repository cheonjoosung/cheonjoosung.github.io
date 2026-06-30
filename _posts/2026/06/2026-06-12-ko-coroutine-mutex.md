---
title: (Kotlin/코틀린) Coroutines Mutex 완전 정리 — 코루틴 동기화
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin Coroutines의 Mutex를 사용한 동기화 원리와 synchronized/ReentrantLock과의 차이, 실전 race condition 해결 패턴을 정리합니다.
---

---

## 1. 왜 코루틴에서는 synchronized를 쓰면 안 되는가?

자바의 `synchronized`와 `ReentrantLock`은 **스레드 블로킹** 기반입니다.  
코루틴은 스레드와 1:1로 매칭되지 않기 때문에 문제가 발생합니다.

```kotlin
// 위험한 패턴: 코루틴 안에서 synchronized 사용
val lock = Any()
var counter = 0

suspend fun increment() {
    synchronized(lock) {
        counter++
        delay(100)  // ⚠️ synchronized 블록 안에서 suspend 함수 호출 불가 (컴파일 에러)
    }
}
```

`synchronized` 블록 안에서는 `suspend` 함수를 호출할 수 없습니다.  
스레드를 점유한 채로 멈추면 다른 코루틴이 그 스레드를 사용할 수 없어 **스레드풀 전체가 고갈**될 위험도 있습니다.

---

## 2. Mutex란?

`kotlinx.coroutines.sync.Mutex`는 **코루틴 친화적인 상호 배제 락**입니다.  
잠금을 시도하는 동안 스레드를 블로킹하지 않고 **코루틴을 suspend** 시킵니다.

```kotlin
import kotlinx.coroutines.sync.Mutex
import kotlinx.coroutines.sync.withLock

val mutex = Mutex()
var counter = 0

suspend fun increment() {
    mutex.withLock {
        counter++
        delay(100)  // suspend 함수 호출 가능!
    }
}
```

---

## 3. race condition 재현과 해결

### 문제 상황

```kotlin
var counter = 0

suspend fun incrementWithoutLock() {
    coroutineScope {
        repeat(1000) {
            launch(Dispatchers.Default) {
                counter++  // race condition 발생
            }
        }
    }
}

// 실행 결과: counter는 1000이 아닐 수 있음 (여러 코루틴이 동시에 ++ 연산)
```

### Mutex로 해결

```kotlin
val mutex = Mutex()
var counter = 0

suspend fun incrementWithLock() {
    coroutineScope {
        repeat(1000) {
            launch(Dispatchers.Default) {
                mutex.withLock {
                    counter++  // 한 번에 하나의 코루틴만 접근
                }
            }
        }
    }
}
// 실행 결과: counter == 1000 보장
```

---

## 4. withLock vs lock/unlock 수동 처리

```kotlin
// 권장: withLock (try-finally로 자동 unlock 보장)
suspend fun safeOperation() {
    mutex.withLock {
        riskyOperation()  // 예외가 발생해도 unlock 보장
    }
}

// 비권장: 수동 lock/unlock (예외 시 unlock 누락 위험)
suspend fun unsafeOperation() {
    mutex.lock()
    try {
        riskyOperation()
    } finally {
        mutex.unlock()  // 직접 작성해야 함, 빠뜨리면 데드락
    }
}
```

`withLock`은 내부적으로 `try-finally`를 사용하므로 항상 `withLock`을 우선 사용하세요.

---

## 5. Mutex vs synchronized vs ReentrantLock 비교

| 구분 | synchronized | ReentrantLock | Mutex |
|------|--------------|---------------|-------|
| 블로킹 방식 | 스레드 블로킹 | 스레드 블로킹 | 코루틴 suspend |
| suspend 함수 호출 | 불가 | 불가 | 가능 |
| 재진입(re-entrant) | 가능 | 가능 | **불가** (재진입 시 데드락) |
| 공정성(fairness) | 보장 안 함 | 옵션 제공 | 보장 안 함 |
| 사용 대상 | 일반 스레드 코드 | 일반 스레드 코드 | 코루틴 코드 |

### Mutex 재진입 주의

```kotlin
val mutex = Mutex()

suspend fun outer() {
    mutex.withLock {
        inner()  // ⚠️ 데드락 발생! Mutex는 재진입 불가
    }
}

suspend fun inner() {
    mutex.withLock {
        // outer에서 이미 잠금을 획득한 상태에서 다시 잠금 시도 → 영원히 대기
    }
}
```

재진입이 필요한 구조라면 락 범위를 재설계하거나, 락을 이미 획득했는지 추적하는 별도 플래그가 필요합니다.

---

## 6. 실전 패턴 — 캐시 동기화

```kotlin
class ImageCache {
    private val mutex = Mutex()
    private val cache = mutableMapOf<String, Bitmap>()

    suspend fun getOrLoad(url: String): Bitmap {
        // 캐시 확인과 로드를 한 번에 묶어 race condition 방지
        return mutex.withLock {
            cache[url] ?: run {
                val bitmap = downloadImage(url)  // suspend 함수
                cache[url] = bitmap
                bitmap
            }
        }
    }
}
```

## 7. 실전 패턴 — 순차 처리 보장

```kotlin
class OrderProcessor {
    private val mutex = Mutex()
    private var isProcessing = false

    suspend fun processOrder(order: Order) {
        mutex.withLock {
            // 동시에 여러 주문이 처리되어 재고가 음수가 되는 것을 방지
            val stock = inventoryRepository.getStock(order.productId)
            if (stock < order.quantity) {
                throw InsufficientStockException()
            }
            inventoryRepository.decreaseStock(order.productId, order.quantity)
            orderRepository.save(order)
        }
    }
}
```

---

## 8. tryLock — 논블로킹 시도

```kotlin
suspend fun tryUpdate(): Boolean {
    return if (mutex.tryLock()) {
        try {
            updateData()
            true
        } finally {
            mutex.unlock()
        }
    } else {
        false  // 이미 다른 코루틴이 락을 점유 중 → 즉시 포기
    }
}
```

이미 진행 중인 작업이 있으면 중복 실행을 막고 즉시 반환하고 싶을 때 유용합니다 (예: 새로고침 버튼 중복 클릭 방지).

---

## 9. Mutex vs Semaphore

| 구분 | Mutex | Semaphore |
|------|-------|-----------|
| 동시 접근 허용 수 | 1 | N (설정 가능) |
| 용도 | 단일 자원 보호 | 동시 실행 수 제한 |

Mutex는 `Semaphore(1)`과 동일하게 동작하며, 실제로 Kotlin 내부에서도 `Mutex`는 `Semaphore(permits = 1)`로 구현되어 있습니다.

---

## 10. 정리

- **`Mutex`**: 코루틴 친화적 락, 스레드 블로킹 없이 suspend로 대기
- `synchronized`/`ReentrantLock`은 코루틴 안에서 사용하면 스레드 고갈 위험, suspend 함수 호출 불가
- `withLock`을 사용해 예외 발생 시에도 unlock 보장
- Mutex는 **재진입 불가** — 같은 코루틴에서 중첩 lock 시 데드락
- 동시 접근 수를 1보다 크게 허용해야 한다면 `Semaphore` 사용
