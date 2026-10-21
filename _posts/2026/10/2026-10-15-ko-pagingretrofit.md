---
title: (Android/Paging) Paging3 + Retrofit — 무한 스크롤 구현
tags: [ Android, Paging, Retrofit ]
style: fill
color: dark
description: Paging3와 Retrofit을 연동해 무한 스크롤 목록을 구현하는 방법과 커서 기반 페이징, 에러 처리, 검색 연동, Repository 구성을 정리합니다.
---

---

## 1. 전체 구조

```
Retrofit API ──▶ PagingSource ──▶ Pager ──▶ ViewModel(Flow<PagingData>) ──▶ UI
  (서버 호출)     (페이지 변환)    (설정)
```

서버가 페이지 번호(offset) 방식인지, 커서(next token) 방식인지에 따라 PagingSource의 **Key 타입**이 달라집니다.

---

## 2. API 정의

```kotlin
interface ArticleApi {
    // 페이지 번호 방식
    @GET("articles")
    suspend fun getArticles(
        @Query("page") page: Int,
        @Query("size") size: Int,
        @Query("q") query: String? = null
    ): ArticleResponse
}

data class ArticleResponse(
    val items: List<ArticleDto>,
    val hasNext: Boolean
)
```

---

## 3. 페이지 번호 방식 PagingSource

```kotlin
class ArticlePagingSource(
    private val api: ArticleApi,
    private val query: String?
) : PagingSource<Int, Article>() {

    override suspend fun load(params: LoadParams<Int>): LoadResult<Int, Article> {
        val page = params.key ?: START_PAGE
        return try {
            val response = api.getArticles(page, params.loadSize, query)

            LoadResult.Page(
                data = response.items.map { it.toDomain() },
                prevKey = if (page == START_PAGE) null else page - 1,
                // 서버가 알려주는 hasNext로 마지막 페이지 판단 (빈 응답 대기보다 정확)
                nextKey = if (response.hasNext) page + 1 else null
            )
        } catch (e: IOException) {
            LoadResult.Error(e)      // 네트워크 오류
        } catch (e: HttpException) {
            LoadResult.Error(e)      // 서버 오류 (4xx, 5xx)
        }
    }

    override fun getRefreshKey(state: PagingState<Int, Article>): Int? =
        state.anchorPosition?.let { state.closestPageToPosition(it)?.prevKey?.plus(1) }

    private companion object { const val START_PAGE = 1 }
}
```

> `CancellationException`은 잡으면 안 됩니다. 위처럼 `IOException`/`HttpException`만 잡거나, 광범위하게 `Exception`을 잡는다면 `CancellationException`은 다시 던져야 합니다.

---

## 4. 커서 방식 (next token) PagingSource

서버가 `nextCursor`를 내려주는 경우 Key를 `String`으로 사용합니다.

```kotlin
class FeedPagingSource(private val api: FeedApi) : PagingSource<String, Post>() {

    override suspend fun load(params: LoadParams<String>): LoadResult<String, Post> =
        try {
            val response = api.getFeed(cursor = params.key, limit = params.loadSize)
            LoadResult.Page(
                data = response.posts,
                prevKey = null,                   // 한 방향(아래로)만 로드
                nextKey = response.nextCursor     // null이면 끝
            )
        } catch (e: Exception) {
            if (e is CancellationException) throw e
            LoadResult.Error(e)
        }

    override fun getRefreshKey(state: PagingState<String, Post>): String? = null
}
```

---

## 5. Repository와 ViewModel

```kotlin
class ArticleRepository @Inject constructor(private val api: ArticleApi) {

    fun getArticles(query: String?): Flow<PagingData<Article>> = Pager(
        config = PagingConfig(pageSize = 20, prefetchDistance = 5),
        pagingSourceFactory = { ArticlePagingSource(api, query) }
    ).flow
}

@HiltViewModel
class ArticleViewModel @Inject constructor(
    repository: ArticleRepository
) : ViewModel() {

    private val query = MutableStateFlow<String?>(null)

    // 검색어가 바뀔 때마다 새 Pager로 교체 (flatMapLatest)
    val articles: Flow<PagingData<Article>> = query
        .debounce(300)
        .flatMapLatest { repository.getArticles(it) }
        .cachedIn(viewModelScope)

    fun search(text: String) { query.value = text.ifBlank { null } }
}
```

> 검색어 변경 시 `flatMapLatest`로 이전 Pager를 취소하고 새로 시작합니다. `debounce`로 타이핑마다 요청하는 것을 방지합니다.

---

## 6. 에러 처리와 재시도

```kotlin
@Composable
fun ArticleScreen(vm: ArticleViewModel = hiltViewModel()) {
    val items = vm.articles.collectAsLazyPagingItems()

    LazyColumn {
        items(items.itemCount, key = items.itemKey { it.id }) { index ->
            items[index]?.let { ArticleItem(it) }
        }

        // 하단(append) 상태
        when (val append = items.loadState.append) {
            is LoadState.Loading -> item { CircularProgressIndicator() }
            is LoadState.Error -> item {
                ErrorFooter(append.error.message) { items.retry() }  // 마지막 실패 지점부터 재시도
            }
            else -> Unit
        }
    }

    // 최초 로드 상태
    if (items.loadState.refresh is LoadState.Loading) FullScreenLoading()
    if (items.loadState.refresh is LoadState.Error) FullScreenError { items.refresh() }
}
```

| 동작 | 설명 |
|------|------|
| `retry()` | 실패한 **마지막 로드만** 다시 시도 |
| `refresh()` | 데이터를 버리고 **처음부터** 다시 로드 |

---

## 7. 정리

- 서버 방식에 맞춰 Key 선택: 페이지 번호 → `Int`, 커서 → `String`
- `LoadResult.Page`로 성공, `LoadResult.Error`로 실패 반환 (예외를 밖으로 던지지 않기)
- 마지막 페이지는 `nextKey = null`로 알림 (서버의 `hasNext` 활용)
- 검색은 `flatMapLatest + debounce`로 Pager를 교체, 마지막에 `cachedIn`
- UI에서 `loadState`로 로딩/에러 표시, `retry()`와 `refresh()` 구분 사용
