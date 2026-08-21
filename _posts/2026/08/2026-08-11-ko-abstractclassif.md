---
title: (Kotlin/코틀린) 추상 클래스 vs 인터페이스 심화 비교
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin의 추상 클래스(abstract class)와 인터페이스(interface)의 차이점, 상태 저장 여부, 생성자, 다중 구현 규칙과 언제 무엇을 선택해야 하는지 정리합니다.
---

---

## 1. 핵심 차이 한눈에 보기

| 구분 | abstract class | interface |
|------|---------------|-----------|
| 상속 수 | 1개만 가능 | 여러 개 구현 가능 |
| 생성자 | 가능 | 불가 |
| 상태(필드) | 가능 (초기화 가능) | 가능하지만 backing field 없음 |
| 추상 멤버 | 선택적 | 기본이 추상 (default 제공 가능) |
| 접근 제어자 | protected 가능 | public만 가능 (Kotlin 기준) |
| 인스턴스화 | 불가 | 불가 |

---

## 2. 상태(State) 보유 — 핵심 차이

```kotlin
// abstract class: 실제 상태 저장 가능
abstract class Counter {
    private var count = 0  // backing field 존재

    fun increment() { count++ }
    fun getCount() = count

    abstract fun onCountChanged(newCount: Int)
}

// interface: backing field 없음 (getter/setter만 정의)
interface Countable {
    val count: Int  // backing field 없음 — 구현 클래스가 제공
        get() = 0   // 이 기본값은 매번 0 반환 (상태 아님)
}

class MyCounter : Countable {
    override var count: Int = 0  // 구현 클래스에서 backing field 제공
}
```

---

## 3. 생성자 — abstract class만 가능

```kotlin
// abstract class: 생성자 파라미터로 초기 상태 주입
abstract class Animal(val name: String, val age: Int) {
    abstract fun speak(): String

    fun introduce() = "${name}(${age}살): ${speak()}"
}

class Dog(name: String, age: Int) : Animal(name, age) {
    override fun speak() = "멍멍!"
}

val dog = Dog("바둑이", 3)
println(dog.introduce())  // 바둑이(3살): 멍멍!

// interface: 생성자 없음 → 상태 주입 불가
interface Speakable {
    fun speak(): String
    // constructor(name: String)  // ❌ 컴파일 에러
}
```

---

## 4. 다중 구현 — interface의 강점

```kotlin
interface Flyable {
    fun fly() = println("날아갑니다")
}

interface Swimmable {
    fun swim() = println("헤엄칩니다")
}

interface Runnable {
    fun run() = println("달립니다")
}

// 여러 인터페이스 동시 구현
class Duck : Flyable, Swimmable, Runnable {
    // 필요한 것만 override
    override fun fly() = println("오리가 날아갑니다")
}

val duck = Duck()
duck.fly()   // 오리가 날아갑니다
duck.swim()  // 헤엄칩니다 (기본 구현 사용)
duck.run()   // 달립니다 (기본 구현 사용)
```

---

## 5. 다이아몬드 문제 해결

```kotlin
interface A {
    fun hello() = println("A")
}

interface B : A {
    override fun hello() = println("B")
}

interface C : A {
    override fun hello() = println("C")
}

class D : B, C {
    // B와 C 모두 hello()를 오버라이드 → 명시적 해결 필요
    override fun hello() {
        super<B>.hello()  // B의 구현 선택
        super<C>.hello()  // C의 구현도 호출 가능
    }
}

D().hello()
// B
// C
```

---

## 6. abstract class와 interface 함께 사용

```kotlin
// 공통 상태는 abstract class, 능력은 interface로 분리
interface Serializable {
    fun serialize(): String
}

interface Cacheable {
    val cacheKey: String
    val ttl: Long get() = 3600L
}

abstract class Entity(val id: Long, val createdAt: Long) {
    abstract val type: String

    fun isValid() = id > 0
}

// Entity의 상태 + 두 인터페이스의 능력 조합
class UserEntity(
    id: Long,
    createdAt: Long,
    val name: String
) : Entity(id, createdAt), Serializable, Cacheable {
    override val type = "user"
    override val cacheKey = "user:$id"

    override fun serialize() = """{"id":$id,"name":"$name"}"""
}
```

---

## 7. sealed class vs abstract class

```kotlin
// sealed class: 상속을 같은 파일/패키지로 제한 + when exhaustive
sealed class NetworkResult<out T> {
    data class Success<T>(val data: T) : NetworkResult<T>()
    data class Error(val code: Int, val message: String) : NetworkResult<Nothing>()
    object Loading : NetworkResult<Nothing>()
}

// 컴파일러가 else 없이 모든 케이스 강제 확인
fun handle(result: NetworkResult<String>) = when (result) {
    is NetworkResult.Success -> println(result.data)
    is NetworkResult.Error   -> println("${result.code}: ${result.message}")
    NetworkResult.Loading    -> println("로딩 중...")
}

// abstract class: 외부에서 자유롭게 확장 가능 (라이브러리 설계 시 유용)
abstract class BaseViewModel : ViewModel() {
    protected abstract fun onInit()
    // 사용하는 쪽에서 자유롭게 상속
}
```

---

## 8. 선택 가이드

```
공통 상태(필드)가 필요한가?
    ↓ Yes → abstract class

생성자 파라미터로 초기화가 필요한가?
    ↓ Yes → abstract class

여러 클래스에서 동시에 구현해야 하는가?
    ↓ Yes → interface

"is-a" 관계이고 단일 상속으로 충분한가?
    ↓ Yes → abstract class

"can-do" 능력을 나타내는가?
    ↓ Yes → interface (Flyable, Serializable, Comparable 등)

상속 계층을 제한하고 싶은가?
    ↓ Yes → sealed class (abstract class의 특수형)
```

---

## 9. 정리

- **`abstract class`**: 상태 보유, 생성자 가능, 단일 상속 — "is-a" 관계
- **`interface`**: backing field 없음, 생성자 없음, 다중 구현 — "can-do" 능력
- 공통 상태가 있으면 abstract class, 능력을 믹스인하려면 interface
- 두 가지를 조합해 `abstract class + interface`로 풍부한 설계 가능
- 상속 계층 제한이 필요하면 `sealed class` 선택
