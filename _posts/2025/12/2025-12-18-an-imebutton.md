---
title: (Android/안드로이드) 키보드(IME) 올라올 때 화면이 밀리거나 버튼이 가려질 때
tags: [Android,Kotlin]
style: fill
color: dark
description: (Android/안드로이드) 키보드(IME) 올라올 때 화면이 밀리거나 버튼이 가려질 때 - windowSoftInputMode (SOFT_INPUT_ADJUST_RESIZE / PAN / NOTHING) 완전 정리
---

## ✨ 개요

안드로이드에서 시스템 키보드(IME)가 올라오면 화면이 “밀리거나”, “특정 버튼이 가려지거나”, “레이아웃이 튄다”는 문제가 매우 흔합니다.
- 로그인/회원가입 화면에서 하단 버튼이 키보드에 가림
- BottomSheet/Dialog/WebView에서 키보드 올라올 때 레이아웃이 깨짐
- EditText 포커스 시 화면이 갑자기 확대/축소됨

이런 문제는 대부분 windowSoftInputMode 설정(그리고 IME Insets 대응)으로 해결합니다.

---

## 1. 요약

- ADJUST_RESIZE: 키보드 높이만큼 루트 레이아웃 높이를 줄임 → 스크롤/리스트/폼 화면에 유리
- ADJUST_PAN: 레이아웃 크기는 유지하고 화면을 위로 “팬” 해서 포커스 영역을 보이게 함 → 단순 화면에 유리
- ADJUST_NOTHING: 아무 조정도 하지 않음 → 직접 Insets 처리할 때
- Android 11+ 및 Edge-to-Edge 환경에서는 WindowInsets(ime) 기반 처리로 가는 것이 정석

---

## 2. `windowSoftInputMode`란?

키보드가 표시될 때 Activity의 Window가 어떻게 반응할지 정하는 옵션입니다.

설정 위치는 대표적으로 두 군데입니다.

### 2.1 AndroidManifest.xml에서 설정

```xml
<activity
    android:name=".LoginActivity"
    android:windowSoftInputMode="adjustResize" />
```

### 2.2 코드에서 설정 (런타임)

```kotlin
window.setSoftInputMode(WindowManager.LayoutParams.SOFT_INPUT_ADJUST_RESIZE)
```
> 주의: 일부 상황에서는 런타임 변경이 즉시 적용되지 않거나 화면 재구성이 필요할 수 있습니다. 가능하면 Manifest에서 명확히 설정하는 편이 안정적입니다.

---

## 3. 핵심 옵션 3종: RESIZE / PAN / NOTHING

### 3.1 `SOFT_INPUT_ADJUST_RESIZE` (가장 많이 씀)

- 키보드가 올라오면 Window가 “리사이즈”됩니다.
- 즉, 키보드 높이만큼 콘텐츠 영역의 높이가 줄어듭니다.
- 장점
  - 폼 화면(EditText 여러 개)에서 스크롤로 자연스럽게 처리됨
  - RecyclerView/ScrollView와 궁합이 좋음
- 단점
  - 화면이 “줄었다 늘었다” 하면서 레이아웃이 튀는 느낌이 날 수 있음
  - 특정 레이아웃(특히 Fullscreen/Translucent/BottomSheet/WebView)에서 기대대로 동작하지 않을 수 있음
- 추천 케이스
  - 로그인/회원가입/설문 등 입력 폼
  - 채팅 화면(리스트 + 입력창)

### 3.2 `SOFT_INPUT_ADJUST_PAN`

- 레이아웃 크기는 유지하고, 키보드가 올라올 때 Window의 내용이 위로 “밀려 올라가며” 포커스된 입력창이 보이게 합니다.
- 장점
  - 레이아웃 크기 변화가 없어서 덜 “튀어 보임”
  - 단순한 화면에서 안정적
- 단점
  - ScrollView/RecyclerView 기반 폼에서는 원하는 만큼 스크롤이 안 되거나 하단 버튼이 여전히 가려질 수 있음
  - “포커스된 뷰” 중심으로 움직이므로 레이아웃 전체 제어는 제한적
- 추천 케이스
  - 입력창이 1~2개인 단순 화면
  - 화면 전체가 크게 리레이아웃되면 안 되는 UI

### 3.2 `SOFT_INPUT_ADJUST_NOTHING`

- 키보드가 올라와도 Window는 아무 조정도 하지 않습니다.
- 장점
  - 앱이 직접 WindowInsets(IME)로 제어할 때 최적
  - Edge-to-Edge/Compose/커스텀 인셋 처리에서 의도대로 만들 수 있음
- 단점
  - 별도 처리가 없으면 하단 UI(버튼, 입력창)가 키보드에 가립니다.
- 추천 케이스
  - WindowInsets 기반으로 직접 처리하는 프로젝트
  - Compose + Edge-to-Edge 환경
  - 커스텀 BottomSheet / Dialog에서 직접 키보드 대응을 구현할 때

---

## 4. 옵션 선택 기준(실무 체크리스트)

- Q1. 입력 폼이 길고 스크롤이 필요한가?
  - 예 → adjustResize 우선 고려
  - 아니오 → adjustPan도 충분
- Q2. 키보드가 올라올 때 레이아웃이 “튀는” 것이 싫은가?
  - 예 → adjustPan 또는 adjustNothing + insets
  - 아니오 → adjustResize
- Q3. Edge-to-Edge(상태바/내비게이션 바 투명) 또는 Insets를 직접 처리하는가?
  - 예 → adjustNothing + IME Insets 처리 권장
  - 아니오 → adjustResize가 가장 단순

---

## 5. WindowInsets로 키보드에 맞춰 버튼 올리기 `adjustNothing` + IME 인셋 처리

하단 버튼이 키보드에 가리지 않게 “버튼만” 올리고 싶을 때 가장 좋은 방법입니다.

```kotlin
import androidx.core.view.ViewCompat
import androidx.core.view.WindowInsetsCompat
import androidx.core.view.updatePadding

fun applyImePadding(bottomBar: View) {
    ViewCompat.setOnApplyWindowInsetsListener(bottomBar) { v, insets ->
        val imeInsets = insets.getInsets(WindowInsetsCompat.Type.ime())
        val navInsets = insets.getInsets(WindowInsetsCompat.Type.navigationBars())

        // 키보드가 올라오면 ime bottom이 커지고,
        // 내려가면 0에 가까워짐
        val bottom = maxOf(imeInsets.bottom, navInsets.bottom)

        v.updatePadding(bottom = bottom)
        insets
    }
}

// 하단 버튼 컨테이너(예: bottomBar)가 있다고 가정
applyImePadding(binding.bottomBar)
```