---
title: (Kotlin/코틀린) Coroutines 테스트 — TestCoroutineDispatcher, runTest
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin 코루틴을 테스트하는 방법과 runTest, TestDispatcher, 가상 시간 제어, delay 스킵 패턴을 정리합니다.
---

---

## 1. 코루틴 테스트가 어려운 이유

코루틴 코드는 비동기로 실행되므로, 일반 테스트에서는 결과가 언제 나오는지 예측하기 어렵습니다.

```kotlin
// 문제 상황: 테스트가 코루틴 완료를 기다리지 않고 끝나버림
@Test
fun badTest() {
    var result = 0
    CoroutineScope(Dispatchers.Default).launch {
        delay(1000)
        result = 42
    }
    assertEquals(42, result)  // ❌ 실패! 코루틴이 끝나기 전에 assert 실행됨
}
```

`kotlinx-coroutines-test` 라이브러리는 이 문제를 해결하는 **가상 시간(Virtual Time)** 기반 테스트 도구를 제공합니다.

**의존성 추가**:
```kotlin
testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.9.0")
```

---

## 2. runTest — 코루틴 테스트의 기본

`runTest`는 테스트 코루틴을 실행하고, 내부의 모든 코루틴이 완료될 때까지 자동으로 기다립니다.

```kotlin
import kotlinx.coroutines.test.runTest
import kotlinx.coroutines.delay
import org.junit.Test
import kotlin.test.assertEquals

class UserRepositoryTest {

    @Test
    fun `사용자 정보를 정상적으로 불러온다`() = runTest {
        // runTest 블록 안은 suspend 컨텍스트 → suspend 함수 직접 호출 가능
        val repository = UserRepository(fakeApi = FakeUserApi())

        val user = repository.fetchUser("user-1")  // suspend 함수

        assertEquals("홍길동", user.name)
    }
}
```

> **runTest의 핵심**: 내부적으로 `TestDispatcher`를 사용해 **가상 시간**으로 동작합니다. `delay(1000)`이 있어도 실제로 1초를 기다리지 않고 즉시 진행됩니다 — 테스트가 빠르게 끝납니다.

---

## 3. 가상 시간 제어 — delay를 실제로 기다리지 않는 원리

```kotlin
@Test
fun `delay가 있어도 테스트는 즉시 끝난다`() = runTest {
    val startTime = System.currentTimeMillis()

    delay(10_000)  // 10초 지연 — 하지만 실제로는 기다리지 않음!

    val elapsed = System.currentTimeMillis() - startTime
    println("실제 경과 시간: ${elapsed}ms")  // 수십 ms 이내 (10초가 아님)

    // currentTime: TestScope의 가상 시간 — 실제로는 10초가 "흐른 것으로 간주"됨
    assertEquals(10_000, currentTime)
}
```

> **가상 시간이란?**: `runTest`는 `delay()` 호출을 실제로 대기하지 않고, 내부 스케줄러의 가상 시계만 앞으로 진행시킵니다. 타이머, 폴링, 재시도 로직처럼 실제 대기 시간이 긴 코드도 테스트는 밀리초 단위로 끝납니다.

---

## 4. TestDispatcher — 코루틴 실행 시점 제어

`runTest`는 기본적으로 `StandardTestDispatcher`를 사용합니다. 코루틴을 즉시 실행하지 않고 **큐에 쌓아두었다가** 명시적으로 진행시켜야 하는 경우도 있습니다.

```kotlin
import kotlinx.coroutines.test.StandardTestDispatcher
import kotlinx.coroutines.test.advanceUntilIdle

@Test
fun `StandardTestDispatcher는 launch를 즉시 실행하지 않는다`() = runTest {
    var executed = false

    launch {
        executed = true
    }

    // launch 직후에는 아직 실행되지 않음 (큐에 대기 중)
    assertEquals(false, executed)

    // advanceUntilIdle(): 대기 중인 모든 코루틴 작업을 완료할 때까지 진행
    advanceUntilIdle()

    assertEquals(true, executed)  // 이제 실행 완료
}

@Test
fun `advanceTimeBy로 특정 시간만큼만 진행`() = runTest {
    var tickCount = 0

    launch {
        repeat(5) {
            delay(1000)
            tickCount++
        }
    }

    advanceTimeBy(3000)  // 가상 시간을 3초만 진행
    runCurrent()          // 그 시점까지 예약된 작업 실행

    assertEquals(3, tickCount)  // 3번의 delay(1000)만 통과됨
}
```

---

## 5. ViewModel 테스트 — Dispatchers.Main 교체

Android ViewModel은 보통 `viewModelScope`(내부적으로 `Dispatchers.Main` 사용)를 사용합니다.
테스트 환경에는 실제 `Dispatchers.Main`이 없으므로 반드시 교체해야 합니다.

```kotlin
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.test.UnconfinedTestDispatcher
import kotlinx.coroutines.test.resetMain
import kotlinx.coroutines.test.setMain
import org.junit.After
import org.junit.Before

class MainDispatcherRule : TestWatcher() {
    private val testDispatcher = UnconfinedTestDispatcher()

    override fun starting(description: Description) {
        // Dispatchers.Main을 테스트용 디스패처로 교체 (Android 프레임워크 의존성 제거)
        Dispatchers.setMain(testDispatcher)
    }

    override fun finished(description: Description) {
        Dispatchers.resetMain()  // 원래 상태로 복구 (다른 테스트에 영향 방지)
    }
}

class ProductViewModelTest {
    @get:Rule
    val mainDispatcherRule = MainDispatcherRule()  // 모든 테스트에 자동 적용

    @Test
    fun `상품 목록 로드 성공 시 uiState가 갱신된다`() = runTest {
        val viewModel = ProductViewModel(fakeRepository)

        viewModel.loadProducts()

        val state = viewModel.uiState.value
        assertEquals(false, state.isLoading)
        assertEquals(3, state.products.size)
    }
}
```

> **UnconfinedTestDispatcher vs StandardTestDispatcher**: `UnconfinedTestDispatcher`는 코루틴을 큐에 쌓지 않고 **즉시 실행**합니다. `advanceUntilIdle()` 호출이 필요 없어 ViewModel 테스트에서 자주 사용됩니다.

---

## 6. TestScope와 예외 처리

`runTest` 내부에서 처리되지 않은 예외는 테스트를 실패시킵니다. 예외를 검증하는 방법도 알아야 합니다.

```kotlin
@Test
fun `네트워크 오류 시 예외가 발생한다`() = runTest {
    val repository = UserRepository(fakeApi = FailingUserApi())

    // assertFailsWith: suspend 함수가 특정 예외를 던지는지 검증
    val exception = assertFailsWith<NetworkException> {
        repository.fetchUser("user-1")
    }

    assertEquals("네트워크 연결 실패", exception.message)
}

@Test
fun `실패 시 에러 상태로 전환된다`() = runTest {
    val viewModel = ProductViewModel(fakeRepository = FailingRepository())

    viewModel.loadProducts()

    val state = viewModel.uiState.value
    assertNotNull(state.errorMessage)  // 예외가 UiState로 잘 변환되었는지 확인
    assertEquals(false, state.isLoading)
}
```

---

## 7. 정리

| API | 역할 |
|-----|------|
| `runTest { }` | 가상 시간 기반 코루틴 테스트 블록 |
| `advanceUntilIdle()` | 대기 중인 모든 코루틴 작업 완료까지 진행 |
| `advanceTimeBy(ms)` | 가상 시간을 지정 시간만큼만 진행 |
| `Dispatchers.setMain/resetMain` | `Dispatchers.Main`을 테스트용으로 교체 |
| `UnconfinedTestDispatcher` | 코루틴을 즉시 실행 (ViewModel 테스트에 적합) |

- `delay()`가 있는 코드도 **가상 시간** 덕분에 테스트는 즉시 끝남
- Android 프로젝트에서는 `MainDispatcherRule`로 `Dispatchers.Main` 교체를 자동화
- 예외 검증은 `assertFailsWith`로 명확하게 표현
