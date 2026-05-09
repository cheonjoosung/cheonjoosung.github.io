---
title: (Android/안드로이드) MVVM 패턴 완전 정리
tags: [ Android, Kotlin ]
style: fill
color: dark
description: Android에서 MVVM(Model-View-ViewModel) 패턴의 개념, 구성 요소, LiveData·StateFlow 기반 실전 구현, DataBinding 연동, MVP·MVI와의 비교까지 깊이 있게 정리합니다.
---

## 개요

- Android 아키텍처 패턴 중 **MVVM(Model-View-ViewModel)** 을 다룹니다.
- MVVM은 **UI와 비즈니스 로직을 분리** 해 테스트 용이성과 유지보수성을 높이는 패턴입니다.
- 이 글에서는 다음을 설명합니다.
  - MVVM 구성 요소와 데이터 흐름
  - MVP / MVI와의 차이
  - ViewModel + LiveData / StateFlow 기반 실전 구현
  - DataBinding 연동
  - Repository 패턴과의 조합

---

## 1. MVVM이란

| 구성 요소 | 역할 |
|-----------|------|
| **Model** | 데이터와 비즈니스 로직 — Repository, UseCase, 데이터 클래스 |
| **View** | UI 렌더링 — Activity, Fragment, XML Layout |
| **ViewModel** | View와 Model 사이의 중재자 — UI 상태 보유, 비즈니스 로직 호출 |

```
View ──────────────────────────────▶ ViewModel
 │   (사용자 이벤트 전달)              │
 │                                    │ (데이터 요청)
 │                                    ▼
 │                                  Model
 │                                    │
 │   (LiveData/StateFlow 관찰)         │ (데이터 반환)
 ◀───────────────────────────────────◀
```

- View는 ViewModel을 **관찰(Observe)** 하며 데이터가 변경되면 자동으로 UI를 갱신합니다.
- ViewModel은 View에 대한 **직접 참조를 갖지 않습니다** — 화면 회전 등 생명주기에 안전합니다.
- Model은 데이터 출처(서버, DB)를 추상화합니다.

---

## 2. MVP / MVVM / MVI 비교

| 항목 | MVP | MVVM | MVI |
|------|-----|------|-----|
| View ↔ 중재자 | 인터페이스(Contract) | 관찰(Observer) | Intent 전송 |
| 상태 표현 | Presenter 메서드 호출 | LiveData / StateFlow | 단일 State 객체 |
| 데이터 흐름 | 양방향 | 양방향 가능 | 단방향 강제 |
| View 참조 | Presenter가 직접 참조 | 없음 | 없음 |
| 테스트 용이성 | 보통 | 좋음 | 매우 좋음 |
| 복잡도 | 낮음 | 보통 | 보통~높음 |
| 적합한 규모 | 소~중 | 소~대 | 중~대 |

---

## 3. ViewModel 기본 구현

### Jetpack ViewModel 의존성

```kotlin
// build.gradle.kts (app)
dependencies {
    implementation("androidx.lifecycle:lifecycle-viewmodel-ktx:2.8.3")
    implementation("androidx.lifecycle:lifecycle-livedata-ktx:2.8.3")
    implementation("androidx.lifecycle:lifecycle-runtime-ktx:2.8.3")
}
```

---

### LiveData 방식 ViewModel

```kotlin
class UserViewModel(
    private val userRepository: UserRepository
) : ViewModel() {

    // 외부에는 읽기 전용 LiveData로 노출
    private val _user = MutableLiveData<User?>()
    val user: LiveData<User?> = _user

    private val _isLoading = MutableLiveData(false)
    val isLoading: LiveData<Boolean> = _isLoading

    private val _errorMessage = MutableLiveData<String?>()
    val errorMessage: LiveData<String?> = _errorMessage

    fun loadUser(userId: Long) {
        viewModelScope.launch {
            _isLoading.value = true
            _errorMessage.value = null

            userRepository.getUser(userId)
                .onSuccess { _user.value = it }
                .onFailure { _errorMessage.value = it.message }

            _isLoading.value = false
        }
    }

    fun updateUserName(userId: Long, newName: String) {
        viewModelScope.launch {
            userRepository.updateName(userId, newName)
                .onSuccess { _user.value = _user.value?.copy(name = newName) }
                .onFailure { _errorMessage.value = "이름 변경에 실패했습니다" }
        }
    }
}
```

---

### StateFlow 방식 ViewModel (권장)

```kotlin
// UI 상태를 하나의 data class로 묶기
data class UserUiState(
    val user: User? = null,
    val isLoading: Boolean = false,
    val errorMessage: String? = null
)

class UserViewModel(
    private val userRepository: UserRepository
) : ViewModel() {

    private val _uiState = MutableStateFlow(UserUiState())
    val uiState: StateFlow<UserUiState> = _uiState.asStateFlow()

    fun loadUser(userId: Long) {
        viewModelScope.launch {
            _uiState.update { it.copy(isLoading = true, errorMessage = null) }

            userRepository.getUser(userId)
                .onSuccess { user ->
                    _uiState.update { it.copy(isLoading = false, user = user) }
                }
                .onFailure { error ->
                    _uiState.update { it.copy(isLoading = false, errorMessage = error.message) }
                }
        }
    }

    fun updateUserName(userId: Long, newName: String) {
        viewModelScope.launch {
            userRepository.updateName(userId, newName)
                .onSuccess {
                    _uiState.update { state ->
                        state.copy(user = state.user?.copy(name = newName))
                    }
                }
                .onFailure {
                    _uiState.update { it.copy(errorMessage = "이름 변경에 실패했습니다") }
                }
        }
    }
}
```

---

## 4. View(Fragment) 구현

### LiveData 관찰

```kotlin
@AndroidEntryPoint
class UserFragment : Fragment(R.layout.fragment_user) {

    private val viewModel: UserViewModel by viewModels()
    private lateinit var binding: FragmentUserBinding

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        binding = FragmentUserBinding.bind(view)

        observeViewModel()
        setupListeners()

        viewModel.loadUser(userId = 1L)
    }

    private fun observeViewModel() {
        // LiveData 관찰 — 생명주기 자동 처리
        viewModel.user.observe(viewLifecycleOwner) { user ->
            user?.let {
                binding.tvName.text = it.name
                binding.tvEmail.text = it.email
            }
        }

        viewModel.isLoading.observe(viewLifecycleOwner) { isLoading ->
            binding.progressBar.isVisible = isLoading
        }

        viewModel.errorMessage.observe(viewLifecycleOwner) { message ->
            message?.let {
                Snackbar.make(binding.root, it, Snackbar.LENGTH_SHORT).show()
            }
        }
    }

    private fun setupListeners() {
        binding.btnUpdate.setOnClickListener {
            val newName = binding.etName.text.toString()
            viewModel.updateUserName(userId = 1L, newName = newName)
        }
    }
}
```

---

### StateFlow 관찰 (권장)

```kotlin
@AndroidEntryPoint
class UserFragment : Fragment(R.layout.fragment_user) {

    private val viewModel: UserViewModel by viewModels()
    private lateinit var binding: FragmentUserBinding

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        binding = FragmentUserBinding.bind(view)

        observeUiState()
        setupListeners()

        viewModel.loadUser(userId = 1L)
    }

    private fun observeUiState() {
        viewLifecycleOwner.lifecycleScope.launch {
            viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.uiState.collect { state ->
                    renderState(state)
                }
            }
        }
    }

    private fun renderState(state: UserUiState) {
        with(binding) {
            progressBar.isVisible = state.isLoading
            tvError.isVisible = state.errorMessage != null
            tvError.text = state.errorMessage

            state.user?.let { user ->
                tvName.text = user.name
                tvEmail.text = user.email
                groupContent.isVisible = true
            } ?: run {
                groupContent.isVisible = false
            }
        }
    }

    private fun setupListeners() {
        binding.btnUpdate.setOnClickListener {
            val newName = binding.etName.text.toString()
            viewModel.updateUserName(userId = 1L, newName = newName)
        }
    }
}
```

---

## 5. Hilt를 이용한 ViewModel 의존성 주입

```kotlin
// ViewModel
@HiltViewModel
class UserViewModel @Inject constructor(
    private val userRepository: UserRepository  // Hilt가 자동 주입
) : ViewModel() { ... }

// Module
@Module
@InstallIn(SingletonComponent::class)
object RepositoryModule {

    @Provides
    @Singleton
    fun provideUserRepository(
        userApi: UserApi,
        userDao: UserDao
    ): UserRepository = UserRepositoryImpl(userApi, userDao)
}

// Fragment — by viewModels()로 Hilt 주입된 ViewModel 획득
@AndroidEntryPoint
class UserFragment : Fragment() {
    private val viewModel: UserViewModel by viewModels()
}
```

---

## 6. DataBinding 연동

DataBinding을 사용하면 XML에서 직접 ViewModel 데이터를 바인딩할 수 있습니다.

```xml
<!-- fragment_user.xml -->
<layout xmlns:android="http://schemas.android.com/apk/res/android">
    <data>
        <variable
            name="viewModel"
            type="com.example.UserViewModel" />
    </data>

    <LinearLayout ...>
        <ProgressBar
            android:visibility="@{viewModel.isLoading ? View.VISIBLE : View.GONE}" />

        <TextView
            android:text="@{viewModel.user.name}"
            android:hint="이름" />

        <TextView
            android:text="@{viewModel.user.email}"
            android:hint="이메일" />

        <Button
            android:onClick="@{() -> viewModel.updateUserName(1L, etName.getText().toString())}"
            android:text="수정" />
    </LinearLayout>
</layout>
```

```kotlin
// Fragment — DataBinding + ViewModel 연결
class UserFragment : Fragment() {
    private val viewModel: UserViewModel by viewModels()
    private lateinit var binding: FragmentUserBinding

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        binding = FragmentUserBinding.bind(view).apply {
            this.viewModel = this@UserFragment.viewModel
            lifecycleOwner = viewLifecycleOwner  // LiveData 자동 관찰
        }
    }
}
```

- `lifecycleOwner` 를 설정하면 LiveData 변경 시 XML 바인딩이 자동으로 갱신됩니다.
- 간단한 표현식은 XML에서 처리하되, 복잡한 로직은 ViewModel에 두는 것이 원칙입니다.

---

## 7. Repository 패턴과 조합

MVVM에서 ViewModel은 Repository를 통해 데이터를 요청합니다.

```kotlin
// Repository 인터페이스 — Domain 레이어
interface UserRepository {
    suspend fun getUser(userId: Long): Result<User>
    suspend fun updateName(userId: Long, name: String): Result<Unit>
}

// Repository 구현 — Data 레이어
class UserRepositoryImpl(
    private val userApi: UserApi,      // 원격 데이터 소스
    private val userDao: UserDao       // 로컬 데이터 소스
) : UserRepository {

    override suspend fun getUser(userId: Long): Result<User> {
        return runCatching {
            // 로컬 캐시 우선, 없으면 서버 호출
            userDao.getUser(userId)
                ?: userApi.getUser(userId).also { userDao.insertUser(it) }
        }
    }

    override suspend fun updateName(userId: Long, name: String): Result<Unit> {
        return runCatching {
            userApi.updateName(userId, name)
            userDao.updateName(userId, name)
        }
    }
}

// ViewModel — Repository만 알고, API/DB는 모름
@HiltViewModel
class UserViewModel @Inject constructor(
    private val userRepository: UserRepository  // 구현체가 아닌 인터페이스에 의존
) : ViewModel() { ... }
```

---

## 8. SharedViewModel — Fragment 간 데이터 공유

같은 Activity의 여러 Fragment가 하나의 ViewModel을 공유할 수 있습니다.

```kotlin
// ViewModel
@HiltViewModel
class CartViewModel @Inject constructor(
    private val cartRepository: CartRepository
) : ViewModel() {

    private val _cartItems = MutableStateFlow<List<CartItem>>(emptyList())
    val cartItems: StateFlow<List<CartItem>> = _cartItems.asStateFlow()

    val totalPrice: StateFlow<Int> = cartItems
        .map { items -> items.sumOf { it.price * it.quantity } }
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), 0)

    fun addItem(item: CartItem) {
        _cartItems.update { current ->
            val existing = current.find { it.id == item.id }
            if (existing != null) {
                current.map { if (it.id == item.id) it.copy(quantity = it.quantity + 1) else it }
            } else {
                current + item
            }
        }
    }

    fun removeItem(itemId: Long) {
        _cartItems.update { it.filter { item -> item.id != itemId } }
    }
}

// Fragment A — activityViewModels()로 Activity 범위 ViewModel 획득
class ProductFragment : Fragment() {
    private val cartViewModel: CartViewModel by activityViewModels()

    private fun onAddToCart(item: CartItem) {
        cartViewModel.addItem(item)
    }
}

// Fragment B — 동일한 ViewModel 인스턴스 공유
class CartFragment : Fragment() {
    private val cartViewModel: CartViewModel by activityViewModels()

    private fun observeCart() {
        viewLifecycleOwner.lifecycleScope.launch {
            viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
                cartViewModel.cartItems.collect { items ->
                    adapter.submitList(items)
                }
            }
        }
    }
}
```

---

## 9. LiveData vs StateFlow 비교

| 항목 | LiveData | StateFlow |
|------|----------|-----------|
| 라이브러리 | AndroidX (Android 전용) | Kotlin Coroutines (플랫폼 무관) |
| 초기값 | 없어도 됨 | **필수** |
| 생명주기 | 자동 처리 (LifecycleOwner) | `repeatOnLifecycle` 필요 |
| 백그라운드 업데이트 | `postValue()` 별도 사용 | `update {}` 동일하게 사용 |
| 변환 연산자 | `map`, `switchMap` (제한적) | Flow 연산자 전부 사용 가능 |
| ViewModel 밖 테스트 | 어려움 | 쉬움 |
| 권장 | 단순 UI 상태 | 복잡한 상태·Flow 변환 필요 시 |

---

## 10. MVVM 적용 시 주의사항

### ❌ ViewModel에서 Context를 직접 참조하지 않는다

```kotlin
// ❌ 메모리 누수 위험 — ViewModel이 Activity보다 오래 살아남음
class BadViewModel(private val context: Context) : ViewModel() { }

// ✅ AndroidViewModel 또는 Hilt로 ApplicationContext 주입
class GoodViewModel(application: Application) : AndroidViewModel(application) {
    private val appContext = getApplication<Application>().applicationContext
}
```

---

### ❌ View에서 ViewModel 상태를 직접 변경하지 않는다

```kotlin
// ❌ View가 ViewModel 내부 상태를 직접 조작
(viewModel.uiState as MutableStateFlow).value = UserUiState()

// ✅ ViewModel의 공개 함수를 호출
viewModel.loadUser(userId)
```

---

### ❌ ViewModel에서 View를 직접 참조하지 않는다

```kotlin
// ❌ View 참조 보유 — 생명주기 문제
class BadViewModel : ViewModel() {
    var fragment: UserFragment? = null
    fun onSuccess() { fragment?.showToast("완료") }  // 위험
}

// ✅ 일회성 이벤트는 SharedFlow나 Channel로 전달
private val _event = MutableSharedFlow<UiEvent>()
val event: SharedFlow<UiEvent> = _event.asSharedFlow()
```

---

## 11. 정리

| 항목 | 내용 |
|------|------|
| 목적 | UI와 비즈니스 로직 분리, 생명주기 안전한 상태 관리 |
| ViewModel | UI 상태 보유, 비즈니스 로직 위임, View 참조 없음 |
| 상태 발행 | LiveData (단순) 또는 StateFlow (복잡·변환 필요) |
| 데이터 접근 | Repository 인터페이스를 통해 추상화 |
| DI | Hilt로 Repository 주입, @HiltViewModel 활용 |
| Fragment 공유 | activityViewModels()로 SharedViewModel 구현 |
| DataBinding | lifecycleOwner 설정으로 LiveData 자동 반영 |
| 주의 | Context 직접 참조 금지, View 직접 참조 금지 |

- MVVM은 Android Jetpack과 가장 자연스럽게 맞는 패턴입니다.
- ViewModel + StateFlow + Repository 조합이 현재 Android 개발의 표준에 가깝습니다.
- 더 복잡한 상태 관리가 필요하다면 MVI 패턴으로 발전시키는 것을 고려하세요.

---

## 참고

- [Android Architecture Guides](https://developer.android.com/topic/architecture)
- [ViewModel 공식 문서](https://developer.android.com/topic/libraries/architecture/viewmodel)
- [StateFlow & SharedFlow 공식 문서](https://developer.android.com/kotlin/flow/stateflow-and-sharedflow)
- [DataBinding 공식 문서](https://developer.android.com/topic/libraries/data-binding)
