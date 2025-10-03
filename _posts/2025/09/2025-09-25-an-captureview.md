---
title: Android 화면 캡쳐 방법 그리고 저장하기- View, 전체화면, RecyclerView
tags: [Android]
style: fill
color: dark
description: Android 화면 캡쳐 방법 그리고 저장하기- View, 전체화면, RecyclerView
---

## ✨ 개요

앱에서 화면을 캡쳐하는 방법은 무엇을 찍고 싶은지에 따라 달라집니다.

- 특정 View만 캡쳐 → Canvas.draw()
- SurfaceView/Map/Video 포함 캡처? → PixelCopy
- 상태바/타앱까지 포함한 전체 화면? → MediaProjection

---

## 1. 특정 View 캡처

```kotlin
fun captureView(view: View, config: Bitmap.Config = Bitmap.Config.ARGB_8888): Bitmap {
    val bmp = Bitmap.createBitmap(view.width, view.height, config)
    val canvas = Canvas(bmp)
    view.draw(canvas)
    return bmp
}
```
- 아직 레이아웃 크기가 정해지지 않았다면(width/height == 0) view.doOnPreDraw { ... } 후 캡처
- RecyclerView 전체(보이지 않는 아이템 포함) 캡처는 아래 4) 참고

---

## 2. Activity 루트 뷰 (앱 화면 스냅샷)

```kotlin
fun captureActivity(activity: Activity): Bitmap {
    val root = activity.window.decorView.rootView
    val bmp = Bitmap.createBitmap(root.width, root.height, Bitmap.Config.ARGB_8888)
    val canvas = Canvas(bmp)
    root.draw(canvas)
    return bmp
}
```
- FLAG_SECURE가 켜진 창은 캡처 불가(보안화면).

---

## 3. RecyclerView "전체 길이" 캡쳐 (보이는 영역을 넘어 전체 스크린샷)

```kotlin
fun captureRecyclerViewAll(rv: RecyclerView): Bitmap {
    val adapter = rv.adapter ?: error("No adapter")
    val widthSpec = View.MeasureSpec.makeMeasureSpec(rv.width, View.MeasureSpec.EXACTLY)
    var totalHeight = 0

    // 높이 계산(주의: 대용량이면 메모리 크래시 가능)
    for (i in 0 until adapter.itemCount) {
        val vh = adapter.createViewHolder(rv, adapter.getItemViewType(i))
        adapter.onBindViewHolder(vh, i)
        vh.itemView.measure(widthSpec, View.MeasureSpec.makeMeasureSpec(0, View.MeasureSpec.UNSPECIFIED))
        totalHeight += vh.itemView.measuredHeight
    }

    val bmp = Bitmap.createBitmap(rv.width, totalHeight, Bitmap.Config.ARGB_8888)
    val canvas = Canvas(bmp)
    var y = 0
    for (i in 0 until adapter.itemCount) {
        val vh = adapter.createViewHolder(rv, adapter.getItemViewType(i))
        adapter.onBindViewHolder(vh, i)
        vh.itemView.measure(widthSpec, View.MeasureSpec.makeMeasureSpec(0, View.MeasureSpec.UNSPECIFIED))
        vh.itemView.layout(0, 0, vh.itemView.measuredWidth, vh.itemView.measuredHeight)
        canvas.save()
        canvas.translate(0f, y.toFloat())
        vh.itemView.draw(canvas)
        canvas.restore()
        y += vh.itemView.measuredHeight
    }
    return bmp
}
```
- 아이템 수가 많으면 아주 큰 비트맵이 생성되어 OOM 위험. 섹션 별로 나눠서 캡처하거나 PDF로 내보내는 전략 고려.

---

## 4. 캡쳐한 Bitmap (갤러리 노출) - MediaStore(Android 10+)

#### 기본 옵션

```kotlin
fun saveToGallery(context: Context, bmp: Bitmap, displayName: String = "capture_${System.currentTimeMillis()}.jpg") {
    val values = ContentValues().apply {
        put(MediaStore.Images.Media.DISPLAY_NAME, displayName)
        put(MediaStore.Images.Media.MIME_TYPE, "image/jpeg")
        put(MediaStore.Images.Media.RELATIVE_PATH, "Pictures/Screenshots")
    }
    val uri = context.contentResolver.insert(MediaStore.Images.Media.EXTERNAL_CONTENT_URI, values) ?: return
    context.contentResolver.openOutputStream(uri)?.use { out ->
        bmp.compress(Bitmap.CompressFormat.JPEG, 90, out)
        out.flush()
    }
}
```
- Android 10(Q)+에서는 별도의 저장 권한 없이 MediaStore로 저장 가능.
- PNG/WebP로 저장 시 MIME_TYPE/포맷만 바꿔주면 됩니다.

---

## 5. 주의사항

- 비어 있는 캡처: Surface 기반(지도/영상)은 draw() 대신 PixelCopy 사용.
- OOM(메모리 부족): 큰 뷰/리스트 전체 캡처는 분할 저장 또는 해상도 축소.
- 보안화면 캡처 불가: FLAG_SECURE가 켜진 창은 정책상 차단.
- 저장 후 갤러리에 안 보여요: MediaStore를 올바른 경로로, 혹은 스캔 대기. 위 예시는 자동 스캔 대상 경로.

---

## 6. 결론

- 자기 앱 내부 뷰만이면 view.draw(canvas)가 가장 간단.
- 지도/영상 포함이면 PixelCopy로 안전하게.
- **전체 화면(타앱 포함)**은 MediaProjection + 사용자 동의.
- 결과물은 MediaStore로 저장해 갤러리에 노출.