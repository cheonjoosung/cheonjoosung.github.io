---
title: Android/안드로이드 ImageView 확장함수 모음
tags: [Kotlin]
style: fill
color: dark
description: Android/안드로이드 ImageView 확장함수 모음 – 로딩/플레이스홀더/SVG/라운드/필터/팔레트 추출
---

## ✨ 개요

- 대상: `ImageView`
- 핵심: URL/URI/리소스 로딩, 플레이스홀더/에러, 크로스페이드, 원형/라운드, 틴트, 흑백/블러, 팔레트 색 추출, 스퀘어/비율 맞춤, 스냅샷
- 선택: **Coil** 또는 **Glide** 중 하나(또는 둘 다)


---

## 1 `의존성`

```kotlin
implementation "io.coil-kt:coil:2.7.0"
implementation "io.coil-kt:coil-base:2.7.0"
// SVG 필요 시
implementation "io.coil-kt:coil-svg:2.7.0"

implementation "com.github.bumptech.glide:glide:4.16.0"
kapt "com.github.bumptech.glide:compiler:4.16.0"
// SVG는 Glide 단독 지원 X → transcode/3rd-party 필요. SVG면 Coil 권장
```

---

## 2 사용 예

```kotlin
import android.graphics.*
        import android.os.Build
        import android.view.View
        import android.widget.ImageView
        import androidx.annotation.ColorInt
        import androidx.annotation.DrawableRes
        import androidx.core.graphics.drawable.toBitmap
        import androidx.palette.graphics.Palette

/** 이미지 틴트 적용 (드로어블 색 입히기) */
fun ImageView.setTint(@ColorInt color: Int) = apply { imageTintList = android.content.res.ColorStateList.valueOf(color) }

/** 원형 클리핑 (하드 클리핑: 이미지를 동그랗게 잘라서 보여줌) */
fun ImageView.clipCircle(enable: Boolean = true) {
    outlineProvider = if (enable) object : ViewOutlineProvider() {
        override fun getOutline(view: View, outline: Outline) {
            val d = view.width.coerceAtMost(view.height)
            outline.setOval(0, 0, d, d)
        }
    } else null
    clipToOutline = enable
}

/** 라운드 코너 클리핑 */
fun ImageView.clipRounded(radiusPx: Int) {
    outlineProvider = object : ViewOutlineProvider() {
        override fun getOutline(view: View, outline: Outline) {
            outline.setRoundRect(0, 0, view.width, view.height, radiusPx.toFloat())
        }
    }
    clipToOutline = true
}

/** 정사각형으로 강제 맞춤(레이아웃 단계에서 width=height) */
fun ImageView.enforceSquare() {
    addOnLayoutChangeListener(object : View.OnLayoutChangeListener {
        override fun onLayoutChange(
            v: View, l: Int, t: Int, r: Int, b: Int, ol: Int, ot: Int, orr: Int, ob: Int
        ) {
            val size = (r - l).coerceAtMost(b - t)
            layoutParams.width = size
            layoutParams.height = size
            requestLayout()
            removeOnLayoutChangeListener(this)
        }
    })
}

/** 현재 표시 중인 비트맵을 얻어 스냅샷(없으면 null) */
fun ImageView.snapshotBitmapOrNull(): Bitmap? =
    drawable?.toBitmap(width.coerceAtLeast(1), height.coerceAtLeast(1))

/** 흑백(그레이스케일) 필터 토글 */
fun ImageView.setGrayscale(enable: Boolean) {
    colorFilter = if (enable) {
        val m = ColorMatrix().apply { setSaturation(0f) }
        ColorMatrixColorFilter(m)
    } else null
}

/** 블러 적용 (API 31+ RenderEffect, 하위는 간단 처리 생략) */
fun ImageView.setBlur(radius: Float) {
    if (Build.VERSION.SDK_INT >= 31) {
        setRenderEffect(RenderEffect.createBlurEffect(radius, radius, Shader.TileMode.CLAMP))
    }
}

/** 팔레트(대표색) 추출 – 콜백으로 dominant 색 전달 */
fun ImageView.extractDominantColor(onReady: (@ColorInt Int) -> Unit) {
    val bmp = snapshotBitmapOrNull() ?: return
    Palette.from(bmp).clearFilters().generate { p ->
        val c = p?.getDominantColor(Color.parseColor("#CCCCCC")) ?: Color.GRAY
        onReady(c)
    }
}
```

---

## 3 Coil 전용 확장 (권장: 가볍고 SVG/애니GIF 수월)

```kotlin
package com.example.imgext

import android.net.Uri
import android.widget.ImageView
import androidx.annotation.DrawableRes
import coil.ImageLoader
import coil.decode.SvgDecoder
import coil.load
import coil.request.CachePolicy
import coil.size.Scale

/** URL 로딩(플레이스홀더/에러/크로스페이드/센터크롭) */
fun ImageView.loadUrlCoil(
    url: String?,
    @DrawableRes placeholder: Int? = null,
    @DrawableRes error: Int? = null,
    crossfade: Boolean = true,
    centerCrop: Boolean = true
) {
    load(url) {
        placeholder?.let { placeholder(it) }
        error?.let { error(it) }
        if (centerCrop) scale(Scale.FILL) else scale(Scale.FIT)
        crossfade(crossfade)
        memoryCachePolicy(CachePolicy.ENABLED)
        diskCachePolicy(CachePolicy.ENABLED)
        allowHardware(true)
    }
}

/** URI/파일/리소스 모두 대응 */
fun ImageView.loadAnyCoil(
    data: Any?,
    @DrawableRes ph: Int? = null,
    @DrawableRes err: Int? = null
) = loadUrlCoil(data?.toString(), ph, err)

/** SVG 전용 (coil-svg 필요) */
fun ImageView.loadSvgCoil(
    url: String,
    @DrawableRes placeholder: Int? = null,
    @DrawableRes error: Int? = null
) {
    val loader = ImageLoader.Builder(context)
        .components { add(SvgDecoder.Factory()) }
        .build()
    this.load(url, loader) {
        placeholder?.let { placeholder(it) }
        error?.let { error(it) }
        crossfade(true)
    }
}

/** 원형 크롭 + 테두리(오버레이) – 클리핑 + 틴트 조합 */
fun ImageView.loadCircleCoil(
    url: String?,
    @DrawableRes ph: Int? = null,
    @DrawableRes err: Int? = null
) {
    clipCircle(true)
    loadUrlCoil(url, ph, err, crossfade = true, centerCrop = true)
}
```

---

## 4 Glide 전용 확장 (안정적 캐시/변환)

```kotlin
package com.example.imgext

import android.graphics.drawable.Drawable
import android.widget.ImageView
import androidx.annotation.DrawableRes
import com.bumptech.glide.Glide
import com.bumptech.glide.load.MultiTransformation
import com.bumptech.glide.load.engine.DiskCacheStrategy
import com.bumptech.glide.load.resource.bitmap.CenterCrop
import com.bumptech.glide.load.resource.bitmap.CircleCrop
import com.bumptech.glide.load.resource.bitmap.RoundedCorners
import com.bumptech.glide.request.RequestListener
import com.bumptech.glide.request.target.Target

/** URL 로딩 – 플레이스홀더/에러/크로스페이드 */
fun ImageView.loadUrlGlide(
    url: String?,
    @DrawableRes placeholder: Int? = null,
    @DrawableRes error: Int? = null,
    centerCrop: Boolean = true,
    skipMemory: Boolean = false,
    diskCache: DiskCacheStrategy = DiskCacheStrategy.AUTOMATIC,
    listener: RequestListener<Drawable>? = null
) {
    val req = Glide.with(this).load(url)
    placeholder?.let { req.placeholder(it) }
    error?.let { req.error(it) }
    if (centerCrop) req.transform(CenterCrop())
    req.diskCacheStrategy(diskCache)
        .skipMemoryCache(skipMemory)
        .listener(listener)
        .into(this)
}

/** 원형 이미지 */
fun ImageView.loadCircleGlide(
    url: String?,
    @DrawableRes ph: Int? = null,
    @DrawableRes err: Int? = null
) {
    Glide.with(this).load(url)
        .apply {
            ph?.let { placeholder(it) }
            err?.let { error(it) }
        }
        .transform(CircleCrop())
        .into(this)
}

/** 라운드 코너(px) */
fun ImageView.loadRoundedGlide(
    url: String?,
    radiusPx: Int,
    @DrawableRes ph: Int? = null,
    @DrawableRes err: Int? = null
) {
    Glide.with(this).load(url)
        .apply {
            ph?.let { placeholder(it) }
            err?.let { error(it) }
        }
        .transform(MultiTransformation(CenterCrop(), RoundedCorners(radiusPx)))
        .into(this)
}
```

---

## 5 사용 예

```kotlin
// Coil
imageView.loadUrlCoil(
    url = "https://picsum.photos/600/400",
    placeholder = R.drawable.ph_img,
    error = R.drawable.ic_broken
)
iconView.setTint(0xFF2962FF.toInt())
avatar.loadCircleCoil(user.profileUrl)

// SVG (Coil)
banner.loadSvgCoil("https://example.com/img.svg", ph = R.drawable.ph_banner)

// Glide
thumb.loadRoundedGlide(photo.url, radiusPx = resources.getDimensionPixelSize(R.dimen.radius_12))
badge.loadCircleGlide(photo.url, ph = R.drawable.ic_user)

// 필터/팔레트
imageView.setGrayscale(true)
imageView.extractDominantColor { color -> toolbar.setBackgroundColor(color) }
```

---

## 6. 실무 팁 & 체크리스트

- 메모리/하드웨어
  + 리스트/그리드: centerCrop + 크기 지정(Glide의 override(w, h) 또는 Coil의 size())로 과한 비트맵 방지
  + allowHardware(true)(Coil) / Glide 기본 하드웨어 가속은 팔레트/픽셀 접근 시 소프트로 바꿔야 할 수 있어요.
- SVG는 Coil 추천: Glide 단독으로는 추가 설정이 필요합니다.
- 원형/라운드
  + “모양만” 필요하면 clipCircle/clipRounded(프레임워크)로 깔끔.
  + “이미지 자체 변환”이 필요하면 라이브러리 변환(CircleCrop/RoundedCorners)로
- 에러/플레이스홀더 디자인 통일: 리소스 세트를 모듈로 묶어 일관성 유지.
- 팔레트 추출은 비용 有: 썸네일 크기에서 추출하거나 비동기로.