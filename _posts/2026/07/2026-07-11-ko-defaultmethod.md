---
title: (Kotlin/코틀린) 인터페이스 default method와 다중 상속 해결
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin 인터페이스의 default method 구현과 여러 인터페이스가 동일 메서드를 제공할 때 발생하는 다중 상속 충돌 해결 방법을 정리합니다.
---

---

## 1. 인터페이스에 구현(default method) 추가

Kotlin 인터페이스는 **추상 메서드뿐만 아니라 구현을 포함한 메서드**를 정의할 수 있습니다.

```kotlin
interface Drawable {
    fun draw()  // 추상 메서드 — 반드시 구현 필요

    fun description(): String = "Drawable 객체"  // default 구현
    fun drawWithBorder() {                        // default 구현
        println("=====")
        draw()
        println("=====")
    }
}

class Circle : Drawable {
    override fun draw() = println("원 그리기")
    // description()과 drawWithBorder()는 오버라이드 선택 사항
}

val circle = Circle()
circle.draw()            // 원 그리기
circle.drawWithBorder()  // ===== \n 원 그리기 \n =====
println(circle.description())  // Drawable 객체
```

---

## 2. 인터페이스 프로퍼티 default

인터페이스는 추상 프로퍼티와 구현이 있는 프로퍼티 모두 가질 수 있습니다.  
단, **backing field(저장 공간)를 가질 수 없어** getter만 제공 가능합니다.

```kotlin
interface Named {
    val name: String          // 추상 — 구현 클래스에서 정의
    val displayName: String   // default getter
        get() = "[$name]"
}

class User(override val name: String) : Named
val user = User("Alice")
println(user.displayName)  // [Alice]
```

---

## 3. 다중 인터페이스 구현

```kotlin
interface Flyable {
    fun move() = println("하늘을 납니다")
}

interface Swimmable {
    fun move() = println("물속을 헤엄칩니다")
}

// 두 인터페이스 모두 move()를 구현 → 충돌!
class Duck : Flyable, Swimmable {
    override fun move() {
        // 어떤 구현을 사용할지 명시적으로 선택
        super<Flyable>.move()
        super<Swimmable>.move()
    }
}

Duck().move()
// 하늘을 납니다
// 물속을 헤엄칩니다
```

`super<인터페이스이름>.메서드()` 문법으로 어느 인터페이스의 구현을 사용할지 지정합니다.

---

## 4. 다중 상속 규칙

| 상황 | 처리 방법 |
|------|----------|
| 인터페이스 A만 구현 제공 | 자동으로 A의 구현 사용 |
| 인터페이스 A, B 모두 구현 제공 | **반드시** 오버라이드하고 `super<A>` 또는 `super<B>` 선택 |
| 클래스 상속 + 인터페이스 충돌 | 클래스 구현이 우선 (Diamond Problem 해결) |

```kotlin
open class Animal {
    open fun sound() = println("...")
}

interface Cat {
    fun sound() = println("야옹")
}

class Kitten : Animal(), Cat {
    // Animal의 sound()가 인터페이스보다 우선
    // 별도 오버라이드 없어도 Animal().sound() 호출
}

Kitten().sound()  // ... (Animal 우선)

// 인터페이스를 우선하려면 명시적 오버라이드
class Cat2 : Animal(), Cat {
    override fun sound() = super<Cat>.sound()
}
Cat2().sound()  // 야옹
```

---

## 5. 인터페이스 vs 추상 클래스

| 구분 | 인터페이스 | 추상 클래스 |
|------|----------|------------|
| 다중 구현 | 가능 | 불가 (단일 상속) |
| 상태(backing field) | 불가 | 가능 |
| 생성자 | 불가 | 가능 |
| default 구현 | 가능 | 가능 |
| 주요 용도 | 행동 계약 | 공통 상태·구현 |

```kotlin
// 인터페이스: 행동 계약
interface Serializable {
    fun serialize(): String
    fun deserialize(data: String): Unit
}

// 추상 클래스: 공통 구현 + 상태
abstract class BaseViewModel {
    private val _isLoading = MutableStateFlow(false)  // backing field 가능
    val isLoading: StateFlow<Boolean> = _isLoading.asStateFlow()

    abstract fun loadData()
}
```

---

## 6. 실전 패턴 — 믹스인(Mixin) 스타일

인터페이스 default 메서드로 기능을 조합하는 믹스인 패턴입니다.

```kotlin
interface Loggable {
    val tag: String get() = this::class.simpleName ?: "Unknown"
    fun log(message: String) = println("[$tag] $message")
}

interface Retryable {
    val maxRetries: Int get() = 3
    suspend fun retry(action: suspend () -> Unit) {
        repeat(maxRetries) { attempt ->
            try { action(); return }
            catch (e: Exception) { log("재시도 ${attempt + 1}/$maxRetries") }
        }
    }
    fun log(message: String) = println(message)
}

class DataRepository : Loggable, Retryable {
    suspend fun fetchData() {
        log("데이터 로드 시작")
        retry { api.fetch() }
    }
}
```

---

## 7. 정리

- Kotlin 인터페이스는 **default 구현 메서드**를 가질 수 있음 (backing field는 불가)
- 두 인터페이스가 **동일 시그니처의 default 메서드를 가지면** 반드시 오버라이드 필요
- `super<인터페이스>.메서드()` 로 특정 인터페이스 구현 명시적 선택
- 클래스 상속과 충돌 시 **클래스 구현이 인터페이스보다 우선**
- 다중 인터페이스 구현으로 믹스인 패턴 구현 가능
