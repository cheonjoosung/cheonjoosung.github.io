---
title: (Android/Hilt) Hilt + ViewModel — @HiltViewModel 완전 정리
tags: [ Android, Hilt ]
style: fill
color: dark
description: "@HiltViewModel로 ViewModel에 의존성을 주입하는 방법과 SavedStateHandle, Compose hiltViewModel(), AssistedInject 사용법을 정리합니다."
---

---

## 1. @HiltViewModel이란?

ViewModel은 `ViewModelProvider.Factory`로 생성되므로 일반 `@Inject constructor`만으로는 주입이 안 됩니다.
**`@HiltViewModel`을 붙이면 Hilt가 Factory를 자동 생성**해 의존성을 주입해줍니다.

```
수동 방식: ViewModelFactory 클래스를 직접 작성 → 의존성마다 수정 필요
Hilt 방식: @HiltViewModel + @Inject constructor → Factory 자동 생성
```

**의존성**:
```kotlin
implementation("com.google.dagger:hilt-android:2.51.1")
ksp("com.google.dagger:hilt-android-compiler:2.51.1")
// Compose에서 hiltViewModel() 사용 시
implementation("androidx.hilt:hilt-navigation-compose:1.2.0")
```

---

## 2. 기본 사용법

```kotlin
@HiltViewModel  // Hilt가 이 ViewModel의 Factory를 생성
class ProductViewModel @Inject constructor(  // 생성자 주입 필수
    private val repository: ProductRepository,
    private val analytics: AnalyticsTracker
) : ViewModel() {

    private val _uiState = MutableStateFlow(ProductUiState())
    val uiState = _uiState.asStateFlow()

    init { loadProducts() }

    private fun loadProducts() {
        viewModelScope.launch {
            _uiState.update { it.copy(products = repository.fetchProducts()) }
        }
    }
}
```

**Activity/Fragment에서 사용**:
```kotlin
@AndroidEntryPoint  // 필수
class ProductActivity : AppCompatActivity() {

    // by viewModels(): Hilt가 자동으로 올바른 Factory 연결
    private val viewModel: ProductViewModel by viewModels()
}
```

**Compose에서 사용**:
```kotlin
@Composable
fun ProductScreen(
    viewModel: ProductViewModel = hiltViewModel()  // Hilt가 주입된 ViewModel 획득
) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    // ...
}
```

> **Context 주입**: ViewModel에서 Context가 필요하면 `@ApplicationContext private val context: Context`로 주입받습니다. Activity Context는 메모리 누수 위험이 있으므로 ViewModel에 넣지 않습니다.

---

## 3. SavedStateHandle 자동 주입

Navigation 인자나 프로세스 종료 후 복원할 상태에 접근하는 `SavedStateHandle`은 **별도 설정 없이** 주입됩니다.

```kotlin
@HiltViewModel
class ProductDetailViewModel @Inject constructor(
    private val savedStateHandle: SavedStateHandle,  // Hilt가 자동 제공
    private val repository: ProductRepository
) : ViewModel() {

    // Navigation 경로 인자 "productId"를 꺼냄
    private val productId: Int = checkNotNull(savedStateHandle["productId"])

    val product = repository.observeProduct(productId)
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), null)
}
```

---

## 4. 런타임 값이 필요한 경우 — AssistedInject

Hilt는 컴파일 타임에 그래프를 구성하므로 **런타임에 결정되는 값**(사용자가 입력한 ID 등)은 직접 주입할 수 없습니다.
대부분은 `SavedStateHandle`로 해결되지만, 직접 값을 넘겨야 할 때는 `@AssistedInject`를 사용합니다.

```kotlin
@HiltViewModel(assistedFactory = ChatViewModel.Factory::class)
class ChatViewModel @AssistedInject constructor(
    private val repository: ChatRepository,   // Hilt가 주입
    @Assisted private val roomId: String      // 런타임에 호출자가 전달
) : ViewModel() {

    @AssistedFactory
    interface Factory {
        fun create(roomId: String): ChatViewModel
    }
}

// Compose 사용
@Composable
fun ChatScreen(roomId: String) {
    val viewModel = hiltViewModel<ChatViewModel, ChatViewModel.Factory>(
        creationCallback = { factory -> factory.create(roomId) }
    )
}
```

---

## 5. 흔한 실수

```kotlin
// ❌ @HiltViewModel 누락 → "Cannot create an instance of ViewModel" 크래시
class MyViewModel @Inject constructor(...) : ViewModel()

// ❌ Activity에 @AndroidEntryPoint 누락 → viewModels()에서 크래시

// ❌ ViewModel에 Activity/Fragment/View 주입 → 메모리 누수
// ✅ @ApplicationContext 또는 필요한 데이터만 주입
```

---

## 6. 정리

- `@HiltViewModel` + `@Inject constructor`: ViewModel Factory 자동 생성
- Activity/Fragment: `@AndroidEntryPoint` + `by viewModels()`
- Compose: `hiltViewModel()` (hilt-navigation-compose 필요)
- `SavedStateHandle`은 자동 주입 — Navigation 인자 접근에 활용
- 런타임 값은 `SavedStateHandle` 또는 `@AssistedInject`로 전달
- ViewModel에는 `@ApplicationContext`만 사용 (Activity Context 금지)
