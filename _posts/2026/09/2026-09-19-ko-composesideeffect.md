---
title: (Android/Compose) Compose Side Effect — LaunchedEffect, SideEffect, DisposableEffect
tags: [ Android, Compose ]
style: fill
color: dark
description: Jetpack Compose의 Side Effect API인 LaunchedEffect, SideEffect, DisposableEffect, rememberCoroutineScope, produceState 사용법과 차이를 정리합니다.
---

---

## 1. Side Effect란?

**Side Effect(부수 효과)** 는 Composable 함수 밖의 상태를 변경하는 작업입니다.
Composable 함수는 원래 "순수 함수"처럼 동작해야 하지만(같은 입력 → 같은 UI), 실제로는 네트워크 호출, 로그 기록, 리스너 등록 같은 부수 효과가 필요합니다.
Compose는 이런 부수 효과를 **재구성(Recomposition)과 안전하게 동기화**하기 위한 전용 API를 제공합니다.

```
왜 그냥 Composable 본문에 쓰면 안 되나?
  → Composable은 재구성될 때마다 다시 실행됨
  → API 호출 코드를 본문에 직접 쓰면 재구성마다 중복 호출됨 ❌

Side Effect API를 쓰면?
  → "언제 실행할지" 키(key)를 지정해 필요할 때만 실행 ✅
```

---

## 2. LaunchedEffect — 코루틴 기반 부수 효과

Composable 안에서 코루틴을 실행해야 할 때 사용합니다. 가장 자주 쓰이는 Side Effect API입니다.

```kotlin
import androidx.compose.runtime.LaunchedEffect

@Composable
fun UserProfileScreen(userId: String, viewModel: ProfileViewModel = viewModel()) {
    // LaunchedEffect(key1 = userId): userId가 바뀔 때마다 새로 실행
    // 최초 컴포지션 시 1회 실행 + userId 값이 변경될 때마다 재실행
    LaunchedEffect(userId) {
        viewModel.loadUser(userId)  // suspend 함수 호출 가능
    }

    // LaunchedEffect(Unit): 딱 한 번만 실행 (컴포지션에 처음 들어올 때)
    LaunchedEffect(Unit) {
        viewModel.logScreenView("ProfileScreen")
    }

    // 화면 UI...
}
```

> **key의 역할**: `LaunchedEffect(key)`는 key 값이 바뀔 때만 이전 코루틴을 취소하고 새로 실행합니다. `Unit`처럼 절대 바뀌지 않는 값을 넣으면 "최초 1회만 실행"이 됩니다.

```kotlin
// key가 여러 개인 경우: 둘 중 하나라도 바뀌면 재실행
@Composable
fun SearchResults(query: String, filter: String) {
    var results by remember { mutableStateOf<List<Item>>(emptyList()) }

    LaunchedEffect(query, filter) {  // query 또는 filter가 바뀔 때마다 재검색
        results = searchApi(query, filter)
    }

    LazyColumn { items(results) { ItemRow(it) } }
}
```

---

## 3. rememberCoroutineScope — 이벤트 콜백에서 코루틴 실행

`LaunchedEffect`는 Composable 본문에서만 사용할 수 있습니다. 버튼 클릭 같은 **콜백 안에서** 코루틴을 실행하려면 `rememberCoroutineScope`가 필요합니다.

```kotlin
import androidx.compose.runtime.rememberCoroutineScope
import kotlinx.coroutines.launch

@Composable
fun ScrollToTopButton(listState: LazyListState) {
    // rememberCoroutineScope: Composable의 생명주기에 묶인 CoroutineScope 생성
    val scope = rememberCoroutineScope()

    Button(onClick = {
        // onClick은 suspend 함수가 아니므로 직접 suspend 함수를 호출할 수 없음
        // → scope.launch로 코루틴을 새로 시작해야 함
        scope.launch {
            listState.animateScrollToItem(0)
        }
    }) {
        Text("맨 위로")
    }
}
```

> **LaunchedEffect vs rememberCoroutineScope**: `LaunchedEffect`는 "재구성 시 자동으로 실행"되는 코루틴이고, `rememberCoroutineScope`는 "사용자 이벤트에 반응해 수동으로 실행"하는 코루틴입니다.

---

## 4. DisposableEffect — 정리(cleanup)가 필요한 부수 효과

리스너 등록/해제처럼 **Composable이 화면에서 사라질 때 정리 작업**이 필요할 때 사용합니다.

```kotlin
import androidx.compose.runtime.DisposableEffect

@Composable
fun LifecycleAwareComponent(lifecycleOwner: LifecycleOwner = LocalLifecycleOwner.current) {
    var isResumed by remember { mutableStateOf(false) }

    // DisposableEffect(key): key가 바뀌거나 Composable이 제거될 때 onDispose 실행
    DisposableEffect(lifecycleOwner) {
        val observer = LifecycleEventObserver { _, event ->
            when (event) {
                Lifecycle.Event.ON_RESUME -> isResumed = true
                Lifecycle.Event.ON_PAUSE  -> isResumed = false
                else -> {}
            }
        }
        // 등록: Composable이 컴포지션에 들어올 때 리스너 등록
        lifecycleOwner.lifecycle.addObserver(observer)

        // onDispose: 반드시 반환해야 함 — Composable이 사라지거나 key가 바뀔 때 실행
        onDispose {
            lifecycleOwner.lifecycle.removeObserver(observer)  // 리스너 해제 (메모리 누수 방지)
        }
    }

    Text(if (isResumed) "화면 활성 상태" else "화면 비활성 상태")
}
```

> **onDispose는 필수**: `DisposableEffect` 블록은 반드시 마지막에 `onDispose { }`를 반환해야 컴파일됩니다. 등록한 리스너/콜백을 여기서 해제하지 않으면 메모리 누수가 발생합니다.

---

## 5. SideEffect — 매 성공적인 재구성마다 실행

Compose가 아닌 시스템(예: 분석 SDK, View 시스템)에 **현재 상태를 그대로 전달**하고 싶을 때 사용합니다.

```kotlin
import androidx.compose.runtime.SideEffect

@Composable
fun AnalyticsScreen(userId: String, analyticsService: AnalyticsService) {
    // SideEffect: 재구성이 "성공적으로 완료될 때마다" 실행 (조건 key 없음)
    // Compose 외부 시스템(analyticsService)에 최신 상태를 동기화하는 용도
    SideEffect {
        analyticsService.setUserProperty("user_id", userId)
    }

    // 화면 UI...
}
```

> **LaunchedEffect vs SideEffect**: `LaunchedEffect`는 코루틴을 실행하고 key가 바뀔 때만 재실행됩니다. `SideEffect`는 코루틴이 아니며 **재구성될 때마다 매번** 실행됩니다. 순수하게 "현재 상태를 외부에 반영"하는 짧은 동기 작업에만 사용합니다.

---

## 6. produceState — 외부 데이터를 State로 변환

콜백 기반 API나 Flow가 아닌 데이터 소스를 Compose State로 변환할 때 사용합니다.

```kotlin
import androidx.compose.runtime.produceState

@Composable
fun rememberNetworkStatus(): State<Boolean> {
    val context = LocalContext.current

    // produceState: 초기값을 주고, 블록 안에서 값을 갱신하는 State 생성
    return produceState(initialValue = false) {
        val connectivityManager = context.getSystemService(ConnectivityManager::class.java)
        val callback = object : ConnectivityManager.NetworkCallback() {
            override fun onAvailable(network: Network) { value = true }   // State 값 갱신
            override fun onLost(network: Network) { value = false }
        }
        connectivityManager.registerDefaultNetworkCallback(callback)

        // awaitDispose: DisposableEffect의 onDispose와 동일한 역할
        awaitDispose {
            connectivityManager.unregisterNetworkCallback(callback)
        }
    }
}

// 사용
@Composable
fun NetworkStatusBanner() {
    val isConnected by rememberNetworkStatus()
    if (!isConnected) {
        Text("네트워크 연결 없음", color = Color.Red)
    }
}
```

---

## 7. rememberUpdatedState — 최신 람다 참조 유지

`LaunchedEffect`가 실행 중일 때, 외부에서 넘어온 람다가 최신 값을 참조하도록 보장합니다.

```kotlin
@Composable
fun Timeout(onTimeout: () -> Unit) {
    // rememberUpdatedState: onTimeout이 재구성으로 바뀌어도 항상 최신 참조 유지
    val currentOnTimeout by rememberUpdatedState(onTimeout)

    // LaunchedEffect(true): 절대 재실행되지 않음 (컴포지션 동안 딱 한 번의 대기)
    LaunchedEffect(true) {
        delay(5000)
        currentOnTimeout()  // 5초 뒤 실행 시점의 "최신" onTimeout 사용
    }
}
```

> **문제 상황**: `LaunchedEffect(true)`는 key가 고정이라 재구성돼도 재시작되지 않습니다. 이때 `onTimeout` 람다를 직접 캡처하면 **최초 컴포지션 시점의 오래된 람다**를 계속 참조하게 됩니다. `rememberUpdatedState`로 감싸면 항상 최신 람다를 사용합니다.

---

## 8. Side Effect API 선택 가이드

```
코루틴을 실행해야 하는가?
  ├── 재구성 시 자동 실행 필요 → LaunchedEffect
  └── 이벤트 콜백에서 수동 실행 → rememberCoroutineScope + launch

정리(cleanup)가 필요한 리스너/콜백 등록인가?
  → DisposableEffect

Compose 외부 시스템에 현재 상태를 매번 동기화해야 하는가?
  → SideEffect

콜백 기반 API를 State로 변환해야 하는가?
  → produceState
```

---

## 9. 정리

| API | 실행 시점 | 용도 |
|-----|----------|------|
| `LaunchedEffect(key)` | key 변경 시 (코루틴) | 네트워크 호출, 지연 작업 |
| `rememberCoroutineScope` | 이벤트 콜백에서 수동 | 클릭 시 애니메이션/스크롤 |
| `DisposableEffect(key)` | key 변경 시 + cleanup 필수 | 리스너 등록/해제 |
| `SideEffect` | 매 성공적 재구성마다 | 외부 시스템에 상태 동기화 |
| `produceState` | 초기값 + 비동기 갱신 | 콜백 API → State 변환 |
| `rememberUpdatedState` | - | 실행 중인 Effect에 최신 람다 전달 |

- 모든 Side Effect API는 **Composable의 생명주기**에 안전하게 연결되어 메모리 누수를 방지
- `DisposableEffect`는 `onDispose` 누락 시 컴파일 에러 — 반드시 정리 코드 작성
