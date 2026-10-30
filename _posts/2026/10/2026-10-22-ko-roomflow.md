---
title: (Android/Room) Room + Flow — 실시간 DB 변경 감지
tags: [ Android, Room ]
style: fill
color: dark
description: Room의 Flow 반환 쿼리로 DB 변경을 실시간 관찰하는 방법과 무효화 추적 원리, distinctUntilChanged, StateFlow 변환, 주의점을 정리합니다.
---

---

## 1. Flow를 반환하는 DAO

쿼리 반환 타입을 `Flow`로 선언하면 **테이블이 변경될 때마다 새 결과를 자동으로 방출**합니다.

```kotlin
@Dao
interface TodoDao {
    @Query("SELECT * FROM todos ORDER BY createdAt DESC")
    fun observeAll(): Flow<List<Todo>>             // suspend가 아님: Flow는 즉시 반환

    @Query("SELECT * FROM todos WHERE id = :id")
    fun observeById(id: Long): Flow<Todo?>         // 없으면 null

    @Insert suspend fun insert(todo: Todo)
    @Update suspend fun update(todo: Todo)
    @Delete suspend fun delete(todo: Todo)
}
```

```
UI ←── Flow<List<Todo>> ←── Room
                              ▲
        insert/update/delete ─┘  (변경 시 쿼리를 다시 실행해 새 값 방출)
```

> 일회성 조회는 `suspend fun`, **계속 관찰**하려면 `Flow`를 사용합니다.

---

## 2. 동작 원리 — 테이블 무효화 추적

Room은 쿼리가 읽는 **테이블이 변경되면 그 테이블을 읽는 Flow의 쿼리를 다시 실행**합니다.

```
todos 테이블에 insert 발생
   → Room: "todos를 관찰하는 쿼리가 있다"
   → 쿼리 재실행 → 새 List<Todo> 방출
```

주의할 점은 **행 단위가 아니라 테이블 단위**로 감지한다는 것입니다. 관찰 중인 쿼리와 무관한 행이 바뀌어도 같은 테이블이면 재실행되고, 결과가 같아도 다시 방출될 수 있습니다.

---

## 3. 불필요한 재방출 제거 — distinctUntilChanged

```kotlin
fun observeTodo(id: Long): Flow<Todo?> =
    dao.observeById(id)
        .distinctUntilChanged()      // 결과가 이전과 같으면 방출하지 않음 (data class equals 비교)
```

다른 행이 변경될 때마다 UI가 다시 그려지는 것을 막습니다. data class의 `equals`가 정확해야 효과가 있습니다.

---

## 4. ViewModel에서 UI 상태로 — stateIn

```kotlin
@HiltViewModel
class TodoViewModel @Inject constructor(
    private val dao: TodoDao
) : ViewModel() {

    val uiState: StateFlow<TodoUiState> = dao.observeAll()
        .map { todos -> TodoUiState.Success(todos) as TodoUiState }
        .catch { emit(TodoUiState.Error(it.message)) }            // DB 오류도 상태로 처리
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5_000),      // 화면 회전 중에도 구독 유지
            initialValue = TodoUiState.Loading
        )

    fun add(title: String) = viewModelScope.launch {
        dao.insert(Todo(title = title))      // insert만 하면 Flow가 알아서 갱신됨
    }
}
```

UI 갱신 코드를 따로 쓰지 않아도 **DB가 곧 상태의 원천**이 됩니다 (단방향 데이터 흐름).

---

## 5. UI에서 수집

```kotlin
// Compose
val state by viewModel.uiState.collectAsStateWithLifecycle()

// View 시스템
viewLifecycleOwner.lifecycleScope.launch {
    viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.uiState.collect { render(it) }
    }
}
```

화면이 보이지 않을 때는 수집을 멈춰 불필요한 쿼리 실행을 막습니다.

---

## 6. 여러 Flow 조합

```kotlin
val summary: Flow<Summary> = combine(
    todoDao.observeAll(),
    settingsDao.observeSettings()
) { todos, settings ->
    Summary(
        total = todos.size,
        done = todos.count { it.done },
        showDone = settings.showDone
    )
}
```

어느 쪽 테이블이 바뀌어도 새 `Summary`가 방출됩니다.

---

## 7. 주의사항

```kotlin
// ❌ 큰 테이블 전체를 Flow로 관찰: 한 행만 바뀌어도 전체 리스트를 다시 읽어 방출
@Query("SELECT * FROM messages") fun observeAll(): Flow<List<Message>>

// ✅ 필요한 범위만 (조건, LIMIT) 또는 Paging 사용
@Query("SELECT * FROM messages WHERE roomId = :roomId ORDER BY id DESC LIMIT 50")
fun observeRecent(roomId: Long): Flow<List<Message>>
```

- 큰 목록은 **Paging3**(`PagingSource`)를 사용하는 것이 효율적입니다
- Flow는 DB 변경 시 **계속 방출**되므로 `first()`로 일회성만 필요하면 `suspend` 쿼리를 쓰세요
- 쿼리는 Room이 내부 IO 스레드에서 실행하므로 `flowOn(Dispatchers.IO)`를 따로 붙일 필요가 없습니다
- 컬럼 하나만 필요하다면 필요한 컬럼만 SELECT해 방출 데이터를 줄이세요

---

## 8. 정리

- DAO 반환 타입을 `Flow`로 하면 **테이블 변경 시 자동으로 새 값 방출**
- 감지 단위는 **테이블**이며, 불필요한 재방출은 `distinctUntilChanged()`로 제거
- ViewModel에서는 `stateIn(WhileSubscribed(5_000))`로 UI 상태 변환
- 쓰기만 하면 UI가 자동 갱신 → **DB가 단일 진실 공급원**
- 큰 데이터는 범위를 제한하거나 Paging 사용, 일회성 조회는 `suspend` 사용
