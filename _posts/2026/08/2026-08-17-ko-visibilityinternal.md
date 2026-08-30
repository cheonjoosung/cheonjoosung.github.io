---
title: (Kotlin/코틀린) 접근 제어자 — internal 모듈 범위 활용
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin의 접근 제어자 public, internal, protected, private 차이와 internal의 모듈 단위 캡슐화를 실전 멀티모듈 패턴과 함께 정리합니다.
---

---

## 1. 접근 제어자 한눈에 비교

| 제어자 | 적용 범위 |
|--------|----------|
| `public` | 어디서나 접근 가능 (기본값) |
| `internal` | **같은 모듈** 안에서만 접근 가능 |
| `protected` | 같은 클래스 + 하위 클래스 |
| `private` | 같은 클래스(또는 파일) 안에서만 |

```kotlin
class Example {
    public val publicProp = "공개"
    internal val internalProp = "모듈 내부"
    protected val protectedProp = "하위 클래스"
    private val privateProp = "클래스 내부"
}
```

---

## 2. Kotlin vs Java 접근 제어자 차이

```kotlin
// Kotlin에는 package-private 없음 (Java의 default)
// Kotlin의 internal이 더 강력한 대안

// Java: package-private (같은 패키지면 접근 가능)
// → 외부 라이브러리에서 같은 패키지명으로 접근 가능 (취약)

// Kotlin: internal (컴파일 모듈 단위로 제한)
// → 다른 모듈에서는 접근 불가 (더 안전한 캡슐화)

internal class InternalService {  // 이 모듈 밖에서 사용 불가
    fun process() = "내부 처리"
}
```

---

## 3. 모듈이란?

Kotlin에서 모듈은 **함께 컴파일되는 파일들의 집합**입니다.

```
단일 Gradle 프로젝트:
├── app/         ← 모듈 1
├── core/        ← 모듈 2
└── data/        ← 모듈 3

app 모듈에서 core 모듈의 internal 멤버 접근 불가
core 모듈 안에서는 internal 멤버 자유롭게 사용
```

---

## 4. internal 실전 활용 — 멀티모듈

```kotlin
// :core 모듈

// 외부에 공개할 API
public interface UserRepository {
    suspend fun getUser(id: Int): User
    suspend fun saveUser(user: User)
}

// 모듈 내부 구현 — 외부에서 직접 접근 불가
internal class UserRepositoryImpl(
    private val dao: UserDao,
    private val api: UserApi
) : UserRepository {
    override suspend fun getUser(id: Int): User { /* ... */ }
    override suspend fun saveUser(user: User) { /* ... */ }
}

// Hilt 등 DI에서 internal 구현체 바인딩
@Module
@InstallIn(SingletonComponent::class)
internal object CoreModule {
    @Provides
    fun provideUserRepository(dao: UserDao, api: UserApi): UserRepository =
        UserRepositoryImpl(dao, api)
}
```

```kotlin
// :app 모듈
// UserRepository (public 인터페이스) → 사용 가능
// UserRepositoryImpl (internal) → 접근 불가 (캡슐화 성공)
class UserViewModel(private val repo: UserRepository) : ViewModel()
```

---

## 5. 파일 수준 private

```kotlin
// Utils.kt

private fun helperFunction() = "내부 도우미"  // 이 파일 안에서만 사용

fun publicFunction(): String {
    return helperFunction()  // 같은 파일 → OK
}
```

```kotlin
// Other.kt
// helperFunction()  // 컴파일 에러: 다른 파일에서 접근 불가
```

---

## 6. protected — 상속 계층에서 활용

```kotlin
abstract class BaseViewModel : ViewModel() {
    // 하위 클래스에서만 사용 가능
    protected fun launchSafely(block: suspend CoroutineScope.() -> Unit) {
        viewModelScope.launch {
            try { block() }
            catch (e: Exception) { handleError(e) }
        }
    }

    protected open fun handleError(e: Exception) {
        println("에러: ${e.message}")
    }
}

class UserViewModel : BaseViewModel() {
    fun loadUser(id: Int) {
        launchSafely {  // protected 함수 사용
            val user = repository.getUser(id)
            _user.value = user
        }
    }

    override fun handleError(e: Exception) {
        super.handleError(e)
        _error.value = e.message
    }
}
```

---

## 7. internal + @PublishedApi

`inline` 함수는 호출 위치에 인라인되므로 내부 구현이 노출됩니다.  
`@PublishedApi`로 internal 멤버를 inline 함수에서 사용할 수 있게 합니다.

```kotlin
@PublishedApi
internal fun internalHelper(): String = "내부 구현"

inline fun publicInlineFunction(): String {
    return internalHelper()  // @PublishedApi 없으면 컴파일 에러
}
```

---

## 8. 접근 제어자 선택 가이드

```
라이브러리/모듈 API로 공개해야 하는가?
    ↓ Yes → public

같은 모듈 안에서만 사용하는가?
    ↓ Yes → internal (멀티모듈 프로젝트의 핵심)

하위 클래스에서만 접근해야 하는가?
    ↓ Yes → protected

이 클래스(또는 파일) 안에서만 사용하는가?
    ↓ Yes → private

기본값은?
    → public (생략 시 자동)
```

---

## 9. 실전 패턴 — 라이브러리 설계

```kotlin
// SDK 설계 시 internal로 구현 세부사항 숨기기
class AnalyticsSDK private constructor(
    private val apiKey: String
) {
    // 공개 API
    fun track(event: String, properties: Map<String, Any> = emptyMap()) {
        internalTracker.send(event, properties)
    }

    fun identify(userId: String) {
        internalTracker.setUser(userId)
    }

    companion object {
        @JvmStatic
        fun create(apiKey: String): AnalyticsSDK = AnalyticsSDK(apiKey)
    }

    // 내부 구현 — SDK 사용자에게 노출 안 됨
    internal val internalTracker = EventTracker(apiKey)
    internal fun flushQueue() = internalTracker.flush()
}

internal class EventTracker(private val apiKey: String) {
    fun send(event: String, properties: Map<String, Any>) { /* ... */ }
    fun setUser(userId: String) { /* ... */ }
    fun flush() { /* ... */ }
}
```

---

## 10. 정리

- **`public`**: 어디서나 접근 가능, Kotlin 기본값
- **`internal`**: 같은 모듈 안에서만 — 멀티모듈 캡슐화의 핵심
- **`protected`**: 클래스 계층에서 하위 클래스 공유
- **`private`**: 클래스 또는 파일 내부만
- Java의 `package-private`보다 `internal`이 더 안전한 캡슐화 제공
- 멀티모듈 프로젝트에서 구현체는 `internal`, 인터페이스는 `public`으로 분리
