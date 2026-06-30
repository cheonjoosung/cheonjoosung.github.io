---
title: (Kotlin/코틀린) Flow flowOn 완전 정리 — 스레드 전환 원리
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin Flow의 flowOn 연산자로 업스트림 스레드를 전환하는 원리를 이해하고, withContext와의 차이점 및 실전 사용 패턴을 정리합니다.
---

---

## 1. Flow의 기본 스레드 동작

Flow는 기본적으로 `collect`를 호출한 코루틴의 스레드에서 실행됩니다.

```kotlin
// Main 스레드에서 collect → 전체가 Main 스레드에서 실행
viewLifecycleOwner.lifecycleScope.launch(Dispatchers.Main) {
    flow {
        // Main 스레드 실행 (문제!)
        val data = database.heavyQuery()  // UI 스레드 블로킹
        emit(data)
    }.collect { data ->
        updateUI(data)  // Main 스레드
    }
}
```

무거운 작업을 Main 스레드에서 실행하면 ANR이 발생할 수 있습니다.  
이를 해결하는 것이 `flowOn`입니다.

---

## 2. flowOn 기본 사용

`flowOn`은 **자신보다 앞(업스트림)에 있는 연산자들의 실행 컨텍스트**를 지정합니다.

```kotlin
flow {
    val data = database.heavyQuery()  // IO 스레드에서 실행
    emit(data)
}
.map { it.toUiModel() }              // IO 스레드에서 실행 (flowOn 앞에 있음)
.flowOn(Dispatchers.IO)              // ← 이 앞의 모든 업스트림을 IO에서 실행
.collect { uiModel ->
    updateUI(uiModel)                 // Main 스레드에서 실행
}
```

**핵심 규칙**: `flowOn`은 **자신 앞(위쪽)의 코드에만 영향**을 줍니다. 뒤(아래쪽)는 영향 없음.

---

## 3. flowOn의 스트림 분리 원리

`flowOn`은 내부적으로 채널(Channel)을 생성해 업스트림과 다운스트림을 분리합니다.

```
업스트림 (IO 스레드)              다운스트림 (Main 스레드)
flow { emit(data) }  →[Channel]→  .collect { updateUI(it) }
.map { transform }
.flowOn(Dispatchers.IO)
```

업스트림은 IO 스레드에서 값을 채널에 넣고,  
다운스트림은 Main 스레드에서 채널에서 값을 꺼내 처리합니다.

---

## 4. 여러 번 flowOn 사용

`flowOn`은 여러 번 사용할 수 있으며, 각각 자신 바로 앞 구간에만 적용됩니다.

```kotlin
flow {
    emit(loadFromNetwork())     // IO 스레드 (첫 번째 flowOn 영향)
}
.flowOn(Dispatchers.IO)        // ← 위 flow 블록에 적용
.map { parseJson(it) }         // Default 스레드 (두 번째 flowOn 영향)
.filter { it.isValid() }       // Default 스레드
.flowOn(Dispatchers.Default)   // ← 위 map, filter에 적용
.collect { data ->
    updateUI(data)              // collect 호출 스레드 (Main)
}
```

```
[IO] → flowOn(IO) → [Channel] → [Default] → flowOn(Default) → [Channel] → [Main]
```

---

## 5. flowOn vs withContext

가장 많이 혼동하는 부분입니다.

```kotlin
// flowOn: 업스트림 전체의 컨텍스트 변경
flow {
    emit(1)
    emit(2)
    emit(3)
}
.map { it * 2 }
.flowOn(Dispatchers.IO)   // emit과 map이 모두 IO에서 실행

// withContext: emit 내 특정 블록만 전환
flow {
    val result = withContext(Dispatchers.IO) {
        // 이 블록만 IO에서 실행
        heavyDatabaseCall()
    }
    emit(result)  // 원래 컨텍스트로 복귀 후 emit
}
```

| 구분 | flowOn | withContext |
|------|--------|-------------|
| 적용 범위 | 앞쪽 연산자 전체 | 해당 블록만 |
| 사용 위치 | 연산자 체인 중간 | emit 내부 |
| 새 코루틴 | 생성 (채널 분리) | 생성 안 함 |
| 권장 상황 | 전체 생산 로직 분리 | 일부만 스레드 전환 |

### Flow 내 withContext 주의사항

```kotlin
// 잘못된 사용: emit과 다른 컨텍스트에서 emit 시도
flow {
    withContext(Dispatchers.IO) {
        emit(heavyCall())  // ⚠️ IllegalStateException 발생 가능
    }
}

// 올바른 방법 1: flowOn 사용
flow {
    emit(heavyCall())
}.flowOn(Dispatchers.IO)

// 올바른 방법 2: 결과를 변수에 저장 후 emit
flow {
    val result = withContext(Dispatchers.IO) { heavyCall() }
    emit(result)
}
```

---

## 6. 실전 패턴

### Repository 레이어에서 flowOn

```kotlin
class UserRepository(
    private val dao: UserDao,
    private val api: UserApi
) {
    // Repository에서 스레드 처리 → ViewModel은 신경 쓸 필요 없음
    fun getUsersFlow(): Flow<List<User>> =
        dao.getUsersFlow()
            .map { entities -> entities.map { it.toUser() } }
            .flowOn(Dispatchers.IO)

    fun getRemoteUsersFlow(): Flow<List<User>> = flow {
        val response = api.getUsers()
        emit(response.body()?.map { it.toUser() } ?: emptyList())
    }.flowOn(Dispatchers.IO)
}
```

```kotlin
// ViewModel: 스레드 처리 불필요
class UserViewModel(private val repo: UserRepository) : ViewModel() {
    val users: StateFlow<List<User>> = repo.getUsersFlow()
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), emptyList())
}
```

### CPU 집약적 작업 — Dispatchers.Default

```kotlin
fun processLargeDataFlow(data: List<RawData>): Flow<ProcessedData> =
    data.asFlow()
        .map { raw -> heavyComputation(raw) }    // CPU 작업
        .filter { it.isValid() }
        .flowOn(Dispatchers.Default)             // CPU-bound → Default

// IO 작업과 조합
flow { emit(loadRawData()) }
    .flowOn(Dispatchers.IO)          // 파일/네트워크 로드는 IO
    .map { processData(it) }         // 가공은 Default
    .flowOn(Dispatchers.Default)
    .collect { saveResult(it) }
```

---

## 7. Dispatcher 선택 가이드 (flowOn 기준)

| 작업 유형 | 적합한 Dispatcher |
|----------|------------------|
| 네트워크 요청 | `Dispatchers.IO` |
| 데이터베이스 읽기/쓰기 | `Dispatchers.IO` |
| 파일 읽기/쓰기 | `Dispatchers.IO` |
| JSON 파싱 (소규모) | `Dispatchers.Default` |
| 이미지 처리, 암호화 | `Dispatchers.Default` |
| UI 업데이트 | `Dispatchers.Main` |
| collect 이후 (다운스트림) | collect 호출 스레드 |

---

## 8. 테스트에서 flowOn

```kotlin
// 테스트 시 Dispatcher 주입으로 flowOn 교체
class UserRepository(
    private val dao: UserDao,
    private val ioDispatcher: CoroutineDispatcher = Dispatchers.IO
) {
    fun getUsersFlow(): Flow<List<User>> =
        dao.getUsersFlow()
            .map { it.map { entity -> entity.toUser() } }
            .flowOn(ioDispatcher)  // 테스트에서 교체 가능
}

// 테스트 코드
@Test
fun `users flow test`() = runTest {
    val testDispatcher = UnconfinedTestDispatcher()
    val repo = UserRepository(fakeDao, ioDispatcher = testDispatcher)

    repo.getUsersFlow().first()  // 테스트 디스패처에서 실행
}
```

---

## 9. 정리

- `flowOn(Dispatcher)`은 **자신 앞의 업스트림 연산자 전체**를 해당 스레드에서 실행
- 다운스트림(collect 이후)은 collect를 호출한 코루틴의 스레드에서 실행
- 여러 번 사용 가능: 각 구간별로 다른 스레드 지정 가능
- Repository 레이어에서 `flowOn(IO)` 처리 → ViewModel/UI는 스레드 신경 불필요
- `withContext`는 emit 블록 안에서 직접 사용 시 예외 발생 가능 → `flowOn` 선호
