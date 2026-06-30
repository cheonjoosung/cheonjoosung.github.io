---
title: (Kotlin/코틀린) Coroutines Semaphore 완전 정리 — 동시 실행 수 제한
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin Coroutines의 Semaphore로 동시 실행 개수를 제한하는 원리와 flatMapMerge의 concurrency 파라미터, API 요청 제한 등 실전 패턴을 정리합니다.
---

---

## 1. Semaphore란?

`kotlinx.coroutines.sync.Semaphore`는 **동시에 접근할 수 있는 코루틴 수를 제한**하는 동기화 도구입니다.  
`Mutex`가 동시 접근을 1개로 제한하는 반면, `Semaphore`는 N개까지 허용합니다.

```kotlin
import kotlinx.coroutines.sync.Semaphore
import kotlinx.coroutines.sync.withPermit

val semaphore = Semaphore(permits = 3)  // 동시에 3개까지 허용

suspend fun limitedTask(id: Int) {
    semaphore.withPermit {
        println("작업 시작: $id")
        delay(1000)
        println("작업 완료: $id")
    }
}
```

---

## 2. 왜 필요한가?

```
서버에 동시 요청 100개 전송 → 서버 과부하, Rate Limit 초과
파일 100개 동시 다운로드 → 메모리/네트워크 자원 고갈
DB 커넥션 풀 — 최대 10개 연결만 허용
```

무제한 병렬 실행은 자원을 고갈시킬 수 있습니다. Semaphore로 **동시 실행 수의 상한선**을 설정합니다.

---

## 3. 동작 원리

```kotlin
val semaphore = Semaphore(2)  // 허가(permit) 2개

suspend fun process(id: Int) {
    semaphore.withPermit {
        println("$id 시작")
        delay(500)
        println("$id 완료")
    }
}

coroutineScope {
    repeat(5) { id ->
        launch { process(id) }
    }
}

// 출력 (동시 2개씩 처리):
// 0 시작
// 1 시작
// (500ms 후)
// 0 완료
// 2 시작
// 1 완료
// 3 시작
// ...
```

```
요청: [0][1][2][3][4]
Semaphore(2):
  슬롯1: [0]----[2]----[4]
  슬롯2: [1]----[3]
```

---

## 4. acquire/release 수동 처리 vs withPermit

```kotlin
// 권장: withPermit (try-finally로 release 보장)
suspend fun safeTask() {
    semaphore.withPermit {
        doWork()
    }
}

// 비권장: 수동 처리 (예외 시 release 누락 위험)
suspend fun unsafeTask() {
    semaphore.acquire()
    try {
        doWork()
    } finally {
        semaphore.release()
    }
}
```

---

## 5. flatMapMerge의 concurrency와 관계

Flow의 `flatMapMerge(concurrency = N)`은 내부적으로 Semaphore와 유사한 방식으로 동시 실행 수를 제한합니다.

```kotlin
// flatMapMerge concurrency: 내장된 동시성 제한
userIds.asFlow()
    .flatMapMerge(concurrency = 3) { id ->
        flow { emit(api.fetchUser(id)) }
    }
    .collect { user -> processUser(user) }
```

```kotlin
// Semaphore로 직접 구현 (더 세밀한 제어 가능)
val semaphore = Semaphore(3)

suspend fun fetchAllUsers(ids: List<String>): List<User> = coroutineScope {
    ids.map { id ->
        async {
            semaphore.withPermit {
                api.fetchUser(id)
            }
        }
    }.awaitAll()
}
```

`flatMapMerge`는 Flow 체인 안에서 간결하지만, Semaphore는 **Flow가 아닌 일반 코루틴 코드**에서도 사용할 수 있어 더 범용적입니다.

---

## 6. 실전 패턴 — API 동시 요청 제한

```kotlin
class ImageDownloader(
    private val api: ImageApi,
    maxConcurrent: Int = 4
) {
    private val semaphore = Semaphore(maxConcurrent)

    suspend fun downloadAll(urls: List<String>): List<Bitmap> = coroutineScope {
        urls.map { url ->
            async {
                semaphore.withPermit {
                    api.download(url)
                }
            }
        }.awaitAll()
    }
}
```

서버에 100개 요청을 동시에 보내는 대신, 항상 최대 4개만 동시에 진행되도록 제한합니다.

---

## 7. 실전 패턴 — DB 커넥션 풀 시뮬레이션

```kotlin
class ConnectionPool(maxConnections: Int = 5) {
    private val semaphore = Semaphore(maxConnections)

    suspend fun <T> withConnection(block: suspend (Connection) -> T): T {
        return semaphore.withPermit {
            val connection = acquireConnection()
            try {
                block(connection)
            } finally {
                releaseConnection(connection)
            }
        }
    }
}

// 사용
repeat(100) { id ->
    launch {
        connectionPool.withConnection { conn ->
            conn.query("SELECT * FROM users WHERE id = $id")
        }
    }
}
// 최대 5개 커넥션만 동시에 사용됨
```

---

## 8. availablePermits로 현재 상태 확인

```kotlin
val semaphore = Semaphore(3)

println(semaphore.availablePermits)  // 3 (아직 사용 안 됨)

semaphore.acquire()
println(semaphore.availablePermits)  // 2

semaphore.release()
println(semaphore.availablePermits)  // 3
```

모니터링이나 디버깅 목적으로 현재 사용 가능한 허가 수를 확인할 수 있습니다.

---

## 9. tryAcquire — 논블로킹 시도

```kotlin
suspend fun tryProcessIfAvailable(): Boolean {
    return if (semaphore.tryAcquire()) {
        try {
            process()
            true
        } finally {
            semaphore.release()
        }
    } else {
        false  // 모든 슬롯이 사용 중이면 즉시 포기
    }
}
```

---

## 10. Mutex vs Semaphore 비교

| 구분 | Mutex | Semaphore(1) | Semaphore(N) |
|------|-------|--------------|--------------|
| 동시 접근 허용 수 | 1 | 1 | N |
| 내부 구현 | Semaphore(1) 기반 | - | - |
| 사용 목적 | 단일 자원 보호 | 단일 자원 보호 | 동시 실행 수 제한 |

```kotlin
// Mutex는 사실 Semaphore(1)의 특수한 경우
val mutex = Mutex()
val equivalentSemaphore = Semaphore(1)
// 동작은 거의 동일
```

---

## 11. 정리

- **`Semaphore(N)`**: 동시에 N개의 코루틴만 특정 자원/작업에 접근 허용
- `withPermit`을 사용해 예외 상황에도 `release` 보장
- API 동시 요청 제한, 다운로드 동시 개수 제한, DB 커넥션 풀 시뮬레이션에 활용
- Flow 체인 내부라면 `flatMapMerge(concurrency)`가 더 간결, 일반 코루틴 코드라면 `Semaphore`가 더 범용적
