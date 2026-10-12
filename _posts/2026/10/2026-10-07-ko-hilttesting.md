---
title: (Android/Hilt) Hilt Testing — @HiltAndroidTest, @TestInstallIn
tags: [ Android, Hilt ]
style: fill
color: dark
description: Hilt 환경에서 테스트하는 방법과 @HiltAndroidTest, @TestInstallIn, @UninstallModules, @BindValue를 이용한 의존성 교체 패턴을 정리합니다.
---

---

## 1. DI가 테스트에 주는 이점

의존성을 외부에서 주입받으면 **테스트에서 가짜 구현으로 교체**할 수 있습니다.
Hilt 테스트의 목표는 "실제 서버/DB 없이, 운영 코드의 Module만 테스트용으로 바꿔 끼우는 것"입니다.

```
운영:   ViewModel → UserRepository → 실제 Retrofit
테스트: ViewModel → UserRepository → FakeUserApi (가짜)
```

---

## 2. 단위 테스트 — Hilt 불필요

ViewModel/Repository를 단위 테스트할 때는 **Hilt 없이 생성자로 직접 주입**하면 됩니다. 이것이 생성자 주입의 장점입니다.

```kotlin
class UserViewModelTest {
    @Test
    fun `사용자 로드 성공`() = runTest {
        val fakeRepository = FakeUserRepository()   // 가짜를 직접 생성자에 전달
        val viewModel = UserViewModel(fakeRepository)

        viewModel.load("user-1")

        assertEquals("홍길동", viewModel.uiState.value.user?.name)
    }
}
```

> Hilt 테스트(`@HiltAndroidTest`)는 **통합/UI 테스트**처럼 실제 Hilt 그래프가 필요한 경우에만 사용합니다.

---

## 3. 설정

```kotlin
dependencies {
    androidTestImplementation("com.google.dagger:hilt-android-testing:2.51.1")
    kspAndroidTest("com.google.dagger:hilt-android-compiler:2.51.1")
    // 로컬 단위 테스트(Robolectric)에서 사용 시
    testImplementation("com.google.dagger:hilt-android-testing:2.51.1")
    kspTest("com.google.dagger:hilt-android-compiler:2.51.1")
}
```

**커스텀 TestRunner**: 테스트 시 Application을 `HiltTestApplication`으로 교체해야 합니다.

```kotlin
class HiltTestRunner : AndroidJUnitRunner() {
    override fun newApplication(cl: ClassLoader?, name: String?, context: Context?): Application =
        super.newApplication(cl, HiltTestApplication::class.java.name, context)
}
```

```kotlin
// build.gradle.kts (app)
android {
    defaultConfig {
        testInstrumentationRunner = "com.example.app.HiltTestRunner"
    }
}
```

---

## 4. @HiltAndroidTest 기본 구조

```kotlin
@HiltAndroidTest  // 이 테스트 클래스에 Hilt 그래프 생성
class ProductScreenTest {

    // HiltAndroidRule: 테스트 전 주입 준비, 반드시 order = 0 (가장 먼저 실행)
    @get:Rule(order = 0)
    val hiltRule = HiltAndroidRule(this)

    @get:Rule(order = 1)
    val composeRule = createAndroidComposeRule<MainActivity>()

    @Inject lateinit var repository: UserRepository  // 테스트 안에서 주입받을 수도 있음

    @Before
    fun setUp() {
        hiltRule.inject()  // @Inject 필드를 채움
    }

    @Test
    fun 상품_목록이_표시된다() {
        composeRule.onNodeWithText("테스트 상품").assertIsDisplayed()
    }
}
```

---

## 5. @TestInstallIn — Module 전체를 교체

테스트 전체에서 운영 Module을 **전역적으로 대체**합니다. 모든 `@HiltAndroidTest`에 적용됩니다.

```kotlin
// 운영 모듈
@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {
    @Binds abstract fun bindUserRepository(impl: UserRepositoryImpl): UserRepository
}

// 테스트 모듈: 운영 모듈을 replaces로 지정해 대체
@Module
@TestInstallIn(
    components = [SingletonComponent::class],
    replaces = [RepositoryModule::class]
)
abstract class FakeRepositoryModule {
    @Binds @Singleton
    abstract fun bindUserRepository(fake: FakeUserRepository): UserRepository
}

class FakeUserRepository @Inject constructor() : UserRepository {
    override suspend fun getUser(id: String) = User(id, "테스트 사용자")
}
```

---

## 6. @UninstallModules + @BindValue — 테스트 하나만 교체

특정 테스트 클래스에서만 Module을 제거하고 **값을 직접 지정**할 때 사용합니다.

```kotlin
@HiltAndroidTest
@UninstallModules(RepositoryModule::class)  // 이 테스트 클래스에서만 운영 모듈 제거
class LoginTest {

    @get:Rule val hiltRule = HiltAndroidRule(this)

    // @BindValue: 이 필드 값을 Hilt 그래프에 직접 바인딩
    @BindValue
    @JvmField
    val repository: UserRepository = mockk(relaxed = true)

    @Test
    fun 로그인_실패시_오류_표시() {
        coEvery { repository.login(any(), any()) } returns Result.failure(Exception())
        // ...
    }
}
```

| 방식 | 범위 | 용도 |
|------|------|------|
| `@TestInstallIn` | 모든 테스트 | 전역 Fake (인메모리 DB, Fake API 등) |
| `@UninstallModules` + `@BindValue` | 해당 테스트 클래스 | 테스트마다 다른 Mock 값 필요할 때 |

---

## 7. 정리

- 단위 테스트는 **Hilt 없이 생성자 주입**으로 충분 (가장 빠르고 간단)
- 통합/UI 테스트는 `@HiltAndroidTest` + `HiltAndroidRule(order = 0)` + `HiltTestRunner`
- `@TestInstallIn(replaces = ...)`: 운영 Module을 전역으로 교체
- `@UninstallModules` + `@BindValue`: 특정 테스트에서만 값 교체
- Dispatcher를 주입해두면 테스트에서 `TestDispatcher`로 쉽게 교체 가능
