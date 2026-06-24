---
title: (Kotlin/코틀린) Kotlin 제네릭 심화 — upper bound, where 절
tags: [ Kotlin, Android ]
style: fill
color: dark
description: Kotlin 제네릭의 upper bound 제약, 다중 제약을 위한 where 절, variance(out/in)와의 관계를 Android 실전 예제로 깊이 정리합니다.
---

## 개요

- Kotlin 제네릭의 **upper bound 제약**과 **`where` 절**을 다룹니다.
- 이 글에서는 다음을 설명합니다.
  - upper bound가 필요한 이유
  - 단일 제약 vs 다중 제약(`where`)
  - upper bound와 variance(`out`/`in`)의 관계
  - Android 실전 예제

---

## 1. upper bound — 타입 매개변수 제약

```kotlin
// 제약 없는 제네릭 — Any? 취급, 멤버 접근 불가
fun <T> printSize(item: T) {
    // println(item.size) // ❌ T에 size가 있다는 보장이 없음
}

// upper bound — T는 Comparable<T>를 구현해야 함
fun <T : Comparable<T>> max(a: T, b: T): T {
    return if (a > b) a else b  // compareTo 사용 가능
}

max(3, 5)        // Int는 Comparable<Int> 구현
max("a", "b")    // String도 Comparable<String> 구현
```

```kotlin
// 클래스 상속 제약
abstract class Animal(val name: String) {
    abstract fun sound(): String
}

class Dog(name: String) : Animal(name) {
    override fun sound() = "멍멍"
}

// T는 Animal의 하위 타입이어야 함
fun <T : Animal> printSound(animal: T) {
    println("${animal.name}: ${animal.sound()}")
}
```

---

## 2. 기본 upper bound — Any?

```kotlin
// 명시하지 않으면 기본 upper bound는 Any?
fun <T> identity(value: T): T = value

// 위 코드는 사실상 다음과 동일
fun <T : Any?> identity(value: T): T = value

// null을 허용하지 않으려면 Any로 제약
fun <T : Any> identityNonNull(value: T): T = value
identityNonNull(null)  // ❌ 컴파일 에러
```

---

## 3. where 절 — 다중 제약

타입 매개변수가 **여러 제약**을 동시에 만족해야 할 때 `where` 절을 사용합니다.

```kotlin
// ❌ 여러 인터페이스를 동시에 만족시키는 단축 문법은 없음
// fun <T : Comparable<T>, Cloneable> sort(item: T) {}  // 불가능한 표현

// ✅ where 절로 다중 제약 표현
fun <T> processItem(item: T) where T : Comparable<T>, T : Cloneable {
    // T는 Comparable과 Cloneable을 모두 구현해야 함
}
```

```kotlin
interface Identifiable {
    val id: String
}

interface Timestamped {
    val createdAt: Long
}

// T는 Identifiable이면서 Timestamped여야 함
fun <T> logEntry(item: T): String where T : Identifiable, T : Timestamped {
    return "[${item.id}] ${item.createdAt}"
}

data class Post(
    override val id: String,
    override val createdAt: Long,
    val content: String
) : Identifiable, Timestamped

logEntry(Post(id = "1", createdAt = 1000L, content = "hello"))
```

```kotlin
// 클래스 + 인터페이스 조합도 가능
abstract class BaseEntity { abstract val id: String }
interface Syncable { fun sync() }

fun <T> syncEntity(entity: T) where T : BaseEntity, T : Syncable {
    println("동기화: ${entity.id}")
    entity.sync()
}
```

---

## 4. upper bound와 variance(out/in)

```kotlin
// out — 생산자(읽기 전용) 위치, upper bound와 함께 자주 사용
class Container<out T : Animal>(private val item: T) {
    fun get(): T = item
    // fun set(value: T) {}  // ❌ out 위치이므로 입력 매개변수 불가
}

val dogContainer: Container<Dog> = Container(Dog("바둑이"))
val animalContainer: Container<Animal> = dogContainer  // ✅ 업캐스팅 가능 (covariant)
```

```kotlin
// in — 소비자(쓰기 전용) 위치
class AnimalProcessor<in T : Animal> {
    fun process(item: T) {
        println("${item.name} 처리 중")
    }
}

val animalProcessor: AnimalProcessor<Animal> = AnimalProcessor()
val dogProcessor: AnimalProcessor<Dog> = animalProcessor  // ✅ 다운캐스팅 가능 (contravariant)
```

| 키워드 | 의미 | upper bound 결합 시 |
|--------|------|---------------------|
| `out T : Bound` | T를 반환만 함 (생산자) | `Container<Dog>` → `Container<Animal>` 대입 가능 |
| `in T : Bound` | T를 매개변수로만 받음 (소비자) | `Processor<Animal>` → `Processor<Dog>` 대입 가능 |
| (없음) `T : Bound` | 양방향 사용 | 무공변(invariant) |

---

## 5. Android 실전 예제

### Repository 공통 인터페이스 제약

```kotlin
interface Entity {
    val id: Long
}

interface BaseRepository<T : Entity> {
    suspend fun getById(id: Long): T?
    suspend fun save(item: T)
    suspend fun delete(id: Long)
}

data class User(override val id: Long, val name: String) : Entity
data class Post(override val id: Long, val title: String) : Entity

class UserRepository : BaseRepository<User> {
    override suspend fun getById(id: Long): User? = /* ... */ null
    override suspend fun save(item: User) { /* ... */ }
    override suspend fun delete(id: Long) { /* ... */ }
}
```

### ViewModel 팩토리에서 다중 제약

```kotlin
interface UiState
interface Resettable { fun reset() }

// State는 UiState이면서 Resettable이어야 함
class BaseViewModel<S> (initialState: S) : ViewModel() where S : UiState, S : Resettable {

    private val _state = MutableStateFlow(initialState)
    val state: StateFlow<S> = _state.asStateFlow()

    fun resetState() {
        _state.value.reset()
    }
}

data class HomeUiState(
    var isLoading: Boolean = false
) : UiState, Resettable {
    override fun reset() { isLoading = false }
}

class HomeViewModel : BaseViewModel<HomeUiState>(HomeUiState())
```

### RecyclerView Adapter 제약

```kotlin
interface ListItem {
    val itemId: Long
}

abstract class BaseAdapter<T : ListItem, VH : RecyclerView.ViewHolder> :
    RecyclerView.Adapter<VH>() {

    protected var items: List<T> = emptyList()

    fun submitItems(newItems: List<T>) {
        items = newItems
        notifyDataSetChanged()
    }

    override fun getItemId(position: Int): Long = items[position].itemId
}

data class UserItem(override val itemId: Long, val name: String) : ListItem

class UserAdapter : BaseAdapter<UserItem, UserViewHolder>() {
    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): UserViewHolder =
        UserViewHolder(/* ... */)

    override fun onBindViewHolder(holder: UserViewHolder, position: Int) {
        holder.bind(items[position])
    }

    override fun getItemCount() = items.size
}
```

---

## 6. 정리

| 항목 | 내용 |
|------|------|
| upper bound | `<T : Bound>` — T가 따라야 할 타입 제약, 멤버 접근 가능하게 함 |
| 기본 upper bound | 명시 없으면 `Any?` |
| `where` 절 | 다중 제약(클래스+인터페이스 등)을 동시에 적용 |
| variance 결합 | `out`은 반환(생산), `in`은 매개변수(소비) 위치 제약과 함께 사용 |
| Android 활용 | Repository/ViewModel/Adapter 공통 기반 클래스 설계 |

- upper bound는 **"제네릭이 무엇을 할 수 있는지"** 를 컴파일러에게 알려주는 계약입니다.
- 제약이 여러 개 필요하면 `where` 절로 명시적으로 표현합니다.

---

## 참고

- [Kotlin 공식 문서 — Generics](https://kotlinlang.org/docs/generics.html)
- [crossinline vs noinline 포스팅 보기](/posts/2026-06-15-ko-crossinline-noinline)
