---
title: (코틀린/Kotlin) flatMapLatest vs flatMapMerge vs flatMapConcat 완전 비교
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin Flow의 flatMapLatest, flatMapMerge, flatMapConcat 세 연산자를 동작 원리와 동시성 처리 방식으로 비교하고, 실전 선택 기준을 정리합니다.
---

---

## 1. flatMap이란?

`flatMap`은 **각 값을 Flow로 변환한 뒤, 그 Flow들을 하나로 합치는** 연산자입니다.

```kotlin
// 기본 아이디어
userIdFlow
    .flatMap { userId ->
        repository.getUserFlow(userId)  // 각 userId → Flow<User>
    }
    .collect { user -> /* 결과 처리 */ }
```

문제는 **이전 Flow가 완료되기 전에 새 값이 오면** 어떻게 처리할지입니다.  
이 동작 방식의 차이가 `flatMapLatest`, `flatMapMerge`, `flatMapConcat`의 핵심입니다.

---

## 2. flatMapLatest

### 동작 원리

새 값이 오면 **이전 Flow를 즉시 취소하고 새 Flow를 시작**합니다.  
항상 최신 값의 결과만 유지합니다.

```kotlin
flow {
    emit("A")
    delay(100)
    emit("B")
    delay(100)
    emit("C")
}.flatMapLatest { value ->
    flow {
        println("시작: $value")
        delay(200)  // A, B는 200ms 이전에 취소됨
        emit("결과: $value")
    }
}.collect { println(it) }

// 출력:
// 시작: A
// 시작: B  ← A 취소
// 시작: C  ← B 취소
// 결과: C  ← C만 완료
```

### 타이밍 다이어그램

```
업스트림:  A ----B ----C
flatMapLatest:
  A: [====취소====]
  B:      [====취소====]
  C:           [============] → 결과C 방출
```

### 실전 사용 — 검색창

```kotlin
class SearchViewModel(private val repo: SearchRepository) : ViewModel() {

    private val _query = MutableStateFlow("")

    val searchResults: StateFlow<List<Item>> = _query
        .debounce(300)
        .distinctUntilChanged()
        .flatMapLatest { query ->
            if (query.isEmpty()) flowOf(emptyList())
            else repo.search(query)  // 이전 검색 자동 취소
        }
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5000),
            initialValue = emptyList()
        )

    fun onQueryChanged(query: String) {
        _query.value = query
    }
}
```

검색어가 바뀔 때마다 이전 검색 요청을 취소하고 새 요청을 보냅니다.  
`debounce(300)`과 조합하면 불필요한 API 호출을 최소화할 수 있습니다.

---

## 3. flatMapMerge

### 동작 원리

새 값이 와도 이전 Flow를 취소하지 않고 **모두 동시에 실행**합니다.  
결과는 완료되는 순서대로 방출됩니다 (순서 보장 없음).

```kotlin
flow {
    emit("A")
    delay(100)
    emit("B")
    delay(100)
    emit("C")
}.flatMapMerge { value ->
    flow {
        delay(when (value) {
            "A" -> 300
            "B" -> 100
            else -> 200
        })
        emit("결과: $value")
    }
}.collect { println(it) }

// 출력 (완료 순서):
// 결과: B  (100ms 후 완료)
// 결과: C  (200ms 후 완료)
// 결과: A  (300ms 후 완료)
```

### 동시성 제한

```kotlin
// concurrency 파라미터로 동시 실행 수 제한 가능
flowOf(1, 2, 3, 4, 5)
    .flatMapMerge(concurrency = 2) { id ->
        repo.fetchItem(id)
    }
    .collect { item -> handleItem(item) }
// 최대 2개의 Flow만 동시에 실행
```

### 실전 사용 — 병렬 다운로드

```kotlin
// 여러 이미지 URL을 병렬로 다운로드
fun downloadImages(urls: List<String>): Flow<Bitmap> = flowOf(*urls.toTypedArray())
    .flatMapMerge(concurrency = 4) { url ->
        flow { emit(imageLoader.download(url)) }
    }

downloadImages(imageUrls)
    .collect { bitmap -> displayImage(bitmap) }
```

```kotlin
// 여러 사용자 데이터를 병렬 조회
userIds.asFlow()
    .flatMapMerge { id ->
        repo.getUserFlow(id)
    }
    .collect { user -> processUser(user) }
```

---

## 4. flatMapConcat

### 동작 원리

**이전 Flow가 완료된 후에만 다음 Flow를 시작**합니다.  
순차 실행, 순서 보장, 동시 실행 없음.

```kotlin
flow {
    emit("A")
    delay(100)
    emit("B")
    delay(100)
    emit("C")
}.flatMapConcat { value ->
    flow {
        println("시작: $value")
        delay(200)
        emit("결과: $value")
        println("완료: $value")
    }
}.collect { println(it) }

// 출력:
// 시작: A
// 완료: A
// 결과: A
// 시작: B
// 완료: B
// 결과: B
// 시작: C
// 완료: C
// 결과: C
```

### 타이밍 다이어그램

```
업스트림: A ----B ----C
flatMapConcat:
  A: [======] → 결과A
  B:          [======] → 결과B  (A 완료 후 시작)
  C:                   [======] → 결과C  (B 완료 후 시작)
```

### 실전 사용 — 순서 보장 처리

```kotlin
// 순서가 중요한 DB 작업
fun processOrdersSequentially(orders: List<Order>): Flow<Result> =
    orders.asFlow()
        .flatMapConcat { order ->
            flow {
                val result = database.processOrder(order)  // 순서대로 처리
                emit(result)
            }
        }
```

```kotlin
// 단계별 초기화 (다음 단계는 이전 단계 완료 후 시작)
val initSteps = listOf(
    { initDatabase() },
    { loadUserPreferences() },
    { setupNetworkLayer() }
)

initSteps.asFlow()
    .flatMapConcat { step ->
        flow { emit(step()) }
    }
    .collect { println("초기화 단계 완료: $it") }
```

---

## 5. 세 연산자 종합 비교

| 구분 | flatMapLatest | flatMapMerge | flatMapConcat |
|------|--------------|--------------|---------------|
| 이전 Flow 처리 | 취소 | 유지 (동시 실행) | 완료 대기 |
| 동시 실행 수 | 1 | N (설정 가능) | 1 |
| 결과 순서 | 최신만 | 완료 순 (보장 없음) | 입력 순 보장 |
| 주요 용도 | 검색, 자동완성 | 병렬 다운로드, 독립 요청 | 순서 중요한 작업 |

---

## 6. 타이밍 비교 시각화

```
업스트림: 1 ──── 2 ──── 3
각 값의 처리 시간: 각각 300ms

flatMapLatest:
  1: [===취소===]
  2:      [===취소===]
  3:           [==========] → 결과3만 방출

flatMapMerge:
  1: [==========] → 결과1
  2:      [==========] → 결과2  (동시 실행)
  3:           [==========] → 결과3  (동시 실행)
  완료 순서가 방출 순서

flatMapConcat:
  1: [==========] → 결과1
  2:             [==========] → 결과2  (1 완료 후 시작)
  3:                         [==========] → 결과3  (2 완료 후 시작)
  총 소요 시간 = 300 + 300 + 300 = 900ms
```

---

## 7. 주의 사항

### flatMapLatest — 취소 처리

```kotlin
// 취소되는 Flow 안에서 취소 불가능한 작업은 주의
flow { emit(1); delay(100); emit(2) }
    .flatMapLatest { value ->
        flow {
            withContext(NonCancellable) {
                // DB write: 취소되면 안 되는 작업은 NonCancellable로 보호
                database.save(value)
            }
            emit(value)
        }
    }
```

### flatMapMerge — 순서 주의

```kotlin
// 결과 순서가 보장되지 않음
// 순서가 중요하다면 flatMapConcat 또는 결과에 인덱스 추가
listOf(3, 1, 2).asFlow()
    .flatMapMerge { delay ->
        flow {
            delay(delay * 100L)
            emit(delay)
        }
    }
    .collect { println(it) }
// 출력: 1, 2, 3 (입력 순서 아님, 완료 순서)
```

### flatMapConcat — 백프레셔

```kotlin
// 업스트림이 빠르게 방출하는데 처리가 느리면 지연 누적
// 필요하다면 buffer()와 함께 사용
fastProducer()
    .buffer(capacity = 10)
    .flatMapConcat { value ->
        slowProcessor(value)
    }
    .collect { handleResult(it) }
```

---

## 8. 선택 가이드

| 상황 | 연산자 |
|------|--------|
| 검색, 자동완성 — 최신 입력만 처리 | `flatMapLatest` |
| 독립적인 요청을 병렬로 처리 | `flatMapMerge` |
| 처리 순서를 보장해야 함 | `flatMapConcat` |
| 새 입력 시 진행 중인 요청 취소 원함 | `flatMapLatest` |
| 이미지/파일 병렬 다운로드 | `flatMapMerge` |
| 순차 DB 작업, 단계별 초기화 | `flatMapConcat` |

---

## 9. 정리

- **`flatMapLatest`**: 새 값 → 이전 취소, 항상 최신 결과만. 검색에 최적
- **`flatMapMerge`**: 모두 동시 실행, 완료 순으로 방출. 병렬 처리에 최적
- **`flatMapConcat`**: 순차 실행, 순서 보장. 의존성 있는 작업에 최적

대부분의 UI 검색/자동완성은 `flatMapLatest`, 독립적인 병렬 요청은 `flatMapMerge`, 순서가 중요한 비즈니스 로직은 `flatMapConcat`을 선택하세요.
