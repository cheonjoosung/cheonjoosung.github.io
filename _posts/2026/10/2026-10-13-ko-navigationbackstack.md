---
title: (Android/Navigation) Navigation Component — 백스택 관리 전략
tags: [ Android, Navigation ]
style: fill
color: dark
description: Navigation Component의 popUpTo, popUpToInclusive, launchSingleTop, saveState를 이용한 백스택 제어와 로그인 흐름, 중첩 그래프 패턴을 정리합니다.
---

---

## 1. 백스택 기본 동작

`navigate()`를 호출할 때마다 Destination이 백스택에 쌓이고, 뒤로 가기는 하나씩 제거합니다.

```
Home → List → Detail      (스택: Home, List, Detail)
        뒤로 → List        (스택: Home, List)
```

문제는 **뒤로 가도 안 되는 화면**(로그인 완료 후 로그인 화면, 결제 완료 화면 등)이 스택에 남는 경우입니다.

---

## 2. popUpTo — 지정 지점까지 스택 제거

이동하면서 특정 Destination **위의 화면들을 제거**합니다.

```xml
<action
    android:id="@+id/action_login_to_home"
    app:destination="@id/homeFragment"
    app:popUpTo="@id/loginFragment"
    app:popUpToInclusive="true" />
```

```kotlin
// 코드로 동일하게
navController.navigate(R.id.homeFragment, null, navOptions {
    popUpTo(R.id.loginFragment) { inclusive = true }
})
```

| 옵션 | 동작 |
|------|------|
| `popUpTo = X` | X 위의 화면을 제거 (X는 남음) |
| `popUpToInclusive = true` | X **자신도 함께 제거** |

로그인 → 홈 이동에서 `inclusive = true`를 쓰면 뒤로 가기 시 로그인 화면으로 돌아가지 않습니다.

---

## 3. launchSingleTop — 중복 화면 방지

같은 화면이 스택 맨 위에 **이미 있으면 새로 쌓지 않고 재사용**합니다.

```kotlin
navController.navigate(R.id.searchFragment, null, navOptions {
    launchSingleTop = true
})
```

```
버튼을 빠르게 두 번 눌러도 Detail이 두 개 쌓이지 않음
```

> 빠른 더블 클릭으로 같은 화면이 중복 이동되는 문제의 간단한 방어책입니다.

---

## 4. saveState / restoreState — 탭 상태 보존

여러 흐름(탭)을 오가며 **각 흐름의 스택을 보존**할 때 사용합니다.

```kotlin
navController.navigate(destinationId, null, navOptions {
    popUpTo(navController.graph.startDestinationId) { saveState = true }
    launchSingleTop = true
    restoreState = true
})
```

Bottom Navigation의 `setupWithNavController`가 내부적으로 이 옵션을 사용합니다.

---

## 5. 중첩 그래프로 흐름 단위 관리

관련된 화면들을 **하나의 흐름(로그인, 주문 등)** 으로 묶으면 한 번에 제거하기 쉽습니다.

```xml
<navigation android:id="@+id/nav_graph" app:startDestination="@id/homeFragment">

    <fragment android:id="@+id/homeFragment" ... />

    <!-- 로그인 흐름: 3개 화면을 하나의 그래프로 -->
    <navigation
        android:id="@+id/auth_graph"
        app:startDestination="@id/loginFragment">
        <fragment android:id="@+id/loginFragment" ... />
        <fragment android:id="@+id/signUpFragment" ... />
        <fragment android:id="@+id/findPasswordFragment" ... />
    </navigation>
</navigation>
```

```kotlin
// 로그인 완료: auth_graph 전체를 스택에서 제거
navController.navigate(R.id.homeFragment, null, navOptions {
    popUpTo(R.id.auth_graph) { inclusive = true }
})
```

---

## 6. 흔한 시나리오별 처리

| 상황 | 처리 |
|------|------|
| 로그인 완료 후 홈 | `popUpTo(login/auth_graph) { inclusive = true }` |
| 스플래시 → 메인 | `popUpTo(splash) { inclusive = true }` |
| 결제 완료 → 주문 내역 | 결제 흐름 그래프 전체 popUpTo |
| 로그아웃 | 전체 스택 제거 후 로그인으로 (`popUpTo(graph.id) { inclusive = true }`) |
| 같은 화면 중복 방지 | `launchSingleTop = true` |

---

## 7. 현재 스택 확인 (디버깅)

```kotlin
navController.addOnDestinationChangedListener { controller, destination, _ ->
    val stack = controller.currentBackStack.value
        .mapNotNull { it.destination.label }
    Log.d("Nav", "현재 스택: $stack")
}
```

---

## 8. 정리

- `popUpTo`로 뒤로 가면 안 되는 화면을 스택에서 제거, `inclusive`로 기준 화면 포함 여부 결정
- `launchSingleTop`으로 동일 화면 중복 쌓임 방지
- `saveState`/`restoreState`로 탭별 스택 보존
- 로그인, 결제처럼 **흐름 단위는 중첩 그래프**로 묶어 한 번에 제거
- 백스택 정책은 화면을 추가할 때마다 "여기서 뒤로 가면 어디로?"를 먼저 정의
