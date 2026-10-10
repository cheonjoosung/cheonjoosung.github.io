---
title: (Android/Hilt) Hilt + Retrofit 연동
tags: [ Android, Hilt, Retrofit ]
style: fill
color: dark
description: Hilt로 OkHttpClient, Retrofit, API 인터페이스를 주입하는 방법과 Interceptor, 인증 토큰, BuildConfig 활용 패턴을 정리합니다.
---

---

## 1. 의존성 체인

Retrofit은 여러 객체가 단계적으로 조립됩니다. Hilt가 이 체인을 자동으로 연결합니다.

```
Interceptor(들) → OkHttpClient → Retrofit → UserApi → Repository → ViewModel
```

---

## 2. NetworkModule 구성

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {

    @Provides
    @Singleton
    fun provideLoggingInterceptor(): HttpLoggingInterceptor =
        HttpLoggingInterceptor().apply {
            // 디버그 빌드에서만 본문 로그 출력 (릴리즈에서는 민감 정보 노출 방지)
            level = if (BuildConfig.DEBUG) HttpLoggingInterceptor.Level.BODY
                    else HttpLoggingInterceptor.Level.NONE
        }

    @Provides
    @Singleton
    fun provideOkHttpClient(
        loggingInterceptor: HttpLoggingInterceptor,  // 위에서 제공한 것을 Hilt가 주입
        authInterceptor: AuthInterceptor             // @Inject constructor로 제공
    ): OkHttpClient =
        OkHttpClient.Builder()
            .addInterceptor(authInterceptor)
            .addInterceptor(loggingInterceptor)
            .connectTimeout(30, TimeUnit.SECONDS)
            .readTimeout(30, TimeUnit.SECONDS)
            .build()

    @Provides
    @Singleton
    fun provideRetrofit(okHttpClient: OkHttpClient): Retrofit =
        Retrofit.Builder()
            .baseUrl(BuildConfig.BASE_URL)  // 빌드 타입별로 다른 서버 주소 사용 가능
            .client(okHttpClient)
            .addConverterFactory(GsonConverterFactory.create())
            .build()

    // API 인터페이스마다 하나씩 제공
    @Provides
    @Singleton
    fun provideUserApi(retrofit: Retrofit): UserApi =
        retrofit.create(UserApi::class.java)

    @Provides
    @Singleton
    fun provideProductApi(retrofit: Retrofit): ProductApi =
        retrofit.create(ProductApi::class.java)
}
```

---

## 3. 인증 Interceptor — 다른 의존성을 주입받는 Interceptor

`@Inject constructor`로 만들면 Interceptor도 Hilt의 도움을 받을 수 있습니다.

```kotlin
class AuthInterceptor @Inject constructor(
    private val tokenStorage: TokenStorage  // 토큰 저장소를 주입받음
) : Interceptor {

    override fun intercept(chain: Interceptor.Chain): Response {
        val token = tokenStorage.getAccessToken()
        val request = chain.request().newBuilder().apply {
            if (token != null) addHeader("Authorization", "Bearer $token")
        }.build()
        return chain.proceed(request)
    }
}

@Singleton
class TokenStorage @Inject constructor(
    @ApplicationContext private val context: Context
) {
    fun getAccessToken(): String? = /* EncryptedSharedPreferences 등에서 조회 */ null
}
```

> Interceptor에서 매번 최신 토큰을 읽도록 **저장소를 주입**하는 것이 핵심입니다. 토큰 값 자체를 주입하면 갱신되어도 옛 값이 유지됩니다.

---

## 4. Repository에서 사용

```kotlin
class UserRepositoryImpl @Inject constructor(
    private val api: UserApi  // Retrofit 체인 전체를 Hilt가 조립해 주입
) : UserRepository {

    override suspend fun getUser(id: String): Result<User> = runCatching {
        api.getUser(id).toDomain()
    }
}
```

ViewModel까지 `@HiltViewModel`로 이어지면 **어디서도 `Retrofit.Builder()`를 직접 호출하지 않습니다.**

---

## 5. 서버가 여러 개인 경우 — @Qualifier

```kotlin
@Qualifier @Retention(AnnotationRetention.BINARY) annotation class MainServer
@Qualifier @Retention(AnnotationRetention.BINARY) annotation class AuthServer

@Provides @Singleton @MainServer
fun provideMainRetrofit(client: OkHttpClient): Retrofit =
    Retrofit.Builder().baseUrl("https://api.example.com/").client(client)
        .addConverterFactory(GsonConverterFactory.create()).build()

@Provides @Singleton @AuthServer
fun provideAuthRetrofit(client: OkHttpClient): Retrofit =
    Retrofit.Builder().baseUrl("https://auth.example.com/").client(client)
        .addConverterFactory(GsonConverterFactory.create()).build()

@Provides @Singleton
fun provideUserApi(@MainServer retrofit: Retrofit): UserApi =
    retrofit.create(UserApi::class.java)
```

---

## 6. 정리

- 체인: `Interceptor → OkHttpClient → Retrofit → Api → Repository → ViewModel`
- `OkHttpClient`, `Retrofit`, API 인터페이스는 `@Provides @Singleton`으로 한 번만 생성
- Interceptor는 `@Inject constructor`로 의존성(토큰 저장소 등)을 주입받음
- 토큰은 값이 아닌 **저장소**를 주입해 항상 최신 값 사용
- 서버가 여러 개면 `@Qualifier`로 Retrofit 구분
- 디버그 로그는 `BuildConfig.DEBUG`로 제어
