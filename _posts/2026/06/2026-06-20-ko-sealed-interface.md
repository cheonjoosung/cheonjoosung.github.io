---
title: (Kotlin/코틀린) Kotlin sealed interface
tags: [ Kotlin, Android ]
style: fill
color: dark
description: Kotlin sealed interface의 개념, sealed class와의 차이, 다중 구현이 필요한 상황에서의 활용법을 Android State/Result 모델링 예제로 정리합니다.
---

## 개요

- Kotlin 1.5에서 도입된 **`sealed interface`** 를 다룹니다.
- 제한된 계층 구조를 표현하는 `sealed class`와 유사하지만, **다중 구현**이 가능한 인터페이스입니다.
- 이 글에서는 다음을 설명합니다.
  - `sealed interface`가 필요한 이유
  - `sealed class`와의 차이
  - 다중 상속이 필요한 상황에서의 활용
  - Android State/Result 모델링 실전 예제

---

## 1. 왜 필요한가

```kotlin
// sealed class — 단일 상속만 가능
sealed class NetworkResult<out T> {
    data class Success<T>(val data: T) : NetworkResult<T>()
    data class Error(val message: String) : NetworkResult<Nothing>()
    object Loading : NetworkResult<Nothing>()
}

// 만약 Success가 다른 클래스도 상속받아야 한다면?
// data class Success<T>(val data: T) : NetworkResult<T>(), Cacheable()  // ❌ 다중 상속 불가
```

```kotlin
// sealed interface — 다른 클래스/인터페이스를 함께 구현 가능
sealed interface NetworkResult<out T>

data class Success<T>(val data: T) : NetworkResult<T>, Cacheable
data class Error(val message: String) : NetworkResult<Nothing>, Loggable
object Loading : NetworkResult<Nothing>

interface Cacheable { fun cacheKey(): String = "default" }
interface Loggable { fun log(): String = "error" }
```

---

## 2. sealed class vs sealed interface

```kotlin
// sealed class — 생성자, 프로퍼티(상태)를 가질 수 있음
sealed class UiState {
    abstract val isLoading: Boolean

    data class Content(override val isLoading: Boolean, val data: List<String>) : UiState()
    data class Error(override val isLoading: Boolean, val message: String) : UiState()
}
```

```kotlin
// sealed interface — 생성자 없음, 다중 구현 가능
sealed interface UiState {
    data class Content(val data: List<String>) : UiState
    data class Error(val message: String) : UiState
    object Loading : UiState
}

// 다른 인터페이스와 함께 구현 가능
interface Parcelable
data class ContentWithParcel(val data: List<String>) : UiState, Parcelable
```

| 항목 | `sealed class` | `sealed interface` |
|------|------------------|----------------------|
| 생성자 | ✅ 가능 (공통 프로퍼티 보관) | ❌ 불가 |
| 다중 구현 | ❌ 단일 상속만 | ✅ 여러 인터페이스 동시 구현 |
| 프로퍼티 상태 | ✅ 가능 | abstract val만 선언 가능 |
| when 완전성 검사 | ✅ 동일하게 지원 | ✅ 동일하게 지원 |
| 주 사용처 | 공통 상태를 가진 계층 | 분류만 필요한 마커 역할 |

---

## 3. when 완전성 검사 — 공통점

```kotlin
sealed interface ApiResult<out T> {
    data class Success<T>(val data: T) : ApiResult<T>
    data class Error(val code: Int, val message: String) : ApiResult<Nothing>
    object Loading : ApiResult<Nothing>
}

fun handle(result: ApiResult<String>) {
    // else 없이 모든 분기 처리 — 컴파일러가 강제
    when (result) {
        is ApiResult.Success -> println(result.data)
        is ApiResult.Error   -> println("에러: ${result.message}")
        ApiResult.Loading    -> println("로딩 중")
    }
}

// 새 구현체를 추가하면 when에서 컴파일 에러 발생 → 누락 방지
```

---

## 4. 다중 분류가 필요한 경우

```kotlin
// 여러 sealed interface를 동시에 구현해 "다중 분류" 표현
sealed interface Shape
sealed interface Drawable

data class Circle(val radius: Double) : Shape, Drawable
data class Rectangle(val width: Double, val height: Double) : Shape, Drawable
object Point : Shape  // Drawable은 아님

fun describe(item: Shape) {
    when (item) {
        is Circle    -> println("원, 반지름 ${item.radius}")
        is Rectangle -> println("사각형 ${item.width}x${item.height}")
        Point        -> println("점")
    }
}

fun render(item: Drawable) {
    println("렌더링: $item")
}
```

---

## 5. Android 실전 예제

### 권한 상태 — 다른 인터페이스와 결합

```kotlin
interface Loggable {
    fun logTag(): String
}

sealed interface PermissionState {
    data class Granted(val permission: String) : PermissionState, Loggable {
        override fun logTag() = "PERMISSION_GRANTED"
    }
    data class Denied(val permission: String, val shouldShowRationale: Boolean) :
        PermissionState, Loggable {
        override fun logTag() = "PERMISSION_DENIED"
    }
    object NotRequested : PermissionState
}

fun handlePermission(state: PermissionState) {
    when (state) {
        is PermissionState.Granted -> startCamera()
        is PermissionState.Denied  ->
            if (state.shouldShowRationale) showRationale() else openSettings()
        PermissionState.NotRequested -> requestPermission()
    }
}
```

### Navigation 라우트 분류

```kotlin
sealed interface Screen {
    val route: String

    object Home : Screen { override val route = "home" }
    data class Detail(val itemId: String) : Screen {
        override val route = "detail/$itemId"
    }
    data class Profile(val userId: String) : Screen {
        override val route = "profile/$userId"
    }
}

fun navigate(screen: Screen) {
    navController.navigate(screen.route)
}
```

### MVI Intent/Effect 분리에 sealed interface 활용

```kotlin
sealed interface HomeIntent {
    object LoadData : HomeIntent
    data class Search(val query: String) : HomeIntent
    data class DeleteItem(val id: Long) : HomeIntent
}

sealed interface HomeEffect {
    data class ShowSnackbar(val message: String) : HomeEffect
    data class NavigateToDetail(val id: Long) : HomeEffect
}

fun reduce(intent: HomeIntent): HomeEffect? = when (intent) {
    HomeIntent.LoadData -> null
    is HomeIntent.Search -> null
    is HomeIntent.DeleteItem -> HomeEffect.ShowSnackbar("삭제됨")
}
```

---

## 6. 선택 기준

```kotlin
// ✅ 공통 프로퍼티/생성자가 필요하면 sealed class
sealed class FormField(open val isValid: Boolean) {
    data class Email(val value: String, override val isValid: Boolean) : FormField(isValid)
}

// ✅ 단순 분류 + 다중 구현이 필요하면 sealed interface
sealed interface ValidationResult
object Valid : ValidationResult
data class Invalid(val reason: String) : ValidationResult
```

| 상황 | 선택 |
|------|------|
| 공통 상태(프로퍼티)를 모든 하위 타입이 가져야 함 | `sealed class` |
| 하위 타입이 다른 클래스/인터페이스도 함께 구현해야 함 | `sealed interface` |
| 단순히 "이 중 하나"라는 분류만 필요 | `sealed interface` (더 가벼움) |

---

## 7. 정리

| 항목 | 내용 |
|------|------|
| `sealed interface` | 제한된 구현체 집합을 갖는 인터페이스, 다중 구현 가능 |
| `sealed class`와 차이 | 생성자 없음 / 다중 상속(구현) 가능 |
| when 완전성 | sealed class와 동일하게 컴파일러가 분기 누락을 검사 |
| Android 활용 | 권한 상태, Navigation 라우트, MVI Intent/Effect 분류 |

- `sealed interface`는 **"공통 상태 없이 분류만 필요할 때"** 의 가벼운 대안입니다.
- 다른 인터페이스와 결합해야 하는 다형성이 필요하면 `sealed class` 대신 선택합니다.

---

## 참고

- [Kotlin 공식 문서 — Sealed classes and interfaces](https://kotlinlang.org/docs/sealed-classes.html)
- [Kotlin star projection 포스팅 보기](/posts/2026-06-17-ko-star-projection)
