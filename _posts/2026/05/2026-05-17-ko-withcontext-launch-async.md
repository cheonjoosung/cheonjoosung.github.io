---
title: (Kotlin/코틀린) withContext vs launch vs async 차이 완전 정리
tags: [ Kotlin, Android ]
style: fill
color: dark
description: 코루틴의 핵심 빌더인 withContext, launch, async의 동작 원리와 반환 타입, 스레드 전환, 병렬 처리 차이를 실전 코드와 함께 깊이 있게 비교합니다.
---

## 개요

- 코루틴에서 가장 자주 쓰이는 세 가지 — **`withContext`**, **`launch`**, **`async`** 를 비교합니다.
- 셋 모두 코루틴 코드를 실행하지만 **목적과 동작이 다릅니다.**
- 이 글에서는 다음을 설명합니다.
  - 각각의 반환 타입과 동작 원리
  - 새 코루틴 생성 여부
  - 스레드 전환 방식
  - 예외 처리 차이
  - 상황별 올바른 선택 기준

---

## 1. 한눈에 보기

| 항목 | `withContext` | `launch` | `async` |
|------|-------------|---------|---------|
| 반환 타입 | `T` (결과값) | `Job` | `Deferred<T>` |
| 새 코루틴 생성 | ❌ (현재 코루틴 유지) | ✅ | ✅ |
| 호출 방식 | `suspend fun` 안에서 직접 | `scope.launch { }` | `scope.async { }` |
| 결과 수신 | 즉시 반환 | 없음 | `await()` |
| 실행 방식 | 순차 (현재 코루틴 중단) | 비동기 (즉시 반환) | 비동기 (즉시 반환) |
| 주 용도 | 스레드 전환 | 부수효과 | 값 계산·병렬 처리 |

---

## 2. withContext

`withContext`는 **현재 코루틴 안에서 Dispatcher(스레드)를 바꿔 코드를 실행** 합니다.
새로운 코루틴을 만들지 않고, 블록이 끝나면 원래 Dispatcher로 돌아옵니다.

```kotlin
suspend fun loadUser(userId: Long): User {
    // IO 스레드에서 네트워크 호출
    val user = withContext(Dispatchers.IO) {
        api.getUser(userId)   // IO 스레드에서 실행
    }
    // 다시 원래 Dispatcher(Main)로 복귀
    println("현재 스레드: ${Thread.currentThread().name}")   // main
    return user
}
```

### withContext의 동작 흐름

```
코루틴(Main):  [실행] → withContext(IO) 진입
                              ↓
IO 스레드:                 [API 호출] → 완료
                              ↓
코루틴(Main):          ← withContext 반환 → [계속 실행]
```

### Repository 계층 — Dispatcher 지정 패턴

`withContext`의 가장 일반적인 용도는 **Repository에서 IO 스레드를 강제** 하는 것입니다.

```kotlin
class UserRepositoryImpl(
    private val userApi: UserApi,
    private val userDao: UserDao
) : UserRepository {

    // ✅ Repository가 직접 Dispatcher 책임 — 호출자는 신경 안 써도 됨
    override suspend fun getUser(userId: Long): Result<User> {
        return withContext(Dispatchers.IO) {
            runCatching {
                userDao.getUser(userId)
                    ?: userApi.getUser(userId).also { userDao.insert(it) }
            }
        }
    }

    override suspend fun saveUser(user: User): Result<Unit> {
        return withContext(Dispatchers.IO) {
            runCatching { userDao.insert(user) }
        }
    }
}

// ✅ ViewModel은 Dispatcher를 몰라도 됨
class UserViewModel @Inject constructor(
    private val userRepository: UserRepository
) : ViewModel() {
    fun loadUser(userId: Long) {
        viewModelScope.launch {   // Main Dispatcher
            val result = userRepository.getUser(userId)   // 내부에서 IO 전환
            result.onSuccess { _uiState.update { s -> s.copy(user = it) } }
        }
    }
}
```

### withContext로 CPU 집중 작업 분리

```kotlin
suspend fun parseJson(raw: String): List<Item> {
    return withContext(Dispatchers.Default) {   // CPU 스레드 풀
        gson.fromJson(raw, Array<Item>::class.java).toList()
    }
}

suspend fun resizeBitmap(bitmap: Bitmap, width: Int, height: Int): Bitmap {
    return withContext(Dispatchers.Default) {
        Bitmap.createScaledBitmap(bitmap, width, height, true)
    }
}
```

---

## 3. launch

`launch`는 **새 코루틴을 시작하고 즉시 `Job`을 반환** 합니다.
결과값이 없는 **부수효과(fire-and-forget)** 작업에 사용합니다.

```kotlin
val job: Job = viewModelScope.launch {
    repository.saveLog(event)   // 반환값 필요 없음
}

// Job으로 코루틴 제어
job.cancel()          // 취소
job.join()            // 완료까지 대기 (suspend)
println(job.isActive) // 실행 중 여부
```

### launch의 동작 흐름

```
launch 호출
    ↓ 즉시 Job 반환 (블로킹 없음)
    ↓
[새 코루틴 비동기 실행]   ← 호출자와 병렬로 실행
```

```kotlin
viewModelScope.launch {
    println("A")            // 1번째
    launch {
        println("B")        // 3번째 (새 코루틴, 즉시 반환)
    }
    println("C")            // 2번째
}
// 출력: A → C → B
```

### launch 실전 패턴

```kotlin
@HiltViewModel
class OrderViewModel @Inject constructor(
    private val orderRepository: OrderRepository
) : ViewModel() {

    // ✅ 결과 필요 없는 저장 작업
    fun submitOrder(order: Order) {
        viewModelScope.launch {
            _uiState.update { it.copy(isLoading = true) }
            try {
                orderRepository.submit(order)
                _effect.send(OrderEffect.ShowToast("주문 완료"))
            } catch (e: Exception) {
                _effect.send(OrderEffect.ShowToast("주문 실패: ${e.message}"))
            } finally {
                _uiState.update { it.copy(isLoading = false) }
            }
        }
    }

    // ✅ 여러 독립 작업을 각각 launch
    fun init(userId: Long) {
        viewModelScope.launch { loadUser(userId) }
        viewModelScope.launch { loadNotices() }   // 독립 실행
        viewModelScope.launch { loadBanners() }   // 독립 실행
    }
}
```

---

## 4. async

`async`는 **새 코루틴을 시작하고 `Deferred<T>`를 반환** 합니다.
`await()`로 결과를 받으며, 핵심 강점은 **병렬 실행** 입니다.

```kotlin
val deferred: Deferred<User> = viewModelScope.async {
    repository.getUser(userId)
}

// 다른 작업 수행 가능...

val user: User = deferred.await()   // 완료까지 중단 후 결과 반환
```

### async 병렬 실행 — 핵심 사용법

```kotlin
// ❌ 순차 실행 — 총 3초 (async의 장점 없음)
suspend fun loadBad(): Dashboard = coroutineScope {
    val user = async { fetchUser() }.await()      // 1초 기다림
    val posts = async { fetchPosts() }.await()    // 그 다음 1초
    val ads = async { fetchAds() }.await()        // 그 다음 1초
    Dashboard(user, posts, ads)
}

// ✅ 병렬 실행 — 총 1초 (가장 느린 작업 기준)
suspend fun loadGood(): Dashboard = coroutineScope {
    val user  = async { fetchUser() }    // 즉시 반환, 병렬 시작
    val posts = async { fetchPosts() }   // 즉시 반환, 병렬 시작
    val ads   = async { fetchAds() }     // 즉시 반환, 병렬 시작

    Dashboard(
        user  = user.await(),    // 셋 다 완료될 때까지 중단
        posts = posts.await(),
        ads   = ads.await()
    )
}
```

### async의 동작 흐름

```
coroutineScope {
    async { fetchUser() }   → [코루틴1 시작] ─────────┐ 동시 실행
    async { fetchPosts() }  → [코루틴2 시작] ─────┐   │
    async { fetchAds() }    → [코루틴3 시작] ─┐   │   │
                                               ↓   ↓   ↓
    user.await()   ← 코루틴3 완료 ← 코루틴2 완료 ← 코루틴1 완료
}
```

---

## 5. withContext vs async — 헷갈리는 차이

둘 다 결과를 반환하지만 **동작 방식이 다릅니다.**

```kotlin
// withContext — 새 코루틴 없음, 현재 코루틴에서 순차 실행
suspend fun withContextExample(): String {
    val a = withContext(Dispatchers.IO) { fetchA() }   // A 완료 후
    val b = withContext(Dispatchers.IO) { fetchB() }   // B 시작
    return "$a $b"   // 총 소요시간 = A + B
}

// async — 새 코루틴 생성, 병렬 실행
suspend fun asyncExample(): String = coroutineScope {
    val a = async(Dispatchers.IO) { fetchA() }   // A 시작
    val b = async(Dispatchers.IO) { fetchB() }   // B 동시 시작
    "${a.await()} ${b.await()}"   // 총 소요시간 = max(A, B)
}
```

| 항목 | `withContext` | `async` |
|------|-------------|---------|
| 코루틴 생성 | ❌ | ✅ |
| 병렬 실행 | ❌ (순차) | ✅ |
| 결과 반환 | 즉시 | `await()` 호출 시 |
| 주 용도 | 스레드 전환 | 병렬 값 계산 |

---

## 6. 예외 처리 차이

```kotlin
// launch — 예외가 즉시 전파됨 (CoroutineExceptionHandler 필요)
val handler = CoroutineExceptionHandler { _, e ->
    println("launch 예외 처리: ${e.message}")
}
viewModelScope.launch(handler) {
    throw RuntimeException("에러")   // handler가 잡음
}

// async — 예외가 await() 호출 시 전파됨
viewModelScope.launch {
    val deferred = async {
        throw RuntimeException("에러")   // 여기서는 전파 안 됨
    }
    try {
        deferred.await()   // 여기서 예외 발생
    } catch (e: Exception) {
        println("async 예외 처리: ${e.message}")
    }
}
```

---

## 7. 상황별 선택 기준

```
결과값이 필요한가?
    ├── NO  → launch  (부수효과, 저장, 로깅)
    └── YES
         ├── 병렬 실행이 필요한가?
         │   ├── YES → async  (여러 API 동시 호출)
         │   └── NO  → withContext  (스레드 전환만 필요)
         └── 스레드만 바꾸고 싶은가?
             └── YES → withContext (Repository IO 전환)
```

### 실전 예제 — 세 가지 함께 사용

```kotlin
@HiltViewModel
class HomeViewModel @Inject constructor(
    private val userRepo: UserRepository,
    private val feedRepo: FeedRepository,
    private val adRepo: AdRepository
) : ViewModel() {

    fun loadHome(userId: Long) {
        viewModelScope.launch {                          // ① 부수효과 시작
            _uiState.update { it.copy(isLoading = true) }

            try {
                val (user, feed, ads) = coroutineScope {
                    val u = async { userRepo.getUser(userId) }    // ② 병렬 시작
                    val f = async { feedRepo.getFeed(userId) }    // ② 병렬 시작
                    val a = async { adRepo.getAds() }             // ② 병렬 시작
                    Triple(u.await(), f.await(), a.await())
                }

                // withContext는 Repository 내부에서 처리
                // userRepo.getUser() 내부: withContext(Dispatchers.IO) { ... }

                _uiState.update {
                    it.copy(isLoading = false, user = user, feed = feed, ads = ads)
                }
            } catch (e: Exception) {
                _uiState.update { it.copy(isLoading = false, error = e.message) }
            }
        }
    }
}
```

---

## 8. 주의사항

### ❌ withContext로 병렬 처리 시도

```kotlin
// ❌ withContext는 순차 실행 — 병렬 아님
suspend fun loadBad() {
    val a = withContext(Dispatchers.IO) { fetchA() }   // A 완료 후
    val b = withContext(Dispatchers.IO) { fetchB() }   // B 시작
}

// ✅ 병렬은 async 사용
suspend fun loadGood() = coroutineScope {
    val a = async { fetchA() }
    val b = async { fetchB() }
    a.await(); b.await()
}
```

---

### ❌ async 결과를 즉시 await

```kotlin
// ❌ 즉시 await — 순차 실행과 같음
val a = async { fetchA() }.await()   // A 완료까지 대기
val b = async { fetchB() }.await()   // 그다음 B

// ✅ 모두 시작 후 await
val dA = async { fetchA() }
val dB = async { fetchB() }
val a = dA.await()
val b = dB.await()
```

---

### ❌ Dispatcher 없이 launch로 IO 작업

```kotlin
// ❌ Main 스레드에서 블로킹 IO — ANR
viewModelScope.launch {
    val data = File("data.txt").readText()   // Main 스레드 블로킹
}

// ✅ withContext로 IO 스레드 전환
viewModelScope.launch {
    val data = withContext(Dispatchers.IO) {
        File("data.txt").readText()
    }
}
```

---

## 9. 정리

| 상황 | 선택 |
|------|------|
| 반환값 없는 비동기 작업 | `launch` |
| 스레드를 바꾸고 결과를 받고 싶다 | `withContext` |
| 여러 작업을 동시에 실행하고 결과를 모은다 | `async` + `await()` |
| Repository IO 작업 Dispatcher 지정 | `withContext(Dispatchers.IO)` |
| CPU 집중 계산 | `withContext(Dispatchers.Default)` |

- `withContext`는 **스레드 전환**, `launch`는 **부수효과**, `async`는 **병렬 값 계산** 이 핵심입니다.
- `async`의 진가는 **먼저 모두 시작하고 나중에 await** 할 때 나타납니다.
- Repository 계층에서 `withContext(Dispatchers.IO)`로 Dispatcher를 책임지면 ViewModel이 깔끔해집니다.

---

## 참고

- [Kotlin Coroutines 공식 문서](https://kotlinlang.org/docs/coroutines-overview.html)
- [Android Coroutines 모범 사례](https://developer.android.com/kotlin/coroutines/coroutines-best-practices)
- [withContext vs async — KotlinConf 발표](https://www.youtube.com/watch?v=a3agLJQ6vt8)
