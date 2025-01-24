---
title: Android 네이버 검색 API 연동 방법 및 샘플 코드
tags: [Android]
style: fill
color: dark
description: Android 네이버 검색 API 연동 방법 및 샘플 코드
---

## 개요
네이버 검색 API를 적용하기 전에 애플리케이션 등록(API 이용신청)이 필요합니다.

[API신청주소](https://developers.naver.com/apps/#/register?defaultScope=search)
이 사이트에 접속하여 API 등록 신청
- "애플리케이션 이름" : 본인의 프로젝트 명 
- "사용 API" : 검색
- "비로그인 오픈 API 서비스 환경" : Android 설정 & 패키지 명 추가

[네이버 검색API 가이드](https://developers.naver.com/docs/serviceapi/search/blog/blog.md#%EB%B8%94%EB%A1%9C%EA%B7%B8)
이 사이트에 접속해서 검색 API 개발 가이드 확인 가능합니다.

네이버 블로그 검색을 한 결과 목록을 RecyclerView 로 보여주는 샘플 코드입니다.  

---

## 1. gradle 설정 추가
사용 라이브러리
- 코루틴, retrofit (네트워크 통신), okhttp (로깅), glide (이미지), liveData 등

### 1-1. libs.version.toml 추가
```kotlin
// libs.version.toml 
[versions]
okhttp = "4.11.0"
lifecycleLivedataKtx = "2.8.7"
lifecycleViewmodelKtx = "2.8.7"
coroutines = "1.7.3"
retrofit = "2.9.0"
lifecycle = "2.6.1"

[libraries]
androidx-lifecycle-livedata-ktx = { group = "androidx.lifecycle", name = "lifecycle-livedata-ktx", version.ref = "lifecycleLivedataKtx" }
androidx-lifecycle-viewmodel-ktx = { group = "androidx.lifecycle", name = "lifecycle-viewmodel-ktx", version.ref = "lifecycleViewmodelKtx" }

coroutines-android = { module = "org.jetbrains.kotlinx:kotlinx-coroutines-android", version.ref = "coroutines" }
coroutines-core = { module = "org.jetbrains.kotlinx:kotlinx-coroutines-core", version.ref = "coroutines" }

lifecycle-viewmodel = { module = "androidx.lifecycle:lifecycle-viewmodel-ktx", version.ref = "lifecycle" }
lifecycle-runtime = { module = "androidx.lifecycle:lifecycle-runtime-ktx", version.ref = "lifecycle" }

retrofit = { module = "com.squareup.retrofit2:retrofit", version.ref = "retrofit" }
retrofit-gson = { module = "com.squareup.retrofit2:converter-gson", version.ref = "retrofit" }
okhttp-logging-interceptor = { group = "com.squareup.okhttp3", name = "logging-interceptor", version.ref = "okhttp" }
```

### 1-2 build.gradle.kts 추가
```kotlin
// liveData
implementation(libs.androidx.lifecycle.livedata.ktx)
implementation(libs.androidx.lifecycle.viewmodel.ktx)

// Retrofit
implementation(libs.retrofit)
implementation(libs.retrofit.gson)

// Coroutines
implementation(libs.coroutines.android)
implementation(libs.coroutines.core)

// Lifecycle
implementation(libs.lifecycle.viewmodel)
implementation(libs.lifecycle.runtime)

// HttpLoggingInterceptor 추가
implementation(libs.okhttp.logging.interceptor)
```

---

## 2. Android Manifest 권한 추가
- 통신을 위한 인터넷 권한 추가

```kotlin
// Android Manifest 선언
<!-- 인터넷 권한 추가 -->
<uses-permission android:name="android.permission.INTERNET" />
```

---

## 3. 레이아웃 소스 코드
### 3-1 activity_main.xml
- 블로그 검색 결과 목록을 보여 줄 RecyclerView 

```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
                                                   xmlns:app="http://schemas.android.com/apk/res-auto"
                                                   xmlns:tools="http://schemas.android.com/tools"
                                                   android:id="@+id/main"
                                                   android:layout_width="match_parent"
                                                   android:layout_height="match_parent"
                                                   tools:context=".MainActivity">

    <androidx.recyclerview.widget.RecyclerView
            android:id="@+id/list"
            android:layout_width="0dp"
            android:layout_height="0dp"
            android:layout_margin="4dp"
            app:layoutManager="androidx.recyclerview.widget.LinearLayoutManager"
            app:layout_constraintBottom_toBottomOf="parent"
            app:layout_constraintEnd_toEndOf="parent"
            app:layout_constraintStart_toStartOf="parent"
            app:layout_constraintTop_toTopOf="parent" />

</androidx.constraintlayout.widget.ConstraintLayout>
```

### 3-2. item_blog.xml
- 블로그 검색 결과 목록에 들어갈 item view

```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
                                                   xmlns:app="http://schemas.android.com/apk/res-auto"
                                                   xmlns:tools="http://schemas.android.com/tools"
                                                   android:layout_width="match_parent"
                                                   android:layout_height="100dp"
                                                   android:orientation="horizontal"
                                                   android:padding="8dp">

    <!-- 검색 키워드 -->
    <TextView
            android:id="@+id/textViewTitle"
            android:layout_width="0dp"
            android:layout_height="wrap_content"
            android:layout_marginHorizontal="8dp"
            android:ellipsize="end"
            android:maxLines="1"
            android:textSize="16sp"
            android:textStyle="bold"
            app:layout_constraintEnd_toEndOf="parent"
            app:layout_constraintStart_toStartOf="parent"
            app:layout_constraintTop_toTopOf="parent"
            tools:text="블로그 제목" />

    <!-- 검색 날짜 -->
    <TextView
            android:id="@+id/textViewDescription"
            android:layout_width="0dp"
            android:layout_height="wrap_content"
            android:layout_marginHorizontal="8dp"
            android:layout_marginTop="2dp"
            android:ellipsize="end"
            android:maxLines="2"
            android:textSize="14sp"
            app:layout_constraintBottom_toTopOf="@id/textViewBloggerName"
            app:layout_constraintEnd_toEndOf="parent"
            app:layout_constraintStart_toStartOf="parent"
            app:layout_constraintTop_toBottomOf="@id/textViewTitle"
            tools:text="블로그 설명" />


    <TextView
            android:id="@+id/textViewBloggerName"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:textSize="14sp"
            app:layout_constraintStart_toStartOf="@id/textViewTitle"
            app:layout_constraintTop_toBottomOf="@id/textViewDescription"
            tools:text="뚜벅뚜벅이" />

    <TextView
            android:id="@+id/textViewDot"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_marginStart="2dp"
            android:text="ㆍ"
            android:textSize="14sp"
            app:layout_constraintStart_toEndOf="@id/textViewBloggerName"
            app:layout_constraintTop_toBottomOf="@id/textViewDescription" />

    <TextView
            android:id="@+id/textViewPostDate"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_marginStart="2dp"
            android:textSize="14sp"
            app:layout_constraintStart_toEndOf="@id/textViewDot"
            app:layout_constraintTop_toBottomOf="@id/textViewDescription"
            tools:text="yyyy-mm-dd" />

</androidx.constraintlayout.widget.ConstraintLayout>
```

---

## 4. 네트워크 관련 소스 코드
- retrofit2 를 사용하고 네트워크 결과를 data class 로 받을 예정

### 4.1 data class

```kotlin
import com.google.gson.annotations.SerializedName

data class BlogSearchResponse(
    val lastBuildDate: String,
    val total: Int,
    val start: Int,
    val display: Int,
    val items: List<BlogItem>,
)

data class BlogItem(
    val title: String,
    val link: String,
    val description: String,
    @SerializedName("bloggername")
    val bloggerName: String,
    @SerializedName("bloggerlink")
    val bloggerLink: String,
    val postdate: String,
    var thumbnail: String = ""
)
```

### 4.2 RetrofitService.kt
- API를 요청할 때 다음 예와 같이 HTTP 요청 헤더에 클라이언트 아이디와 클라이언트 시크릿을 추가해야 합니다.

```kotlin
import retrofit2.http.GET
import retrofit2.http.Header
import retrofit2.http.Query

interface RetrofitService {

    @GET("v1/search/blog.json")
    suspend fun searchBlogs(
        @Header("X-Naver-Client-Id") clientId: String = "애플리케이션 등록 시 발급받은 클라이언트 아이디 값",
        @Header("X-Naver-Client-Secret") clientSecret: String = "애플리케이션 등록 시 발급받은 클라이언트 시크릿 값",
        @Query("query") query: String,      //UTF-8 Encoding
        @Query("display") display: Int = 10,//한번에 표시할 검색 결과 수 기본값 10 최댓값 100
        @Query("start") start: Int = 1,     //검색 시작 위치 기본값 1 최댓값 1000
        @Query("sort") sort: String = "sim" //sim:정확도순 내림차순 default, date: 날짜순 내림차순
    ): BlogSearchResponse
}
```

### 4.3 RemoteRepository.kt
```kotlin
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.flow

class RemoteRepository(private val retrofitService: RetrofitService) {

    fun searchBlogs(query: String, display: Int = 10, start: Int = 1, sort: String = "sim"): Flow<BlogSearchResponse> = flow {
        val response = retrofitService.searchBlogs(query = query, display = display, start = start, sort = sort)
        emit(response) // API 응답 데이터를 방출
    }
}
```

---

## 5. UI 소스 코드
- viewModel 를 사용하여 데이터를 관리하고 flow 활용

### 5.1 MainViewModel.kt
```kotlin
import androidx.lifecycle.ViewModel
import androidx.lifecycle.ViewModelProvider
import androidx.lifecycle.viewModelScope
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.launch

class MainViewModel(private val repository: RemoteRepository) : ViewModel() {

    private val _blogs = MutableStateFlow<List<BlogItem>>(emptyList())
    val blogs: StateFlow<List<BlogItem>> = _blogs

    fun searchBlogs(query: String, display: Int = 10, start: Int = 1, sort: String = "sim") {
        viewModelScope.launch {
            repository.searchBlogs(query, display, start, sort).collect { response ->
                _blogs.value = response.items // BlogItem 리스트 업데이트
            }
        }
    }
}

class BlogViewModelFactory(private val repository: RemoteRepository) : ViewModelProvider.Factory {
    override fun <T : ViewModel> create(modelClass: Class<T>): T {
        if (modelClass.isAssignableFrom(MainViewModel::class.java)) {
            @Suppress("UNCHECKED_CAST")
            return MainViewModel(repository) as T
        }
        throw IllegalArgumentException("Unknown ViewModel class")
    }
}
```

### 5.2 MainActivity.kt
```kotlin
import android.os.Bundle
import android.util.Log
import androidx.activity.viewModels
import androidx.appcompat.app.AppCompatActivity
import androidx.lifecycle.lifecycleScope
import com.js.sample.databinding.ActivityMainBinding
import kotlinx.coroutines.flow.collectLatest
import kotlinx.coroutines.launch

class MainActivity : AppCompatActivity() {

    private lateinit var binding: ActivityMainBinding

    private val viewModel: MainViewModel by viewModels {
        BlogViewModelFactory(RemoteRepository(RetrofitClient.service))
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityMainBinding.inflate(layoutInflater)
        setContentView(binding.root)

        val adapter = BlogListAdapter {
            Log.e("TAG", "clicked $it")
        }
        binding.list.adapter = adapter

        // ViewModel의 데이터 관찰
        lifecycleScope.launch {
            viewModel.blogs.collectLatest { blogs ->
                if (blogs.isNotEmpty()) {
                    adapter.submitList(blogs)
                }
            }
        }

        // 검색 실행
        viewModel.searchBlogs("강남역 맛집")
    }

}
```

### 5.3 BlogListAdapter.kt
- title 에는 html tag 가 포함되어있는데 제거하고 싶다면 Jsoup removeHtmlTags 를 이용

```kotlin
import android.view.LayoutInflater
import android.view.ViewGroup
import androidx.recyclerview.widget.DiffUtil
import androidx.recyclerview.widget.ListAdapter
import androidx.recyclerview.widget.RecyclerView
import com.js.sample.databinding.ItemBlogBinding
import java.text.SimpleDateFormat
import java.util.Locale

class BlogListAdapter(
    private val onSelect: (BlogItem) -> Unit,
) : ListAdapter<BlogItem, BlogListAdapter.BlogViewHolder>(BlogDiffCallback()) {

    inner class BlogViewHolder(private val binding: ItemBlogBinding) :
        RecyclerView.ViewHolder(binding.root) {

        fun bind(item: BlogItem) {
            with(binding) {
                root.setOnClickListener { onSelect(item) }

                textViewTitle.text = item.title
                textViewDescription.text = item.description
                textViewBloggerName.text = item.bloggerName
                textViewPostDate.text = formatDate(item.postdate)
            }

        }
    }

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): BlogViewHolder {
        val binding = ItemBlogBinding.inflate(LayoutInflater.from(parent.context), parent, false)
        return BlogViewHolder(binding)
    }

    override fun onBindViewHolder(holder: BlogViewHolder, position: Int) {
        val item = getItem(position)
        holder.bind(item)
    }

    class BlogDiffCallback : DiffUtil.ItemCallback<BlogItem>() {
        override fun areItemsTheSame(oldItem: BlogItem, newItem: BlogItem): Boolean {
            // 각 항목의 고유성을 비교 (예: 키워드)
            return oldItem.title == newItem.title
        }

        override fun areContentsTheSame(oldItem: BlogItem, newItem: BlogItem): Boolean {
            // 전체 내용 비교 (예: 키워드와 날짜)
            return oldItem == newItem
        }
    }

    fun formatDate(input: String): String {
        return try {
            val inputFormat = SimpleDateFormat("yyyyMMdd", Locale.getDefault())
            val outputFormat = SimpleDateFormat("yyyy-MM-dd", Locale.getDefault())
            val date = inputFormat.parse(input) ?: ""
            outputFormat.format(date)
        } catch (e: Exception) {
            input
        }
    }
}
```