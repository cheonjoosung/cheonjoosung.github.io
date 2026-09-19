---
title: (Android/Compose) Compose Custom Layout — Layout, SubcomposeLayout
tags: [ Android, Compose ]
style: fill
color: dark
description: Jetpack Compose에서 Layout, SubcomposeLayout으로 커스텀 레이아웃을 만드는 방법과 measure/placement 원리를 정리합니다.
---

---

## 1. Custom Layout이 필요한 이유

`Row`, `Column`, `Box`로 표현할 수 없는 특수한 배치가 필요할 때 `Layout` Composable을 직접 사용합니다.
예: 자식들을 지그재그로 배치, 원형으로 배치, 남은 공간에 맞춰 동적 배치 등.

```
일반 레이아웃:              Custom Layout:
Row { A, B, C }            Layout { measure → 직접 계산 → place }
  → 가로로 순서대로 배치       → 원하는 알고리즘으로 자유롭게 배치
```

Compose의 레이아웃 과정은 **Measure(측정) → Place(배치)** 2단계로 이루어집니다.

---

## 2. Layout Composable 기본 구조

`Layout`은 자식들의 크기를 측정하고 위치를 직접 지정할 수 있는 저수준 API입니다.

```kotlin
import androidx.compose.ui.layout.Layout
import androidx.compose.ui.layout.MeasurePolicy

@Composable
fun SimpleColumn(
    modifier: Modifier = Modifier,
    content: @Composable () -> Unit
) {
    Layout(
        content = content,   // 배치할 자식 Composable들
        modifier = modifier
    ) { measurables, constraints ->
        // measurables: 아직 측정되지 않은 자식들의 목록
        // constraints: 이 레이아웃이 가질 수 있는 최대/최소 크기 제약

        // 1단계 Measure: 각 자식을 측정해 Placeable(배치 가능한 결과) 획득
        val placeables = measurables.map { measurable ->
            measurable.measure(constraints)  // 제약 조건 내에서 스스로 크기 결정하게 함
        }

        // 전체 레이아웃 크기 계산: 너비는 가장 넓은 자식, 높이는 모든 자식의 합
        val width = placeables.maxOf { it.width }
        val height = placeables.sumOf { it.height }

        // 2단계 Layout: 계산된 크기로 레이아웃 확정 후 자식 배치
        layout(width, height) {
            var yPosition = 0
            // placeRelative: 각 자식을 실제 좌표에 배치 (RTL 언어 자동 대응)
            placeables.forEach { placeable ->
                placeable.placeRelative(x = 0, y = yPosition)
                yPosition += placeable.height  // 다음 자식은 이전 자식 아래에 배치
            }
        }
    }
}

// 사용: Column과 동일하게 동작하는 커스텀 레이아웃
SimpleColumn {
    Text("첫 번째 줄")
    Text("두 번째 줄")
    Text("세 번째 줄")
}
```

> **measure는 한 번만 호출 가능**: 성능을 위해 각 `measurable.measure()`는 한 번만 호출할 수 있습니다. 같은 자식을 여러 크기로 측정해야 한다면 `SubcomposeLayout`을 사용해야 합니다.

---

## 3. 실전 예제 — Flow Layout (자동 줄바꿈 배치)

태그, 칩처럼 가로 공간이 부족하면 자동으로 다음 줄로 넘어가는 레이아웃입니다.

```kotlin
@Composable
fun FlowRow(
    modifier: Modifier = Modifier,
    horizontalSpacing: Dp = 8.dp,
    verticalSpacing: Dp = 8.dp,
    content: @Composable () -> Unit
) {
    Layout(content = content, modifier = modifier) { measurables, constraints ->
        val hSpacingPx = horizontalSpacing.roundToPx()
        val vSpacingPx = verticalSpacing.roundToPx()

        // 자식들을 최대 너비 제약으로 측정 (자식 스스로의 크기 결정)
        val placeables = measurables.map { it.measure(constraints) }

        // 줄바꿈 로직: 현재 줄의 누적 너비가 최대 너비를 넘으면 새 줄 시작
        val rows = mutableListOf<MutableList<androidx.compose.ui.layout.Placeable>>()
        var currentRow = mutableListOf<androidx.compose.ui.layout.Placeable>()
        var currentRowWidth = 0

        placeables.forEach { placeable ->
            // 현재 줄에 이 아이템을 추가하면 넘치는지 확인
            if (currentRowWidth + placeable.width > constraints.maxWidth && currentRow.isNotEmpty()) {
                rows.add(currentRow)              // 현재 줄 확정
                currentRow = mutableListOf()      // 새 줄 시작
                currentRowWidth = 0
            }
            currentRow.add(placeable)
            currentRowWidth += placeable.width + hSpacingPx
        }
        if (currentRow.isNotEmpty()) rows.add(currentRow)  // 마지막 줄 추가

        // 전체 높이 = 각 줄의 최대 높이 합 + 줄 간격
        val totalHeight = rows.sumOf { row -> row.maxOf { it.height } } +
            (rows.size - 1).coerceAtLeast(0) * vSpacingPx

        layout(constraints.maxWidth, totalHeight) {
            var yPosition = 0
            rows.forEach { row ->
                var xPosition = 0
                val rowHeight = row.maxOf { it.height }
                row.forEach { placeable ->
                    placeable.placeRelative(x = xPosition, y = yPosition)
                    xPosition += placeable.width + hSpacingPx  // 다음 아이템은 오른쪽에
                }
                yPosition += rowHeight + vSpacingPx  // 다음 줄은 아래에
            }
        }
    }
}

// 사용: 태그 목록이 자동으로 줄바꿈됨
FlowRow {
    listOf("Kotlin", "Compose", "Android", "MVVM", "Coroutine").forEach { tag ->
        AssistChip(onClick = {}, label = { Text(tag) })
    }
}
```

---

## 4. Modifier.layout — 단일 자식 커스텀 측정

전체 레이아웃이 아니라 **단일 Composable의 측정/배치**만 커스텀하고 싶을 때 사용합니다.

```kotlin
import androidx.compose.ui.layout.layout

// 자식을 부모 대비 특정 baseline에 맞춰 배치하는 Modifier
fun Modifier.firstBaselineToTop(firstBaselineToTop: Dp) = this.layout { measurable, constraints ->
    // measurable을 측정해 placeable 획득
    val placeable = measurable.measure(constraints)

    // 텍스트의 첫 줄 베이스라인 위치 조회
    val firstBaseline = placeable[FirstBaseline]

    // 원하는 위치(firstBaselineToTop)에 baseline이 오도록 y 좌표 역산
    val placeableY = firstBaselineToTop.roundToPx() - firstBaseline
    val height = placeable.height + placeableY

    layout(placeable.width, height) {
        placeable.placeRelative(0, placeableY)
    }
}

// 사용: 텍스트의 첫 줄 베이스라인이 상단에서 정확히 32dp에 위치
Text(
    text = "제목",
    modifier = Modifier.firstBaselineToTop(32.dp)
)
```

---

## 5. SubcomposeLayout — 조건부/지연 측정

자식의 측정 결과에 따라 **다른 자식의 내용을 결정**해야 할 때 사용합니다.
일반 `Layout`은 모든 자식을 한 번에 측정하지만, `SubcomposeLayout`은 필요할 때 그룹별로 나눠서 subcompose(재구성)할 수 있습니다.

```kotlin
import androidx.compose.ui.layout.SubcomposeLayout

// 콘텐츠 높이를 측정한 후, 그 높이에 맞춰 사이드바를 구성하는 레이아웃
@Composable
fun MatchHeightLayout(
    mainContent: @Composable () -> Unit,
    sidebarContent: @Composable (height: Dp) -> Unit
) {
    SubcomposeLayout { constraints ->
        // 1단계: "main" 슬롯을 먼저 subcompose하고 측정
        val mainPlaceables = subcompose("main", mainContent).map {
            it.measure(constraints)
        }
        val mainHeight = mainPlaceables.maxOfOrNull { it.height } ?: 0

        // 2단계: main의 측정 결과(높이)를 알고 나서 "sidebar" 슬롯을 subcompose
        // → 일반 Layout에서는 불가능한, "측정 결과에 따른 조건부 컴포지션"
        val sidebarPlaceables = subcompose("sidebar") {
            sidebarContent(mainHeight.toDp())
        }.map { it.measure(constraints.copy(minHeight = mainHeight, maxHeight = mainHeight)) }

        val totalWidth = (mainPlaceables + sidebarPlaceables).sumOf { it.width }

        layout(totalWidth, mainHeight) {
            var x = 0
            mainPlaceables.forEach { it.placeRelative(x, 0); x += it.width }
            sidebarPlaceables.forEach { it.placeRelative(x, 0); x += it.width }
        }
    }
}
```

> **Layout vs SubcomposeLayout**: 일반적인 배치는 `Layout`으로 충분합니다. `SubcomposeLayout`은 "다른 자식의 측정 결과를 알아야 이 자식의 내용을 결정할 수 있는" 특수한 경우에만 사용합니다 (예: `BoxWithConstraints`, 지연 로딩 리스트 내부 구현).

---

## 6. intrinsics — 실제 측정 없이 크기 예측

부모가 자식을 배치하기 전에 "이 자식이 대략 얼마나 클지" 미리 알아야 할 때 사용합니다.

```kotlin
@Composable
fun EqualHeightRow(content: @Composable () -> Unit) {
    Row(
        // IntrinsicSize.Max: 모든 자식 중 가장 높은 자식에 맞춰 전체 행의 높이 통일
        modifier = Modifier.height(IntrinsicSize.Max)
    ) {
        content()
    }
}

// 사용: 왼쪽/오른쪽 카드의 내용 길이가 달라도 높이가 자동으로 맞춰짐
EqualHeightRow {
    Card(modifier = Modifier.weight(1f).fillMaxHeight()) {
        Text("짧은 내용", modifier = Modifier.padding(16.dp))
    }
    Card(modifier = Modifier.weight(1f).fillMaxHeight()) {
        Text("이것은 훨씬 긴 내용입니다.\n여러 줄에 걸쳐 표시됩니다.", modifier = Modifier.padding(16.dp))
    }
}
```

---

## 7. 정리

| API | 용도 |
|-----|------|
| `Layout` | 자식들을 직접 측정(measure)하고 배치(place)하는 커스텀 레이아웃 |
| `Modifier.layout` | 단일 Composable의 측정/배치 커스텀 |
| `SubcomposeLayout` | 측정 결과에 따라 다른 자식의 내용을 조건부로 결정 |
| `IntrinsicSize` | 실제 배치 전 자식의 예상 크기 조회 (Max/Min) |

- Compose 레이아웃은 항상 **Measure → Place** 2단계
- `measure()`는 자식마다 **한 번만** 호출 가능 (여러 번 필요하면 SubcomposeLayout)
- 커스텀 레이아웃은 `Row`/`Column`/`Box` 조합으로 해결이 안 될 때만 사용 (복잡도가 높음)
