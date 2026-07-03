---
title: (코틀린/Kotlin) Coroutines Structured Concurrency 완전 정리
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin Coroutines의 핵심 설계 철학인 Structured Concurrency(구조화된 동시성)의 원리와 Job 트리, 취소 전파, 예외 처리를 정리합니다.
---

---

## 1. Structured Concurrency란?

**구조화된 동시성**은 코루틴의 생명주기가 **부모-자식 관계의 트리 구조**로 관리되는 설계 철학입니다.

```
부모 코루틴이 끝나지 않으면 자식 코루틴도 끝나지 않은 것으로 간주
부모가 취소되면 모든 자식도 자동으로 취소됨
자식의 예외는 부모에게 전파됨
```

이 원칙 덕분에 **코루틴이 누수되거나 좀비처럼 떠도는 일이 없습니다.**

---

## 2. 구조화되지 않은 동시성의 문제

```kotlin
// GlobalScope: 구조화되지 않음 — 부모와 연결이 끊김
fun loadData() {
    GlobalScope.launch {
        delay(5000)
        println("데이터 로드 완료")  // 화면이 이미 닫혀도 계속 실행됨!
    }
}
```

- 호출한 함수가 끝나도 코루틴은 계속 실행
- 추적이 안 되어 메모리 누수, 불필요한 작업 지속
- 취소할 방법이 명확하지 않음

---

## 3. 구조화된 동시성의 기본 형태

```kotlin
suspend fun loadUserProfile() = coroutineScope {  // 새로운 Job 생성
    val user = async { fetchUser() }       // 자식 코루틴 1
    val posts = async { fetchPosts() }     // 자식 코루틴 2

    UserProfile(user.await(), posts.await())
    // coroutineScope는 모든 자식이 완료될 때까지 대기
}
```

```
coroutineScope (부모)
  ├── async { fetchUser() }   (자식1)
  └── async { fetchPosts() }  (자식2)

부모는 자식1, 자식2가 모두 끝나야 자신도 끝남
```

---

## 4. Job 트리와 취소 전파

### 부모 취소 → 모든 자식 취소

```kotlin
val parentJob = launch {
    launch {
        delay(1000)
        println("자식1 완료")  // 실행 안 됨
    }
    launch {
        delay(1000)
        println("자식2 완료")  // 실행 안 됨
    }
    delay(2000)
}

delay(100)
parentJob.cancel()  // 부모 취소 → 모든 자식도 즉시 취소
```

```
parentJob (취소됨)
  ├── 자식1 (자동 취소됨)
  └── 자식2 (자동 취소됨)
```

### 자식 예외 → 부모와 형제 모두 취소 (기본 Job)

```kotlin
coroutineScope {
    launch {
        delay(100)
        throw RuntimeException("자식1 실패")
    }
    launch {
        delay(1000)
        println("자식2 완료")  // 실행되지 않음! 자식1의 예외로 전체 취소
    }
}
```

```
자식1 예외 발생 → 부모에게 전파 → 부모가 취소 → 형제(자식2)도 취소
```

이것이 **기본 Job의 동작**입니다. 한 자식이 실패하면 전체가 실패로 간주됩니다.

---

## 5. SupervisorJob — 예외 전파 차단

자식의 실패가 다른 형제에게 영향을 주지 않도록 하려면 `supervisorScope`를 사용합니다.

```kotlin
supervisorScope {
    launch {
        delay(100)
        throw RuntimeException("자식1 실패")
        // 자식1만 실패, 부모와 형제는 영향 없음
    }
    launch {
        delay(1000)
        println("자식2 완료")  // 정상 실행됨!
    }
}
```

```
SupervisorJob
  ├── 자식1 (실패, 격리됨)
  └── 자식2 (영향 없이 정상 완료)
```

---

## 6. coroutineScope vs supervisorScope

| 구분 | coroutineScope | supervisorScope |
|------|---------------|-------------------|
| 자식 예외 전파 | 부모와 형제에게 전파 | 격리 (해당 자식만 실패) |
| 사용 시점 | 모든 자식 결과가 함께 필요할 때 | 독립적인 작업들을 병렬 실행할 때 |
| 예시 | 여러 API를 합쳐 화면 1개 구성 | 여러 위젯을 독립적으로 갱신 |

```kotlin
// coroutineScope: 모든 데이터가 필요한 화면 — 하나가 실패하면 전체 실패 처리
suspend fun loadDashboard() = coroutineScope {
    val user = async { fetchUser() }
    val stats = async { fetchStats() }
    Dashboard(user.await(), stats.await())
}

// supervisorScope: 독립적인 위젯들 — 하나 실패해도 나머지는 정상 표시
suspend fun loadWidgets() = supervisorScope {
    launch { loadWeatherWidget() }   // 실패해도 무관
    launch { loadNewsWidget() }      // 독립적으로 정상 작동
    launch { loadCalendarWidget() }
}
```

---

## 7. CoroutineScope 직접 생성 시 주의

```kotlin
// 위험: 직접 생성한 Scope는 자동으로 부모-자식 관계에 묶이지 않음
class MyRepository {
    private val scope = CoroutineScope(SupervisorJob() + Dispatchers.IO)

    fun startBackgroundWork() {
        scope.launch {
            // 이 Scope의 생명주기를 누군가 명시적으로 관리해야 함
        }
    }

    fun cleanup() {
        scope.cancel()  // 직접 정리해야 누수 방지
    }
}
```

ViewModel처럼 **생명주기가 자동 관리되는 Scope**(`viewModelScope`, `lifecycleScope`)를 우선 사용하고, 직접 Scope를 만든다면 반드시 정리 로직을 작성해야 합니다.

---

## 8. 구조화된 동시성의 핵심 보장사항

```kotlin
suspend fun structuredExample() = coroutineScope {
    println("시작")

    launch {
        delay(1000)
        println("자식 작업 완료")
    }

    println("끝")
    // ⚠️ "끝"이 출력되어도 함수는 자식이 끝날 때까지 반환하지 않음!
}

// 출력 순서:
// 시작
// 끝
// (1초 후)
// 자식 작업 완료
// → 그 후에야 structuredExample() 함수가 실제로 반환
```

`coroutineScope`는 **모든 자식이 완료되기 전까지 자기 자신도 완료되지 않습니다.**  
이것이 "구조화"의 핵심: 부모 함수가 끝났다는 것은 모든 자식 작업도 끝났다는 것을 의미합니다.

---

## 9. 실전 패턴 — 타임아웃과 구조화된 동시성

```kotlin
suspend fun loadWithTimeout(): Result<Data> = coroutineScope {
    try {
        withTimeout(5000) {
            val data = async { fetchData() }
            Result.success(data.await())
        }
    } catch (e: TimeoutCancellationException) {
        Result.failure(e)
        // withTimeout 내부의 모든 자식 코루틴도 함께 취소됨
    }
}
```

---

## 10. 정리

- **Structured Concurrency**: 코루틴이 부모-자식 트리로 관리되어 생명주기가 명확
- 부모 취소 → 모든 자식 자동 취소, 자식 예외 → 부모에게 전파 (기본 Job)
- `GlobalScope`는 이 구조에서 벗어나므로 가급적 사용 지양
- 독립적인 작업은 `supervisorScope`로 예외를 격리, 함께 필요한 작업은 `coroutineScope`로 묶기
- `coroutineScope`/`supervisorScope`는 **모든 자식이 끝나야 자신도 끝남**을 보장
