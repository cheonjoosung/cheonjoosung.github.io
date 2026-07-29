---
title: (Kotlin/코틀린) 위임 패턴 — by keyword로 interface delegation
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin의 by 키워드로 인터페이스 위임을 구현하는 방법과 상속 대신 위임을 선택해야 하는 이유, 실전 활용 패턴을 정리합니다.
---

---

## 1. 위임(Delegation)이란?

**위임**은 객체의 기능 일부를 다른 객체에 넘기는 패턴입니다.  
상속(`extends`) 대신 **구성(Composition)**으로 코드를 재사용합니다.

```kotlin
// 상속: "is-a" 관계
class Stack<T> : ArrayList<T>()  // ArrayList의 모든 메서드가 노출됨 (의도치 않은 API 포함)

// 위임: "has-a" 관계
class Stack<T>(private val list: ArrayList<T> = ArrayList()) {
    fun push(item: T) = list.add(item)
    fun pop(): T = list.removeLast()
    fun peek(): T = list.last()
    // add(), set(), remove() 등 불필요한 메서드가 노출되지 않음
}
```

---

## 2. by 키워드 — 인터페이스 위임

Kotlin의 `by` 키워드는 **보일러플레이트 없이** 인터페이스 위임을 자동으로 처리합니다.

```kotlin
interface Printer {
    fun print(text: String)
    fun printLine(text: String) = println("[$text]")
}

class ConsolePrinter : Printer {
    override fun print(text: String) = println(text)
}

// by 없이 수동 위임: 반복적인 위임 코드 필요
class Document(private val printer: ConsolePrinter) : Printer {
    override fun print(text: String) = printer.print(text)
    override fun printLine(text: String) = printer.printLine(text)
}

// by 키워드로 자동 위임: 컴파일러가 위임 코드 생성
class Document(printer: ConsolePrinter) : Printer by printer
```

`by printer`는 `Printer` 인터페이스의 모든 메서드를 `printer` 객체에 자동으로 위임합니다.

---

## 3. 일부만 오버라이드

위임한 인터페이스의 메서드 중 일부만 재정의할 수 있습니다.

```kotlin
interface Logger {
    fun log(message: String)
    fun warn(message: String)
    fun error(message: String)
}

class SimpleLogger : Logger {
    override fun log(message: String) = println("[LOG] $message")
    override fun warn(message: String) = println("[WARN] $message")
    override fun error(message: String) = println("[ERROR] $message")
}

// log만 재정의, warn/error는 SimpleLogger에 위임
class TimestampLogger(logger: SimpleLogger) : Logger by logger {
    override fun log(message: String) {
        println("[${System.currentTimeMillis()}][LOG] $message")
    }
    // warn, error는 SimpleLogger.warn/error 가 호출됨
}

val logger = TimestampLogger(SimpleLogger())
logger.log("서버 시작")   // [1721390400000][LOG] 서버 시작
logger.warn("CPU 높음")  // [WARN] CPU 높음  (SimpleLogger에 위임)
```

---

## 4. 여러 인터페이스 동시 위임

```kotlin
interface Flyable {
    fun fly(): String
}

interface Swimmable {
    fun swim(): String
}

class Bird : Flyable {
    override fun fly() = "날개로 날아갑니다"
}

class Fish : Swimmable {
    override fun swim() = "지느러미로 헤엄칩니다"
}

// 여러 인터페이스를 각각 다른 객체에 위임
class Duck(
    bird: Bird = Bird(),
    fish: Fish = Fish()
) : Flyable by bird, Swimmable by fish

val duck = Duck()
println(duck.fly())   // 날개로 날아갑니다
println(duck.swim())  // 지느러미로 헤엄칩니다
```

---

## 5. 실전 패턴 — Android ViewModel 위임

```kotlin
interface LoadingDelegate {
    val isLoading: StateFlow<Boolean>
    fun setLoading(loading: Boolean)
}

class LoadingDelegateImpl : LoadingDelegate {
    private val _isLoading = MutableStateFlow(false)
    override val isLoading: StateFlow<Boolean> = _isLoading.asStateFlow()
    override fun setLoading(loading: Boolean) { _isLoading.value = loading }
}

interface ErrorDelegate {
    val error: StateFlow<String?>
    fun setError(message: String?)
}

class ErrorDelegateImpl : ErrorDelegate {
    private val _error = MutableStateFlow<String?>(null)
    override val error: StateFlow<String?> = _error.asStateFlow()
    override fun setError(message: String?) { _error.value = message }
}

// ViewModel: 두 위임 객체로 상태 관리 분리
class UserViewModel(
    private val repository: UserRepository,
    loadingDelegate: LoadingDelegateImpl = LoadingDelegateImpl(),
    errorDelegate: ErrorDelegateImpl = ErrorDelegateImpl()
) : ViewModel(),
    LoadingDelegate by loadingDelegate,
    ErrorDelegate by errorDelegate {

    fun loadUser(id: String) {
        viewModelScope.launch {
            setLoading(true)
            setError(null)
            try {
                val user = repository.getUser(id)
                _user.value = user
            } catch (e: Exception) {
                setError(e.message)
            } finally {
                setLoading(false)
            }
        }
    }
}
```

---

## 6. 실전 패턴 — Decorator 패턴

```kotlin
interface Cache<K, V> {
    fun get(key: K): V?
    fun put(key: K, value: V)
    fun remove(key: K)
}

class MemoryCache<K, V> : Cache<K, V> {
    private val map = mutableMapOf<K, V>()
    override fun get(key: K): V? = map[key]
    override fun put(key: K, value: V) { map[key] = value }
    override fun remove(key: K) { map.remove(key) }
}

// 로깅 기능을 위임으로 추가 (기존 구현 변경 없음)
class LoggingCache<K, V>(
    private val delegate: Cache<K, V>
) : Cache<K, V> by delegate {
    override fun put(key: K, value: V) {
        println("캐시 저장: $key")
        delegate.put(key, value)
    }

    override fun get(key: K): V? {
        val result = delegate.get(key)
        println("캐시 ${if (result != null) "히트" else "미스"}: $key")
        return result
    }
}

val cache = LoggingCache(MemoryCache<String, String>())
cache.put("user:1", "Alice")  // 캐시 저장: user:1
cache.get("user:1")           // 캐시 히트: user:1
cache.get("user:2")           // 캐시 미스: user:2
```

---

## 7. 상속 vs 위임 선택 기준

| 상황 | 선택 |
|------|------|
| "is-a" 관계 (Dog is an Animal) | 상속 |
| "has-a" 관계 (Car has an Engine) | 위임 |
| 슈퍼클래스 API 전체가 필요한 경우 | 상속 |
| 슈퍼클래스 API 일부만 노출하고 싶은 경우 | 위임 |
| 여러 클래스의 기능을 합쳐야 할 때 | 위임 (다중 위임) |
| 런타임에 구현체를 교체해야 할 때 | 위임 |

**"상속보다 합성을 선호하라(Favor composition over inheritance)"** — Effective Java

---

## 8. 정리

- `by` 키워드: 인터페이스의 모든 메서드를 지정 객체에 자동 위임
- 필요한 메서드만 `override`로 재정의, 나머지는 위임 객체가 처리
- 여러 인터페이스를 서로 다른 객체에 각각 위임 가능
- 상속의 한계(단일 상속, 불필요한 API 노출)를 위임으로 해결
- Android에서 ViewModel 기능 분리, Decorator 패턴 구현에 효과적
