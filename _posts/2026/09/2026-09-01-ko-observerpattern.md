---
title: (Kotlin/코틀린) 행동 패턴 — 옵저버 패턴 (Observer)
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin으로 옵저버 패턴을 구현하는 방법과 Flow, LiveData와의 관계, 실전 이벤트 시스템 구축 패턴을 정리합니다.
---

---

## 1. 옵저버 패턴이란?

**한 객체(Subject)의 상태 변화를 여러 객체(Observer)에게 자동으로 알리는** 패턴입니다.

```
Subject (발행자)
    ↓ 상태 변경
    ↓ notify()
Observer1, Observer2, Observer3 ... (구독자들)
```

---

## 2. 기본 구현

```kotlin
// 옵저버 인터페이스
fun interface Observer<T> {
    fun onChanged(value: T)
}

// Subject
class Subject<T>(initialValue: T) {
    private val observers = mutableListOf<Observer<T>>()
    private var value: T = initialValue
        set(new) {
            field = new
            notifyObservers(new)
        }

    fun subscribe(observer: Observer<T>) = observers.add(observer)
    fun unsubscribe(observer: Observer<T>) = observers.remove(observer)
    fun getValue() = value
    fun setValue(new: T) { value = new }

    private fun notifyObservers(value: T) =
        observers.forEach { it.onChanged(value) }
}

// 사용
val counter = Subject(0)

val logger = Observer<Int> { println("값 변경: $it") }
val alarm  = Observer<Int> { if (it > 5) println("⚠️ 임계값 초과: $it") }

counter.subscribe(logger)
counter.subscribe(alarm)

counter.setValue(3)   // 값 변경: 3
counter.setValue(7)   // 값 변경: 7 / ⚠️ 임계값 초과: 7

counter.unsubscribe(alarm)
counter.setValue(10)  // 값 변경: 10 (alarm 없음)
```

---

## 3. Delegates.observable로 간결하게

```kotlin
import kotlin.properties.Delegates

class StockPrice {
    var price: Double by Delegates.observable(0.0) { _, old, new ->
        listeners.forEach { it(old, new) }
    }

    private val listeners = mutableListOf<(Double, Double) -> Unit>()

    fun addListener(listener: (old: Double, new: Double) -> Unit) {
        listeners.add(listener)
    }
}

val stock = StockPrice()
stock.addListener { old, new ->
    val change = ((new - old) / old * 100)
    println("${if (change >= 0) "▲" else "▼"} ${"%.2f".format(Math.abs(change))}% ($old → $new)")
}

stock.price = 50000.0
stock.price = 52000.0  // ▲ 4.00% (50000.0 → 52000.0)
stock.price = 48000.0  // ▼ 7.69% (52000.0 → 48000.0)
```

---

## 4. Flow로 구현 — 현대적 방식

```kotlin
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow

class UserRepository {
    private val _currentUser = MutableStateFlow<User?>(null)
    val currentUser: StateFlow<User?> = _currentUser.asStateFlow()

    fun login(user: User) { _currentUser.value = user }
    fun logout() { _currentUser.value = null }
}

// 구독 (ViewModel)
class ProfileViewModel(private val repo: UserRepository) : ViewModel() {
    val user = repo.currentUser  // StateFlow를 그대로 노출

    init {
        viewModelScope.launch {
            repo.currentUser.collect { user ->
                println("사용자 변경: ${user?.name ?: "로그아웃"}")
            }
        }
    }
}
```

---

## 5. 실전 패턴 — EventBus

```kotlin
object EventBus {
    private val _events = MutableSharedFlow<Any>(extraBufferCapacity = 64)
    val events = _events.asSharedFlow()

    suspend fun publish(event: Any) = _events.emit(event)

    inline fun <reified T> CoroutineScope.subscribe(
        crossinline handler: suspend (T) -> Unit
    ) = launch {
        events.filterIsInstance<T>().collect { handler(it) }
    }
}

// 이벤트 정의
data class UserLoggedIn(val userId: String)
data class CartUpdated(val itemCount: Int)

// 구독
viewModelScope.subscribe<UserLoggedIn> { event ->
    println("로그인: ${event.userId}")
}

// 발행
viewModelScope.launch {
    EventBus.publish(UserLoggedIn("user-123"))
}
```

---

## 6. 정리

- 옵저버 패턴: Subject의 상태 변화를 Observer에게 자동 전파
- `Delegates.observable`: 프로퍼티 변경 감지에 가장 간단
- `StateFlow` / `SharedFlow`: Kotlin 현대적 옵저버, 코루틴과 완벽 통합
- 구독/해지 시 메모리 누수 주의 — `Lifecycle`과 함께 사용 권장
