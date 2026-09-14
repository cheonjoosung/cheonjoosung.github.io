---
title: (Android/Compose) Compose 상태 관리 — remember, mutableStateOf, State Hoisting
tags: [ Android, Compose ]
style: fill
color: dark
description: Jetpack Compose의 상태 관리 핵심인 remember, mutableStateOf, rememberSaveable, State Hoisting 패턴을 정리합니다.
---

---

## 1. Compose에서 상태란?

Compose에서 **State(상태)** 는 UI를 결정하는 모든 값입니다.
상태가 바뀌면 해당 상태를 읽는 Composable이 자동으로 재구성(Recomposition)됩니다.

```
상태(State) 변경
    ↓
해당 상태를 읽는 Composable 재구성
    ↓
새 상태에 맞는 UI 렌더링
```

---

## 2. remember — 리컴포지션 사이에서 값 유지

`remember`가 없으면 Recomposition마다 값이 초기화됩니다.

```kotlin
@Composable
fun RememberExample() {
    // remember 없이: Recomposition마다 0으로 초기화 → 버튼이 동작하지 않음
    var badCount = mutableStateOf(0)  // ❌ remember 없음

    // remember 있음: 최초 1회만 초기화, 이후 Recomposition에서 값 유지
    var goodCount by remember { mutableStateOf(0) }  // ✅

    Column {
        Text("동작 안 함: ${badCount.value}")
        Text("동작함: $goodCount")
        Button(onClick = { goodCount++ }) { Text("증가") }
    }
}
```

> **언제 재구성되나?** `goodCount`가 변경될 때만 이 Composable이 재구성됩니다. `remember`는 재구성 사이에 값을 보존해 주는 메모리입니다.

---

## 3. mutableStateOf — 변경 추적 상태

`mutableStateOf`는 값 변경을 Compose가 **추적**할 수 있는 상태 객체를 만듭니다.

```kotlin
@Composable
fun StateTypes() {
    // 기본형: State 객체를 직접 다루는 방식 (.value로 접근)
    val countState = remember { mutableStateOf(0) }
    Text("카운트: ${countState.value}")
    Button(onClick = { countState.value++ }) { Text("증가") }

    // by 위임: getValue/setValue를 자동 위임 → .value 없이 직접 접근 (권장)
    var count by remember { mutableStateOf(0) }
    Text("카운트: $count")        // countState.value 대신 count
    Button(onClick = { count++ }) { Text("증가") }  // countState.value++ 대신 count++

    // 다양한 타입의 상태
    var text    by remember { mutableStateOf("") }
    var isCheck by remember { mutableStateOf(false) }
    var items   by remember { mutableStateOf(listOf<String>()) }
}
```

---

## 4. rememberSaveable — 화면 회전에도 유지

`remember`는 Recomposition에서는 값을 유지하지만, **화면 회전이나 앱이 재시작되면 초기화**됩니다.
`rememberSaveable`은 Bundle에 저장되어 구성 변경(Configuration Change) 후에도 값을 유지합니다.

```kotlin
@Composable
fun SurveyForm() {
    // remember: 화면 회전 시 "" 로 초기화됨 ❌
    var badText by remember { mutableStateOf("") }

    // rememberSaveable: 화면 회전 후에도 입력값 유지 ✅
    var name by rememberSaveable { mutableStateOf("") }
    var age  by rememberSaveable { mutableStateOf("") }

    Column(modifier = Modifier.padding(16.dp)) {
        OutlinedTextField(
            value = name,
            onValueChange = { name = it },
            label = { Text("이름") },
            modifier = Modifier.fillMaxWidth()
        )
        Spacer(modifier = Modifier.height(8.dp))
        OutlinedTextField(
            value = age,
            onValueChange = { age = it },
            label = { Text("나이") },
            keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Number),
            modifier = Modifier.fillMaxWidth()
        )
    }
}
```

> **remember vs rememberSaveable**: 일반적인 UI 상태는 `remember`로 충분합니다. 폼 입력값, 스크롤 위치처럼 화면 회전 후에도 유지해야 하는 값은 `rememberSaveable`을 사용합니다.

---

## 5. State Hoisting — 상태 끌어올리기

State Hoisting은 **상태를 컴포넌트 밖(상위)으로 이동**해 Stateless Composable을 만드는 패턴입니다.
"상태를 가진 부모"가 "상태 없는 자식"을 제어합니다.

```kotlin
// ❌ Stateful: 상태를 내부에 가짐 → 재사용/테스트 어려움
@Composable
fun StatefulCounter() {
    var count by remember { mutableStateOf(0) }
    Column {
        Text("카운트: $count")
        Button(onClick = { count++ }) { Text("증가") }
    }
}

// ✅ Stateless: 상태를 외부에서 주입받음 → 재사용/테스트 쉬움
@Composable
fun StatelessCounter(
    count: Int,              // 현재 값 (부모에서 내려옴)
    onIncrement: () -> Unit  // 이벤트 콜백 (부모에서 내려옴)
) {
    Column {
        Text("카운트: $count")
        Button(onClick = onIncrement) { Text("증가") }
    }
}

// 상태를 가진 부모 Composable
@Composable
fun CounterScreen() {
    var count by remember { mutableStateOf(0) }

    // Stateless 자식에 상태와 콜백을 전달
    StatelessCounter(
        count = count,
        onIncrement = { count++ }
    )
}
```

> **State Hoisting의 장점**:
> - 같은 Stateless Composable을 여러 곳에서 다른 상태로 재사용 가능
> - 테스트 시 원하는 상태값으로 쉽게 주입 가능
> - Composable의 책임이 "UI 렌더링"으로 명확해짐

---

## 6. 실전 — 탭 메뉴와 State Hoisting

```kotlin
// Stateless: 선택된 탭과 콜백만 받음
@Composable
fun TabRow(
    tabs: List<String>,
    selectedIndex: Int,
    onTabSelect: (Int) -> Unit
) {
    Row(modifier = Modifier.fillMaxWidth()) {
        tabs.forEachIndexed { index, title ->
            Tab(
                selected = index == selectedIndex,  // 현재 선택 탭인지 비교
                onClick = { onTabSelect(index) },
                text = { Text(title) },
                modifier = Modifier.weight(1f)  // 모든 탭이 동일한 너비
            )
        }
    }
}

// Stateless: 탭별 콘텐츠를 인덱스에 따라 렌더링
@Composable
fun TabContent(selectedIndex: Int) {
    when (selectedIndex) {
        0 -> Text("홈 화면 내용", modifier = Modifier.padding(16.dp))
        1 -> Text("검색 화면 내용", modifier = Modifier.padding(16.dp))
        2 -> Text("프로필 화면 내용", modifier = Modifier.padding(16.dp))
    }
}

// 상태를 가진 부모: 탭 선택 상태를 소유하고 자식에게 전달
@Composable
fun TabScreen() {
    val tabs = listOf("홈", "검색", "프로필")
    var selectedTab by remember { mutableStateOf(0) }  // 상태는 여기에!

    Column {
        TabRow(
            tabs = tabs,
            selectedIndex = selectedTab,
            onTabSelect = { selectedTab = it }  // 콜백: 선택된 인덱스를 상태에 반영
        )
        Divider()
        TabContent(selectedIndex = selectedTab)
    }
}
```

---

## 7. 상태 관리 계층 선택

```
로컬 UI 상태 (입력값, 열림/닫힘)
    → remember / rememberSaveable

화면 단위 상태 (목록, 선택 항목)
    → ViewModel + StateFlow → collectAsState()

앱 전역 상태 (로그인 여부, 테마)
    → ViewModel (싱글턴) + StateFlow
```

```kotlin
// ViewModel 상태를 Compose에서 구독하는 방법
@Composable
fun ProductList(viewModel: ProductViewModel = viewModel()) {
    // collectAsState(): Flow/StateFlow를 Compose State로 변환
    // → products가 바뀌면 자동 재구성
    val products by viewModel.products.collectAsState()

    LazyColumn {
        items(products) { product ->
            ProductItem(product)
        }
    }
}
```

---

## 8. 정리

| 함수 | 용도 |
|------|------|
| `remember { }` | 리컴포지션 사이에서 값 유지 |
| `mutableStateOf` | 변경 추적 가능한 상태 객체 생성 |
| `by` 위임 | `.value` 없이 상태에 직접 접근 |
| `rememberSaveable` | 화면 회전 후에도 값 유지 |
| State Hoisting | 상태를 상위로 이동 → Stateless Composable 만들기 |
| `collectAsState()` | Flow/StateFlow를 Compose State로 변환 |

- **Stateless Composable**을 만들수록 재사용성과 테스트 용이성이 올라감
- 상태는 "필요한 가장 낮은 공통 조상"에 위치시키는 것이 원칙
