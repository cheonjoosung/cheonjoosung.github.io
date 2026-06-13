---
title: (코틀린/Kotlin) Flow 연산자 완전 정리 — zip, combine, merge
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin Flow의 zip, combine, merge 연산자를 동작 원리와 타이밍 차이로 비교하고, 실전 사용 패턴을 정리합니다.
---

---

## 1. 왜 Flow 병합 연산자가 필요한가?

실제 앱에서는 **여러 데이터 소스를 동시에 관찰**해야 하는 경우가 많습니다.

```
사용자 정보 Flow + 권한 상태 Flow → UI 상태 Flow
검색어 Flow + 필터 Flow → 검색 결과 Flow
로딩 Flow + 에러 Flow + 데이터 Flow → UiState Flow
```

이런 시나리오에서 `zip`, `combine`, `merge`가 활용됩니다.

---

## 2. zip

### 동작 원리

두 Flow에서 **각각 하나씩 짝을 맞춰** 방출합니다.  
한쪽이 느리면 빠른 쪽이 기다립니다. **둘 다 값을 내야 한 쌍이 완성**됩니다.

```kotlin
val flow1 = flowOf(1, 2, 3)
val flow2 = flowOf("A", "B", "C")

flow1.zip(flow2) { number, letter ->
    "$number$letter"
}.collect { println(it) }

// 출력:
// 1A
// 2B
// 3C
```

### 타이밍 특성

```
flow1: 1 -------- 2 -------- 3
flow2: --A ----B ----C
zip:   --1A ----2B ----3C

→ 둘 다 값이 나올 때까지 대기 후 쌍을 이뤄 방출
```

### 개수가 다르면?

```kotlin
val flow1 = flowOf(1, 2, 3)
val flow2 = flowOf("A", "B")  // 2개만

flow1.zip(flow2) { n, s -> "$n$s" }
    .collect { println(it) }

// 출력:
// 1A
// 2B
// (3은 짝이 없으므로 방출 안 됨, zip이 종료)
```

짝이 없는 값은 버려지며, **더 짧은 Flow가 완료되면 전체가 완료**됩니다.

### 실전 사용

```kotlin
// 사용자 목록 + 프로필 이미지를 짝 맞춰 결합
val users: Flow<List<User>> = repo.getUsersFlow()
val avatars: Flow<List<String>> = repo.getAvatarUrlsFlow()

users.zip(avatars) { userList, avatarList ->
    userList.zip(avatarList) { user, avatar ->
        user.copy(avatarUrl = avatar)
    }
}.collect { usersWithAvatars ->
    adapter.submitList(usersWithAvatars)
}
```

```kotlin
// API 요청과 로딩 애니메이션을 동기화
val apiResult: Flow<Result<Data>> = repo.fetchData()
val minDelay: Flow<Unit> = flow { delay(500); emit(Unit) } // 최소 500ms 로딩 표시

apiResult.zip(minDelay) { result, _ -> result }
    .collect { result -> handleResult(result) }
```

---

## 3. combine

### 동작 원리

두 Flow 중 **어느 한 쪽이 새 값을 방출하면**, 나머지 Flow의 **최신 값**과 결합해 방출합니다.  
두 Flow 모두 최소 1개씩 값을 방출해야 첫 결합이 이루어집니다.

```kotlin
val flow1 = MutableStateFlow(1)
val flow2 = MutableStateFlow("A")

combine(flow1, flow2) { number, letter ->
    "$number$letter"
}.collect { println(it) }

// flow1.value = 2 → "2A" 방출
// flow2.value = "B" → "2B" 방출
// flow1.value = 3 → "3B" 방출
```

### 타이밍 특성

```
flow1: 1 -------- 2 ---------- 3
flow2: ----A ----------B --------
combine: ---1A --- 2A - 2B -- 3B

→ 어느 한 쪽이 바뀌면 즉시 최신 값 조합 방출
```

### zip vs combine 핵심 차이

| 구분 | zip | combine |
|------|-----|---------|
| 트리거 | 양쪽 모두 새 값 | 어느 한 쪽 새 값 |
| 상대 값 | 상대의 새 값 | 상대의 최신 값 |
| 개수 불일치 | 짧은 쪽 기준으로 종료 | 관계없음 |
| 주요 용도 | 1:1 쌍 결합 | 상태 조합 |

### 실전 사용

```kotlin
// 검색어 + 필터 조합으로 실시간 검색
class SearchViewModel : ViewModel() {
    private val query = MutableStateFlow("")
    private val filter = MutableStateFlow(Filter.ALL)

    val results: StateFlow<List<Item>> = combine(query, filter) { q, f ->
        Pair(q, f)
    }.flatMapLatest { (q, f) ->
        repository.search(q, f)
    }.stateIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(5000),
        initialValue = emptyList()
    )
}
```

```kotlin
// 로딩 + 데이터 + 에러를 하나의 UiState로
data class UiState(
    val isLoading: Boolean = false,
    val data: List<Item> = emptyList(),
    val error: String? = null
)

val uiState: StateFlow<UiState> = combine(
    isLoading,
    dataFlow,
    errorFlow
) { loading, data, error ->
    UiState(loading, data, error)
}.stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), UiState())
```

---

## 4. merge

### 동작 원리

여러 Flow를 **하나의 Flow로 합칩니다**.  
각 Flow에서 값이 방출되는 즉시 그대로 내보냅니다. 결합 변환 없음.

```kotlin
val flow1 = flow {
    emit(1)
    delay(100)
    emit(2)
}

val flow2 = flow {
    delay(50)
    emit(10)
    delay(100)
    emit(20)
}

merge(flow1, flow2).collect { println(it) }

// 출력 (시간 순):
// 1    (0ms)
// 10   (50ms)
// 2    (100ms)
// 20   (150ms)
```

### 타이밍 특성

```
flow1: 1 --------- 2
flow2: ----10 ---------20
merge: 1 --10 --- 2 --20

→ 방출 순서는 실제 시간 순서 (어떤 Flow인지 구분 없음)
```

### 실전 사용

```kotlin
// 여러 소스에서 오는 이벤트를 하나로 합치기
val keyboardEvents: Flow<InputEvent> = keyboardSource.events()
val gamepadEvents: Flow<InputEvent> = gamepadSource.events()
val touchEvents: Flow<InputEvent> = touchSource.events()

merge(keyboardEvents, gamepadEvents, touchEvents)
    .collect { event -> handleInput(event) }
```

```kotlin
// 여러 에러 소스 모니터링
val networkErrors = networkMonitor.errors()
val databaseErrors = dbMonitor.errors()
val authErrors = authMonitor.errors()

merge(networkErrors, databaseErrors, authErrors)
    .collect { error -> showErrorDialog(error) }
```

```kotlin
// flattenMerge: Flow<Flow<T>>를 flatten
val multipleFlows: Flow<Flow<Int>> = flowOf(
    flowOf(1, 2),
    flowOf(3, 4),
    flowOf(5, 6)
)

multipleFlows.flattenMerge(concurrency = 2)  // 동시 수집 Flow 수 제한
    .collect { println(it) }
```

---

## 5. 세 연산자 종합 비교

| 연산자 | 입력 | 출력 타입 | 방출 타이밍 | 변환 함수 |
|--------|------|----------|------------|----------|
| `zip` | Flow A + Flow B | Pair/변환 결과 | 양쪽 모두 새 값 | 있음 |
| `combine` | Flow A + Flow B | 변환 결과 | 어느 한 쪽 새 값 | 있음 |
| `merge` | Flow A, B, C... | 원본 타입 그대로 | 각 Flow 방출 즉시 | 없음 |

```
zip:     [1,A] [2,B] [3,C]   ← 쌍 완성 대기
combine: [최신A, 최신B]      ← 어느 쪽이든 변경 시
merge:   1, A, 2, B, 3, C   ← 방출 즉시 (타입 동일)
```

---

## 6. 선택 가이드

| 상황 | 연산자 |
|------|--------|
| 두 Flow의 값을 1:1로 쌍 맞추기 | `zip` |
| 여러 상태 중 하나라도 바뀌면 UI 갱신 | `combine` |
| 여러 이벤트 소스를 하나로 모으기 | `merge` |
| 검색어 + 필터 + 정렬 조합 | `combine` |
| 페이지 데이터를 순서대로 결합 | `zip` |
| 여러 에러/이벤트 스트림 통합 | `merge` |

---

## 7. 주의 사항

```kotlin
// combine은 초기값이 있어야 첫 방출 가능
val a = MutableStateFlow(0)   // 초기값 있음
val b = flow { delay(1000); emit(1) }  // 초기값 없음

combine(a, b) { x, y -> x + y }
    .collect { println(it) }
// b가 1000ms 후 첫 값을 방출하기 전까지는 아무것도 나오지 않음
// → StateFlow나 초기값 있는 Flow를 사용하는 것이 안전
```

```kotlin
// merge는 타입이 동일해야 함
val intFlow: Flow<Int> = flowOf(1, 2)
val stringFlow: Flow<String> = flowOf("A", "B")

// merge(intFlow, stringFlow)  // 컴파일 에러 — 타입 불일치
// 필요하다면 공통 타입으로 변환 후 merge
merge(intFlow.map { it.toString() }, stringFlow)
    .collect { println(it) }
```

---

## 8. 정리

- **`zip`**: 두 Flow의 값을 1:1 쌍으로 결합, 짧은 쪽 기준으로 종료
- **`combine`**: 어느 한 쪽이 바뀌면 상대의 최신 값과 결합, 상태 관리에 최적
- **`merge`**: 여러 Flow를 하나로 합치기, 타입 동일, 변환 없음

UI 상태 관리에는 `combine`, 이벤트 통합에는 `merge`, 1:1 매핑에는 `zip`을 기본으로 선택하세요.
