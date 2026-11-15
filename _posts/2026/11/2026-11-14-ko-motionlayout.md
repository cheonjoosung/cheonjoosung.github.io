---
title: (Android/UI) MotionLayout 기초 — ConstraintSet 애니메이션
tags: [ Android, Animation ]
style: fill
color: dark
description: MotionLayout의 Scene 파일 구성과 ConstraintSet, Transition, KeyFrame, OnSwipe 드래그 연동, 진행 상태 리스너 사용법을 예제로 정리합니다.
---

---

## 1. MotionLayout이란?

`ConstraintLayout`을 확장한 레이아웃으로, **두 개의 ConstraintSet(시작/끝) 사이의 전환**을 선언적으로 애니메이션합니다.
특히 **스와이프나 드래그와 애니메이션 진행도를 연동**할 수 있는 것이 강점입니다.

```
ConstraintSet(start)  ──  Transition  ──▶  ConstraintSet(end)
      레이아웃 A         (시간/드래그에 따라 보간)       레이아웃 B
```

**의존성**: ConstraintLayout 2.0 이상이면 포함됩니다.
```kotlin
implementation("androidx.constraintlayout:constraintlayout:2.1.4")
```

---

## 2. 레이아웃 — 루트를 MotionLayout으로

```xml
<androidx.constraintlayout.motion.widget.MotionLayout
    android:id="@+id/motionLayout"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    app:layoutDescription="@xml/scene_main">

    <!-- 여기에는 뷰와 id만 선언. 위치/크기는 Scene 파일에서 정의 -->
    <View
        android:id="@+id/box"
        android:layout_width="64dp"
        android:layout_height="64dp"
        android:background="@color/purple_500" />

</androidx.constraintlayout.motion.widget.MotionLayout>
```

> 이 레이아웃의 자식 뷰는 **제약(constraint)을 Scene의 ConstraintSet에서** 정의합니다. 레이아웃 XML에 제약을 직접 쓰면 Scene 값으로 덮입니다.

---

## 3. MotionScene 파일

`res/xml/scene_main.xml`에 시작/끝 상태와 전환을 정의합니다.

```xml
<MotionScene xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:motion="http://schemas.android.com/apk/res-auto">

    <!-- 전환: start → end -->
    <Transition
        motion:constraintSetStart="@id/start"
        motion:constraintSetEnd="@id/end"
        motion:duration="1000">

        <!-- 터치(드래그)로 진행도 제어 -->
        <OnSwipe
            motion:touchAnchorId="@id/box"
            motion:touchAnchorSide="right"
            motion:dragDirection="dragRight" />
    </Transition>

    <!-- 시작 상태 -->
    <ConstraintSet android:id="@+id/start">
        <Constraint android:id="@id/box"
            android:layout_width="64dp" android:layout_height="64dp"
            motion:layout_constraintStart_toStartOf="parent"
            motion:layout_constraintTop_toTopOf="parent" />
    </ConstraintSet>

    <!-- 끝 상태 -->
    <ConstraintSet android:id="@+id/end">
        <Constraint android:id="@id/box"
            android:layout_width="64dp" android:layout_height="64dp"
            motion:layout_constraintEnd_toEndOf="parent"
            motion:layout_constraintBottom_toBottomOf="parent" />
    </ConstraintSet>
</MotionScene>
```

`OnSwipe`가 있으면 박스를 **손가락으로 끌어** 시작↔끝 사이를 이동할 수 있습니다.

---

## 4. 코드로 제어

```kotlin
binding.motionLayout.transitionToEnd()            // 시작 → 끝 애니메이션
binding.motionLayout.transitionToStart()          // 끝 → 시작
binding.motionLayout.progress = 0.5f              // 진행도를 직접 지정 (0f~1f)

// 상태 지정 (애니메이션 없이 이동)
binding.motionLayout.setTransition(R.id.start, R.id.end)
```

---

## 5. 클릭으로 시작 — OnClick

```xml
<Transition motion:constraintSetStart="@id/start" motion:constraintSetEnd="@id/end">
    <OnClick
        motion:targetId="@id/box"
        motion:clickAction="toggle" />       <!-- toggle, transitionToEnd, transitionToStart -->
</Transition>
```

---

## 6. 속성 변화 — CustomAttribute

크기/위치 외에 **색상, 텍스트 크기 등 임의의 속성**도 전환할 수 있습니다.

```xml
<Constraint android:id="@id/box">
    <CustomAttribute
        motion:attributeName="backgroundColor"
        motion:customColorValue="#9C27B0" />
</Constraint>
```

```xml
<!-- end 상태 -->
<Constraint android:id="@id/box">
    <CustomAttribute
        motion:attributeName="backgroundColor"
        motion:customColorValue="#FF5722" />
</Constraint>
```

`setBackgroundColor`처럼 **setter가 있는 속성**이면 사용할 수 있습니다.

---

## 7. KeyFrameSet — 중간 지점 제어

직선 이동이 아닌 **경로와 중간 값**을 지정합니다.

```xml
<Transition ...>
    <KeyFrameSet>
        <!-- 진행도 50%에서 위쪽으로 휘어 이동 -->
        <KeyPosition
            motion:motionTarget="@id/box"
            motion:framePosition="50"
            motion:keyPositionType="parentRelative"
            motion:percentX="0.5"
            motion:percentY="0.2" />

        <!-- 진행도 50%에서 가장 커졌다가 끝에서 원래 크기 -->
        <KeyAttribute
            motion:motionTarget="@id/box"
            motion:framePosition="50"
            android:scaleX="1.6"
            android:scaleY="1.6" />
    </KeyFrameSet>
</Transition>
```

| 키 종류 | 용도 |
|--------|------|
| `KeyPosition` | 이동 경로 곡선 조정 |
| `KeyAttribute` | 중간 지점의 scale, alpha, rotation 등 |
| `KeyCycle` | 흔들림 같은 주기적 변화 |

---

## 8. 진행 리스너

```kotlin
binding.motionLayout.setTransitionListener(object : MotionLayout.TransitionListener {
    override fun onTransitionChange(layout: MotionLayout, startId: Int, endId: Int, progress: Float) {
        // progress 0f..1f — 스크롤에 맞춰 다른 뷰를 갱신할 때 사용
    }
    override fun onTransitionCompleted(layout: MotionLayout, currentId: Int) {
        // 완료: currentId == R.id.end 이면 끝 상태
    }
    override fun onTransitionStarted(layout: MotionLayout, startId: Int, endId: Int) {}
    override fun onTransitionTrigger(layout: MotionLayout, triggerId: Int, positive: Boolean, progress: Float) {}
})
```

---

## 9. 대표 활용 — 접히는 헤더

`NestedScrollView`/`RecyclerView`와 함께 스크롤에 따라 **헤더가 접히는** 효과를 만듭니다.

```xml
<OnSwipe
    motion:touchAnchorId="@id/recyclerView"
    motion:touchAnchorSide="top"
    motion:dragDirection="dragUp"
    motion:onTouchUp="decelerate" />
```

RecyclerView 위로 스크롤하면 헤더 이미지가 작아지고 제목이 툴바로 이동하는 패턴이 이 설정으로 구현됩니다.

---

## 10. 정리

- MotionLayout은 **두 ConstraintSet 사이의 Transition**을 선언적으로 정의
- 레이아웃에는 뷰만 두고, 위치/크기는 **MotionScene**의 ConstraintSet에서 정의
- `OnSwipe`(드래그), `OnClick`(탭)으로 시작 방식을 지정
- `CustomAttribute`로 색 등 임의 속성, `KeyFrameSet`으로 경로와 중간 값을 제어
- 단순한 요소 하나의 애니메이션은 `animate()`가 더 간단하고, **드래그와 연동되는 복합 전환**에 MotionLayout이 적합
- Compose에서는 `MotionLayout` 대신 `AnimatedVisibility`, `animate*AsState` 등을 사용
