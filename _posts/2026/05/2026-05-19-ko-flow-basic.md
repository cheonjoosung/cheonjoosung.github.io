---
title: (Kotlin/코틀린) Flow 기초 완전 정리 - Cold Stream 생성과 collect
tags: [ Kotlin, Android ]
style: fill
color: dark
description: Kotlin Flow의 핵심 개념인 Cold Stream, flow 빌더, emit, collect, flowOn, 중간·종단 연산자, 에러 처리까지 Android 실전 코드와 함께 깊이 있게 정리합니다.
---

## 개요

- Kotlin **Flow** 의 기초를 다룹니다.
- Flow는 **비동기 데이터 스트림** 을 선언적으로 처리하는 코루틴 기반 API입니다.
- 이 글에서는 다음을 설명합니다.
  - Flow란 무엇인가 — Cold Stream 개념
  - `flow { }` 빌더와 `emit`
  - `collect` — Flow 소비
  - 중간 연산자 — `map`, `filter`, `transform`, `take`
  - 종단 연산자 — `collect`, `toList`, `first`, `single`
  - `flowOn` — 실행 스레드 지정
  - 에러 처리 — `catch`, `onCompletion`
  - Android 실전 패턴

---

## 1. Flow란

Flow는 **순서가 있는 비동기 값의 흐름**입니다.

```
Flow<Int>: 1 → 2 → 3 → 4 → 5 → [완료]
           ↑
           각 값이 비동기로 emit 됨
```

### RxJava Observable과의 비교

| 항목 | Flow | RxJava Observable |
|------|------|------------------|
| 기반 | 코루틴 (suspend) | 자체 스케줄러 |
| 학습 곡선 | 낮음 | 높음 |
| 취소 | 코루틴 취소와 통합 | Disposable 수동 관리 |
| 에러 처리 | `catch` | `onError` |
| Kotlin 친화성 | ✅ 네이티브 | 별도 라이브러리 |
| 백프레셔 | 코루틴 중단으로 자동 처리 | 명시적 설정 필요 |

---

## 2. Cold Stream vs Hot Stream

Flow의 핵심 특성은 **Cold Stream**입니다.

### Cold Stream (Flow)

```kotlin
val coldFlow: Flow<Int> = flow {
    println("Flow 시작")   // collect 할 때마다 실행됨
    emit(1)
    emit(2)
    emit(3)
}

// 구독자마다 독립적으로 실행
coldFlow.collect { println("구독자1: $it") }
// 출력: Flow 시작 → 1 → 2 → 3

coldFlow.collect { println("구독자2: $it") }
// 출력: Flow 시작 → 1 → 2 → 3 (처음부터 다시 시작)
```

### Hot Stream (SharedFlow, StateFlow)

```kotlin
val hotFlow = MutableSharedFlow<Int>()

// 구독자가 없어도 이미 값을 emit 중
scope.launch { hotFlow.emit(1) }

// 나중에 구독하면 이미 지나간 값은 받지 못함
hotFlow.collect { println(it) }   // 구독 이후의 값만 수신
```

| 항목 | Cold (Flow) | Hot (SharedFlow/StateFlow) |
|------|------------|--------------------------|
| 실행 시점 | collect 시 시작 | 독립적으로 실행 |
| 구독자 수 | 구독자마다 독립 실행 | 모든 구독자가 공유 |
| 지난 값 | 항상 처음부터 | 구독 이후 값만 |
| 주 용도 | 단발 데이터, API 응답 | 상태, 이벤트 브로드캐스트 |

---

## 3. Flow 빌더 — flow { }

`flow { }` 블록 안에서 `emit()`으로 값을 하나씩 방출합니다.

```kotlin
// 기본 flow 빌더
val numberFlow: Flow<Int> = flow {
    for (i in 1..5) {
        delay(100)   // 비동기 대기 (suspend 가능)
        emit(i)      // 값 방출
    }
}

// 무한 스트림
val tickFlow: Flow<Long> = flow {
    var count = 0L
    while (true) {
        delay(1000)
        emit(count++)
    }
}
```

### 다양한 Flow 빌더

```kotlin
// flowOf — 고정 값
val flow1: Flow<Int> = flowOf(1, 2, 3, 4, 5)

// asFlow — 컬렉션/범위 변환
val flow2: Flow<Int> = listOf(1, 2, 3).asFlow()
val flow3: Flow<Int> = (1..10).asFlow()

// channelFlow — 다른 코루틴에서 emit
val flow4: Flow<Int> = channelFlow {
    launch { send(1) }   // 다른 코루틴에서 send
    launch { send(2) }
}

// emptyFlow — 즉시 완료
val flow5: Flow<Nothing> = emptyFlow()
```

---

## 4. collect — Flow 소비

`collect`는 **Flow를 구독하고 값을 소비**하는 종단 연산자입니다. `suspend fun`입니다.

```kotlin
val scope = CoroutineScope(Dispatchers.Main)

scope.launch {
    numberFlow.collect { value ->
        println("받은 값: $value")
    }
}
// 출력:
// 받은 값: 1
// 받은 값: 2
// ...
// 받은 값: 5
```

### collect의 특성

```kotlin
// collect는 Flow가 완료될 때까지 현재 코루틴을 중단
viewModelScope.launch {
    println("collect 시작")

    simpleFlow.collect { value ->
        println("값: $value")
    }

    println("collect 완료")   // Flow 완료 후 실행
}
```

---

## 5. 중간 연산자 (Intermediate Operators)

중간 연산자는 **Flow를 변환하지만 새 Flow를 반환**합니다. `collect` 전까지는 실행되지 않습니다.

### map — 값 변환

```kotlin
(1..5).asFlow()
    .map { it * it }          // 1 → 4 → 9 → 16 → 25
    .collect { println(it) }
```

### filter — 조건 필터

```kotlin
(1..10).asFlow()
    .filter { it % 2 == 0 }   // 짝수만: 2, 4, 6, 8, 10
    .collect { println(it) }
```

### transform — 유연한 변환 (map + filter 합친 것)

```kotlin
(1..5).asFlow()
    .transform { value ->
        if (value % 2 == 0) {
            emit("짝수: $value")    // 조건에 따라 emit 횟수 조절 가능
            emit("(${value * 2})")  // 여러 번 emit 가능
        }
    }
    .collect { println(it) }
// 출력: 짝수: 2, (4), 짝수: 4, (8)
```

### take — 개수 제한

```kotlin
(1..100).asFlow()
    .take(3)                  // 처음 3개만
    .collect { println(it) }  // 1, 2, 3
```

### onEach — 사이드 이펙트 (로깅)

```kotlin
(1..5).asFlow()
    .onEach { println("처리 중: $it") }   // 값 변경 없이 사이드 이펙트
    .map { it * 2 }
    .collect { println("결과: $it") }
```

### distinctUntilChanged — 연속 중복 제거

```kotlin
flowOf(1, 1, 2, 2, 2, 3, 1)
    .distinctUntilChanged()
    .collect { println(it) }   // 1, 2, 3, 1
```

### debounce — 마지막 값만 (검색어 입력)

```kotlin
searchQueryFlow
    .debounce(300)   // 300ms 동안 새 값이 없으면 방출
    .filter { it.isNotBlank() }
    .collect { query -> searchRepository.search(query) }
```

---

## 6. 종단 연산자 (Terminal Operators)

종단 연산자는 **Flow를 소비하고 값을 반환**합니다. `suspend fun`입니다.

```kotlin
val flow = flowOf(1, 2, 3, 4, 5)

// collect — 모든 값 소비
flow.collect { println(it) }

// toList — List로 변환
val list: List<Int> = flow.toList()   // [1, 2, 3, 4, 5]

// toSet — Set으로 변환
val set: Set<Int> = flow.toSet()

// first — 첫 번째 값
val first: Int = flow.first()         // 1
val firstEven = flow.first { it % 2 == 0 }  // 2

// last — 마지막 값
val last: Int = flow.last()           // 5

// single — 값이 딱 하나일 때
val single: Int = flowOf(42).single() // 42 (2개 이상이면 예외)

// count — 개수
val count: Int = flow.count()         // 5
val evenCount = flow.count { it % 2 == 0 }  // 2

// reduce — 누적 연산
val sum: Int = flow.reduce { acc, value -> acc + value }  // 15

// fold — 초기값 있는 누적 연산
val sumWithInit = flow.fold(100) { acc, value -> acc + value }  // 115
```

---

## 7. flowOn — 실행 스레드 지정

`flowOn`은 **업스트림(위쪽) Flow의 실행 Dispatcher를 변경**합니다.

```kotlin
flow {
    println("emit 스레드: ${Thread.currentThread().name}")  // IO 스레드
    emit(api.fetchData())
}
.map {
    println("map 스레드: ${Thread.currentThread().name}")   // IO 스레드
    it.toDomain()
}
.flowOn(Dispatchers.IO)   // 위쪽 flow와 map이 IO에서 실행
.collect {
    println("collect 스레드: ${Thread.currentThread().name}")  // Main 스레드
    binding.tvData.text = it.name
}
```

### flowOn 동작 원리

```
[emit]  →  [map]  →  flowOn(IO)  →  [collect]
   IO           IO                      Main (호출자 Dispatcher)
   ↑                                    ↑
   flowOn 위쪽은 IO                    flowOn 아래(collect)는 원래 Dispatcher
```

### Repository에서 flowOn 사용

```kotlin
class ProductRepositoryImpl @Inject constructor(
    private val productDao: ProductDao
) : ProductRepository {

    // Room이 이미 IO를 처리하지만, 추가 변환 작업을 Default에서
    override fun observeProducts(): Flow<List<Product>> {
        return productDao.observeAll()
            .map { entities -> entities.map { it.toDomain() } }
            .flowOn(Dispatchers.Default)   // map 변환을 Default에서 실행
    }
}
```

---

## 8. 에러 처리

### catch — 업스트림 예외 처리

```kotlin
flow {
    emit(1)
    throw RuntimeException("에러 발생")
    emit(2)   // 실행 안 됨
}
.catch { e ->
    println("에러 처리: ${e.message}")
    emit(-1)   // 에러 시 대체 값 emit 가능
}
.collect { println(it) }
// 출력: 1 → 에러 처리: 에러 발생 → -1
```

### onCompletion — 완료/에러 시 공통 처리

```kotlin
flow {
    emit(1)
    emit(2)
}
.onCompletion { cause ->
    if (cause == null) println("정상 완료")
    else println("에러 완료: ${cause.message}")
}
.collect { println(it) }
// 출력: 1 → 2 → 정상 완료
```

### catch + onCompletion 함께

```kotlin
apiFlow
    .map { it.toDomain() }
    .catch { e ->
        _uiState.update { it.copy(error = e.message) }
    }
    .onCompletion {
        _uiState.update { it.copy(isLoading = false) }
    }
    .collect { data ->
        _uiState.update { it.copy(data = data) }
    }
```

---

## 9. Android 실전 패턴

### Repository — DB 실시간 관찰

```kotlin
class UserRepositoryImpl(
    private val userDao: UserDao,
    private val userApi: UserApi
) : UserRepository {

    // Room Flow — DB 변경 시 자동으로 새 값 emit
    override fun observeUser(userId: Long): Flow<User?> {
        return userDao.observeUser(userId)
            .map { it?.toDomain() }
            .flowOn(Dispatchers.IO)
    }

    // 네트워크 + DB 조합
    override fun observeProducts(): Flow<List<Product>> = flow {
        // 1. 로컬 캐시 즉시 emit
        emit(userDao.getAll().map { it.toDomain() })

        // 2. 서버에서 최신 데이터 가져와 DB 갱신
        val fresh = userApi.getProducts()
        userDao.insertAll(fresh.map { it.toEntity() })

        // 3. DB Room Flow가 자동으로 갱신 emit
    }.flowOn(Dispatchers.IO)
}
```

### ViewModel — Flow 수집

```kotlin
@HiltViewModel
class UserViewModel @Inject constructor(
    private val userRepository: UserRepository
) : ViewModel() {

    private val _uiState = MutableStateFlow(UserUiState())
    val uiState: StateFlow<UserUiState> = _uiState.asStateFlow()

    init {
        observeUser(userId = 1L)
    }

    private fun observeUser(userId: Long) {
        userRepository.observeUser(userId)
            .onEach { user ->
                _uiState.update { it.copy(user = user, isLoading = false) }
            }
            .catch { e ->
                _uiState.update { it.copy(error = e.message, isLoading = false) }
            }
            .launchIn(viewModelScope)   // collect 대신 launchIn 사용
    }
}
```

### Fragment — Flow 관찰

```kotlin
@AndroidEntryPoint
class UserFragment : Fragment(R.layout.fragment_user) {

    private val viewModel: UserViewModel by viewModels()

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)

        viewLifecycleOwner.lifecycleScope.launch {
            viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.uiState.collect { state ->
                    binding.progressBar.isVisible = state.isLoading
                    binding.tvName.text = state.user?.name ?: ""
                    state.error?.let {
                        Snackbar.make(binding.root, it, Snackbar.LENGTH_SHORT).show()
                    }
                }
            }
        }
    }
}
```

### 검색 — debounce 실전 패턴

```kotlin
@HiltViewModel
class SearchViewModel @Inject constructor(
    private val searchRepository: SearchRepository
) : ViewModel() {

    private val _query = MutableStateFlow("")
    private val _results = MutableStateFlow<List<SearchResult>>(emptyList())
    val results: StateFlow<List<SearchResult>> = _results.asStateFlow()

    init {
        _query
            .debounce(300)                    // 300ms 대기
            .filter { it.length >= 2 }        // 2글자 이상
            .distinctUntilChanged()           // 동일 쿼리 중복 제거
            .flatMapLatest { query ->          // 이전 검색 취소 후 새 검색
                searchRepository.search(query)
            }
            .onEach { _results.value = it }
            .catch { /* 에러 처리 */ }
            .launchIn(viewModelScope)
    }

    fun onQueryChanged(query: String) {
        _query.value = query
    }
}
```

---

## 10. launchIn — collect 간결하게

```kotlin
// collect를 직접 호출하는 방법
viewModelScope.launch {
    flow.collect { /* 처리 */ }
}

// launchIn으로 간결하게
flow
    .onEach { /* 처리 */ }
    .launchIn(viewModelScope)   // 내부적으로 launch + collect
```

---

## 11. 주의사항

### ❌ flow 빌더 안에서 다른 코루틴에서 emit

```kotlin
// ❌ flow 빌더는 단일 코루틴에서만 emit 가능
val badFlow = flow {
    launch { emit(1) }   // IllegalStateException
}

// ✅ channelFlow 사용
val goodFlow = channelFlow {
    launch { send(1) }
    launch { send(2) }
}
```

---

### ❌ collect를 반복 호출 — 구독자마다 독립 실행

```kotlin
// Cold Flow는 collect마다 처음부터 다시 실행
val networkFlow = flow {
    emit(api.fetchData())   // collect할 때마다 API 호출
}

// ❌ 두 번 collect → API 두 번 호출
networkFlow.collect { updateUI(it) }
networkFlow.collect { updateCache(it) }

// ✅ stateIn으로 Hot으로 변환 후 공유
val sharedFlow = networkFlow
    .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), null)
```

---

### ❌ collect를 Main 스레드에서 블로킹 방식으로 호출

```kotlin
// ❌ runBlocking으로 Main 스레드 블로킹
runBlocking { flow.collect { } }

// ✅ lifecycleScope 또는 viewModelScope 사용
viewLifecycleOwner.lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        flow.collect { }
    }
}
```

---

## 12. 정리

| 항목 | 내용 |
|------|------|
| Flow | 비동기 Cold Stream — collect 시 실행 시작 |
| `flow { }` 빌더 | `emit()`으로 값 방출, `suspend` 허용 |
| `collect` | Flow 소비 — 완료까지 코루틴 중단 |
| 중간 연산자 | `map`, `filter`, `transform`, `take`, `debounce` 등 |
| 종단 연산자 | `collect`, `toList`, `first`, `reduce` 등 |
| `flowOn` | 업스트림 Dispatcher 변경 |
| `catch` | 업스트림 예외 처리 |
| `onCompletion` | 완료/에러 공통 처리 |
| `launchIn` | `launch + collect` 간결화 |

- Flow의 핵심은 **"collect 전까지 아무것도 실행되지 않는다"** 는 Cold Stream 특성입니다.
- `flowOn`으로 emit 스레드를, `collect` 호출 스코프로 소비 스레드를 제어합니다.
- Android에서는 `repeatOnLifecycle(STARTED)` 안에서 `collect`하는 패턴을 표준으로 사용합니다.

---

## 참고

- [Kotlin Flow 공식 문서](https://kotlinlang.org/docs/flow.html)
- [Android Flow 가이드](https://developer.android.com/kotlin/flow)
- [Cold vs Hot Flow — Kotlin 공식 블로그](https://elizarov.medium.com/cold-flows-hot-channels-d74769805f9)
