---
title: (Android/Retrofit) Retrofit + Flow 연동 패턴
tags: [ Android, Retrofit ]
style: fill
color: dark
description: Retrofit 호출을 Flow로 감싸 로딩, 성공, 실패 상태를 방출하는 패턴과 주기적 폴링, 검색 debounce, 재시도, 캐시와 네트워크 결합 방법을 정리합니다.
---

---

## 1. Retrofit은 Flow를 직접 반환하지 않는다

Retrofit의 suspend 함수는 **값 하나를 반환**합니다. 한 번 응답하고 끝나는 요청이므로 `Flow` 반환 타입을 선언하지 않습니다.

```kotlin
interface ArticleApi {
    @GET("articles")
    suspend fun getArticles(@Query("q") query: String): List<Article>   // ✅

    // ❌ suspend fun ...: Flow<List<Article>> 는 의미 없음
}
```

Flow는 **Repository/ViewModel에서 호출을 감싸** 상태나 반복 요청을 표현할 때 사용합니다.

---

## 2. 로딩/성공/실패를 방출하는 Flow

```kotlin
sealed interface Resource<out T> {
    data object Loading : Resource<Nothing>
    data class Success<T>(val data: T) : Resource<T>
    data class Error(val message: String) : Resource<Nothing>
}

fun <T> apiFlow(block: suspend () -> T): Flow<Resource<T>> = flow {
    emit(Resource.Loading)
    emit(Resource.Success(block()))
}.catch { e ->
    if (e is CancellationException) throw e          // 취소는 전파
    emit(Resource.Error(e.toUserMessage()))
}

class ArticleRepository @Inject constructor(private val api: ArticleApi) {
    fun getArticles(query: String): Flow<Resource<List<Article>>> =
        apiFlow { api.getArticles(query) }
}
```

`flow { }` 안에서 suspend 호출은 **collector의 코루틴**에서 실행됩니다. 스레드를 지정하려면 `flowOn`을 사용하세요 (Retrofit suspend는 메인에서도 안전하므로 보통 필요 없습니다).

---

## 3. ViewModel에서 StateFlow로

```kotlin
@HiltViewModel
class ArticleViewModel @Inject constructor(
    private val repository: ArticleRepository
) : ViewModel() {

    private val query = MutableStateFlow("")

    val state: StateFlow<Resource<List<Article>>> = query
        .debounce(300)                                    // 타이핑이 멈춘 뒤 요청
        .distinctUntilChanged()                           // 같은 검색어 중복 방지
        .flatMapLatest { q -> repository.getArticles(q) } // 새 검색어가 오면 이전 요청 취소
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), Resource.Loading)

    fun onQueryChange(text: String) { query.value = text }
}
```

| 연산자 | 역할 |
|--------|------|
| `debounce` | 짧은 간격 입력을 묶어 요청 횟수 감소 |
| `distinctUntilChanged` | 동일 값 반복 방지 |
| `flatMapLatest` | 최신 입력만 처리, 이전 진행 요청은 **자동 취소** |

---

## 4. 주기적 폴링

```kotlin
fun pollStatus(orderId: Long, intervalMs: Long = 5_000): Flow<OrderStatus> = flow {
    while (true) {
        emit(api.getOrderStatus(orderId))          // 요청 결과 방출
        delay(intervalMs)
    }
}.takeWhile { it != OrderStatus.DONE }             // 완료되면 종료 (마지막 값 제외)
 .catch { /* 일시 오류는 무시하고 다음 주기에 재시도하려면 retry 사용 */ }
```

> `takeWhile`은 조건을 만족하지 않는 값을 **방출하지 않고** 종료합니다. 마지막 `DONE` 값도 받아야 한다면 `transformWhile`을 사용하세요.

앱이 백그라운드로 가면 수집이 멈추도록 `WhileSubscribed`/`repeatOnLifecycle`과 함께 쓰세요.

---

## 5. 재시도 — retryWhen

```kotlin
fun <T> Flow<T>.retryOnNetworkError(maxRetries: Int = 3): Flow<T> =
    retryWhen { cause, attempt ->
        if (cause is IOException && attempt < maxRetries) {
            delay(1_000L * (attempt + 1))          // 점진적으로 대기 시간 증가
            true                                   // 재시도
        } else {
            false                                  // 포기하고 예외 전달
        }
    }

repository.getArticles(q).retryOnNetworkError()
```

400/401 같은 클라이언트 오류는 **재시도해도 소용없으므로** 네트워크 오류(IOException)와 일부 5xx만 재시도하세요.

---

## 6. 캐시 후 네트워크 — 두 단계 방출

화면에 **캐시를 먼저 보여주고** 서버 최신 데이터로 갱신합니다.

```kotlin
fun getArticles(): Flow<List<Article>> = flow {
    emit(dao.getAll())                              // 1. 로컬 캐시 즉시 방출
    val remote = api.getArticles("")                // 2. 서버 데이터 요청
    dao.replaceAll(remote.map { it.toEntity() })
    emit(dao.getAll())                              // 3. 갱신된 데이터 방출
}.catch { emit(dao.getAll()) }                      // 오프라인이면 캐시로 대체
```

더 견고한 구조는 **DB의 `Flow`를 단일 진실 공급원**으로 두고, 서버 동기화는 별도로 `refresh()` 하는 방식입니다 (Room + Flow 글 참고).

---

## 7. 병렬 요청 합치기

```kotlin
fun loadHome(): Flow<HomeData> = flow {
    val data = coroutineScope {
        val banners = async { api.getBanners() }
        val articles = async { api.getArticles("") }
        HomeData(banners.await(), articles.await())   // 둘 다 끝나면 합쳐서 방출
    }
    emit(data)
}

// 서로 독립적으로 갱신되는 두 Flow를 합칠 때
val ui = combine(userFlow, notificationFlow) { user, noti -> Home(user, noti) }
```

---

## 8. 정리

- Retrofit suspend는 값 하나를 반환 → **Repository에서 `flow { }`로 감싸** 상태를 방출
- `Resource`(Loading/Success/Error)로 UI 상태 표현, 취소 예외는 다시 던지기
- 검색은 `debounce` + `distinctUntilChanged` + **`flatMapLatest`** (이전 요청 자동 취소)
- 폴링은 `while + delay`, 종료 조건은 `takeWhile`/`transformWhile`
- 재시도는 `retryWhen`으로 **네트워크 오류에 한해** 지수적 대기
- 캐시 우선 표시 후 갱신, 또는 DB Flow를 단일 진실 공급원으로 구성
