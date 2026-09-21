---
title: (Android/Compose) Compose ViewModel 연동 + StateFlow
tags: [ Android, Compose ]
style: fill
color: dark
description: Jetpack Compose에서 ViewModel과 StateFlow를 연동하는 방법과 UiState 패턴, 이벤트 처리, SavedStateHandle 활용을 정리합니다.
---

---

## 1. Compose + ViewModel 아키텍처

Compose는 UI만 담당하고, 비즈니스 로직과 상태는 ViewModel이 관리합니다.
ViewModel은 화면 회전에도 살아남고, Composable은 언제든 재구성될 수 있으므로 **상태의 진실 공급원(Single Source of Truth)은 항상 ViewModel**에 둡니다.

```
UI (Composable)  ←── StateFlow 구독 ──  ViewModel
      │                                     │
      └──────── 사용자 이벤트 전달 ─────────┘
```

**의존성 추가**:
```kotlin
implementation("androidx.lifecycle:lifecycle-viewmodel-compose:2.8.4")
```

---

## 2. UiState 패턴 — 화면 상태를 하나의 데이터 클래스로

여러 개의 상태 변수 대신, 화면에 필요한 모든 상태를 하나의 불변 데이터 클래스로 묶습니다.

```kotlin
// 화면에 필요한 모든 상태를 하나로 묶은 UiState
data class ProductListUiState(
    val products: List<Product> = emptyList(),
    val isLoading: Boolean = false,
    val errorMessage: String? = null
)

class ProductViewModel(private val repository: ProductRepository) : ViewModel() {
    // MutableStateFlow: ViewModel 내부에서만 값 변경 가능
    private val _uiState = MutableStateFlow(ProductListUiState())
    // StateFlow: 외부(Compose)에는 읽기 전용으로 노출 (캡슐화)
    val uiState: StateFlow<ProductListUiState> = _uiState.asStateFlow()

    init {
        loadProducts()
    }

    fun loadProducts() {
        // update: 이전 상태를 기반으로 새 상태 생성 (원자적 업데이트)
        _uiState.update { it.copy(isLoading = true, errorMessage = null) }

        viewModelScope.launch {
            try {
                val products = repository.fetchProducts()
                _uiState.update { it.copy(products = products, isLoading = false) }
            } catch (e: Exception) {
                _uiState.update { it.copy(isLoading = false, errorMessage = e.message) }
            }
        }
    }
}
```

> **왜 여러 상태 변수 대신 하나의 UiState?**: 상태들이 서로 연관되어 있을 때(로딩 중이면서 동시에 에러가 있으면 안 됨 등) 하나의 데이터 클래스로 묶으면 불일치 상태를 방지하고, 화면에 필요한 상태를 한눈에 파악할 수 있습니다.

---

## 3. Compose에서 StateFlow 구독

`collectAsState` 또는 생명주기를 인식하는 `collectAsStateWithLifecycle`로 구독합니다.

```kotlin
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import androidx.lifecycle.viewmodel.compose.viewModel

@Composable
fun ProductListScreen(
    viewModel: ProductViewModel = viewModel()  // ViewModel 인스턴스 자동 획득
) {
    // collectAsStateWithLifecycle: 화면이 백그라운드일 때 자동으로 구독 중지
    // (collectAsState보다 배터리/리소스 효율적, 권장 방식)
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    Box(modifier = Modifier.fillMaxSize()) {
        when {
            uiState.isLoading -> {
                CircularProgressIndicator(modifier = Modifier.align(Alignment.Center))
            }
            uiState.errorMessage != null -> {
                ErrorMessage(
                    message = uiState.errorMessage,
                    onRetry = { viewModel.loadProducts() }  // 사용자 이벤트 → ViewModel에 위임
                )
            }
            else -> {
                LazyColumn {
                    items(uiState.products, key = { it.id }) { product ->
                        ProductItem(product)
                    }
                }
            }
        }
    }
}
```

> **collectAsState vs collectAsStateWithLifecycle**: `collectAsState`는 Composable이 Composition에 있는 한 계속 구독합니다. `collectAsStateWithLifecycle`은 Activity/Fragment가 STOPPED 상태면 구독을 멈춰 불필요한 리소스 사용을 막습니다 — 실무에서는 후자를 권장합니다.

---

## 4. 일회성 이벤트 처리 — SharedFlow

토스트 메시지, 화면 이동처럼 **한 번만 발생해야 하는 이벤트**는 StateFlow가 아닌 SharedFlow로 처리합니다.
StateFlow는 항상 최신 값을 유지하므로 화면 재구성 시 이벤트가 중복 발생할 수 있습니다.

```kotlin
sealed class UiEvent {
    data class ShowToast(val message: String) : UiEvent()
    data class NavigateToDetail(val productId: Int) : UiEvent()
}

class ProductViewModel : ViewModel() {
    // SharedFlow: 값을 유지하지 않고 이벤트처럼 흘려보냄 (구독 시점 이후 것만 수신)
    private val _uiEvent = MutableSharedFlow<UiEvent>()
    val uiEvent: SharedFlow<UiEvent> = _uiEvent.asSharedFlow()

    fun onProductClick(productId: Int) {
        viewModelScope.launch {
            _uiEvent.emit(UiEvent.NavigateToDetail(productId))
        }
    }

    fun onDeleteError() {
        viewModelScope.launch {
            _uiEvent.emit(UiEvent.ShowToast("삭제에 실패했습니다"))
        }
    }
}

@Composable
fun ProductListScreen(viewModel: ProductViewModel = viewModel()) {
    val context = LocalContext.current

    // LaunchedEffect(Unit): Composable이 처음 컴포지션에 들어올 때 한 번 실행
    // → 이벤트 Flow를 구독하는 코루틴 시작
    LaunchedEffect(Unit) {
        viewModel.uiEvent.collect { event ->
            when (event) {
                is UiEvent.ShowToast ->
                    Toast.makeText(context, event.message, Toast.LENGTH_SHORT).show()
                is UiEvent.NavigateToDetail ->
                    { /* navController.navigate("detail/${event.productId}") */ }
            }
        }
    }

    // 화면 UI...
}
```

---

## 5. SavedStateHandle — 프로세스 종료 후에도 상태 복원

시스템에 의해 프로세스가 종료되었다가 재시작될 때도 상태를 복원하고 싶다면 `SavedStateHandle`을 사용합니다.

```kotlin
class SearchViewModel(
    private val savedStateHandle: SavedStateHandle,  // ViewModel 생성자에 자동 주입
    private val repository: SearchRepository
) : ViewModel() {

    // SavedStateHandle의 값을 StateFlow로 노출 — 프로세스 재생성 후에도 복원됨
    val query: StateFlow<String> = savedStateHandle.getStateFlow("query", "")

    fun onQueryChange(newQuery: String) {
        // set()으로 저장 → 프로세스가 죽어도 Bundle에 보존되어 복원 가능
        savedStateHandle["query"] = newQuery
    }

    // Navigation 인자도 SavedStateHandle로 받을 수 있음
    val productId: Int = savedStateHandle.get<Int>("productId") ?: -1
}
```

---

## 6. 실전 — combine으로 여러 Flow 합성

여러 데이터 소스(검색어 + 필터 + 원본 목록)를 조합해 최종 UiState를 만드는 패턴입니다.

```kotlin
class ProductListViewModel(private val repository: ProductRepository) : ViewModel() {
    private val _searchQuery = MutableStateFlow("")
    private val _selectedCategory = MutableStateFlow<String?>(null)

    fun onSearchChange(query: String) { _searchQuery.value = query }
    fun onCategorySelect(category: String?) { _selectedCategory.value = category }

    // combine: 세 Flow 중 하나라도 바뀌면 자동으로 재계산되는 파생 상태
    val uiState: StateFlow<ProductListUiState> = combine(
        repository.observeProducts(),  // 원본 상품 목록 (DB Flow)
        _searchQuery,
        _selectedCategory
    ) { products, query, category ->
        // 세 값을 조합해 최종 필터링된 목록 계산
        val filtered = products
            .filter { query.isBlank() || it.name.contains(query, ignoreCase = true) }
            .filter { category == null || it.category == category }
        ProductListUiState(products = filtered)
    }.stateIn(
        scope = viewModelScope,
        // WhileSubscribed: 구독자가 있을 때만 활성, 화면 이탈 5초 후 자동 정지 (리소스 절약)
        started = SharingStarted.WhileSubscribed(5000),
        initialValue = ProductListUiState(isLoading = true)
    )
}
```

> **stateIn의 SharingStarted.WhileSubscribed(5000)**: 화면 회전처럼 짧은 순간 구독자가 0이 되는 경우를 대비해 5초의 유예 시간을 둡니다. 이 시간이 지나야 Flow가 실제로 정지되어, 화면 회전 때마다 데이터를 재조회하는 낭비를 막습니다.

---

## 7. 정리

| 개념 | 용도 |
|------|------|
| UiState 데이터 클래스 | 화면에 필요한 모든 상태를 하나로 관리 |
| `StateFlow` | 지속적인 상태 구독 (현재 값 유지) |
| `SharedFlow` | 일회성 이벤트 (토스트, 네비게이션) |
| `collectAsStateWithLifecycle` | 생명주기를 인식하는 안전한 구독 방식 |
| `SavedStateHandle` | 프로세스 종료 후에도 상태 복원 |
| `combine` + `stateIn` | 여러 Flow를 조합한 파생 상태 생성 |

- 상태의 진실 공급원은 항상 **ViewModel**, Composable은 표시와 이벤트 전달만 담당
- StateFlow(상태)와 SharedFlow(이벤트)를 목적에 맞게 구분해서 사용
- `collectAsStateWithLifecycle`을 기본으로 사용해 불필요한 백그라운드 구독 방지
