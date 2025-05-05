---
title: Android ConstraintLayout 비율 설정하기 (layout_constraintDimensionRatio 사용법)
tags: [Android]
style: fill
color: dark
description: ConstraintLayout의 layout_constraintDimensionRatio 속성을 이용해 뷰의 가로 세로 비율을 쉽게 설정하는 방법을 예제와 함께 알아봅니다.
---

## ✨ 개요

Android의 `ConstraintLayout`은 강력한 제약 기반 레이아웃 시스템을 제공합니다.  
그중 `layout_constraintDimensionRatio` 속성은 뷰의 **가로:세로 비율을 설정**하는 데 사용되며, 이미지, 카드뷰, 박스형 UI 등에 자주 활용됩니다.

---

## 1. ✅ 속성 구조

### 1.1 기본 사용법

```xml
app:layout_constraintDimensionRatio="1:1" 
app:layout_constraintDimensionRatio="W,1:2" 
app:layout_constraintDimensionRatio="H:3:1" 
```
- "W,H"는 가로:세로 비율
- "H,2:1" → 세로 기준, 가로가 2배
- "W,3:4" → 가로 기준, 세로가 4/3 배
- "16:9" → 기준 없이 시스템이 판단

---

## 2. ✅ 예시

### 2.1 📌 정사각형 박스 만들기

```xml
<androidx.constraintlayout.widget.ConstraintLayout
    android:layout_width="match_parent"
    android:layout_height="wrap_content">

    <View
        android:id="@+id/square"
        android:layout_width="0dp"
        android:layout_height="0dp"
        android:background="#FF8888"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintTop_toTopOf="parent"
        app:layout_constraintDimensionRatio="1:1" />
        
</androidx.constraintlayout.widget.ConstraintLayout>
```

- 0dp로 설정하면 제약 조건을 기반으로 크기를 계산
- 비율이 1:1이면 정사각형이 됩니다

### 2.2. 📌 방향 지정 예시 (W, H)

#### 2.2.1 가로 기준으로 세로 2배 (W,1:2)

```xml
<View
    android:id="@+id/square"
    android:layout_width="0dp"
    android:layout_height="wrap_content"
    android:background="#FF8888"
    app:layout_constraintStart_toStartOf="parent"
    app:layout_constraintEnd_toEndOf="parent"
    app:layout_constraintTop_toTopOf="parent"
    app:layout_constraintDimensionRatio="W,1:2" />
```

- 가로를 기준으로 세로가 2배
- 주로 이미지 뷰에서 세로 긴 디자인에 사용
- layout_width = 0dp 설정

#### 2.2.2 세로 기준으로 가로 3배 (H,3:1)

```xml
<View
    android:id="@+id/square"
    android:layout_width="wrap_content"
    android:layout_height="0dp"
    android:background="#FF8888"
    app:layout_constraintStart_toStartOf="parent"
    app:layout_constraintEnd_toEndOf="parent"
    app:layout_constraintTop_toTopOf="parent"
    app:layout_constraintDimensionRatio="H,3:1" />
```

- 세로를 기준으로 가로가 3배
- 슬라이더, 썸네일 등 가로 중심 UI에 적합
- layout_height = 0dp 설정

---

## 3.✅ 주의사항

- layout_width or layout_height 사용 시 반드시 0dp 이어야 함
- wrap_content or match_parent 사용 불가

--- 

## 4.🧠 결론

- layout_constraintDimensionRatio는 미디어, 카드, 프로필 등 다양한 UI에 최적
- 비율 기반 UI는 반응형 대응에 매우 유리
- 반드시 가로나 세로 중 하나는 0dp로 설정해야 적용됨