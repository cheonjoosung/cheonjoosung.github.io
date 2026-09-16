---
title: (Android/Compose) Compose Navigation — NavGraph, NavController
tags: [ Android, Compose ]
style: fill
color: dark
description: Jetpack Compose Navigation의 NavController, NavHost, NavGraph 구성 방법과 화면 간 데이터 전달, 딥링크 처리를 정리합니다.
---

---

## 1. Compose Navigation이란?

Compose Navigation은 Composable 함수를 "화면"으로 삼아 앱 내 이동을 관리하는 라이브러리입니다.
기존 Fragment 기반 Navigation과 달리, **Composable 함수가 곧 화면**이 됩니다.

```
NavHost (네비게이션 컨테이너)
  ├── composable("home") { HomeScreen() }
  ├── composable("detail/{id}") { DetailScreen() }
  └── composable("profile") { ProfileScreen() }
       ↑
NavController.navigate("detail/42") 호출 시 해당 화면으로 이동
```

**의존성 추가 (build.gradle.kts)**:
```kotlin
implementation("androidx.navigation:navigation-compose:2.7.7")
```

---

## 2. 기본 구성

NavController와 NavHost만 있으면 네비게이션이 동작합니다.

```kotlin
import androidx.navigation.NavController
import androidx.navigation.compose.NavHost
import androidx.navigation.compose.composable
import androidx.navigation.compose.rememberNavController

@Composable
fun AppNavigation() {
    // rememberNavController: NavController를 생성하고 Composition에 저장
    // NavController는 Activity 생명주기에 맞게 상태 보존
    val navController = rememberNavController()

    // NavHost: 현재 경로에 맞는 Composable을 보여주는 컨테이너
    NavHost(
        navController = navController,
        startDestination = "home"  // 앱 시작 시 보여줄 화면의 경로
    ) {
        // composable(): 경로(route)와 화면 Composable을 연결
        composable("home") {
            HomeScreen(
                // navController를 직접 넘기는 대신 람다로 이동 함수를 넘김 (권장)
                onNavigateToDetail = { id -> navController.navigate("detail/$id") },
                onNavigateToProfile = { navController.navigate("profile") }
            )
        }

        composable("detail/{id}") { backStackEntry ->
            // backStackEntry.arguments: 경로에서 추출한 파라미터
            val id = backStackEntry.arguments?.getString("id") ?: ""
            DetailScreen(
                id = id,
                onBack = { navController.popBackStack() }  // 이전 화면으로 이동
            )
        }

        composable("profile") {
            ProfileScreen(onBack = { navController.popBackStack() })
        }
    }
}
```

> **왜 람다로 넘기나?** `navController`를 Composable에 직접 전달하면 해당 Composable이 Navigation에 의존하게 됩니다. 람다로 넘기면 Composable은 "화면 이동"이라는 행동만 알고, 구체적인 방법은 모릅니다 → 테스트와 재사용이 쉬워집니다.

---

## 3. 화면 간 데이터 전달

경로(route)에 파라미터를 포함해 데이터를 전달합니다.

```kotlin
NavHost(navController, startDestination = "home") {

    composable("home") {
        HomeScreen(
            onProductClick = { productId ->
                // 경로에 ID를 포함해 이동
                navController.navigate("product/$productId")
            }
        )
    }

    // {productId}: 경로 파라미터 선언
    composable(
        route = "product/{productId}",
        arguments = listOf(
            // navArgument: 파라미터 타입과 기본값 지정
            navArgument("productId") {
                type = NavType.IntType  // String, Int, Boolean, Float 지원
                defaultValue = 0        // 파라미터 없을 때 기본값
            }
        )
    ) { backStackEntry ->
        val productId = backStackEntry.arguments?.getInt("productId") ?: 0
        ProductDetailScreen(productId = productId)
    }

    // 선택적 쿼리 파라미터: "search?query=kotlin&page=1" 형태
    composable(
        route = "search?query={query}&page={page}",
        arguments = listOf(
            navArgument("query") { defaultValue = "" },
            navArgument("page") { type = NavType.IntType; defaultValue = 1 }
        )
    ) { backStackEntry ->
        val query = backStackEntry.arguments?.getString("query") ?: ""
        val page  = backStackEntry.arguments?.getInt("page") ?: 1
        SearchScreen(query = query, page = page)
    }
}
```

---

## 4. 결과 반환 (화면 B → 화면 A)

이전 화면으로 결과를 돌려줄 때 `SavedStateHandle`을 사용합니다.

```kotlin
// 화면 B: 선택한 값을 이전 화면으로 돌려줌
@Composable
fun ColorPickerScreen(navController: NavController) {
    val colors = listOf("빨강", "초록", "파랑")
    Column {
        colors.forEach { color ->
            Button(onClick = {
                // previousBackStackEntry: 이전 화면의 BackStackEntry
                navController.previousBackStackEntry
                    ?.savedStateHandle                // SavedStateHandle에 결과 저장
                    ?.set("selectedColor", color)
                navController.popBackStack()          // 이전 화면으로 이동
            }) {
                Text(color)
            }
        }
    }
}

// 화면 A: 화면 B가 돌려준 결과를 관찰
@Composable
fun PaletteScreen(navController: NavController) {
    // currentBackStackEntry의 SavedStateHandle에서 결과 관찰
    val selectedColor by navController.currentBackStackEntry
        ?.savedStateHandle
        ?.getStateFlow("selectedColor", "미선택")
        ?.collectAsState()
        ?: remember { mutableStateOf("미선택") }

    Column {
        Text("선택된 색상: $selectedColor")
        Button(onClick = { navController.navigate("colorPicker") }) {
            Text("색상 선택")
        }
    }
}
```

---

## 5. 중첩 NavGraph — 논리적 화면 묶음

연관된 화면들을 그룹으로 묶어 관리합니다.

```kotlin
NavHost(navController, startDestination = "auth") {

    // 인증 관련 화면을 하나의 그래프로 묶기
    navigation(
        startDestination = "auth/login",  // 그룹의 시작 화면
        route = "auth"                    // 그룹의 경로
    ) {
        composable("auth/login") {
            LoginScreen(
                onLoginSuccess = {
                    // 로그인 성공 시 main 그래프로 이동하고 auth 그래프 스택 제거
                    navController.navigate("main") {
                        popUpTo("auth") { inclusive = true }
                    }
                },
                onSignUpClick = { navController.navigate("auth/signup") }
            )
        }
        composable("auth/signup") {
            SignUpScreen(onBack = { navController.popBackStack() })
        }
    }

    // 메인 앱 화면
    navigation(startDestination = "main/home", route = "main") {
        composable("main/home") { HomeScreen() }
        composable("main/search") { SearchScreen() }
        composable("main/profile") { ProfileScreen() }
    }
}
```

---

## 6. 타입 안전한 Navigation (Navigation 2.8+)

문자열 경로 대신 Kotlin 객체로 안전하게 이동합니다.

```kotlin
import kotlinx.serialization.Serializable

// @Serializable: 경로 직렬화에 사용
@Serializable
object HomeRoute  // 파라미터 없는 화면

@Serializable
data class ProductRoute(val productId: Int)  // 파라미터 있는 화면

NavHost(navController, startDestination = HomeRoute) {
    composable<HomeRoute> {
        HomeScreen(
            onProductClick = { id ->
                navController.navigate(ProductRoute(productId = id))
            }
        )
    }

    composable<ProductRoute> { backStackEntry ->
        // toRoute(): BackStackEntry에서 타입 안전하게 파라미터 추출
        val route = backStackEntry.toRoute<ProductRoute>()
        ProductDetailScreen(productId = route.productId)
    }
}
```

> **타입 안전 Navigation의 장점**: 경로를 문자열로 작성할 때 오타가 있어도 컴파일 시 알 수 없지만, 객체로 작성하면 컴파일 시점에 타입 오류를 잡을 수 있습니다.

---

## 7. 정리

| 개념 | 역할 |
|------|------|
| `NavController` | 화면 이동 제어 (`navigate`, `popBackStack`) |
| `NavHost` | 현재 경로에 맞는 Composable 표시 |
| `composable(route)` | 경로와 화면 Composable 연결 |
| `navArgument` | 경로 파라미터 타입/기본값 설정 |
| `SavedStateHandle` | 화면 B에서 화면 A로 결과 반환 |
| `navigation(route)` | 관련 화면을 논리적으로 그룹화 |

- `navController`를 Composable에 직접 전달하지 말고 **람다**로 이동 행동을 전달
- `popUpTo`: 특정 화면까지의 스택을 제거해 뒤로 가기 동작을 원하는 대로 제어
- Navigation 2.8+부터 타입 안전 API 사용 권장
