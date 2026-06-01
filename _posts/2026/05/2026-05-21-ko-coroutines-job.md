---
title: (Kotlin/코틀린) Job vs SupervisorJob — 자식 코루틴 실패 전파
tags: [ Kotlin ]
style: fill
color: dark
description: Job과 SupervisorJob의 차이, 자식 코루틴 실패 전파 방식, supervisorScope, Android ViewModel 실전 패턴까지 깊이 있게 정리합니다.
---

## 개요

- Kotlin Coroutines의 **Job과 SupervisorJob** 을 다룹니다.
- 자식 코루틴이 실패했을 때 실패가 어떻게 전파되는지 이해하는 것이 핵심입니다.
- 이 글에서는 다음을 설명합니다.
  - Job의 부모-자식 관계와 실패 전파
  - SupervisorJob — 자식 실패가 형제에게 전파되지 않음
  - `supervisorScope` vs `coroutineScope`
  - Android 실전 패턴 (viewModelScope, 독립 작업)

---

## 1. Job의 부모-자식 관계

코루틴을 `launch`하면 새 Job이 생성되고, 부모 Job의 자식이 됩니다.

```kotlin
val parentJob = Job()
val scope = CoroutineScope(parentJob)

val child1 = scope.launch { delay(1000); println("child1 완료") }
val child2 = scope.launch { delay(2000); println("child2 완료") }

println(child1.parent === parentJob)  // true
println(child2.parent === parentJob)  // true
```

**부모-자식 Job의 규칙:**

```
1. 부모가 취소되면 → 모든 자식이 취소됨
2. 자식이 모두 완료될 때까지 → 부모가 완료되지 않음
3. 자식이 실패하면 → 부모와 다른 모든 자식이 취소됨  ← Job의 핵심 동작
```

---

## 2. Job — 자식 실패가 전체에 전파

```kotlin
val scope = CoroutineScope(Job())

val child1 = scope.launch {
    delay(1000)
    println("child1: 정상 완료")
}

val child2 = scope.launch {
    delay(500)
    throw RuntimeException("child2 실패!")  // ← 예외 발생
}

val child3 = scope.launch {
    delay(2000)
    println("child3: 정상 완료")  // ← 실행 안 됨!
}

delay(3000)
println("child1 상태: ${child1.isCancelled}")  // true ← child2 실패로 취소됨
println("child3 상태: ${child3.isCancelled}")  // true ← child2 실패로 취소됨
```

```
Job 실패 전파:
child2 실패 → 부모 Job 실패 → child1, child3 취소
```

---

## 3. SupervisorJob — 자식 실패가 형제에게 전파되지 않음

```kotlin
val scope = CoroutineScope(SupervisorJob())  // SupervisorJob 사용

val child1 = scope.launch {
    delay(1000)
    println("child1: 정상 완료")  // ← 실행됨 ✅
}

val child2 = scope.launch {
    delay(500)
    throw RuntimeException("child2 실패!")
}

val child3 = scope.launch {
    delay(2000)
    println("child3: 정상 완료")  // ← 실행됨 ✅
}

delay(3000)
println("child1 상태: ${child1.isCompleted}")  // true — 정상 완료
println("child3 상태: ${child3.isCompleted}")  // true — 정상 완료
```

```
SupervisorJob 실패 전파:
child2 실패 → child2만 실패, child1·child3 영향 없음
```

---

## 4. Job vs SupervisorJob 비교

```kotlin
// Job — 하나가 실패하면 전체 취소
// 사용 시점: 작업들이 서로 의존적일 때 (하나 실패 시 나머지 의미 없음)
val downloadScope = CoroutineScope(Job())
downloadScope.launch { downloadPart1() }
downloadScope.launch { downloadPart2() }  // 실패 시 part1도 취소
downloadScope.launch { downloadPart3() }

// SupervisorJob — 각 자식이 독립적
// 사용 시점: 작업들이 독립적일 때 (하나 실패해도 나머지는 계속)
val dashboardScope = CoroutineScope(SupervisorJob())
dashboardScope.launch { loadNews() }       // 실패해도
dashboardScope.launch { loadWeather() }    // 실패해도
dashboardScope.launch { loadStocks() }     // 각자 독립적으로 실행
```

| 항목 | Job | SupervisorJob |
|------|-----|---------------|
| 자식 실패 시 | 부모 + 모든 형제 취소 | 해당 자식만 실패 |
| 부모 취소 시 | 모든 자식 취소 | 모든 자식 취소 (동일) |
| 사용 시점 | 상호 의존적 작업 | 독립적 작업 |
| Android 예 | 분할 다운로드 | 독립 API 병렬 호출 |

---

## 5. coroutineScope vs supervisorScope

함수 내부에서 일시적으로 스코프를 만들 때 사용합니다.

### coroutineScope — 자식 실패 시 전체 취소

```kotlin
suspend fun loadDashboard() = coroutineScope {
    val news    = async { loadNews() }
    val weather = async { loadWeather() }

    // news 또는 weather 중 하나가 실패하면
    // 다른 하나도 취소되고 coroutineScope가 예외를 던짐
    DashboardData(news.await(), weather.await())
}

// 호출부
try {
    val data = loadDashboard()
} catch (e: Exception) {
    println("대시보드 로딩 실패: $e")
}
```

---

### supervisorScope — 자식 실패가 형제에게 전파되지 않음

```kotlin
suspend fun loadDashboard() = supervisorScope {
    val newsDeferred    = async { loadNews() }
    val weatherDeferred = async { loadWeather() }
    val stocksDeferred  = async { loadStocks() }

    val news = try {
        newsDeferred.await()
    } catch (e: Exception) {
        println("뉴스 로딩 실패, 기본값 사용")
        emptyList()   // 실패 시 기본값 — 다른 작업 영향 없음
    }

    val weather = try {
        weatherDeferred.await()
    } catch (e: Exception) {
        println("날씨 로딩 실패")
        null
    }

    val stocks = try {
        stocksDeferred.await()
    } catch (e: Exception) {
        println("주식 로딩 실패")
        emptyList()
    }

    DashboardData(news, weather, stocks)  // 성공한 것만 사용
}
```

---

## 6. 실패 전파 흐름 정리

```
coroutineScope {
    async { 실패 } → 부모(coroutineScope) 취소 → 다른 자식 취소 → 예외 전파
}

supervisorScope {
    async { 실패 } → 해당 async만 실패 → 다른 자식 영향 없음
    ※ await()를 직접 호출하면 예외가 await() 호출 지점에서 발생
}
```

---

## 7. Android 실전 예제 ① — viewModelScope

`viewModelScope`는 내부적으로 `SupervisorJob + Dispatchers.Main`으로 구성됩니다.

```kotlin
// viewModelScope 내부 구성 (Android 소스 코드 참고)
// val viewModelScope = CoroutineScope(SupervisorJob() + Dispatchers.Main.immediate)

@HiltViewModel
class HomeViewModel @Inject constructor(
    private val newsRepo: NewsRepository,
    private val weatherRepo: WeatherRepository
) : ViewModel() {

    private val _state = MutableStateFlow(HomeState())
    val state: StateFlow<HomeState> = _state.asStateFlow()

    fun loadHome() {
        // SupervisorJob이므로 각 launch가 독립적
        viewModelScope.launch {
            runCatching { newsRepo.getLatestNews() }
                .onSuccess { news -> _state.update { it.copy(news = news) } }
                .onFailure { _state.update { it.copy(newsError = true) } }
        }

        viewModelScope.launch {
            runCatching { weatherRepo.getCurrentWeather() }
                .onSuccess { weather -> _state.update { it.copy(weather = weather) } }
                .onFailure { _state.update { it.copy(weatherError = true) } }
        }
        // 뉴스 실패해도 날씨는 정상 표시 ✅
    }
}
```

---

## 8. Android 실전 예제 ② — 독립적 병렬 작업

```kotlin
@HiltViewModel
class OrderViewModel @Inject constructor(
    private val orderRepo: OrderRepository,
    private val inventoryRepo: InventoryRepository,
    private val couponRepo: CouponRepository
) : ViewModel() {

    // 상호 의존적 — 하나 실패 시 전체 의미 없음 → coroutineScope
    fun placeOrder(orderId: String) {
        viewModelScope.launch {
            _state.update { it.copy(isLoading = true) }
            runCatching {
                coroutineScope {
                    val order     = async { orderRepo.getOrder(orderId) }
                    val inventory = async { inventoryRepo.checkStock(orderId) }
                    // 둘 다 성공해야 의미 있음 — 하나 실패 시 모두 취소
                    Pair(order.await(), inventory.await())
                }
            }.onSuccess { (order, inventory) ->
                processOrder(order, inventory)
            }.onFailure { e ->
                _state.update { it.copy(isLoading = false, error = e.message) }
            }
        }
    }

    // 독립적 — 각자 로딩 가능 → supervisorScope
    fun loadOrderDetails(orderId: String) {
        viewModelScope.launch {
            supervisorScope {
                launch {
                    runCatching { orderRepo.getOrder(orderId) }
                        .onSuccess { _state.update { it.copy(order = it.order) } }
                }
                launch {
                    runCatching { couponRepo.getAppliedCoupons(orderId) }
                        .onSuccess { coupons -> _state.update { it.copy(coupons = coupons) } }
                }
            }
        }
    }
}
```

---

## 9. Job 상태 흐름

```
생성(New)
    ↓ start() 또는 첫 실행
활성(Active)
    ↓ 완료 또는 cancel()
완료 중(Completing / Cancelling)
    ↓ 자식 완료 대기
완료(Completed) 또는 취소됨(Cancelled)
```

```kotlin
val job = launch { delay(1000) }

println(job.isActive)     // true
println(job.isCompleted)  // false
println(job.isCancelled)  // false

delay(1500)

println(job.isActive)     // false
println(job.isCompleted)  // true
println(job.isCancelled)  // false
```

---

## 10. 정리

| 항목 | Job | SupervisorJob |
|------|-----|---------------|
| 자식 실패 전파 | 부모 + 모든 형제 취소 | 해당 자식만 실패 |
| `coroutineScope` | 동일 동작 | — |
| `supervisorScope` | — | 동일 동작 |
| viewModelScope | SupervisorJob 기반 ✅ | — |
| 사용 시점 | 작업이 상호 의존적 | 작업이 독립적 |

- 자식 작업이 **서로 의존적** 이라면 `Job` / `coroutineScope` — 하나 실패 시 전체 취소가 맞습니다.
- 자식 작업이 **독립적** 이라면 `SupervisorJob` / `supervisorScope` — 각자 실패를 처리합니다.
- `viewModelScope`는 이미 `SupervisorJob` 기반이므로, 별도 처리 없이 독립적 launch가 가능합니다.

---

## 참고

- [Kotlin Coroutines 공식 문서 — 예외 처리](https://kotlinlang.org/docs/exception-handling.html)
- [Coroutines 취소 포스팅 보기](/posts/2026-05-12-ko-coroutines-cancel)
- [CoroutineExceptionHandler 포스팅 보기](/posts/2026-05-14-ko-coroutines-exception)
