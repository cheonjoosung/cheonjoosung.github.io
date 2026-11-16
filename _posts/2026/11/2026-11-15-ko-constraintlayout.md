---
title: (Android/UI) ConstraintLayout 심화 — Barrier, Chain, Group
tags: [ Android, ConstraintLayout ]
style: fill
color: dark
description: ConstraintLayout의 Barrier, Chain, Group, Guideline, Flow 같은 고급 기능으로 중첩 없는 복잡한 레이아웃을 구성하는 방법을 정리합니다.
---

---

## 1. 왜 고급 기능이 필요한가?

기본 제약(상하좌우 연결)만으로는 **크기가 변하는 뷰** 때문에 정렬이 깨지거나, 여러 뷰를 한 번에 숨기는 처리가 번거롭습니다.
Barrier, Chain, Group 등을 사용하면 **중첩 레이아웃 없이** 복잡한 화면을 평평하게(flat) 구성할 수 있어 성능에도 유리합니다.

---

## 2. Guideline — 보이지 않는 기준선

화면의 비율이나 고정 위치에 기준선을 두고 뷰를 정렬합니다.

```xml
<androidx.constraintlayout.widget.Guideline
    android:id="@+id/guideline"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:orientation="vertical"
    app:layout_constraintGuide_percent="0.4" />    <!-- 가로 40% 위치 -->

<TextView
    android:layout_width="0dp"
    android:layout_height="wrap_content"
    app:layout_constraintStart_toEndOf="@id/guideline"
    app:layout_constraintEnd_toEndOf="parent" />
```

- `guide_percent`: 비율, `guide_begin`/`guide_end`: 시작/끝으로부터의 고정 거리(dp)

---

## 3. Barrier — 가장 긴 뷰를 기준으로 정렬

**문제**: 라벨의 글자 수가 다를 때, 입력창을 "가장 긴 라벨 오른쪽"에 맞추고 싶다.

```xml
<TextView android:id="@+id/labelName" android:text="이름" ... />
<TextView android:id="@+id/labelEmail" android:text="이메일 주소" ... />

<!-- 두 라벨 중 가장 오른쪽 끝에 생기는 가상의 벽 -->
<androidx.constraintlayout.widget.Barrier
    android:id="@+id/labelBarrier"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    app:barrierDirection="end"
    app:constraint_referenced_ids="labelName,labelEmail" />

<EditText
    android:id="@+id/inputName"
    app:layout_constraintStart_toEndOf="@id/labelBarrier"    <!-- 가장 긴 라벨 뒤에 -->
    app:layout_constraintBaseline_toBaselineOf="@id/labelName" />
<EditText
    android:id="@+id/inputEmail"
    app:layout_constraintStart_toEndOf="@id/labelBarrier"
    app:layout_constraintBaseline_toBaselineOf="@id/labelEmail" />
```

> **Guideline과의 차이**: Guideline은 고정 위치이고, Barrier는 **참조한 뷰들의 크기에 따라 위치가 동적으로 변합니다.** 다국어로 문구 길이가 달라져도 레이아웃이 깨지지 않습니다.

---

## 4. Chain — 여러 뷰를 한 줄로 연결

뷰들을 **양방향으로 연결**하면 체인이 되며, 첫 번째 뷰에 `chainStyle`을 지정합니다.

```xml
<Button android:id="@+id/b1"
    app:layout_constraintStart_toStartOf="parent"
    app:layout_constraintEnd_toStartOf="@id/b2"
    app:layout_constraintHorizontal_chainStyle="spread" />
<Button android:id="@+id/b2"
    app:layout_constraintStart_toEndOf="@id/b1"
    app:layout_constraintEnd_toStartOf="@id/b3" />
<Button android:id="@+id/b3"
    app:layout_constraintStart_toEndOf="@id/b2"
    app:layout_constraintEnd_toEndOf="parent" />
```

| chainStyle | 배치 |
|-----------|------|
| `spread` | 남은 공간을 균등 분배 (기본) |
| `spread_inside` | 양 끝은 붙이고 사이만 균등 분배 |
| `packed` | 가운데로 모아서 배치 |

**비율 분배**: 체인에서 `0dp` + `layout_constraintHorizontal_weight`로 LinearLayout의 weight처럼 비율을 줄 수 있습니다.

```xml
<Button android:layout_width="0dp" app:layout_constraintHorizontal_weight="1" ... />
<Button android:layout_width="0dp" app:layout_constraintHorizontal_weight="2" ... />   <!-- 1:2 -->
```

---

## 5. Group — 여러 뷰 한 번에 제어

```xml
<androidx.constraintlayout.widget.Group
    android:id="@+id/detailGroup"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    app:constraint_referenced_ids="detailTitle,detailBody,detailImage" />
```

```kotlin
binding.detailGroup.isVisible = false       // 세 뷰를 한꺼번에 숨김
```

> Group은 `visibility`만 일괄 제어합니다. 개별 뷰에 직접 `visibility`를 설정하면 Group이 나중에 덮어쓸 수 있으니 **둘을 섞어 쓰지 마세요.** (최신 버전에서는 `ConstraintLayout 2.0+`의 동작을 확인하세요)

---

## 6. 비율과 크기 제약

```xml
<!-- 가로 대비 세로 비율 고정 (16:9 썸네일) -->
<ImageView
    android:layout_width="0dp"
    android:layout_height="0dp"
    app:layout_constraintDimensionRatio="16:9"
    app:layout_constraintStart_toStartOf="parent"
    app:layout_constraintEnd_toEndOf="parent" />

<!-- 최소/최대 크기 -->
<TextView
    android:layout_width="0dp"
    app:layout_constraintWidth_min="100dp"
    app:layout_constraintWidth_max="300dp"
    app:layout_constrainedWidth="true" />
```

---

## 7. Flow — 줄바꿈되는 정렬

`LinearLayout` 중첩이나 `FlexboxLayout` 없이 **태그/칩처럼 가로로 배치하다 넘치면 다음 줄**로 넘깁니다.

```xml
<androidx.constraintlayout.helper.widget.Flow
    android:layout_width="0dp"
    android:layout_height="wrap_content"
    app:constraint_referenced_ids="tag1,tag2,tag3,tag4,tag5"
    app:flow_wrapMode="chain"
    app:flow_horizontalGap="8dp"
    app:flow_verticalGap="8dp"
    app:layout_constraintStart_toStartOf="parent"
    app:layout_constraintEnd_toEndOf="parent" />
```

`Flow`가 참조하는 뷰들은 별도 제약 없이 Flow가 배치합니다.

---

## 8. 뷰 가시성과 제약 — goneMargin

연결 대상이 `gone`이 되면 그 제약은 0 거리로 처리됩니다. 이때 간격을 유지하려면 `goneMargin`을 사용합니다.

```xml
<TextView
    android:id="@+id/subtitle"
    app:layout_constraintTop_toBottomOf="@id/title"
    android:layout_marginTop="8dp"
    app:layout_goneMarginTop="0dp" />          <!-- title이 gone이면 마진 0 -->
```

---

## 9. 성능 팁

- 중첩된 `LinearLayout`/`RelativeLayout`은 **ConstraintLayout 하나로 평탄화**하면 측정 횟수가 줄어듭니다
- 단, `match_constraint(0dp)`를 가진 복잡한 체인/Barrier가 많으면 측정이 두 번 이상 일어나므로 **지나치게 복잡한 제약은 피하세요**
- 단순한 세로/가로 나열은 `LinearLayout`이 더 읽기 쉽고 충분합니다

---

## 10. 정리

- **Guideline**: 비율/고정 위치의 기준선
- **Barrier**: 참조 뷰들의 가장 끝에 **동적으로** 생기는 벽 (가변 길이 정렬)
- **Chain**: 여러 뷰를 연결해 `spread`/`packed`와 weight로 분배
- **Group**: 여러 뷰의 가시성 일괄 제어, **Flow**: 줄바꿈 배치
- `dimensionRatio`, `min/max`, `goneMargin`으로 세밀한 크기/간격 제어
- 평평한 구조는 성능에 유리하지만 과도하게 복잡하면 오히려 역효과
