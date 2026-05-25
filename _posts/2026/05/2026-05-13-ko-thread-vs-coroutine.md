---
title: (Kotlin/코틀린) 스레드 vs 코루틴 완전 비교 - 동작 원리부터 실전 전환까지
tags: [ Kotlin, Android ]
style: fill
color: dark
description: Java/Kotlin 스레드와 코루틴의 동작 원리, 생성 비용, 메모리, 컨텍스트 스위칭, 에러 처리, 취소, 코드 가독성까지 실전 코드와 함께 깊이 있게 비교합니다.
---

## 개요

- **스레드(Thread)** 와 **코루틴(Coroutine)** 을 깊이 있게 비교합니다.
- 단순 기능 비교가 아니라 **내부 동작 원리 차이**까지 다룹니다.
- 이 글에서는 다음을 설명합니다.
  - 스레드와 코루틴의 동작 원리
  - 생성 비용·메모리·컨텍스트 스위칭 비교
  - 동시성 처리·에러 처리·취소 처리 비교
  - 코드 가독성 비교 — 콜백 지옥 → 코루틴
  - Android 실전 전환 예제

---

## 1. 스레드(Thread)란

스레드는 **프로세스 안에서 독립적으로 실행되는 작업 단위**입니다. OS가 직접 관리합니다.

```
프로세스
├── 메인 스레드  (UI)
├── 스레드 A     (네트워크)
├── 스레드 B     (파일 I/O)
└── 스레드 C     (계산)
    ↑
    OS 스케줄러가 CPU 시간 분배
```

```kotlin
// 스레드 생성과 실행
val thread = Thread {
    println("스레드 실행: ${Thread.currentThread().name}")
    Thread.sleep(1000)   // 스레드 전체 블로킹
    println("1초 후")
}
thread.start()
thread.join()   // 완료까지 현재 스레드 블로킹
```

### 스레드 풀 (Thread Pool)

매번 스레드를 새로 만드는 비용을 줄이기 위해 **미리 만들어 둔 스레드를 재사용**합니다.

```kotlin
// Executors로 스레드 풀 생성
val executor = Executors.newFixedThreadPool(4)

executor.submit {
    println("풀의 스레드에서 실행")
}

executor.shutdown()
```

---

## 2. 코루틴(Coroutine)이란

코루틴은 **스레드 위에서 동작하는 경량 실행 단위**입니다. OS가 아닌 **코루틴 런타임**이 관리합니다.

```
스레드 1개
├── 코루틴 A  → 중단(네트워크 대기) → 재개
├── 코루틴 B  → 중단(DB 대기)      → 재개
├── 코루틴 C  → 중단(delay)        → 재개
└── 코루틴 D  → 실행 중
    ↑
    코루틴 런타임이 스레드 위에서 스케줄링
```

```kotlin
// 코루틴 생성과 실행
val scope = CoroutineScope(Dispatchers.IO)
val job = scope.launch {
    println("코루틴 실행: ${Thread.currentThread().name}")
    delay(1000)   // 코루틴만 중단, 스레드는 다른 코루틴 처리
    println("1초 후")
}
job.join()
```

---

## 3. 동작 원리 비교

### 스레드의 블로킹

스레드가 `sleep()` 또는 I/O 대기 중일 때 **스레드 전체가 멈춥니다**. CPU는 이 스레드를 쉬게 두고 다른 스레드에 CPU를 할당합니다.

```
시간 →
스레드 A: [실행][실행][대기(sleep)][대기][대기][실행]
                       ↑
                  스레드 블로킹 — CPU 낭비
```

### 코루틴의 중단

코루틴이 `delay()` 또는 `suspend fun` 호출 중일 때 **코루틴만 멈추고 스레드는 다른 코루틴을 실행**합니다.

```
시간 →
스레드 A: [코루틴1 실행][코루틴2 실행][코루틴3 실행][코루틴1 재개]
                         ↑
              코루틴1이 대기 중인 동안 스레드는 다른 코루틴 처리
```

```kotlin
// 스레드 — 3개 병렬 작업에 스레드 3개 필요
repeat(3) {
    Thread {
        Thread.sleep(1000)   // 스레드 3개가 각각 블로킹
        println("완료 $it")
    }.start()
}

// 코루틴 — 3개 병렬 작업에 스레드 1개로 가능
repeat(3) {
    scope.launch {
        delay(1000)          // 스레드 1개가 3개 코루틴 모두 처리
        println("완료 $it")
    }
}
```

---

## 4. 생성 비용 비교

### 스레드 생성 비용

스레드는 OS 자원을 직접 사용합니다. 생성에 **수 밀리초**, 기본 스택 메모리 **512KB ~ 1MB** 가 필요합니다.

```kotlin
// 스레드 10만 개 생성 시도 → OutOfMemoryError 또는 수 초 소요
val threads = List(100_000) {
    Thread {
        Thread.sleep(1000)
    }
}
threads.forEach { it.start() }
// 실제로 이 코드는 시스템 한계에 의해 실패합니다
```

### 코루틴 생성 비용

코루틴은 **Kotlin 객체(Continuation)** 로 구현됩니다. 생성에 **수 마이크로초**, 메모리는 수백 바이트 수준입니다.

```kotlin
// 코루틴 10만 개 — 문제 없이 동작
val scope = CoroutineScope(Dispatchers.Default)

val jobs = List(100_000) {
    scope.launch {
        delay(1000)
    }
}
jobs.forEach { it.join() }
// 약 1초 후 모두 완료 — 메모리 여유
```

| 항목 | 스레드 | 코루틴 |
|------|--------|--------|
| 생성 시간 | ~1ms | ~1μs (1000배 빠름) |
| 기본 스택 메모리 | 512KB~1MB | 수백 바이트 |
| 동시 실행 가능 수 | 수백~수천 | 수만~수십만 |
| 관리 주체 | OS 커널 | 코루틴 런타임 |

---

## 5. 컨텍스트 스위칭 비교

### 스레드의 컨텍스트 스위칭

스레드 전환 시 OS가 **레지스터 상태, 스택 포인터, PC(Program Counter)** 를 저장/복원합니다. 비용이 큽니다.

```
스레드 A → 스레드 B 전환 시:
1. 스레드 A의 CPU 레지스터, 스택 포인터 저장 (커널 메모리)
2. 스레드 B의 CPU 레지스터, 스택 포인터 복원
3. CPU 실행 재개
→ 수 마이크로초 ~ 수십 마이크로초 소요
```

### 코루틴의 컨텍스트 스위칭

코루틴 전환 시 **Continuation 객체(힙 메모리)** 를 저장/복원합니다. OS 개입 없이 JVM 위에서 처리됩니다.

```
코루틴 A → 코루틴 B 전환 시:
1. 코루틴 A의 실행 지점, 로컬 변수를 Continuation 객체에 저장 (힙)
2. 코루틴 B의 Continuation에서 실행 지점 복원
3. 같은 스레드에서 계속 실행
→ 수십 나노초 수준 — OS 개입 없음
```

```kotlin
// Continuation — suspend fun 컴파일 결과 (개념적 표현)
// suspend fun fetchUser(userId: Long): User { ... }
// 컴파일러가 아래와 같이 변환
fun fetchUser(userId: Long, continuation: Continuation<User>): Any {
    // 중단 지점마다 상태(label)를 저장
    // 재개 시 label로 이어서 실행
}
```

---

## 6. 메모리 사용량 비교

```kotlin
import kotlinx.coroutines.*
import kotlin.system.measureTimeMillis

fun main() {
    val runtime = Runtime.getRuntime()

    // 스레드 1만 개 메모리 측정
    val beforeThread = runtime.totalMemory() - runtime.freeMemory()
    val threads = List(10_000) {
        Thread { Thread.sleep(10_000) }.also { it.start() }
    }
    val afterThread = runtime.totalMemory() - runtime.freeMemory()
    println("스레드 1만 개 메모리: ${(afterThread - beforeThread) / 1024 / 1024} MB")
    // 결과: 약 5,000MB (스레드당 약 512KB)
    threads.forEach { it.interrupt() }

    // 코루틴 1만 개 메모리 측정
    val scope = CoroutineScope(Dispatchers.Default)
    val beforeCoroutine = runtime.totalMemory() - runtime.freeMemory()
    val jobs = List(10_000) {
        scope.launch { delay(10_000) }
    }
    val afterCoroutine = runtime.totalMemory() - runtime.freeMemory()
    println("코루틴 1만 개 메모리: ${(afterCoroutine - beforeCoroutine) / 1024 / 1024} MB")
    // 결과: 약 10MB (코루틴당 약 1KB)
    scope.cancel()
}
```

---

## 7. 동시성 처리 비교

### 스레드 — synchronized로 공유 자원 보호

```kotlin
var counter = 0
val lock = Any()

val threads = List(1000) {
    Thread {
        repeat(1000) {
            synchronized(lock) {   // 락 획득 → 실행 → 락 해제
                counter++
            }
        }
    }
}
threads.forEach { it.start() }
threads.forEach { it.join() }
println(counter)   // 1,000,000
```

### 코루틴 — Mutex로 공유 자원 보호

```kotlin
var counter = 0
val mutex = Mutex()

val scope = CoroutineScope(Dispatchers.Default)
val jobs = List(1000) {
    scope.launch {
        repeat(1000) {
            mutex.withLock {   // 코루틴 중단으로 대기 — 스레드 블로킹 없음
                counter++
            }
        }
    }
}
jobs.forEach { it.join() }
println(counter)   // 1,000,000
```

### 코루틴 — Atomic으로 더 간단하게

```kotlin
val counter = AtomicInteger(0)

val scope = CoroutineScope(Dispatchers.Default)
val jobs = List(1000) {
    scope.launch {
        repeat(1000) {
            counter.incrementAndGet()   // 원자적 연산 — 락 불필요
        }
    }
}
jobs.forEach { it.join() }
println(counter.get())   // 1,000,000
```

| 동기화 방법 | 스레드 블로킹 | 코루틴 중단 | 사용 |
|------------|-------------|------------|------|
| `synchronized` | O | X | 스레드 |
| `ReentrantLock` | O | X | 스레드 |
| `Mutex` | X | O | 코루틴 |
| `AtomicXxx` | X | X | 스레드·코루틴 모두 |

---

## 8. 에러 처리 비교

### 스레드의 에러 처리

```kotlin
// ❌ 스레드에서 발생한 예외는 외부에서 잡기 어렵다
val thread = Thread {
    throw RuntimeException("스레드 내부 오류")
}
thread.start()

try {
    thread.join()
} catch (e: Exception) {
    // ❌ 이 catch는 스레드 내부 예외를 잡지 못함
    println("잡힘: $e")
}

// ✅ UncaughtExceptionHandler로 처리
thread.setUncaughtExceptionHandler { t, e ->
    println("스레드 ${t.name} 에서 예외: ${e.message}")
}
```

### 코루틴의 에러 처리

```kotlin
// ✅ try-catch로 자연스럽게 처리
val scope = CoroutineScope(Dispatchers.IO)

scope.launch {
    try {
        val user = fetchUser()
        processUser(user)
    } catch (e: NetworkException) {
        println("네트워크 오류: ${e.message}")
    } catch (e: Exception) {
        println("예외 발생: ${e.message}")
    }
}

// ✅ CoroutineExceptionHandler — 최상위 예외 핸들러
val handler = CoroutineExceptionHandler { _, throwable ->
    println("처리되지 않은 예외: ${throwable.message}")
}

val scope2 = CoroutineScope(Dispatchers.IO + handler)
scope2.launch {
    throw RuntimeException("코루틴 내부 오류")
    // handler가 처리
}

// ✅ Result + runCatching으로 함수형 에러 처리
scope.launch {
    val result = runCatching { fetchUser() }
    result
        .onSuccess { user -> println("성공: ${user.name}") }
        .onFailure { e -> println("실패: ${e.message}") }
}
```

---

## 9. 취소 처리 비교

### 스레드의 취소 — 강제 취소 불가

```kotlin
// 스레드는 외부에서 강제 종료할 방법이 없다
val thread = Thread {
    while (true) {            // 무한루프
        Thread.sleep(100)
        println("실행 중")
    }
}
thread.start()

// thread.stop()  →  deprecated, 데이터 오염 위험
// thread.interrupt()  →  협력적 취소 (무시 가능)
thread.interrupt()

// interrupt 적용 코드 (협력 필요)
val thread2 = Thread {
    while (!Thread.currentThread().isInterrupted) {   // 직접 확인
        try {
            Thread.sleep(100)
            println("실행 중")
        } catch (e: InterruptedException) {
            println("인터럽트 발생 — 종료")
            break
        }
    }
}
```

### 코루틴의 취소 — 구조화된 취소

```kotlin
val scope = CoroutineScope(Dispatchers.IO)

val job = scope.launch {
    repeat(1000) { i ->
        // suspend 함수 호출 시점마다 취소 여부 자동 확인
        delay(100)
        println("실행 중 $i")
    }
}

delay(350)
job.cancel()   // 취소 요청
job.join()     // 완료 대기
println("코루틴 취소 완료")

// 취소 불가능한 블로킹 코드에서 수동으로 확인
val job2 = scope.launch(Dispatchers.Default) {
    repeat(1000) { i ->
        if (!isActive) return@launch   // 취소 확인
        // CPU 집중 작업 (delay 없음)
        heavyComputation()
        println("계산 완료 $i")
    }
}
```

### 부모-자식 취소 전파

```kotlin
val parentJob = scope.launch {
    val child1 = launch {
        delay(1000)
        println("자식1 완료")
    }
    val child2 = launch {
        delay(2000)
        println("자식2 완료")
    }
}

delay(500)
parentJob.cancel()   // 부모 취소 → 자식1, 자식2 모두 취소
// "자식1 완료", "자식2 완료" 출력 안 됨
```

| 항목 | 스레드 | 코루틴 |
|------|--------|--------|
| 취소 방법 | `interrupt()` (협력 필요) | `cancel()` (자동 전파) |
| 취소 확인 | `isInterrupted` 직접 확인 | suspend 함수에서 자동 확인 |
| 자식 취소 | 직접 관리 필요 | 부모 취소 시 자동 취소 |
| 리소스 정리 | `finally` | `finally` + `NonCancellable` |

---

## 10. 코드 가독성 비교

비동기 작업이 중첩될수록 스레드 방식은 **콜백 지옥**이 발생합니다.

### 스레드 방식 — 콜백 중첩

```kotlin
// 로그인 → 유저 정보 조회 → 피드 로드 → UI 업데이트
fun loadFeed(userId: Long) {
    Thread {
        try {
            val token = api.login(userId)             // 블로킹

            Thread {
                try {
                    val user = api.getUser(token)     // 블로킹

                    Thread {
                        try {
                            val feed = api.getFeed(user.id)  // 블로킹

                            runOnUiThread {
                                showFeed(feed)        // UI 업데이트
                            }
                        } catch (e: Exception) {
                            runOnUiThread { showError(e) }
                        }
                    }.start()
                } catch (e: Exception) {
                    runOnUiThread { showError(e) }
                }
            }.start()
        } catch (e: Exception) {
            runOnUiThread { showError(e) }
        }
    }.start()
}
```

### 코루틴 방식 — 순차적 코드

```kotlin
// 동일한 작업 — 읽기 쉽고 에러 처리도 한 곳에
fun loadFeed(userId: Long) {
    viewModelScope.launch {
        try {
            val token = api.login(userId)        // 중단
            val user  = api.getUser(token)       // 중단
            val feed  = api.getFeed(user.id)     // 중단
            showFeed(feed)                       // 메인 스레드
        } catch (e: Exception) {
            showError(e)                         // 한 곳에서 처리
        }
    }
}
```

---

## 11. 실전 전환 예제 — Thread → Coroutine

### 예제 ① 단순 비동기 작업

```kotlin
// Before — Thread
fun fetchAndShow(userId: Long) {
    Thread {
        val user = repository.getUser(userId)   // 블로킹
        runOnUiThread {
            binding.tvName.text = user.name
        }
    }.start()
}

// After — Coroutine
fun fetchAndShow(userId: Long) {
    lifecycleScope.launch {
        val user = repository.getUser(userId)   // IO 스레드에서 중단
        binding.tvName.text = user.name         // 메인 스레드로 자동 복귀
    }
}
```

---

### 예제 ② 병렬 실행

```kotlin
// Before — Thread (CountDownLatch로 동기화)
fun loadDashboard(userId: Long) {
    val latch = CountDownLatch(2)
    var user: User? = null
    var posts: List<Post>? = null

    Thread {
        user = repository.getUser(userId)
        latch.countDown()
    }.start()

    Thread {
        posts = repository.getPosts(userId)
        latch.countDown()
    }.start()

    Thread {
        latch.await()   // 둘 다 완료까지 블로킹
        runOnUiThread {
            showDashboard(user!!, posts!!)
        }
    }.start()
}

// After — Coroutine (async 병렬 처리)
fun loadDashboard(userId: Long) {
    viewModelScope.launch {
        val (user, posts) = coroutineScope {
            val u = async { repository.getUser(userId) }
            val p = async { repository.getPosts(userId) }
            Pair(u.await(), p.await())
        }
        showDashboard(user, posts)
    }
}
```

---

### 예제 ③ 타임아웃 처리

```kotlin
// Before — Thread (Future + ExecutorService)
fun fetchWithTimeout(userId: Long): User? {
    val executor = Executors.newSingleThreadExecutor()
    val future = executor.submit<User> {
        repository.getUser(userId)   // 블로킹
    }
    return try {
        future.get(3, TimeUnit.SECONDS)   // 3초 타임아웃
    } catch (e: TimeoutException) {
        future.cancel(true)
        null
    } finally {
        executor.shutdown()
    }
}

// After — Coroutine (withTimeout)
suspend fun fetchWithTimeout(userId: Long): User? {
    return try {
        withTimeout(3_000L) {
            repository.getUser(userId)
        }
    } catch (e: TimeoutCancellationException) {
        null
    }
}
```

---

### 예제 ④ 재시도 로직

```kotlin
// Before — Thread (반복문 + sleep)
fun fetchWithRetry(userId: Long, maxRetry: Int = 3): User? {
    repeat(maxRetry) { attempt ->
        try {
            return repository.getUser(userId)
        } catch (e: Exception) {
            if (attempt == maxRetry - 1) return null
            Thread.sleep(1000L * (attempt + 1))   // 스레드 블로킹
        }
    }
    return null
}

// After — Coroutine (Flow retry 또는 반복문)
suspend fun fetchWithRetry(userId: Long, maxRetry: Int = 3): User? {
    repeat(maxRetry) { attempt ->
        try {
            return repository.getUser(userId)
        } catch (e: Exception) {
            if (attempt == maxRetry - 1) return null
            delay(1000L * (attempt + 1))   // 코루틴만 중단
        }
    }
    return null
}
```

---

## 12. 스레드를 써야 하는 경우

코루틴이 항상 답은 아닙니다. **다음 경우에는 스레드(또는 스레드 풀)가 더 적합합니다.**

| 상황 | 이유 |
|------|------|
| JNI / Native 라이브러리 연동 | 코루틴 중단이 네이티브 레이어와 호환되지 않음 |
| Java 라이브러리가 스레드 기반 블로킹 API만 제공 | `Dispatchers.IO`로 감싸서 코루틴과 함께 사용 |
| 실시간 오디오·카메라 (타이밍 중요) | OS 스케줄러 직접 제어 필요 |
| 스레드 우선순위 세밀 제어 | `Thread.setPriority()` 필요 시 |

```kotlin
// 블로킹 Java 라이브러리 → Dispatchers.IO로 감싸기
suspend fun readLegacyFile(path: String): String {
    return withContext(Dispatchers.IO) {   // IO 스레드에서 블로킹 허용
        LegacyFileReader.read(path)        // 블로킹 Java API
    }
}
```

---

## 13. 전체 비교 요약

| 항목 | 스레드 | 코루틴 |
|------|--------|--------|
| 관리 주체 | OS 커널 | 코루틴 런타임 (JVM) |
| 생성 비용 | 높음 (1~수ms) | 낮음 (수μs) |
| 메모리 | 512KB ~ 1MB / 개 | 수백 바이트 / 개 |
| 동시 실행 수 | 수백~수천 | 수만~수십만 |
| 블로킹 시 | 스레드 전체 대기 | 해당 코루틴만 중단 |
| 컨텍스트 전환 | OS 개입, 수μs~수십μs | JVM, 수십ns |
| 동기화 | `synchronized`, `Lock` | `Mutex`, `Atomic` |
| 에러 처리 | `UncaughtExceptionHandler` | `try-catch`, `CoroutineExceptionHandler` |
| 취소 | `interrupt()` (협력 필요) | `cancel()` (자동 전파) |
| 코드 가독성 | 콜백 중첩 | 순차적 코드 |
| 생명주기 관리 | 수동 | `viewModelScope` 자동 |

---

## 14. 정리

```
I/O 많은 작업 (네트워크, DB, 파일)
    → 코루틴 — 스레드 낭비 없이 동시 처리

CPU 집중 작업 (이미지 처리, 암호화)
    → 코루틴 + Dispatchers.Default — 스레드 풀 활용

JNI / 블로킹 Java API
    → 코루틴 + Dispatchers.IO — 블로킹 격리

실시간 타이밍 중요 (오디오, 카메라)
    → 전용 스레드 + 우선순위 설정
```

- 코루틴은 스레드를 **대체하는 것이 아니라 스레드 위에서 동작**합니다.
- 코루틴의 핵심 이점은 **"중단해도 스레드를 낭비하지 않는다"** 는 것입니다.
- Android 개발에서는 `viewModelScope` + 코루틴이 현재 가장 권장되는 방식입니다.

---

## 참고

- [Kotlin Coroutines 공식 문서](https://kotlinlang.org/docs/coroutines-overview.html)
- [Android Coroutines 가이드](https://developer.android.com/kotlin/coroutines)
- [Coroutines 성능 분석 — JetBrains Blog](https://blog.jetbrains.com/kotlin/2021/01/kotlin-coroutines-1-4-2-released/)
- [Java Thread vs Kotlin Coroutine — 공식 비교](https://kotlinlang.org/docs/coroutines-and-threads.html)
