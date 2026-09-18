---
title: (Android/Compose) Compose Theme — MaterialTheme, colors, typography
tags: [ Android, Compose ]
style: fill
color: dark
description: Jetpack Compose의 MaterialTheme 구조와 ColorScheme, Typography, Shapes 커스터마이징, 다크 모드 대응 방법을 정리합니다.
---

---

## 1. Compose Theme이란?

`MaterialTheme`은 앱 전체에 일관된 디자인(색상, 타이포그래피, 도형)을 적용하는 컨테이너입니다.
Composable들이 하드코딩된 색상 대신 **테마 값을 참조**하게 만들어, 테마 하나만 바꾸면 앱 전체 디자인이 바뀝니다.

```
MaterialTheme
  ├── colorScheme  (primary, secondary, background, error 등)
  ├── typography   (headlineLarge, bodyMedium, labelSmall 등)
  └── shapes       (small, medium, large — 모서리 둥글기)
```

---

## 2. ColorScheme 정의

Material 3부터는 `ColorScheme`으로 색상 체계를 관리합니다. 라이트/다크 모드를 각각 정의합니다.

```kotlin
import androidx.compose.material3.darkColorScheme
import androidx.compose.material3.lightColorScheme
import androidx.compose.material3.MaterialTheme
import androidx.compose.ui.graphics.Color

// 라이트 모드 색상 체계
private val LightColors = lightColorScheme(
    primary = Color(0xFF6750A4),           // 주요 강조색 (버튼, 활성 상태)
    onPrimary = Color(0xFFFFFFFF),         // primary 위에 놓이는 콘텐츠 색 (텍스트/아이콘)
    primaryContainer = Color(0xFFEADDFF),  // primary의 은은한 배경 버전
    secondary = Color(0xFF625B71),         // 보조 강조색
    background = Color(0xFFFFFBFE),        // 화면 배경색
    onBackground = Color(0xFF1C1B1F),      // 배경 위 텍스트 색
    surface = Color(0xFFFFFBFE),           // 카드, 시트 등의 표면색
    onSurface = Color(0xFF1C1B1F),
    error = Color(0xFFB3261E)              // 에러 상태 색상
)

// 다크 모드 색상 체계 — 같은 역할, 다른 값
private val DarkColors = darkColorScheme(
    primary = Color(0xFFD0BCFF),
    onPrimary = Color(0xFF381E72),
    primaryContainer = Color(0xFF4F378B),
    secondary = Color(0xFFCCC2DC),
    background = Color(0xFF1C1B1F),
    onBackground = Color(0xFFE6E1E5),
    surface = Color(0xFF1C1B1F),
    onSurface = Color(0xFFE6E1E5),
    error = Color(0xFFF2B8B5)
)
```

> **on{Color} 네이밍 규칙**: `onPrimary`는 "primary 위에 얹는 색"을 의미합니다. 버튼 배경이 `primary`라면 그 위의 텍스트는 `onPrimary`를 사용해 항상 대비가 보장됩니다.

---

## 3. Typography 정의

텍스트 스타일을 계층별로 정의합니다. Material 3는 크기에 따라 `display / headline / title / body / label` 5단계로 구성됩니다.

```kotlin
import androidx.compose.material3.Typography
import androidx.compose.ui.text.TextStyle
import androidx.compose.ui.text.font.FontFamily
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.unit.sp

val AppTypography = Typography(
    // 화면 제목처럼 가장 큰 텍스트
    headlineLarge = TextStyle(
        fontFamily = FontFamily.Default,
        fontWeight = FontWeight.Bold,
        fontSize = 32.sp,
        lineHeight = 40.sp
    ),
    // 섹션 제목
    titleMedium = TextStyle(
        fontFamily = FontFamily.Default,
        fontWeight = FontWeight.SemiBold,
        fontSize = 18.sp,
        lineHeight = 24.sp
    ),
    // 본문 텍스트 (가장 많이 사용)
    bodyMedium = TextStyle(
        fontFamily = FontFamily.Default,
        fontWeight = FontWeight.Normal,
        fontSize = 14.sp,
        lineHeight = 20.sp
    ),
    // 캡션, 버튼 텍스트처럼 작은 텍스트
    labelSmall = TextStyle(
        fontFamily = FontFamily.Default,
        fontWeight = FontWeight.Medium,
        fontSize = 11.sp,
        lineHeight = 16.sp
    )
)
```

---

## 4. 앱 테마 조합하기

`ColorScheme`, `Typography`, `Shapes`를 하나의 테마 Composable로 묶습니다.

```kotlin
import androidx.compose.foundation.isSystemInDarkTheme
import androidx.compose.material3.Shapes
import androidx.compose.foundation.shape.RoundedCornerShape
import androidx.compose.runtime.Composable

private val AppShapes = Shapes(
    small = RoundedCornerShape(4.dp),   // 칩, 작은 버튼
    medium = RoundedCornerShape(8.dp),  // 카드
    large = RoundedCornerShape(16.dp)   // 바텀시트, 다이얼로그
)

@Composable
fun MyAppTheme(
    // isSystemInDarkTheme(): 기기의 다크 모드 설정을 자동으로 감지
    darkTheme: Boolean = isSystemInDarkTheme(),
    content: @Composable () -> Unit
) {
    // 조건에 따라 라이트/다크 색상 체계 선택
    val colorScheme = if (darkTheme) DarkColors else LightColors

    MaterialTheme(
        colorScheme = colorScheme,
        typography = AppTypography,
        shapes = AppShapes,
        content = content  // 이 테마가 적용될 하위 Composable 트리
    )
}

// 앱 최상단에서 감싸기
@Composable
fun App() {
    MyAppTheme {
        // 이 블록 안의 모든 Composable이 테마 적용받음
        MainScreen()
    }
}
```

---

## 5. 테마 값 사용하기

하드코딩 대신 `MaterialTheme` 객체를 통해 값을 참조합니다.

```kotlin
@Composable
fun ThemedCard() {
    Card(
        // ❌ 하드코딩: Color(0xFF6750A4) → 다크모드 대응 안 됨
        // ✅ 테마 참조: 다크모드에서 자동으로 올바른 색 적용
        colors = CardDefaults.cardColors(
            containerColor = MaterialTheme.colorScheme.primaryContainer
        ),
        shape = MaterialTheme.shapes.medium  // 테마의 medium 모양(8dp 둥근 모서리)
    ) {
        Column(modifier = Modifier.padding(16.dp)) {
            Text(
                text = "제목",
                style = MaterialTheme.typography.titleMedium,  // 테마 타이포그래피
                color = MaterialTheme.colorScheme.onPrimaryContainer
            )
            Text(
                text = "본문 내용입니다.",
                style = MaterialTheme.typography.bodyMedium,
                color = MaterialTheme.colorScheme.onSurfaceVariant
            )
        }
    }
}
```

> **핵심 원칙**: Composable 내부에서 `Color(0xFF...)`를 직접 쓰지 않고 항상 `MaterialTheme.colorScheme.xxx`를 참조하면, 다크 모드나 브랜드 리뉴얼 시 테마 정의만 바꾸면 앱 전체가 갱신됩니다.

---

## 6. 다이나믹 컬러 (Android 12+)

기기 배경화면에서 추출한 색상을 앱 테마에 자동 적용하는 기능입니다.

```kotlin
import android.os.Build
import androidx.compose.material3.dynamicDarkColorScheme
import androidx.compose.material3.dynamicLightColorScheme
import androidx.compose.ui.platform.LocalContext

@Composable
fun MyAppTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    dynamicColor: Boolean = true,  // 다이나믹 컬러 사용 여부
    content: @Composable () -> Unit
) {
    val colorScheme = when {
        // Android 12(API 31) 이상 + 다이나믹 컬러 옵션 켜짐
        dynamicColor && Build.VERSION.SDK_INT >= Build.VERSION_CODES.S -> {
            val context = LocalContext.current
            if (darkTheme) dynamicDarkColorScheme(context)
            else dynamicLightColorScheme(context)
        }
        // 그 외에는 직접 정의한 색상 체계 사용
        darkTheme -> DarkColors
        else -> LightColors
    }

    MaterialTheme(colorScheme = colorScheme, typography = AppTypography, content = content)
}
```

---

## 7. CompositionLocal — 커스텀 테마 값 전파

Material에 없는 커스텀 값(간격, 커스텀 색상 등)을 트리 전체에 전파하고 싶을 때 사용합니다.

```kotlin
import androidx.compose.runtime.compositionLocalOf
import androidx.compose.runtime.CompositionLocalProvider

// 앱 전용 간격(spacing) 값
data class AppSpacing(val small: Dp = 4.dp, val medium: Dp = 16.dp, val large: Dp = 32.dp)

// compositionLocalOf: 값이 없을 때 기본값 제공, 하위 트리 어디서든 조회 가능
val LocalAppSpacing = compositionLocalOf { AppSpacing() }

@Composable
fun MyAppTheme(content: @Composable () -> Unit) {
    // CompositionLocalProvider: 하위 트리에 커스텀 값 주입
    CompositionLocalProvider(LocalAppSpacing provides AppSpacing()) {
        MaterialTheme(colorScheme = LightColors, content = content)
    }
}

// 사용: props로 전달받지 않고도 어디서든 조회 가능
@Composable
fun SomeDeepChildComposable() {
    val spacing = LocalAppSpacing.current
    Box(modifier = Modifier.padding(spacing.medium)) {
        Text("커스텀 간격 적용")
    }
}
```

---

## 8. 정리

| 구성 요소 | 역할 | 접근 방법 |
|-----------|------|----------|
| `ColorScheme` | 색상 체계 (라이트/다크) | `MaterialTheme.colorScheme` |
| `Typography` | 텍스트 스타일 계층 | `MaterialTheme.typography` |
| `Shapes` | 모서리 둥글기 | `MaterialTheme.shapes` |
| `CompositionLocal` | 커스텀 테마 값 전파 | `LocalXxx.current` |

- 색상/타이포/모양을 **하드코딩하지 않고 테마를 참조**하는 것이 핵심
- `isSystemInDarkTheme()`로 다크 모드 자동 대응
- Android 12+에서는 `dynamicColorScheme`으로 배경화면 기반 색상 지원 가능
