---
title: (Kotlin/코틀린) Kotlin 단위 테스트 — JUnit5 + Kotlin
tags: [ Kotlin ]
style: fill
color: dark
description: JUnit5를 Kotlin에서 활용하는 방법과 DisplayName, Nested, ParameterizedTest, assertAll 등 Kotlin 친화적 테스트 작성법을 정리합니다.
---

---

## 1. JUnit5와 Kotlin의 궁합

JUnit5(Jupiter)는 JUnit4보다 훨씬 유연한 테스트 작성을 지원하며, Kotlin의 문법(백틱 함수명, 람다 등)과 특히 잘 어울립니다.

```
JUnit4:                          JUnit5 + Kotlin:
fun testAddition() { }           @Test
                                  fun `두 수를 더하면 합계가 반환된다`() { }
                                  ← 한글 함수명으로 테스트 의도를 명확히 표현
```

**의존성 추가**:
```kotlin
testImplementation("org.junit.jupiter:junit-jupiter:5.11.0")
testRuntimeOnly("org.junit.platform:junit-platform-launcher")
```

---

## 2. 백틱 함수명 — 읽기 쉬운 테스트 이름

Kotlin은 백틱(`` ` ``)으로 감싸면 공백과 한글을 포함한 함수명을 만들 수 있어, 테스트 이름 자체가 문서 역할을 합니다.

```kotlin
import org.junit.jupiter.api.Test
import org.junit.jupiter.api.Assertions.assertEquals

class CalculatorTest {

    // 백틱으로 자연어에 가까운 테스트 이름 작성 — 실패 리포트에서 바로 의도 파악 가능
    @Test
    fun `두 양수를 더하면 두 수의 합이 반환된다`() {
        val result = Calculator.add(3, 5)
        assertEquals(8, result)
    }

    @Test
    fun `0으로 나누면 예외가 발생한다`() {
        val exception = assertThrows<ArithmeticException> {
            Calculator.divide(10, 0)
        }
        assertEquals("0으로 나눌 수 없습니다", exception.message)
    }
}
```

---

## 3. @DisplayName — 테스트 이름 커스터마이징

함수명과 별개로 리포트에 표시될 이름을 지정할 수 있습니다.

```kotlin
import org.junit.jupiter.api.DisplayName

@DisplayName("장바구니 계산 로직")  // 클래스 전체의 표시 이름
class ShoppingCartTest {

    @Test
    @DisplayName("상품을 담으면 총액이 증가한다")  // 이 테스트만의 표시 이름
    fun addItem_increasesTotal() {
        val cart = ShoppingCart()
        cart.addItem(Product("키보드", 50000))

        assertEquals(50000, cart.totalPrice)
    }
}
```

---

## 4. @Nested — 계층적 테스트 구조화

관련된 테스트들을 시나리오별로 그룹화해 가독성을 높입니다.

```kotlin
import org.junit.jupiter.api.Nested

class UserAccountTest {

    private lateinit var account: UserAccount

    @BeforeEach
    fun setUp() {
        account = UserAccount(balance = 10000)
    }

    // @Nested: 안쪽 클래스로 테스트를 논리적 그룹으로 묶음
    @Nested
    @DisplayName("입금 시나리오")
    inner class DepositScenarios {

        @Test
        fun `양수 금액을 입금하면 잔액이 증가한다`() {
            account.deposit(5000)
            assertEquals(15000, account.balance)
        }

        @Test
        fun `음수 금액을 입금하려 하면 예외가 발생한다`() {
            assertThrows<IllegalArgumentException> {
                account.deposit(-1000)
            }
        }
    }

    @Nested
    @DisplayName("출금 시나리오")
    inner class WithdrawScenarios {

        @Test
        fun `잔액 이내 금액은 정상 출금된다`() {
            account.withdraw(3000)
            assertEquals(7000, account.balance)
        }

        @Test
        fun `잔액을 초과해 출금하면 예외가 발생한다`() {
            assertThrows<InsufficientBalanceException> {
                account.withdraw(20000)
            }
        }
    }
}
```

> **@Nested의 장점**: 테스트 리포트에서 "입금 시나리오"와 "출금 시나리오"가 계층 구조로 표시되어, 관련 테스트를 한눈에 그룹으로 파악할 수 있습니다.

---

## 5. @ParameterizedTest — 여러 입력값을 하나의 테스트로

같은 로직을 다른 입력값으로 반복 검증할 때, 테스트 코드 중복 없이 표현합니다.

```kotlin
import org.junit.jupiter.params.ParameterizedTest
import org.junit.jupiter.params.provider.ValueSource
import org.junit.jupiter.params.provider.CsvSource

class EmailValidatorTest {

    // ValueSource: 단일 값 목록을 순서대로 파라미터에 주입
    @ParameterizedTest
    @ValueSource(strings = ["a@b.com", "user.name@example.co.kr", "test123@gmail.com"])
    fun `유효한 이메일 형식은 통과한다`(email: String) {
        assertTrue(EmailValidator.isValid(email))
    }

    @ParameterizedTest
    @ValueSource(strings = ["invalid", "@nodomain.com", "no-at-sign.com", ""])
    fun `잘못된 이메일 형식은 거부된다`(email: String) {
        assertFalse(EmailValidator.isValid(email))
    }

    // CsvSource: 여러 파라미터를 쌍으로 주입 (입력값, 기대값)
    @ParameterizedTest
    @CsvSource(
        "10, 20, 30",
        "0, 0, 0",
        "-5, 5, 0",
        "100, -50, 50"
    )
    fun `두 수를 더하면 세 번째 값과 같다`(a: Int, b: Int, expected: Int) {
        assertEquals(expected, Calculator.add(a, b))
    }
}
```

> **테스트 개수 확인**: `@ValueSource`에 값 3개, `@CsvSource`에 4개를 주면 각각 3번, 4번 실행되어 리포트에 개별 결과로 표시됩니다. 반복문으로 감싼 하나의 테스트보다 실패 지점을 훨씬 명확히 파악할 수 있습니다.

---

## 6. assertAll — 여러 assertion을 한 번에 검증

기본적으로 `assertEquals`가 실패하면 바로 테스트가 중단되어, 그 아래 검증은 실행되지 않습니다.
`assertAll`은 모든 검증을 실행한 뒤 실패한 항목을 **한꺼번에** 보고합니다.

```kotlin
import org.junit.jupiter.api.Assertions.assertAll

@Test
fun `사용자 생성 시 모든 필드가 올바르게 설정된다`() {
    val user = User.create(name = "홍길동", email = "hong@example.com", age = 30)

    // assertAll: 하나가 실패해도 나머지 검증을 계속 진행 후 모든 실패를 함께 리포트
    assertAll(
        "생성된 사용자 검증",
        { assertEquals("홍길동", user.name) },
        { assertEquals("hong@example.com", user.email) },
        { assertEquals(30, user.age) },
        { assertNotNull(user.id) },
        { assertTrue(user.createdAt <= System.currentTimeMillis()) }
    )
}
```

> **왜 유용한가?**: `assertAll` 없이 5개 검증을 순서대로 작성하면, 첫 번째 실패 시 나머지 4개는 실행조차 안 되어 "무엇이 더 잘못됐는지" 한 번에 알 수 없습니다. `assertAll`은 모든 필드를 한 번의 실행으로 검증합니다.

---

## 7. @BeforeEach / @AfterEach — 테스트 전후 공통 작업

```kotlin
import org.junit.jupiter.api.BeforeEach
import org.junit.jupiter.api.AfterEach

class DatabaseTest {

    private lateinit var database: TestDatabase

    // @BeforeEach: 각 테스트 실행 직전에 매번 실행 (테스트 간 상태 격리)
    @BeforeEach
    fun setUp() {
        database = TestDatabase.createInMemory()
        database.seed(sampleUsers)
    }

    // @AfterEach: 각 테스트 실행 직후에 매번 실행 (자원 정리)
    @AfterEach
    fun tearDown() {
        database.close()
    }

    @Test
    fun `사용자 조회가 정상 동작한다`() {
        val user = database.findUser("user-1")
        assertNotNull(user)
    }
}
```

---

## 8. 정리

| 어노테이션/함수 | 용도 |
|----------------|------|
| 백틱 함수명 | 테스트 의도를 자연어로 명확히 표현 |
| `@DisplayName` | 리포트에 표시될 커스텀 이름 지정 |
| `@Nested` | 관련 테스트를 시나리오별로 계층화 |
| `@ParameterizedTest` | 여러 입력값을 하나의 테스트로 반복 검증 |
| `assertAll` | 여러 검증을 모두 실행 후 한꺼번에 실패 보고 |
| `@BeforeEach` / `@AfterEach` | 테스트마다 반복되는 준비/정리 작업 |

- Kotlin의 백틱 함수명 + JUnit5의 `@DisplayName`을 조합하면 테스트가 살아있는 문서가 됨
- `@ParameterizedTest`로 반복 검증 코드를 줄이고 실패 지점을 명확히 표시
- `@Nested`로 테스트 클래스가 커질수록 생기는 복잡도를 시나리오 단위로 관리
