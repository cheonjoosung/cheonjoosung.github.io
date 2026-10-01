---
title: (Kotlin/코틀린) Flow 테스트 — Turbine 라이브러리
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin Flow를 테스트하는 방법과 Turbine 라이브러리의 test, awaitItem, awaitError, expectNoEvents 사용법을 정리합니다.
---

---

## 1. Flow 테스트가 어려운 이유

Flow는 비동기 스트림이라 "언제 값이 나오는지", "몇 개가 나오는지", "에러가 나는지"를 검증하기가 번거롭습니다.

```kotlin
// 문제 상황: toList()로 모으면 무한 Flow나 에러 발생 시나리오 테스트가 어려움
@Test
fun badTest() = runTest {
    val results = someFlow().toList()  // Flow가 완료되지 않으면 영원히 대기
    assertEquals(listOf(1, 2, 3), results)
}
```

**Turbine**은 Flow를 하나씩 순서대로 검증할 수 있는 테스트 전용 라이브러리입니다.

**의존성 추가**:
```kotlin
testImplementation("app.cash.turbine:turbine:1.2.0")
```

---

## 2. test { } — 기본 사용법

```kotlin
import app.cash.turbine.test
import kotlinx.coroutines.flow.flowOf
import kotlinx.coroutines.test.runTest
import org.junit.Test

class FlowTest {

    @Test
    fun `flow가 순서대로 값을 방출한다`() = runTest {
        val flow = flowOf(1, 2, 3)

        // test { }: Flow를 구독하고 방출되는 이벤트를 하나씩 검증할 수 있는 블록 시작
        flow.test {
            // awaitItem(): 다음 방출값을 기다렸다가 반환
            assertEquals(1, awaitItem())
            assertEquals(2, awaitItem())
            assertEquals(3, awaitItem())

            // awaitComplete(): Flow가 정상 종료되었는지 확인
            awaitComplete()
        }
    }
}
```

> **test 블록의 원리**: `test { }`는 내부적으로 Flow를 구독하는 코루틴을 실행하고, `awaitItem()` 호출마다 다음 이벤트가 도착할 때까지 (가상 시간으로) 대기합니다. 블록이 끝나면 자동으로 구독을 취소합니다.

---

## 3. awaitError — 에러 방출 검증

```kotlin
import kotlinx.coroutines.flow.flow

@Test
fun `데이터 조회 실패 시 에러가 방출된다`() = runTest {
    val errorFlow = flow<Int> {
        emit(1)
        throw RuntimeException("네트워크 오류")
    }

    errorFlow.test {
        assertEquals(1, awaitItem())       // 첫 값은 정상 방출

        // awaitError(): 다음 이벤트가 예외인지 확인하고 그 예외를 반환
        val error = awaitError()
        assertEquals("네트워크 오류", error.message)

        // 에러 이후에는 Flow가 종료된 것으로 간주 — 추가 awaitItem 불필요
    }
}
```

---

## 4. expectNoEvents — 이벤트가 없음을 검증

특정 시점까지 **아무 값도 방출되지 않아야** 하는 시나리오를 검증합니다 (디바운스, 스로틀 등).

```kotlin
import kotlinx.coroutines.flow.debounce
import kotlinx.coroutines.flow.MutableSharedFlow

@Test
fun `debounce는 짧은 간격의 입력을 무시한다`() = runTest {
    val searchQueryFlow = MutableSharedFlow<String>()
    val debouncedFlow = searchQueryFlow.debounce(500)

    debouncedFlow.test {
        searchQueryFlow.emit("k")
        searchQueryFlow.emit("ko")
        searchQueryFlow.emit("kot")

        // expectNoEvents(): 이 시점까지 아무 이벤트도 없어야 함 (아직 500ms 안 지남)
        expectNoEvents()

        advanceTimeBy(500)  // 가상 시간을 500ms 진행 → debounce 시간 경과

        // 마지막 입력값("kot")만 방출됨 (중간값들은 무시됨)
        assertEquals("kot", awaitItem())
    }
}
```

---

## 5. StateFlow / SharedFlow 테스트

ViewModel의 `StateFlow`를 테스트할 때 자주 마주치는 패턴입니다.

```kotlin
class ProductViewModel(private val repository: ProductRepository) : ViewModel() {
    private val _uiState = MutableStateFlow(ProductUiState())
    val uiState: StateFlow<ProductUiState> = _uiState.asStateFlow()

    fun loadProducts() {
        viewModelScope.launch {
            _uiState.update { it.copy(isLoading = true) }
            val products = repository.fetchProducts()
            _uiState.update { it.copy(products = products, isLoading = false) }
        }
    }
}

@Test
fun `상품 로드 시 로딩 → 완료 순서로 상태가 변경된다`() = runTest {
    val fakeRepository = FakeProductRepository()
    val viewModel = ProductViewModel(fakeRepository)

    viewModel.uiState.test {
        // StateFlow는 구독 즉시 현재 값을 방출 — 초기 상태부터 검증
        val initial = awaitItem()
        assertEquals(false, initial.isLoading)

        viewModel.loadProducts()

        // 로딩 시작 상태
        val loading = awaitItem()
        assertEquals(true, loading.isLoading)

        // 로딩 완료 상태
        val loaded = awaitItem()
        assertEquals(false, loaded.isLoading)
        assertEquals(3, loaded.products.size)

        // cancelAndIgnoreRemainingEvents(): StateFlow는 무한 스트림이므로
        // 검증이 끝나면 명시적으로 구독 취소 (awaitComplete는 호출 불가)
        cancelAndIgnoreRemainingEvents()
    }
}
```

> **StateFlow는 완료되지 않는다**: `StateFlow`는 값을 계속 유지하는 무한 스트림이라 `awaitComplete()`를 호출하면 영원히 대기합니다. 검증이 끝나면 `cancelAndIgnoreRemainingEvents()`로 마무리해야 합니다.

---

## 6. skipItems / awaitItem 조합 — 중간값 건너뛰기

관심 없는 중간 이벤트는 건너뛰고 원하는 시점만 검증할 수 있습니다.

```kotlin
@Test
fun `여러 중간 상태 중 최종 결과만 검증한다`() = runTest {
    viewModel.uiState.test {
        skipItems(1)  // 초기 상태 하나 건너뛰기

        viewModel.loadProducts()

        skipItems(1)  // 로딩 중 상태 건너뛰기

        val finalState = awaitItem()  // 최종 완료 상태만 검증
        assertEquals(3, finalState.products.size)

        cancelAndIgnoreRemainingEvents()
    }
}
```

---

## 7. 여러 Flow 동시 테스트 — turbineScope

두 개 이상의 Flow가 상호작용하는 시나리오를 검증할 때 사용합니다.

```kotlin
import app.cash.turbine.turbineScope

@Test
fun `이벤트 발생 시 상태와 알림이 함께 갱신된다`() = runTest {
    turbineScope {
        // 두 Flow를 동시에 구독하는 turbine 생성
        val stateTurbine = viewModel.uiState.testIn(backgroundScope)
        val eventTurbine = viewModel.uiEvent.testIn(backgroundScope)

        viewModel.onSubmit()

        // 각 Flow를 독립적으로 검증
        assertEquals(true, stateTurbine.awaitItem().isSubmitting)
        assertEquals(UiEvent.ShowToast("제출 완료"), eventTurbine.awaitItem())

        stateTurbine.cancelAndIgnoreRemainingEvents()
        eventTurbine.cancelAndIgnoreRemainingEvents()
    }
}
```

---

## 8. 정리

| API | 용도 |
|-----|------|
| `flow.test { }` | Flow 구독 및 이벤트 순차 검증 시작 |
| `awaitItem()` | 다음 방출값 대기 후 반환 |
| `awaitError()` | 다음 이벤트가 예외인지 확인 |
| `awaitComplete()` | Flow가 정상 종료되었는지 확인 (무한 Flow에는 사용 불가) |
| `expectNoEvents()` | 특정 시점까지 이벤트 없음을 검증 |
| `skipItems(n)` | 관심 없는 중간 이벤트 n개 건너뛰기 |
| `cancelAndIgnoreRemainingEvents()` | StateFlow 등 무한 스트림 검증 후 마무리 |

- `runTest`와 함께 사용하면 `delay`, `debounce` 등 시간 기반 연산자도 가상 시간으로 빠르게 검증
- `StateFlow`는 `awaitComplete()` 대신 반드시 `cancelAndIgnoreRemainingEvents()`로 종료
- 여러 Flow를 동시에 검증할 때는 `turbineScope` + `testIn(backgroundScope)` 사용
