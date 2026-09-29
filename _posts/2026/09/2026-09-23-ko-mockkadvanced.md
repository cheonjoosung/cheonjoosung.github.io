---
title: (Kotlin/코틀린) MockK 심화 — spy, slot, captureArgument
tags: [ Kotlin ]
style: fill
color: dark
description: MockK의 심화 기능인 spy, slot, capture, verify 순서 검증, 정적/객체 모킹 방법을 정리합니다.
---

---

## 1. spy — 실제 객체를 부분적으로 모킹

`mock`은 모든 함수가 가짜로 동작하지만, `spy`는 **실제 객체를 감싸서** 특정 함수만 선택적으로 재정의합니다.

```kotlin
import io.mockk.spyk
import io.mockk.every

class Calculator {
    fun add(a: Int, b: Int) = a + b
    fun multiply(a: Int, b: Int) = a * b
    fun complexOperation(a: Int, b: Int) = add(a, b) * multiply(a, b)
}

@Test
fun `spy는 재정의하지 않은 함수는 실제 로직을 수행`() {
    // spyk: 실제 Calculator 인스턴스를 감싼 spy 객체 생성
    val calculatorSpy = spyk(Calculator())

    // add()만 가짜 값으로 재정의 — 나머지 함수는 실제 구현 그대로 동작
    every { calculatorSpy.add(2, 3) } returns 100

    assertEquals(100, calculatorSpy.add(2, 3))       // 스터빙된 값
    assertEquals(6,   calculatorSpy.multiply(2, 3))  // 실제 로직 그대로 (재정의 안 함)

    // complexOperation 내부에서 add()를 호출 → 스터빙된 100이 사용됨
    // 100 * 6 = 600 (실제 add였다면 5 * 6 = 30)
    assertEquals(600, calculatorSpy.complexOperation(2, 3))
}
```

> **mock vs spy**: `mock`은 완전히 가짜 객체(모든 함수가 스터빙 필요), `spy`는 실제 객체를 기반으로 **일부만** 가짜로 바꿉니다. 레거시 클래스의 특정 부분만 격리하고 싶을 때 유용합니다.

---

## 2. slot — 전달된 인자 캡처

함수에 전달된 실제 인자값을 검증하고 싶을 때 `slot`을 사용합니다.

```kotlin
import io.mockk.slot
import io.mockk.CapturingSlot

@Test
fun `저장 시 전달된 객체의 내용을 검증한다`() {
    val mockRepository = mockk<UserRepository>()
    val userSlot = slot<User>()  // User 타입 인자를 담을 슬롯

    // capture(slot): save()에 전달된 인자를 slot에 저장하도록 설정
    every { mockRepository.save(capture(userSlot)) } just Runs

    val userService = UserService(mockRepository)
    userService.registerUser(name = "홍길동", email = "hong@example.com")

    // slot.captured: 실제로 전달된 인자값에 접근
    val savedUser = userSlot.captured
    assertEquals("홍길동", savedUser.name)
    assertEquals("hong@example.com", savedUser.email)
    assertNotNull(savedUser.id)  // registerUser 내부에서 자동 생성된 ID까지 검증 가능
}
```

> **verify(match {})와의 차이**: 단순 비교라면 `verify { mockRepository.save(match { it.name == "홍길동" }) }`로도 충분합니다. `slot`은 캡처한 객체를 **여러 필드에 걸쳐 상세히 검증**하거나, 이후 로직에서 재사용하고 싶을 때 유리합니다.

---

## 3. mutableListOf + capture — 여러 번 호출된 인자 모두 캡처

```kotlin
import io.mockk.mutableListOf

@Test
fun `여러 번 호출된 알림 내용을 모두 검증한다`() {
    val mockNotifier = mockk<NotificationService>()
    val capturedMessages = mutableListOf<String>()

    // capture(list): 호출될 때마다 인자가 리스트에 순서대로 누적됨
    every { mockNotifier.send(any(), capture(capturedMessages)) } just Runs

    val batchService = BatchNotificationService(mockNotifier)
    batchService.notifyAll(
        userIds = listOf("u1", "u2", "u3"),
        template = "안녕하세요, {name}님"
    )

    // 3번 호출되었고, 각 호출의 메시지가 순서대로 저장됨
    assertEquals(3, capturedMessages.size)
    assertTrue(capturedMessages[0].contains("u1") || capturedMessages[0].contains("안녕"))
}
```

---

## 4. verify 순서 검증 — verifyOrder / verifySequence

여러 함수가 **정확한 순서로 호출**되었는지 검증해야 할 때 사용합니다.

```kotlin
import io.mockk.verifyOrder
import io.mockk.verifySequence

class PaymentServiceTest {
    private val mockValidator = mockk<PaymentValidator>(relaxed = true)
    private val mockGateway = mockk<PaymentGateway>(relaxed = true)
    private val mockLogger = mockk<Logger>(relaxed = true)

    @Test
    fun `결제는 검증 후 처리되고 마지막에 로그가 남는다`() {
        val service = PaymentService(mockValidator, mockGateway, mockLogger)

        service.processPayment(amount = 10000)

        // verifyOrder: 지정한 호출들이 "이 순서대로" 일어났는지 검증
        // (다른 호출이 중간에 끼어 있어도 상대적 순서만 맞으면 통과)
        verifyOrder {
            mockValidator.validate(10000)
            mockGateway.charge(10000)
            mockLogger.log("결제 완료: 10000원")
        }
    }

    @Test
    fun `정확히 이 순서로만 호출되어야 한다`() {
        val service = PaymentService(mockValidator, mockGateway, mockLogger)
        service.processPayment(amount = 5000)

        // verifySequence: 이 세 호출이 "다른 호출 없이 연속으로" 정확히 이 순서로만 발생
        verifySequence {
            mockValidator.validate(5000)
            mockGateway.charge(5000)
            mockLogger.log("결제 완료: 5000원")
        }
    }
}
```

> **verifyOrder vs verifySequence**: `verifyOrder`는 상대적 순서만 확인(중간에 다른 호출 허용), `verifySequence`는 지정한 호출들만 정확히 그 순서로 전체를 구성해야 통과합니다.

---

## 5. object / 최상위 함수 모킹 — mockkObject, mockkStatic

Kotlin 고유의 `object` 싱글턴이나 최상위 함수도 MockK로 모킹할 수 있습니다 (Mockito에서는 어려운 영역입니다).

```kotlin
import io.mockk.mockkObject
import io.mockk.unmockkObject
import io.mockk.mockkStatic

// 싱글턴 object
object DateProvider {
    fun today(): LocalDate = LocalDate.now()
}

@Test
fun `object 싱글턴의 함수를 모킹한다`() {
    // mockkObject: 해당 object의 함수 호출을 가로챌 수 있게 설정
    mockkObject(DateProvider)
    every { DateProvider.today() } returns LocalDate.of(2026, 1, 1)

    val result = DateProvider.today()
    assertEquals(LocalDate.of(2026, 1, 1), result)

    // 테스트 후 원래 상태로 복구 (다른 테스트에 영향 방지 — 필수!)
    unmockkObject(DateProvider)
}

@Test
fun `최상위 확장 함수를 모킹한다`() {
    // mockkStatic: 특정 클래스에 대한 확장 함수(최상위 함수)를 모킹
    mockkStatic("kotlin.io.FilesKt")

    val mockFile = mockk<File>()
    every { mockFile.readText() } returns "테스트 내용"

    assertEquals("테스트 내용", mockFile.readText())

    unmockkStatic("kotlin.io.FilesKt")
}
```

> **object/static 모킹 주의사항**: 전역 상태를 변경하므로 반드시 `unmockkObject`/`unmockkStatic`으로 복구해야 합니다. `@After`에 `unmockkAll()`을 넣어 모든 테스트 후 자동 정리하는 것이 안전합니다.

```kotlin
@After
fun tearDown() {
    unmockkAll()  // 이 테스트 클래스에서 사용한 모든 mock/spy/object 모킹 초기화
}
```

---

## 6. 실전 예제 — 복합 시나리오

```kotlin
class OrderServiceTest {
    private val mockRepository = mockk<OrderRepository>()
    private val mockPaymentGateway = mockk<PaymentGateway>()
    private val orderSlot = slot<Order>()

    @Test
    fun `주문 생성 시 올바른 상태로 저장되고 결제가 순서대로 처리된다`() {
        every { mockRepository.save(capture(orderSlot)) } just Runs
        every { mockPaymentGateway.charge(any()) } returns PaymentResult.Success

        val service = OrderService(mockRepository, mockPaymentGateway)
        service.createOrder(productId = "P001", amount = 30000)

        // 캡처한 인자로 저장 내용 검증
        assertEquals("P001", orderSlot.captured.productId)
        assertEquals(OrderStatus.PENDING, orderSlot.captured.status)

        // 호출 순서 검증: 저장 먼저, 결제는 그 다음
        verifyOrder {
            mockRepository.save(any())
            mockPaymentGateway.charge(30000)
        }
    }
}
```

---

## 7. 정리

| API | 용도 |
|-----|------|
| `spyk(실제객체)` | 실제 객체 기반, 일부 함수만 재정의 |
| `slot<T>()` + `capture()` | 전달된 인자값을 캡처해 상세 검증 |
| `mutableListOf` + `capture()` | 여러 번 호출된 인자를 모두 캡처 |
| `verifyOrder { }` | 상대적 호출 순서 검증 |
| `verifySequence { }` | 정확한 전체 호출 순서 검증 |
| `mockkObject` / `mockkStatic` | Kotlin object, 최상위 함수 모킹 |
| `unmockkAll()` | 모든 모킹 상태 초기화 (테스트 후 필수) |

- `spy`는 레거시 코드의 일부만 격리해 테스트할 때 유용
- `slot`/`capture`로 단순 값 비교를 넘어 복잡한 객체의 내부 상태까지 검증 가능
- object/static 모킹은 전역 상태 변경이므로 반드시 `unmockkAll()`로 정리
