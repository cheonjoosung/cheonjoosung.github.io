---
title: (Kotlin/코틀린) Flow retry & retryWhen 완전 정리 — 네트워크 재시도 패턴
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin Flow의 retry와 retryWhen 연산자로 네트워크 오류 재시도, 지수 백오프, 조건부 재시도 패턴을 구현하는 방법을 정리합니다.
---

---

## 1. 왜 재시도 로직이 필요한가?

네트워크 요청은 일시적인 오류가 빈번합니다.

```
- 와이파이 → LTE 전환 순간 연결 끊김
- 서버 일시적 과부하 (503 Service Unavailable)
- 타임아웃
- DNS 조회 실패
```

이런 **일시적 오류**는 재시도하면 성공할 가능성이 높습니다.  
매번 수동으로 재시도 로직을 작성하는 대신, Flow의 `retry`와 `retryWhen`으로 선언적으로 처리할 수 있습니다.

---

## 2. retry

### 기본 사용

```kotlin
flow {
    emit(api.fetchData())  // 실패 시 예외 발생
}
.retry(3)  // 최대 3번 재시도
.collect { data -> handleData(data) }
```

예외가 발생하면 Flow를 처음부터 재실행합니다. 최대 `retries`번까지 재시도합니다.

### 조건부 재시도

```kotlin
flow {
    emit(api.fetchData())
}
.retry(3) { cause ->
    // true 반환 → 재시도, false 반환 → 예외 전파
    cause is IOException  // IOException일 때만 재시도
}
.catch { e ->
    // 3번 재시도 후에도 실패하거나, 재시도 조건 false면 여기서 처리
    handleError(e)
}
.collect { handleData(it) }
```

### 재시도 간 딜레이 추가

```kotlin
flow {
    emit(api.fetchData())
}
.retry(3) { cause ->
    if (cause is IOException) {
        delay(1000)  // 1초 후 재시도
        true
    } else {
        false
    }
}
```

---

## 3. retryWhen

`retry`보다 세밀한 제어가 필요할 때 사용합니다. 현재 시도 횟수(`attempt`)를 파라미터로 받습니다.

```kotlin
flow {
    emit(api.fetchData())
}
.retryWhen { cause, attempt ->
    // cause: 발생한 예외
    // attempt: 0부터 시작하는 시도 횟수 (0 = 첫 번째 재시도)
    if (cause is IOException && attempt < 3) {
        delay(1000)
        true   // 재시도
    } else {
        false  // 포기
    }
}
```

---

## 4. 지수 백오프 (Exponential Backoff)

재시도 간격을 시도마다 지수적으로 늘립니다. 서버 과부하를 방지하는 표준 패턴입니다.

```
1번 재시도: 1초 후
2번 재시도: 2초 후
3번 재시도: 4초 후
4번 재시도: 8초 후
```

```kotlin
fun <T> Flow<T>.retryWithExponentialBackoff(
    maxRetries: Int = 3,
    initialDelay: Long = 1000L,
    maxDelay: Long = 16000L,
    factor: Double = 2.0,
    retryOn: (Throwable) -> Boolean = { it is IOException }
): Flow<T> = retryWhen { cause, attempt ->
    if (retryOn(cause) && attempt < maxRetries) {
        val delayMs = (initialDelay * factor.pow(attempt.toDouble()))
            .toLong()
            .coerceAtMost(maxDelay)
        delay(delayMs)
        true
    } else {
        false
    }
}
```

```kotlin
// 사용
repository.fetchUserFlow()
    .retryWithExponentialBackoff(
        maxRetries = 4,
        initialDelay = 500L,
        retryOn = { it is IOException || it is HttpException }
    )
    .catch { e -> emit(Result.failure(e)) }
    .collect { handleResult(it) }
```

---

## 5. HTTP 상태 코드별 조건부 재시도

```kotlin
sealed class ApiException(message: String) : Exception(message) {
    class NetworkError(message: String) : ApiException(message)
    class ServerError(val code: Int, message: String) : ApiException(message)
    class ClientError(val code: Int, message: String) : ApiException(message)
}

fun <T> Flow<T>.retryOnServerError(maxRetries: Int = 3): Flow<T> =
    retryWhen { cause, attempt ->
        when {
            cause is ApiException.NetworkError && attempt < maxRetries -> {
                delay(1000L * (attempt + 1))
                true
            }
            cause is ApiException.ServerError && cause.code == 503 && attempt < maxRetries -> {
                // 503 Service Unavailable: 재시도 유효
                delay(2000L * (attempt + 1))
                true
            }
            cause is ApiException.ClientError -> {
                // 4xx 클라이언트 오류: 재시도해도 의미 없음
                false
            }
            else -> false
        }
    }
```

---

## 6. ViewModel 실전 패턴

```kotlin
class UserViewModel(
    private val repository: UserRepository
) : ViewModel() {

    private val _uiState = MutableStateFlow<UiState>(UiState.Loading)
    val uiState: StateFlow<UiState> = _uiState.asStateFlow()

    fun loadUser(userId: String) {
        viewModelScope.launch {
            repository.getUserFlow(userId)
                .retryWhen { cause, attempt ->
                    if (cause is IOException && attempt < 3) {
                        _uiState.value = UiState.Retrying(attempt.toInt() + 1)
                        delay(1000L * (attempt + 1))
                        true
                    } else {
                        false
                    }
                }
                .onStart { _uiState.value = UiState.Loading }
                .catch { e -> _uiState.value = UiState.Error(e.message ?: "Unknown error") }
                .collect { user -> _uiState.value = UiState.Success(user) }
        }
    }
}

sealed class UiState {
    object Loading : UiState()
    data class Retrying(val attempt: Int) : UiState()
    data class Success(val user: User) : UiState()
    data class Error(val message: String) : UiState()
}
```

---

## 7. retry vs retryWhen 비교

| 구분 | retry | retryWhen |
|------|-------|-----------|
| 시도 횟수 접근 | 불가 | `attempt` 파라미터로 접근 |
| 조건 지정 | 람다로 가능 | 람다로 가능 |
| 딜레이 | 람다 내 delay() | 람다 내 delay() |
| 지수 백오프 | 복잡함 | 자연스러운 구현 |
| 주요 용도 | 단순 재시도 | 세밀한 재시도 제어 |

---

## 8. 주의 사항

```kotlin
// retry는 Flow를 처음부터 재실행
// → emit 전 초기화 코드도 다시 실행됨
flow {
    val token = authManager.getToken()  // 재시도 시 매번 새 토큰 획득
    emit(api.fetchData(token))
}
.retry(3)

// 재시도가 불필요한 예외는 즉시 전파
flow { emit(api.fetchData()) }
    .retry(3) { cause ->
        cause !is CancellationException  // 코루틴 취소는 재시도 안 함
        && cause is IOException
    }
```

---

## 9. 정리

- **`retry(n)`**: 최대 n번 재시도, 간단한 재시도에 적합
- **`retryWhen { cause, attempt }`**: 시도 횟수 기반 세밀한 제어, 지수 백오프에 적합
- 네트워크 오류(`IOException`)는 재시도, 클라이언트 오류(4xx)는 즉시 실패 처리
- 지수 백오프 패턴으로 서버 과부하 방지
