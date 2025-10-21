---
title: Android/안드로이드 실무에서 자주 쓰는 확장함수 모음
tags: [Kotlin]
style: fill
color: dark
description: Android/안드로이드 실무에서 자주 쓰는 확장함수 모음 – 한 번 붙여두고 계속 쓰는 유틸
---

## ✨ 개요

안드로이드 개발에서 매번 반복하는 뷰 토글, 클릭 디바운스, 키보드 열고 닫기, 안전한 네비게이션 등은 **확장함수**로 정리하면 생산성이 확 올라갑니다.  
아래 스니펫은 **View/UI · Activity/Fragment · 네비/인텐트 · 코루틴 · RecyclerView · WebView · 미디어/저장소**까지 넓게 커버합니다.

> 💡 권장: 파일 하나로 묶어 `common/ui/Ext.kt` 같은 위치에 두고 전역 재사용

---

## 1 View / UI

```kotlin
// 가시성 토글
var View.isVisibleEx: Boolean
    get() = visibility == View.VISIBLE
    set(v) { visibility = if (v) View.VISIBLE else View.GONE }

fun View.show() { isVisibleEx = true }
fun View.hide() { isVisibleEx = false }
fun View.invisible() { visibility = View.INVISIBLE }

// 클릭 디바운스(중복탭 방지)
fun View.setOnSafeClick(intervalMs: Long = 500, block: (View) -> Unit) {
    var last = 0L
    setOnClickListener {
        val now = System.currentTimeMillis()
        if (now - last >= intervalMs) { last = now; block(it) }
    }
}

// dp/px 변환
val Int.dp: Int get() = (this * Resources.getSystem().displayMetrics.density).toInt()
val Float.dp: Int get() = (this * Resources.getSystem().displayMetrics.density).toInt()
val Int.px: Int get() = (this / Resources.getSystem().displayMetrics.density).toInt()

// 소프트키보드
fun View.showKeyboard() {
    requestFocus()
    val imm = context.getSystemService(InputMethodManager::class.java)
    imm?.showSoftInput(this, InputMethodManager.SHOW_IMPLICIT)
}
fun View.hideKeyboard() {
    val imm = context.getSystemService(InputMethodManager::class.java)
    imm?.hideSoftInputFromWindow(windowToken, 0)
}

// 스낵바 원라인
fun View.snack(msg: CharSequence, duration: Int = Snackbar.LENGTH_SHORT) =
    Snackbar.make(this, msg, duration).show()

// 라운드 코너 배경(즉석)
fun View.roundBg(radiusDp: Int, @ColorInt color: Int) {
    background = GradientDrawable().apply {
        cornerRadius = radiusDp.dp.toFloat()
        setColor(color)
    }
}
```

---

## 2 Context / Activity / Fragment

```kotlin
// 토스트
fun Context.toast(msg: CharSequence) = Toast.makeText(this, msg, Toast.LENGTH_SHORT).show()

// 안전한 startActivity
fun Context.startActivitySafe(intent: Intent, onFail: (() -> Unit)? = null) {
    if (intent.resolveActivity(packageManager) != null) startActivity(intent) else onFail?.invoke()
}

// 앱 설정 화면 열기
fun Context.openAppSettings() = startActivity(
    Intent(Settings.ACTION_APPLICATION_DETAILS_SETTINGS).setData(Uri.parse("package:$packageName"))
)

// 상태바/내비바 숨기기 (Immersive)
fun Activity.setImmersive(enabled: Boolean) {
    WindowCompat.setDecorFitsSystemWindows(window, !enabled)
    WindowCompat.getInsetsController(window, window.decorView)?.apply {
        systemBarsBehavior = WindowInsetsController.BEHAVIOR_SHOW_TRANSIENT_BARS_BY_SWIPE
        if (enabled) hide(WindowInsets.Type.systemBars()) else show(WindowInsets.Type.systemBars())
    }
}

// Fragment 트랜잭션 + runOnCommit
fun FragmentManager.replaceCommit(
    @IdRes containerId: Int,
    fragment: Fragment,
    onCommitted: (() -> Unit)? = null
) {
    beginTransaction()
        .replace(containerId, fragment)
        .runOnCommit { onCommitted?.invoke() }
        .commit()
}

// Fragment 안전한 viewBinding (메모리릭 방지)
inline fun <T> Fragment.viewBinding(crossinline bind: (View) -> T) =
    object : Lazy<T> {
        private var cache: T? = null
        override val value: T
            get() = cache ?: bind(requireView()).also {
                viewLifecycleOwnerLiveData.observe(this@viewBinding) { owner ->
                    owner.lifecycle.addObserver(object : DefaultLifecycleObserver {
                        override fun onDestroy(owner: LifecycleOwner) { cache = null }
                    })
                }
                cache = it
            }
        override fun isInitialized() = cache != null
    }
```

---

## 3 인텐트/공유/권한

```kotlin
// 텍스트 공유
fun Context.shareText(text: String, title: String = "공유하기") {
    ShareCompat.IntentBuilder(this).setType("text/plain").setText(text).setChooserTitle(title).startChooser()
}

// 이미지 공유 (content:// 만 권장)
fun Context.shareImage(uri: Uri, title: String = "이미지 공유") {
    ShareCompat.IntentBuilder(this).setType("image/*").setStream(uri).setChooserTitle(title).startChooser()
}

// 런타임 권한 한 번에 (ActivityResult API 필요 위치에서 호출)
fun Fragment.requestPermissions(
    vararg perms: String,
    onResult: (Map<String, Boolean>) -> Unit
) {
    registerForActivityResult(ActivityResultContracts.RequestMultiplePermissions(), onResult)
        .launch(perms)
}
```

---

## 4 코루틴/Flow

```kotlin
// ViewModel IO 실행
fun ViewModel.launchIO(block: suspend CoroutineScope.() -> Unit) =
    viewModelScope.launch(Dispatchers.IO, block = block)

// Lifecycle 안전 수집
fun <T> LifecycleOwner.collectLatestStarted(flow: Flow<T>, block: suspend (T) -> Unit) {
    lifecycleScope.launch {
        lifecycle.repeatOnLifecycle(Lifecycle.State.STARTED) {
            flow.collectLatest(block)
        }
    }
}

// withTimeoutOrNull 래퍼
suspend fun <T> withTimeoutOrNullMs(ms: Long, block: suspend () -> T) =
    withTimeoutOrNull(ms) { block() }
```

---

## 5 RecyclerView

```kotlin
// 간격 데코레이션
class SpacingDecoration(@Px private val h: Int = 0, @Px private val v: Int = 0) : RecyclerView.ItemDecoration() {
    override fun getItemOffsets(out: Rect, view: View, parent: RecyclerView, state: RecyclerView.State) {
        out.set(v, h, v, h)
    }
}

// 끝에 닿으면 더 불러오기
fun RecyclerView.onReachEnd(threshold: Int = 3, onLoadMore: () -> Unit) {
    addOnScrollListener(object : RecyclerView.OnScrollListener() {
        override fun onScrolled(rv: RecyclerView, dx: Int, dy: Int) {
            val lm = rv.layoutManager as? LinearLayoutManager ?: return
            val last = lm.findLastVisibleItemPosition()
            val total = rv.adapter?.itemCount ?: return
            if (total > 0 && last >= total - 1 - threshold) onLoadMore()
        }
    })
}
```

---

## 6 이미지/비트맵/저장소

```kotlin
// Bitmap → 갤러리 저장 (Android 10+)
fun Context.saveBitmapToGallery(
    bmp: Bitmap,
    displayName: String = "IMG_${System.currentTimeMillis()}.jpg",
    mime: String = "image/jpeg",
    quality: Int = 90
): Uri? {
    val values = ContentValues().apply {
        put(MediaStore.Images.Media.DISPLAY_NAME, displayName)
        put(MediaStore.Images.Media.MIME_TYPE, mime)
        put(MediaStore.Images.Media.RELATIVE_PATH, "Pictures/MyApp")
    }
    val uri = contentResolver.insert(MediaStore.Images.Media.EXTERNAL_CONTENT_URI, values) ?: return null
    contentResolver.openOutputStream(uri)?.use { out -> bmp.compress(Bitmap.CompressFormat.JPEG, quality, out) }
    return uri
}

// View → Bitmap 스냅샷
fun View.snapshot(): Bitmap =
    Bitmap.createBitmap(width, height, Bitmap.Config.ARGB_8888).also { draw(Canvas(it)) }
```

---

## 7. 결론

- “매번 쓰는 것”부터 확장함수로 올려두면 중복/보일러플레이트가 크게 줄고, 팀 내 코딩 컨벤션을 통일하기 쉬워집니다.
- 필요 시 위 스니펫을 모듈 공통 라이브러리로 뽑아 배포해 보세요. (예: :core-ui, :core-ktx)