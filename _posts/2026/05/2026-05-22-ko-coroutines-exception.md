---
title: (Kotlin/코틀린) CoroutineExceptionHandler와 supervisorScope
tags: [ Kotlin ]
style: fill
color: dark
description: CoroutineExceptionHandler의 동작 원리, 적용 위치, supervisorScope와의 조합, runCatching 패턴, Android ViewModel 실전 예외 처리까지 깊이 있게 정리합니다.
---

## 개요

- Kotlin Coroutines의 **예외 처리** 를 다룹니다.
- `CoroutineExceptionHandler`, `supervisorScope`, `runCatching`을 언제 어떻게 사용하는지 정리합니다.
- 이 글에서는 다음을 설명합니다.
  - 코루틴 예외 전파 방식
  - `CoroutineExceptionHandler` — 전역 예외 처리기
  - `supervisorScope` — 예외 격리
  - `launch` vs `async`의 예외 처리 차이
  - `runCatching` — Kotlin 스타일 예외 처리
  - Android 실전 패턴

---

## 1. 코루틴 예외 전파 방식

코루틴의 예외 전파는 **launch**와 **async**가 다르게 동작합니다.

```kotlin
// launch — 예외가 즉시 전파됨
val job = launch {
    throw RuntimeException("launch 예외")
    // 즉시 부모로 전파 → 형제 코루틴도 취소
}

// async — 예외가 await() 호출 시점에 전파됨
val deferred = async {
    throw RuntimeException("async 예외")
    "결과"
}
// 이 시점엔 예외 없음

try {
    deferred.await()  // ← 여기서 예외 발생
} catch (e: RuntimeException) {
    println("async 예외 잡힘: $e")
}
```

---

## 2. CoroutineExceptionHandler

`launch`로 시작된 코루틴에서 처리되지 않은 예외를 잡는 **전역 핸들러** 입니다.

```kotlin
val handler = CoroutineExceptionHandler { context, exception ->
    println("처리되지 않은 예외: $exception")
    println("코루틴: ${context[CoroutineName]}")
    // 로깅, 에러 리포팅 등
}

val scope = CoroutineScope(SupervisorJob() + handler)

scope.launch(CoroutineName("작업1")) {
    throw RuntimeException("오류 발생!")
    // handler가 받아서 처리 ✅
}

scope.launch(CoroutineName("작업2")) {
    delay(1000)
    println("작업2 완료")  // 정상 실행 ✅ (SupervisorJob이므로)
}
```

---

## 3. CoroutineExceptionHandler 적용 위치

### ❌ 자식 코루틴에만 설정 — 작동 안 함

```kotlin
val scope = CoroutineScope(Job())

scope.launch {
    val handler = CoroutineExceptionHandler { _, e -> println("처리: $e") }

    launch(handler) {   // ❌ 자식에 설정 — 부모로 전파되어 handler 무시됨
        throw RuntimeException("오류")
    }
}
```

### ✅ 루트 코루틴 또는 스코프에 설정

```kotlin
val handler = CoroutineExceptionHandler { _, e ->
    println("전역 처리: $e")
}

// 방법 1 — CoroutineScope에 설정
val scope = CoroutineScope(SupervisorJob() + handler)
scope.launch {
    throw RuntimeException("오류")  // handler가 받음 ✅
}

// 방법 2 — 루트 launch에 설정
val rootScope = CoroutineScope(SupervisorJob())
rootScope.launch(handler) {   // 루트 코루틴에 직접 설정 ✅
    throw RuntimeException("오류")
}
```

### CoroutineExceptionHandler가 동작하는 조건

```
✔ SupervisorJob 또는 supervisorScope 환경의 자식 코루틴
✔ 루트 코루틴 (부모가 없는 코루틴)
✘ async + await() — await()에서 예외를 받아 try-catch로 처리
✘ coroutineScope 내부 — 예외가 coroutineScope 밖으로 전파됨
```

---

## 4. launch vs async 예외 처리 패턴

### launch — try-catch로 내부 처리

```kotlin
scope.launch {
    try {
        val result = fetchData()
        processResult(result)
    } catch (e: IOException) {
        println("네트워크 오류: $e")
    } catch (e: CancellationException) {
        throw e  // 취소는 재전파
    }
}
```

### async — await() 호출 시 예외 처리

```kotlin
val deferred = scope.async {
    fetchData()  // 예외 발생 시 deferred에 저장
}

// await() 시점에 예외 처리
try {
    val result = deferred.await()
    processResult(result)
} catch (e: IOException) {
    println("데이터 로딩 실패: $e")
}
```

### async + supervisorScope — 독립 병렬 처리

```kotlin
suspend fun loadAll(): AllData = supervisorScope {
    val userDeferred    = async { userRepo.getUser() }
    val productDeferred = async { productRepo.getProducts() }
    val noticeDeferred  = async { noticeRepo.getNotices() }

    // 각각 독립적으로 예외 처리
    val user = try {
        userDeferred.await()
    } catch (e: Exception) {
        null
    }

    val products = try {
        productDeferred.await()
    } catch (e: Exception) {
        emptyList()
    }

    val notices = try {
        noticeDeferred.await()
    } catch (e: Exception) {
        emptyList()
    }

    AllData(user, products, notices)
}
```

---

## 5. runCatching — Kotlin 스타일 예외 처리

`runCatching`은 `Result<T>`를 반환하는 Kotlin 표준 함수로, 코루틴 예외 처리를 간결하게 만듭니다.

```kotlin
suspend fun fetchUser(id: Long): Result<User> = runCatching {
    userApi.getUser(id)  // 예외 발생 시 Result.failure()로 래핑
}

// 호출부
viewModelScope.launch {
    fetchUser(userId)
        .onSuccess { user ->
            _state.update { it.copy(user = user) }
        }
        .onFailure { e ->
            if (e is CancellationException) throw e  // 취소 재전파
            _state.update { it.copy(error = e.message) }
        }
}
```

### runCatching 주의사항 — CancellationException

```kotlin
// ❌ CancellationException도 catch됨 — 취소가 무시됨
suspend fun riskyOperation() {
    runCatching {
        delay(1000)
    }.onFailure { e ->
        println("실패: $e")
        // CancellationException도 여기로 옴 → 재전파 안 하면 취소 무시
    }
}

// ✅ CancellationException 재전파
suspend fun safeOperation() {
    runCatching {
        delay(1000)
    }.onFailure { e ->
        if (e is CancellationException) throw e   // 취소 재전파 필수
        handleError(e)
    }
}

// ✅ 확장 함수로 패턴화
inline fun <T> Result<T>.onFailureExceptCancellation(
    crossinline action: (Throwable) -> Unit
): Result<T> = onFailure { e ->
    if (e is CancellationException) throw e
    action(e)
}

// 사용
runCatching { fetchData() }
    .onSuccess { processData(it) }
    .onFailureExceptCancellation { e -> showError(e.message) }
```

---

## 6. supervisorScope 예외 처리 패턴

```kotlin
// supervisorScope — 자식 예외가 형제에게 전파되지 않음
// 단, await()는 예외를 발생시킴

suspend fun loadHomeData(): HomeData = supervisorScope {
    // 병렬로 시작
    val bannerDeferred  = async { bannerRepo.getBanners() }
    val productDeferred = async { productRepo.getFeatured() }
    val eventDeferred   = async { eventRepo.getActiveEvents() }

    // 실패해도 나머지 계속 로딩
    val banners = runCatching { bannerDeferred.await() }
        .getOrDefault(emptyList())

    val products = runCatching { productDeferred.await() }
        .getOrDefault(emptyList())

    val events = runCatching { eventDeferred.await() }
        .getOrElse { e ->
            if (e is CancellationException) throw e
            emptyList()
        }

    HomeData(banners, products, events)
}
```

---

## 7. Android 실전 예제 ① — ViewModel 예외 처리

```kotlin
@HiltViewModel
class ProductViewModel @Inject constructor(
    private val productRepo: ProductRepository
) : ViewModel() {

    private val _state = MutableStateFlow(ProductState())
    val state: StateFlow<ProductState> = _state.asStateFlow()

    // 방법 1 — try-catch
    fun loadProducts() {
        viewModelScope.launch {
            _state.update { it.copy(isLoading = true) }
            try {
                val products = productRepo.getProducts()
                _state.update { it.copy(isLoading = false, products = products) }
            } catch (e: CancellationException) {
                throw e  // 취소 재전파
            } catch (e: IOException) {
                _state.update { it.copy(isLoading = false, error = "네트워크 오류") }
            } catch (e: Exception) {
                _state.update { it.copy(isLoading = false, error = e.message) }
            }
        }
    }

    // 방법 2 — runCatching (간결)
    fun loadProductsV2() {
        viewModelScope.launch {
            _state.update { it.copy(isLoading = true) }
            runCatching { productRepo.getProducts() }
                .onSuccess { products ->
                    _state.update { it.copy(isLoading = false, products = products) }
                }
                .onFailure { e ->
                    if (e is CancellationException) throw e
                    _state.update { it.copy(isLoading = false, error = e.message) }
                }
        }
    }

    // 방법 3 — 병렬 + 독립 처리
    fun loadDashboard() {
        viewModelScope.launch {
            supervisorScope {
                launch {
                    runCatching { productRepo.getFeatured() }
                        .onSuccess { _state.update { it.copy(featured = it.featured) } }
                        .onFailureExceptCancellation { /* 조용히 실패 */ }
                }
                launch {
                    runCatching { productRepo.getRecommended() }
                        .onSuccess { recommended -> _state.update { it.copy(recommended = recommended) } }
                        .onFailureExceptCancellation { /* 조용히 실패 */ }
                }
            }
        }
    }
}
```

---

## 8. Android 실전 예제 ② — 전역 에러 처리

```kotlin
// Application 레벨 CoroutineExceptionHandler
class App : Application() {

    val appCoroutineScope = CoroutineScope(
        SupervisorJob() +
        Dispatchers.Main +
        CoroutineExceptionHandler { _, throwable ->
            // Firebase Crashlytics 등에 리포팅
            FirebaseCrashlytics.getInstance().recordException(throwable)
            println("처리되지 않은 코루틴 예외: $throwable")
        }
    )

    override fun onTerminate() {
        super.onTerminate()
        appCoroutineScope.cancel()
    }
}

// 전역 스코프를 필요한 곳에서 사용
class SomeManager(private val app: App) {
    fun doBackgroundWork() {
        app.appCoroutineScope.launch {
            // 예외 발생 시 전역 handler가 처리
            riskyWork()
        }
    }
}
```

---

## 9. 예외 처리 패턴 선택 기준

```
상황 1 — 단일 작업, 실패 시 UI에 에러 표시
→ viewModelScope.launch + runCatching

상황 2 — 여러 작업이 모두 성공해야 의미 있을 때
→ coroutineScope + async + try-catch (await 호출 시)

상황 3 — 여러 작업이 독립적, 각자 실패 처리
→ supervisorScope + launch or async + runCatching

상황 4 — 처리되지 않은 예외 글로벌 처리 (로깅, 크래시 리포팅)
→ CoroutineExceptionHandler + SupervisorJob (루트 스코프에)

상황 5 — async 결과를 나중에 사용할 때
→ async + try-catch on await()
```

---

## 10. 정리

| 항목 | 내용 |
|------|------|
| `launch` 예외 | 즉시 부모로 전파 → try-catch 내부에서 처리 |
| `async` 예외 | `await()` 호출 시 발생 → try-catch on await() |
| `CoroutineExceptionHandler` | 루트/스코프에 설정, launch 처리되지 않은 예외 |
| `supervisorScope` | 자식 예외 격리 — 형제에게 전파 안 함 |
| `runCatching` | 간결한 Result 기반 처리, CancellationException 재전파 주의 |
| Android | viewModelScope (SupervisorJob 기반) + runCatching |

- `CancellationException`은 절대 삼키지 말고 반드시 재전파하세요.
- `viewModelScope`는 이미 `SupervisorJob` 기반이므로 `launch` 간 예외는 독립적으로 처리됩니다.
- 전역 로깅이 필요하면 `CoroutineExceptionHandler`를 루트 스코프에 적용하세요.

---

## 참고

- [Kotlin Coroutines 공식 문서 — 예외 처리](https://kotlinlang.org/docs/exception-handling.html)
- [Coroutines 취소 포스팅 보기](/posts/2026-05-12-ko-coroutines-cancel)
- [Job vs SupervisorJob 포스팅 보기](/posts/2026-05-13-ko-coroutines-job)
