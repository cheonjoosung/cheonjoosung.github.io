---
title: SharingStarted 전략 완전 정리 — WhileSubscribed, Eagerly, Lazily
tags: [ Kotlin ]
style: fill
color: dark
description: stateIn/shareIn의 SharingStarted 전략 3가지(WhileSubscribed, Eagerly, Lazily)를 비교하고 실전에서 올바르게 선택하는 방법을 정리합니다.
---

---

## 1. SharingStarted란?

`stateIn`과 `shareIn`은 Cold Flow를 Hot Flow로 변환하는 함수입니다.  
이때 **언제 업스트림 Flow 수집을 시작하고 중단할지**를 결정하는 것이 `SharingStarted` 전략입니다.

```kotlin
val hotFlow = coldFlow.stateIn(
    scope = viewModelScope,
    started = SharingStarted.WhileSubscribed(5000), // 전략 지정
    initialValue = emptyList()
)
```

전략 선택이 잘못되면 **불필요한 리소스 낭비** 또는 **UI 업데이트 누락**이 발생할 수 있습니다.

---

## 2. 세 가지 전략 한눈에 비교

| 전략 | 업스트림 시작 | 업스트림 중단 | 재구독 시 |
|------|--------------|--------------|----------|
| `WhileSubscribed` | 첫 구독자 등장 시 | 마지막 구독자 해제 후 N ms | 새 업스트림 시작 |
| `Eagerly` | scope 생성 즉시 | scope 취소 시 | 기존 값 유지 |
| `Lazily` | 첫 구독자 등장 시 | scope 취소 시 | 기존 값 유지 |

---

## 3. WhileSubscribed

### 동작 원리

```
구독자 등장 → 업스트림 시작
마지막 구독자 해제 → (stopTimeout ms 후) 업스트림 중단
새 구독자 등장 → 업스트림 재시작
```

```kotlin
SharingStarted.WhileSubscribed(
    stopTimeoutMillis = 5000,       // 구독자 0명 후 업스트림 중단까지 대기 시간 (기본 0)
    replayExpirationMillis = Long.MAX_VALUE // 캐시 만료 시간 (기본 무한)
)
```

### 실전 사용

```kotlin
class MainViewModel(
    private val userRepository: UserRepository
) : ViewModel() {

    val userList: StateFlow<List<User>> = userRepository.getUsersFlow()
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5000),
            initialValue = emptyList()
        )
}
```

### stopTimeoutMillis = 5000 인 이유

화면 회전 시 Activity가 destroy → recreate 됩니다.  
이 과정에서 구독자가 잠깐 0명이 되는데, 5초 여유를 주면 **업스트림을 불필요하게 재시작하지 않습니다.**

```
Activity destroy → 구독자 0명 → (5초 카운트) → 5초 내 재구독 → 업스트림 유지!
                                               → 5초 초과       → 업스트림 중단
```

### 언제 사용

- **네트워크 요청, DB 쿼리** 등 비용이 큰 업스트림
- 백그라운드 진입 시 리소스를 아끼고 싶을 때
- Android ViewModel에서 가장 권장되는 전략

---

## 4. Eagerly

### 동작 원리

```
scope 생성 즉시 → 업스트림 시작
구독자 유무 무관 → 항상 실행
scope 취소 → 업스트림 중단
```

```kotlin
val hotFlow = coldFlow.stateIn(
    scope = viewModelScope,
    started = SharingStarted.Eagerly,
    initialValue = 0
)
```

### 특징

- 구독자가 없어도 업스트림이 계속 실행됨
- 구독자가 나중에 붙으면 **최신 값부터** 받음 (과거 이벤트 누락)
- `shareIn`에서 `replay = 0`이면 구독 전 이벤트는 전혀 못 받음

### 언제 사용

- **앱 시작 시 즉시 데이터 수집이 필요**한 경우
- 업스트림 비용이 낮고 항상 최신 상태를 유지해야 할 때
- 구독자 유무와 관계없이 상태를 추적해야 하는 경우

```kotlin
// 예: 앱 시작 시 위치 추적
val locationFlow = locationSource.asFlow()
    .stateIn(
        scope = applicationScope,    // Application 레벨 scope
        started = SharingStarted.Eagerly,
        initialValue = Location.UNKNOWN
    )
```

---

## 5. Lazily

### 동작 원리

```
첫 구독자 등장 → 업스트림 시작
구독자가 0명이 되어도 → 업스트림 유지 (scope 취소까지)
scope 취소 → 업스트림 중단
```

```kotlin
val hotFlow = coldFlow.stateIn(
    scope = viewModelScope,
    started = SharingStarted.Lazily,
    initialValue = 0
)
```

### Eagerly vs Lazily 비교

```
Eagerly: scope 생성 즉시 시작 → 구독자 없어도 실행
Lazily:  첫 구독자 등장 시 시작 → 이후 구독자 없어도 계속 실행
```

### 언제 사용

- 업스트림 시작 비용이 크지만 **한 번 시작하면 유지**하고 싶을 때
- 나중에 구독자가 붙더라도 **이전 이벤트를 놓치면 안 될** 때
- 단, WhileSubscribed보다 리소스 효율이 낮음

---

## 6. stateIn vs shareIn에서의 전략 차이

```kotlin
// stateIn: StateFlow 생성 (항상 최신 값 1개 보관)
val state: StateFlow<T> = flow.stateIn(
    scope = viewModelScope,
    started = SharingStarted.WhileSubscribed(5000),
    initialValue = defaultValue
)

// shareIn: SharedFlow 생성 (replay 설정으로 캐시 조절)
val shared: SharedFlow<T> = flow.shareIn(
    scope = viewModelScope,
    started = SharingStarted.WhileSubscribed(5000),
    replay = 1
)
```

`stateIn`은 내부적으로 `replay = 1`인 SharedFlow이므로 나중에 구독해도 최신 값을 받습니다.  
`shareIn`의 `replay = 0`이면 구독 전 이벤트는 완전히 누락됩니다.

---

## 7. 커스텀 SharingStarted

특수한 경우 직접 구현할 수도 있습니다.

```kotlin
object SharingStartedWhileVisible : SharingStarted {
    override fun command(subscriptionCount: StateFlow<Int>): Flow<SharingCommand> =
        subscriptionCount
            .map { count ->
                if (count > 0) SharingCommand.START
                else SharingCommand.STOP_AND_RESET_REPLAY_CACHE
            }
            .distinctUntilChanged()
}
```

---

## 8. 전략 선택 가이드

| 상황 | 권장 전략 |
|------|----------|
| Android ViewModel + 네트워크/DB | `WhileSubscribed(5000)` |
| 앱 시작 시 즉시 데이터 필요 | `Eagerly` |
| 첫 구독 전까지 지연, 이후 계속 유지 | `Lazily` |
| 이벤트 누락 없이 재구독 | `WhileSubscribed` + `replay` |
| 백그라운드 리소스 최소화 | `WhileSubscribed` |

---

## 9. 실전 예제 — ViewModel 패턴

```kotlin
class ProductViewModel(
    private val repo: ProductRepository
) : ViewModel() {

    // 네트워크 요청: WhileSubscribed로 리소스 절약
    val products: StateFlow<List<Product>> = repo.getProductsFlow()
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5000),
            initialValue = emptyList()
        )

    // 로컬 DB: Lazily로 첫 접근 시에만 시작
    val favorites: StateFlow<List<Product>> = repo.getFavoritesFlow()
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.Lazily,
            initialValue = emptyList()
        )
}
```

```kotlin
// Fragment에서 수집
viewLifecycleOwner.lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.products.collect { products ->
            adapter.submitList(products)
        }
    }
}
```

---

## 10. 정리

- **`WhileSubscribed(5000)`**: 구독자 있을 때만 실행, Android ViewModel 기본값
- **`Eagerly`**: scope 생성 즉시 실행, 항상 최신 상태 필요 시
- **`Lazily`**: 첫 구독 후 계속 유지, 이벤트 누락 방지 + 지연 시작

대부분의 Android 앱에서는 **`WhileSubscribed(5000)`** + `stateIn`이 가장 안전하고 효율적인 선택입니다.
