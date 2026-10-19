---
title: (Android/Paging) Paging3 기초 — PagingSource, Pager, PagingData
tags: [ Android, Paging ]
style: fill
color: dark
description: Jetpack Paging3의 핵심 구성 요소인 PagingSource, Pager, PagingData, PagingDataAdapter와 LoadState 처리 방법을 정리합니다.
---

---

## 1. Paging3란?

대용량 목록을 **필요한 만큼씩 나눠서 로드**해 보여주는 Jetpack 라이브러리입니다.
스크롤이 끝에 가까워지면 다음 페이지를 자동으로 요청하고, 메모리에는 일부만 유지합니다.

```
PagingSource  →  Pager  →  Flow<PagingData>  →  PagingDataAdapter / LazyPagingItems  →  화면
 (데이터 로드)   (설정/조립)   (ViewModel 노출)          (UI에서 소비)
```

**의존성**:
```kotlin
implementation("androidx.paging:paging-runtime-ktx:3.3.2")
// Compose 사용 시
implementation("androidx.paging:paging-compose:3.3.2")
```

---

## 2. PagingSource — 페이지 데이터 로드

"몇 번째 페이지를 어떻게 가져올지" 정의합니다. 제네릭은 `<Key, Value>`(페이지 키, 아이템)입니다.

```kotlin
class ArticlePagingSource(
    private val api: ArticleApi
) : PagingSource<Int, Article>() {

    // 최초 로드 및 다음/이전 페이지 로드 시 호출
    override suspend fun load(params: LoadParams<Int>): LoadResult<Int, Article> {
        val page = params.key ?: 1  // 첫 로드는 key가 null

        return try {
            val articles = api.getArticles(page = page, size = params.loadSize)

            LoadResult.Page(
                data = articles,
                prevKey = if (page == 1) null else page - 1,    // null이면 이전 페이지 없음
                nextKey = if (articles.isEmpty()) null else page + 1  // null이면 마지막 페이지
            )
        } catch (e: Exception) {
            LoadResult.Error(e)  // 에러를 던지지 말고 결과로 반환
        }
    }

    // 새로고침/무효화 후 어느 페이지부터 다시 로드할지 결정
    override fun getRefreshKey(state: PagingState<Int, Article>): Int? =
        state.anchorPosition?.let { anchor ->
            state.closestPageToPosition(anchor)?.prevKey?.plus(1)
                ?: state.closestPageToPosition(anchor)?.nextKey?.minus(1)
        }
}
```

> `load()` 안에서 예외를 밖으로 던지면 안 됩니다. `LoadResult.Error`로 감싸 반환해야 Paging이 에러 상태와 재시도를 관리합니다.

---

## 3. Pager와 PagingData — ViewModel에서 노출

```kotlin
@HiltViewModel
class ArticleViewModel @Inject constructor(
    private val api: ArticleApi
) : ViewModel() {

    val articles: Flow<PagingData<Article>> = Pager(
        config = PagingConfig(
            pageSize = 20,              // 한 번에 가져올 개수
            prefetchDistance = 5,       // 끝에서 몇 개 남았을 때 미리 로드할지
            enablePlaceholders = false,
            initialLoadSize = 40        // 첫 로드는 크게 (기본 pageSize*3)
        ),
        pagingSourceFactory = { ArticlePagingSource(api) }
    ).flow
        .cachedIn(viewModelScope)  // 화면 회전/재구독에도 캐시 유지 (필수)
}
```

> **`cachedIn(viewModelScope)`** 를 빠뜨리면 구독할 때마다 처음부터 다시 로드합니다. 변환 연산(`map`, `filter`)은 `cachedIn` **앞에** 적용하세요.

---

## 4. UI 소비 — View 시스템

```kotlin
class ArticleAdapter : PagingDataAdapter<Article, ArticleViewHolder>(DIFF) {

    override fun onBindViewHolder(holder: ArticleViewHolder, position: Int) {
        // placeholder가 켜져 있으면 null일 수 있음
        getItem(position)?.let { holder.bind(it) }
    }

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int) =
        ArticleViewHolder.create(parent)

    companion object {
        val DIFF = object : DiffUtil.ItemCallback<Article>() {
            override fun areItemsTheSame(a: Article, b: Article) = a.id == b.id
            override fun areContentsTheSame(a: Article, b: Article) = a == b
        }
    }
}

// Fragment
viewLifecycleOwner.lifecycleScope.launch {
    viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.articles.collectLatest { adapter.submitData(it) }
    }
}
```

---

## 5. UI 소비 — Compose

```kotlin
@Composable
fun ArticleList(viewModel: ArticleViewModel = hiltViewModel()) {
    val articles = viewModel.articles.collectAsLazyPagingItems()

    LazyColumn {
        items(
            count = articles.itemCount,
            key = articles.itemKey { it.id }
        ) { index ->
            articles[index]?.let { ArticleItem(it) }
        }
    }
}
```

---

## 6. LoadState — 로딩/에러 상태 표시

```kotlin
// View 시스템
adapter.addLoadStateListener { loadStates ->
    binding.progress.isVisible = loadStates.refresh is LoadState.Loading
    binding.retryButton.isVisible = loadStates.refresh is LoadState.Error
}
binding.retryButton.setOnClickListener { adapter.retry() }

// 하단 로딩/에러 푸터
recyclerView.adapter = adapter.withLoadStateFooter(MyLoadStateAdapter { adapter.retry() })
```

| LoadState | 의미 |
|-----------|------|
| `refresh` | 최초 로드 / 새로고침 |
| `append` | 다음 페이지 로드 (아래로) |
| `prepend` | 이전 페이지 로드 (위로) |

---

## 7. 새로고침

```kotlin
adapter.refresh()                        // 처음부터 다시 로드
swipeRefreshLayout.setOnRefreshListener { adapter.refresh() }
```

---

## 8. 정리

- **PagingSource**: 페이지 단위 로드 로직 (`LoadResult.Page` / `LoadResult.Error`)
- **Pager + PagingConfig**: 페이지 크기, 프리페치 등 설정, `Flow<PagingData>` 생성
- ViewModel에서 `cachedIn(viewModelScope)` 필수
- UI는 `PagingDataAdapter` 또는 `collectAsLazyPagingItems()`
- 로딩/에러는 `LoadState`로 처리하고 `retry()`/`refresh()` 제공
