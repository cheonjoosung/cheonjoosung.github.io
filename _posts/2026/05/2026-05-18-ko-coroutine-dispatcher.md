---
title: (Kotlin/코틀린) 코루틴 Dispatcher 심화 - IO, Main, Default, Unconfined 완전 정리
tags: [ Kotlin, Android ]
style: fill
color: dark
description: 코루틴 Dispatcher의 동작 원리, IO·Main·Default·Unconfined의 차이, 스레드 풀 내부 구조, 커스텀 Dispatcher 생성, Android 실전 적용 패턴까지 깊이 있게 정리합니다.
---

## 개요

- 코루틴의 **Dispatcher(디스패처)** 를 깊이 있게 다룹니다.
- Dispatcher는 코루틴이 **어떤 스레드(풀)에서 실행될지** 결정합니다.
- 이 글에서는 다음을 설명합니다.
  - Dispatcher의 역할과 내부 구조
  - `Dispatchers.IO` — 네트워크·파일
  - `Dispatchers.Main` — UI 스레드
  - `Dispatchers.Default` — CPU 집중 연산
  - `Dispatchers.Unconfined` — 특수 목적
  - 커스텀 Dispatcher 생성
  - Android 실전 적용 패턴

---

## 1. Dispatcher란

Dispatcher는 `CoroutineContext`의 구성 요소로, **코루틴을 어떤 스레드에서 실행할지 결정하는 스케줄러**입니다.

```kotlin
// Dispatcher 지정 방법
viewModelScope.launch(Dispatchers.IO) { /* IO 스레드 */ }
viewModelScope.launch(Dispatchers.Default) { /* CPU 스레드 */ }

withContext(Dispatchers.IO) { /* IO 스레드로 전환 */ }
withContext(Dispatchers.Main) { /* 메인 스레드로 전환 */ }
```

### Dispatcher와 스레드 풀 관계

```
Dispatchers.Main     → 메인 스레드 1개
Dispatchers.Default  → CPU 코어 수만큼 (최소 2개)
Dispatchers.IO       → 최대 64개 (또는 CPU 코어 수 중 큰 값)
Dispatchers.Unconfined → 스레드 고정 없음
```

---

## 2. Dispatchers.IO

네트워크 요청, 파일 읽기/쓰기, 데이터베이스 쿼리처럼 **I/O 대기가 발생하는 작업**에 사용합니다.

### 내부 구조

```
Dispatchers.IO
├── 스레드 수: 최대 64개 (또는 Runtime.availableProcessors() 중 큰 값)
├── 특징: I/O 대기 중 스레드를 다른 코루틴에 재활용
└── Default와 스레드 풀 공유 (별도 풀 아님)
```

```kotlin
// IO — 네트워크, DB, 파일
suspend fun fetchUser(userId: Long): User {
    return withContext(Dispatchers.IO) {
        userApi.getUser(userId)   // 네트워크 대기 → 스레드를 다른 코루틴에 양보
    }
}

suspend fun readFile(path: String): String {
    return withContext(Dispatchers.IO) {
        File(path).readText()     // 파일 I/O 대기
    }
}

suspend fun queryDatabase(userId: Long): UserEntity? {
    return withContext(Dispatchers.IO) {
        userDao.getUser(userId)   // DB 쿼리 대기
    }
}
```

### 왜 IO 스레드가 많은가

```
IO 작업 특성:
- CPU는 거의 사용 안 함
- 대부분의 시간을 "대기" 상태로 보냄
    → 스레드를 많이 만들어도 CPU 경쟁 없음
    → 64개 스레드가 동시에 네트워크 대기 가능

Default 작업 특성:
- CPU를 지속적으로 사용
    → 스레드가 코어 수보다 많으면 컨텍스트 스위칭 비용만 늘어남
    → CPU 코어 수 = 최적 스레드 수
```

---

## 3. Dispatchers.Main

**메인(UI) 스레드**에서 코루틴을 실행합니다. Android에서는 뷰 업데이트, 애니메이션 제어 등 UI 작업에 사용합니다.

```kotlin
// Main — UI 업데이트
viewModelScope.launch(Dispatchers.Main) {
    binding.tvName.text = "홍길동"
    binding.progressBar.isVisible = false
}
```

### Main vs Main.immediate

```kotlin
// Dispatchers.Main
// — 현재 스레드가 메인이어도 dispatch(큐에 넣기) 후 실행
viewModelScope.launch(Dispatchers.Main) {
    println("메인에서 실행")   // 큐를 거쳐서 실행 (약간의 지연 가능)
}

// Dispatchers.Main.immediate
// — 이미 메인 스레드이면 즉시 실행 (dispatch 생략)
viewModelScope.launch(Dispatchers.Main.immediate) {
    println("즉시 실행")   // 메인 스레드에서 호출 시 즉시
}
```

| | `Main` | `Main.immediate` |
|--|--------|-----------------|
| 메인 스레드에서 호출 | 큐에 넣고 실행 | 즉시 실행 |
| 다른 스레드에서 호출 | 큐에 넣고 실행 | 큐에 넣고 실행 |
| viewModelScope 기본값 | | ✅ |

`viewModelScope`의 기본 Dispatcher가 `Main.immediate`인 이유는 **불필요한 지연 없이 즉시 UI를 업데이트**하기 위해서입니다.

### 실전 — IO 후 Main으로 전환

```kotlin
@HiltViewModel
class ProductViewModel @Inject constructor(
    private val repository: ProductRepository
) : ViewModel() {

    fun loadProduct(id: Long) {
        // viewModelScope 기본 = Main.immediate
        viewModelScope.launch {
            _uiState.update { it.copy(isLoading = true) }   // Main

            val product = withContext(Dispatchers.IO) {      // IO로 전환
                repository.getProduct(id)                    // 네트워크
            }                                               // 다시 Main

            _uiState.update { it.copy(product = product, isLoading = false) }  // Main
        }
    }
}
```

---

## 4. Dispatchers.Default

**CPU 집중 연산**에 사용합니다. 이미지 처리, JSON 파싱, 정렬, 암호화 등 계산량이 많은 작업에 적합합니다.

### 내부 구조

```
Dispatchers.Default
├── 스레드 수: Runtime.getRuntime().availableProcessors() (최소 2)
│   예) 8코어 CPU → 8개 스레드
├── 특징: CPU를 지속적으로 사용하는 작업에 최적화
└── IO와 스레드 풀 공유 (IO는 추가 스레드를 빌려 씀)
```

```kotlin
// Default — CPU 집중 작업
suspend fun parseProducts(json: String): List<Product> {
    return withContext(Dispatchers.Default) {
        Gson().fromJson(json, Array<Product>::class.java).toList()
    }
}

suspend fun sortProducts(products: List<Product>): List<Product> {
    return withContext(Dispatchers.Default) {
        products.sortedWith(
            compareByDescending<Product> { it.rating }.thenBy { it.name }
        )
    }
}

suspend fun encryptData(data: ByteArray, key: SecretKey): ByteArray {
    return withContext(Dispatchers.Default) {
        val cipher = Cipher.getInstance("AES/GCM/NoPadding")
        cipher.init(Cipher.ENCRYPT_MODE, key)
        cipher.doFinal(data)
    }
}

suspend fun resizeBitmap(bitmap: Bitmap, maxWidth: Int): Bitmap {
    return withContext(Dispatchers.Default) {
        val ratio = maxWidth.toFloat() / bitmap.width
        val height = (bitmap.height * ratio).toInt()
        Bitmap.createScaledBitmap(bitmap, maxWidth, height, true)
    }
}
```

### Default vs IO 선택 기준

```
CPU를 쉬지 않고 사용하는가?
    → YES: Dispatchers.Default
    → NO (대기 시간이 있는가?): Dispatchers.IO

예)
    JSON 파싱   → Default (CPU 연산)
    네트워크    → IO (네트워크 대기)
    이미지 처리 → Default (CPU 연산)
    파일 읽기   → IO (디스크 대기)
    DB 쿼리     → IO (DB 대기)
    암호화      → Default (CPU 연산)
    압축        → Default (CPU 연산)
```

---

## 5. Dispatchers.Unconfined

`Unconfined`는 **특정 스레드에 고정되지 않는 특수 Dispatcher**입니다.

### 동작 원리

```kotlin
// Unconfined — 처음엔 호출자 스레드, 중단 후엔 재개한 스레드에서 실행
viewModelScope.launch(Dispatchers.Unconfined) {
    println("1: ${Thread.currentThread().name}")   // main (호출자 스레드)
    delay(100)
    println("2: ${Thread.currentThread().name}")   // kotlinx.coroutines.DefaultExecutor
    // delay 이후 재개된 스레드에서 그대로 실행
}
```

```
호출 → [Main 스레드에서 시작]
          ↓ delay(중단)
       [재개 스레드에서 계속] ← 어떤 스레드인지 보장 없음
```

### Unconfined 주의사항

```kotlin
// ❌ Unconfined를 프로덕션 코드에 사용하지 않는다
viewModelScope.launch(Dispatchers.Unconfined) {
    delay(100)
    binding.tvName.text = "완료"   // 어떤 스레드인지 모름 → 크래시 가능
}

// ✅ 테스트에서 제어용으로만 사용
@Test
fun `즉시 실행 테스트`() = runTest {
    val scope = CoroutineScope(Dispatchers.Unconfined)
    // 테스트에서 시간 제어 목적으로 활용
}
```

| Dispatcher | 실제 사용 | 테스트 |
|------------|---------|--------|
| `Main` | ✅ UI 업데이트 | `TestCoroutineScheduler` |
| `IO` | ✅ 네트워크·DB | `UnconfinedTestDispatcher` |
| `Default` | ✅ CPU 연산 | `UnconfinedTestDispatcher` |
| `Unconfined` | ⚠️ 테스트 전용 | ✅ |

---

## 6. limitedParallelism — IO 스레드 수 제한

`Dispatchers.IO.limitedParallelism(n)`으로 동시에 실행할 코루틴 수를 제한할 수 있습니다.

```kotlin
// 데이터베이스 연결이 최대 4개인 경우
val dbDispatcher = Dispatchers.IO.limitedParallelism(4)

suspend fun queryWithLimit(id: Long): Data {
    return withContext(dbDispatcher) {
        db.query(id)   // 최대 4개 코루틴만 동시에 DB 접근
    }
}

// API Rate Limit 대응 — 초당 최대 2개 요청
val rateLimitedDispatcher = Dispatchers.IO.limitedParallelism(2)

suspend fun fetchWithRateLimit(url: String): Response {
    return withContext(rateLimitedDispatcher) {
        httpClient.get(url)
    }
}
```

---

## 7. 커스텀 Dispatcher

`ExecutorService`로 직접 Dispatcher를 만들 수 있습니다.

```kotlin
// 단일 스레드 Dispatcher — 순서 보장이 필요할 때
val singleThreadDispatcher = Executors.newSingleThreadExecutor()
    .asCoroutineDispatcher()

// 특정 크기의 스레드 풀
val customDispatcher = Executors.newFixedThreadPool(4)
    .asCoroutineDispatcher()

// 사용 후 반드시 닫아야 함 (ExecutorService 해제)
singleThreadDispatcher.close()
customDispatcher.close()

// 실전 — Hilt로 주입
@Module
@InstallIn(SingletonComponent::class)
object DispatcherModule {

    @IoDispatcher
    @Provides
    fun provideIoDispatcher(): CoroutineDispatcher = Dispatchers.IO

    @DefaultDispatcher
    @Provides
    fun provideDefaultDispatcher(): CoroutineDispatcher = Dispatchers.Default

    @MainDispatcher
    @Provides
    fun provideMainDispatcher(): CoroutineDispatcher = Dispatchers.Main
}

// Qualifier 정의
@Qualifier @Retention(AnnotationRetention.BINARY)
annotation class IoDispatcher

@Qualifier @Retention(AnnotationRetention.BINARY)
annotation class DefaultDispatcher

@Qualifier @Retention(AnnotationRetention.BINARY)
annotation class MainDispatcher

// Repository에서 주입받아 사용 → 테스트 시 교체 용이
class UserRepositoryImpl @Inject constructor(
    @IoDispatcher private val ioDispatcher: CoroutineDispatcher
) : UserRepository {
    override suspend fun getUser(id: Long): User {
        return withContext(ioDispatcher) {
            api.getUser(id)
        }
    }
}
```

---

## 8. 테스트에서 Dispatcher 교체

Hilt로 Dispatcher를 주입하면 테스트 시 **`UnconfinedTestDispatcher`** 로 교체해 즉시 실행할 수 있습니다.

```kotlin
class UserRepositoryTest {

    private val testDispatcher = UnconfinedTestDispatcher()

    @Test
    fun `getUser 성공 시 User 반환`() = runTest(testDispatcher) {
        // UnconfinedTestDispatcher로 교체 → delay 없이 즉시 실행
        val repository = UserRepositoryImpl(
            ioDispatcher = testDispatcher,   // 테스트용 Dispatcher 주입
            userApi = FakeUserApi()
        )

        val result = repository.getUser(1L)

        assertEquals("홍길동", result.name)
    }
}
```

---

## 9. 전체 비교표

| Dispatcher | 스레드 수 | 용도 | 금지 사항 |
|------------|---------|------|---------|
| `Main` | 1 (UI 스레드) | UI 업데이트, LiveData 발행 | 블로킹 작업 |
| `Main.immediate` | 1 (UI 스레드) | viewModelScope 기본값 | 블로킹 작업 |
| `IO` | 최대 64개 | 네트워크, DB, 파일 | CPU 집중 작업 |
| `Default` | CPU 코어 수 | JSON 파싱, 이미지 처리, 정렬 | 블로킹 IO |
| `Unconfined` | 고정 없음 | 테스트 전용 | 프로덕션 UI 작업 |

---

## 10. 실전 적용 — 전체 패턴

```kotlin
// Repository — IO 강제
class SearchRepositoryImpl @Inject constructor(
    @IoDispatcher private val ioDispatcher: CoroutineDispatcher,
    @DefaultDispatcher private val defaultDispatcher: CoroutineDispatcher,
    private val searchApi: SearchApi
) : SearchRepository {

    override suspend fun search(query: String): List<SearchResult> {
        // 네트워크 요청 — IO
        val rawResponse = withContext(ioDispatcher) {
            searchApi.search(query)
        }
        // JSON 파싱 + 정렬 — Default
        return withContext(defaultDispatcher) {
            rawResponse.items
                .map { it.toDomain() }
                .sortedByDescending { it.score }
        }
    }
}

// ViewModel — Dispatcher 신경 안 써도 됨
@HiltViewModel
class SearchViewModel @Inject constructor(
    private val searchRepository: SearchRepository
) : ViewModel() {

    fun search(query: String) {
        viewModelScope.launch {   // Main.immediate
            _uiState.update { it.copy(isLoading = true) }
            val results = searchRepository.search(query)   // 내부에서 IO+Default 처리
            _uiState.update { it.copy(isLoading = false, results = results) }
        }
    }
}
```

---

## 11. 주의사항

### ❌ Main 스레드에서 블로킹 작업

```kotlin
// ❌ Main 스레드에서 네트워크 → NetworkOnMainThreadException
viewModelScope.launch {   // Main
    val user = userApi.getUser(1L)   // 블로킹 → 크래시
}

// ✅ IO 스레드로 전환
viewModelScope.launch {
    val user = withContext(Dispatchers.IO) { userApi.getUser(1L) }
}
```

---

### ❌ CPU 작업에 IO Dispatcher 사용

```kotlin
// ❌ IO Dispatcher로 CPU 집중 작업 → 스레드 낭비
withContext(Dispatchers.IO) {
    repeat(1_000_000) { /* 계산 */ }   // IO 스레드 점유
}

// ✅ Default 사용
withContext(Dispatchers.Default) {
    repeat(1_000_000) { /* 계산 */ }
}
```

---

### ❌ Dispatcher 없이 Room suspend 함수 호출

```kotlin
// ✅ Room suspend 함수는 자체적으로 IO 처리 — withContext 불필요
@Dao
interface UserDao {
    @Query("SELECT * FROM users WHERE id = :id")
    suspend fun getUser(id: Long): UserEntity?   // Room이 IO 보장
}

// ❌ 중복 withContext — 불필요하지만 동작에 문제는 없음
suspend fun getUser(id: Long) = withContext(Dispatchers.IO) {
    userDao.getUser(id)   // 이미 IO에서 실행됨
}
```

---

## 12. 정리

| Dispatcher | 한 줄 요약 |
|------------|----------|
| `Main` | UI 스레드 — 뷰 접근은 여기서만 |
| `Main.immediate` | Main과 같지만 이미 메인이면 즉시 실행 |
| `IO` | 대기가 많은 작업 — 최대 64개 스레드 |
| `Default` | CPU 연산 — 코어 수만큼 스레드 |
| `Unconfined` | 테스트 전용 — 프로덕션 금지 |
| `limitedParallelism` | IO 스레드 수 제한 — Rate Limit 대응 |

- **"어디서 실행하냐"** 가 Dispatcher의 핵심입니다.
- Repository가 `withContext`로 Dispatcher를 책임지면 ViewModel은 Dispatcher를 몰라도 됩니다.
- Dispatcher를 Hilt로 주입하면 테스트 시 교체가 쉬워집니다.

---

## 참고

- [Kotlin Coroutines 공식 문서 — Dispatchers](https://kotlinlang.org/docs/coroutine-context-and-dispatchers.html)
- [Android Coroutines 모범 사례](https://developer.android.com/kotlin/coroutines/coroutines-best-practices)
- [Dispatchers.IO vs Default — KotlinConf](https://youtu.be/w0kfnydnFWI)
