---
title: (Kotlin/코틀린) 싱글턴 패턴(Singleton Pattern) 완전 정리
tags: [ Kotlin, Android ]
style: fill
color: dark
description: Kotlin의 object, companion object, by lazy를 활용한 싱글턴 패턴 구현, 멀티스레드 안전성, Android Hilt 적용까지 깊이 있게 정리합니다.
---

## 개요

- 생성 패턴(Creational Pattern) 중 **싱글턴 패턴(Singleton Pattern)** 을 다룹니다.
- 싱글턴 패턴은 **클래스의 인스턴스가 오직 하나만 존재하도록 보장** 하고, 전역 접근점을 제공하는 패턴입니다.
- 이 글에서는 다음을 설명합니다.
  - 싱글턴 패턴이 필요한 이유
  - Kotlin `object` / `companion object` / `by lazy` 활용
  - 멀티스레드 안전성
  - Android 실전 예제 (Room, SharedPreferences, Hilt)
  - 싱글턴의 단점과 주의사항

---

## 1. 왜 싱글턴 패턴이 필요한가

```kotlin
// ❌ 매번 새 인스턴스 생성 — 리소스 낭비, 상태 불일치
fun processOrder(order: Order) {
    val db = DatabaseManager()  // 매번 새 연결 생성
    db.save(order)
}

fun getUser(id: Long) {
    val db = DatabaseManager()  // 또 다른 새 연결 — 이전과 다른 인스턴스
    db.find(id)
}
```

**싱글턴이 적합한 경우:**

```
✔ DB 연결 풀 — 연결을 매번 새로 만들면 비용이 큼
✔ 설정(Configuration) 객체 — 앱 전체에서 동일한 설정을 공유
✔ 로거(Logger) — 로그를 한 곳에서 처리
✔ 캐시(Cache) — 동일한 캐시 저장소를 전체에서 공유
✔ 이벤트 버스 — 컴포넌트 간 이벤트를 단일 채널로 전달
```

---

## 2. Kotlin object — 가장 기본적인 싱글턴

Kotlin에서는 `object` 키워드 하나로 싱글턴을 완벽하게 구현합니다.

```kotlin
object AppLogger {
    private val logs = mutableListOf<String>()

    fun log(message: String) {
        val entry = "[${System.currentTimeMillis()}] $message"
        logs.add(entry)
        println(entry)
    }

    fun getLogs(): List<String> = logs.toList()

    fun clear() = logs.clear()
}

// 사용 — 인스턴스 생성 없이 바로 접근
AppLogger.log("앱 시작")
AppLogger.log("사용자 로그인")
println(AppLogger.getLogs())
```

**Kotlin `object`의 특징:**

```
✔ 스레드 안전 — JVM 클래스 로딩 메커니즘으로 보장
✔ Lazy 초기화 — 처음 접근 시 단 한 번만 생성
✔ 인터페이스 구현 가능
✔ 별도 코드 없이 싱글턴 보장
```

```kotlin
// 인터페이스 구현도 가능
interface EventTracker {
    fun track(event: String, params: Map<String, Any> = emptyMap())
}

object FirebaseTracker : EventTracker {
    override fun track(event: String, params: Map<String, Any>) {
        println("Firebase 이벤트: $event, params: $params")
        // FirebaseAnalytics.logEvent(event, params.toBundle())
    }
}

// 사용
FirebaseTracker.track("button_click", mapOf("screen" to "home", "button" to "login"))
```

---

## 3. companion object — 클래스와 결합된 싱글턴

`companion object`는 클래스 내부에 싱글턴을 두는 방식으로, 팩토리 메서드와 상수 정의에 자주 활용됩니다.

```kotlin
class UserRepository private constructor(
    private val api: UserApi,
    private val dao: UserDao
) {
    suspend fun getUser(id: Long) = api.fetchUser(id)
    suspend fun saveUser(user: User) = dao.insert(user)

    companion object {
        @Volatile
        private var instance: UserRepository? = null

        fun getInstance(api: UserApi, dao: UserDao): UserRepository =
            instance ?: synchronized(this) {
                instance ?: UserRepository(api, dao).also { instance = it }
            }
    }
}

// 사용
val repo = UserRepository.getInstance(api, dao)
```

```kotlin
// 상수와 팩토리 메서드를 함께 정의하는 패턴
class ApiConfig private constructor(
    val baseUrl: String,
    val timeout: Long,
    val headers: Map<String, String>
) {
    companion object {
        const val DEFAULT_TIMEOUT = 10_000L
        const val MAX_RETRY       = 3

        fun production() = ApiConfig(
            baseUrl  = "https://api.example.com",
            timeout  = DEFAULT_TIMEOUT,
            headers  = mapOf("Accept" to "application/json")
        )

        fun staging() = ApiConfig(
            baseUrl  = "https://staging.api.example.com",
            timeout  = 30_000L,
            headers  = mapOf("Accept" to "application/json", "X-Debug" to "true")
        )
    }
}

val config = if (BuildConfig.DEBUG) ApiConfig.staging() else ApiConfig.production()
```

---

## 4. by lazy — 지연 초기화 싱글턴

초기화 비용이 크고, 사용하지 않을 수도 있다면 `by lazy`를 활용합니다.

```kotlin
class HeavyAnalyticsEngine private constructor() {
    init {
        // 초기화 시 무거운 작업 수행
        println("AnalyticsEngine 초기화 — SDK 로딩 중...")
    }

    fun trackEvent(name: String) = println("이벤트 추적: $name")

    companion object {
        // 처음 접근 시 한 번만 초기화, 기본값은 스레드 안전(SYNCHRONIZED)
        val instance: HeavyAnalyticsEngine by lazy { HeavyAnalyticsEngine() }
    }
}

// 이 시점에 초기화 (최초 호출 시)
HeavyAnalyticsEngine.instance.trackEvent("app_start")
// 이미 초기화됨 — 재생성 없이 반환
HeavyAnalyticsEngine.instance.trackEvent("screen_view")
```

### lazy 모드 비교

```kotlin
// SYNCHRONIZED (기본값) — 멀티스레드 안전, 성능 약간 낮음
val instance1 by lazy(LazyThreadSafetyMode.SYNCHRONIZED) { HeavyClass() }

// NONE — 단일 스레드 전용, 가장 빠름
val instance2 by lazy(LazyThreadSafetyMode.NONE) { HeavyClass() }

// PUBLICATION — 여러 스레드가 동시에 초기화 시도 가능, 먼저 완료된 값 사용
val instance3 by lazy(LazyThreadSafetyMode.PUBLICATION) { HeavyClass() }
```

| 모드 | 스레드 안전 | 성능 | 사용 시점 |
|------|------------|------|-----------|
| SYNCHRONIZED | ✅ | 보통 | 기본값, 멀티스레드 환경 |
| NONE | ❌ | 빠름 | 단일 스레드 확실할 때 |
| PUBLICATION | ✅ (약함) | 보통 | 초기화 비용이 낮을 때 |

---

## 5. object vs companion object vs by lazy 비교

```kotlin
// ① object — 독립적인 싱글턴
object EventBus {
    private val listeners = mutableMapOf<String, MutableList<(Any) -> Unit>>()

    fun subscribe(event: String, listener: (Any) -> Unit) {
        listeners.getOrPut(event) { mutableListOf() }.add(listener)
    }

    fun publish(event: String, data: Any) {
        listeners[event]?.forEach { it(data) }
    }
}

// ② companion object — 클래스에 종속된 팩토리/상수
class Session private constructor(val token: String, val userId: Long) {
    companion object {
        fun create(token: String, userId: Long) = Session(token, userId)
        val EMPTY = Session("", -1L)
    }
}

// ③ by lazy — 클래스 프로퍼티의 지연 초기화
class AppModule {
    val database: AppDatabase by lazy { AppDatabase.build() }
    val repository: UserRepository by lazy { UserRepository(database.userDao()) }
}
```

| 항목 | `object` | `companion object` | `by lazy` |
|------|----------|--------------------|-----------|
| 선언 위치 | 최상위 / 중첩 | 클래스 내부 | 프로퍼티 |
| 클래스 연관 | 독립적 | 클래스에 종속 | 인스턴스에 종속 |
| 초기화 시점 | 첫 접근 시 | 첫 접근 시 | 첫 접근 시 |
| 스레드 안전 | ✅ JVM 보장 | `@Volatile` 필요 | ✅ (SYNCHRONIZED) |
| 주 용도 | 유틸, 상수, 이벤트 | 팩토리, 상수 | 지연·비용 큰 초기화 |

---

## 6. Android 실전 예제 ① — Room Database

```kotlin
@Database(entities = [User::class, Product::class], version = 1)
abstract class AppDatabase : RoomDatabase() {
    abstract fun userDao(): UserDao
    abstract fun productDao(): ProductDao

    companion object {
        @Volatile
        private var instance: AppDatabase? = null

        fun getInstance(context: Context): AppDatabase =
            instance ?: synchronized(this) {
                instance ?: Room.databaseBuilder(
                    context.applicationContext,  // Application Context — 누수 방지
                    AppDatabase::class.java,
                    "app_database"
                )
                .fallbackToDestructiveMigration()
                .build()
                .also { instance = it }
            }
    }
}

// 사용
val db    = AppDatabase.getInstance(context)
val users = db.userDao().getAll()
```

---

## 7. Android 실전 예제 ② — SharedPreferences 래퍼

```kotlin
object AppPreferences {
    private lateinit var prefs: SharedPreferences

    fun init(context: Context) {
        prefs = context.applicationContext
            .getSharedPreferences("app_prefs", Context.MODE_PRIVATE)
    }

    var authToken: String?
        get() = prefs.getString("auth_token", null)
        set(value) = prefs.edit().putString("auth_token", value).apply()

    var userId: Long
        get() = prefs.getLong("user_id", -1L)
        set(value) = prefs.edit().putLong("user_id", value).apply()

    var isDarkMode: Boolean
        get() = prefs.getBoolean("dark_mode", false)
        set(value) = prefs.edit().putBoolean("dark_mode", value).apply()

    fun clear() = prefs.edit().clear().apply()
}

// Application에서 초기화
class App : Application() {
    override fun onCreate() {
        super.onCreate()
        AppPreferences.init(this)
    }
}

// 어디서든 사용
AppPreferences.authToken = "Bearer eyJhbGci..."
val token = AppPreferences.authToken
```

---

## 8. Android 실전 예제 ③ — Hilt로 싱글턴 관리 (권장)

직접 싱글턴을 구현하는 대신 **Hilt**로 수명주기를 위임하는 것이 Android 권장 방식입니다.

```kotlin
// Hilt Module — @Singleton으로 앱 수명주기와 동일하게 관리
@Module
@InstallIn(SingletonComponent::class)
object AppModule {

    @Provides
    @Singleton
    fun provideAppDatabase(@ApplicationContext context: Context): AppDatabase =
        Room.databaseBuilder(context, AppDatabase::class.java, "app_db")
            .build()

    @Provides
    @Singleton
    fun provideUserRepository(
        api: UserApi,
        dao: UserDao
    ): UserRepository = UserRepository(api, dao)
}
```

```kotlin
// ViewModel — @Inject로 싱글턴 인스턴스 자동 주입
@HiltViewModel
class UserViewModel @Inject constructor(
    private val userRepository: UserRepository  // Hilt가 싱글턴 관리
) : ViewModel() {

    fun loadUser(id: Long) {
        viewModelScope.launch {
            userRepository.getUser(id)
        }
    }
}
```

```kotlin
// 테스트 — FakeModule로 싱글턴 교체
@Module
@TestInstallIn(
    components = [SingletonComponent::class],
    replaces   = [AppModule::class]
)
object FakeAppModule {

    @Provides
    @Singleton
    fun provideUserRepository(): UserRepository = FakeUserRepository()
}
```

---

## 9. 싱글턴의 단점과 주의사항

### ❌ 전역 상태로 인한 테스트 어려움

```kotlin
// ❌ object 직접 사용 — 테스트 격리 불가
class OrderService {
    fun placeOrder(order: Order) {
        DatabaseManager.save(order)  // 테스트에서도 실제 DB 접근
    }
}

// ✅ 인터페이스 주입 — 테스트 시 Fake로 교체 가능
class OrderService(private val repository: OrderRepository) {
    fun placeOrder(order: Order) {
        repository.save(order)
    }
}
```

---

### ❌ Android Context 누수

```kotlin
// ❌ Activity Context를 object에 저장 — 메모리 누수
object BadSingleton {
    lateinit var context: Context  // Activity가 종료돼도 GC 불가
}

// ✅ Application Context만 허용
object GoodSingleton {
    lateinit var appContext: Context

    fun init(context: Context) {
        appContext = context.applicationContext  // Application Context만 저장
    }
}
```

---

### ❌ lateinit init() 누락

```kotlin
object AppPreferences {
    private lateinit var prefs: SharedPreferences

    fun getValue(): String {
        return prefs.getString("key", "") ?: ""
        // init() 호출 전 접근 시 UninitializedPropertyAccessException 💥
    }
}

// ✅ 초기화 여부를 확인하거나, Application에서 반드시 init() 호출
class App : Application() {
    override fun onCreate() {
        super.onCreate()
        AppPreferences.init(this)  // 앱 시작 시 반드시 초기화
    }
}
```

---

## 10. 싱글턴 직접 구현 vs Hilt

| 항목 | object 직접 구현 | Hilt `@Singleton` |
|------|-----------------|-------------------|
| 전역 접근 | 직접 참조 | `@Inject`로 주입 |
| 테스트 | 어려움 (전역 상태) | 쉬움 (모듈 교체) |
| 수명주기 | 수동 관리 | 자동 (`@Singleton` 등) |
| 의존성 교체 | 코드 수정 필요 | 모듈 교체로 가능 |
| 적합한 경우 | 간단한 유틸, 상수 | 비즈니스 로직, Repository |

---

## 11. 정리

| 항목 | 내용 |
|------|------|
| `object` | 독립 싱글턴, 스레드 안전, JVM 보장 |
| `companion object` | 클래스 연관 싱글턴, 팩토리·상수 |
| `by lazy` | 지연 초기화, 비용 큰 객체에 적합 |
| Android 활용 | Room, SharedPreferences, Hilt `@Singleton` |
| 주의사항 | Application Context 사용, 테스트 격리, init() 누락 방지 |

- 싱글턴은 **"딱 하나만 있어야 하는 자원"** 에만 제한적으로 사용하세요.
- Android에서는 Hilt를 통해 수명주기를 관리하는 것이 가장 안전하고 테스트하기 좋습니다.
- 유틸성 객체나 앱 전역 상수처럼 단순한 경우에는 `object`로 충분합니다.

---

## 참고

- [Kotlin object 공식 문서](https://kotlinlang.org/docs/object-declarations.html)
- [Android Hilt 공식 문서](https://developer.android.com/training/dependency-injection/hilt-android)
- [빌더 패턴 포스팅 보기](/posts/2026-05-01-builder)
- [팩토리 패턴 포스팅 보기](/posts/2026-05-02-factory)
