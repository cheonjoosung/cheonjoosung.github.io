---
title: (Android/UI) RecyclerView Drag & Drop — ItemTouchHelper
tags: [ Android, RecyclerView ]
style: fill
color: dark
description: ItemTouchHelper로 RecyclerView 아이템의 순서를 드래그로 변경하는 방법과 핸들 드래그, 드래그 중 시각 효과, 순서 저장 처리를 정리합니다.
---

---

## 1. 드래그로 순서 변경

`ItemTouchHelper`의 드래그 방향을 지정하고 `onMove`에서 **목록의 순서를 바꾸면** 됩니다.

```kotlin
val callback = object : ItemTouchHelper.SimpleCallback(
    ItemTouchHelper.UP or ItemTouchHelper.DOWN,      // 드래그 방향
    0                                                // 스와이프 없음
) {
    override fun onMove(
        rv: RecyclerView,
        viewHolder: RecyclerView.ViewHolder,
        target: RecyclerView.ViewHolder
    ): Boolean {
        val from = viewHolder.bindingAdapterPosition
        val to = target.bindingAdapterPosition
        if (from == RecyclerView.NO_POSITION || to == RecyclerView.NO_POSITION) return false

        adapter.moveItem(from, to)                   // 데이터 순서 변경 + notifyItemMoved
        return true
    }

    override fun onSwiped(viewHolder: RecyclerView.ViewHolder, direction: Int) {}
}

val touchHelper = ItemTouchHelper(callback)
touchHelper.attachToRecyclerView(recyclerView)
```

기본 동작은 **아이템을 길게 누르면** 드래그가 시작됩니다.

---

## 2. 어댑터에서 순서 변경

```kotlin
class ItemAdapter(private val items: MutableList<Item>) : RecyclerView.Adapter<VH>() {

    fun moveItem(from: Int, to: Int) {
        // 리스트에서 위치를 바꿈
        val item = items.removeAt(from)
        items.add(to, item)
        notifyItemMoved(from, to)                    // 이동 애니메이션
    }
    // ...
}
```

> `notifyDataSetChanged()`를 쓰면 이동 애니메이션이 없고 드래그가 끊깁니다. 반드시 **`notifyItemMoved`**를 사용하세요.

`ListAdapter`(불변 리스트)를 쓴다면 드래그 중에는 임시 목록을 별도로 유지하다가, 드롭 시점에 확정해 `submitList`합니다.

---

## 3. 드래그 핸들 — 지정한 아이콘으로만 드래그

전체 아이템이 아니라 **핸들(≡ 아이콘)을 눌렀을 때만** 드래그가 시작되도록 합니다.

```kotlin
// 길게 누르기 드래그를 끔
override fun isLongPressDragEnabled() = false
```

```kotlin
// ViewHolder의 핸들에서 직접 드래그 시작
holder.dragHandle.setOnTouchListener { _, event ->
    if (event.actionMasked == MotionEvent.ACTION_DOWN) {
        touchHelper.startDrag(holder)
    }
    false
}
```

리스트 안의 버튼 등 다른 터치 요소와 충돌을 피하고, **드래그할 수 있다는 것을 시각적으로 명확**하게 알려줍니다.

---

## 4. 드래그 중 시각 효과

```kotlin
override fun onSelectedChanged(viewHolder: RecyclerView.ViewHolder?, actionState: Int) {
    super.onSelectedChanged(viewHolder, actionState)
    if (actionState == ItemTouchHelper.ACTION_STATE_DRAG) {
        // 드래그 시작: 살짝 띄우고 반투명
        viewHolder?.itemView?.animate()?.scaleX(1.03f)?.scaleY(1.03f)?.alpha(0.9f)?.setDuration(150)?.start()
        viewHolder?.itemView?.elevation = 16f
    }
}

override fun clearView(rv: RecyclerView, viewHolder: RecyclerView.ViewHolder) {
    super.clearView(rv, viewHolder)
    // 드롭 완료: 원래 상태로 복구
    viewHolder.itemView.animate().scaleX(1f).scaleY(1f).alpha(1f).setDuration(150).start()
    viewHolder.itemView.elevation = 0f

    viewModel.saveOrder(adapter.currentItems())      // 드롭이 끝난 시점에 순서 저장
}
```

> **순서 저장은 `clearView`(드롭 완료)에서** 하세요. `onMove`는 드래그 중 **여러 번** 호출되므로 그때마다 서버/DB에 저장하면 비효율적입니다.

---

## 5. 특정 아이템은 이동 제한

```kotlin
override fun getMovementFlags(rv: RecyclerView, vh: RecyclerView.ViewHolder): Int {
    // 헤더나 고정 아이템은 드래그 불가
    if (vh is HeaderViewHolder) return makeMovementFlags(0, 0)
    return makeMovementFlags(ItemTouchHelper.UP or ItemTouchHelper.DOWN, 0)
}

// 이동 대상이 될 수 있는지 (헤더 위치로는 드롭 불가)
override fun canDropOver(rv: RecyclerView, current: RecyclerView.ViewHolder, target: RecyclerView.ViewHolder) =
    target !is HeaderViewHolder
```

---

## 6. 그리드 드래그

`GridLayoutManager`에서는 **상하좌우** 모두 허용합니다.

```kotlin
ItemTouchHelper.SimpleCallback(
    ItemTouchHelper.UP or ItemTouchHelper.DOWN or ItemTouchHelper.LEFT or ItemTouchHelper.RIGHT,
    0
)
```

---

## 7. 스와이프와 드래그 함께 사용

```kotlin
ItemTouchHelper.SimpleCallback(
    ItemTouchHelper.UP or ItemTouchHelper.DOWN,                 // 드래그
    ItemTouchHelper.LEFT or ItemTouchHelper.RIGHT               // 스와이프 삭제
)
```

하나의 콜백에서 `onMove`(순서 변경)와 `onSwiped`(삭제)를 모두 구현하면 됩니다.

---

## 8. 접근성 — 드래그가 어려운 사용자

드래그는 **모든 사용자가 쉽게 쓸 수 있는 동작이 아닙니다.** 접근성 서비스(TalkBack) 사용자를 위해 대체 수단을 제공하세요.

```kotlin
// 컨텍스트 메뉴나 버튼으로 "위로 이동 / 아래로 이동" 제공
ViewCompat.addAccessibilityAction(itemView, "위로 이동") { _, _ -> moveUp(position); true }
ViewCompat.addAccessibilityAction(itemView, "아래로 이동") { _, _ -> moveDown(position); true }
```

---

## 9. 정리

- `SimpleCallback(UP or DOWN, 0)`의 `onMove`에서 목록을 바꾸고 **`notifyItemMoved`** 호출
- 기본은 롱프레스 드래그, 핸들만 허용하려면 `isLongPressDragEnabled = false` + `startDrag(holder)`
- 드래그 중 효과는 `onSelectedChanged`, 복구와 **순서 저장은 `clearView`**에서
- 헤더 등은 `getMovementFlags`, `canDropOver`로 제한
- 스와이프와 한 콜백에서 함께 처리 가능, 접근성 대체 동작도 고려
