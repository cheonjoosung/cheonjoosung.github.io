---
title: Android RecyclerView로 이미지 캐러셀(슬라이더) 만들기 - 자동 스크롤 포함
tags: [ Android ]
style: fill
color: dark
description: Android에서 RecyclerView와 SnapHelper, Handler를 이용해 가볍고 커스터마이징 가능한 이미지 캐러셀 UI를 구현하는 방법을 소개합니다.
---

## ✨ 개요

`ViewPager` 또는 `ViewPager2`를 대체하여 **더 유연하고 커스터마이징이 쉬운 좌우 스와이프 UI**를 만들고 싶다면?  
**RecyclerView + SnapHelper** 조합을 사용해 **카드형 좌우 슬라이드 뷰**를 만들어보세요.

이 방식은 **캐러셀(Carousel)** 혹은 **카드 스와이프 UI** 구현에 자주 사용됩니다.

---

## 1. ✅ 주요 구성 요소

| 기술            | 설명 |
|-----------------|------|
| RecyclerView    | 이미지 리스트 표시 |
| PagerSnapHelper | 한 화면에 하나의 이미지 정렬 |
| Handler         | 일정 시간 간격으로 자동 스크롤 |
| Glide           | 이미지 로딩 라이브러리 (네트워크/리소스 이미지 모두 가능) |

---

## 2. ✅ 구현 예제

APP 수준 Gradle 에 의존성을 추가
> implementation 'com.github.bumptech.glide:glide:4.16.0'

AndroidManifest.xml 에 인터넷 권한을 추가해주세요.
> <uses-permission android:name="android.permission.INTERNET" /\>

### 2.1 XML 레이아웃

```xml
<androidx.recyclerview.widget.RecyclerView
    android:id="@+id/carouselRecyclerView"
    android:layout_width="match_parent"
    android:layout_height="200dp"
    android:clipToPadding="false"
    android:paddingStart="32dp"
    android:paddingEnd="32dp"
    android:overScrollMode="never" />
```
- clipToPadding="false" 설정으로 카드가 바깥쪽으로 드러나 보이는 효과

### 2.2 Card Item Layout

```xml
<androidx.cardview.widget.CardView xmlns:android="http://schemas.android.com/apk/res/android"
                                   android:layout_width="300dp"
                                   android:layout_height="300dp"
                                   android:layout_margin="8dp"
                                   android:elevation="4dp"
                                   android:radius="12dp">

    <ImageView
            android:id="@+id/imageView"
            android:layout_width="match_parent"
            android:layout_height="300dp"
            android:scaleType="centerCrop" />
</androidx.cardview.widget.CardView>
```

### 2.3 ImageAdapter

```kotlin
class ImageAdapter(private val items: List<String>) :
    RecyclerView.Adapter<ImageAdapter.ImageViewHolder>() {

    inner class ImageViewHolder(private val binding: ItemImageBinding) :
        RecyclerView.ViewHolder(binding.root) {
        fun bind(url: String) {
            Glide.with(binding.imageView)
                .load(url)
                .into(binding.imageView)
        }
    }

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): ImageViewHolder {
        val binding = ItemImageBinding.inflate(LayoutInflater.from(parent.context), parent, false)
        return ImageViewHolder(binding)
    }

    override fun onBindViewHolder(holder: ImageViewHolder, position: Int) {
        holder.bind(items[position])
    }

    override fun getItemCount(): Int = items.size
}
```

### 2.4. MainActivity

```kotlin
class MainActivity : AppCompatActivity() {

    private lateinit var binding: ActivityMainBinding

    private val TAG: String = "MainActivity"

    private val handler = Handler(Looper.getMainLooper())
    private var currentIndex = 0
    private val interval: Long = 3000L

    private val imageUrls = listOf(
        "https://picsum.photos/seed/1/600/400",
        "https://picsum.photos/seed/2/600/400",
        "https://picsum.photos/seed/3/600/400",
        "https://picsum.photos/seed/4/600/400"
    )

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityMainBinding.inflate(layoutInflater)
        setContentView(binding.root)

        val adapter = ImageAdapter(imageUrls)
        val layoutManager = LinearLayoutManager(this, LinearLayoutManager.HORIZONTAL, false)

        binding.carouselRecyclerView.layoutManager = layoutManager
        binding.carouselRecyclerView.adapter = adapter

        PagerSnapHelper().attachToRecyclerView(binding.carouselRecyclerView)

        startAutoScroll()
    }

    private fun startAutoScroll() {
        handler.postDelayed(object : Runnable {
            override fun run() {
                val itemCount = binding.carouselRecyclerView.adapter?.itemCount ?: 0
                if (itemCount == 0) return

                currentIndex = (currentIndex + 1) % itemCount
                binding.carouselRecyclerView.smoothScrollToPosition(currentIndex)

                handler.postDelayed(this, interval)
            }
        }, interval)
    }

    override fun onDestroy() {
        super.onDestroy()
        handler.removeCallbacksAndMessages(null)
    }
}
```

---

## 3.🧠 **장점 요약**

| 항목        | 설명                               |
| --------- | -------------------------------- |
| ✅ 성능      | ViewPager보다 가벼움                  |
| ✅ 유연성     | 여백, 슬라이드 간격, 클릭 이벤트 등 자유롭게 구성 가능 |
| ✅ 확장성     | 자동 슬라이드, 무한 루프, 애니메이션 등 확장 용이    |
| ✅ 코드 재사용성 | 일반 RecyclerView 구조이므로 재활용 쉬움     |

---

## 4.🧠 **결론**

- RecyclerView + PagerSnapHelper + Handler 조합으로 가볍고 유연한 이미지 캐러셀 UI를 만들 수 있습니다.
- 앱 배너, 이벤트 슬라이드 등 다양한 화면에 적용 가능하며 ViewPager2보다 제약이 적습니다.
- 사용자 경험(UX)을 높이기 위해 자동 스크롤, 클릭 이벤트, 점 인디케이터 등을 추가로 구현해 보세요!