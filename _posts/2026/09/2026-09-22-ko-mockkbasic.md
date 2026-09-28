---
title: (Kotlin/코틀린) MockK 기초 — mock, every, verify
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin 전용 모킹 라이브러리 MockK의 mock, every, verify, relaxed mock 사용법을 예제 중심으로 정리합니다.
---

---

## 1. MockK란?

MockK는 **Kotlin을 위해 설계된** 모킹 라이브러리입니다.
Mockito는 Java 기반이라 Kotlin의 `final` 클래스, 최상위 함수, `object` 등을 모킹하기 까다롭지만, MockK는 이를 기본 지원합니다.

```
왜 Mock이 필요한가?
  실제 객체(DB, 네트워크, 외부 API) 대신
  → 가짜 객체(Mock)로 대체해 원하는 동작을 지정
  → 외부 의존성 없이 순수하게 로직만 테스트
```

**의존성 추가**:
```kotlin
testImplementation("io.mockk:mockk:1.13.12")
```

---

## 2. mock — 가짜 객체 생성

```kotlin
import io.mockk.mockk
import io.mockk.every
import org.junit.Test
import kotlin.test.assertEquals

interface UserApi {
    fun getUser(id: String): User
    fun deleteUser(id: String): Boolean
}

class UserApiTest {
    @Test
    fun `getUser 호출 시 지정한 값을 반환한다`() {
        // mockk(): UserApi의 가짜 구현체 생성 (실제 네트워크 호출 없음)
        val fakeApi = mockk<UserApi>()

        // every { }: 특정 함수 호출 시 반환할 값을 미리 지정 (스터빙)
        every { fakeApi.getUser("user-1") } returns User(id = "user-1", name = "홍길동")

        val user = fakeApi.getUser("user-1")

        assertEquals("홍길동", user.name)  // 지정한 값이 실제로 반환됨
    }
}
```

> **mock의 기본 동작**: `every`로 스터빙하지 않은 함수를 호출하면 예외가 발생합니다. 모든 상호작용을 명시적으로 정의해야 하는 "엄격한(strict)" 모킹 방식입니다.

---

## 3. every — 다양한 반환 시나리오

```kotlin
class ProductServiceTest {
    private val mockApi = mockk<ProductApi>()

    @Test
    fun `다양한 스터빙 패턴`() {
        // 1. 고정값 반환
        every { mockApi.getPrice("A001") } returns 10000

        // 2. 파라미터에 관계없이 항상 같은 값 반환
        every { mockApi.getDiscountRate(any()) } returns 0.1

        // 3. 예외를 던지도록 스터빙
        every { mockApi.getPrice("INVALID") } throws IllegalArgumentException("잘못된 상품 코드")

        // 4. 호출마다 다른 값을 순서대로 반환 (andThen 체이닝)
        every { mockApi.getStock("A001") } returns 10 andThen 5 andThen 0

        assertEquals(10000, mockApi.getPrice("A001"))
        assertEquals(0.1, mockApi.getDiscountRate("아무값"))

        assertFailsWith<IllegalArgumentException> {
            mockApi.getPrice("INVALID")
        }

        // 호출할 때마다 andThen에 지정한 순서대로 값이 반환됨
        assertEquals(10, mockApi.getStock("A001"))  // 1번째 호출
        assertEquals(5,  mockApi.getStock("A001"))  // 2번째 호출
        assertEquals(0,  mockApi.getStock("A001"))  // 3번째 호출 (품절)
    }
}
```

---

## 4. verify — 함수 호출 여부 검증

`every`가 "어떻게 동작할지"를 정의한다면, `verify`는 "실제로 호출되었는지"를 검증합니다.

```kotlin
import io.mockk.verify
import io.mockk.Runs
import io.mockk.just

class OrderServiceTest {
    private val mockNotifier = mockk<NotificationService>()
    private val mockRepository = mockk<OrderRepository>()

    @Test
    fun `주문 완료 시 알림을 발송한다`() {
        // Unit을 반환하는 함수는 just Runs로 스터빙
        every { mockRepository.save(any()) } just Runs
        every { mockNotifier.send(any(), any()) } just Runs

        val orderService = OrderService(mockRepository, mockNotifier)
        orderService.completeOrder(orderId = "order-1", userId = "user-1")

        // verify: 정확히 지정된 인자로 호출되었는지 검증
        verify { mockNotifier.send(userId = "user-1", message = "주문이 완료되었습니다") }

        // verify(exactly = n): 정확히 n번 호출되었는지 검증
        verify(exactly = 1) { mockRepository.save(any()) }

        // verify(exactly = 0): 호출되지 않았음을 검증 (음성 테스트)
        verify(exactly = 0) { mockNotifier.send(match { it != "user-1" }, any()) }
    }
}
```

> **every vs verify**: `every`는 테스트 준비 단계(Given)에서 Mock의 동작을 정의하고, `verify`는 검증 단계(Then)에서 상호작용이 예상대로 일어났는지 확인합니다.

---

## 5. relaxed mock — 모든 함수에 기본값 자동 반환

모든 함수를 일일이 스터빙하기 번거로울 때, "관심 없는 나머지 함수는 기본값을 반환"하도록 설정합니다.

```kotlin
@Test
fun `relaxed mock은 스터빙하지 않은 함수도 기본값을 반환`() {
    // relaxed = true: 스터빙 안 한 함수 호출 시 예외 대신 기본값(0, "", false, null 등) 반환
    val relaxedApi = mockk<ProductApi>(relaxed = true)

    // getPrice를 스터빙하지 않았지만 예외 없이 0 반환 (Int의 기본값)
    val price = relaxedApi.getPrice("아무거나")
    assertEquals(0, price)

    // 필요한 것만 골라서 스터빙 가능
    every { relaxedApi.getPrice("A001") } returns 5000
    assertEquals(5000, relaxedApi.getPrice("A001"))
}
```

> **일반 mock vs relaxed mock**: 여러 의존성 중 테스트와 무관한 부분이 많다면 `relaxed = true`로 편의성을 높이고, 정확한 값 검증이 중요한 핵심 로직에는 일반 `mockk()`로 엄격하게 스터빙하는 것이 좋습니다.

---

## 6. 실전 예제 — Repository 테스트

```kotlin
class UserViewModel(private val repository: UserRepository) {
    var uiState = UserUiState()
        private set

    suspend fun loadUser(id: String) {
        uiState = uiState.copy(isLoading = true)
        try {
            val user = repository.fetchUser(id)
            uiState = uiState.copy(user = user, isLoading = false)
        } catch (e: Exception) {
            uiState = uiState.copy(errorMessage = e.message, isLoading = false)
        }
    }
}

class UserViewModelTest {
    private val mockRepository = mockk<UserRepository>()

    @Test
    fun `사용자 로드 성공 시 uiState에 반영된다`() = runTest {
        // Given: 성공 시나리오 스터빙
        coEvery { mockRepository.fetchUser("user-1") } returns User("user-1", "홍길동")

        val viewModel = UserViewModel(mockRepository)

        // When
        viewModel.loadUser("user-1")

        // Then
        assertEquals("홍길동", viewModel.uiState.user?.name)
        assertEquals(false, viewModel.uiState.isLoading)
        coVerify(exactly = 1) { mockRepository.fetchUser("user-1") }
    }

    @Test
    fun `사용자 로드 실패 시 에러 메시지가 설정된다`() = runTest {
        // Given: 실패 시나리오 스터빙
        coEvery { mockRepository.fetchUser(any()) } throws RuntimeException("네트워크 오류")

        val viewModel = UserViewModel(mockRepository)

        viewModel.loadUser("user-1")

        assertEquals("네트워크 오류", viewModel.uiState.errorMessage)
    }
}
```

> **coEvery / coVerify**: `fetchUser`가 `suspend` 함수이므로 일반 `every`/`verify` 대신 코루틴 전용 `coEvery`/`coVerify`를 사용합니다.

---

## 7. 정리

| API | 용도 |
|-----|------|
| `mockk<T>()` | T 타입의 가짜 객체 생성 |
| `every { } returns` | 함수 호출 시 반환값 스터빙 |
| `every { } throws` | 함수 호출 시 예외 발생 스터빙 |
| `verify { }` | 함수가 호출되었는지 검증 |
| `mockk(relaxed = true)` | 스터빙 안 한 함수도 기본값 자동 반환 |
| `coEvery` / `coVerify` | suspend 함수 전용 스터빙/검증 |

- MockK는 Kotlin의 `final` 클래스, top-level 함수, `object`까지 모킹 가능 (Mockito와의 차이점)
- Given(every) → When(실행) → Then(verify) 순서로 테스트 구조화
- 과도한 Mock 사용은 테스트를 구현에 종속시킬 수 있음 — 핵심 협력 객체에만 사용 권장
