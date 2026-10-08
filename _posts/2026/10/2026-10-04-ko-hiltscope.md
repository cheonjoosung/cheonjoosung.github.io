---
title: (Android/Hilt) Hilt Scope — Singleton, ActivityScoped, ViewModelScoped
tags: [ Android, Hilt ]
style: fill
color: dark
description: Hilt의 컴포넌트 계층 구조와 @Singleton, @ActivityScoped, @ViewModelScoped 등 스코프별 생명주기와 선택 기준을 정리합니다.
---

---

## 1. 스코프란?

기본적으로 Hilt는 **주입을 요청할 때마다 새 인스턴스**를 만듭니다.
스코프를 지정하면 해당 컴포넌트의 생명주기 동안 **같은 인스턴스를 재사용**합니다.

```kotlin
// 스코프 없음: 주입받는 곳마다 새 인스턴스
class Logger @Inject constructor()

// @Singleton: 앱 전체에서 단 하나의 인스턴스 공유
@Singleton
class Logger @Inject constructor()
```

---

## 2. 컴포넌트 계층과 스코프

```
SingletonComponent                     @Singleton
  └─ ActivityRetainedComponent         @ActivityRetainedScoped
       ├─ ViewModelComponent           @ViewModelScoped
       └─ ActivityComponent            @ActivityScoped
            └─ FragmentComponent       @FragmentScoped
                 └─ ViewComponent      @ViewScoped
```

- 상위 컴포넌트의 객체는 하위에서 **주입받을 수 있습니다** (Singleton → Activity OK)
- 하위 컴포넌트의 객체는 상위에서 **주입받을 수 없습니다** (Activity → Singleton 불가, 컴파일 에러)

---

## 3. 스코프별 생명주기

| 어노테이션 | 인스턴스 생성 | 인스턴스 소멸 |
|-----------|--------------|--------------|
| `@Singleton` | 앱 프로세스 시작 후 첫 요청 | 앱 프로세스 종료 |
| `@ActivityRetainedScoped` | Activity 최초 생성 | Activity **finish** (화면 회전에는 유지) |
| `@ViewModelScoped` | ViewModel 생성 | ViewModel clear |
| `@ActivityScoped` | Activity 생성 | Activity destroy (회전 시 재생성) |
| `@FragmentScoped` | Fragment 생성 | Fragment destroy |

---

## 4. 사용 예시

```kotlin
// 앱 전체 공유: DB, 네트워크 클라이언트, Repository
@Singleton
class UserRepositoryImpl @Inject constructor(
    private val api: UserApi
) : UserRepository

// ViewModel 안에서만 공유: 같은 ViewModel이 쓰는 여러 UseCase가 상태를 공유할 때
@ViewModelScoped
class CartManager @Inject constructor()

@HiltViewModel
class CheckoutViewModel @Inject constructor(
    private val cartManager: CartManager,       // 이 ViewModel에 묶인 인스턴스
    private val orderUseCase: OrderUseCase      // 내부에서도 같은 CartManager 사용
) : ViewModel()
```

Module에서는 `@Provides` / `@Binds` 함수에 스코프를 붙입니다.

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object AppModule {
    @Provides
    @Singleton  // Retrofit은 한 번만 만들어 재사용
    fun provideRetrofit(): Retrofit = Retrofit.Builder()/*...*/.build()
}
```

> **스코프 불일치 규칙**: `@InstallIn(SingletonComponent::class)` Module 안에서는 `@ActivityScoped` 같은 하위 스코프를 쓸 수 없습니다. 스코프는 설치된 컴포넌트와 맞거나 그 이하여야 합니다.

---

## 5. 스코프를 무조건 붙이면 안 되는 이유

스코프는 **인스턴스를 메모리에 유지**하므로 비용이 있습니다.

```kotlin
// ❌ 모든 클래스에 @Singleton → 쓰지 않아도 메모리에 계속 남음, 상태 공유로 인한 버그 위험
@Singleton class StringFormatter @Inject constructor()

// ✅ 상태가 없고 가벼운 클래스는 스코프 없이 — 필요할 때 생성, GC가 정리
class StringFormatter @Inject constructor()
```

**스코프를 붙이는 기준**
- 생성 비용이 크다 (Retrofit, OkHttp, Room DB)
- 상태를 공유해야 한다 (캐시, 로그인 세션)
- 하나만 있어야 한다 (DB 인스턴스)

**붙이지 않는 기준**
- 상태가 없는 유틸/매퍼/포매터
- 생성 비용이 작다

---

## 6. @Singleton 사용 시 주의 — 스레드 안전성

Singleton은 여러 스레드가 동시에 접근하므로 **내부 상태가 변경 가능하다면 동기화**가 필요합니다.

```kotlin
@Singleton
class MemoryCache @Inject constructor() {
    // ❌ HashMap은 스레드 안전하지 않음
    private val cache = HashMap<String, Any>()

    // ✅ 동시성 안전한 컬렉션 사용
    private val safeCache = ConcurrentHashMap<String, Any>()
}
```

---

## 7. 정리

- 스코프가 없으면 **주입마다 새 인스턴스**, 스코프가 있으면 **컴포넌트 생명주기 동안 재사용**
- 계층: Singleton → ActivityRetained → ViewModel / Activity → Fragment → View
- 상위 객체는 하위에서 주입 가능, 반대는 불가
- `@Singleton`(앱 전체), `@ViewModelScoped`(ViewModel), `@ActivityRetainedScoped`(회전 유지)
- 상태 없고 가벼운 클래스는 스코프를 붙이지 않는 것이 기본
- Singleton의 가변 상태는 스레드 안전하게 관리
