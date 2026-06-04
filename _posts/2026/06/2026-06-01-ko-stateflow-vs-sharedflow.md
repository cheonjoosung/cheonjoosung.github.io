---
title: (Kotlin/코틀린) StateFlow vs SharedFlow 완전 비교
tags: [ Kotlin, Android ]
style: fill
color: dark
description: StateFlow와 SharedFlow의 차이를 개념부터 실전까지 완전히 정리합니다. Hot Stream 특성, replay·버퍼 설정, ViewModel 패턴, UI 이벤트 처리까지 다룹니다.
---

## 개요

- Kotlin Flow 중 **Hot Stream** 에 해당하는 `StateFlow`와 `SharedFlow`를 비교합니다.
- 이 글에서는 다음을 설명합니다.
  - Cold Stream vs Hot Stream 차이
  - `StateFlow` — 상태(State) 보관용 Hot Flow
  - `SharedFlow` — 이벤트(Event) 전파용 Hot Flow
  - 두 Flow의 핵심 차이 비교
  - Android ViewModel에서의 실전 패턴

---

## 1. Cold Stream vs Hot Stream

```kotlin
// Cold Stream — collect 시작 시 데이터 생성, 구독자마다 독립 실행
val coldFlow = flow {
    println("데이터 생성 시작")
    emit(1)
    emit(2)
    emit(3)
}

// 2명이 구독 → 각각 독립적으로 "데이터 생성 시작" 출력
coldFlow.collect { println("A: $it") }
coldFlow.collect { println("B: $it") }
```

```kotlin
// Hot Stream — 구독 여부와 무관하게 데이터 흐름, 구독자 간 공유
val hotFlow = MutableStateFlow(0)

// 2명이 구독 → 같은 데이터 스트림 공유
hotFlow.collect { println("A: $it") }
hotFlow.collect { println("B: $it") }

hotFlow.value = 1  // A, B 모두 1을 받음
```

| 항목 | Cold Flow | Hot Flow (StateFlow/SharedFlow) |
|------|-----------|--------------------------------|
| 데이터 생성 시점 | collect 시점 | 독립적으로 실행 |
| 구독자 간 공유 | ❌ 각자 독립 | ✅ 공유 |
| 구독자 없을 때 | 실행 안 함 | 계속 실행 가능 |
| 대표 사용처 | API 호출, DB 조회 | UI 상태, 이벤트 |

---

## 2. StateFlow

`StateFlow`는 **항상 하나의 최신 값을 보관**하는 Hot Flow입니다.

```kotlin
// MutableStateFlow — 값 변경 가능
val _uiState = MutableStateFlow(UiState())
val uiState: StateFlow<UiState> = _uiState.asStateFlow()

// 값 읽기
val current = uiState.value

// 값 변경
_uiState.value = UiState(isLoading = true)

// update — thread-safe 변경
_uiState.update { it.copy(isLoading = false, data = newData) }
```

### StateFlow 핵심 특성

```kotlin
val flow = MutableStateFlow(0)

// ① 항상 초기값 필요
val flow1 = MutableStateFlow(0)         // ✅
// val flow2 = MutableStateFlow<Int>()  // ❌ 컴파일 에러

// ② 중복 값 무시 (distinctUntilChanged 내장)
flow.value = 1  // emit
flow.value = 1  // 무시 — 같은 값이므로 collect 호출 안 됨
flow.value = 2  // emit

// ③ 새 구독자는 최신 값 즉시 수신 (replay = 1)
flow.value = 42
flow.collect { println(it) }  // 즉시 42 출력 (이전 방출이었어도)
```

### ViewModel에서 StateFlow 패턴

```kotlin
data class HomeUiState(
    val isLoading: Boolean = false,
    val users: List<User> = emptyList(),
    val error: String? = null
)

class HomeViewModel(private val userRepository: UserRepository) : ViewModel() {

    private val _uiState = MutableStateFlow(HomeUiState())
    val uiState: StateFlow<HomeUiState> = _uiState.asStateFlow()

    fun loadUsers() {
        viewModelScope.launch {
            _uiState.update { it.copy(isLoading = true, error = null) }

            userRepository.getUsers()
                .onSuccess { users ->
                    _uiState.update { it.copy(isLoading = false, users = users) }
                }
                .onFailure { e ->
                    _uiState.update { it.copy(isLoading = false, error = e.message) }
                }
        }
    }
}

// Fragment에서 수집
viewLifecycleOwner.lifecycleScope.launch {
    viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.uiState.collect { state ->
            binding.progressBar.isVisible = state.isLoading
            adapter.submitList(state.users)
            state.error?.let { showError(it) }
        }
    }
}
```

---

## 3. SharedFlow

`SharedFlow`는 **여러 구독자에게 이벤트를 방송(Broadcast)** 하는 Hot Flow입니다.  
초기값이 없고, 같은 값을 연속으로 emit해도 모두 전달됩니다.

```kotlin
// MutableSharedFlow 생성
val _events = MutableSharedFlow<UiEvent>()
val events: SharedFlow<UiEvent> = _events.asSharedFlow()

// 이벤트 방출
viewModelScope.launch {
    _events.emit(UiEvent.ShowSnackbar("저장 완료"))
}

// tryEmit — suspend 없이 즉시 방출 시도 (버퍼 여유 있을 때만 성공)
_events.tryEmit(UiEvent.NavigateTo("detail"))
```

### SharedFlow 생성 파라미터

```kotlin
MutableSharedFlow<T>(
    replay      = 0,   // 새 구독자에게 재전송할 이전 이벤트 수 (기본 0)
    extraBufferCapacity = 0,  // 추가 버퍼 크기
    onBufferOverflow = BufferOverflow.SUSPEND  // 버퍼 초과 시 동작
)
```

```kotlin
// replay = 1 — 새 구독자가 마지막 이벤트 1개 즉시 수신
val flow = MutableSharedFlow<String>(replay = 1)
flow.emit("Hello")

// 나중에 구독한 구독자도 "Hello" 수신
flow.collect { println(it) }  // "Hello" 출력

// replay = 0 (기본) — 구독 이전 이벤트 수신 불가
val flow2 = MutableSharedFlow<String>()
flow2.emit("Hello")  // 구독자 없으면 유실

flow2.collect { println(it) }  // 이후 emit된 이벤트만 수신
```

```kotlin
// BufferOverflow 전략
MutableSharedFlow<Int>(
    extraBufferCapacity = 64,
    onBufferOverflow = BufferOverflow.DROP_OLDEST   // 오래된 것 버림
)

MutableSharedFlow<Int>(
    extraBufferCapacity = 64,
    onBufferOverflow = BufferOverflow.DROP_LATEST   // 최신 것 버림
)

MutableSharedFlow<Int>(
    onBufferOverflow = BufferOverflow.SUSPEND       // 버퍼 찰 때까지 대기 (기본)
)
```

### ViewModel에서 SharedFlow 패턴 — 단발성 이벤트

```kotlin
sealed class UiEvent {
    data class ShowSnackbar(val message: String) : UiEvent()
    data class NavigateTo(val route: String) : UiEvent()
    object ShowLoginDialog : UiEvent()
}

class HomeViewModel : ViewModel() {

    private val _uiState = MutableStateFlow(HomeUiState())
    val uiState: StateFlow<HomeUiState> = _uiState.asStateFlow()

    // 단발성 이벤트 — SharedFlow 사용
    private val _uiEvent = MutableSharedFlow<UiEvent>()
    val uiEvent: SharedFlow<UiEvent> = _uiEvent.asSharedFlow()

    fun onSaveClick() {
        viewModelScope.launch {
            val success = repository.save()
            if (success) {
                _uiEvent.emit(UiEvent.ShowSnackbar("저장 완료"))
            } else {
                _uiEvent.emit(UiEvent.ShowSnackbar("저장 실패"))
            }
        }
    }

    fun onLoginRequired() {
        viewModelScope.launch {
            _uiEvent.emit(UiEvent.ShowLoginDialog)
        }
    }
}

// Fragment
viewLifecycleOwner.lifecycleScope.launch {
    viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.uiEvent.collect { event ->
            when (event) {
                is UiEvent.ShowSnackbar -> showSnackbar(event.message)
                is UiEvent.NavigateTo   -> navigate(event.route)
                UiEvent.ShowLoginDialog -> showLoginDialog()
            }
        }
    }
}
```

---

## 4. StateFlow vs SharedFlow 핵심 차이

| 항목 | StateFlow | SharedFlow |
|------|-----------|------------|
| 초기값 | ✅ 필수 | ❌ 없음 |
| 현재 값 접근 | ✅ `.value` | ❌ 없음 |
| 중복 값 emit | ❌ 무시 (distinctUntilChanged) | ✅ 모두 전달 |
| 새 구독자 최신값 | ✅ 즉시 수신 (replay=1) | 설정에 따라 다름 (기본 replay=0) |
| replay 설정 | 고정 (1) | 자유 설정 (0~N) |
| 주 사용처 | UI 상태 | 단발성 이벤트 |
| LiveData 대응 | LiveData 대체 | SingleLiveEvent 대체 |

---

## 5. 언제 무엇을 쓸까

```kotlin
// ✅ StateFlow — UI 상태처럼 "현재 어떤 상태인가"를 표현할 때
data class ProfileUiState(
    val name: String = "",
    val profileImageUrl: String = "",
    val isFollowing: Boolean = false
)
private val _uiState = MutableStateFlow(ProfileUiState())

// ✅ SharedFlow — "어떤 일이 발생했다"는 단발 이벤트를 전달할 때
sealed class ProfileEvent {
    object FollowSuccess : ProfileEvent()
    object FollowFailed : ProfileEvent()
    data class ShowToast(val msg: String) : ProfileEvent()
}
private val _event = MutableSharedFlow<ProfileEvent>()
```

```kotlin
// ❌ 잘못된 사용 — StateFlow로 단발 이벤트 처리 시 문제 발생
private val _snackbarState = MutableStateFlow<String?>(null)

// 화면 회전 시 이미 소비된 이벤트가 다시 emit됨
// → null 처리 로직 필요, 복잡성 증가

// ✅ 올바른 사용 — 단발 이벤트는 SharedFlow
private val _snackbarEvent = MutableSharedFlow<String>()
viewModelScope.launch { _snackbarEvent.emit("저장 완료") }
// collect 전에 방출되면 유실되지만, Channel로 대안 가능
```

---

## 6. Channel vs SharedFlow — 이벤트 처리 비교

단발성 이벤트 처리 시 `Channel`과 `SharedFlow`를 비교합니다.

```kotlin
// Channel — 단일 소비자, 구독 전 이벤트도 버퍼에 보존
private val _eventChannel = Channel<UiEvent>(Channel.BUFFERED)
val eventFlow = _eventChannel.receiveAsFlow()

viewModelScope.launch {
    _eventChannel.send(UiEvent.ShowSnackbar("메시지"))
}

// SharedFlow (replay=0) — 다중 소비자 가능, 구독 전 이벤트 유실 가능
private val _event = MutableSharedFlow<UiEvent>()
```

| 항목 | Channel | SharedFlow |
|------|---------|------------|
| 소비자 수 | 단일 (1명이 소비) | 다중 (여러 명 동시 수신) |
| 구독 전 이벤트 | 버퍼에 보존 | 기본 유실 (replay로 조정) |
| 소비 방식 | 1번만 소비 | 모든 구독자에게 전달 |
| 주 사용처 | ViewModel → 단일 View | ViewModel → 여러 View |

---

## 7. stateIn으로 Cold Flow를 StateFlow로 변환

```kotlin
class HomeViewModel(private val repository: UserRepository) : ViewModel() {

    // Cold Flow (DB/API) → StateFlow로 변환
    val users: StateFlow<List<User>> = repository.observeUsers()
        .stateIn(
            scope          = viewModelScope,
            started        = SharingStarted.WhileSubscribed(5_000),
            initialValue   = emptyList()
        )

    // Cold Flow → SharedFlow로 변환
    val recentSearches: SharedFlow<List<String>> = repository.getRecentSearches()
        .shareIn(
            scope   = viewModelScope,
            started = SharingStarted.WhileSubscribed(5_000),
            replay  = 1
        )
}
```

`SharingStarted` 전략:

| 전략 | 동작 | 사용 시점 |
|------|------|-----------|
| `WhileSubscribed(5000)` | 구독자 없으면 5초 후 업스트림 취소 | ViewModel (권장) |
| `Eagerly` | 즉시 시작, 취소 안 함 | 항상 필요한 데이터 |
| `Lazily` | 첫 구독 시작, 취소 안 함 | 한 번만 필요한 데이터 |

---

## 8. 정리

| 항목 | 내용 |
|------|------|
| StateFlow | 초기값 필수, 최신 값 보관, 중복 무시 → **UI 상태** |
| SharedFlow | 초기값 없음, replay 설정 자유, 중복 허용 → **단발 이벤트** |
| `.value` | StateFlow만 가능, 현재 값 동기 접근 |
| `replay` | StateFlow = 1 고정, SharedFlow = 자유 설정 |
| `stateIn` | Cold Flow를 StateFlow로 변환, WhileSubscribed 권장 |
| 이벤트 처리 | 단일 소비 → Channel, 다중 소비 → SharedFlow |

- UI **상태**(현재 어떤 상태인가) → `StateFlow`
- UI **이벤트**(무언가 발생했다) → `SharedFlow` 또는 `Channel`

---

## 참고

- [Kotlin Coroutines 취소 포스팅 보기](/posts/2026-05-12-ko-coroutines-cancel)
- [Job vs SupervisorJob 포스팅 보기](/posts/2026-05-13-ko-coroutines-job)
- [CoroutineExceptionHandler 포스팅 보기](/posts/2026-05-14-ko-coroutines-exception)
- [Kotlin Flow 공식 문서](https://kotlinlang.org/docs/flow.html)
