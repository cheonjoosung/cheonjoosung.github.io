---
title: (Android/Compose) Compose UI 테스트 — ComposeTestRule
tags: [ Android, Compose ]
style: fill
color: dark
description: Jetpack Compose UI 테스트 작성법과 ComposeTestRule, Finder/Assertion/Action API, 상태 및 비동기 테스트 패턴을 정리합니다.
---

---

## 1. Compose UI 테스트란?

Compose UI 테스트는 **실제 화면을 렌더링하지 않고도** Composable의 동작을 검증하는 테스트입니다.
"화면에 텍스트가 있는가", "버튼을 클릭하면 상태가 바뀌는가"를 코드로 자동 검증합니다.

```
테스트 흐름:
setContent { 테스트할 Composable }
    ↓
Finder로 노드 찾기 (onNodeWithText, onNodeWithTag 등)
    ↓
Action 수행 (performClick, performTextInput 등)
    ↓
Assertion 검증 (assertIsDisplayed, assertTextEquals 등)
```

**의존성 추가 (build.gradle.kts)**:
```kotlin
androidTestImplementation("androidx.compose.ui:ui-test-junit4:1.7.0")
debugImplementation("androidx.compose.ui:ui-test-manifest:1.7.0")
```

---

## 2. ComposeTestRule 기본 설정

`createComposeRule()`로 테스트 대상 Composable을 렌더링할 환경을 만듭니다.

```kotlin
import androidx.compose.ui.test.junit4.createComposeRule
import org.junit.Rule
import org.junit.Test

class CounterTest {
    // @get:Rule: JUnit이 각 테스트 전에 자동으로 초기화하는 테스트 규칙
    @get:Rule
    val composeTestRule = createComposeRule()

    @Test
    fun counter_incrementsOnClick() {
        // setContent: 실제 Activity 없이 Composable을 테스트 환경에 렌더링
        composeTestRule.setContent {
            Counter()  // 이전 글에서 만든 카운터 Composable
        }

        // 초기 상태 검증: "카운트: 0" 텍스트가 화면에 있는지 확인
        composeTestRule.onNodeWithText("카운트: 0").assertIsDisplayed()

        // "증가" 버튼을 찾아 클릭 수행
        composeTestRule.onNodeWithText("증가").performClick()

        // 클릭 후 상태 변화 검증
        composeTestRule.onNodeWithText("카운트: 1").assertIsDisplayed()
    }
}
```

---

## 3. Finder API — 노드 찾기

원하는 Composable을 화면 트리에서 찾는 방법들입니다.

```kotlin
@Test
fun finder_examples() {
    composeTestRule.setContent {
        LoginScreen(onLoginClick = { _, _ -> })
    }

    // 텍스트로 찾기 — 정확히 일치
    composeTestRule.onNodeWithText("로그인")

    // 텍스트로 찾기 — 부분 일치 허용
    composeTestRule.onNodeWithText("로그", substring = true)

    // contentDescription으로 찾기 (아이콘, 이미지 등 텍스트 없는 요소)
    composeTestRule.onNodeWithContentDescription("뒤로 가기")

    // testTag로 찾기 — 가장 안정적인 방법 (텍스트 변경/다국어에 영향 없음)
    composeTestRule.onNodeWithTag("email_input")

    // 여러 개가 매칭될 때 — onAllNodesWithText로 리스트 조회
    composeTestRule.onAllNodesWithText("삭제")[0].performClick()  // 첫 번째 항목 클릭

    // 조건 조합: 버튼 역할이면서 활성화된 노드만
    composeTestRule.onNode(
        hasClickAction() and isEnabled()
    )
}
```

**testTag로 찾기 위한 Composable 설정**:

```kotlin
import androidx.compose.ui.platform.testTag

@Composable
fun LoginScreen(onLoginClick: (String, String) -> Unit) {
    var email by remember { mutableStateOf("") }

    OutlinedTextField(
        value = email,
        onValueChange = { email = it },
        // testTag: 테스트에서만 사용하는 식별자, UI에는 영향 없음
        modifier = Modifier.testTag("email_input")
    )
}
```

> **testTag를 권장하는 이유**: 텍스트 기반 검색(`onNodeWithText`)은 다국어 지원이나 문구 수정 시 테스트가 깨집니다. `testTag`는 UI 문구와 무관하게 안정적으로 요소를 식별합니다.

---

## 4. Action API — 사용자 동작 시뮬레이션

```kotlin
@Test
fun login_formInteraction() {
    var loggedInEmail = ""
    var loggedInPassword = ""

    composeTestRule.setContent {
        LoginScreen(onLoginClick = { email, password ->
            loggedInEmail = email
            loggedInPassword = password
        })
    }

    // performTextInput: 텍스트 필드에 문자열 입력
    composeTestRule.onNodeWithTag("email_input").performTextInput("test@example.com")
    composeTestRule.onNodeWithTag("password_input").performTextInput("password123")

    // performClick: 클릭 이벤트 발생
    composeTestRule.onNodeWithText("로그인").performClick()

    // 콜백이 올바른 인자로 호출되었는지 검증
    assertEquals("test@example.com", loggedInEmail)
    assertEquals("password123", loggedInPassword)
}

@Test
fun swipe_and_scroll_actions() {
    composeTestRule.setContent { MessageList(sampleMessages) }

    // performScrollToNode: 특정 노드가 보일 때까지 스크롤
    composeTestRule.onNodeWithTag("message_list")
        .performScrollToNode(hasText("마지막 메시지"))

    // performTouchInput { swipeLeft() }: 스와이프 제스처 시뮬레이션 (삭제 등)
    composeTestRule.onNodeWithTag("item_5").performTouchInput { swipeLeft() }
}
```

---

## 5. Assertion API — 상태 검증

```kotlin
@Test
fun assertion_examples() {
    composeTestRule.setContent { ProductCard(product = sampleProduct) }

    // 화면에 표시되는지 확인
    composeTestRule.onNodeWithText(sampleProduct.name).assertIsDisplayed()

    // 존재하지 않음을 확인 (조건부 렌더링 검증)
    composeTestRule.onNodeWithText("품절").assertDoesNotExist()

    // 활성화/비활성화 상태 확인
    composeTestRule.onNodeWithText("장바구니 담기").assertIsEnabled()

    // 텍스트 내용 정확히 일치 확인
    composeTestRule.onNodeWithTag("price_text").assertTextEquals("29,000원")

    // 개수 검증
    composeTestRule.onAllNodesWithTag("product_item").assertCountEquals(5)

    // 체크박스/스위치 등의 선택 상태 확인
    composeTestRule.onNodeWithTag("agree_checkbox").assertIsOff()
}
```

---

## 6. 비동기 상태 대기 — waitUntil

네트워크 응답, 애니메이션처럼 즉시 반영되지 않는 상태를 기다려야 할 때 사용합니다.

```kotlin
@Test
fun productList_loadsAfterDelay() {
    composeTestRule.setContent {
        ProductListScreen(viewModel = FakeProductViewModel())  // 테스트용 가짜 ViewModel
    }

    // 로딩 인디케이터가 처음엔 보임
    composeTestRule.onNodeWithTag("loading_indicator").assertIsDisplayed()

    // waitUntil: 조건이 true가 될 때까지 최대 timeoutMillis 동안 대기
    // Compose 테스트에서는 Thread.sleep() 대신 반드시 이 방식을 사용해야 함
    composeTestRule.waitUntil(timeoutMillis = 5000) {
        composeTestRule.onAllNodesWithTag("product_item")
            .fetchSemanticsNodes().isNotEmpty()  // 상품 아이템이 하나라도 나타났는지 확인
    }

    // 로딩이 끝났으므로 인디케이터는 사라져야 함
    composeTestRule.onNodeWithTag("loading_indicator").assertDoesNotExist()
}
```

> **Thread.sleep()을 쓰면 안 되는 이유**: Compose 테스트는 자체적으로 "Idle 상태"를 추적합니다. `waitUntil`은 Compose의 유휴 상태와 동기화되어 정확한 타이밍에 검증하지만, `sleep`은 불필요하게 느리거나 타이밍이 어긋나 flaky(불안정)한 테스트를 만듭니다.

---

## 7. 상태 호이스팅과 테스트 용이성

State Hoisting 패턴을 적용한 Composable은 테스트가 훨씬 간단해집니다.

```kotlin
// Stateless Composable — 외부에서 상태 주입 가능 → 테스트하기 쉬움
@Test
fun statelessCounter_displaysCorrectCount() {
    composeTestRule.setContent {
        // 원하는 count 값을 직접 주입해 다양한 상태를 손쉽게 테스트
        StatelessCounter(count = 42, onIncrement = {})
    }

    composeTestRule.onNodeWithText("카운트: 42").assertIsDisplayed()
}

@Test
fun statelessCounter_callsCallback() {
    var incrementCalled = false

    composeTestRule.setContent {
        StatelessCounter(count = 0, onIncrement = { incrementCalled = true })
    }

    composeTestRule.onNodeWithText("증가").performClick()

    assertTrue(incrementCalled)  // 콜백이 실제로 호출되었는지 검증
}
```

> **Stateful Composable을 직접 테스트하기 어려운 이유**: 내부에 `remember { mutableStateOf(0) }`가 있으면 외부에서 초기 상태를 42처럼 임의로 설정할 수 없습니다. State Hoisting을 적용하면 이런 제약이 사라집니다.

---

## 8. 정리

| API 분류 | 대표 함수 | 용도 |
|---------|-----------|------|
| Finder | `onNodeWithText`, `onNodeWithTag` | 화면에서 노드 찾기 |
| Action | `performClick`, `performTextInput` | 사용자 동작 시뮬레이션 |
| Assertion | `assertIsDisplayed`, `assertTextEquals` | 상태/내용 검증 |
| 비동기 대기 | `waitUntil` | 네트워크/애니메이션 완료 대기 |

- `testTag`를 적극 활용해 텍스트 변경에 영향받지 않는 안정적인 테스트 작성
- `Thread.sleep()` 대신 `waitUntil`로 비동기 상태를 안전하게 검증
- State Hoisting 패턴을 적용한 Composable일수록 테스트 작성이 쉬워짐
