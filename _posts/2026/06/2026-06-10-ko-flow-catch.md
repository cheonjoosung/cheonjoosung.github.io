---
title: (Kotlin/코틀린) Flow catch & onCompletion 완전 정리 — 에러/완료 처리
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin Flow의 catch와 onCompletion 연산자로 에러 처리, 완료 감지, 리소스 정리를 선언적으로 구현하는 방법을 정리합니다.
---

---

## 1. Flow의 에러 처리 방식

일반 코루틴에서는 `try-catch`로 에러를 처리합니다.  
Flow에서는 연산자 체인에 `catch`를 추가하는 선언적 방식을 사용합니다.

```kotlin
// try-catch 방식 (작동하지만 Flow스럽지 않음)
try {
    dataFlow.collect { data -> process(data) }
} catch (e: Exception) {
    handleError(e)
}

// Flow 방식 (권장)
dataFlow
    .catch { e -> handleError(e) }
    .collect { data -> process(data) }
```

---

## 2. catch

### 기본 동작

`catch`는 **업스트림에서 발생한 예외**를 잡아 처리합니다.  
`catch` 이후(다운스트림)에서 발생한 예외는 잡지 않습니다.

```kotlin
flow {
    emit(1)
    throw IOException("네트워크 오류")
    emit(2)  // 실행되지 않음
}
.catch { e ->
    println("에러 처리: ${e.message}")
    emit(-1)  // 에러 발생 시 대체 값 방출 가능
}
.collect { value ->
    println("수집: $value")
}

// 출력:
// 수집: 1
// 에러 처리: 네트워크 오류
// 수집: -1
```

### catch 위치 — 업스트림에만 영향

```kotlin
flow { emit(loadData()) }           // 에러 발생 가능
    .map { transform(it) }          // 에러 발생 가능
    .catch { e ->                   // 위의 모든 업스트림 에러를 처리
        emit(defaultValue)
    }
    .collect { value ->
        riskyOperation(value)       // 여기서 발생하는 에러는 catch가 잡지 않음!
    }
```

```kotlin
// collect 내 에러를 처리하려면 try-catch 사용
.collect { value ->
    try {
        riskyOperation(value)
    } catch (e: Exception) {
        handleError(e)
    }
}
```

### 에러 타입별 처리

```kotlin
apiFlow
    .catch { e ->
        when (e) {
            is IOException -> emit(Result.failure(NetworkException("네트워크 오류")))
            is HttpException -> when (e.code()) {
                401 -> emit(Result.failure(AuthException("인증 실패")))
                404 -> emit(Result.failure(NotFoundException("리소스 없음")))
                else -> emit(Result.failure(e))
            }
            is CancellationException -> throw e  // 취소는 반드시 재전파
            else -> emit(Result.failure(e))
        }
    }
```

### catch에서 새 값 방출

```kotlin
// 에러 발생 시 캐시 데이터로 대체
remoteDataFlow
    .catch { e ->
        if (e is IOException) {
            emitAll(localCacheFlow)  // 캐시 Flow 전체를 방출
        } else {
            throw e  // 다른 에러는 재전파
        }
    }
    .collect { data -> updateUI(data) }
```

---

## 3. onCompletion

Flow가 **완료, 취소, 에러** 중 어떤 이유로든 종료될 때 호출됩니다.  
`cause` 파라미터로 종료 이유를 알 수 있습니다.

```kotlin
flow {
    emit(1)
    emit(2)
}
.onCompletion { cause ->
    if (cause != null) {
        println("에러로 종료: ${cause.message}")
    } else {
        println("정상 완료")
    }
}
.collect { println("수집: $it") }

// 출력:
// 수집: 1
// 수집: 2
// 정상 완료
```

### 에러 발생 시

```kotlin
flow {
    emit(1)
    throw RuntimeException("오류 발생")
}
.onCompletion { cause ->
    println("종료 이유: $cause")  // cause = RuntimeException("오류 발생")
}
.catch { e ->
    println("catch 처리: ${e.message}")
}
.collect { println("수집: $it") }

// 출력:
// 수집: 1
// 종료 이유: java.lang.RuntimeException: 오류 발생
// catch 처리: 오류 발생
```

### catch vs onCompletion 실행 순서

`onCompletion`은 `catch`보다 먼저 실행됩니다.

```kotlin
flow { throw IOException("에러") }
    .onCompletion { cause -> println("onCompletion: $cause") }  // 1번째
    .catch { e -> println("catch: ${e.message}") }              // 2번째
    .collect { }

// 출력:
// onCompletion: java.io.IOException: 에러
// catch: 에러
```

---

## 4. onCompletion에서 값 방출

`onCompletion`은 `catch`와 달리 **에러 여부와 관계없이 항상 실행**되며, 추가 값을 방출할 수 있습니다.

```kotlin
flow {
    emit("데이터1")
    emit("데이터2")
}
.onCompletion { cause ->
    if (cause == null) {
        emit("모든 데이터 로드 완료")  // 정상 완료 시에만 추가 방출
    }
}
.collect { println(it) }

// 출력:
// 데이터1
// 데이터2
// 모든 데이터 로드 완료
```

---

## 5. 실전 패턴

### 로딩 상태 처리

```kotlin
class ProductViewModel(private val repo: ProductRepository) : ViewModel() {

    private val _isLoading = MutableStateFlow(false)
    val isLoading: StateFlow<Boolean> = _isLoading.asStateFlow()

    private val _products = MutableStateFlow<List<Product>>(emptyList())
    val products: StateFlow<List<Product>> = _products.asStateFlow()

    private val _error = MutableStateFlow<String?>(null)
    val error: StateFlow<String?> = _error.asStateFlow()

    fun loadProducts() {
        viewModelScope.launch {
            repo.getProductsFlow()
                .onStart { _isLoading.value = true }
                .onCompletion { _isLoading.value = false }  // 항상 로딩 해제
                .catch { e -> _error.value = e.message }
                .collect { products -> _products.value = products }
        }
    }
}
```

### 리소스 정리

```kotlin
fun readFileFlow(path: String): Flow<String> = flow {
    val reader = BufferedReader(FileReader(path))
    reader.forEachLine { emit(it) }
    reader.close()  // 정상 완료 시 닫힘
}
.onCompletion { cause ->
    // 에러/취소 포함 어떤 경우에도 리소스 정리 보장
    println("파일 읽기 종료, 에러: $cause")
}
```

### UI 상태와 연동

```kotlin
// 로딩 인디케이터 + 에러 스낵바 조합
dataFlow
    .onStart { showLoading() }
    .onCompletion { hideLoading() }
    .catch { e ->
        showErrorSnackbar(e.message)
        emit(emptyList())
    }
    .collect { data -> renderData(data) }
```

---

## 6. catch, onCompletion, onStart 완전 정리

```kotlin
flow {
    emit(loadData())
}
.onStart {
    // Flow 수집 시작 직전, 값 방출 가능
    emit(CachedData)
    _isLoading.value = true
}
.map { transform(it) }
.catch { e ->
    // 업스트림 에러 처리, emit 가능
    emit(defaultValue)
}
.onCompletion { cause ->
    // 종료 시 항상 실행 (정상/에러/취소 무관), emit 가능
    _isLoading.value = false
}
.collect { data ->
    // 최종 소비, 여기서의 에러는 catch가 못 잡음
    updateUI(data)
}
```

| 연산자 | 실행 시점 | emit 가능 | 에러 처리 |
|--------|----------|----------|----------|
| `onStart` | collect 직전 | O | X |
| `catch` | 업스트림 에러 발생 시 | O | O |
| `onCompletion` | 종료 시 항상 | O | X (cause로 확인) |

---

## 7. CancellationException 주의

```kotlin
flow { emit(loadData()) }
    .catch { e ->
        if (e is CancellationException) throw e  // 취소는 반드시 재전파!
        handleError(e)
    }
```

`CancellationException`을 삼키면 코루틴 취소가 제대로 동작하지 않습니다.

---

## 8. 정리

- **`catch`**: 업스트림 에러를 처리, `emit`으로 대체 값 제공 가능, 다운스트림 에러는 못 잡음
- **`onCompletion`**: 정상/에러/취소 모든 종료 시 실행, 리소스 정리와 로딩 상태 해제에 활용
- **`onStart`**: 수집 시작 전 초기화, 로딩 상태 활성화에 활용
- `CancellationException`은 `catch`에서 반드시 재전파
- `onCompletion`이 `catch`보다 먼저 실행됨
