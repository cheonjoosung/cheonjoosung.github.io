---
title: (Android/UI) RecyclerView SwipeToDismiss — ItemTouchHelper
tags: [ Android, RecyclerView ]
style: fill
color: dark
description: ItemTouchHelper로 RecyclerView 아이템을 밀어서 삭제하는 스와이프 구현과 배경 아이콘 그리기, 실행 취소 Snackbar, 확인 상태 유지 방법을 정리합니다.
---

---

## 1. ItemTouchHelper

RecyclerView 아이템의 **스와이프와 드래그**를 처리하는 유틸리티입니다.
`ItemTouchHelper.Callback`(또는 `SimpleCallback`)에서 방향과 동작을 정의합니다.

```
스와이프: 아이템을 좌/우로 밀어 삭제/보관
드래그  : 아이템을 길게 눌러 순서 변경 (다음 글에서)
```

---

## 2. 기본 스와이프 삭제

```kotlin
val callback = object : ItemTouchHelper.SimpleCallback(
    0,                                               // 드래그 방향 (사용 안 함)
    ItemTouchHelper.LEFT or ItemTouchHelper.RIGHT    // 스와이프 방향
) {
    // 드래그 이동은 사용하지 않으므로 false
    override fun onMove(
        rv: RecyclerView, vh: RecyclerView.ViewHolder, target: RecyclerView.ViewHolder
    ) = false

    override fun onSwiped(viewHolder: RecyclerView.ViewHolder, direction: Int) {
        val position = viewHolder.bindingAdapterPosition
        viewModel.delete(adapter.currentList[position])
    }
}

ItemTouchHelper(callback).attachToRecyclerView(recyclerView)
```

> `adapterPosition` 대신 **`bindingAdapterPosition`**(androidx.recyclerview 1.2+)을 사용하세요. `ConcatAdapter` 등에서도 올바른 위치를 반환합니다. 값이 `NO_POSITION`(-1)일 수 있으니 확인이 필요합니다.

---

## 3. 특정 아이템은 스와이프 막기

```kotlin
override fun getSwipeDirs(rv: RecyclerView, vh: RecyclerView.ViewHolder): Int {
    // 헤더나 고정 아이템은 스와이프 불가
    return if (vh is HeaderViewHolder) 0 else super.getSwipeDirs(rv, vh)
}
```

---

## 4. 배경과 아이콘 그리기 — onChildDraw

스와이프할 때 아이템 뒤에 **빨간 배경과 휴지통 아이콘**을 보여줍니다.

```kotlin
private val background = ColorDrawable(Color.parseColor("#E53935"))
private val deleteIcon = ContextCompat.getDrawable(context, R.drawable.ic_delete)!!

override fun onChildDraw(
    c: Canvas, rv: RecyclerView, vh: RecyclerView.ViewHolder,
    dX: Float, dY: Float, actionState: Int, isCurrentlyActive: Boolean
) {
    super.onChildDraw(c, rv, vh, dX, dY, actionState, isCurrentlyActive)   // 아이템 이동은 기본 처리
    val item = vh.itemView

    if (dX == 0f) { background.setBounds(0, 0, 0, 0); return }

    if (dX > 0) {   // 오른쪽으로 스와이프 → 왼쪽에 배경 노출
        background.setBounds(item.left, item.top, item.left + dX.toInt(), item.bottom)
    } else {        // 왼쪽으로 스와이프 → 오른쪽에 배경 노출
        background.setBounds(item.right + dX.toInt(), item.top, item.right, item.bottom)
    }
    background.draw(c)

    // 아이콘은 배경 가운데(세로)에 배치
    val margin = (item.height - deleteIcon.intrinsicHeight) / 2
    val top = item.top + margin
    if (dX > 0) {
        deleteIcon.setBounds(item.left + margin, top, item.left + margin + deleteIcon.intrinsicWidth, top + deleteIcon.intrinsicHeight)
    } else {
        deleteIcon.setBounds(item.right - margin - deleteIcon.intrinsicWidth, top, item.right - margin, top + deleteIcon.intrinsicHeight)
    }
    deleteIcon.draw(c)
}
```

> `super.onChildDraw`를 호출해야 아이템이 손가락을 따라 움직입니다. 배경은 아이템 **아래**에 그려져야 하므로 `super` 호출 후가 아니라 상황에 맞게 순서를 확인하세요 (배경이 아이템을 가리면 `onChildDraw` 대신 `onChildDrawOver` 사용 여부 점검).

---

## 5. 스와이프 임계값과 속도

```kotlin
override fun getSwipeThreshold(vh: RecyclerView.ViewHolder) = 0.4f        // 40% 이상 밀어야 삭제 (기본 0.5)
override fun getSwipeEscapeVelocity(defaultValue: Float) = defaultValue * 2   // 빠르게 밀어야 인정
```

실수로 삭제되는 것을 줄이려면 임계값을 높이거나 속도 조건을 강화합니다.

---

## 6. 실행 취소 — Snackbar

삭제는 되돌릴 수 있어야 사용성이 좋습니다. **즉시 데이터에서 지우지 말고** UI에서만 제거한 뒤 Snackbar로 복구 기회를 줍니다.

```kotlin
override fun onSwiped(viewHolder: RecyclerView.ViewHolder, direction: Int) {
    val position = viewHolder.bindingAdapterPosition
    val item = adapter.currentList[position]

    viewModel.removeTemporarily(item)                     // 목록에서만 제거 (실제 삭제는 보류)

    Snackbar.make(binding.root, "삭제되었습니다", Snackbar.LENGTH_LONG)
        .setAction("실행 취소") { viewModel.restore(item, position) }
        .addCallback(object : Snackbar.Callback() {
            override fun onDismissed(sb: Snackbar, event: Int) {
                // 실행 취소가 눌리지 않고 사라졌다면 실제 삭제 확정
                if (event != DISMISS_EVENT_ACTION) viewModel.commitDelete(item)
            }
        })
        .show()
}
```

---

## 7. 스와이프 후 되돌리기 — 확인 단계가 필요할 때

삭제 확인이 필요하면 `onSwiped` 안에서 아이템을 원래 상태로 복구합니다.

```kotlin
override fun onSwiped(viewHolder: RecyclerView.ViewHolder, direction: Int) {
    val position = viewHolder.bindingAdapterPosition
    adapter.notifyItemChanged(position)           // 스와이프된 뷰를 원위치로 복구

    AlertDialog.Builder(context)
        .setMessage("삭제할까요?")
        .setPositiveButton("삭제") { _, _ -> viewModel.delete(adapter.currentList[position]) }
        .setNegativeButton("취소", null)
        .show()
}
```

`notifyItemChanged`를 호출하지 않으면 아이템이 화면 밖에 걸친 채 남습니다.

---

## 8. 정리

- `ItemTouchHelper.SimpleCallback(0, LEFT or RIGHT)`로 스와이프 구성, `attachToRecyclerView`로 연결
- 위치는 `bindingAdapterPosition` 사용 (`NO_POSITION` 확인)
- 배경과 아이콘은 `onChildDraw`에서 `dX` 방향에 따라 그림
- 임계값(`getSwipeThreshold`)과 속도(`getSwipeEscapeVelocity`)로 오삭제 방지
- 삭제는 **Snackbar로 실행 취소** 제공, 확인이 필요하면 `notifyItemChanged`로 복구 후 다이얼로그
- `ListAdapter`와 함께 쓸 때는 데이터를 바꾸면 `submitList`로 갱신
