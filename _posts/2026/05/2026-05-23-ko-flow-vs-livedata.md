---
title: (Kotlin/코틀린) Flow vs LiveData — 언제 무엇을 쓸까
tags: [ Kotlin, Android ]
style: fill
color: dark
description: Kotlin Flow와 Android LiveData의 동작 원리, 생명주기 처리, 스레드 안전성, 변환 연산자, 테스트 용이성 차이를 실전 코드와 함께 비교하고 상황별 선택 기준을 정리합니다.
---

## 개요

- Android에서 UI 상태를 관찰하는 두 축 **Flow** 와 **LiveData** 를 비교합니다.
- 둘 다 "데이터 변경을 UI에 전달"하는 역할이지만 **설계 철학과 동작 방식이 다릅니다.**
- 이 글에서는 다음을 설명합니다.
  - LiveData의 동작 원리와 특징
  - Flow의 동작 원리와 특징
  - 생명주기·스레드·변환·테스트 측면 비교
  - `StateFlow`, `SharedFlow`와 LiveData 비교
  - 실전 마이그레이션 패턴
  - 상황별 선택 기준

---

## 1. LiveData란

`LiveData`는 **Android Lifecycle을 인식하는 Observable 데이터 홀더**입니다.  
`androidx.lifecycle` 패키지에 속하며 Android 전용입니다.

```kotlin
class UserViewModel : ViewModel() {

    // MutableLiveData — 내부에서 쓰기 가능
    private val _user = MutableLiveData<User?>()
    // LiveData — 외부에는 읽기 전용으로 노출
    val user: LiveData<User?> = _user

    fun loadUser(userId: Long) {
        viewModelScope.launch {
            val result = userRepository.getUser(userId)
            _user.value = result   // 메인 스레드에서 값 변경
            // _user.postValue(result)   // 백그라운드 스레드에서 값 변경
        }
    }
}

// Fragment에서 관찰
viewModel.user.observe(viewLifecycleOwner) { user ->
    binding.tvName.text = user?.name
}
```

### LiveData의 핵심 특성

```
LiveData
├── 생명주기 자동 처리 — STARTED 이상일 때만 업데이트 전달
├── 마지막 값 유지 — 구독 시점에 최신 값 즉시 수신
├── 메인 스레드 전달 보장 — observe 콜백은 항상 Main
└── Android 전용 — LifecycleOwner 필요
```

---

## 2. Flow란

`Flow`는 **Kotlin 코루틴 기반의 비동기 Cold 스트림**입니다.  
`kotlinx.coroutines` 패키지에 속하며 **플랫폼에 독립적**입니다.

```kotlin
class UserViewModel : ViewModel() {

    private val _uiState = MutableStateFlow(UserUiState())
    val uiState: StateFlow<UserUiState> = _uiState.asStateFlow()

    fun loadUser(userId: Long) {
        viewModelScope.launch {
            userRepository.getUser(userId)
                .onSuccess { user ->
                    _uiState.update { it.copy(user = user) }
                }
        }
    }
}

// Fragment에서 관찰
viewLifecycleOwner.lifecycleScope.launch {
    viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.uiState.collect { state ->
            binding.tvName.text = state.user?.name
        }
    }
}
```

---

## 3. 생명주기 처리 비교

### LiveData — 자동으로 생명주기 인식

```kotlin
// observe() — LifecycleOwner만 넘기면 자동으로 생명주기 처리
viewModel.user.observe(viewLifecycleOwner) { user ->
    binding.tvName.text = user?.name
    // DESTROYED 시 자동 해제 — 추가 코드 없음
}
```

```
LiveData 동작:
STARTED → 업데이트 전달 ✅
STOPPED → 업데이트 전달 ❌ (무시)
DESTROYED → Observer 자동 제거
```

### Flow — repeatOnLifecycle 필요

```kotlin
// ❌ 단순 collect — 백그라운드에서도 계속 수집
lifecycleScope.launch {
    viewModel.uiState.collect { }   // onStop 이후에도 실행
}

// ✅ repeatOnLifecycle — LiveData와 동일한 생명주기 동작
viewLifecycleOwner.lifecycleScope.launch {
    viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.uiState.collect { }   // STARTED ↔ STOPPED 자동 처리
    }
}
```

| 항목 | LiveData | Flow + repeatOnLifecycle |
|------|----------|--------------------------|
| 생명주기 자동 처리 | ✅ observe()만으로 | ⚠️ repeatOnLifecycle 필요 |
| 백그라운드 수신 차단 | ✅ 자동 | ✅ (설정 시) |
| 메모리 누수 방지 | ✅ 자동 | ✅ (설정 시) |
| 코드 복잡도 | 낮음 | 보통 |

---

## 4. 스레드 안전성 비교

### LiveData — 메인 스레드 전달 보장

```kotlin
viewModelScope.launch(Dispatchers.IO) {
    val user = api.getUser(1L)   // IO 스레드

    // ✅ value — 메인 스레드에서만 호출 가능
    // _user.value = user   // 백그라운드에서 호출 시 예외

    // ✅ postValue — 어떤 스레드에서도 안전
    _user.postValue(user)
}

// observe 콜백은 항상 메인 스레드 보장
viewModel.user.observe(viewLifecycleOwner) { user ->
    binding.tvName.text = user?.name   // 메인 스레드 확정
}
```

### Flow — Dispatcher로 명시적 제어

```kotlin
viewModelScope.launch {   // Main Dispatcher
    val user = withContext(Dispatchers.IO) { api.getUser(1L) }

    // StateFlow update는 스레드 안전 (내부적으로 원자적 연산)
    _uiState.update { it.copy(user = user) }
}

// collect는 호출 스코프의 Dispatcher에서 실행
viewLifecycleOwner.lifecycleScope.launch {   // Main
    viewModel.uiState.collect { state ->
        binding.tvName.text = state.user?.name   // 메인 스레드
    }
}
```

---

## 5. 초기값 비교

```kotlin
// LiveData — 초기값 없어도 됨
private val _user = MutableLiveData<User?>()   // 초기값 null (선택)
private val _count = MutableLiveData<Int>(0)   // 초기값 지정

// observe 시 값이 없으면 콜백 호출 안 됨
viewModel.user.observe(viewLifecycleOwner) { user ->
    // user가 아직 없으면 이 콜백 호출 자체가 안 됨
}

// StateFlow — 초기값 필수
private val _uiState = MutableStateFlow(UserUiState())   // ✅ 초기값 필수
// private val _uiState = MutableStateFlow<User?>(null)  // null도 가능

// collect 시 즉시 현재 값 수신
viewModel.uiState.collect { state ->
    // 구독 즉시 초기값 수신 — 항상 호출됨
}
```

---

## 6. 변환 연산자 비교

### LiveData — 변환 연산자가 제한적

```kotlin
val user: LiveData<User?> = MutableLiveData()

// map — 값 변환
val userName: LiveData<String> = user.map { it?.name ?: "" }

// switchMap — LiveData를 반환하는 변환
val userId = MutableLiveData<Long>()
val userDetail: LiveData<User> = userId.switchMap { id ->
    repository.getUserLiveData(id)   // LiveData 반환 함수 필요
}

// distinctUntilChanged — 중복 제거
val distinctUser = user.distinctUntilChanged()

// MediatorLiveData — 여러 LiveData 합치기
val mediator = MediatorLiveData<String>().apply {
    addSource(user) { value = it?.name }
    addSource(userName) { value = it }
}
```

### Flow — 풍부한 연산자

```kotlin
val userFlow: Flow<User?> = MutableStateFlow(null)

// 기본 변환
userFlow
    .map { it?.name ?: "" }
    .filter { it.isNotBlank() }
    .distinctUntilChanged()
    .debounce(300)
    .collect { }

// 여러 Flow 결합
val nameFlow = userFlow.map { it?.name ?: "" }
val ageFlow = userFlow.map { it?.age ?: 0 }

combine(nameFlow, ageFlow) { name, age ->
    "$name ($age세)"
}.collect { println(it) }

// zip — 쌍으로 결합
val flow1 = flowOf(1, 2, 3)
val flow2 = flowOf("a", "b", "c")
flow1.zip(flow2) { num, str -> "$num$str" }
    .collect { println(it) }   // 1a, 2b, 3c

// flatMapLatest — 최신 값으로 새 Flow 시작
searchQuery
    .flatMapLatest { query -> repository.search(query) }
    .collect { results -> showResults(results) }
```

| 연산자 | LiveData | Flow |
|--------|----------|------|
| map | ✅ | ✅ |
| filter | ❌ | ✅ |
| distinctUntilChanged | ✅ | ✅ |
| debounce | ❌ | ✅ |
| combine | MediatorLiveData | ✅ combine |
| flatMap 계열 | switchMap (제한적) | ✅ flatMapLatest 등 |
| zip | ❌ | ✅ |
| retry | ❌ | ✅ |
| catch | ❌ | ✅ |

---

## 7. 에러 처리 비교

```kotlin
// LiveData — 에러 전달 수단이 없음 (별도 래핑 필요)
sealed class Result<T> {
    data class Success<T>(val data: T) : Result<T>()
    data class Error<T>(val exception: Exception) : Result<T>()
}

private val _result = MutableLiveData<Result<User>>()

// Flow — catch 연산자로 자연스럽게 처리
userRepository.getUserFlow(userId)
    .map { it.toDomain() }
    .catch { e ->
        _uiState.update { it.copy(error = e.message) }
    }
    .onCompletion {
        _uiState.update { it.copy(isLoading = false) }
    }
    .launchIn(viewModelScope)
```

---

## 8. 테스트 비교

### LiveData 테스트 — InstantTaskExecutorRule 필요

```kotlin
@get:Rule
val instantTaskRule = InstantTaskExecutorRule()   // LiveData 동기 실행

class UserViewModelTest {

    private lateinit var viewModel: UserViewModel

    @Before
    fun setup() {
        viewModel = UserViewModel(FakeUserRepository())
    }

    @Test
    fun `loadUser 성공 시 LiveData에 User 반영`() {
        viewModel.loadUser(1L)

        val user = viewModel.user.value   // InstantTaskExecutorRule로 즉시 값 조회
        assertEquals("홍길동", user?.name)
    }
}
```

### Flow 테스트 — 더 간결

```kotlin
class UserViewModelTest {

    @Test
    fun `loadUser 성공 시 uiState에 User 반영`() = runTest {
        val viewModel = UserViewModel(FakeUserRepository())

        viewModel.loadUser(1L)

        // StateFlow는 즉시 현재 값 조회 가능
        assertEquals("홍길동", viewModel.uiState.value.user?.name)
    }

    @Test
    fun `Flow 방출 순서 테스트 — Turbine`() = runTest {
        val viewModel = UserViewModel(FakeUserRepository())

        viewModel.uiState.test {
            val initial = awaitItem()   // 초기값
            assertTrue(initial.isLoading.not())

            viewModel.loadUser(1L)

            val loading = awaitItem()   // 로딩 중
            assertTrue(loading.isLoading)

            val loaded = awaitItem()    // 로딩 완료
            assertEquals("홍길동", loaded.user?.name)
        }
    }
}
```

| 항목 | LiveData | Flow (StateFlow) |
|------|----------|-----------------|
| 테스트 의존성 | InstantTaskExecutorRule | 없음 |
| Android 의존성 | 있음 | 없음 (순수 Kotlin) |
| 현재 값 조회 | `.value` (Rule 필요) | `.value` (즉시) |
| 방출 순서 검증 | 어려움 | Turbine 라이브러리 |
| 플랫폼 독립성 | ❌ Android 전용 | ✅ 순수 JVM |

---

## 9. StateFlow / SharedFlow vs LiveData

### StateFlow — LiveData와 가장 가까운 대체재

```kotlin
// LiveData와 StateFlow 1:1 대응
private val _count = MutableLiveData(0)
val count: LiveData<Int> = _count

// ↓ StateFlow로 대체
private val _count = MutableStateFlow(0)
val count: StateFlow<Int> = _count.asStateFlow()

// 값 변경
_count.value = 1           // StateFlow
_count.postValue(1)        // LiveData (백그라운드)
_count.update { it + 1 }  // StateFlow (원자적 변경 — 권장)
```

### SharedFlow — 일회성 이벤트용

```kotlin
// LiveData로 일회성 이벤트 처리 (자주 쓰는 안티패턴)
private val _showToast = MutableLiveData<String?>()
val showToast: LiveData<String?> = _showToast

// 화면 복원 시 토스트가 다시 뜨는 문제 발생
// 처리 후 null로 초기화해야 하는 번거로움

// ✅ SharedFlow로 일회성 이벤트 처리
private val _effect = MutableSharedFlow<UiEffect>()
val effect: SharedFlow<UiEffect> = _effect.asSharedFlow()

viewModelScope.launch {
    _effect.emit(UiEffect.ShowToast("저장 완료"))
    // 한 번만 전달, 화면 복원 시 재전달 없음
}
```

---

## 10. 실전 마이그레이션 — LiveData → Flow

```kotlin
// Before — LiveData
class UserViewModel : ViewModel() {
    private val _user = MutableLiveData<User?>()
    val user: LiveData<User?> = _user

    private val _isLoading = MutableLiveData(false)
    val isLoading: LiveData<Boolean> = _isLoading

    private val _error = MutableLiveData<String?>()
    val error: LiveData<String?> = _error

    fun loadUser(userId: Long) {
        viewModelScope.launch {
            _isLoading.value = true
            try {
                _user.value = userRepository.getUser(userId)
            } catch (e: Exception) {
                _error.value = e.message
            } finally {
                _isLoading.value = false
            }
        }
    }
}

// Fragment — Before
viewModel.user.observe(viewLifecycleOwner) { binding.tvName.text = it?.name }
viewModel.isLoading.observe(viewLifecycleOwner) { binding.progress.isVisible = it }
viewModel.error.observe(viewLifecycleOwner) { it?.let { showToast(it) } }
```

```kotlin
// After — StateFlow + SharedFlow
data class UserUiState(
    val user: User? = null,
    val isLoading: Boolean = false,
    val error: String? = null
)
sealed class UserEffect {
    data class ShowToast(val message: String) : UserEffect()
}

class UserViewModel : ViewModel() {
    private val _uiState = MutableStateFlow(UserUiState())
    val uiState: StateFlow<UserUiState> = _uiState.asStateFlow()

    private val _effect = MutableSharedFlow<UserEffect>()
    val effect: SharedFlow<UserEffect> = _effect.asSharedFlow()

    fun loadUser(userId: Long) {
        viewModelScope.launch {
            _uiState.update { it.copy(isLoading = true, error = null) }
            userRepository.getUser(userId)
                .onSuccess { user ->
                    _uiState.update { it.copy(user = user, isLoading = false) }
                }
                .onFailure { e ->
                    _uiState.update { it.copy(isLoading = false, error = e.message) }
                    _effect.emit(UserEffect.ShowToast(e.message ?: "오류"))
                }
        }
    }
}

// Fragment — After
viewLifecycleOwner.lifecycleScope.launch {
    viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
        launch {
            viewModel.uiState.collect { state ->
                binding.tvName.text = state.user?.name ?: ""
                binding.progress.isVisible = state.isLoading
            }
        }
        launch {
            viewModel.effect.collect { effect ->
                when (effect) {
                    is UserEffect.ShowToast -> showToast(effect.message)
                }
            }
        }
    }
}
```

---

## 11. 상황별 선택 기준

```
LiveData를 선택할 때
    ├── 레거시 코드베이스 — 코루틴 도입 전 프로젝트
    ├── DataBinding XML과 함께 사용
    └── 팀이 Flow에 익숙하지 않을 때

Flow(StateFlow/SharedFlow)를 선택할 때
    ├── 신규 프로젝트 — 현재 Android 표준
    ├── 다양한 연산자가 필요할 때 (debounce, combine, retry 등)
    ├── 비즈니스 로직을 Domain 레이어에서 테스트할 때
    ├── 일회성 이벤트 처리 (SharedFlow)
    └── Kotlin Multiplatform 대응 예정
```

---

## 12. 전체 비교표

| 항목 | LiveData | StateFlow | SharedFlow |
|------|----------|-----------|------------|
| 패키지 | androidx.lifecycle | kotlinx.coroutines | kotlinx.coroutines |
| 플랫폼 | Android 전용 | 멀티플랫폼 | 멀티플랫폼 |
| 초기값 | 선택 | **필수** | 없음 |
| 마지막 값 유지 | ✅ | ✅ | ❌ (기본) |
| 생명주기 처리 | 자동 | repeatOnLifecycle 필요 | repeatOnLifecycle 필요 |
| 연산자 | 제한적 | Flow 전체 | Flow 전체 |
| 에러 처리 | 별도 래핑 | catch | catch |
| 일회성 이벤트 | ⚠️ 복잡 | ❌ 부적합 | ✅ |
| 테스트 | Rule 필요 | 간결 | Turbine |
| DataBinding | ✅ | ⚠️ (설정 필요) | ❌ |

---

## 13. 정리

| 상황 | 선택 |
|------|------|
| UI 상태 관리 (신규) | `StateFlow` |
| UI 상태 관리 (레거시) | `LiveData` 유지 또는 단계적 전환 |
| 일회성 이벤트 (토스트, 내비게이션) | `SharedFlow` |
| DataBinding XML 연동 | `LiveData` |
| 연속 데이터 스트림, 복잡한 변환 | `Flow` |
| Domain 레이어 (Android 독립) | `Flow` |

- **현재 Android 공식 권장은 `StateFlow` + `SharedFlow`** 입니다.
- `LiveData`는 DataBinding 연동 또는 레거시 유지가 필요한 경우에 사용합니다.
- 신규 프로젝트라면 처음부터 `StateFlow`를 도입하는 것이 장기적으로 유리합니다.

---

## 참고

- [Android 공식 — LiveData vs Flow](https://developer.android.com/topic/architecture/ui-layer/stateholders)
- [StateFlow & SharedFlow 공식 문서](https://developer.android.com/kotlin/flow/stateflow-and-sharedflow)
- [LiveData → Flow 마이그레이션 가이드](https://developer.android.com/kotlin/flow/migrate)
- [repeatOnLifecycle API 문서](https://developer.android.com/topic/libraries/architecture/coroutines#repeatonlifecycle)
