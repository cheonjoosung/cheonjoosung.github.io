---
title: Android Glide 사용법 - 이미지 로딩 라이브러리  
tags: [Android]
style: fill
color: dark
description: Android Glide 사용법 - 이미지 로딩 라이브러리
---

## ✨ 개요

안드로이드 앱에서 이미지 로딩은 **UI 성능과 UX에 직접적인 영향을 주는 요소**입니다.  
Glide는 Google이 공식 채택한 이미지 로딩 라이브러리로, 빠르고 유연하며 확장성이 뛰어납니다.

이 포스팅에서는 Glide의 기본 사용법부터 고급 옵션, 실무에서 자주 쓰는 팁까지 정리합니다.

---

## 1. Bumptech에서 개발한 Android 이미지 로딩 라이브러리

- **비동기 로딩**, **메모리/디스크 캐시**, **GIF 지원**
- View lifecycle을 인식하여 **자동으로 로딩 중단/해제**

> ✅ Glide는 [Google 공식 문서](https://developer.android.com/topic/performance/graphics/load-bitmap)에서도 추천되는 이미지 로더입니다.


---

## 2. Glide 설정 방법

### Gradle 추가

```gradle
dependencies {
    implementation 'com.github.bumptech.glide:glide:4.16.0'
    kapt 'com.github.bumptech.glide:compiler:4.16.0'
}
```
- Kotlin 프로젝트는 kapt 필수
- ProGuard 사용 시 Glide 예외 처리도 필요

---

## 3. Glide 기본 사용법

#### 기본 사용법

```kotlin
Glide.with(context)
    .load("https://example.com/image.jpg")
    .into(imageView)

/*
.load(R.drawable.local_image)    // 리소스
.load(File("/path/image.jpg"))   // 파일
.load(Uri.parse("..."))          // URI
.load(url)                       // URL
 */
```

#### 썸네일

```kotlin
Glide.with(context)
    .load(고해상도 이미지)
    .thumbnail(0.1f)  // 저해상도 10% 먼저 보여주기
    .into(imageView)
```

#### 비트맵으로 다루기

```kotlin
Glide.with(context)
    .asBitmap()
    .load(url)
    .into(object : CustomTarget<Bitmap>() {
        override fun onResourceReady(resource: Bitmap, transition: Transition<in Bitmap>?) {
            imageView.setImageBitmap(resource)
        }

        override fun onLoadCleared(placeholder: Drawable?) {}
    })
```

---

## 4. 주요 옵션

#### 기본 옵션

```kotlin
Glide.with(context)
    .load(url)
    .placeholder(R.drawable.loading)
    .error(R.drawable.error)
    .override(300, 300)           // 리사이즈
    .centerCrop()                 // 꽉 차게 자르기
    .circleCrop()                 // 동그랗게 자르기
    .into(imageView)

/*
.diskCacheStrategy(DiskCacheStrategy.ALL)     // 원본 + 리사이즈 모두 캐시
.skipMemoryCache(true)                        // 메모리 캐시 비활성화
 */
```

#### 전역 설정

```kotlin
@GlideModule
class MyAppGlideModule : AppGlideModule() {
    override fun applyOptions(context: Context, builder: GlideBuilder) {
        builder.setDefaultRequestOptions(
            RequestOptions()
                .diskCacheStrategy(DiskCacheStrategy.AUTOMATIC)
                .format(DecodeFormat.PREFER_RGB_565) // 메모리 절약
        )
    }
}
```

---

## 5. 주의사항

| 항목            | 주의 내용                                       |
| ------------- | ------------------------------------------- |
| RecyclerView  | ViewHolder 재활용 주의 (Glide는 자동 취소 처리함)        |
| Glide 객체      | 반드시 `.with(context)`에 올바른 context 사용        |
| Fragment 사용 시 | `Glide.with(fragment)` 형태 권장                |
| 메모리 최적화       | `.override()`, `.format(PREFER_RGB_565)` 활용 |

---

## 6. 결론

Glide는 다음과 같은 상황에서 최고의 선택입니다:
- 복잡한 이미지 처리 (GIF, WebP, 썸네일 등)
- RecyclerView, ViewPager2와의 연동
- 다양한 커스터마이징이 필요한 경우

빠르고, 메모리 효율적이며, 안정적인 이미지 로딩이 필요하다면 Glide는 여전히 안드로이드 앱 개발의 표준입니다.