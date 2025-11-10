---
title: (CS/컴퓨터공학) 뮤텍스 vs 세마포어 이해하기 
tags: [CS]
style: fill
color: dark
description: CS/컴퓨터공학 뮤텍스 vs 세마포어 그림으로 이해하는 동기화, 임계구역, 교착상태 + Kotlin 예제
---

## ✨ 개요

동시성에서 가장 중요한 건 **임계구역(critical section)** 을 올바르게 보호하는 것입니다.  
두 대표 도구 **뮤텍스(Mutex)** 와 **세마포어(Semaphore)** 를 그림과 코드로 한 번에 정리합니다.

---

## 1. 핵심 개념 한 장 요약

### 1.1 임계구역(Critical Section)
공유 자원(공유 변수, 파일, 소켓 등)에 접근하는 코드 구간 → **동시에 1개 스레드/코루틴만** 들어가야 안전.

```text
[Thread A] -----> ┌──────── 임계구역 ────────┐ -----> (done)
[Thread B] -x- 대기 └─────────────────────────┘
```

### 1.2 뮤텍스 vs 세마포어

| 구분 | 뮤텍스 (Mutual Exclusion) | 세마포어 (Semaphore) |
|---|---|---|
| 카운트 | 1(이진 락) | 0 이상(카운팅) |
| 소유권 | **소유자 개념 O** (잠근 스레드만 해제) | **소유권 X** (누구나 release 가능) |
| 쓰임새 | 임계구역 **단독 진입 보장** | 동시 허용치(**N개**) 제어, 풀이용(커넥션/작업 슬롯) |
| 전형적 API | lock()/unlock() | acquire()/release() |

---

## 2 왜 동기화가 필요한가? (경쟁 조건 → 데이터 오염)

```kotlin
var counter = 0

fun race() = runBlocking {
    val jobs = List(1_000) {
        launch(Dispatchers.Default) {
            repeat(1_000) { counter++ }  // 임계구역: 읽고-증가-쓰기
        }
    }
    jobs.joinAll()
    println("counter = $counter") // 기대 1,000,000 이지만 매번 작게 나옴(경쟁 조건)
}
```
- 증상: 동시에 counter를 읽고 쓰면서 값이 유실(원자성/가시성 X).

---

## 3 뮤텍스로 임계구역 보호 (JVM 락 & 코루틴 Mutex)

### 3.1 JVM ReentrantLock

```kotlin
val lock = ReentrantLock()
var counter = 0

fun fixedWithReentrantLock() = runBlocking {
    val jobs = List(1_000) {
        launch(Dispatchers.Default) {
            repeat(1_000) {
                lock.lock()
                try { counter++ } finally { lock.unlock() }
            }
        }
    }
    jobs.joinAll()
    println("counter = $counter") // 1,000,000
}
```

### 3.2 `synchronized` 블록

```kotlin
val monitor = Any()
var counter = 0

fun fixedWithSynchronized() = runBlocking {
    val jobs = List(1_000) {
        launch(Dispatchers.Default) {
            repeat(1_000) {
                synchronized(monitor) { counter++ }
            }
        }
    }
    jobs.joinAll()
    println("counter = $counter")
}
```

### 3.3 코루틴 Mutex (suspend 친화)

```kotlin
val mutex = kotlinx.coroutines.sync.Mutex()
var counter = 0

fun fixedWithCoroutineMutex() = runBlocking {
    val jobs = List(1_000) {
        launch(Dispatchers.Default) {
            repeat(1_000) {
                mutex.lock()
                try { counter++ } finally { mutex.unlock() }
                // 또는: mutex.withLock { counter++ }
            }
        }
    }
    jobs.joinAll()
    println("counter = $counter")
}
```

---

## 4. 세마포어로 동시 허용치 제어 (슬롯/풀)

상황: 외부 API를 동시에 최대 10개까지만 호출하고 싶다.

```kotlin
val semaphore = java.util.concurrent.Semaphore(10) // 허용 동시성 10
suspend fun limitedCall(id: Int) {
    withContext(Dispatchers.IO) {
        semaphore.acquire() // 여유 슬롯 없으면 대기
        try {
            // 네트워크/IO 처리
            delay(200)
            println("done $id on ${Thread.currentThread().name}")
        } finally {
            semaphore.release()
        }
    }
}

fun demoSemaphore() = runBlocking {
    coroutineScope {
        (1..50).map { async { limitedCall(it) } }.awaitAll()
    }
}

// [슬롯 10개]  [■■■■■■■■■■]  ← acquire로 슬롯 소비 / release로 반환
// 요청 50개 → 동시에 최대 10개만 실행, 나머지는 대기 큐에서 대기
```

---

## 5. 교착상태(Deadlock)와 예방

### 5.1 교착상태란?

서로가 서로의 자원을 기다려 영원히 진행 불가인 상태.
```text
Thread A: lock1 → lock2 대기
Thread B: lock2 → lock1 대기
(서로가 서로를 기다리며 멈춤)
```

### 5.2 잘못된 코드

```kotlin
val lock1 = ReentrantLock()
val lock2 = ReentrantLock()

fun deadlockDemo() = runBlocking {
    val a = launch(Dispatchers.Default) {
        lock1.lock(); delay(50); lock2.lock()
        try { println("A done") } finally { lock2.unlock(); lock1.unlock() }
    }
    val b = launch(Dispatchers.Default) {
        lock2.lock(); delay(50); lock1.lock()
        try { println("B done") } finally { lock1.unlock(); lock2.unlock() }
    }
    withTimeoutOrNull(1_000) { joinAll(a, b) } ?: println("Deadlock detected")
}
```

### 5.3 예방 패턴

1. 고정된 락 순서(Ordering)
   - 모든 스레드가 lock1 → lock2 순서로만 잠그도록 규약.
   - ```kotlin
     fun <T> withBoth(l1: ReentrantLock, l2: ReentrantLock, block: () -> T): T {
        val (first, second) = if (System.identityHashCode(l1) < System.identityHashCode(l2)) l1 to l2 else l2 to l1
        first.lock(); try {
            second.lock(); try { return block() } finally { second.unlock() }
        } finally { first.unlock() }
     }
     
     ```
2. 타임아웃 있는 tryLock
    - 대기 시간을 제한해 교착을 회피하고 재시도/롤백 전략 적용.
    - ```kotlin
      fun safeTryLock(l1: ReentrantLock, l2: ReentrantLock): Boolean {
        if (!l1.tryLock(100, TimeUnit.MILLISECONDS)) return false
        if (!l2.tryLock(100, TimeUnit.MILLISECONDS)) { l1.unlock(); return false }
        return true
      }
      ```
3. 락 그레인(범위) 최소화 / 불필요한 중첩 회피
   - 임계구역을 최소 범위로 줄이고, I/O를 임계구역 밖으로 옮기기.

---

## 6. 생산자-소비자: 세마포어/큐로 back-pressure 만들기

```kotlin
val slots = Semaphore(5) // 버퍼 크기 5
val queue = ArrayDeque<Int>()
val monitor = Any()

fun producerConsumer() = runBlocking {
    val producer = launch(Dispatchers.Default) {
        (1..50).forEach { item ->
            slots.acquire()        // 버퍼 여유 없으면 생산자 대기
            synchronized(monitor) { queue.addLast(item) }
        }
    }
    val consumer = launch(Dispatchers.Default) {
        repeat(50) {
            val item = synchronized(monitor) { queue.removeFirst() }
            // 처리
            delay(30)
            slots.release()        // 버퍼 공간 반환 → 생산자 깨움
        }
    }
    joinAll(producer, consumer)
}
```

---

## 7. 임계구역을 하나만 통과 → 뮤텍스(ReentrantLock / synchronized / 코루틴 Mutex)

- 동시 허용치/풀 관리 → 세마포어(예: DB 커넥션, API 동시 호출 제한)
- 코루틴 환경 → Mutex, Channel, Semaphore(kotlinx)로 suspend 기반 설계
- 락 경합이 잦다 → 락 범위 최소화, 읽기-쓰기 분리, lock-free(원자 변수) 검토

---

## 8. 하이브리드: 세그먼트+페이지

일부 아키텍처/시대에는 **세그먼트로 큰 논리 영역을 잡고, 내부는 페이지로 관리**(외부 단편화↓, 보호/논리 경계 유지).  
하지만 현재 범용 데스크톱/서버의 메모리 관리 핵심은 **페이징**이며, 세그멘테이션은 대체로 **축소**되었습니다.

> 예) x86-64는 실질적으로 **페이징 중심**이며, 세그먼트는 FS/GS 기반 TLS 등 제한적으로만 사용.

---

## 9. 기타

```text
1. 뮤텍스 보호 흐름                         
[Request CS] -> [try lock] --성공--> [Critical Section] -> [unlock] -> [Done]
                         \--실패--> [Wait Queue] (스케줄러에 의해 대기)

2. 세마포어 슬롯 흐름                         
초기 카운트 N
acquire() → 카운트-1 (0 미만이면 대기)
release() → 카운트+1 (대기자 깨움)
```