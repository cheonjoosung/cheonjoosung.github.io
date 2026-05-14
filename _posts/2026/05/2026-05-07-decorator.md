---
title: (Kotlin/코틀린) 데코레이터 패턴(Decorator Pattern) 완전 정리
tags: [ Kotlin, Android ]
style: fill
color: dark
description: 데코레이터 패턴의 개념, 상속 대신 합성으로 기능을 동적으로 추가하는 방법, OkHttp Interceptor, Kotlin 위임(by) 활용까지 깊이 있게 정리합니다.
---

## 개요

- 구조 패턴(Structural Pattern) 중 **데코레이터 패턴(Decorator Pattern)** 을 다룹니다.
- 데코레이터 패턴은 **객체에 기능을 동적으로 추가** 하는 패턴입니다.
- 상속 없이 합성(Composition)으로 기능을 조합할 수 있어 OCP를 자연스럽게 실현합니다.
- 이 글에서는 다음을 설명합니다.
  - 데코레이터 패턴이 필요한 이유
  - 기본 구조와 Kotlin 구현
  - Kotlin `by` 위임을 활용한 간결한 데코레이터
  - Android 실전 예제 (OkHttp Interceptor, Logger, Cache)

---

## 1. 왜 데코레이터 패턴이 필요한가

### ❌ 상속으로 기능 조합 — 클래스 폭발

```kotlin
open class TextFileReader { open fun read(): String = "파일 내용" }

// 기능 조합마다 서브클래스가 필요
class BufferedTextFileReader : TextFileReader()        // 버퍼링
class EncryptedTextFileReader : TextFileReader()       // 암호화
class CompressedTextFileReader : TextFileReader()      // 압축
class BufferedEncryptedReader : TextFileReader()       // 버퍼링 + 암호화
class BufferedCompressedReader : TextFileReader()      // 버퍼링 + 압축
class EncryptedCompressedReader : TextFileReader()     // 암호화 + 압축
class BufferedEncryptedCompressedReader : TextFileReader() // 셋 다 — 조합이 늘수록 기하급수적 증가
```

---

### ✅ 데코레이터로 해결 — 기능을 레이어처럼 조합

```kotlin
val reader = CompressedDecorator(
    EncryptedDecorator(
        BufferedDecorator(
            TextFileReader()
        )
    )
)
// 클래스 추가 없이 원하는 기능만 조합
```

---

## 2. 기본 구조

```kotlin
// 공통 인터페이스
interface FileReader {
    fun read(): String
}

// 실제 구현체 (Component)
class TextFileReader(private val path: String) : FileReader {
    override fun read(): String {
        println("파일 읽기: $path")
        return "파일 원본 내용"
    }
}

// 기본 데코레이터 — 같은 인터페이스를 구현하고 내부에 FileReader를 보유
abstract class FileReaderDecorator(
    private val wrapped: FileReader   // 감쌀 대상
) : FileReader {
    override fun read(): String = wrapped.read()  // 기본은 그대로 위임
}

// 버퍼링 데코레이터
class BufferedDecorator(wrapped: FileReader) : FileReaderDecorator(wrapped) {
    private var cache: String? = null
    override fun read(): String {
        if (cache == null) {
            println("[Buffered] 캐시 없음 — 읽기 수행")
            cache = super.read()
        } else {
            println("[Buffered] 캐시 반환")
        }
        return cache!!
    }
}

// 암호화 데코레이터
class EncryptedDecorator(wrapped: FileReader) : FileReaderDecorator(wrapped) {
    override fun read(): String {
        val content = super.read()
        println("[Encrypted] 복호화 처리")
        return decrypt(content)
    }
    private fun decrypt(data: String) = "[$data 복호화됨]"
}

// 로깅 데코레이터
class LoggingDecorator(wrapped: FileReader) : FileReaderDecorator(wrapped) {
    override fun read(): String {
        println("[Log] read() 호출")
        val result = super.read()
        println("[Log] read() 완료 — 길이: ${result.length}")
        return result
    }
}
```

```kotlin
// 사용 — 런타임에 기능 조합
val reader = LoggingDecorator(
    BufferedDecorator(
        EncryptedDecorator(
            TextFileReader("/data/secret.txt")
        )
    )
)

reader.read()
// [Log] read() 호출
// [Buffered] 캐시 없음 — 읽기 수행
// [Encrypted] 복호화 처리
// 파일 읽기: /data/secret.txt
// [Log] read() 완료 — 길이: 15

reader.read()
// [Log] read() 호출
// [Buffered] 캐시 반환     ← 두 번째는 캐시에서 반환
// [Log] read() 완료 — 길이: 15
```

---

## 3. Kotlin by 위임 — 간결한 데코레이터

Kotlin의 `by` 키워드로 보일러플레이트를 제거할 수 있습니다.

```kotlin
interface Logger {
    fun log(message: String)
    fun warn(message: String)
    fun error(message: String)
}

class ConsoleLogger : Logger {
    override fun log(message: String)   = println("[INFO] $message")
    override fun warn(message: String)  = println("[WARN] $message")
    override fun error(message: String) = println("[ERROR] $message")
}

// by로 위임 — 오버라이드하지 않은 메서드는 자동으로 wrapped에 위임
class TimestampLogger(private val wrapped: Logger) : Logger by wrapped {
    override fun log(message: String) {
        wrapped.log("[${System.currentTimeMillis()}] $message")  // log만 오버라이드
    }
    // warn, error는 by로 자동 위임
}

class TagLogger(private val tag: String, private val wrapped: Logger) : Logger by wrapped {
    override fun log(message: String)   = wrapped.log("[$tag] $message")
    override fun warn(message: String)  = wrapped.warn("[$tag] $message")
    override fun error(message: String) = wrapped.error("[$tag] $message")
}
```

```kotlin
val logger = TagLogger(
    tag     = "Network",
    wrapped = TimestampLogger(ConsoleLogger())
)

logger.log("요청 시작")    // [INFO] [Network] [1716000000000] 요청 시작
logger.warn("재시도 중")   // [WARN] [Network] 재시도 중
logger.error("연결 실패") // [ERROR] [Network] 연결 실패
```

---

## 4. 함수형 데코레이터 — 고차 함수 활용

Kotlin에서는 고차 함수로 데코레이터를 더 간결하게 표현할 수 있습니다.

```kotlin
// 함수를 감싸는 데코레이터
fun <T> withLogging(tag: String, block: () -> T): T {
    println("[$tag] 시작")
    return block().also { println("[$tag] 완료") }
}

fun <T> withRetry(times: Int, block: () -> T): T {
    repeat(times - 1) {
        try { return block() } catch (e: Exception) { println("재시도 ${it + 1}회") }
    }
    return block()
}

fun <T> withTiming(tag: String, block: () -> T): T {
    val start = System.currentTimeMillis()
    return block().also {
        println("[$tag] 소요시간: ${System.currentTimeMillis() - start}ms")
    }
}

// 조합해서 사용
suspend fun fetchProducts(): List<Product> =
    withLogging("Products") {
        withRetry(3) {
            withTiming("Products") {
                api.getProducts()
            }
        }
    }
```

---

## 5. Repository 데코레이터 — 캐시 레이어

```kotlin
interface ProductRepository {
    suspend fun getProducts(): List<Product>
    suspend fun getProduct(id: Long): Product?
}

// 실제 구현
class RemoteProductRepository(private val api: ProductApi) : ProductRepository {
    override suspend fun getProducts() = api.fetchProducts()
    override suspend fun getProduct(id: Long) = api.fetchProduct(id)
}

// 캐시 데코레이터 — 기존 Repository를 감싸서 캐시 기능 추가
class CachedProductRepository(
    private val remote: ProductRepository,
    private val cacheExpireMs: Long = 5 * 60 * 1000L  // 5분
) : ProductRepository {
    private var cachedProducts: List<Product>? = null
    private var cacheTimestamp: Long = 0L

    override suspend fun getProducts(): List<Product> {
        val now = System.currentTimeMillis()
        if (cachedProducts != null && now - cacheTimestamp < cacheExpireMs) {
            println("캐시 반환")
            return cachedProducts!!
        }
        println("원격 조회 후 캐시 저장")
        return remote.getProducts().also {
            cachedProducts  = it
            cacheTimestamp  = now
        }
    }

    override suspend fun getProduct(id: Long): Product? =
        cachedProducts?.find { it.id == id } ?: remote.getProduct(id)
}

// 로깅 데코레이터
class LoggingProductRepository(
    private val wrapped: ProductRepository
) : ProductRepository {
    override suspend fun getProducts(): List<Product> {
        println("[Repo] getProducts() 호출")
        return wrapped.getProducts().also { println("[Repo] ${it.size}개 반환") }
    }

    override suspend fun getProduct(id: Long): Product? {
        println("[Repo] getProduct($id) 호출")
        return wrapped.getProduct(id)
    }
}
```

```kotlin
// 조합 — Remote → Cache → Logging 순서로 감쌈
val repository: ProductRepository = LoggingProductRepository(
    CachedProductRepository(
        RemoteProductRepository(api)
    )
)

// Hilt로 주입할 때
@Module
@InstallIn(SingletonComponent::class)
object RepositoryModule {

    @Provides
    @Singleton
    fun provideProductRepository(api: ProductApi): ProductRepository =
        LoggingProductRepository(
            CachedProductRepository(
                RemoteProductRepository(api)
            )
        )
}
```

---

## 6. Android 실전 예제 — OkHttp Interceptor

OkHttp의 `Interceptor`가 데코레이터 패턴의 대표적인 Android 사례입니다.

```kotlin
// 인증 토큰 추가 인터셉터
class AuthInterceptor(private val tokenProvider: () -> String?) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val original = chain.request()
        val token = tokenProvider() ?: return chain.proceed(original)

        val authenticated = original.newBuilder()
            .header("Authorization", "Bearer $token")
            .build()
        return chain.proceed(authenticated)
    }
}

// 로깅 인터셉터
class LoggingInterceptor : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val request = chain.request()
        println("→ ${request.method} ${request.url}")
        val response = chain.proceed(request)
        println("← ${response.code} ${request.url}")
        return response
    }
}

// 재시도 인터셉터
class RetryInterceptor(private val maxRetries: Int = 3) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        var attempt = 0
        var response: Response
        do {
            response = chain.proceed(chain.request())
            attempt++
        } while (!response.isSuccessful && attempt < maxRetries)
        return response
    }
}

// OkHttpClient에 인터셉터 조합 — 데코레이터 체인
val client = OkHttpClient.Builder()
    .addInterceptor(AuthInterceptor { tokenManager.getToken() })
    .addInterceptor(LoggingInterceptor())
    .addInterceptor(RetryInterceptor(maxRetries = 3))
    .build()
```

---

## 7. 정리

| 항목 | 내용 |
|------|------|
| 목적 | 객체에 기능을 동적으로 추가 (상속 없이) |
| 핵심 구조 | 같은 인터페이스를 구현 + 내부에 원본 객체를 보유 |
| Kotlin 도구 | `by` 위임으로 보일러플레이트 제거 |
| 함수형 변형 | 고차 함수로 간결하게 표현 가능 |
| Android 사례 | OkHttp Interceptor, 캐시 Repository, Logger 체인 |
| 어댑터와 차이 | 어댑터는 인터페이스 변환, 데코레이터는 인터페이스 유지하며 기능 추가 |

- 데코레이터 패턴은 **"런타임에 레이어를 쌓듯 기능을 조합"** 하는 것이 핵심입니다.
- 상속 계층을 늘리지 않고 기능을 확장할 수 있어 OCP를 자연스럽게 지킵니다.

---

## 참고

- Design Patterns — GoF (Gang of Four)
- [OkHttp Interceptor 공식 문서](https://square.github.io/okhttp/features/interceptors/)
- [어댑터 패턴 포스팅 보기](/posts/2026-05-06-adapter)
