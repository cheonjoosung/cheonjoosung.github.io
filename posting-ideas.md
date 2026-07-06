# 포스팅 아이디어 목록 (400개)

> 기존 포스팅과 중복 제거 후 작성된 목록입니다.  
> 작성 완료 시 체크박스를 체크해주세요.

---

## 🟣 Kotlin (100개)

### Coroutines / Flow (25개)

- [x] 1. Coroutines 기초 — suspend fun, launch vs async
- [x] 2. Coroutines 취소 — cancel(), isActive, CancellationException
- [x] 3. Job vs SupervisorJob — 자식 코루틴 실패 전파
- [x] 4. CoroutineExceptionHandler와 supervisorScope
- [x] 5. viewModelScope vs lifecycleScope vs GlobalScope 비교
- [x] 6. withContext vs launch vs async 차이 완전 정리
- [x] 7. Coroutines Dispatcher 심화 — IO, Main, Default, Unconfined
- [x] 8. Flow 기초 — Cold Stream 생성과 collect
- [x] 9. Flow vs LiveData — 언제 무엇을 쓸까
- [x] 10. StateFlow vs SharedFlow 완전 비교
- [x] 11. stateIn vs shareIn — Flow를 Hot으로 변환
- [x] 12. SharingStarted 전략 — WhileSubscribed, Eagerly, Lazily
- [x] 13. Flow 연산자 — zip, combine, merge
- [x] 14. flatMapLatest vs flatMapMerge vs flatMapConcat
- [x] 15. Flow debounce & throttleFirst — 검색창 실전 적용
- [x] 16. Flow retry & retryWhen — 네트워크 재시도 패턴
- [x] 17. Flow buffer & conflate — 배압(Backpressure) 처리
- [x] 18. Flow flowOn — 스레드 전환 원리
- [x] 19. Flow catch & onCompletion — 에러/완료 처리
- [x] 20. Flow — buildList, buildMap과 함께 쓰는 패턴
- [x] 21. Coroutines Mutex — 코루틴 동기화
- [x] 22. Coroutines Semaphore — 동시 실행 수 제한
- [x] 23. SharedFlow vs Channel — 일회성 이벤트 선택 기준
- [x] 24. repeatOnLifecycle vs launchWhenStarted 비교
- [x] 25. Coroutines structured concurrency 완전 정리

### 언어 기능 심화 (25개)

- [x] 26. Value class (Inline class) — 타입 안전성과 성능
- [x] 27. SAM 변환 (Functional Interface) — Java 람다 대체
- [x] 28. crossinline vs noinline 차이
- [x] 29. Kotlin 제네릭 심화 — upper bound, where 절
- [x] 30. Kotlin star projection (*) 언제 쓰는가
- [x] 31. Kotlin sealed interface
- [x] 32. Data object — object에 equals/hashCode/toString 자동 생성
- [x] 33. Enum entries — values() 대신 entries 쓰는 이유
- [x] 34. when 표현식 심화 — subject 변수, 조건 결합
- [x] 35. 꼬리 재귀 (tailrec) — 스택 오버플로우 방지
- [x] 36. Kotlin 커링 (Currying) 패턴
- [x] 37. Kotlin 함수 합성 (Function Composition)
- [x] 38. operator fun invoke — 객체를 함수처럼 호출
- [x] 39. operator fun get/set — 배열처럼 접근
- [ ] 40. operator fun iterator — for 루프 커스터마이징
- [ ] 41. operator rangeTo, contains
- [ ] 42. Kotlin 내부 클래스 — inner class vs nested class
- [ ] 43. Kotlin 인터페이스 default method와 다중 상속 해결
- [ ] 44. 확장함수 vs 멤버함수 우선순위 규칙
- [ ] 45. Nullable receiver 확장함수
- [ ] 46. Kotlin null 처리 전략 — requireNotNull, checkNotNull, error
- [ ] 47. preconditions — require, check, assert 차이
- [ ] 48. Kotlin 비트 연산자 — and, or, xor, shl, shr
- [ ] 49. Kotlin 수 체계 — Int, Long, UInt, ULong 범위와 변환
- [ ] 50. Kotlin 문자열 포매팅 — format, padStart, padEnd

### 위임 / 직렬화 / 상호운용 (15개)

- [ ] 51. Kotlin 위임 패턴 — by keyword로 interface delegation
- [ ] 52. 커스텀 프로퍼티 델리게이트 만들기
- [ ] 53. ObservableProperty / VetoableProperty
- [ ] 54. Map delegation — Map에서 프로퍼티 읽기
- [ ] 55. Kotlin 직렬화 — kotlinx.serialization 기초
- [ ] 56. kotlinx.serialization 심화 — @SerialName, Polymorphism
- [ ] 57. Gson vs Moshi vs kotlinx.serialization 완전 비교
- [ ] 58. Kotlin Annotation 만들기 — @interface, RetentionPolicy
- [ ] 59. Kotlin Reflection — KClass, KProperty, KFunction
- [ ] 60. Java 상호운용 — @JvmStatic, @JvmField, @JvmOverloads
- [ ] 61. @JvmName, @JvmSuppressWildcards 활용법
- [ ] 62. Kotlin 파일 입출력 — readText, writeText, forEachLine
- [ ] 63. Kotlin Script (.kts) 활용
- [ ] 64. Kotlin Gradle DSL 기초
- [ ] 65. Kotlin 2.0 주요 변경사항 — K2 컴파일러

### 컬렉션 / 함수형 (15개)

- [ ] 66. groupBy, partition, associate 실전 활용
- [ ] 67. fold vs reduce vs scan 차이
- [ ] 68. flatMap vs flatten 비교
- [ ] 69. windowed, chunked, zipWithNext
- [ ] 70. buildList, buildMap, buildSet
- [ ] 71. takeIf, takeUnless 실전 패턴
- [ ] 72. Comparable & Comparator 구현
- [ ] 73. 추상 클래스 vs 인터페이스 심화 비교
- [ ] 74. Kotlin 메모이제이션 패턴
- [ ] 75. Kotlin object expression (익명 객체) vs lambda 비교
- [ ] 76. 함수 타입과 함수 리터럴 완전 정리
- [ ] 77. 고차 함수 실전 패턴 — 전략 패턴 대체
- [ ] 78. Kotlin Arrow 라이브러리 기초 — Either, Option
- [ ] 79. Kotlin 접근 제어자 — internal 모듈 범위 활용
- [ ] 80. 패키지 레벨 함수 vs 싱글턴 object 선택 기준

### 테스트 / 코드 품질 (10개)

- [ ] 81. Coroutines 테스트 — TestCoroutineDispatcher, runTest
- [ ] 82. MockK 기초 — mock, every, verify
- [ ] 83. MockK 심화 — spy, slot, captureArgument
- [ ] 84. Flow 테스트 — Turbine 라이브러리
- [ ] 85. Kotlin 단위 테스트 — JUnit5 + Kotlin
- [ ] 86. TDD 실전 — Kotlin으로 레드-그린-리팩터
- [ ] 87. detekt 정적 분석 설정
- [ ] 88. ktlint 코드 스타일 설정
- [ ] 89. Kotlin 코드 품질 — Sonar 연동
- [ ] 90. Kotlin 멀티플랫폼(KMP) 개요

### 행동 패턴 (10개)

- [ ] 91. 행동 패턴 — 옵저버 패턴 (Observer)
- [ ] 92. 행동 패턴 — 전략 패턴 (Strategy)
- [ ] 93. 행동 패턴 — 커맨드 패턴 (Command)
- [ ] 94. 행동 패턴 — 템플릿 메서드 패턴 (Template Method)
- [ ] 95. 행동 패턴 — 책임 연쇄 패턴 (Chain of Responsibility)
- [ ] 96. 행동 패턴 — 상태 패턴 (State)
- [ ] 97. 행동 패턴 — 이터레이터 패턴 (Iterator)
- [ ] 98. 행동 패턴 — 중재자 패턴 (Mediator)
- [ ] 99. 행동 패턴 — 방문자 패턴 (Visitor)
- [ ] 100. 행동 패턴 — 메멘토 패턴 (Memento)

---

## 🟢 Android (100개)

### Jetpack Compose (15개)

- [ ] 1. Jetpack Compose 기초 — Composable, State, @Preview
- [ ] 2. Compose 상태 관리 — remember, mutableStateOf, State Hoisting
- [ ] 3. Compose LazyColumn / LazyGrid — RecyclerView 대체
- [ ] 4. Compose Navigation — NavGraph, NavController
- [ ] 5. Compose Animation — AnimatedVisibility, animateDpAsState
- [ ] 6. Compose Theme — MaterialTheme, colors, typography
- [ ] 7. Compose Custom Layout — Layout, SubcomposeLayout
- [ ] 8. Compose ViewModel 연동 + StateFlow
- [ ] 9. Compose Side Effect — LaunchedEffect, SideEffect, DisposableEffect
- [ ] 10. Compose UI 테스트 — ComposeTestRule
- [ ] 11. Compose vs XML View 비교 — 언제 무엇을 선택할까
- [ ] 12. Compose Modifier 완전 정리
- [ ] 13. Compose 성능 최적화 — remember, derivedStateOf, key
- [ ] 14. Compose Bottom Sheet / Dialog 구현
- [ ] 15. Compose Scaffold + TopBar + BottomBar 구성

### Hilt 의존성 주입 (8개)

- [ ] 16. Hilt 의존성 주입 기초 — @HiltAndroidApp, @Inject
- [ ] 17. Hilt + ViewModel — @HiltViewModel 완전 정리
- [ ] 18. Hilt Module — @Provides vs @Binds 차이
- [ ] 19. Hilt Scope — Singleton, ActivityScoped, ViewModelScoped
- [ ] 20. Hilt + Room 연동
- [ ] 21. Hilt + Retrofit 연동
- [ ] 22. Hilt Testing — @HiltAndroidTest, @TestInstallIn
- [ ] 23. Hilt vs Koin vs Dagger2 비교

### Navigation Component (5개)

- [ ] 24. Navigation Component 기초 — NavGraph, NavController
- [ ] 25. Navigation Component — Safe Args 타입 안전 전달
- [ ] 26. Navigation Component — Bottom Navigation 연동
- [ ] 27. Navigation Component — Deep Link 처리
- [ ] 28. Navigation Component — 백스택 관리 전략

### Jetpack Paging (3개)

- [ ] 29. Paging3 기초 — PagingSource, Pager, PagingData
- [ ] 30. Paging3 + Retrofit — 무한 스크롤 구현
- [ ] 31. Paging3 + Room — 오프라인 캐시 전략

### Room 심화 (6개)

- [ ] 32. Room Migration 전략 — 안전한 스키마 변경
- [ ] 33. Room FTS (Full Text Search) — 전문 검색
- [ ] 34. Room Relation — one-to-many, many-to-many
- [ ] 35. Room TypeConverter — 커스텀 타입 저장
- [ ] 36. Room Transaction — 원자적 DB 작업
- [ ] 37. Room + Flow — 실시간 DB 변경 감지

### Retrofit / 네트워크 (8개)

- [ ] 38. Retrofit suspend vs Call 비교
- [ ] 39. Retrofit 에러 응답 파싱 — ErrorBody 처리
- [ ] 40. Retrofit 파일 업로드 — Multipart, ProgressBody
- [ ] 41. Retrofit 동적 BaseUrl 변경 패턴
- [ ] 42. OkHttp Cache 전략 — 오프라인 캐시
- [ ] 43. OkHttp SSL Pinning 구현
- [ ] 44. Retrofit + Kotlin Sealed class 에러 처리 패턴
- [ ] 45. Retrofit + Flow 연동 패턴

### UI / 커스텀 뷰 (12개)

- [ ] 46. CustomView 만들기 — onDraw, onMeasure, onLayout
- [ ] 47. Canvas 그리기 — Path, Paint, Bitmap 조작
- [ ] 48. Android 애니메이션 — ObjectAnimator, ValueAnimator
- [ ] 49. Android Transition API — 화면 전환 애니메이션
- [ ] 50. Shared Element Transition — 자연스러운 화면 이동
- [ ] 51. MotionLayout 기초 — ConstraintSet 애니메이션
- [ ] 52. ConstraintLayout 심화 — Barrier, Chain, Group
- [ ] 53. RecyclerView ItemDecoration 커스터마이징
- [ ] 54. RecyclerView ItemAnimator 커스텀
- [ ] 55. RecyclerView SwipeToDismiss — ItemTouchHelper
- [ ] 56. RecyclerView Drag & Drop — ItemTouchHelper
- [ ] 57. GestureDetector — 스와이프, 롱프레스, 더블탭 감지

### 백그라운드 / 시스템 (10개)

- [ ] 58. Service vs IntentService vs JobIntentService 비교
- [ ] 59. Foreground Service 구현 — 알림과 함께
- [ ] 60. AlarmManager vs WorkManager 선택 기준
- [ ] 61. BroadcastReceiver 종류와 사용법 — 정적 vs 동적
- [ ] 62. ContentProvider 기초 — 앱 간 데이터 공유
- [ ] 63. FileProvider — 외부 앱에 파일 URI 공유
- [ ] 64. MediaStore — 사진/동영상 접근 (Scoped Storage)
- [ ] 65. Android 12+ 알림 채널 설정 및 권한 요청
- [ ] 66. CameraX 라이브러리 — 사진 촬영, 미리보기
- [ ] 67. App Widget 만들기 — RemoteViews

### Firebase / 서드파티 (8개)

- [ ] 68. Firebase Crashlytics 연동 및 이슈 추적
- [ ] 69. Firebase Analytics 이벤트 설계
- [ ] 70. Firebase Remote Config — A/B 테스트
- [ ] 71. FCM 푸시 알림 구현 — Token 관리, 페이로드 처리
- [ ] 72. Firebase App Distribution — 테스트 배포 자동화
- [ ] 73. ML Kit — 텍스트 인식 (OCR) 기초
- [ ] 74. Google Maps SDK 연동 — 마커, 경로, 커스텀
- [ ] 75. Fused Location Provider — 정확한 위치 수집

### 보안 / 성능 (8개)

- [ ] 76. EncryptedSharedPreferences — 민감 데이터 암호화
- [ ] 77. Network Security Config — 인증서 고정, 클리어텍스트 차단
- [ ] 78. Android Security — Root 감지 및 무결성 검증
- [ ] 79. Android Baseline Profiles — 앱 시작 속도 최적화
- [ ] 80. LeakCanary — 메모리 누수 탐지와 수정
- [ ] 81. Android Profiler 사용법 — CPU, Memory, Network
- [ ] 82. APK 크기 줄이기 — R8, 리소스 최적화, App Bundle
- [ ] 83. 다크 모드 대응 — Night Mode, color selector

### 빌드 / 배포 / 테스트 (9개)

- [ ] 84. Build Variant — productFlavor로 환경 분리
- [ ] 85. buildConfig 필드 활용 — API 키, 환경별 설정
- [ ] 86. ProGuard / R8 난독화 규칙 작성
- [ ] 87. Android 앱 서명 — keystore 관리 및 자동화
- [ ] 88. Android Testing — Espresso UI 테스트
- [ ] 89. Android Testing — Robolectric 기초
- [ ] 90. Android Testing — MockK + ViewModel 단위 테스트
- [ ] 91. CI/CD — GitHub Actions로 APK 자동 빌드/배포
- [ ] 92. Modular Architecture — 기능별 모듈 분리 전략

### 기타 실전 (8개)

- [ ] 93. In-App Review API — 앱 내 리뷰 요청
- [ ] 94. Android Shortcuts — 정적/동적/고정 단축키
- [ ] 95. 폴더블 기기 대응 — WindowSizeClass, 적응형 레이아웃
- [ ] 96. Adaptive Icon 설정
- [ ] 97. Android Accessibility — TalkBack, Content Description
- [ ] 98. Android Network 상태 감지 — ConnectivityManager
- [ ] 99. Android dp/sp/px 단위 완전 정리 + 변환 확장함수
- [ ] 100. Android In-App Update API — 강제/유연 업데이트

---

## ☕ Java (100개)

### Stream / 함수형 (12개)

- [ ] 1. Stream API 기초 — filter, map, collect
- [ ] 2. Stream API 심화 — flatMap, reduce, groupingBy
- [ ] 3. Parallel Stream 성능과 주의사항
- [ ] 4. Optional 완벽 가이드 — orElse, orElseGet, orElseThrow
- [ ] 5. Lambda 표현식 기초
- [ ] 6. 함수형 인터페이스 — Function, Predicate, Consumer, Supplier
- [ ] 7. Method Reference — :: 4가지 유형
- [ ] 8. 익명 클래스 vs Lambda 비교
- [ ] 9. Stream Collector 커스텀 — Collector.of()
- [ ] 10. Stream 성능 최적화 — short-circuit, lazy evaluation
- [ ] 11. Stream vs for-loop 성능 비교
- [ ] 12. Comparator 체이닝 — thenComparing, reversed

### 제네릭 / 타입 시스템 (8개)

- [ ] 13. 제네릭 wildcard — ? extends vs ? super (PECS 원칙)
- [ ] 14. 제네릭 Type Erasure — 런타임 타입 정보 소실
- [ ] 15. 제네릭 bounded type parameter
- [ ] 16. Enum 심화 — abstract method, interface 구현
- [ ] 17. Annotation 만들기 — @interface, RetentionPolicy
- [ ] 18. Reflection API — Class, Method, Field 동적 접근
- [ ] 19. 동적 프록시 — Proxy.newProxyInstance
- [ ] 20. ClassLoader 동작 원리

### 새 Java 기능 (Java 8~21) (12개)

- [ ] 21. 날짜/시간 — java.time API 완전 정리
- [ ] 22. 날짜/시간 — DateTimeFormatter 패턴
- [ ] 23. 날짜/시간 — Duration vs Period
- [ ] 24. var (지역 변수 타입 추론) — Java 10+
- [ ] 25. text block (멀티라인 문자열) — Java 15+
- [ ] 26. Record — Java 16+ 불변 데이터 클래스
- [ ] 27. Sealed class — Java 17+
- [ ] 28. instanceof 패턴 매칭 심화 — switch 표현식
- [ ] 29. switch 표현식 — Java 14+
- [ ] 30. 인터페이스 private method — Java 9+
- [ ] 31. 가상 스레드 (Virtual Threads) — Java 21
- [ ] 32. Pattern Matching for switch — Java 21

### JVM 내부 / 메모리 (10개)

- [ ] 33. Java 메모리 모델 (JMM) — happens-before 관계
- [ ] 34. GC 심화 — G1GC vs ZGC vs Shenandoah 비교
- [ ] 35. JIT 컴파일러 동작 원리 — 인터프리터 vs 컴파일
- [ ] 36. String Pool — intern()과 메모리 관리
- [ ] 37. StringBuilder 내부 구현 — char[], capacity 확장
- [ ] 38. 바이트코드 이해 — javap 활용
- [ ] 39. StackOverflowError 원인과 해결
- [ ] 40. OutOfMemoryError 종류별 원인과 대응
- [ ] 41. Java 모듈 시스템 — module-info.java
- [ ] 42. JShell — REPL 환경 활용

### 동시성 / 스레드 (12개)

- [ ] 43. ExecutorService와 Thread Pool 완전 정리
- [ ] 44. Future vs CompletableFuture 기초
- [ ] 45. CompletableFuture 심화 — thenApply, thenCompose, exceptionally
- [ ] 46. Fork/Join Framework
- [ ] 47. volatile 심화 — happens-before, 가시성 보장
- [ ] 48. synchronized 심화 — lock, condition, wait/notify
- [ ] 49. ReentrantLock vs synchronized 비교
- [ ] 50. ReadWriteLock — 읽기/쓰기 분리
- [ ] 51. ConcurrentHashMap 내부 동작 원리
- [ ] 52. BlockingQueue — producer-consumer 패턴
- [ ] 53. ThreadLocal 활용 — 스레드별 독립 저장소
- [ ] 54. 불변 객체 (Immutable Object) 설계 원칙

### 객체 설계 (8개)

- [ ] 55. equals() & hashCode() 계약 (Contract) 완전 정리
- [ ] 56. 방어적 복사 (Defensive Copy)
- [ ] 57. Cloneable과 clone()의 문제점
- [ ] 58. Serializable 직렬화와 보안 취약점
- [ ] 59. Comparable vs Comparator 심화
- [ ] 60. 인터페이스 default method와 다중 상속
- [ ] 61. 추상 클래스 vs 인터페이스 실전 선택 기준
- [ ] 62. Java LRU Cache 구현 — LinkedHashMap 활용

### 디자인 패턴 — GoF 행동 패턴 Java 구현 (14개)

- [ ] 63. 옵저버 패턴 (Observer) — EventListener 구현
- [ ] 64. 전략 패턴 (Strategy) — 알고리즘 교체
- [ ] 65. 커맨드 패턴 (Command) — undo/redo 구현
- [ ] 66. 책임 연쇄 패턴 (Chain of Responsibility)
- [ ] 67. 상태 패턴 (State) — 상태 기계 구현
- [ ] 68. 템플릿 메서드 패턴 (Template Method)
- [ ] 69. 이터레이터 패턴 (Iterator)
- [ ] 70. 중재자 패턴 (Mediator)
- [ ] 71. 방문자 패턴 (Visitor)
- [ ] 72. 메멘토 패턴 (Memento) — 상태 저장/복원
- [ ] 73. 플라이웨이트 패턴 (Flyweight)
- [ ] 74. 브리지 패턴 (Bridge)
- [ ] 75. 인터프리터 패턴 (Interpreter)
- [ ] 76. 널 객체 패턴 (Null Object)

### 자료구조 / 알고리즘 Java 구현 (12개)

- [ ] 77. 이진 트리 (Binary Tree) 구현
- [ ] 78. 이진 탐색 트리 (BST) — 삽입, 삭제, 탐색
- [ ] 79. 힙 (Heap) 구현 — PriorityQueue 내부
- [ ] 80. 트라이 (Trie) — 문자열 탐색
- [ ] 81. 그래프 — 인접 행렬 vs 인접 리스트
- [ ] 82. BFS & DFS 구현
- [ ] 83. 동적 프로그래밍 — 메모이제이션 vs 타뷸레이션
- [ ] 84. DP 유형 — LCS, LIS, 배낭 문제
- [ ] 85. 그리디 — 증명 방법과 대표 문제
- [ ] 86. 이분 탐색 응용 — parametric search
- [ ] 87. 슬라이딩 윈도우 패턴
- [ ] 88. 투 포인터 패턴

### 테스트 / 빌드 / 코드 품질 (12개)

- [ ] 89. JUnit5 기초 — @Test, @BeforeEach, @ParameterizedTest
- [ ] 90. Mockito 기초 — mock, when, verify
- [ ] 91. Mockito 심화 — ArgumentCaptor, InOrder
- [ ] 92. TDD (Test-Driven Development) 실전
- [ ] 93. BDD (Behavior-Driven Development) — Given/When/Then
- [ ] 94. 통합 테스트 vs 단위 테스트 설계 원칙
- [ ] 95. Maven vs Gradle 비교
- [ ] 96. SLF4J + Logback 로깅 설정
- [ ] 97. 의존성 주입 원리 — DI 컨테이너 직접 구현
- [ ] 98. Checkstyle, SpotBugs, PMD 코드 품질 도구
- [ ] 99. Java IO vs NIO — InputStream, ByteBuffer, Channel
- [ ] 100. 소켓 프로그래밍 기초 — TCP ServerSocket 구현

---

## 🔵 Computer Science (100개)

### 운영체제 (18개)

- [ ] 1. CPU 스케줄링 — FCFS, SJF, Round Robin, Priority 비교
- [ ] 2. 컨텍스트 스위칭 (Context Switching) 원리와 비용
- [ ] 3. 인터럽트 (Interrupt) 동작 원리
- [ ] 4. 시스템 콜 (System Call) — 유저 모드 vs 커널 모드
- [ ] 5. 가상 메모리 — 페이지 폴트, 스왑
- [ ] 6. 메모리 할당 전략 — 최초/최적/최악 적합
- [ ] 7. 교착상태 4가지 조건과 해결 전략 (예방, 회피, 탐지)
- [ ] 8. 공유 메모리 vs 메시지 패싱 IPC
- [ ] 9. 파일 시스템 구조 — inode, directory entry
- [ ] 10. Blocking vs Non-Blocking vs Async I/O 비교
- [ ] 11. 스핀락 (Spinlock) vs 뮤텍스 vs 세마포어 비교
- [ ] 12. 캐시 메모리 — L1/L2/L3, 지역성 원리
- [ ] 13. 페이지 교체 알고리즘 — FIFO, LRU, Optimal
- [ ] 14. 스래싱 (Thrashing) 원인과 해결
- [ ] 15. 프로세스 생성 — fork(), exec()
- [ ] 16. 고아 프로세스 vs 좀비 프로세스
- [ ] 17. 실시간 OS (RTOS) 개념
- [ ] 18. 멀티코어 CPU — 병렬성 vs 동시성

### 네트워크 (18개)

- [ ] 19. HTTP/1.1 vs HTTP/2 vs HTTP/3 비교
- [ ] 20. HTTPS 동작 원리 — TLS Handshake 상세
- [ ] 21. DNS 동작 원리 — 재귀 vs 반복 질의, TTL
- [ ] 22. CDN (Content Delivery Network) 원리
- [ ] 23. 로드 밸런서 — L4 vs L7, 알고리즘 비교
- [ ] 24. Forward Proxy vs Reverse Proxy 비교
- [ ] 25. WebSocket vs HTTP Polling vs SSE
- [ ] 26. gRPC vs REST API 비교
- [ ] 27. OAuth 2.0 인증 흐름 완전 정리
- [ ] 28. API Rate Limiting 구현 전략
- [ ] 29. IP 주소 체계 — IPv4 vs IPv6, 서브넷 마스크
- [ ] 30. NAT (Network Address Translation) 동작 원리
- [ ] 31. HTTP 캐시 — Cache-Control, ETag, Last-Modified
- [ ] 32. 쿠키 vs 세션 vs 토큰 기반 인증 비교
- [ ] 33. HTTPS 인증서 구조와 신뢰 체계 (CA)
- [ ] 34. 네트워크 패킷 스니핑과 방어
- [ ] 35. ARP 프로토콜 동작 원리
- [ ] 36. ICMP — ping과 traceroute 원리

### 데이터베이스 (18개)

- [ ] 37. 인덱스 원리 — B-Tree vs B+Tree
- [ ] 38. 트랜잭션 ACID 속성 완전 정리
- [ ] 39. 격리 수준 — Read Uncommitted ~ Serializable
- [ ] 40. 정규화 1NF~3NF, BCNF
- [ ] 41. 조인 종류 — INNER, LEFT, RIGHT, FULL, CROSS, SELF
- [ ] 42. 인덱스 설계 원칙 — 복합 인덱스, Cardinality
- [ ] 43. 쿼리 최적화 — EXPLAIN 읽는 법
- [ ] 44. N+1 문제 원인과 해결 방법
- [ ] 45. 낙관적 락 vs 비관적 락 비교
- [ ] 46. 복제 (Replication) — Master-Slave, GTID
- [ ] 47. 샤딩 (Sharding) 전략 — Range, Hash, Directory
- [ ] 48. CAP 정리 완전 정리
- [ ] 49. Redis 기초 — 자료구조 5종과 활용 패턴
- [ ] 50. Redis 캐싱 전략 — Cache Aside, Write Through, Write Back
- [ ] 51. Redis vs Memcached 비교
- [ ] 52. 파티셔닝 vs 샤딩 차이
- [ ] 53. 데이터베이스 커넥션 풀 — HikariCP
- [ ] 54. 인덱스 Covering Index

### 자료구조 (14개)

- [ ] 55. 이진 탐색 트리 (BST) — 삽입, 삭제, 탐색 복잡도
- [ ] 56. AVL 트리 — 자가 균형 이진 탐색 트리
- [ ] 57. 레드-블랙 트리 개요
- [ ] 58. B-Tree vs B+Tree 비교
- [ ] 59. 트라이 (Trie) — 문자열 탐색과 자동완성
- [ ] 60. 우선순위 큐 (Priority Queue) — 힙 내부 구현
- [ ] 61. 세그먼트 트리 (Segment Tree)
- [ ] 62. 펜윅 트리 (Fenwick Tree / BIT)
- [ ] 63. 유니온 파인드 (Union-Find)
- [ ] 64. 그래프 표현 — 인접 행렬 vs 인접 리스트
- [ ] 65. 블룸 필터 (Bloom Filter)
- [ ] 66. LRU Cache — 구현 원리
- [ ] 67. 원형 버퍼 (Circular Buffer)
- [ ] 68. 희소 테이블 (Sparse Table)

### 알고리즘 (12개)

- [ ] 69. BFS vs DFS 완전 비교 — 선택 기준
- [ ] 70. 다익스트라 최단경로 알고리즘
- [ ] 71. 벨만-포드 — 음수 간선 처리
- [ ] 72. 플로이드-워셜 — 모든 쌍 최단경로
- [ ] 73. 크루스칼, 프림 — 최소 신장 트리
- [ ] 74. 위상 정렬 (Topological Sort)
- [ ] 75. DP 유형 — LCS, LIS, 배낭 문제 패턴
- [ ] 76. 그리디 — 증명 방법과 대표 문제 유형
- [ ] 77. 백트래킹 — N-Queen, 순열/조합
- [ ] 78. 비트마스킹 기초와 활용 패턴
- [ ] 79. 분할 정복 (Divide & Conquer) 패턴
- [ ] 80. 에라토스테네스의 체 — 소수 판별 최적화

### 시스템 설계 (10개)

- [ ] 81. URL 단축기 시스템 설계
- [ ] 82. Rate Limiter 설계 — 토큰 버킷, 슬라이딩 윈도우
- [ ] 83. 채팅 시스템 설계 — WebSocket, 메시지 저장
- [ ] 84. 알림 시스템 설계 — Push, Email, SMS 멀티채널
- [ ] 85. 검색 자동완성 설계 — Trie + 캐싱
- [ ] 86. 피드 시스템 — Fanout on Write vs Read
- [ ] 87. 분산 트랜잭션 — Saga 패턴
- [ ] 88. 이벤트 소싱 (Event Sourcing) + CQRS
- [ ] 89. 메시지 큐 — Kafka 기초 개념
- [ ] 90. 마이크로서비스 vs 모놀리식 비교

### 보안 / DevOps / 소프트웨어 공학 (10개)

- [ ] 91. SQL Injection 원리와 Prepared Statement 방어
- [ ] 92. XSS (Cross-Site Scripting) 원리와 방어
- [ ] 93. CSRF (Cross-Site Request Forgery) 방어 전략
- [ ] 94. 비밀번호 해싱 — bcrypt, Argon2, PBKDF2 비교
- [ ] 95. Docker 기초 — 이미지, 컨테이너, Dockerfile
- [ ] 96. CI/CD 파이프라인 설계 원칙
- [ ] 97. 블루-그린 배포 vs 카나리 배포 비교
- [ ] 98. DRY, KISS, YAGNI 원칙 실전 적용
- [ ] 99. 12 Factor App 원칙
- [ ] 100. 기술 부채 (Technical Debt) 측정과 관리 전략
