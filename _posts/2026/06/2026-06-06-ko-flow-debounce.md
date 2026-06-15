---
title: Flow debounce & throttleFirst 완전 정리 — 검색창 실전 적용
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin Flow의 debounce와 throttleFirst(sample) 연산자를 동작 원리와 함께 비교하고, 검색창·버튼 중복 클릭 방지 등 실전 패턴을 정리합니다.
---

---

## 1. 왜 필요한가?

사용자 입력은 연속적으로 빠르게 발생합니다.

```
검색창 입력: "k" → "ko" → "kot" → "kotl" → "kotli" → "kotlin"
버튼 클릭: 사용자가 빠르게 여러 번 클릭
스크롤 이벤트: 스크롤 중 수십 ms마다 방출
```

모든 이벤트를 처리하면 **불필요한 API 호출, 과도한 연산, UX 저하**가 발생합니다.  
이를 해결하는 두 가지 전략이 `debounce`와 `throttleFirst`입니다.

---

## 2. debounce

### 동작 원리

마지막 값 방출 후 **지정한 시간 동안 새 값이 없으면** 그 값을 전달합니다.  
연속 입력 중에는 방출을 보류하고, **입력이 멈춘 후** 한 번만 방출합니다.

```
입력: k--ko--kot--kotl--kotli--kotlin(300ms 대기)
debounce(300ms):
                                      kotlin ← 이것만 방출
```

```kotlin
val queryFlow = MutableStateFlow("")

queryFlow
    .debounce(300)
    .collect { query ->
        // 입력이 300ms 멈춘 후에만 호출됨
        searchApi(query)
    }
```

### 검색창 실전 적용

```kotlin
class SearchViewModel(
    private val repository: SearchRepository
) : ViewModel() {

    private val _query = MutableStateFlow("")
    val query: StateFlow<String> = _query.asStateFlow()

    val results: StateFlow<List<SearchItem>> = _query
        .debounce(300)                    // 300ms 정지 후 방출
        .distinctUntilChanged()           // 동일 값 중복 제거
        .filter { it.isNotBlank() }       // 빈 문자열 제거
        .flatMapLatest { query ->
            repository.search(query)      // 이전 요청 자동 취소
                .catch { emit(emptyList()) }
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

```kotlin
// Fragment에서 EditText와 연결
binding.searchEditText.addTextChangedListener { text ->
    viewModel.onQueryChanged(text.toString())
}

viewLifecycleOwner.lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.results.collect { items ->
            adapter.submitList(items)
        }
    }
}
```

---

## 3. throttleFirst (sample)

### 동작 원리

지정한 시간 동안 **처음 들어온 값만 전달**하고, 그 시간 내에 들어오는 나머지는 무시합니다.  
이벤트가 연속으로 오더라도 **첫 번째 이벤트만 즉시 처리**합니다.

```
입력: click─click─click─(1000ms)─click─click
throttleFirst(1000ms):
      click                         click
      ↑ 즉시 처리         ↑ 다음 이벤트 허용
```

### Kotlin Flow에서 구현

Flow에는 `throttleFirst`가 내장되어 있지 않아 직접 구현이 필요합니다.

```kotlin
fun <T> Flow<T>.throttleFirst(windowDuration: Long): Flow<T> = flow {
    var lastEmitTime = 0L
    collect { value ->
        val currentTime = System.currentTimeMillis()
        if (currentTime - lastEmitTime >= windowDuration) {
            lastEmitTime = currentTime
            emit(value)
        }
    }
}
```

또는 `sample` 연산자로 유사하게 구현:

```kotlin
// sample: 지정 시간마다 가장 최근 값을 방출 (throttleLast와 유사)
clickFlow
    .sample(1000)
    .collect { handleClick() }
```

### 버튼 중복 클릭 방지

```kotlin
// 버튼 클릭을 Flow로 변환
fun View.clicks(): Flow<Unit> = callbackFlow {
    setOnClickListener { trySend(Unit) }
    awaitClose { setOnClickListener(null) }
}

// ViewModel
viewLifecycleOwner.lifecycleScope.launch {
    binding.submitButton.clicks()
        .throttleFirst(1000)     // 1초 내 중복 클릭 무시
        .collect {
            viewModel.submitOrder()
        }
}
```

---

## 4. debounce vs throttleFirst 비교

| 구분 | debounce | throttleFirst |
|------|----------|---------------|
| 방출 타이밍 | 입력 멈춘 후 | 첫 이벤트 즉시 |
| 연속 입력 처리 | 마지막 값만 | 처음 값만 |
| 대기 방식 | 마지막 후 N ms 대기 | N ms 쿨다운 |
| 주요 용도 | 검색창, 자동완성 | 버튼 클릭, 결제 |

```
사용자 입력: ●●●●●●●●●●
debounce:                    ● ← 멈춘 후 마지막 값
throttleFirst: ●             ● ← 시작 즉시, 이후 쿨다운
```

---

## 5. 함께 쓰면 좋은 연산자

```kotlin
// debounce + distinctUntilChanged: 동일 값 재검색 방지
_query
    .debounce(300)
    .distinctUntilChanged()  // "kotlin" 입력 후 지웠다 다시 "kotlin" → 중복 제거
    .flatMapLatest { searchApi(it) }

// debounce + filter: 최소 글자 수 조건
_query
    .debounce(300)
    .filter { it.length >= 2 }  // 2글자 이상만 검색
    .flatMapLatest { searchApi(it) }
```

---

## 6. 정리

- **`debounce(ms)`**: 입력이 ms 동안 멈춘 후 방출 → 검색창, 폼 자동완성
- **`throttleFirst(ms)`**: ms 동안 첫 이벤트만 방출 → 버튼 클릭, 결제, 스크롤
- 검색창에는 `debounce + distinctUntilChanged + flatMapLatest` 조합이 표준
