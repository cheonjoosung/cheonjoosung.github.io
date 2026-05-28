---
title: (Kotlin/코틀린) viewModelScope vs lifecycleScope vs GlobalScope 완전 비교
tags: [ Kotlin, Android ]
style: fill
color: dark
description: Android 코루틴에서 가장 많이 쓰이는 viewModelScope, lifecycleScope, GlobalScope의 동작 원리와 생명주기, 차이점, 올바른 선택 기준을 실전 코드와 함께 정리합니다.
---

## 개요

- Android에서 코루틴을 시작할 때 반드시 선택해야 하는 **CoroutineScope** 를 비교합니다.
- 잘못된 Scope 선택은 **메모리 누수·크래시·불필요한 작업 지속** 으로 이어집니다.
- 이 글에서는 다음을 설명합니다.
  - CoroutineScope의 역할
  - `viewModelScope` — ViewModel 생명주기 연동
  - `lifecycleScope` — Activity/Fragment 생명주기 연동
  - `GlobalScope` — 앱 전역 Scope, 사용을 피해야 하는 이유
  - `rememberCoroutineScope` — Jetpack Compose
  - 상황별 올바른 선택 기준

---

## 1. CoroutineScope란

코루틴은 반드시 **CoroutineScope** 안에서 시작됩니다. Scope는 다음 두 가지를 담당합니다.

```
CoroutineScope
├── CoroutineContext (실행 환경)
│   ├── Job          — 생명주기 관리, 취소 전파
│   ├── Dispatcher   — 어떤 스레드에서 실행할지
│   └── ExceptionHandler
└── 코루틴을 launch/async 할 수 있는 확장 함수 제공
```

```kotlin
// Scope를 직접 만들면 직접 관리해야 함
val scope = CoroutineScope(Dispatchers.IO + SupervisorJob())

scope.launch { /* 작업 */ }

// 반드시 직접 취소해야 함 — 안 하면 누수
scope.cancel()
```

Android에서는 생명주기와 연동된 **공식 Scope** 를 사용하면 이 관리를 자동으로 해줍니다.

---

## 2. viewModelScope

`viewModelScope`는 **ViewModel이 살아있는 동안만 유지되는 Scope** 입니다.

### 내부 구현

```kotlin
// androidx.lifecycle:lifecycle-viewmodel-ktx 내부
val ViewModel.viewModelScope: CoroutineScope
    get() {
        val scope = this.getTag(JOB_KEY)
        if (scope != null) return scope

        return CloseableCoroutineScope(
            SupervisorJob() + Dispatchers.Main.immediate
        ).also { this.setTagIfAbsent(JOB_KEY, it) }
    }

// ViewModel.onCleared() 호출 시 자동으로 scope.cancel()
```

- 기본 Dispatcher: `Dispatchers.Main.immediate`
- Job: `SupervisorJob()` — 자식 코루틴 하나가 실패해도 나머지 유지
- **ViewModel이 소멸(onCleared)될 때 자동 취소**

### 생명주기

```
Activity 시작
    └── ViewModel 생성 → viewModelScope 생성
         └── 화면 회전 (Activity 재생성)
              └── ViewModel 유지 → viewModelScope 유지  ← 핵심 강점
         └── Activity 완전 종료 or ViewModel 소멸
              └── viewModelScope.cancel() 자동 호출
```

### 실전 코드

```kotlin
@HiltViewModel
class UserViewModel @Inject constructor(
    private val userRepository: UserRepository
) : ViewModel() {

    private val _uiState = MutableStateFlow(UserUiState())
    val uiState: StateFlow<UserUiState> = _uiState.asStateFlow()

    // ✅ 데이터 로드 — ViewModel 생명주기에 맞게 자동 취소
    fun loadUser(userId: Long) {
        viewModelScope.launch {
            _uiState.update { it.copy(isLoading = true) }
            userRepository.getUser(userId)
                .onSuccess { user ->
                    _uiState.update { it.copy(isLoading = false, user = user) }
                }
                .onFailure { e ->
                    _uiState.update { it.copy(isLoading = false, error = e.message) }
                }
        }
    }

    // ✅ 화면 회전 시에도 중단되지 않고 계속 실행
    fun uploadFile(file: File) {
        viewModelScope.launch {
            _uiState.update { it.copy(isUploading = true) }
            fileRepository.upload(file)   // 화면 회전해도 업로드 계속
            _uiState.update { it.copy(isUploading = false) }
        }
    }
}
```

### viewModelScope를 써야 할 때

```
✅ 네트워크 요청 / DB 조회
✅ 화면 회전에 영향받지 않아야 하는 작업
✅ UI 상태(StateFlow, LiveData) 업데이트
✅ 데이터 저장/삭제 등 비즈니스 로직
```

---

## 3. lifecycleScope

`lifecycleScope`는 **Activity 또는 Fragment의 생명주기에 연동된 Scope** 입니다.

### 내부 구현

```kotlin
// androidx.lifecycle:lifecycle-runtime-ktx 내부
val LifecycleOwner.lifecycleScope: LifecycleCoroutineScope
    get() = lifecycle.coroutineScope

// Lifecycle.State.DESTROYED 시 자동 취소
```

- 기본 Dispatcher: `Dispatchers.Main.immediate`
- **Activity/Fragment가 DESTROYED 될 때 자동 취소**
- Fragment에서는 `viewLifecycleOwner.lifecycleScope` 사용 권장

### 생명주기

```
Activity 시작 → lifecycleScope 생성
    ├── onStart()
    ├── onResume()
    ├── onPause()
    ├── onStop()
    └── onDestroy() → lifecycleScope.cancel() 자동 호출

화면 회전:
    Activity onDestroy() → lifecycleScope 취소
    Activity onCreate()  → lifecycleScope 새로 생성
    ↑ viewModelScope와의 핵심 차이
```

### 실전 코드

```kotlin
@AndroidEntryPoint
class UserFragment : Fragment(R.layout.fragment_user) {

    private val viewModel: UserViewModel by viewModels()
    private lateinit var binding: FragmentUserBinding

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        binding = FragmentUserBinding.bind(view)

        // ✅ UI 관찰 — viewLifecycleOwner.lifecycleScope 사용
        viewLifecycleOwner.lifecycleScope.launch {
            viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.uiState.collect { state ->
                    renderState(state)
                }
            }
        }

        // ✅ 클릭 이벤트에서 코루틴 시작
        binding.btnSave.setOnClickListener {
            viewLifecycleOwner.lifecycleScope.launch {
                val text = binding.etInput.text.toString()
                // Fragment가 살아있는 동안만 실행
                viewModel.saveText(text)
            }
        }
    }
}
```

### Fragment에서 주의할 점

```kotlin
// ❌ Fragment의 lifecycleScope — Fragment 재사용 시 문제
class MyFragment : Fragment() {
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        lifecycleScope.launch {          // Fragment 생명주기 기준
            viewModel.uiState.collect { // View가 없는데 UI 접근 가능
                binding.tvName.text = it.user?.name   // 크래시 위험
            }
        }
    }
}

// ✅ viewLifecycleOwner.lifecycleScope — View 생명주기 기준
class MyFragment : Fragment() {
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        viewLifecycleOwner.lifecycleScope.launch {   // View 생명주기 기준
            viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.uiState.collect {
                    binding.tvName.text = it.user?.name   // 안전
                }
            }
        }
    }
}
```

### lifecycleScope를 써야 할 때

```
✅ UI 상태 관찰 (StateFlow, SharedFlow collect)
✅ View와 직접 연결된 애니메이션, 트랜지션
✅ 클릭 이벤트에서 시작하는 단발성 작업
✅ Fragment의 View 생명주기에 맞는 작업
```

---

## 4. repeatOnLifecycle — lifecycleScope와 함께 쓰는 핵심 API

`repeatOnLifecycle`은 **특정 Lifecycle.State에 진입할 때 블록을 시작하고, 벗어나면 취소** 합니다.

```kotlin
viewLifecycleOwner.lifecycleScope.launch {
    viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
        // STARTED 진입 시 시작, STOPPED 시 취소, 다시 STARTED 시 재시작
        viewModel.uiState.collect { state ->
            renderState(state)
        }
    }
}
```

### Lifecycle.State 선택 기준

| State | 진입 시점 | 취소 시점 | 사용 예 |
|-------|----------|----------|--------|
| `CREATED` | onCreate | onDestroy | 화면 비표시 상태도 수신 |
| `STARTED` | onStart | onStop | **일반 UI 관찰 — 가장 많이 사용** |
| `RESUMED` | onResume | onPause | 포그라운드 전용 작업 |

```kotlin
// ✅ 권장 패턴 — STARTED 상태에서만 Flow 수집
viewLifecycleOwner.lifecycleScope.launch {
    viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
        launch { viewModel.uiState.collect { renderState(it) } }
        launch { viewModel.uiEvent.collect { handleEvent(it) } }
    }
}
```

---

## 5. GlobalScope

`GlobalScope`는 **앱 프로세스 전체 생명주기와 연동된 Scope** 입니다.

### 내부 구현

```kotlin
// 코틀린 표준 라이브러리 내부
object GlobalScope : CoroutineScope {
    override val coroutineContext: CoroutineContext
        get() = EmptyCoroutineContext
}
```

- 취소 불가능 (Job이 없음)
- 앱이 종료될 때까지 살아있음
- 구조화된 동시성에서 완전히 벗어남

### GlobalScope를 피해야 하는 이유

```kotlin
class UserFragment : Fragment() {

    // ❌ GlobalScope — 심각한 문제 발생
    fun loadUser(userId: Long) {
        GlobalScope.launch {
            val user = repository.getUser(userId)

            // Fragment가 이미 종료됐을 수 있음
            binding.tvName.text = user.name   // 크래시 위험 (binding null)

            // Fragment가 종료돼도 네트워크 요청 계속
            // → 메모리 누수, 불필요한 서버 부하
        }
    }
}
```

```
GlobalScope 문제점:
1. Fragment/Activity 종료 후에도 계속 실행 → 메모리 누수
2. View 참조가 null인데 접근 → NullPointerException
3. 취소할 수 없음 → 불필요한 네트워크/DB 작업 지속
4. 테스트 어려움 → 테스트 종료 후에도 코루틴 실행
```

### GlobalScope가 허용되는 경우

```kotlin
// ⚠️ 극히 드물게 허용 — 앱 전체에서 살아있어야 하는 작업
object AppLogger {
    fun startLogging() {
        // 앱 생명주기와 동일해야 하는 로깅
        GlobalScope.launch(Dispatchers.IO) {
            logChannel.consumeEach { log ->
                writeToFile(log)
            }
        }
    }
}

// ✅ 대부분은 Application 수준 Scope로 대체 가능
class MyApplication : Application() {
    val applicationScope = CoroutineScope(SupervisorJob() + Dispatchers.Default)

    override fun onTerminate() {
        super.onTerminate()
        applicationScope.cancel()
    }
}
```

---

## 6. Application Scope — GlobalScope 대안

`GlobalScope` 대신 **Application 수준의 커스텀 Scope** 를 만들면 생명주기 관리가 가능합니다.

```kotlin
// Application Scope 정의
@HiltComponent
class MyApplication : Application() {

    // Hilt로 주입하거나 직접 생성
    val applicationScope = CoroutineScope(
        SupervisorJob() + Dispatchers.Default
    )

    override fun onTerminate() {
        super.onTerminate()
        applicationScope.cancel()
    }
}

// Hilt Module로 제공
@Module
@InstallIn(SingletonComponent::class)
object CoroutineScopeModule {

    @Singleton
    @Provides
    fun provideApplicationScope(): CoroutineScope =
        CoroutineScope(SupervisorJob() + Dispatchers.Default)
}

// Repository 또는 Service에서 주입받아 사용
@Singleton
class SyncRepository @Inject constructor(
    @ApplicationScope private val applicationScope: CoroutineScope,
    private val syncApi: SyncApi
) {
    fun startPeriodicSync() {
        applicationScope.launch {
            while (isActive) {
                syncApi.sync()
                delay(15 * 60 * 1000L)   // 15분마다
            }
        }
    }
}
```

---

## 7. rememberCoroutineScope — Jetpack Compose

Compose에서는 `rememberCoroutineScope()`로 Composable 생명주기에 연동된 Scope를 사용합니다.

```kotlin
@Composable
fun UserScreen(viewModel: UserViewModel = hiltViewModel()) {

    // Composable이 컴포지션에서 벗어나면 자동 취소
    val scope = rememberCoroutineScope()

    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    Column {
        Text(text = uiState.user?.name ?: "")

        Button(
            onClick = {
                // ✅ 클릭 이벤트에서 코루틴 시작
                scope.launch {
                    viewModel.loadUser(userId = 1L)
                }
            }
        ) {
            Text("불러오기")
        }
    }
}
```

---

## 8. 전체 비교표

| 항목 | `viewModelScope` | `lifecycleScope` | `GlobalScope` | `applicationScope` |
|------|-----------------|-----------------|--------------|-------------------|
| 위치 | ViewModel | Activity/Fragment | 전역 | Application |
| 취소 시점 | ViewModel 소멸 | DESTROYED | 앱 종료 | 앱 종료 (수동) |
| 화면 회전 | 유지 ✅ | 취소·재생성 | 유지 | 유지 |
| 메모리 누수 위험 | 없음 | 없음 | **있음** | 없음 |
| 생명주기 자동 관리 | ✅ | ✅ | ❌ | 수동 cancel 필요 |
| 기본 Dispatcher | Main.immediate | Main.immediate | EmptyContext | 지정 필요 |
| 사용 권장 | ✅ 강력 권장 | ✅ 강력 권장 | ❌ 지양 | ⚠️ 필요 시 |

---

## 9. 상황별 선택 기준

```
비즈니스 로직 (API, DB)
    → viewModelScope

UI 관찰 (StateFlow, SharedFlow collect)
    → viewLifecycleOwner.lifecycleScope + repeatOnLifecycle

클릭 이벤트 단발성 작업
    → lifecycleScope.launch (또는 viewModelScope)

Compose UI 이벤트
    → rememberCoroutineScope

앱 전역에서 살아야 하는 작업
    → applicationScope (GlobalScope 대신)
```

---

## 10. 주의사항

### ❌ Fragment에서 lifecycleScope와 viewLifecycleOwner.lifecycleScope 혼용

```kotlin
// ❌ Fragment의 lifecycleScope — View 소멸 후에도 실행
lifecycleScope.launch {
    viewModel.flow.collect { binding.tvName.text = it }  // 크래시 위험
}

// ✅ viewLifecycleOwner.lifecycleScope — View 생명주기 기준
viewLifecycleOwner.lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.flow.collect { binding.tvName.text = it }  // 안전
    }
}
```

---

### ❌ ViewModel에서 lifecycleScope 사용

```kotlin
// ❌ ViewModel은 LifecycleOwner가 아님
class UserViewModel : ViewModel() {
    fun load() {
        lifecycleScope.launch { ... }   // 컴파일 에러
    }
}

// ✅ viewModelScope 사용
class UserViewModel : ViewModel() {
    fun load() {
        viewModelScope.launch { ... }
    }
}
```

---

### ❌ GlobalScope로 UI 접근

```kotlin
// ❌ GlobalScope + UI 접근 — 크래시 보장
GlobalScope.launch(Dispatchers.Main) {
    delay(5000)
    binding.tvName.text = "완료"   // Fragment 종료 후 binding null
}

// ✅ lifecycleScope — 자동 취소로 안전
lifecycleScope.launch {
    delay(5000)
    binding.tvName.text = "완료"   // Fragment 종료 시 자동 취소
}
```

---

## 11. 정리

| Scope | 핵심 한 줄 요약 |
|-------|--------------|
| `viewModelScope` | ViewModel과 운명 공동체 — 화면 회전에 강함 |
| `lifecycleScope` | Activity/Fragment View와 운명 공동체 |
| `repeatOnLifecycle` | lifecycleScope와 함께 써서 포그라운드만 수집 |
| `GlobalScope` | 취소 불가, 메모리 누수 — 사용 지양 |
| `applicationScope` | GlobalScope 대신 쓰는 앱 전역 Scope |
| `rememberCoroutineScope` | Compose Composable 생명주기 연동 |

- `viewModelScope`는 **비즈니스 로직**, `lifecycleScope`는 **UI 관찰** 이 기본 원칙입니다.
- `GlobalScope`는 취소할 수 없으므로 Android 앱에서는 사용하지 마세요.
- Fragment에서는 항상 **`viewLifecycleOwner.lifecycleScope`** 를 사용하세요.

---

## 참고

- [Android Coroutines 공식 가이드](https://developer.android.com/kotlin/coroutines)
- [repeatOnLifecycle API 문서](https://developer.android.com/topic/libraries/architecture/coroutines#repeatonlifecycle)
- [ViewModel 생명주기 공식 문서](https://developer.android.com/topic/libraries/architecture/viewmodel)
- [Coroutines Best Practices — Android](https://developer.android.com/kotlin/coroutines/coroutines-best-practices)
