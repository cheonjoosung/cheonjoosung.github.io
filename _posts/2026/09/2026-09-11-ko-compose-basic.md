---
title: (Android/Compose) Jetpack Compose 기초 — Composable, State, @Preview
tags: [ Android, Compose ]
style: fill
color: dark
description: Jetpack Compose의 핵심 개념인 Composable 함수, State 관리, @Preview 어노테이션을 예제 중심으로 정리합니다.
---

---

## 1. Jetpack Compose란?

Jetpack Compose는 Android의 **선언형 UI 프레임워크**입니다.
기존 XML 레이아웃 방식과 달리, **코드로 UI를 직접 기술**합니다.

```
기존 XML 방식:                  Compose 방식:
activity_main.xml (UI 정의)    @Composable fun MyScreen() {
    ↓                              Text("Hello")
MainActivity.kt (UI 제어)      }
  findViewById, setText 등
```

상태(State)가 변경되면 UI가 자동으로 재구성(Recomposition)됩니다—뷰를 직접 조작하지 않아도 됩니다.

---

## 2. @Composable 함수

UI를 구성하는 기본 단위입니다. `@Composable` 어노테이션을 붙인 함수가 화면을 그립니다.

```kotlin
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.ui.tooling.preview.Preview

// @Composable: 이 함수가 UI를 그린다는 선언
@Composable
fun Greeting(name: String) {
    // Text: 텍스트를 화면에 표시하는 내장 Composable
    Text(text = "안녕하세요, $name!")
}

// 중첩 Composable: Composable 안에서 다른 Composable을 호출
@Composable
fun UserCard(name: String, email: String) {
    // Column: 자식들을 세로로 배치
    Column(modifier = Modifier.padding(16.dp)) {
        Text(text = name, style = MaterialTheme.typography.titleMedium)
        Text(text = email, style = MaterialTheme.typography.bodySmall)
    }
}
```

> **Composable 규칙**: Composable 함수는 반드시 다른 Composable 함수 안에서만 호출할 수 있습니다. 일반 함수에서 직접 호출할 수 없습니다.

---

## 3. State — UI 상태 관리

Compose에서 화면 변경의 핵심은 **State**입니다.
`mutableStateOf`로 만든 상태가 바뀌면, 해당 상태를 읽는 Composable이 자동으로 다시 그려집니다(Recomposition).

```kotlin
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.setValue

@Composable
fun Counter() {
    // remember: 리컴포지션이 일어나도 값을 유지 (매번 초기화되지 않음)
    // mutableStateOf: 값이 바뀌면 이 Composable을 재구성하도록 트래킹
    var count by remember { mutableStateOf(0) }
    // by 키워드: getValue/setValue를 자동으로 위임 → count.value 대신 count로 사용 가능

    Column(
        horizontalAlignment = Alignment.CenterHorizontally,
        modifier = Modifier.padding(16.dp)
    ) {
        Text(
            text = "카운트: $count",
            style = MaterialTheme.typography.headlineMedium
        )
        Spacer(modifier = Modifier.height(8.dp))
        Button(onClick = { count++ }) {  // 클릭 시 count가 증가 → 자동 재구성
            Text("증가")
        }
        Button(onClick = { count-- }) {
            Text("감소")
        }
    }
}
```

> **remember란?**: `remember`가 없으면 Recomposition(재구성)이 일어날 때마다 `mutableStateOf(0)`이 다시 실행되어 값이 초기화됩니다. `remember`는 최초 1회만 실행하고 이후 재사용합니다.

---

## 4. Recomposition 이해

상태가 변하면 **해당 상태를 읽는 부분만** 선택적으로 재구성됩니다.

```kotlin
@Composable
fun RecompositionExample() {
    var text by remember { mutableStateOf("") }

    Column {
        // TextField: 사용자 입력을 받는 Composable
        TextField(
            value = text,
            onValueChange = { text = it },  // 입력할 때마다 text 상태 업데이트
            label = { Text("입력하세요") }
        )

        // text가 변경될 때마다 이 부분만 재구성됨
        Text("입력한 내용: $text")
        Text("글자 수: ${text.length}자")

        // 이 부분은 text와 무관하므로 재구성되지 않음
        Text("항상 고정된 텍스트", color = Color.Gray)
    }
}
```

---

## 5. @Preview — 에디터에서 UI 미리보기

Android Studio에서 실기기/에뮬레이터 없이 UI를 바로 확인하는 기능입니다.

```kotlin
// @Preview: Android Studio Design 탭에서 미리보기 가능
@Preview(
    name = "라이트 모드",        // 미리보기 이름
    showBackground = true,      // 배경 표시
    backgroundColor = 0xFFFFFFFF // 흰색 배경
)
@Composable
fun GreetingPreview() {
    MyAppTheme {  // 앱 테마를 감싸야 Material 스타일 적용
        Greeting(name = "Android 개발자")
    }
}

// 여러 화면 크기를 동시에 미리보기
@Preview(name = "폰 세로", device = Devices.PHONE, showSystemUi = true)
@Preview(name = "태블릿", device = Devices.TABLET, showSystemUi = true)
@Composable
fun ResponsivePreview() {
    MyAppTheme {
        Counter()
    }
}

// 다크 모드 미리보기
@Preview(
    name = "다크 모드",
    uiMode = Configuration.UI_MODE_NIGHT_YES,
    showBackground = true
)
@Composable
fun DarkModePreview() {
    MyAppTheme {
        UserCard(name = "홍길동", email = "hong@example.com")
    }
}
```

---

## 6. 실전 — 간단한 로그인 화면

배운 개념을 조합해 실제 화면을 만들어 봅니다.

```kotlin
@Composable
fun LoginScreen(onLoginClick: (String, String) -> Unit) {
    // 각 입력값을 별도 상태로 관리
    var email by remember { mutableStateOf("") }
    var password by remember { mutableStateOf("") }

    Column(
        modifier = Modifier
            .fillMaxSize()      // 화면 전체 크기
            .padding(24.dp),
        verticalArrangement = Arrangement.Center,
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        Text(
            text = "로그인",
            style = MaterialTheme.typography.headlineLarge
        )
        Spacer(modifier = Modifier.height(32.dp))

        // 이메일 입력
        OutlinedTextField(
            value = email,
            onValueChange = { email = it },
            label = { Text("이메일") },
            keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Email),
            modifier = Modifier.fillMaxWidth()
        )
        Spacer(modifier = Modifier.height(16.dp))

        // 비밀번호 입력 (입력값 숨김)
        OutlinedTextField(
            value = password,
            onValueChange = { password = it },
            label = { Text("비밀번호") },
            visualTransformation = PasswordVisualTransformation(),
            modifier = Modifier.fillMaxWidth()
        )
        Spacer(modifier = Modifier.height(24.dp))

        // 버튼: 이메일/비밀번호가 모두 입력된 경우에만 활성화
        Button(
            onClick = { onLoginClick(email, password) },
            enabled = email.isNotBlank() && password.isNotBlank(),
            modifier = Modifier.fillMaxWidth()
        ) {
            Text("로그인")
        }
    }
}

@Preview(showBackground = true)
@Composable
fun LoginScreenPreview() {
    MyAppTheme {
        LoginScreen(onLoginClick = { _, _ -> })
    }
}
```

---

## 7. 정리

| 개념 | 역할 |
|------|------|
| `@Composable` | UI를 그리는 함수 선언 |
| `remember { }` | 리컴포지션 사이에서 값 유지 |
| `mutableStateOf` | 변경 추적 상태 생성 → 값 변경 시 재구성 |
| `by` 위임 | `.value` 없이 상태 값에 직접 접근 |
| `@Preview` | Android Studio에서 UI 즉시 미리보기 |

- Compose는 **선언형**: "무엇을 보여줄지"만 기술하면 프레임워크가 자동으로 그림
- 상태가 변하면 해당 상태를 **읽는 부분만** 재구성 (전체 재구성 아님)
- `@Preview`를 적극 활용하면 에뮬레이터 없이도 빠른 UI 개발 가능
