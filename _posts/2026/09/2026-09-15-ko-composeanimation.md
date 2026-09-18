---
title: (Android/Compose) Compose Animation — AnimatedVisibility, animateDpAsState
tags: [ Android, Compose ]
style: fill
color: dark
description: Jetpack Compose의 애니메이션 API인 AnimatedVisibility, animate*AsState, updateTransition, animatedContent를 예제 중심으로 정리합니다.
---

---

## 1. Compose Animation 개요

Compose는 선언적 UI답게 **상태 변화에 따른 애니메이션**을 간결하게 표현합니다.
"값 A에서 값 B로 부드럽게 변경"이라는 개념을 상태 변수 하나로 표현합니다.

```
상태 변경 (false → true)
    ↓
Compose가 애니메이션으로 UI 전환
    ↓
개발자는 최종 상태(true일 때 UI)만 선언
```

---

## 2. animate*AsState — 단순 값 애니메이션

단일 값(크기, 색상, 투명도 등)을 부드럽게 변경합니다. 가장 간단한 애니메이션 API입니다.

```kotlin
import androidx.compose.animation.core.animateDpAsState
import androidx.compose.animation.core.animateFloatAsState
import androidx.compose.animation.core.animateColorAsState
import androidx.compose.animation.core.tween
import androidx.compose.animation.core.spring

@Composable
fun AnimatedCard() {
    var expanded by remember { mutableStateOf(false) }

    // animateDpAsState: Dp 값을 애니메이션으로 변경
    // expanded 상태에 따라 48.dp ↔ 200.dp 사이를 부드럽게 이동
    val cardHeight by animateDpAsState(
        targetValue = if (expanded) 200.dp else 48.dp,
        animationSpec = tween(durationMillis = 300),  // 300ms 동안 선형 보간
        label = "cardHeight"  // 디버깅용 레이블
    )

    // animateFloatAsState: Float 값 (투명도, 회전각 등)
    val rotation by animateFloatAsState(
        targetValue = if (expanded) 180f else 0f,
        animationSpec = spring(dampingRatio = 0.6f),  // 스프링 물리 효과
        label = "rotation"
    )

    // animateColorAsState: Color 값
    val backgroundColor by animateColorAsState(
        targetValue = if (expanded) MaterialTheme.colorScheme.primaryContainer
                      else MaterialTheme.colorScheme.surface,
        label = "bgColor"
    )

    Card(
        modifier = Modifier
            .fillMaxWidth()
            .height(cardHeight)  // 애니메이션되는 높이
            .clickable { expanded = !expanded },
        colors = CardDefaults.cardColors(containerColor = backgroundColor)
    ) {
        Row(
            modifier = Modifier.padding(16.dp),
            verticalAlignment = Alignment.CenterVertically
        ) {
            Text("카드 제목", modifier = Modifier.weight(1f))
            // 화살표 아이콘이 180도 회전
            Icon(
                imageVector = Icons.Default.ExpandMore,
                contentDescription = null,
                modifier = Modifier.rotate(rotation)  // 애니메이션되는 회전
            )
        }
    }
}
```

> **animationSpec 종류**:
> - `tween(durationMillis)`: 지정 시간 동안 선형 보간 (가장 일반적)
> - `spring(dampingRatio, stiffness)`: 스프링 물리 효과 (자연스러운 느낌)
> - `snap()`: 즉시 변경 (애니메이션 없음)
> - `keyframes { }`: 중간 지점을 직접 지정

---

## 3. AnimatedVisibility — 나타남/사라짐 애니메이션

컴포넌트가 화면에 나타나거나 사라질 때의 애니메이션입니다.

```kotlin
import androidx.compose.animation.AnimatedVisibility
import androidx.compose.animation.fadeIn
import androidx.compose.animation.fadeOut
import androidx.compose.animation.slideInVertically
import androidx.compose.animation.slideOutVertically
import androidx.compose.animation.expandVertically
import androidx.compose.animation.shrinkVertically

@Composable
fun NotificationBanner(message: String?) {
    // message가 null이 아닐 때 배너 표시
    AnimatedVisibility(
        visible = message != null,
        enter = slideInVertically { -it } + fadeIn(),  // 위에서 슬라이드 + 페이드인
        exit  = slideOutVertically { -it } + fadeOut()  // 위로 슬라이드 + 페이드아웃
    ) {
        // 이 블록은 visible=true일 때만 컴포지션에 존재
        Surface(
            color = MaterialTheme.colorScheme.inverseSurface,
            modifier = Modifier.fillMaxWidth()
        ) {
            Text(
                text = message ?: "",
                modifier = Modifier.padding(16.dp),
                color = MaterialTheme.colorScheme.inverseOnSurface
            )
        }
    }
}

// enter/exit 애니메이션 조합 예시
@Composable
fun ExpandableContent(visible: Boolean, content: @Composable () -> Unit) {
    AnimatedVisibility(
        visible = visible,
        // expandVertically: 높이가 0에서 실제 크기로 확장
        enter = expandVertically(expandFrom = Alignment.Top),
        // shrinkVertically: 높이가 실제 크기에서 0으로 축소
        exit  = shrinkVertically(shrinkTowards = Alignment.Top)
    ) {
        content()
    }
}
```

---

## 4. Crossfade — 컨텐츠 전환 애니메이션

다른 상태에 해당하는 컴포넌트를 전환할 때 페이드 효과를 줍니다.

```kotlin
import androidx.compose.animation.Crossfade

@Composable
fun TabContent(selectedTab: String) {
    // Crossfade: targetState가 바뀌면 이전 컴포넌트는 페이드아웃, 새 컴포넌트는 페이드인
    Crossfade(
        targetState = selectedTab,
        animationSpec = tween(durationMillis = 200),
        label = "tabContent"
    ) { tab ->
        // tab은 현재 표시 중인 상태 (전환 중 이전/새 값 모두 렌더링됨)
        when (tab) {
            "home"    -> HomeContent()
            "search"  -> SearchContent()
            "profile" -> ProfileContent()
        }
    }
}
```

---

## 5. AnimatedContent — 복잡한 컨텐츠 전환

컨텐츠 변경 시 진입/퇴장 애니메이션을 세밀하게 제어합니다.

```kotlin
import androidx.compose.animation.AnimatedContent
import androidx.compose.animation.togetherWith

@Composable
fun CounterWithAnimation() {
    var count by remember { mutableStateOf(0) }

    Row(verticalAlignment = Alignment.CenterVertically) {
        Button(onClick = { count-- }) { Text("-") }

        AnimatedContent(
            targetState = count,
            // transitionSpec: 진입과 퇴장 애니메이션을 함께 정의
            transitionSpec = {
                // 숫자가 증가할 때: 위에서 들어오고 아래로 나감
                // 숫자가 감소할 때: 아래서 들어오고 위로 나감
                if (targetState > initialState) {
                    slideInVertically { -it } togetherWith slideOutVertically { it }
                } else {
                    slideInVertically { it } togetherWith slideOutVertically { -it }
                }
            },
            label = "counter"
        ) { targetCount ->
            Text(
                text = "$targetCount",
                style = MaterialTheme.typography.headlineLarge,
                modifier = Modifier.padding(horizontal = 24.dp)
            )
        }

        Button(onClick = { count++ }) { Text("+") }
    }
}
```

---

## 6. updateTransition — 여러 값의 동기화된 전환

하나의 상태 변화에 여러 속성이 동시에 애니메이션되어야 할 때 사용합니다.

```kotlin
import androidx.compose.animation.core.updateTransition
import androidx.compose.animation.core.animateDp
import androidx.compose.animation.core.animateColor

enum class ButtonState { Idle, Loading, Success }

@Composable
fun AnimatedButton() {
    var buttonState by remember { mutableStateOf(ButtonState.Idle) }

    // updateTransition: buttonState 변화를 추적하는 Transition 생성
    val transition = updateTransition(targetState = buttonState, label = "button")

    // 각 속성이 같은 Transition에서 파생 → 동기화 보장
    val width by transition.animateDp(label = "width") { state ->
        when (state) {
            ButtonState.Idle    -> 200.dp  // 기본 너비
            ButtonState.Loading -> 56.dp   // 원형 인디케이터 크기
            ButtonState.Success -> 200.dp  // 기본 너비로 복귀
        }
    }

    val backgroundColor by transition.animateColor(label = "bgColor") { state ->
        when (state) {
            ButtonState.Idle    -> MaterialTheme.colorScheme.primary
            ButtonState.Loading -> MaterialTheme.colorScheme.surfaceVariant
            ButtonState.Success -> Color(0xFF4CAF50)  // 초록색
        }
    }

    Button(
        onClick = {
            // 상태 전환: Idle → Loading → Success
            if (buttonState == ButtonState.Idle) buttonState = ButtonState.Loading
        },
        modifier = Modifier.width(width),  // 애니메이션되는 너비
        colors = ButtonDefaults.buttonColors(containerColor = backgroundColor)
    ) {
        when (buttonState) {
            ButtonState.Idle    -> Text("제출")
            ButtonState.Loading -> CircularProgressIndicator(
                modifier = Modifier.size(24.dp),
                color = Color.White,
                strokeWidth = 2.dp
            )
            ButtonState.Success -> Icon(Icons.Default.Check, contentDescription = null)
        }
    }
}
```

---

## 7. 정리

| API | 용도 | 특징 |
|-----|------|------|
| `animate*AsState` | 단일 값 애니메이션 | 가장 간단, 개별 값에 적용 |
| `AnimatedVisibility` | 나타남/사라짐 | `enter` / `exit` 설정 |
| `Crossfade` | 컨텐츠 페이드 전환 | 단순 상태 전환에 적합 |
| `AnimatedContent` | 복잡한 컨텐츠 전환 | 진입/퇴장 방향 세밀 제어 |
| `updateTransition` | 여러 값 동기화 애니메이션 | 여러 속성이 함께 전환될 때 |

- `animationSpec`: `tween` (시간 기반), `spring` (물리 기반), `snap` (즉시 변경)
- `label` 파라미터: 필수는 아니지만 Android Studio Animation Preview에서 식별에 사용
- 성능 팁: 애니메이션 값은 `Modifier`에서 직접 사용 (`graphicsLayer`로 GPU 가속 활용)
