---
title: (Android/안드로이드) MVI 패턴 완전 정리
tags: [ Android, Kotlin ]
style: fill
color: dark
description: Android에서 MVI(Model-View-Intent) 패턴의 개념, 구성 요소, 단방향 데이터 흐름, ViewModel·StateFlow 기반 실전 구현까지 깊이 있게 정리합니다.
---

## 개요

- Android 아키텍처 패턴 중 **MVI(Model-View-Intent)** 를 다룹니다.
- MVI는 **단방향 데이터 흐름(Unidirectional Data Flow)** 을 기반으로 상태를 예측 가능하게 관리하는 패턴입니다.
- 이 글에서는 다음을 설명합니다.
  - MVI 구성 요소와 데이터 흐름
  - MVP / MVVM과의 차이
  - ViewModel + StateFlow + sealed class 기반 실전 구현
  - 사이드 이펙트(Side Effect) 처리

---

## 1. MVI란

| 구성 요소 | 역할 |
|-----------|------|
| **Model** | 화면의 상태(State) 전체를 표현하는 불변 데이터 클래스 |
| **View** | 상태를 렌더링하고, 사용자 행동을 Intent로 전달 |
| **Intent** | 사용자의 행동 또는 이벤트 — 상태 변경 의도를 표현 |

```
사용자 행동
    ↓
  Intent
    ↓
ViewModel (상태 계산)
    ↓
  State
    ↓
  View (렌더링)
    ↓
사용자 행동 (반복)
```

- 데이터는 **한 방향으로만** 흐릅니다.
- View는 상태를 **직접 변경하지 않고**, Intent를 보낼 뿐입니다.
- 같은 State를 넣으면 항상 같은 UI가 나옵니다 — **예측 가능성** 확보.

---

## 2. MVP / MVVM / MVI 비교

| 항목 | MVP | MVVM | MVI |
|------|-----|------|-----|
| 상태 표현 | 분산 (여러 변수) | 분산 (LiveData 여러 개) | 단일 State 객체 |
| 데이터 흐름 | 양방향 | 양방향 가능 | 단방향 강제 |
| 상태 추적 | 어려움 | 보통 | 쉬움 (State 로그) |
| 테스트 용이성 | 보통 | 좋음 | 매우 좋음 |
| 복잡도 | 낮음 | 보통 | 보통~높음 |
| 적합한 규모 | 소~중 | 중 | 중~대 |

---

## 3. 핵심 구성 요소 설계

### State — 화면 상태 전체를 하나의 불변 객체로

```kotlin
// 상품 목록 화면의 상태
data class ProductState(
    val isLoading: Boolean = false,
    val products: List<Product> = emptyList(),
    val error: String? = null,
    val query: String = ""
)
```

- `data class`로 만들어 `copy()`로 부분 변경합니다.
- 상태의 모든 필드가 한 곳에 모여 있어 현재 화면 상태를 한눈에 파악할 수 있습니다.

---

### Intent — 사용자 행동을 sealed class로 표현

```kotlin
sealed class ProductIntent {
    object LoadProducts : ProductIntent()
    data class SearchProducts(val query: String) : ProductIntent()
    data class AddToCart(val product: Product) : ProductIntent()
    data class ToggleFavorite(val productId: Long) : ProductIntent()
    object RefreshProducts : ProductIntent()
}
```

- `sealed class`로 가능한 모든 행동을 열거합니다.
- `when` 분기에서 컴파일 타임에 누락을 체크할 수 있습니다.

---

### Effect — 일회성 사이드 이펙트 (화면 전환, 토스트 등)

```kotlin
sealed class ProductEffect {
    data class ShowToast(val message: String) : ProductEffect()
    data class NavigateToDetail(val productId: Long) : ProductEffect()
    object NavigateToCart : ProductEffect()
}
```

- State에 넣기 어려운 **일회성** 이벤트를 별도로 관리합니다.
- 토스트, 스낵바, 화면 전환 등이 해당됩니다.

---

## 4. ViewModel 구현

```kotlin
@HiltViewModel
class ProductViewModel @Inject constructor(
    private val getProductsUseCase: GetProductsUseCase,
    private val addToCartUseCase: AddToCartUseCase
) : ViewModel() {

    // 상태 — 외부에는 읽기 전용으로 노출
    private val _state = MutableStateFlow(ProductState())
    val state: StateFlow<ProductState> = _state.asStateFlow()

    // 사이드 이펙트 — 소비 후 사라지는 이벤트
    private val _effect = Channel<ProductEffect>(Channel.BUFFERED)
    val effect = _effect.receiveAsFlow()

    // Intent 처리 진입점
    fun handleIntent(intent: ProductIntent) {
        when (intent) {
            is ProductIntent.LoadProducts       -> loadProducts()
            is ProductIntent.SearchProducts     -> searchProducts(intent.query)
            is ProductIntent.AddToCart          -> addToCart(intent.product)
            is ProductIntent.ToggleFavorite     -> toggleFavorite(intent.productId)
            is ProductIntent.RefreshProducts    -> loadProducts(isRefresh = true)
        }
    }

    private fun loadProducts(isRefresh: Boolean = false) {
        viewModelScope.launch {
            _state.update { it.copy(isLoading = true, error = null) }

            getProductsUseCase()
                .onSuccess { products ->
                    _state.update {
                        it.copy(isLoading = false, products = products)
                    }
                }
                .onFailure { error ->
                    _state.update {
                        it.copy(isLoading = false, error = error.message)
                    }
                    _effect.send(ProductEffect.ShowToast("상품을 불러오지 못했습니다"))
                }
        }
    }

    private fun searchProducts(query: String) {
        viewModelScope.launch {
            _state.update { it.copy(query = query, isLoading = true) }

            getProductsUseCase(query = query)
                .onSuccess { products ->
                    _state.update { it.copy(isLoading = false, products = products) }
                }
                .onFailure { error ->
                    _state.update { it.copy(isLoading = false, error = error.message) }
                }
        }
    }

    private fun addToCart(product: Product) {
        viewModelScope.launch {
            addToCartUseCase(product)
                .onSuccess {
                    _effect.send(ProductEffect.ShowToast("${product.name}을 장바구니에 담았습니다"))
                }
                .onFailure {
                    _effect.send(ProductEffect.ShowToast("장바구니 추가에 실패했습니다"))
                }
        }
    }

    private fun toggleFavorite(productId: Long) {
        _state.update { currentState ->
            val updatedProducts = currentState.products.map { product ->
                if (product.id == productId) product.copy(isFavorite = !product.isFavorite)
                else product
            }
            currentState.copy(products = updatedProducts)
        }
    }
}
```

---

## 5. View(Fragment) 구현

```kotlin
@AndroidEntryPoint
class ProductFragment : Fragment(R.layout.fragment_product) {

    private val viewModel: ProductViewModel by viewModels()
    private lateinit var binding: FragmentProductBinding
    private lateinit var adapter: ProductAdapter

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        binding = FragmentProductBinding.bind(view)

        setupRecyclerView()
        observeState()
        observeEffect()
        setupListeners()

        // 최초 진입 시 Intent 전송
        viewModel.handleIntent(ProductIntent.LoadProducts)
    }

    // ① 상태 구독 — State가 바뀔 때마다 UI 갱신
    private fun observeState() {
        viewLifecycleOwner.lifecycleScope.launch {
            viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.state.collect { state ->
                    renderState(state)
                }
            }
        }
    }

    private fun renderState(state: ProductState) {
        with(binding) {
            progressBar.isVisible = state.isLoading
            recyclerView.isVisible = !state.isLoading && state.error == null
            tvError.isVisible = state.error != null
            tvError.text = state.error

            adapter.submitList(state.products)

            if (searchView.query.toString() != state.query) {
                searchView.setQuery(state.query, false)
            }
        }
    }

    // ② 사이드 이펙트 구독 — 일회성 이벤트 처리
    private fun observeEffect() {
        viewLifecycleOwner.lifecycleScope.launch {
            viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.effect.collect { effect ->
                    handleEffect(effect)
                }
            }
        }
    }

    private fun handleEffect(effect: ProductEffect) {
        when (effect) {
            is ProductEffect.ShowToast ->
                Toast.makeText(requireContext(), effect.message, Toast.LENGTH_SHORT).show()

            is ProductEffect.NavigateToDetail ->
                findNavController().navigate(
                    ProductFragmentDirections.actionToDetail(effect.productId)
                )

            is ProductEffect.NavigateToCart ->
                findNavController().navigate(R.id.action_to_cart)
        }
    }

    // ③ 사용자 행동 → Intent 전송
    private fun setupListeners() {
        binding.btnRefresh.setOnClickListener {
            viewModel.handleIntent(ProductIntent.RefreshProducts)
        }

        binding.searchView.setOnQueryTextListener(object : SearchView.OnQueryTextListener {
            override fun onQueryTextSubmit(query: String?): Boolean {
                viewModel.handleIntent(ProductIntent.SearchProducts(query.orEmpty()))
                return true
            }
            override fun onQueryTextChange(newText: String?) = false
        })

        binding.fabCart.setOnClickListener {
            viewModel.handleIntent(ProductIntent.NavigateToCart.also {
                // Effect로 처리하므로 별도 Intent 불필요
            })
        }
    }

    private fun setupRecyclerView() {
        adapter = ProductAdapter(
            onItemClick = { product ->
                viewModel.handleIntent(ProductIntent.AddToCart(product))
            },
            onFavoriteClick = { product ->
                viewModel.handleIntent(ProductIntent.ToggleFavorite(product.id))
            }
        )
        binding.recyclerView.adapter = adapter
    }
}
```

---

## 6. Reducer 패턴으로 상태 변경 분리

상태 변경 로직이 복잡해지면 **Reducer**를 별도로 분리할 수 있습니다.

```kotlin
// Reducer — 현재 상태 + 액션 → 새 상태
object ProductReducer {

    fun reduce(state: ProductState, action: ProductAction): ProductState = when (action) {
        is ProductAction.LoadingStarted  -> state.copy(isLoading = true, error = null)
        is ProductAction.ProductsLoaded  -> state.copy(isLoading = false, products = action.products)
        is ProductAction.LoadingFailed   -> state.copy(isLoading = false, error = action.message)
        is ProductAction.QueryChanged    -> state.copy(query = action.query)
        is ProductAction.FavoriteToggled -> state.copy(
            products = state.products.map { p ->
                if (p.id == action.productId) p.copy(isFavorite = !p.isFavorite) else p
            }
        )
    }
}

// 내부 액션 (Intent와 구분)
sealed class ProductAction {
    object LoadingStarted : ProductAction()
    data class ProductsLoaded(val products: List<Product>) : ProductAction()
    data class LoadingFailed(val message: String) : ProductAction()
    data class QueryChanged(val query: String) : ProductAction()
    data class FavoriteToggled(val productId: Long) : ProductAction()
}
```

```kotlin
// ViewModel에서 Reducer 사용
private fun dispatch(action: ProductAction) {
    _state.update { currentState -> ProductReducer.reduce(currentState, action) }
}

private fun loadProducts() {
    viewModelScope.launch {
        dispatch(ProductAction.LoadingStarted)
        getProductsUseCase()
            .onSuccess { dispatch(ProductAction.ProductsLoaded(it)) }
            .onFailure { dispatch(ProductAction.LoadingFailed(it.message ?: "오류")) }
    }
}
```

- Reducer는 **순수 함수** 로, 같은 State + Action이면 항상 같은 결과를 반환합니다.
- 상태 변경 로직을 ViewModel 밖으로 꺼내 단독으로 테스트할 수 있습니다.

---

## 7. 테스트

### ViewModel 단위 테스트

```kotlin
class ProductViewModelTest {

    private lateinit var viewModel: ProductViewModel
    private val fakeGetProductsUseCase = FakeGetProductsUseCase()
    private val fakeAddToCartUseCase = FakeAddToCartUseCase()

    @Before
    fun setup() {
        Dispatchers.setMain(UnconfinedTestDispatcher())
        viewModel = ProductViewModel(fakeGetProductsUseCase, fakeAddToCartUseCase)
    }

    @Test
    fun `LoadProducts Intent → 상품 목록이 State에 반영된다`() = runTest {
        // given
        val products = listOf(Product(1L, "상품 A"), Product(2L, "상품 B"))
        fakeGetProductsUseCase.result = Result.success(products)

        // when
        viewModel.handleIntent(ProductIntent.LoadProducts)

        // then
        val state = viewModel.state.value
        assert(!state.isLoading)
        assert(state.products == products)
        assert(state.error == null)
    }

    @Test
    fun `LoadProducts 실패 → error State와 ShowToast Effect가 발행된다`() = runTest {
        // given
        fakeGetProductsUseCase.result = Result.failure(Exception("서버 오류"))

        // when
        viewModel.handleIntent(ProductIntent.LoadProducts)

        // then
        val state = viewModel.state.value
        assert(state.error == "서버 오류")

        val effect = viewModel.effect.first()
        assert(effect is ProductEffect.ShowToast)
    }

    @Test
    fun `ToggleFavorite → 해당 상품의 isFavorite이 반전된다`() = runTest {
        // given
        val product = Product(1L, "상품 A", isFavorite = false)
        fakeGetProductsUseCase.result = Result.success(listOf(product))
        viewModel.handleIntent(ProductIntent.LoadProducts)

        // when
        viewModel.handleIntent(ProductIntent.ToggleFavorite(1L))

        // then
        val updatedProduct = viewModel.state.value.products.first { it.id == 1L }
        assert(updatedProduct.isFavorite)
    }

    @After
    fun tearDown() {
        Dispatchers.resetMain()
    }
}
```

### Reducer 단위 테스트

```kotlin
class ProductReducerTest {

    @Test
    fun `LoadingStarted → isLoading true, error null`() {
        val state = ProductState(error = "이전 오류")
        val result = ProductReducer.reduce(state, ProductAction.LoadingStarted)
        assert(result.isLoading)
        assert(result.error == null)
    }

    @Test
    fun `ProductsLoaded → isLoading false, products 반영`() {
        val products = listOf(Product(1L, "상품 A"))
        val state = ProductState(isLoading = true)
        val result = ProductReducer.reduce(state, ProductAction.ProductsLoaded(products))
        assert(!result.isLoading)
        assert(result.products == products)
    }
}
```

---

## 8. MVI 적용 시 주의사항

### ❌ State에 일회성 이벤트를 넣지 않는다

```kotlin
// ❌ 토스트 메시지를 State에 넣으면 화면 복원 시 다시 표시됨
data class ProductState(
    val toastMessage: String? = null  // 위험
)

// ✅ Effect Channel로 분리
_effect.send(ProductEffect.ShowToast("메시지"))
```

---

### ❌ View에서 상태를 직접 변경하지 않는다

```kotlin
// ❌ View가 직접 상태 조작
viewModel.state.value.products.toMutableList().add(newProduct)

// ✅ Intent를 통해 ViewModel에 위임
viewModel.handleIntent(ProductIntent.AddProduct(newProduct))
```

---

### ✅ State는 항상 불변으로 유지한다

```kotlin
// ❌ 가변 컬렉션 사용
data class ProductState(
    val products: MutableList<Product> = mutableListOf()  // 위험
)

// ✅ 불변 컬렉션 사용
data class ProductState(
    val products: List<Product> = emptyList()
)
```

---

## 9. MVI 전체 흐름 요약

```
┌─────────────────────────────────────────────────────┐
│                        View                         │
│  - State 구독 → renderState()                       │
│  - Effect 구독 → handleEffect()                     │
│  - 사용자 행동 → handleIntent(Intent) 전송           │
└──────────────────────┬──────────────────────────────┘
                       │ Intent
                       ▼
┌─────────────────────────────────────────────────────┐
│                    ViewModel                        │
│  - handleIntent() 로 Intent 수신                    │
│  - UseCase 호출 (비즈니스 로직)                      │
│  - Reducer로 State 계산                              │
│  - StateFlow로 State 발행                           │
│  - Channel로 Effect 발행                            │
└──────────┬────────────────────────────┬─────────────┘
           │ State (StateFlow)          │ Effect (Channel)
           ▼                            ▼
      View 렌더링                  일회성 이벤트 처리
   (반복 수신 가능)              (토스트·네비게이션 등)
```

---

## 10. 정리

| 항목 | 내용 |
|------|------|
| 데이터 흐름 | 단방향 (Intent → ViewModel → State → View) |
| 상태 관리 | 단일 State 객체 (data class) |
| 행동 표현 | sealed class Intent |
| 상태 발행 | StateFlow |
| 사이드 이펙트 | sealed class Effect + Channel |
| 상태 변경 | Reducer (순수 함수) |
| 테스트 | ViewModel·Reducer 독립 테스트 가능 |

- MVI는 **상태를 예측 가능하고 추적 가능하게 만드는 패턴** 입니다.
- 복잡한 화면, 상태가 많은 화면일수록 MVI의 장점이 두드러집니다.
- 단순한 화면에서는 오버엔지니어링이 될 수 있으니 팀 상황에 맞게 도입하세요.

---

## 참고

- [Android Architecture Guides](https://developer.android.com/topic/architecture)
- [Kotlin StateFlow 공식 문서](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-state-flow/)
- [Kotlin Channel 공식 문서](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-channel/)
