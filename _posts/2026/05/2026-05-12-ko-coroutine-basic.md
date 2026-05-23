---
title: (Kotlin/코틀린) 코루틴 기초 완전 정리 - suspend fun, launch vs async
tags: [ Kotlin, Android ]
style: fill
color: dark
description: Kotlin 코루틴의 핵심 개념인 suspend fun의 동작 원리, CoroutineScope, launch와 async의 차이, runBlocking까지 Android 실전 코드와 함께 깊이 있게 정리합니다.
---

## 개요

- Kotlin의 **코루틴(Coroutine)** 기초를 다룹니다.
- 코루틴은 **비동기 코드를 동기 코드처럼 읽히게** 작성할 수 있는 Kotlin의 핵심 기능입니다.
- 이 글에서는 다음을 설명합니다.
  - 코루틴이란 무엇인가 — 스레드와의 차이
  - `suspend fun` 동작 원리
  - `CoroutineScope`와 `CoroutineContext`
  - `launch` vs `async` 차이
  - `runBlocking` 사용법과 주의사항
  - Android 실전 예제

---

## 1. 코루틴이란

코루틴은 **중단(suspend)하고 재개(resume)할 수 있는 실행 단위**입니다.

```
일반 함수:  호출 → 실행 → 반환  (중단 불가)
코루틴:     호출 → 실행 → 중단 → 재개 → 반환  (중단 가능)
```

### 스레드 vs 코루틴

| 항목 | 스레드 | 코루틴 |
|------|--------|--------|
| 생성 비용 | 높음 (OS 자원) | 낮음 (가벼운 객체) |
| 동시 실행 수 | 수백~수천 | 수만~수십만 |
| 블로킹 | 스레드 전체 대기 | 해당 코루틴만 중단 |
| 문맥 전환 | OS 스케줄러 | 코루틴 런타임 |
| 코드 스타일 | 콜백/블로킹 | 순차적 코드 |

```kotlin
// 스레드 방식 — 콜백 지옥
fun loadUserThread(userId: Long, callback: (User) -> Unit) {
    Thread {
        val user = api.getUser(userId)   // 스레드 블로킹
        runOnUiThread { callback(user) }
    }.start()
}

// 코루틴 방식 — 순차적으로 읽히는 코드
suspend fun loadUser(userId: Long): User {
    return api.getUser(userId)   // 코루틴만 중단, 스레드는 자유
}
```

---

## 2. 의존성 설정

```kotlin
// build.gradle.kts
dependencies {
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.8.1")
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.8.1")

    // Android ViewModel에서 viewModelScope 사용
    implementation("androidx.lifecycle:lifecycle-viewmodel-ktx:2.8.3")
    implementation("androidx.lifecycle:lifecycle-runtime-ktx:2.8.3")
}
```

---

## 3. suspend fun — 중단 가능한 함수

`suspend` 키워드는 **"이 함수는 중단될 수 있다"** 는 선언입니다.

```kotlin
// suspend fun 선언
suspend fun fetchUser(userId: Long): User {
    delay(1000)              // 1초 중단 (스레드는 블로킹되지 않음)
    return api.getUser(userId)
}
```

### suspend fun의 규칙

```kotlin
// ✅ suspend fun은 코루틴 또는 다른 suspend fun 안에서만 호출 가능
suspend fun fetchData(): String {
    delay(500)
    return "data"
}

suspend fun processData() {
    val data = fetchData()   // ✅ suspend fun 안에서 호출
    println(data)
}

// ❌ 일반 함수에서 직접 호출 불가
fun normalFunction() {
    val data = fetchData()   // 컴파일 에러
}
```

### delay vs Thread.sleep

```kotlin
suspend fun example() {
    // ✅ delay — 코루틴만 중단, 스레드는 다른 코루틴 처리 가능
    delay(1000)

    // ❌ Thread.sleep — 스레드 자체를 블로킹 (코루틴 이점 사라짐)
    Thread.sleep(1000)
}
```

---

## 4. CoroutineScope — 코루틴의 범위

코루틴은 반드시 **CoroutineScope** 안에서 시작됩니다. Scope는 코루틴의 **생명주기**를 관리합니다.

```kotlin
// CoroutineScope 직접 생성
val scope = CoroutineScope(Dispatchers.IO)
scope.launch {
    // IO 스레드에서 실행
}
scope.cancel()   // 모든 자식 코루틴 취소

// Android 제공 Scope — 생명주기 자동 관리
class MyViewModel : ViewModel() {
    init {
        viewModelScope.launch { ... }  // ViewModel 소멸 시 자동 취소
    }
}

class MyFragment : Fragment() {
    override fun onViewCreated(...) {
        viewLifecycleOwner.lifecycleScope.launch { ... }  // View 소멸 시 자동 취소
    }
}
```

### CoroutineContext — 코루틴의 실행 환경

```kotlin
// CoroutineContext = Dispatcher + Job + CoroutineName + ...
val scope = CoroutineScope(
    Dispatchers.IO +           // 어떤 스레드에서 실행할지
    SupervisorJob() +          // 자식 실패가 부모에게 전파되지 않음
    CoroutineName("MyScope")   // 디버깅용 이름
)
```

| Dispatcher | 실행 스레드 | 사용 시점 |
|------------|------------|---------|
| `Dispatchers.Main` | 메인(UI) 스레드 | UI 업데이트 |
| `Dispatchers.IO` | IO 스레드 풀 | 네트워크, 파일, DB |
| `Dispatchers.Default` | CPU 스레드 풀 | 계산, 파싱 |
| `Dispatchers.Unconfined` | 호출자 스레드 | 특수 목적 |

---

## 5. launch — 결과 없는 코루틴

`launch`는 **반환값이 필요 없는 코루틴**을 시작합니다. `Job` 객체를 반환합니다.

```kotlin
val job: Job = scope.launch {
    println("코루틴 시작")
    delay(1000)
    println("1초 후")
}

// Job으로 코루틴 제어
job.cancel()    // 취소
job.join()      // 완료까지 대기 (suspend fun)
job.isActive    // 실행 중 여부
```

### launch 실전 예제

```kotlin
@HiltViewModel
class UserViewModel @Inject constructor(
    private val userRepository: UserRepository
) : ViewModel() {

    private val _uiState = MutableStateFlow(UserUiState())
    val uiState: StateFlow<UserUiState> = _uiState.asStateFlow()

    fun loadUser(userId: Long) {
        viewModelScope.launch {                          // ① 코루틴 시작
            _uiState.update { it.copy(isLoading = true) }

            try {
                val user = userRepository.getUser(userId)  // ② suspend fun 호출
                _uiState.update { it.copy(isLoading = false, user = user) }
            } catch (e: Exception) {
                _uiState.update { it.copy(isLoading = false, error = e.message) }
            }
        }
    }

    fun deleteUser(userId: Long) {
        viewModelScope.launch {
            userRepository.deleteUser(userId)
            // 반환값 필요 없음 — launch 적합
        }
    }
}
```

---

## 6. async — 결과 있는 코루틴

`async`는 **반환값이 있는 코루틴**을 시작합니다. `Deferred<T>` 객체를 반환하며, `await()`로 결과를 받습니다.

```kotlin
val deferred: Deferred<String> = scope.async {
    delay(1000)
    "결과값"    // 마지막 표현식이 반환값
}

val result: String = deferred.await()   // 완료까지 대기 후 결과 반환
println(result)   // "결과값"
```

### async의 핵심 — 병렬 실행

`async`의 진가는 **여러 작업을 동시에 실행**할 때 발휘됩니다.

```kotlin
suspend fun loadDashboard(): Dashboard {

    // ❌ 순차 실행 — 총 3초 소요
    val user    = fetchUser()      // 1초
    val posts   = fetchPosts()     // 1초
    val notices = fetchNotices()   // 1초

    return Dashboard(user, posts, notices)
}

suspend fun loadDashboardFast(): Dashboard {

    // ✅ 병렬 실행 — 총 1초 소요 (가장 느린 작업 기준)
    val userDeferred    = coroutineScope { async { fetchUser() } }
    val postsDeferred   = coroutineScope { async { fetchPosts() } }
    val noticesDeferred = coroutineScope { async { fetchNotices() } }

    return Dashboard(
        user    = userDeferred.await(),
        posts   = postsDeferred.await(),
        notices = noticesDeferred.await()
    )
}
```

더 간결하게는 `coroutineScope { }` 안에서 함께 선언합니다.

```kotlin
suspend fun loadDashboardFast(): Dashboard = coroutineScope {
    val user    = async { fetchUser() }
    val posts   = async { fetchPosts() }
    val notices = async { fetchNotices() }

    Dashboard(
        user    = user.await(),
        posts   = posts.await(),
        notices = notices.await()
    )
}
```

---

## 7. launch vs async 완전 비교

| 항목 | `launch` | `async` |
|------|----------|---------|
| 반환 타입 | `Job` | `Deferred<T>` |
| 결과 수신 | 없음 | `await()`로 수신 |
| 예외 발생 시 | 즉시 전파 | `await()` 호출 시 전파 |
| 주요 용도 | 부수효과 (저장, 전송) | 값 계산, 병렬 처리 |
| 취소 | `job.cancel()` | `deferred.cancel()` |

```kotlin
// launch — 반환값 없는 작업
viewModelScope.launch {
    repository.saveUser(user)   // 저장 후 결과 불필요
}

// async — 반환값 있는 작업
viewModelScope.launch {
    val result = async { repository.getUser(userId) }.await()
    // result 활용
}

// async 병렬 처리 — 가장 일반적인 사용
viewModelScope.launch {
    val (profile, settings) = coroutineScope {
        val p = async { repository.getProfile(userId) }
        val s = async { repository.getSettings(userId) }
        Pair(p.await(), s.await())
    }
    // profile, settings 동시에 준비됨
}
```

---

## 8. runBlocking — 코루틴과 일반 코드 연결

`runBlocking`은 **현재 스레드를 블로킹**하면서 코루틴을 실행합니다. 주로 테스트와 `main()` 함수에서 사용합니다.

```kotlin
// main 함수에서 코루틴 진입점으로 사용
fun main() = runBlocking {
    val result = async { fetchData() }.await()
    println(result)
}

// 테스트 코드에서 사용
@Test
fun `유저 데이터를 올바르게 로드한다`() = runBlocking {
    val user = userRepository.getUser(1L)
    assertEquals("홍길동", user.name)
}
```

### runBlocking 주의사항

```kotlin
// ❌ Android 메인 스레드에서 runBlocking 금지 — ANR 발생
class MainActivity : AppCompatActivity() {
    override fun onCreate(...) {
        runBlocking {          // UI 스레드 블로킹 → ANR
            delay(3000)
        }
    }
}

// ✅ 메인 스레드에서는 lifecycleScope 사용
class MainActivity : AppCompatActivity() {
    override fun onCreate(...) {
        lifecycleScope.launch {    // 메인 스레드 블로킹 없음
            delay(3000)
        }
    }
}
```

---

## 9. withContext — 스레드 전환

`withContext`는 **코루틴 안에서 Dispatcher를 전환**할 때 사용합니다. 새 코루틴을 시작하지 않고 현재 코루틴의 실행 스레드만 바꿉니다.

```kotlin
// 네트워크 → 메인 스레드 전환 패턴
suspend fun loadAndDisplay(userId: Long) {
    val user = withContext(Dispatchers.IO) {
        api.getUser(userId)       // IO 스레드에서 네트워크 호출
    }
    withContext(Dispatchers.Main) {
        tvName.text = user.name   // 메인 스레드에서 UI 업데이트
    }
}

// Repository 계층에서 Dispatcher 지정
class UserRepositoryImpl(
    private val userApi: UserApi,
    private val userDao: UserDao
) : UserRepository {

    override suspend fun getUser(userId: Long): User {
        return withContext(Dispatchers.IO) {   // IO 스레드 강제
            userDao.getUser(userId)
                ?: userApi.getUser(userId).also { userDao.insert(it) }
        }
    }
}
```

### withContext vs launch vs async 비교

```kotlin
// withContext — 결과를 반환하며 현재 코루틴에서 실행 (새 코루틴 없음)
val result = withContext(Dispatchers.IO) { fetchData() }

// launch — 새 코루틴 시작, 결과 없음
launch(Dispatchers.IO) { saveData() }

// async — 새 코루틴 시작, 결과 있음
val deferred = async(Dispatchers.IO) { fetchData() }
```

---

## 10. coroutineScope vs GlobalScope

```kotlin
// ✅ coroutineScope — 부모 코루틴의 범위를 상속, 구조화된 동시성
suspend fun doWork() = coroutineScope {
    val a = async { task1() }
    val b = async { task2() }
    a.await() + b.await()
    // 자식 코루틴 하나가 실패하면 나머지도 취소됨
    // 부모가 취소되면 자식도 취소됨
}

// ❌ GlobalScope — 앱 생명주기와 연결, 메모리 누수 위험
GlobalScope.launch {
    // 취소 불가, 생명주기 관리 안 됨 — 사용 자제
    delay(Long.MAX_VALUE)
}
```

---

## 11. Android 실전 예제 — 병렬 API 호출

```kotlin
@HiltViewModel
class HomeViewModel @Inject constructor(
    private val userRepository: UserRepository,
    private val postRepository: PostRepository,
    private val noticeRepository: NoticeRepository
) : ViewModel() {

    private val _uiState = MutableStateFlow(HomeUiState())
    val uiState: StateFlow<HomeUiState> = _uiState.asStateFlow()

    fun loadHome(userId: Long) {
        viewModelScope.launch {
            _uiState.update { it.copy(isLoading = true) }

            try {
                // 세 API를 병렬 호출 — 총 소요시간 = 가장 느린 하나
                val (user, posts, notices) = coroutineScope {
                    val u = async { userRepository.getUser(userId) }
                    val p = async { postRepository.getPosts(userId) }
                    val n = async { noticeRepository.getNotices() }
                    Triple(u.await(), p.await(), n.await())
                }

                _uiState.update {
                    it.copy(
                        isLoading = false,
                        user      = user,
                        posts     = posts,
                        notices   = notices
                    )
                }
            } catch (e: Exception) {
                _uiState.update {
                    it.copy(isLoading = false, errorMessage = e.message)
                }
            }
        }
    }
}
```

---

## 12. 주의사항

### ❌ suspend fun을 일반 스레드에서 블로킹 호출하지 않는다

```kotlin
// ❌ runBlocking으로 메인 스레드 블로킹
fun onClick() {
    val user = runBlocking { repository.getUser(1L) }   // ANR 위험
}

// ✅ launch로 비동기 실행
fun onClick() {
    lifecycleScope.launch {
        val user = repository.getUser(1L)
        tvName.text = user.name
    }
}
```

---

### ❌ async 결과를 즉시 await하면 병렬의 의미가 없다

```kotlin
// ❌ 이렇게 쓰면 launch와 동일 — 순차 실행
val user = async { fetchUser() }.await()   // 완료 대기
val post = async { fetchPost() }.await()   // 그다음 시작

// ✅ async 먼저 모두 시작하고, 나중에 await
val userDeferred = async { fetchUser() }
val postDeferred = async { fetchPost() }
val user = userDeferred.await()   // 둘 다 동시 진행 중
val post = postDeferred.await()
```

---

### ❌ GlobalScope 사용을 피한다

```kotlin
// ❌ GlobalScope — 생명주기 미관리, 메모리 누수
GlobalScope.launch { repository.saveLog(event) }

// ✅ 적절한 scope 사용
viewModelScope.launch { repository.saveLog(event) }
```

---

## 13. 정리

| 항목 | 내용 |
|------|------|
| 코루틴 | 중단/재개 가능한 실행 단위 — 스레드보다 가볍고 효율적 |
| `suspend fun` | 중단 가능한 함수 — 코루틴 또는 다른 suspend fun 안에서만 호출 |
| `CoroutineScope` | 코루틴의 생명주기 관리 — Android는 `viewModelScope`, `lifecycleScope` 사용 |
| `launch` | 반환값 없는 코루틴 — `Job` 반환 |
| `async` | 반환값 있는 코루틴 — `Deferred<T>` 반환, `await()`로 결과 수신 |
| `async` 병렬 처리 | 여러 `async`를 먼저 시작하고 나중에 `await()` — 총 소요시간 단축 |
| `withContext` | 스레드 전환 — 새 코루틴 없이 Dispatcher 변경 |
| `runBlocking` | 메인/테스트 진입점 — Android 메인 스레드 사용 금지 |
| `coroutineScope` | 구조화된 동시성 — 자식 실패 시 나머지 취소 |

- 코루틴의 핵심은 **"중단(suspend)은 스레드를 블로킹하지 않는다"** 는 것입니다.
- `launch`는 부수효과, `async`는 값 계산, `withContext`는 스레드 전환에 사용합니다.
- Android에서는 `GlobalScope` 대신 항상 `viewModelScope` 또는 `lifecycleScope`를 사용하세요.

---

## 참고

- [Kotlin Coroutines 공식 문서](https://kotlinlang.org/docs/coroutines-overview.html)
- [Android Coroutines 가이드](https://developer.android.com/kotlin/coroutines)
- [kotlinx.coroutines GitHub](https://github.com/Kotlin/kotlinx.coroutines)
