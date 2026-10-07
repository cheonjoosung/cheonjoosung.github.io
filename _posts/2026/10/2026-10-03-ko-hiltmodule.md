---
title: (Android/Hilt) Hilt Module — @Provides vs @Binds 차이
tags: [ Android, Hilt ]
style: fill
color: dark
description: Hilt Module에서 @Provides와 @Binds의 차이, 인터페이스 바인딩, 외부 라이브러리 객체 제공, @Qualifier 사용법을 정리합니다.
---

---

## 1. Module이 필요한 경우

`@Inject constructor`로 해결되지 않는 경우에 Module로 Hilt에게 객체 생성 방법을 알려줍니다.

```
@Inject constructor로 불가능한 경우:
  1. 인터페이스 (구현체를 어떻게 고를지 모름)
  2. 외부 라이브러리 클래스 (Retrofit, Room 등 — 생성자에 @Inject를 붙일 수 없음)
  3. 빌더 패턴으로 만들어야 하는 객체
```

Module은 `@Module` + `@InstallIn(컴포넌트)`로 선언합니다.

---

## 2. @Provides — 직접 객체를 만들어 제공

함수 본문에서 객체를 직접 생성해 반환합니다. 외부 라이브러리 객체에 적합합니다.

```kotlin
@Module
@InstallIn(SingletonComponent::class)  // 앱 전체에서 사용 가능한 컴포넌트에 설치
object NetworkModule {  // @Provides만 있으면 object 사용 (인스턴스 불필요)

    @Provides
    @Singleton
    fun provideOkHttpClient(): OkHttpClient =
        OkHttpClient.Builder()
            .connectTimeout(30, TimeUnit.SECONDS)
            .build()

    // 파라미터는 Hilt가 자동으로 찾아서 주입 (위의 OkHttpClient)
    @Provides
    @Singleton
    fun provideRetrofit(client: OkHttpClient): Retrofit =
        Retrofit.Builder()
            .baseUrl("https://api.example.com/")
            .client(client)
            .addConverterFactory(GsonConverterFactory.create())
            .build()

    @Provides
    @Singleton
    fun provideUserApi(retrofit: Retrofit): UserApi =
        retrofit.create(UserApi::class.java)
}
```

---

## 3. @Binds — 인터페이스와 구현체 연결

"이 인터페이스가 요청되면 이 구현체를 쓴다"는 **매핑**만 선언합니다. 객체를 직접 만들지 않습니다.

```kotlin
interface UserRepository {
    suspend fun getUser(id: String): User
}

// 구현체는 @Inject constructor로 생성 방법을 알려줘야 함
class UserRepositoryImpl @Inject constructor(
    private val api: UserApi,
    private val dao: UserDao
) : UserRepository {
    override suspend fun getUser(id: String): User = dao.find(id) ?: api.fetch(id)
}

@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {  // @Binds는 abstract class + abstract fun

    @Binds
    @Singleton
    abstract fun bindUserRepository(
        impl: UserRepositoryImpl  // 파라미터: 구현체
    ): UserRepository             // 반환 타입: 인터페이스
}
```

이제 `UserRepository`를 요청하는 곳(ViewModel 등)에는 `UserRepositoryImpl`이 주입됩니다.
테스트에서는 이 바인딩만 가짜 구현체로 교체하면 됩니다.

---

## 4. @Provides vs @Binds 비교

| | @Binds | @Provides |
|---|--------|-----------|
| 용도 | 인터페이스 → 구현체 매핑 | 객체를 직접 생성 |
| Module 형태 | `abstract class` | `object` (또는 class) |
| 함수 | `abstract fun`, 파라미터 1개 | 본문 있는 일반 fun |
| 구현체 조건 | `@Inject constructor` 필요 | 불필요 (직접 생성) |
| 생성 코드 | 적음 (더 효율적) | 약간 더 많음 |

**선택 기준**: 내가 만든 클래스의 인터페이스 바인딩이면 `@Binds`, 외부 라이브러리 객체나 빌더 생성이면 `@Provides`.

> `@Binds`와 `@Provides`를 한 Module에 섞어야 한다면 `abstract class` 안에 `companion object`로 `@Provides`를 두면 됩니다. 아니면 Module을 분리하는 것이 깔끔합니다.

---

## 5. @Qualifier — 같은 타입이 여러 개일 때

같은 타입(`String`, `CoroutineDispatcher` 등)을 구분해야 하면 Qualifier를 만듭니다.

```kotlin
@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class IoDispatcher

@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class DefaultDispatcher

@Module
@InstallIn(SingletonComponent::class)
object DispatcherModule {
    @Provides @IoDispatcher
    fun provideIoDispatcher(): CoroutineDispatcher = Dispatchers.IO

    @Provides @DefaultDispatcher
    fun provideDefaultDispatcher(): CoroutineDispatcher = Dispatchers.Default
}

// 사용: Qualifier로 원하는 것을 지정
class UserRepositoryImpl @Inject constructor(
    private val api: UserApi,
    @IoDispatcher private val ioDispatcher: CoroutineDispatcher
)
```

Dispatcher를 주입하면 테스트에서 `TestDispatcher`로 쉽게 교체할 수 있어 코루틴 테스트에도 유리합니다.

---

## 6. @InstallIn 컴포넌트 선택

| 컴포넌트 | 생명주기 |
|---------|---------|
| `SingletonComponent` | 앱 전체 |
| `ActivityRetainedComponent` | Activity (회전해도 유지) |
| `ViewModelComponent` | ViewModel |
| `ActivityComponent` | Activity |
| `FragmentComponent` | Fragment |

> Module을 설치한 컴포넌트가 **제공되는 객체의 사용 가능 범위**를 결정합니다. 스코프는 다음 글에서 자세히 다룹니다.

---

## 7. 정리

- 인터페이스, 외부 라이브러리, 빌더 객체는 `@Inject constructor`가 불가 → **Module** 사용
- `@Provides`: 함수 본문에서 직접 생성 (Retrofit, Room 등 외부 객체)
- `@Binds`: 인터페이스 ↔ 구현체 매핑 (abstract, 코드 더 효율적)
- 같은 타입이 여러 개면 `@Qualifier`로 구분
- Dispatcher를 주입하면 테스트 교체가 쉬워짐
