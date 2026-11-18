---
title: (Android/UI) RecyclerView ItemAnimator 커스텀
tags: [ Android, RecyclerView ]
style: fill
color: dark
description: RecyclerView의 DefaultItemAnimator 동작과 SimpleItemAnimator를 확장해 추가, 삭제, 이동, 변경 애니메이션을 커스텀하는 방법과 성능 주의점을 정리합니다.
---

---

## 1. ItemAnimator란?

RecyclerView에서 아이템이 **추가, 삭제, 이동, 변경**될 때 실행되는 애니메이션을 담당합니다.
기본값은 `DefaultItemAnimator`(페이드, 이동)이며, 교체하거나 확장할 수 있습니다.

```kotlin
recyclerView.itemAnimator = DefaultItemAnimator()      // 기본
recyclerView.itemAnimator = null                       // 애니메이션 끄기
```

---

## 2. 애니메이션이 동작하려면 — 올바른 갱신

`notifyDataSetChanged()`는 **전체 갱신**이라 아이템 단위 애니메이션이 동작하지 않습니다. 변경 내용을 알려줘야 합니다.

```kotlin
adapter.notifyItemInserted(position)        // 추가 애니메이션
adapter.notifyItemRemoved(position)         // 삭제 애니메이션
adapter.notifyItemMoved(from, to)           // 이동 애니메이션
adapter.notifyItemChanged(position)         // 변경 애니메이션

// ListAdapter + DiffUtil을 쓰면 자동으로 적절한 notify 호출
listAdapter.submitList(newList)
```

> **`ListAdapter`/`DiffUtil`을 쓰고 `areItemsTheSame`을 id로 정확히 구현**해야 변경이 추가/삭제/이동으로 올바르게 해석되어 애니메이션이 자연스럽습니다.

---

## 3. 기본 애니메이터 속성 조정

```kotlin
(recyclerView.itemAnimator as DefaultItemAnimator).apply {
    addDuration = 300
    removeDuration = 300
    moveDuration = 250
    changeDuration = 200
    supportsChangeAnimations = false      // 변경 시 깜빡임(crossfade)이 싫다면 끄기
}
```

> **`supportsChangeAnimations = false`**: 텍스트만 바뀌는 `notifyItemChanged`에서 아이템이 **깜빡이는(페이드 아웃/인)** 현상을 없앱니다. 좋아요 버튼, 카운트 갱신 등에서 흔히 설정합니다.

---

## 4. SimpleItemAnimator 확장 — 커스텀 애니메이션

`SimpleItemAnimator`의 `animateXxx` 메서드를 구현합니다. 각 메서드는 **애니메이션을 시작해야 하면 `true`**를 반환하고, 끝나면 `dispatchXxxFinished`를 호출해야 합니다.

```kotlin
class SlideInItemAnimator : SimpleItemAnimator() {

    private val pendingAdds = mutableListOf<RecyclerView.ViewHolder>()

    override fun animateAdd(holder: RecyclerView.ViewHolder): Boolean {
        // 시작 상태 설정: 오른쪽 밖에서 투명하게
        holder.itemView.translationX = holder.itemView.width.toFloat()
        holder.itemView.alpha = 0f
        pendingAdds.add(holder)
        return true                                   // 애니메이션 실행 예정
    }

    override fun runPendingAnimations() {
        if (pendingAdds.isEmpty()) return
        pendingAdds.toList().forEach { holder ->
            holder.itemView.animate()
                .translationX(0f).alpha(1f)
                .setDuration(addDuration)
                .setListener(object : AnimatorListenerAdapter() {
                    override fun onAnimationEnd(animation: Animator) {
                        holder.itemView.animate().setListener(null)
                        dispatchAddFinished(holder)   // 반드시 호출
                        dispatchFinishedWhenDone()
                    }
                })
                .start()
        }
        pendingAdds.clear()
    }

    // 나머지 메서드는 단순 구현 또는 애니메이션 없이 종료
    override fun animateRemove(holder: RecyclerView.ViewHolder) = false.also { dispatchRemoveFinished(holder) }
    override fun animateMove(h: RecyclerView.ViewHolder, fx: Int, fy: Int, tx: Int, ty: Int) = false.also { dispatchMoveFinished(h) }
    override fun animateChange(o: RecyclerView.ViewHolder, n: RecyclerView.ViewHolder?, fx: Int, fy: Int, tx: Int, ty: Int) = false.also { dispatchChangeFinished(o, true) }
    override fun endAnimation(item: RecyclerView.ViewHolder) { item.itemView.animate().cancel() }
    override fun endAnimations() {}
    override fun isRunning() = pendingAdds.isNotEmpty()
}
```

> **`dispatchXxxFinished`를 호출하지 않으면** RecyclerView가 해당 ViewHolder를 계속 "애니메이션 중"으로 보아 **재활용되지 않고 화면이 이상해집니다.** 모든 경로(취소 포함)에서 호출하는지 확인하세요.

직접 구현이 부담스러우면 `jp.wasabeef:recyclerview-animators` 같은 라이브러리의 `SlideInLeftAnimator`, `ScaleInAnimator`를 사용하는 것도 좋습니다.

---

## 5. 순차(Staggered) 등장 — 레이아웃 애니메이션

아이템이 처음 표시될 때 **하나씩 순서대로** 나타나게 하려면 `LayoutAnimation`을 사용합니다.

```xml
<!-- res/anim/layout_animation_fall_down.xml -->
<layoutAnimation xmlns:android="http://schemas.android.com/apk/res/android"
    android:animation="@anim/item_fall_down"
    android:delay="15%"
    android:animationOrder="normal" />
```

```kotlin
recyclerView.layoutAnimation =
    AnimationUtils.loadLayoutAnimation(context, R.anim.layout_animation_fall_down)
recyclerView.scheduleLayoutAnimation()                // 데이터 로드 후 호출
```

ItemAnimator는 **변경(추가/삭제 등)**, LayoutAnimation은 **최초 표시** 때 쓴다고 구분하면 됩니다.

---

## 6. 변경 애니메이션 — Payload로 부분 갱신

전체 아이템을 다시 바인딩하지 않고 **바뀐 부분만** 갱신하면 깜빡임 없이 부드럽습니다.

```kotlin
class DiffCallback : DiffUtil.ItemCallback<Item>() {
    override fun areItemsTheSame(a: Item, b: Item) = a.id == b.id
    override fun areContentsTheSame(a: Item, b: Item) = a == b

    override fun getChangePayload(old: Item, new: Item): Any? =
        if (old.likeCount != new.likeCount) "LIKE_CHANGED" else null
}

override fun onBindViewHolder(holder: VH, position: Int, payloads: MutableList<Any>) {
    if (payloads.contains("LIKE_CHANGED")) {
        holder.updateLike(getItem(position))          // 좋아요 영역만 갱신 + 개별 애니메이션
    } else {
        super.onBindViewHolder(holder, position, payloads)
    }
}
```

---

## 7. 주의사항

- 애니메이션 중 **ViewHolder가 재사용**되므로, 뷰 속성(`alpha`, `translation`)을 바꾸면 **원래 값으로 복원**해야 합니다 (`endAnimation`, `onViewRecycled`)
- 아이템 수가 매우 많이 한 번에 변경될 때 애니메이션은 부담이 됩니다 → 대량 갱신은 애니메이션을 꺼도 됩니다
- `setHasFixedSize(true)`는 아이템 추가/삭제 시 크기 변경이 없을 때만 의미가 있습니다
- 접근성 설정에서 애니메이션을 줄인 사용자를 고려하세요

---

## 8. 정리

- `ItemAnimator`는 추가/삭제/이동/변경 시의 애니메이션을 담당 (기본 `DefaultItemAnimator`)
- `notifyDataSetChanged` 대신 **`notifyItemXxx` 또는 `ListAdapter` + DiffUtil**을 써야 애니메이션이 동작
- `supportsChangeAnimations = false`로 변경 시 깜빡임 제거, 시간은 `addDuration` 등으로 조정
- 커스텀은 `SimpleItemAnimator` 확장, **`dispatchXxxFinished` 호출 필수**
- 최초 순차 등장은 `LayoutAnimation`, 부분 변경은 **payload**로 갱신
