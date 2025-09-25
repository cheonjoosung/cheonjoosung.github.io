---
title: 안드로이드 Glide 사용법 + 고급 심화편
tags: [Android]
style: fill
color: dark
description: 안드로이드 Glide 사용법 + 고급 심화편
---
---

## ✨ 개요

안드로이드에서 이미지 로딩은 앱 성능과 사용자 경험에 직접적인 영향을 미칩니다.  
그중 가장 많이 쓰이는 이미지 라이브러리인 **Glide**는 빠른 속도, 강력한 캐싱, 다양한 커스터마이징 기능을 제공합니다.

이 포스팅에서는 **Glide의 기본 사용법부터 실무 고급 기능까지** 단계별로 정리해 드립니다.

---

## 1. Glide란?

- Bumptech에서 개발한 Android 이미지 로딩 라이브러리
- **비동기 로딩**, **메모리/디스크 캐싱**, **GIF 지원**, **Lifecycle-aware**
- Glide vs Picasso vs Coil 중 가장 커스터마이징이 뛰어남

---

## 2. Gradle 설정

```gradle
dependencies {
    implementation("com.github.bumptech.glide:glide:4.16.0")
    kapt("com.github.bumptech.glide:compiler:4.16.0")
}
``` 
- Kotlin 프로젝트에서는 kapt 추가 필수,
- ProGuard 사용 시 Glide 관련 예외도 추가 필요

---

## 3. 기본 사용법

```kotlin
Glide.with(context)
    .load(url)
    .placeholder(R.drawable.loading)
    .error(R.drawable.error)
    .into(imageView)
/*
.load(R.drawable.image)       // 리소스
.load(File("/path/image.jpg")) // 파일
.load(Uri.parse(...))         // Uri
 */
```

---

## 4. 실무에서 많이 쓰는 옵션들

- 크기 지정

```kotlin
.override(300, 300) // 픽셀 단위 리사이즈
```

- centerCrop, fitCenter, circleCrop

```kotlin
.centerCrop()       // 꽉 차게 자름
    .fitCenter()        // 이미지 전체 보이게
    .circleCrop()       // 동그란 썸네일
```

- 캐싱 전략

```kotlin
.diskCacheStrategy(DiskCacheStrategy.ALL) // 원본+리사이즈 이미지 캐시
    .skipMemoryCache(true) // 메모리 캐시 무시
```


--- 

## 5. 고급 사용법

- 커스텀 Target

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

- 썸네일 

```kotlin
Glide.with(context)
    .load(highQualityUrl)
    .thumbnail(0.1f) // 저해상도 빠른 썸네일 먼저 보여줌
    .into(imageView)
```

- GIF 로딩

```kotlin
Glide.with(context)
    .asGif()
    .load("https://example.com/animated.gif")
    .into(imageView)
```

- GlideModule로 전역 설정 커스터마이징

```kotlin
@GlideModule
class MyAppGlideModule : AppGlideModule() {
    override fun applyOptions(context: Context, builder: GlideBuilder) {
        builder.setDefaultRequestOptions(
            RequestOptions()
                .format(DecodeFormat.PREFER_RGB_565) // 메모리 절약
                .diskCacheStrategy(DiskCacheStrategy.AUTOMATIC)
        )
    }
}
```

---

## 6. 자주 사용하는 조합

| 조합                   | 설명                           |
| -------------------- | ---------------------------- |
| Glide + RecyclerView | 썸네일 리스트, 프로필 목록 등            |
| Glide + ViewPager2   | 배너 슬라이더                      |
| Glide + Shimmer      | 로딩 중 효과와 함께 사용               |
| Glide + Coroutines   | bitmap 처리 후 imageView에 직접 할당 |

---

## 7. 결론

- 빠르고 유연한 Android 이미지 로딩 라이브러리
- GIF, 썸네일, 캐싱 전략까지 모두 지원  
- Custom Target, GlideModule 등 실무 기능 풍부