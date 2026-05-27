---
title: (Kotlin/코틀린) Coroutines 취소 — cancel(), isActive, CancellationException
tags: [ Kotlin ]
style: fill
color: dark
description: 코루틴 취소의 동작 원리, cancel()과 isActive, CancellationException 처리, 취소 불가 작업 처리, Android 실전 패턴까지 깊이 있게 정리합니다.
---

## 개요

- Kotlin Coroutines의 **취소(Cancellation)** 메커니즘을 다룹니다.
- 코루틴은 협력적 취소(Cooperative Cancellation) 방식으로 동작합니다.
- 이 글에서는 다음을 설명합니다.
  - `cancel()`의 동작 원리
  - `isActive`와 `ensureActive()`로 취소 지점 만들기
  - `CancellationException` 처리
  - `withContext(NonCancellable)` — 취소 불가 작업
  - `finally` 블록에서 리소스 정리
  - Android 실전 패턴

---

## 1. 코루틴 취소의 기본 원리

### 협력적 취소 (Cooperative Cancellation)

코루틴은 **스스로 취소 신호를 확인해야** 멈춥니다. 취소는 자동으로 즉시 중단되지 않습니다.

```kotlin
// cancel()은 즉시 중단이 아님 — 취소 신호를 보낼 뿐
val job = launch {
    repeat(1000) { i ->
        println("작업 $i")
        Thread.sleep(100)   // ❌ 취소 신호를 확인하지 않는 블로킹 코드
    }
}
delay(500)
job.cancel()    // 취소 신호 전송
job.join()      // 완료 대기
println("종료")
// → 작업이 취소되지 않고 계속 실행됨
```

```kotlin
// ✅ suspend 함수는 취소 신호를 자동으로 확인
val job = launch {
    repeat(1000) { i ->
        println("작업 $i")
        delay(100)  // delay는 취소 가능한 suspend 함수 — 취소 시 즉시 중단
    }
}
delay(500)
job.cancel()
job.join()
println("종료")
// → 약 5번 실행 후 중단됨
```

**취소 가능한 suspend 함수들:**

```
delay(), yield(), withContext(), withTimeout()
IO 관련: suspendCancellableCoroutine
Retrofit/Room suspend 함수 모두 포함
```

---

## 2. isActive — 취소 신호 수동 확인

CPU 집약적 작업처럼 suspend 함수가 없는 루프에서는 `isActive`로 직접 확인합니다.

```kotlin
val job = launch(Dispatchers.Default) {
    var count = 0L
    // 취소 신호를 수동으로 확인
    while (isActive) {          // ✅ 취소 시 루프 종료
        count++
        if (count % 1_000_000L == 0L) println("count: $count")
    }
    println("코루틴 종료 — count: $count")
}

delay(1000)
job.cancel()
job.join()
println("완료")
```

```kotlin
// isActive 대신 ensureActive() — 취소 시 CancellationException 발생
val job = launch(Dispatchers.Default) {
    repeat(Int.MAX_VALUE) { i ->
        ensureActive()          // 취소 상태면 CancellationException throw
        heavyComputation(i)
    }
}
```

### isActive vs ensureActive() 비교

```kotlin
// isActive — 취소 여부를 Boolean으로 반환, 직접 처리
if (!isActive) return@launch

// ensureActive() — 취소 시 CancellationException을 throw
// 중간 정리 로직 없이 빠르게 빠져나갈 때 적합
ensureActive()
```

---

## 3. CancellationException

`cancel()`은 내부적으로 `CancellationException`을 발생시킵니다.

```kotlin
val job = launch {
    try {
        delay(1000)
        println("이 줄은 실행 안 됨")
    } catch (e: CancellationException) {
        println("취소됨: ${e.message}")
        throw e  // ✅ 반드시 재던져야 취소가 정상 전파됨
    }
}

delay(100)
job.cancel(CancellationException("사용자가 취소"))
job.join()
```

### ❌ CancellationException을 삼키면 취소가 전파되지 않음

```kotlin
val job = launch {
    try {
        delay(1000)
    } catch (e: Exception) {
        // ❌ CancellationException까지 잡아버림 — 취소 무시
        println("예외: $e")
    }
    println("취소됐는데도 계속 실행됨")  // 실행됨!
}

delay(100)
job.cancel()
```

### ✅ CancellationException만 구분해서 처리

```kotlin
val job = launch {
    try {
        delay(1000)
    } catch (e: CancellationException) {
        throw e                          // 취소는 재전파
    } catch (e: IOException) {
        println("네트워크 오류: $e")      // 그 외 예외만 처리
    }
}

// 또는 — catch 이후 isActive 확인
val job2 = launch {
    try {
        riskyOperation()
    } catch (e: Exception) {
        if (e is CancellationException) throw e   // 취소 재전파
        handleError(e)
    }
}
```

---

## 4. cancel() 상세

```kotlin
// 기본 취소
job.cancel()

// 메시지와 함께 취소
job.cancel(CancellationException("화면에서 벗어남"))

// 취소 후 완료 대기 — cancelAndJoin()
job.cancelAndJoin()  // job.cancel() + job.join() 축약

// 특정 자식만 취소
val parent = launch {
    val child1 = launch { delay(1000); println("child1") }
    val child2 = launch { delay(2000); println("child2") }

    child1.cancel()  // child1만 취소
    // child2는 계속 실행
}
```

---

## 5. finally — 리소스 정리

코루틴이 취소될 때도 `finally`는 실행됩니다.

```kotlin
val job = launch {
    val connection = openDatabaseConnection()
    try {
        processData(connection)
    } finally {
        // 취소 시에도 반드시 실행 — 리소스 해제
        connection.close()
        println("연결 종료")
    }
}

delay(100)
job.cancel()
job.join()
// → "연결 종료" 출력됨
```

### use() — AutoCloseable 자동 정리

```kotlin
val job = launch {
    FileReader("data.txt").use { reader ->
        // 취소 시 자동으로 reader.close() 호출
        while (isActive) {
            val line = reader.readLine() ?: break
            processLine(line)
        }
    }
}
```

---

## 6. withContext(NonCancellable) — 취소 불가 작업

`finally`에서 suspend 함수를 호출하거나, 반드시 완료돼야 하는 작업에 사용합니다.

```kotlin
val job = launch {
    try {
        delay(1000)
        println("작업 완료")
    } finally {
        // ❌ 취소 상태에서 delay() 호출 — 즉시 CancellationException 발생
        delay(100)  // 실행 안 됨

        // ✅ NonCancellable — 취소 상태에서도 suspend 함수 실행 가능
        withContext(NonCancellable) {
            println("정리 작업 시작")
            delay(100)           // 여기선 정상 실행
            saveToDatabase()     // 반드시 완료해야 하는 DB 저장
            println("정리 완료")
        }
    }
}

delay(100)
job.cancel()
job.join()
// → "정리 작업 시작" → "정리 완료" 출력됨
```

> `NonCancellable`은 `finally` 블록 안에서만 사용하세요. 일반 로직에 사용하면 취소가 불가능한 코루틴이 됩니다.

---

## 7. withTimeout / withTimeoutOrNull

시간 초과 시 자동으로 취소합니다.

```kotlin
// withTimeout — 시간 초과 시 TimeoutCancellationException (CancellationException 서브클래스)
try {
    val result = withTimeout(3000L) {
        fetchDataFromNetwork()  // 3초 안에 완료 안 되면 취소
    }
    println("결과: $result")
} catch (e: TimeoutCancellationException) {
    println("시간 초과")
}

// withTimeoutOrNull — 시간 초과 시 null 반환 (예외 없음)
val result = withTimeoutOrNull(3000L) {
    fetchDataFromNetwork()
}

if (result == null) {
    println("시간 초과 — 기본값 사용")
} else {
    println("결과: $result")
}
```

---

## 8. Android 실전 예제 ① — ViewModel에서 취소

```kotlin
@HiltViewModel
class SearchViewModel @Inject constructor(
    private val searchRepository: SearchRepository
) : ViewModel() {

    private val _state = MutableStateFlow(SearchState())
    val state: StateFlow<SearchState> = _state.asStateFlow()

    private var searchJob: Job? = null

    fun search(query: String) {
        // 이전 검색 취소 — 최신 검색만 실행
        searchJob?.cancel()

        if (query.isBlank()) {
            _state.update { it.copy(results = emptyList()) }
            return
        }

        searchJob = viewModelScope.launch {
            _state.update { it.copy(isLoading = true) }
            delay(300)   // 디바운스 — 300ms 내 새 검색이 오면 이 코루틴은 취소됨

            runCatching { searchRepository.search(query) }
                .onSuccess { results ->
                    _state.update { it.copy(isLoading = false, results = results) }
                }
                .onFailure { e ->
                    if (e is CancellationException) return@launch  // 취소는 무시
                    _state.update { it.copy(isLoading = false, error = e.message) }
                }
        }
    }

    // viewModelScope은 ViewModel이 cleared될 때 자동 취소
    // 별도 cancel() 불필요
}
```

---

## 9. Android 실전 예제 ② — lifecycleScope와 화면 전환

```kotlin
class ProductFragment : Fragment() {

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)

        // viewLifecycleOwner.lifecycleScope — Fragment View가 destroy되면 자동 취소
        viewLifecycleOwner.lifecycleScope.launch {
            viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
                // STARTED → STOPPED 될 때 자동 취소
                // STARTED 로 재진입 시 재실행
                viewModel.state.collect { state -> renderState(state) }
            }
        }
    }
}
```

---

## 10. 취소 관련 함수 한눈에 보기

| 함수 / 프로퍼티 | 역할 |
|----------------|------|
| `job.cancel()` | 코루틴에 취소 신호 전송 |
| `job.cancelAndJoin()` | 취소 후 완료까지 대기 |
| `isActive` | 현재 코루틴 활성 여부 확인 |
| `ensureActive()` | 취소 상태면 CancellationException throw |
| `yield()` | 취소 확인 + 스레드 양보 |
| `withTimeout()` | 시간 초과 시 취소 + 예외 |
| `withTimeoutOrNull()` | 시간 초과 시 null 반환 |
| `NonCancellable` | finally 내부에서 취소 무시 |

---

## 11. 정리

| 항목 | 내용 |
|------|------|
| 취소 방식 | 협력적 — 코루틴이 스스로 확인해야 멈춤 |
| 자동 취소 확인 | `delay()`, `withContext()` 등 suspend 함수 |
| 수동 취소 확인 | `isActive`, `ensureActive()` |
| 취소 예외 | `CancellationException` — 반드시 재전파 |
| 리소스 정리 | `finally` 블록, `use()` |
| 취소 불가 작업 | `withContext(NonCancellable)` — finally 내부 한정 |
| Android | viewModelScope, lifecycleScope 자동 취소 |

- 코루틴 취소는 **협력적** 입니다. `delay()`, `yield()` 같은 suspend 함수가 없는 루프라면 반드시 `isActive`로 직접 확인하세요.
- `CancellationException`을 `catch (e: Exception)`으로 삼키지 말고, 반드시 재전파하세요.

---

## 참고

- [Kotlin Coroutines 공식 문서 — 취소](https://kotlinlang.org/docs/cancellation-and-timeouts.html)
- [Job vs SupervisorJob 포스팅 보기](/posts/2026-05-13-ko-coroutines-job)
