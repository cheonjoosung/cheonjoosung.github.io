---
title: Android 성능 최적화 - 메모리 최적화 핵심 전략 총정리
tags: [Android]
style: fill
color: dark
description: Android 성능 최적화 - 메모리 최적화 핵심 전략 총정리
---
---

## ✨ 개요

안드로이드 앱을 개발하다 보면, **메모리 사용량이 많아 앱이 느려지거나 강제 종료되는** 현상을 경험하게 됩니다.  
특히 이미지 처리, 리스트 뷰, 배경 작업 등을 다룰 때는 메모리 누수나 과도한 할당을 피하는 것이 매우 중요합니다.

이 포스팅에서는 **안드로이드 메모리 최적화에 중점을 두고**, 실전에서 꼭 챙겨야 할 포인트를 정리합니다.

---

## 1. 메모리 누수(Leak)를 방지하라

#### 🔥 가장 흔한 원인: `Context`를 잘못 참조할 때

```kotlin
class MyManager(context: Context) {
    private val context = context // ❌ Activity context 저장 시 누수 위험
}

// `ApplicationContext` 사용
val context = context.applicationContext
```
> 📌 Activity나 Fragment를 참조하는 객체는 생명주기와 함께 해제되지 않으면 누수 발생

---

## 2. 불필요한 리소스는 해제하라

#### 🎯예시 viewBinding, Handler, Animation 등

```kotlin
override fun onDestroyView() {
    binding = null // ✅
}

override fun onDestroy() {
    handler.removeCallbacksAndMessages(null) // ✅
}
``` 

---

## 3. 이미지 로딩 최적화

#### ✅ Glide/Picasso 사용 + 리사이징 옵션 지정

```kotlin
Glide.with(context)
    .load(url)
    .override(300, 300) // 적절한 사이즈 지정
    .into(imageView)
```
- `into()` 대상은 재사용 가능한 ViewHolder로 제한
- 큰 이미지 리사이징 필수 (inSampleSize 또는 .override() 사용)
- 캐싱 전략 고려

---

## 4. 메모리 릭 디버깅 도구 사용

| 도구                             | 설명                   |
| ------------------------------ | -------------------- |
| **LeakCanary**                 | 메모리 누수 자동 감지 라이브러리   |
| **Android Profiler**           | 메모리 사용량, 할당 객체 추적 가능 |
| **MAT (Memory Analyzer Tool)** | 힙 덤프 분석 도구 (디테일 확인용) |


--- 

## 5. RecyclerView / ViewPager 최적화

- ViewHolder 재사용하지 않고 매번 inflate → ❌ 메모리 낭비
- 이미지 캐싱 안 함 → ❌ 스크롤 시 OutOfMemoryError 발생

✅ ViewHolder 패턴 준수 + 이미지 라이브러리 캐싱 사용

---

## 6. 결론

안드로이드 앱의 성능은 메모리 최적화에서 시작됩니다.
앱이 잘 동작하더라도 백그라운드에서 누적되는 메모리 문제가 있다면 결국 크래시로 이어질 수 있습니다.

- 메모리는 보이지 않는 성능 문제
- 작은 누수가 큰 문제로 이어짐
- 개발 단계에서부터 습관처럼 관리하는 것이 중요

지금 당장 LeakCanary를 적용해보시고, 앱의 메모리 흐름을 시각화해보세요.
작은 관심이 앱 품질을 결정합니다. 💪