---
title: (Android/Hilt) Hilt vs Koin vs Dagger2 비교
tags: [ Android, Hilt ]
style: fill
color: dark
description: Android DI 라이브러리인 Hilt, Koin, Dagger2의 동작 방식, 장단점, 성능과 학습 곡선을 비교하고 선택 기준을 정리합니다.
---

---

## 1. 세 라이브러리 개요

| | Dagger2 | Hilt | Koin |
|---|---------|------|------|
| 제작 | Google (Square 기반) | Google | Kotlin 커뮤니티 |
| 방식 | 어노테이션 + **코드 생성** | Dagger 위의 추상화 | DSL + **런타임 해석** |
| 검증 시점 | 컴파일 타임 | 컴파일 타임 | **런타임** |
| 언어 | Java/Kotlin | Java/Kotlin (Android 전용) | Kotlin 전용 |
| 멀티플랫폼(KMP) | 불가 | 불가 | **가능** |

---

## 2. Dagger2 — 강력하지만 복잡

Component, Subcomponent, Module을 모두 직접 정의해야 합니다.

```kotlin
@Component(modules = [AppModule::class])
interface AppComponent {
    fun inject(activity: MainActivity)

    @Component.Factory
    interface Factory {
        fun create(@BindsInstance context: Context): AppComponent
    }
}

// Application에서 직접 Component 생성/보관
class MyApp : Application() {
    val appComponent: AppComponent by lazy {
        DaggerAppComponent.factory().create(this)
    }
}

// Activity에서 직접 주입 호출
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        (application as MyApp).appComponent.inject(this)
        super.onCreate(savedInstanceState)
    }
}
```

- 장점: 가장 유연, 컴파일 타임 검증, 런타임 오버헤드 거의 없음
- 단점: **보일러플레이트가 많고 학습 곡선이 가파름** (Component/Subcomponent 설계 필요)

---

## 3. Hilt — Dagger를 Android에 맞게 단순화

Android 생명주기별 Component를 **미리 정의**해두어 설정이 대폭 줄었습니다.

```kotlin
@HiltAndroidApp class MyApp : Application()

@AndroidEntryPoint
class MainActivity : AppCompatActivity() {
    @Inject lateinit var analytics: AnalyticsTracker
}

@HiltViewModel
class MainViewModel @Inject constructor(private val repo: UserRepository) : ViewModel()
```

- 장점: 보일러플레이트 최소, **컴파일 타임 검증**, Jetpack(ViewModel, Navigation, WorkManager) 공식 통합, 테스트 지원
- 단점: Android 전용, 빌드 시간 증가(KSP로 완화), 에러 메시지가 난해할 때가 있음, 컴포넌트 구조 커스터마이징 제한

---

## 4. Koin — 코드 생성 없는 Kotlin DSL

어노테이션 프로세서 없이 **Kotlin 코드로 의존성을 선언**합니다.

```kotlin
val appModule = module {
    single { Retrofit.Builder().baseUrl("...").build() }
    single<UserApi> { get<Retrofit>().create(UserApi::class.java) }
    single<UserRepository> { UserRepositoryImpl(get()) }
    viewModel { MainViewModel(get()) }
}

class MyApp : Application() {
    override fun onCreate() {
        super.onCreate()
        startKoin {
            androidContext(this@MyApp)
            modules(appModule)
        }
    }
}

class MainActivity : AppCompatActivity() {
    private val viewModel: MainViewModel by viewModel()  // 주입
}
```

- 장점: 설정이 매우 간단, **빌드 속도 빠름**(코드 생성 없음), 학습 곡선 낮음, **KMP 지원**
- 단점: 의존성 누락이 **런타임 크래시**로 드러남, 대규모 그래프에서 성능 저하 가능

> **Koin의 런타임 검증 대책**: `koinApplication { checkModules() }` 테스트로 그래프 완전성을 검증해 위험을 줄일 수 있습니다.

---

## 5. 핵심 차이: 컴파일 타임 vs 런타임

```
Hilt/Dagger:  의존성 누락 → 빌드 실패 (배포 전에 발견)
Koin:         의존성 누락 → 앱 실행 중 크래시 (해당 화면 진입 시 발견)
```

대신 Koin은 빌드가 빠르고 코드가 단순합니다. 둘은 **안전성 vs 단순성**의 트레이드오프입니다.

---

## 6. 비교 요약

| 항목 | Dagger2 | Hilt | Koin |
|------|---------|------|------|
| 학습 곡선 | 높음 | 중간 | **낮음** |
| 보일러플레이트 | 많음 | 적음 | **매우 적음** |
| 빌드 속도 | 느림 | 느림 | **빠름** |
| 안전성(컴파일 검증) | **O** | **O** | X (런타임) |
| 런타임 성능 | **최상** | **최상** | 양호 |
| Jetpack 통합 | 수동 | **공식 지원** | 확장 라이브러리 |
| KMP 지원 | X | X | **O** |

---

## 7. 선택 기준

- **Hilt**: 일반적인 Android 앱의 **기본 선택**. 공식 지원, 컴파일 타임 안전성, 팀 합류 시 학습 자료 풍부
- **Koin**: 소규모 프로젝트/빠른 프로토타입, **KMP(Kotlin Multiplatform)** 프로젝트, 빌드 시간이 큰 부담인 경우
- **Dagger2**: 대규모 레거시 프로젝트 유지보수, Hilt의 제약을 벗어난 세밀한 컴포넌트 설계가 필요한 경우 (신규 프로젝트에는 비권장)

---

## 8. 정리

- 세 라이브러리 모두 "의존성을 외부에서 주입"이라는 목적은 같고 **생성 방식과 검증 시점**이 다름
- Dagger2: 강력하나 복잡 → Hilt가 Android용으로 단순화
- Koin: 코드 생성 없는 DSL, 간단하고 빠르지만 런타임 검증
- 신규 Android 프로젝트는 **Hilt**, KMP나 경량 프로젝트는 **Koin**을 우선 고려
- 어떤 도구든 핵심은 **생성자 주입 + 인터페이스 의존**이라는 설계 원칙
