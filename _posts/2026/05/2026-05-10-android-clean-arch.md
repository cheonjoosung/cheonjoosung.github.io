---
title: (Android/안드로이드) 클린 아키텍처(Clean Architecture) 완전 정리
tags: [ Android, Kotlin ]
style: fill
color: dark
description: Android 클린 아키텍처의 개념, Presentation·Domain·Data 레이어 설계, UseCase·Repository 패턴, MVVM과의 조합, 멀티 모듈 구성까지 깊이 있게 정리합니다.
---

## 개요

- Android 앱 설계 원칙 중 **클린 아키텍처(Clean Architecture)** 를 다룹니다.
- 클린 아키텍처는 **관심사 분리(Separation of Concerns)** 를 통해 코드의 테스트 용이성·유지보수성·확장성을 높이는 설계 원칙입니다.
- 이 글에서는 다음을 설명합니다.
  - 클린 아키텍처의 핵심 원칙과 계층 구조
  - Presentation / Domain / Data 레이어 설계
  - UseCase 와 Repository 패턴 구현
  - MVVM 패턴과의 조합
  - 멀티 모듈 프로젝트 구성

---

## 1. 클린 아키텍처란

Robert C. Martin(Uncle Bob)이 제안한 설계 원칙으로, 코드를 **동심원(레이어)** 으로 나눠 **바깥쪽이 안쪽에만 의존** 하도록 강제합니다.

```
┌─────────────────────────────────────────┐
│           Presentation Layer            │  ← UI, ViewModel, State
│  ┌───────────────────────────────────┐  │
│  │          Domain Layer             │  │  ← UseCase, Entity, Repository 인터페이스
│  │  ┌─────────────────────────────┐  │  │
│  │  │        Data Layer           │  │  │  ← Repository 구현, API, DB
│  │  └─────────────────────────────┘  │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘

의존 방향: Presentation → Domain ← Data
                   (Data는 Domain에 의존, Presentation도 Domain에 의존)
```

### 핵심 규칙

| 규칙 | 설명 |
|------|------|
| **의존성 규칙** | 안쪽 레이어는 바깥쪽을 몰라야 한다 |
| **Domain 독립성** | Domain은 Android 프레임워크에 의존하지 않는다 |
| **경계(Boundary)** | 레이어 간 통신은 인터페이스(추상화)를 통해 이루어진다 |

---

## 2. 레이어별 역할

### Presentation Layer

- **무엇을 담나:** Activity, Fragment, ViewModel, UI State, UI Event
- **역할:** 사용자 입력을 받아 UseCase를 호출하고, 결과를 화면에 표시
- **의존:** Domain Layer만 알고, Data Layer는 모름

### Domain Layer

- **무엇을 담나:** UseCase, Entity(도메인 모델), Repository 인터페이스
- **역할:** 앱의 비즈니스 규칙 정의 — Android SDK, Retrofit, Room 등에 **일절 의존하지 않음**
- **의존:** 순수 Kotlin만 사용 (플랫폼 독립)

### Data Layer

- **무엇을 담나:** Repository 구현체, Remote DataSource(Retrofit), Local DataSource(Room), DTO, Mapper
- **역할:** 실제 데이터를 가져와 Domain 모델로 변환해 반환
- **의존:** Domain Layer의 Repository 인터페이스를 구현

---

## 3. 프로젝트 패키지 구조

```
app/
├── presentation/
│   ├── product/
│   │   ├── ProductFragment.kt
│   │   ├── ProductViewModel.kt
│   │   └── ProductUiState.kt
│   └── cart/
│       ├── CartFragment.kt
│       └── CartViewModel.kt
│
├── domain/
│   ├── model/
│   │   ├── Product.kt          ← 도메인 엔티티
│   │   └── CartItem.kt
│   ├── repository/
│   │   ├── ProductRepository.kt  ← 인터페이스
│   │   └── CartRepository.kt
│   └── usecase/
│       ├── GetProductsUseCase.kt
│       ├── AddToCartUseCase.kt
│       └── GetCartItemsUseCase.kt
│
└── data/
    ├── remote/
    │   ├── ProductApi.kt
    │   └── dto/
    │       └── ProductDto.kt
    ├── local/
    │   ├── ProductDao.kt
    │   └── entity/
    │       └── ProductEntity.kt
    ├── mapper/
    │   └── ProductMapper.kt
    └── repository/
        ├── ProductRepositoryImpl.kt
        └── CartRepositoryImpl.kt
```

---

## 4. Domain Layer 구현

### 도메인 엔티티 — 순수 Kotlin 데이터 클래스

```kotlin
// domain/model/Product.kt
data class Product(
    val id: Long,
    val name: String,
    val price: Int,
    val imageUrl: String,
    val category: String,
    val stock: Int,
    val isFavorite: Boolean = false
) {
    // 비즈니스 규칙을 엔티티 안에 캡슐화
    val isInStock: Boolean get() = stock > 0
    val discountedPrice: Int get() = if (price >= 10_000) (price * 0.9).toInt() else price
}
```

---

### Repository 인터페이스 — Domain이 계약을 정의

```kotlin
// domain/repository/ProductRepository.kt
interface ProductRepository {
    suspend fun getProducts(category: String? = null): Result<List<Product>>
    suspend fun getProduct(productId: Long): Result<Product>
    suspend fun toggleFavorite(productId: Long): Result<Unit>
    fun observeProducts(): Flow<List<Product>>  // 실시간 관찰
}
```

---

### UseCase — 단일 비즈니스 로직 단위

```kotlin
// domain/usecase/GetProductsUseCase.kt
class GetProductsUseCase(
    private val productRepository: ProductRepository
) {
    // 'operator fun invoke' — useCase() 형태로 호출 가능
    suspend operator fun invoke(category: String? = null): Result<List<Product>> {
        return productRepository.getProducts(category)
            .map { products ->
                // 비즈니스 규칙: 재고 있는 상품 먼저, 이름 순 정렬
                products
                    .sortedWith(compareByDescending<Product> { it.isInStock }.thenBy { it.name })
            }
    }
}
```

```kotlin
// domain/usecase/AddToCartUseCase.kt
class AddToCartUseCase(
    private val cartRepository: CartRepository,
    private val productRepository: ProductRepository
) {
    suspend operator fun invoke(productId: Long, quantity: Int = 1): Result<Unit> {
        // 비즈니스 규칙: 재고 확인 후 장바구니 추가
        val productResult = productRepository.getProduct(productId)

        return productResult.fold(
            onSuccess = { product ->
                if (!product.isInStock) {
                    Result.failure(IllegalStateException("품절된 상품입니다"))
                } else {
                    cartRepository.addItem(productId, quantity)
                }
            },
            onFailure = { Result.failure(it) }
        )
    }
}
```

- UseCase는 **하나의 비즈니스 동작만** 담당합니다.
- Repository를 직접 조합하거나 검증 로직을 담을 수 있습니다.
- 순수 Kotlin으로 작성되어 JUnit으로 단독 테스트가 가능합니다.

---

## 5. Data Layer 구현

### DTO — API 응답 모델

```kotlin
// data/remote/dto/ProductDto.kt
data class ProductDto(
    @SerializedName("product_id") val productId: Long,
    @SerializedName("product_name") val productName: String,
    @SerializedName("product_price") val productPrice: Int,
    @SerializedName("image_url") val imageUrl: String,
    @SerializedName("category_name") val categoryName: String,
    @SerializedName("stock_count") val stockCount: Int
)
```

---

### Mapper — DTO ↔ 도메인 엔티티 변환

```kotlin
// data/mapper/ProductMapper.kt
fun ProductDto.toDomain(): Product = Product(
    id = productId,
    name = productName,
    price = productPrice,
    imageUrl = imageUrl,
    category = categoryName,
    stock = stockCount
)

fun ProductEntity.toDomain(): Product = Product(
    id = id,
    name = name,
    price = price,
    imageUrl = imageUrl,
    category = category,
    stock = stock,
    isFavorite = isFavorite
)

fun Product.toEntity(): ProductEntity = ProductEntity(
    id = id,
    name = name,
    price = price,
    imageUrl = imageUrl,
    category = category,
    stock = stock,
    isFavorite = isFavorite
)
```

---

### Repository 구현체

```kotlin
// data/repository/ProductRepositoryImpl.kt
class ProductRepositoryImpl(
    private val productApi: ProductApi,       // Remote DataSource
    private val productDao: ProductDao        // Local DataSource
) : ProductRepository {

    override suspend fun getProducts(category: String?): Result<List<Product>> {
        return runCatching {
            // 네트워크 우선, 실패 시 로컬 캐시 반환
            try {
                val remoteProducts = productApi.getProducts(category)
                    .map { it.toDomain() }

                // 로컬 캐시 갱신
                productDao.deleteAll()
                productDao.insertAll(remoteProducts.map { it.toEntity() })

                remoteProducts
            } catch (e: Exception) {
                // 오프라인: 로컬 캐시 반환
                productDao.getAll().map { it.toDomain() }
                    .takeIf { it.isNotEmpty() }
                    ?: throw e
            }
        }
    }

    override suspend fun getProduct(productId: Long): Result<Product> {
        return runCatching {
            productDao.getProduct(productId)?.toDomain()
                ?: productApi.getProduct(productId).toDomain()
        }
    }

    override suspend fun toggleFavorite(productId: Long): Result<Unit> {
        return runCatching {
            productDao.toggleFavorite(productId)
        }
    }

    override fun observeProducts(): Flow<List<Product>> {
        return productDao.observeAll().map { entities ->
            entities.map { it.toDomain() }
        }
    }
}
```

---

## 6. Presentation Layer 구현 (MVVM + 클린 아키텍처)

```kotlin
// presentation/product/ProductUiState.kt
data class ProductUiState(
    val products: List<Product> = emptyList(),
    val isLoading: Boolean = false,
    val errorMessage: String? = null,
    val selectedCategory: String? = null
)

sealed class ProductUiEvent {
    data class ShowToast(val message: String) : ProductUiEvent()
    data class NavigateToDetail(val productId: Long) : ProductUiEvent()
}
```

```kotlin
// presentation/product/ProductViewModel.kt
@HiltViewModel
class ProductViewModel @Inject constructor(
    private val getProductsUseCase: GetProductsUseCase,     // UseCase 주입
    private val addToCartUseCase: AddToCartUseCase
) : ViewModel() {

    private val _uiState = MutableStateFlow(ProductUiState())
    val uiState: StateFlow<ProductUiState> = _uiState.asStateFlow()

    private val _uiEvent = Channel<ProductUiEvent>(Channel.BUFFERED)
    val uiEvent = _uiEvent.receiveAsFlow()

    init {
        loadProducts()
    }

    fun loadProducts(category: String? = null) {
        viewModelScope.launch {
            _uiState.update { it.copy(isLoading = true, errorMessage = null) }

            // UseCase 호출 — ViewModel은 비즈니스 규칙을 몰라도 됨
            getProductsUseCase(category)
                .onSuccess { products ->
                    _uiState.update {
                        it.copy(isLoading = false, products = products, selectedCategory = category)
                    }
                }
                .onFailure { error ->
                    _uiState.update { it.copy(isLoading = false, errorMessage = error.message) }
                    _uiEvent.send(ProductUiEvent.ShowToast("상품을 불러오지 못했습니다"))
                }
        }
    }

    fun addToCart(productId: Long) {
        viewModelScope.launch {
            addToCartUseCase(productId)
                .onSuccess {
                    _uiEvent.send(ProductUiEvent.ShowToast("장바구니에 담았습니다"))
                }
                .onFailure { error ->
                    _uiEvent.send(ProductUiEvent.ShowToast(error.message ?: "오류가 발생했습니다"))
                }
        }
    }

    fun onProductClick(productId: Long) {
        viewModelScope.launch {
            _uiEvent.send(ProductUiEvent.NavigateToDetail(productId))
        }
    }
}
```

---

## 7. Hilt 의존성 주입 설정

```kotlin
// di/DomainModule.kt
@Module
@InstallIn(ViewModelComponent::class)
object DomainModule {

    @Provides
    fun provideGetProductsUseCase(
        productRepository: ProductRepository
    ): GetProductsUseCase = GetProductsUseCase(productRepository)

    @Provides
    fun provideAddToCartUseCase(
        cartRepository: CartRepository,
        productRepository: ProductRepository
    ): AddToCartUseCase = AddToCartUseCase(cartRepository, productRepository)
}

// di/DataModule.kt
@Module
@InstallIn(SingletonComponent::class)
object DataModule {

    @Provides
    @Singleton
    fun provideProductRepository(
        productApi: ProductApi,
        productDao: ProductDao
    ): ProductRepository = ProductRepositoryImpl(productApi, productDao)

    @Provides
    @Singleton
    fun provideCartRepository(
        cartDao: CartDao
    ): CartRepository = CartRepositoryImpl(cartDao)
}
```

---

## 8. 멀티 모듈 구성 (고급)

규모가 커지면 레이어를 **독립 Gradle 모듈** 로 분리해 빌드 속도와 의존성 방향을 명확히 할 수 있습니다.

```
:app                    ← 진입점 (모든 모듈 조합)
:presentation           ← Fragment, ViewModel, UI
:domain                 ← UseCase, Entity, Repository 인터페이스
:data                   ← Repository 구현체, API, DB
:core:network           ← Retrofit, OkHttp 공통 설정
:core:database          ← Room 공통 설정
:core:ui                ← 공통 UI 컴포넌트
```

```kotlin
// domain/build.gradle.kts — Android 의존성 없음
plugins {
    id("java-library")          // 순수 Kotlin 모듈
    alias(libs.plugins.kotlin.jvm)
}

dependencies {
    implementation(libs.kotlinx.coroutines.core)
    // Retrofit, Room, Android SDK 의존 없음
}
```

```kotlin
// data/build.gradle.kts
plugins {
    alias(libs.plugins.android.library)
    alias(libs.plugins.kotlin.android)
}

dependencies {
    implementation(project(":domain"))  // Domain에 의존
    implementation(libs.retrofit)
    implementation(libs.room.runtime)
}
```

---

## 9. 테스트

### UseCase 단위 테스트 — 순수 Kotlin, Android 불필요

```kotlin
class GetProductsUseCaseTest {

    private val fakeProductRepository = FakeProductRepository()
    private val useCase = GetProductsUseCase(fakeProductRepository)

    @Test
    fun `재고 있는 상품이 먼저 정렬된다`() = runTest {
        // given
        fakeProductRepository.products = listOf(
            Product(1L, "품절 상품", 5000, "", "", stock = 0),
            Product(2L, "재고 상품 B", 3000, "", "", stock = 5),
            Product(3L, "재고 상품 A", 4000, "", "", stock = 3)
        )

        // when
        val result = useCase()

        // then
        val products = result.getOrThrow()
        assert(products[0].name == "재고 상품 A")  // 재고 있는 것 중 이름 순
        assert(products[1].name == "재고 상품 B")
        assert(products[2].name == "품절 상품")    // 품절은 마지막
    }

    @Test
    fun `AddToCartUseCase — 품절 상품은 실패를 반환한다`() = runTest {
        // given
        val outOfStockProduct = Product(1L, "품절 상품", 5000, "", "", stock = 0)
        fakeProductRepository.productDetail = outOfStockProduct
        val addToCartUseCase = AddToCartUseCase(FakeCartRepository(), fakeProductRepository)

        // when
        val result = addToCartUseCase(productId = 1L)

        // then
        assert(result.isFailure)
        assert(result.exceptionOrNull()?.message == "품절된 상품입니다")
    }
}
```

---

### Repository 테스트 — Fake DataSource 활용

```kotlin
class ProductRepositoryImplTest {

    private val fakeApi = FakeProductApi()
    private val fakeDao = FakeProductDao()
    private val repository = ProductRepositoryImpl(fakeApi, fakeDao)

    @Test
    fun `API 성공 시 로컬 DB에 캐싱한다`() = runTest {
        // given
        val dtos = listOf(ProductDto(1L, "상품 A", 5000, "", "", 10))
        fakeApi.productsResponse = dtos

        // when
        repository.getProducts()

        // then
        assert(fakeDao.insertedEntities.size == 1)
        assert(fakeDao.insertedEntities[0].name == "상품 A")
    }

    @Test
    fun `API 실패 시 로컬 캐시를 반환한다`() = runTest {
        // given
        fakeApi.shouldThrow = true
        fakeDao.cachedEntities = listOf(ProductEntity(1L, "캐시 상품", 3000, "", "", 5))

        // when
        val result = repository.getProducts()

        // then
        assert(result.isSuccess)
        assert(result.getOrThrow()[0].name == "캐시 상품")
    }
}
```

---

## 10. 클린 아키텍처 체크리스트

| 항목 | 확인 |
|------|------|
| Domain이 Android SDK를 import하지 않는가 | ✅ |
| Repository가 인터페이스(Domain)로 정의되는가 | ✅ |
| ViewModel이 UseCase만 알고 Repository 구현을 모르는가 | ✅ |
| DTO와 도메인 엔티티가 분리되어 있는가 | ✅ |
| UseCase가 단일 책임을 갖는가 | ✅ |
| 각 레이어가 단독으로 테스트 가능한가 | ✅ |

---

## 11. 클린 아키텍처 vs 일반 MVVM 비교

| 항목 | 일반 MVVM | 클린 아키텍처 + MVVM |
|------|-----------|---------------------|
| ViewModel이 Repository를 직접 호출 | O | X (UseCase 경유) |
| 비즈니스 규칙 위치 | ViewModel 혼재 | Domain UseCase에 집중 |
| Domain 독립성 | 없음 | Android SDK 의존 없음 |
| 테스트 속도 | Android 에뮬레이터 필요 | 순수 JVM 테스트 |
| 초기 구조 복잡도 | 낮음 | 높음 |
| 규모 확장성 | 중소규모 적합 | 중대규모 적합 |

---

## 12. 정리

| 레이어 | 핵심 구성 요소 | 의존 방향 |
|--------|--------------|----------|
| Presentation | ViewModel, UiState, Fragment | → Domain |
| Domain | UseCase, Entity, Repository 인터페이스 | 없음 (독립) |
| Data | Repository 구현체, DTO, Mapper, API, DAO | → Domain |

- 클린 아키텍처의 핵심은 **의존성 방향 통제** 입니다 — 안쪽(Domain)은 바깥을 모릅니다.
- 작은 프로젝트에 적용하면 오버엔지니어링이 될 수 있습니다. 팀 규모·앱 복잡도에 맞게 단계적으로 도입하세요.
- MVVM + Repository까지만 적용하다가, 비즈니스 로직이 복잡해지면 UseCase를 추가하는 방식으로 점진적으로 발전시키는 것을 권장합니다.

---

## 참고

- [Android Architecture Guides](https://developer.android.com/topic/architecture)
- [Now in Android 샘플 (Google)](https://github.com/android/nowinandroid)
- [Clean Architecture — Robert C. Martin](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
- [Hilt 공식 문서](https://developer.android.com/training/dependency-injection/hilt-android)
