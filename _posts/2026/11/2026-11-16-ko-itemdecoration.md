---
title: (Android/UI) RecyclerView ItemDecoration 커스터마이징
tags: [ Android, RecyclerView ]
style: fill
color: dark
description: RecyclerView ItemDecoration의 getItemOffsets, onDraw, onDrawOver로 구분선, 간격, 그리드 여백, 스티키 헤더를 구현하는 방법을 정리합니다.
---

---

## 1. ItemDecoration이란?

아이템 뷰 자체를 수정하지 않고, RecyclerView가 아이템 **주변과 위에 장식**을 추가하는 도구입니다.

```
getItemOffsets  : 아이템 주변에 여백 확보 (공간 만들기)
onDraw          : 아이템 아래(뒤)에 그리기
onDrawOver      : 아이템 위(앞)에 그리기
```

아이템 레이아웃에 마진/구분선 뷰를 넣는 대신 사용하면 **재사용성과 유지보수성**이 좋아집니다.

---

## 2. 간격 추가 — getItemOffsets

```kotlin
class SpacingDecoration(private val spacePx: Int) : RecyclerView.ItemDecoration() {

    override fun getItemOffsets(
        outRect: Rect, view: View, parent: RecyclerView, state: RecyclerView.State
    ) {
        val position = parent.getChildAdapterPosition(view)
        if (position == RecyclerView.NO_POSITION) return

        outRect.bottom = spacePx                     // 아이템 아래 여백
        if (position == 0) outRect.top = spacePx     // 첫 아이템 위 여백
    }
}

recyclerView.addItemDecoration(SpacingDecoration(16.dp))
```

> `outRect`는 아이템 **바깥쪽으로 확보할 여백**입니다. 이 값만큼 아이템이 차지하는 공간이 늘어납니다.

---

## 3. 구분선 — onDraw

```kotlin
class DividerDecoration(
    context: Context,
    @ColorRes color: Int = R.color.divider,
    private val heightPx: Int = 1,
    private val insetStart: Int = 0
) : RecyclerView.ItemDecoration() {

    private val paint = Paint().apply { this.color = ContextCompat.getColor(context, color) }

    override fun getItemOffsets(outRect: Rect, view: View, parent: RecyclerView, state: RecyclerView.State) {
        outRect.bottom = heightPx                    // 구분선 높이만큼 공간 확보
    }

    override fun onDraw(canvas: Canvas, parent: RecyclerView, state: RecyclerView.State) {
        val left = parent.paddingLeft + insetStart
        val right = parent.width - parent.paddingRight

        for (i in 0 until parent.childCount - 1) {   // 마지막 아이템 아래는 그리지 않음
            val child = parent.getChildAt(i)
            val top = child.bottom + (child.layoutParams as RecyclerView.LayoutParams).bottomMargin
            canvas.drawRect(left.toFloat(), top.toFloat(), right.toFloat(), (top + heightPx).toFloat(), paint)
        }
    }
}
```

기본 제공 `DividerItemDecoration(context, LinearLayoutManager.VERTICAL)`도 있지만, 여백(inset)이나 마지막 아이템 처리가 필요하면 직접 구현합니다.

---

## 4. 그리드 여백 — 모서리 간격 균등

그리드에서 아이템 사이 간격이 **겹치지 않고 균등**하도록 계산합니다.

```kotlin
class GridSpacingDecoration(
    private val spanCount: Int,
    private val spacing: Int,
    private val includeEdge: Boolean
) : RecyclerView.ItemDecoration() {

    override fun getItemOffsets(outRect: Rect, view: View, parent: RecyclerView, state: RecyclerView.State) {
        val position = parent.getChildAdapterPosition(view)
        if (position == RecyclerView.NO_POSITION) return
        val column = position % spanCount

        if (includeEdge) {
            outRect.left = spacing - column * spacing / spanCount
            outRect.right = (column + 1) * spacing / spanCount
            if (position < spanCount) outRect.top = spacing
            outRect.bottom = spacing
        } else {
            outRect.left = column * spacing / spanCount
            outRect.right = spacing - (column + 1) * spacing / spanCount
            if (position >= spanCount) outRect.top = spacing
        }
    }
}
```

> 아이템에 직접 `margin`을 주면 **인접 아이템의 마진이 합쳐져** 간격이 두 배가 되거나 가장자리가 어긋납니다. 위 공식은 이를 방지합니다.

---

## 5. 스티키 헤더 — onDrawOver

날짜별 그룹처럼 **스크롤해도 화면 상단에 고정되는** 헤더를 아이템 위에 그립니다.

```kotlin
class StickyHeaderDecoration(
    private val isHeader: (Int) -> Boolean,                 // 위치가 헤더인지
    private val headerLayout: (Int) -> View                  // 헤더 뷰 생성
) : RecyclerView.ItemDecoration() {

    override fun onDrawOver(canvas: Canvas, parent: RecyclerView, state: RecyclerView.State) {
        val topChild = parent.getChildAt(0) ?: return
        val topPosition = parent.getChildAdapterPosition(topChild)
        if (topPosition == RecyclerView.NO_POSITION) return

        val headerPos = (topPosition downTo 0).firstOrNull(isHeader) ?: return
        val header = headerLayout(headerPos)
        measureHeader(parent, header)

        // 다음 헤더가 올라오면 밀어내기
        val nextHeaderTop = findNextHeaderTop(parent, headerPos)
        val offset = if (nextHeaderTop != null && nextHeaderTop < header.height)
            nextHeaderTop - header.height else 0

        canvas.save()
        canvas.translate(0f, offset.toFloat())
        header.draw(canvas)
        canvas.restore()
    }
    // measureHeader, findNextHeaderTop은 헤더 크기 측정/다음 헤더 위치 탐색
}
```

구현이 복잡하므로 라이브러리(`sticky-headers`)나 `ConcatAdapter`+별도 뷰를 고려하는 것도 방법입니다.

---

## 6. 둥근 배경/그룹 카드 — onDraw

연속된 아이템을 하나의 카드처럼 묶어 **첫/마지막 아이템에만 둥근 모서리**를 줄 수 있습니다.

```kotlin
override fun onDraw(c: Canvas, parent: RecyclerView, state: RecyclerView.State) {
    for (i in 0 until parent.childCount) {
        val child = parent.getChildAt(i)
        val pos = parent.getChildAdapterPosition(child)
        val isFirst = pos == 0
        val isLast = pos == state.itemCount - 1
        // 위치에 따라 위/아래 모서리를 다르게 그린 배경을 child 영역에 draw
        drawCardBackground(c, child, isFirst, isLast)
    }
}
```

---

## 7. 주의사항

- `onDraw`는 매 프레임 호출되므로 **`Paint`/`Rect`를 멤버로 재사용**하세요 (객체 생성 금지)
- 아이템이 추가/삭제되어도 Decoration은 자동 갱신되지 않습니다. 필요하면 `recyclerView.invalidateItemDecorations()`
- `getChildAdapterPosition`이 `NO_POSITION`을 반환할 수 있으니 **항상 확인**하세요 (삭제 애니메이션 중)
- 같은 Decoration을 중복 추가하지 않도록 주의 (`addItemDecoration`은 누적됨)
- 여러 Decoration을 함께 쓰면 `getItemOffsets`의 여백이 **합산**됩니다

---

## 8. 정리

- `getItemOffsets`로 **공간 확보**, `onDraw`/`onDrawOver`로 **아이템 뒤/앞에 그리기**
- 간격은 마진 대신 Decoration에서 계산하면 그리드에서도 **균등**
- 구분선은 `getItemOffsets`로 높이를 확보하고 `onDraw`로 그림 (마지막 항목 제외 가능)
- 스티키 헤더는 `onDrawOver`에서 현재 헤더와 다음 헤더의 밀어내기를 계산
- `onDraw`에서는 객체를 재사용하고 `NO_POSITION`을 방어
