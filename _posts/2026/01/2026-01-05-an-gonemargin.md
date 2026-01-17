---
title: (Android/안드로이드) Layout goneMarginStart 완전 정리
tags: [Android,Kotlin]
style: fill
color: dark
description: (Android/안드로이드) Layout goneMarginStart 완전 정리 - layout_goneMarginStart가 언제, 어떻게 적용되는가?
---

## 개요

- ConstraintLayout을 쓰다 보면 이런 요구가 자주 생깁니다.
  - 옆에 있는 뷰가 사라지면(GONE) 간격을 0으로 붙이고
  - 뷰가 보이면(VISIBLE) 간격을 8dp 같은 값으로 유지하고 싶다
  - 이때 사용하는 속성이 바로 layout_goneMarginStart(일반적으로 “goneMarginStart”라고 부름)입니다.

---

## 1. goneMarginStart는 무엇인가?

- goneMarginStart는 정확히 말하면 ConstraintLayout의 속성인:
  - app:layout_goneMarginStart
  - app:layout_goneMarginEnd
  - app:layout_goneMarginTop
  - app:layout_goneMarginBottom
- 핵심 역할은:
  - 제약(Constraint) 상대 뷰가 GONE일 때만 적용되는 “대체 마진”입니다.
  - 즉, 기본 마진(layout_marginStart)과 다르게 상대가 GONE인 경우에만 별도의 마진 값을 쓰는 개념입니다.

---

## 2. 언제 적용되는가? (가장 중요한 포인트)

- ✅ 적용 조건
  - goneMarginStart는 아래 조건이 모두 맞을 때 적용됩니다.
  - ConstraintLayout 안의 뷰가 어떤 방향의 constraint를 다른 뷰에 걸고 있고 그 constraint의 대상 뷰가 GONE 상태일 때

---

## 3. 예제로 이해하기 (가장 흔한 케이스)

- 상황 
  - textView가 imageView의 오른쪽에 붙어 있다
  - imageView가 보이면 간격 8dp
  - imageView가 GONE이면 간격 0dp로 붙이고 싶다
- 코드
  - ```xml
    <ImageView
        android:id="@+id/ivIcon"
        android:layout_width="24dp"
        android:layout_height="24dp"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent" />

    <TextView
        android:id="@+id/tvTitle"
        android:layout_width="0dp"
        android:layout_height="wrap_content"
        android:layout_marginStart="8dp"
        app:layout_goneMarginStart="0dp"
        app:layout_constraintStart_toEndOf="@id/ivIcon"
        app:layout_constraintTop_toTopOf="@id/ivIcon" />
    ```
- 동작 결과
  - ivIcon이 VISIBLE/INVISIBLE이면 → marginStart = 8dp 적용
  - ivIcon이 GONE이면 → goneMarginStart = 0dp 적용

---

## 4. INVISIBLE에서는 왜 적용 안 되나?

- 여기서 매우 흔한 오해가 있습니다.
  - INVISIBLE = 보이지 않지만 자리 차지함
  - GONE = 보이지 않고 자리도 차지하지 않음
- goneMarginStart는 이름 그대로 GONE일 때만 동작합니다.
- INVISIBLE에는 적용되지 않습니다.

---