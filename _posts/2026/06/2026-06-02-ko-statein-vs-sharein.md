---
title: (Kotlin/코틀린) stateIn vs shareIn — Flow를 Hot으로 변환
tags: [ Kotlin, Android ]
style: fill
color: dark
description: Cold Flow를 Hot Stream으로 변환하는 stateIn과 shareIn의 차이, SharingStarted 전략, Android ViewModel 실전 패턴을 완전히 정리합니다.
---

## 개요

- Cold Flow를 Hot Stream(`StateFlow` / `SharedFlow`)으로 변환하는 `stateIn`과 `shareIn`을 다룹니다.
- 이 글에서는 다음을 설명합니다.
  - 왜 Cold Flow를 Hot으로 변환해야 하는가
  - `stateIn` — Cold Flow → StateFlow
  - `shareIn` — Cold Flow → SharedFlow
  - `SharingStarted` 전략 완전 비교
  - Android ViewModel 실전 패턴

---

## 1. 왜 변환이 필요한가

```kotlin
// ❌ 문제 — Cold Flow를 직접 노출할 때
class HomeViewModel(private val repository: UserRepository) : ViewModel() {

    // collect할 때마다 새로운 DB/네트워크 요청 발생
    val users: Flow<List<User>> = repository.observeUsers()
}

// Fragment A가 collect → DB 쿼리 1회
// Fragment B도 collect → DB 쿼리 또 1회
// 화면 회전 → collect 재시작 → DB 쿼리 또 1회
```

```kotlin
// ✅ 해결 — stateIn/shareIn으로 Hot Stream 변환
class HomeViewModel(private val repository: UserRepository) : ViewModel() {

    // 단 1개의 업스트림(DB 쿼리)을 구독자 모두가 공유
    val users: StateFlow<List<User>> = repository.observeUsers()
        .stateIn(
            scope        = viewModelScope,
            started      = SharingStarted.WhileSubscribed(5_000),
            initialValue = emptyList()
        )
}
```

---

## 2. stateIn

Cold Flow를 **`StateFlow`** 로 변환합니다.

```kotlin
fun <T> Flow<T>.stateIn(
    scope: CoroutineScope,
    started: SharingStarted,
    initialValue: T
): StateFlow<T>
```

```kotlin
class ProfileViewModel(
    private val repository: ProfileRepository,
    private val userId: String
) : ViewModel() {

    // DB Flow → StateFlow
    val profile: StateFlow<Profile?> = repository.observeProfile(userId)
        .stateIn(
            scope        = viewModelScope,
            started      = SharingStarted.WhileSubscribed(5_000),
            initialValue = null
        )

    // 여러 Flow를 결합 후 StateFlow로 변환
    val uiState: StateFlow<ProfileUiState> = combine(
        repository.observeProfile(userId),
        repository.observeFollowerCount(userId)
    ) { profile, followerCount ->
        ProfileUiState(profile = profile, followerCount = followerCount)
    }.stateIn(
        scope        = viewModelScope,
        started      = SharingStarted.WhileSubscribed(5_000),
        initialValue = ProfileUiState()
    )
}
```

### stateIn 특성

```kotlin
val stateFlow = coldFlow.stateIn(scope, started, initialValue)

// ① 항상 현재 값 접근 가능
val current = stateFlow.value

// ② 새 구독자는 최신 값 즉시 수신
stateFlow.collect { /* 즉시 최신 값 수신 */ }

// ③ 중복 값 무시 (distinctUntilChanged 내장)
// 같은 값이 emit되어도 구독자에게 전달되지 않음
```

---

## 3. shareIn

Cold Flow를 **`SharedFlow`** 로 변환합니다.

```kotlin
fun <T> Flow<T>.shareIn(
    scope: CoroutineScope,
    started: SharingStarted,
    replay: Int = 0
): SharedFlow<T>
```

```kotlin
class NotificationViewModel(
    private val repository: NotificationRepository
) : ViewModel() {

    // 알림 스트림 — 여러 구독자가 공유, 최신 1개 재전송
    val notifications: SharedFlow<Notification> = repository.observeNotifications()
        .shareIn(
            scope  = viewModelScope,
            started = SharingStarted.WhileSubscribed(5_000),
            replay  = 1
        )

    // 재시도 이벤트 — replay = 0 (과거 이벤트 불필요)
    val retryEvent: SharedFlow<Unit> = repository.observeRetryTrigger()
        .shareIn(
            scope   = viewModelScope,
            started = SharingStarted.Eagerly,
            replay  = 0
        )
}
```

### shareIn 특성

```kotlin
// replay = 0 (기본) — 구독 시점 이후 이벤트만 수신
val shared0 = coldFlow.shareIn(scope, SharingStarted.Eagerly, replay = 0)

// replay = 1 — 새 구독자가 마지막 이벤트 1개 즉시 수신
val shared1 = coldFlow.shareIn(scope, SharingStarted.Eagerly, replay = 1)

// replay = N — 마지막 N개 이벤트 재전송
val sharedN = coldFlow.shareIn(scope, SharingStarted.Eagerly, replay = 3)
```

---

## 4. SharingStarted 전략 완전 비교

### WhileSubscribed

```kotlin
// 구독자가 있는 동안만 업스트림 실행
// stopTimeoutMillis — 구독자 0명 후 업스트림을 중단할 때까지 대기 시간
SharingStarted.WhileSubscribed(stopTimeoutMillis = 5_000)

// 예시 — 화면 이동 시 업스트림 처리
// 1. Fragment가 collect 시작 → 업스트림(DB 쿼리) 시작
// 2. Fragment가 onStop → collect 중단 → 구독자 0명
// 3. 5초 대기 후 업스트림 취소 (화면 회전 시 불필요한 재시작 방지)
// 4. 새 Fragment가 collect → 업스트림 재시작

val uiState = repository.observeData()
    .stateIn(
        scope        = viewModelScope,
        started      = SharingStarted.WhileSubscribed(5_000),
        initialValue = UiState()
    )
```

```kotlin
// replayExpirationMillis — 구독자 0명 후 replay 캐시 유지 시간 (기본 Long.MAX_VALUE)
SharingStarted.WhileSubscribed(
    stopTimeoutMillis     = 5_000,
    replayExpirationMillis = 0     // 구독자 없어지면 캐시 즉시 제거
)
```

### Eagerly

```kotlin
// scope 생성 즉시 업스트림 시작, 취소 안 함
SharingStarted.Eagerly

// 사용 시점 — 항상 필요한 데이터, 앱 전역 스트림
val appConfig: StateFlow<AppConfig> = configRepository.observe()
    .stateIn(
        scope        = viewModelScope,
        started      = SharingStarted.Eagerly,
        initialValue = AppConfig.Default
    )
// ViewModel 생성 시 즉시 DB 구독 시작
// 구독자 없어도 업스트림 유지
```

### Lazily

```kotlin
// 첫 번째 구독자가 나타날 때 업스트림 시작, 이후 취소 안 함
SharingStarted.Lazily

// 사용 시점 — 한 번 시작하면 끝까지 필요한 데이터
val userSession: StateFlow<UserSession?> = sessionRepository.observe()
    .stateIn(
        scope        = viewModelScope,
        started      = SharingStarted.Lazily,
        initialValue = null
    )
// 처음 collect 전까지 업스트림 시작 안 함
// 한 번 시작 후 구독자 없어도 유지
```

### 전략 비교표

| 전략 | 시작 시점 | 종료 시점 | 권장 사용처 |
|------|-----------|-----------|-------------|
| `WhileSubscribed(5000)` | 첫 구독자 | 마지막 구독자 해제 후 5초 | ViewModel (일반적) |
| `Eagerly` | scope 생성 즉시 | scope 취소 시 | 항상 필요한 전역 데이터 |
| `Lazily` | 첫 구독자 | scope 취소 시 | 한 번만 필요한 데이터 |

---

## 5. stateIn vs shareIn 비교

| 항목 | stateIn | shareIn |
|------|---------|---------|
| 반환 타입 | `StateFlow<T>` | `SharedFlow<T>` |
| 초기값 | ✅ 필수 | ❌ 없음 (replay로 대체) |
| 현재 값 접근 | ✅ `.value` | ❌ 없음 |
| 중복 값 처리 | 무시 (distinctUntilChanged) | 모두 전달 |
| replay | 고정 (1) | 자유 설정 |
| 주 사용처 | UI 상태 | 이벤트·알림 스트림 |

---

## 6. Android ViewModel 실전 패턴

### 기본 패턴

```kotlin
class HomeViewModel(
    private val userRepository: UserRepository,
    private val postRepository: PostRepository
) : ViewModel() {

    // 상태 — stateIn
    val users: StateFlow<List<User>> = userRepository.observeUsers()
        .stateIn(
            scope        = viewModelScope,
            started      = SharingStarted.WhileSubscribed(5_000),
            initialValue = emptyList()
        )

    // 이벤트 스트림 — shareIn
    val newPostAlerts: SharedFlow<Post> = postRepository.observeNewPosts()
        .shareIn(
            scope   = viewModelScope,
            started = SharingStarted.WhileSubscribed(5_000),
            replay  = 0
        )
}
```

### Flow 변환 후 stateIn

```kotlin
class SearchViewModel(private val repository: SearchRepository) : ViewModel() {

    private val _query = MutableStateFlow("")

    // 검색어 입력 → debounce → API 호출 → StateFlow
    val searchResults: StateFlow<List<SearchResult>> = _query
        .debounce(300)
        .filter { it.isNotBlank() }
        .flatMapLatest { query -> repository.search(query) }
        .stateIn(
            scope        = viewModelScope,
            started      = SharingStarted.WhileSubscribed(5_000),
            initialValue = emptyList()
        )

    fun onQueryChanged(query: String) {
        _query.value = query
    }
}
```

### combine 후 stateIn

```kotlin
class DashboardViewModel(
    private val userRepo: UserRepository,
    private val statsRepo: StatsRepository
) : ViewModel() {

    data class DashboardUiState(
        val user: User? = null,
        val stats: Stats? = null,
        val isLoading: Boolean = true
    )

    val uiState: StateFlow<DashboardUiState> = combine(
        userRepo.observeCurrentUser(),
        statsRepo.observeStats()
    ) { user, stats ->
        DashboardUiState(user = user, stats = stats, isLoading = false)
    }.stateIn(
        scope        = viewModelScope,
        started      = SharingStarted.WhileSubscribed(5_000),
        initialValue = DashboardUiState()
    )
}
```

### Fragment에서 수집

```kotlin
class HomeFragment : Fragment() {

    private val viewModel: HomeViewModel by viewModels()

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)

        viewLifecycleOwner.lifecycleScope.launch {
            viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {

                // StateFlow 수집
                launch {
                    viewModel.users.collect { users ->
                        adapter.submitList(users)
                    }
                }

                // SharedFlow 수집
                launch {
                    viewModel.newPostAlerts.collect { post ->
                        showNewPostBanner(post)
                    }
                }
            }
        }
    }
}
```

---

## 7. 흔한 실수

```kotlin
// ❌ 잘못된 사용 — stateIn 없이 collect마다 중복 업스트림 실행
val users = repository.observeUsers()  // Cold Flow 그대로 노출

// Fragment A collect → DB 쿼리 1번
// Fragment B collect → DB 쿼리 또 1번

// ✅ stateIn으로 공유
val users = repository.observeUsers()
    .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), emptyList())
```

```kotlin
// ❌ 잘못된 사용 — Eagerly를 ViewModel에서 남용
// 구독자 없어도 업스트림이 계속 실행 → 불필요한 리소스 소비
val data = repository.observeData()
    .stateIn(viewModelScope, SharingStarted.Eagerly, null)

// ✅ ViewModel에서는 WhileSubscribed(5_000) 권장
val data = repository.observeData()
    .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), null)
```

```kotlin
// ❌ 잘못된 사용 — stopTimeoutMillis = 0 으로 설정
// 화면 회전 시 업스트림이 즉시 취소 후 재시작 → 불필요한 네트워크/DB 요청
SharingStarted.WhileSubscribed(0)

// ✅ 5초 여유를 두어 화면 회전·일시적 구독 해제 처리
SharingStarted.WhileSubscribed(5_000)
```

---

## 8. 정리

| 항목 | 내용 |
|------|------|
| `stateIn` | Cold Flow → StateFlow, 초기값 필수, UI 상태에 적합 |
| `shareIn` | Cold Flow → SharedFlow, replay 자유, 이벤트 스트림에 적합 |
| `WhileSubscribed(5000)` | ViewModel 기본 전략, 화면 회전 대응 |
| `Eagerly` | 즉시 시작, 전역 데이터 |
| `Lazily` | 첫 구독 시작, 한 번만 필요한 데이터 |
| 권장 패턴 | ViewModel에서 `stateIn` + `WhileSubscribed(5_000)` |

- `stateIn` / `shareIn` 으로 **업스트림 중복 실행을 방지**하고 구독자 간 데이터를 공유합니다.
- ViewModel에서는 `SharingStarted.WhileSubscribed(5_000)` 이 사실상 표준입니다.

---

## 참고

- [StateFlow vs SharedFlow 포스팅 보기](/posts/2026-06-01-ko-stateflow-vs-sharedflow)
- [Coroutines 취소 포스팅 보기](/posts/2026-05-12-ko-coroutines-cancel)
- [Kotlin Flow 공식 문서](https://kotlinlang.org/docs/flow.html)
