---
title: (Android) MVP 패턴 완전 정리
tags: [ Android, Kotlin ]
style: fill
color: dark
description: Android에서 MVP(Model-View-Presenter) 패턴의 개념, 구성 요소, Contract 인터페이스 설계, 실전 구현, 테스트까지 깊이 있게 정리합니다.
---

## 개요

- Android 아키텍처 패턴 중 **MVP(Model-View-Presenter)** 를 다룹니다.
- MVP는 UI 로직을 View에서 분리해 **테스트 가능한 Presenter** 에 위임하는 패턴입니다.
- 이 글에서는 다음을 설명합니다.
  - MVP 구성 요소와 데이터 흐름
  - MVC와의 차이점
  - Contract 인터페이스 설계
  - Activity / Fragment 기반 실전 구현
  - 메모리 누수 방지와 생명주기 처리
  - MVVM / MVI와의 비교

---

## 1. MVP란

| 구성 요소 | 역할 |
|-----------|------|
| **Model** | 데이터 및 비즈니스 로직 — Repository, UseCase, Data Source |
| **View** | UI 표시와 사용자 입력 수신 — Activity, Fragment, CustomView |
| **Presenter** | View와 Model 사이의 중재자 — UI 로직 처리, View에 결과 전달 |

```
사용자 행동
    ↓
  View  ──────→  Presenter  ──────→  Model
  (Activity)    (UI 로직)      (Repository/UseCase)
    ↑                │
    └────────────────┘
         View 업데이트 지시
```

- View는 Presenter를 알고, Presenter는 View 인터페이스를 알고, Model은 View를 모릅니다.
- Presenter에 Android 프레임워크 의존성이 없어 **순수 JUnit으로 테스트** 가 가능합니다.

---

## 2. MVC와의 차이

| 항목 | MVC | MVP |
|------|-----|-----|
| UI 로직 위치 | Controller (Activity) | Presenter (별도 클래스) |
| View ↔ Model 직접 참조 | 가능 | 불가 (Presenter가 중재) |
| 테스트 | Activity 의존으로 어려움 | Presenter 단독 테스트 가능 |
| View 업데이트 | Controller가 직접 변경 | Presenter가 인터페이스로 지시 |

- Android에서 MVC를 적용하면 Activity가 Controller 역할까지 맡아 비대해집니다.
- MVP는 UI 로직을 Presenter로 분리해 Activity를 얇게 유지합니다.

---

## 3. Contract 인터페이스 설계

MVP의 핵심은 **View와 Presenter의 계약(Contract)을 인터페이스로 정의** 하는 것입니다.

```kotlin
interface ProductContract {

    interface View {
        fun showLoading()
        fun hideLoading()
        fun showProducts(products: List<Product>)
        fun showError(message: String)
        fun showEmptyView()
        fun showToast(message: String)
        fun navigateToDetail(productId: Long)
    }

    interface Presenter {
        fun onViewCreated()
        fun onSearchQueryChanged(query: String)
        fun onProductClicked(product: Product)
        fun onFavoriteClicked(product: Product)
        fun onRefreshRequested()
        fun onViewDestroyed()
    }
}
```

- `View` 인터페이스 — Presenter가 View에 무엇을 시킬 수 있는지 정의합니다.
- `Presenter` 인터페이스 — View가 Presenter에 무엇을 알릴 수 있는지 정의합니다.
- 둘을 하나의 `Contract`로 묶으면 관련 코드를 한곳에서 파악할 수 있습니다.

---

## 4. Model 구현

```kotlin
data class Product(
    val id: Long,
    val name: String,
    val price: Int,
    val imageUrl: String,
    val isFavorite: Boolean = false
)

// Repository 인터페이스 (DIP 적용)
interface ProductRepository {
    suspend fun getProducts(query: String = ""): Result<List<Product>>
    suspend fun toggleFavorite(productId: Long): Result<Unit>
}

// 실제 구현체
class ProductRepositoryImpl(
    private val api: ProductApi,
    private val dao: ProductDao
) : ProductRepository {
    override suspend fun getProducts(query: String): Result<List<Product>> =
        runCatching {
            if (query.isBlank()) api.fetchProducts()
            else api.searchProducts(query)
        }

    override suspend fun toggleFavorite(productId: Long): Result<Unit> =
        runCatching { dao.toggleFavorite(productId) }
}
```

---

## 5. Presenter 구현

```kotlin
class ProductPresenter(
    private val productRepository: ProductRepository,
    private val coroutineScope: CoroutineScope = CoroutineScope(Dispatchers.Main + SupervisorJob())
) : ProductContract.Presenter {

    // View는 WeakReference로 참조 — 메모리 누수 방지
    private var view: ProductContract.View? = null

    fun attachView(view: ProductContract.View) {
        this.view = view
    }

    fun detachView() {
        this.view = null
    }

    override fun onViewCreated() {
        loadProducts()
    }

    override fun onSearchQueryChanged(query: String) {
        loadProducts(query = query)
    }

    override fun onProductClicked(product: Product) {
        view?.navigateToDetail(product.id)
    }

    override fun onFavoriteClicked(product: Product) {
        coroutineScope.launch {
            productRepository.toggleFavorite(product.id)
                .onSuccess {
                    view?.showToast(
                        if (product.isFavorite) "즐겨찾기에서 제거했습니다"
                        else "즐겨찾기에 추가했습니다"
                    )
                }
                .onFailure {
                    view?.showToast("즐겨찾기 변경에 실패했습니다")
                }
        }
    }

    override fun onRefreshRequested() {
        loadProducts()
    }

    override fun onViewDestroyed() {
        detachView()
        coroutineScope.cancel()
    }

    private fun loadProducts(query: String = "") {
        coroutineScope.launch {
            view?.showLoading()

            productRepository.getProducts(query)
                .onSuccess { products ->
                    view?.hideLoading()
                    if (products.isEmpty()) view?.showEmptyView()
                    else view?.showProducts(products)
                }
                .onFailure { error ->
                    view?.hideLoading()
                    view?.showError(error.message ?: "알 수 없는 오류가 발생했습니다")
                }
        }
    }
}
```

**핵심 포인트:**

- `view`를 nullable로 관리해 `onViewDestroyed()` 후에도 안전하게 처리합니다.
- `coroutineScope.cancel()`로 View 소멸 시 진행 중인 작업을 중단합니다.
- Presenter에 `Context`, `Activity` 등 Android 의존성이 없습니다.

---

## 6. View(Activity) 구현

```kotlin
@AndroidEntryPoint
class ProductActivity : AppCompatActivity(), ProductContract.View {

    private lateinit var binding: ActivityProductBinding
    private lateinit var presenter: ProductPresenter
    private lateinit var adapter: ProductAdapter

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityProductBinding.inflate(layoutInflater)
        setContentView(binding.root)

        setupPresenter()
        setupRecyclerView()
        setupListeners()

        presenter.onViewCreated()
    }

    private fun setupPresenter() {
        val repository = ProductRepositoryImpl(
            api = RetrofitClient.productApi,
            dao = AppDatabase.getInstance(this).productDao()
        )
        presenter = ProductPresenter(repository)
        presenter.attachView(this)
    }

    override fun onDestroy() {
        super.onDestroy()
        presenter.onViewDestroyed()
    }

    // ── View 인터페이스 구현 ──────────────────────────────

    override fun showLoading() {
        binding.progressBar.isVisible = true
        binding.recyclerView.isVisible = false
        binding.tvError.isVisible = false
        binding.tvEmpty.isVisible = false
    }

    override fun hideLoading() {
        binding.progressBar.isVisible = false
    }

    override fun showProducts(products: List<Product>) {
        binding.recyclerView.isVisible = true
        binding.tvEmpty.isVisible = false
        adapter.submitList(products)
    }

    override fun showError(message: String) {
        binding.tvError.isVisible = true
        binding.tvError.text = message
    }

    override fun showEmptyView() {
        binding.tvEmpty.isVisible = true
        binding.recyclerView.isVisible = false
    }

    override fun showToast(message: String) {
        Toast.makeText(this, message, Toast.LENGTH_SHORT).show()
    }

    override fun navigateToDetail(productId: Long) {
        startActivity(ProductDetailActivity.newIntent(this, productId))
    }

    // ── 사용자 행동 → Presenter 전달 ─────────────────────

    private fun setupListeners() {
        binding.swipeRefreshLayout.setOnRefreshListener {
            presenter.onRefreshRequested()
            binding.swipeRefreshLayout.isRefreshing = false
        }

        binding.searchView.setOnQueryTextListener(object : SearchView.OnQueryTextListener {
            override fun onQueryTextSubmit(query: String?): Boolean {
                presenter.onSearchQueryChanged(query.orEmpty())
                return true
            }
            override fun onQueryTextChange(newText: String?) = false
        })
    }

    private fun setupRecyclerView() {
        adapter = ProductAdapter(
            onItemClick  = { product -> presenter.onProductClicked(product) },
            onFavoriteClick = { product -> presenter.onFavoriteClicked(product) }
        )
        binding.recyclerView.adapter = adapter
    }
}
```

---

## 7. View(Fragment) 구현

Fragment에서는 `onDestroyView()`에서 View를 분리합니다.

```kotlin
class ProductFragment : Fragment(R.layout.fragment_product), ProductContract.View {

    private var _binding: FragmentProductBinding? = null
    private val binding get() = _binding!!

    private lateinit var presenter: ProductPresenter
    private lateinit var adapter: ProductAdapter

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        _binding = FragmentProductBinding.bind(view)

        setupPresenter()
        setupRecyclerView()
        setupListeners()

        presenter.onViewCreated()
    }

    private fun setupPresenter() {
        val repository = ProductRepositoryImpl(
            api = RetrofitClient.productApi,
            dao = AppDatabase.getInstance(requireContext()).productDao()
        )
        // Fragment의 viewLifecycleOwner.lifecycleScope 전달
        presenter = ProductPresenter(
            productRepository = repository,
            coroutineScope = viewLifecycleOwner.lifecycleScope
        )
        presenter.attachView(this)
    }

    override fun onDestroyView() {
        super.onDestroyView()
        presenter.detachView()   // View 분리 — 메모리 누수 방지
        _binding = null
    }

    override fun onDestroy() {
        super.onDestroy()
        presenter.onViewDestroyed()
    }

    // View 인터페이스 구현 — Activity와 동일
    override fun showLoading()                          { binding.progressBar.isVisible = true }
    override fun hideLoading()                          { binding.progressBar.isVisible = false }
    override fun showProducts(products: List<Product>)  { adapter.submitList(products) }
    override fun showError(message: String)             { binding.tvError.text = message }
    override fun showEmptyView()                        { binding.tvEmpty.isVisible = true }
    override fun showToast(message: String)             { Toast.makeText(requireContext(), message, Toast.LENGTH_SHORT).show() }
    override fun navigateToDetail(productId: Long)      { findNavController().navigate(ProductFragmentDirections.actionToDetail(productId)) }

    private fun setupListeners() {
        binding.btnRefresh.setOnClickListener {
            presenter.onRefreshRequested()
        }
    }

    private fun setupRecyclerView() {
        adapter = ProductAdapter(
            onItemClick     = { product -> presenter.onProductClicked(product) },
            onFavoriteClick = { product -> presenter.onFavoriteClicked(product) }
        )
        binding.recyclerView.adapter = adapter
    }
}
```

---

## 8. 메모리 누수 방지

MVP의 가장 큰 위험은 **Presenter가 View(Activity/Fragment)를 강하게 참조해 메모리 누수가 발생** 하는 것입니다.

### ❌ 누수 발생 — 강한 참조

```kotlin
class ProductPresenter(
    private val view: ProductContract.View  // Activity를 직접 참조 — 누수 위험
) : ProductContract.Presenter {
    // 비동기 작업 중 Activity가 종료되면 GC 불가
}
```

---

### ✅ 방법 1 — onViewDestroyed()에서 null 처리

```kotlin
class ProductPresenter : ProductContract.Presenter {
    private var view: ProductContract.View? = null

    fun attachView(view: ProductContract.View) { this.view = view }
    fun detachView() { this.view = null }   // View 소멸 시 참조 제거

    // 모든 View 접근은 ?. 로 안전하게 처리
    private fun updateUI() {
        view?.showProducts(emptyList())
    }
}
```

---

### ✅ 방법 2 — WeakReference 사용

```kotlin
import java.lang.ref.WeakReference

class ProductPresenter : ProductContract.Presenter {
    private var viewRef: WeakReference<ProductContract.View>? = null

    fun attachView(view: ProductContract.View) {
        viewRef = WeakReference(view)
    }

    fun detachView() {
        viewRef?.clear()
        viewRef = null
    }

    private val view get() = viewRef?.get()

    private fun updateUI() {
        view?.showProducts(emptyList())  // GC 되면 자동으로 null
    }
}
```

---

## 9. 테스트

Presenter에 Android 의존성이 없으므로 **순수 JUnit + Mock으로 단위 테스트** 가 가능합니다.

```kotlin
class ProductPresenterTest {

    // View 목(Mock) — 인터페이스를 직접 구현
    private class FakeProductView : ProductContract.View {
        var isLoading = false
        var products: List<Product> = emptyList()
        var errorMessage: String? = null
        var isEmptyViewShown = false
        val toastMessages = mutableListOf<String>()
        var navigatedProductId: Long? = null

        override fun showLoading()                          { isLoading = true }
        override fun hideLoading()                          { isLoading = false }
        override fun showProducts(products: List<Product>)  { this.products = products }
        override fun showError(message: String)             { errorMessage = message }
        override fun showEmptyView()                        { isEmptyViewShown = true }
        override fun showToast(message: String)             { toastMessages.add(message) }
        override fun navigateToDetail(productId: Long)      { navigatedProductId = productId }
    }

    private lateinit var presenter: ProductPresenter
    private lateinit var fakeView: FakeProductView
    private val fakeRepository = FakeProductRepository()

    @Before
    fun setup() {
        fakeView = FakeProductView()
        presenter = ProductPresenter(
            productRepository = fakeRepository,
            coroutineScope = CoroutineScope(UnconfinedTestDispatcher())
        )
        presenter.attachView(fakeView)
    }

    @Test
    fun `onViewCreated → 상품 목록이 View에 표시된다`() = runTest {
        // given
        val products = listOf(Product(1L, "상품 A", 10000, ""))
        fakeRepository.result = Result.success(products)

        // when
        presenter.onViewCreated()

        // then
        assert(!fakeView.isLoading)
        assert(fakeView.products == products)
        assert(fakeView.errorMessage == null)
    }

    @Test
    fun `onViewCreated 실패 → 에러 메시지가 표시된다`() = runTest {
        // given
        fakeRepository.result = Result.failure(Exception("서버 오류"))

        // when
        presenter.onViewCreated()

        // then
        assert(!fakeView.isLoading)
        assert(fakeView.errorMessage == "서버 오류")
    }

    @Test
    fun `onViewCreated 결과 빈 배열 → 빈 화면이 표시된다`() = runTest {
        // given
        fakeRepository.result = Result.success(emptyList())

        // when
        presenter.onViewCreated()

        // then
        assert(fakeView.isEmptyViewShown)
    }

    @Test
    fun `onProductClicked → navigateToDetail 호출된다`() = runTest {
        // given
        val product = Product(42L, "상품 B", 5000, "")

        // when
        presenter.onProductClicked(product)

        // then
        assert(fakeView.navigatedProductId == 42L)
    }

    @Test
    fun `onViewDestroyed 이후 비동기 결과가 와도 View 접근하지 않는다`() = runTest {
        // when
        presenter.onViewDestroyed()
        fakeRepository.result = Result.success(listOf(Product(1L, "상품 A", 10000, "")))

        // View가 detach된 상태에서 작업 완료
        presenter.onViewCreated()

        // then — View 메서드 호출 없어야 함
        assert(fakeView.products.isEmpty())
    }

    @After
    fun tearDown() {
        presenter.onViewDestroyed()
    }
}
```

---

## 10. Hilt로 의존성 주입

```kotlin
// Module
@Module
@InstallIn(ActivityComponent::class)
object ProductModule {

    @Provides
    fun provideProductRepository(
        api: ProductApi,
        dao: ProductDao
    ): ProductRepository = ProductRepositoryImpl(api, dao)

    @Provides
    fun provideProductPresenter(
        repository: ProductRepository
    ): ProductPresenter = ProductPresenter(repository)
}

// Activity
@AndroidEntryPoint
class ProductActivity : AppCompatActivity(), ProductContract.View {

    @Inject lateinit var presenter: ProductPresenter

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        presenter.attachView(this)
        presenter.onViewCreated()
    }

    override fun onDestroy() {
        super.onDestroy()
        presenter.onViewDestroyed()
    }
}
```

---

## 11. MVP / MVVM / MVI 비교

| 항목 | MVP | MVVM | MVI |
|------|-----|------|-----|
| UI 로직 위치 | Presenter | ViewModel | ViewModel |
| View ↔ 로직 연결 | 인터페이스 | DataBinding / 옵저버 | StateFlow 구독 |
| 상태 표현 | 분산 메서드 호출 | LiveData / StateFlow | 단일 State 객체 |
| 데이터 흐름 | 양방향 | 양방향 가능 | 단방향 강제 |
| 테스트 | Presenter 단독 가능 | ViewModel 단독 가능 | ViewModel + Reducer |
| 생명주기 관리 | 수동 (attach/detach) | 자동 (ViewModel) | 자동 (ViewModel) |
| 메모리 누수 위험 | 있음 (주의 필요) | 낮음 | 낮음 |
| 적합한 규모 | 소~중 | 중 | 중~대 |

---

## 12. 정리

| 항목 | 내용 |
|------|------|
| 핵심 아이디어 | UI 로직을 Presenter로 분리해 Activity를 얇게 유지 |
| 계약 정의 | View·Presenter 인터페이스를 Contract로 묶어 관리 |
| 테스트 | Android 의존성 없는 Presenter를 순수 JUnit으로 테스트 |
| 누수 방지 | onDestroy/onDestroyView에서 detachView() 필수 |
| DI 연계 | Hilt로 Repository → Presenter 주입 |
| 한계 | 생명주기 수동 관리, 화면 복잡도 증가 시 Presenter 비대해짐 |

- MVP는 **"Activity는 View 역할만 한다"** 는 명확한 구분이 핵심입니다.
- 화면이 단순하고 빠른 개발이 필요할 때 적합하며, 복잡한 상태 관리가 필요하다면 MVVM 또는 MVI를 고려하세요.

---

## 참고

- [Android Architecture Guides](https://developer.android.com/topic/architecture)
- [MVI 패턴 포스팅 보기](/posts/2026-04-14-android-mvi)
