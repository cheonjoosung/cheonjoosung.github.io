---
title: Kotlin by lazy VS lateinit var 차이점과 사용법 완벽 가이드
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin by lazy VS lateinit var 차이점과 사용법 완벽 가이드
---

## 개요

Kotlin은 초기화 지연(Delayed Initialization)을 지원하기 위해 
두 가지 강력한 키워드인 **by lazy**와 **lateinit var**를 제공합니다. 이 두 가지는 초기화를 지연시켜 메모리 효율성을 높이고, 
복잡한 초기화 로직을 간결하게 처리할 수 있도록 돕습니다.

이번 포스트에서는 by lazy와 lateinit var의 동작 원리, 차이점, 그리고 실용적인 사용 예제를 소개합니다.

---

## 1. by lazy

by lazy는 읽기 전용 프로퍼티의 초기화를 지연(Lazy Initialization)시키는 데 사용됩니다.
- 초기화는 해당 프로퍼티가 처음 호출될 때 한 번만 수행
- **스레드 안전(Thread-Safe)**한 기본 동작을 제공

#### 1.1 기본문법

```kotlin
val propertyName: Type by lazy {
  // 초기화 코드
}
```
- val: by lazy는 val로 선언된 읽기 전용 프로퍼티에서만 사용 가능합니다.
- 초기화 블록: 초기화 블록은 호출 시 실행됩니다.

#### 1.2 동작 원리
- 프로퍼티에 처음 접근할 때 초기화 코드가 실행됩니다.
- 이후에는 캐싱된 값을 반환하여 불필요한 초기화 작업을 방지합니다.

---

## 2. lateinit var

lateinit var는 읽기/쓰기 가능한 프로퍼티의 초기화를 나중에 설정할 수 있도록 합니다.
- 초기화를 지연시키되, 반드시 초기화되어야 할 프로퍼티에 사용합니다.
- 초기화 전에 접근하려고 하면 UninitializedPropertyAccessException이 발생합니다.

#### 2.1 기본문법

```kotlin
lateinit var propertyName: Type
```
- var: lateinit은 변경 가능한 var 프로퍼티에서만 사용할 수 있습니다.
- 초기화는 생성자 또는 다른 함수에서 나중에 수행됩니다.

#### 2.2 동작 원리
- lateinit으로 선언된 프로퍼티는 초기값 없이 선언됩니다
- 초기화되지 않은 상태에서 접근하려 하면 런타임 예외가 발생합니다.

---

## 3. "by lazy 와 lateinit var" 차이점


| 특징                      | `by lazy`                          | `lateinit var`                 |
|---------------------------|-------------------------------------|---------------------------------|
| **사용 대상**              | `val` (읽기 전용 프로퍼티)           | `var` (읽기/쓰기 가능 프로퍼티) |
| **초기화 시점**             | **첫 호출 시 자동 초기화**            | **명시적으로 초기화 필요**       |
| **스레드 안전성**           | 기본적으로 스레드 안전(Thread-Safe) | 스레드 안전 보장 안 됨          |
| **초기화 여부 확인**        | 불가능                              | `::propertyName.isInitialized` |
| **에러 발생 여부**          | 호출 전 초기화 코드는 실행되지 않음   | 초기화되지 않은 상태에서 접근 시 예외 발생 |
| **초기화 대상**             | 객체 또는 복잡한 연산 결과           | 외부에서 값을 할당해야 하는 프로퍼티 |


---

## 4. 실용적인 사용 예제

#### 4.1 by lazy 사용 예제: 무거운 연산 결과 지연 초기화

- 예제: 데이터베이스 연결

```kotlin
val databaseConnection: Database by lazy {
  println("Initializing database connection...")
  Database.connect("url", "username", "password")
}

fun main() {
  println("Before accessing databaseConnection")
  println(databaseConnection) // 여기서 초기화 발생
  println(databaseConnection) // 재사용
}
/*** 출력
 * Before accessing databaseConnection
 * Initializing database connection...
 * Database(url, username)
 * Database(url, username)
 */
```

활용 상황
- 초기화 비용이 큰 작업(데이터베이스 연결, 파일 읽기 등)
- 프로퍼티가 사용되지 않을 수도 있는 경우

#### 4.2 lateinit var 사용 예제: 의존성 주입(DI)

- 예제: Android View 바인딩

```kotlin
class MainActivity : AppCompatActivity() {
  lateinit var binding: ActivityMainBinding

  override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    binding = ActivityMainBinding.inflate(layoutInflater)
    setContentView(binding.root)

    binding.textView.text = "Hello, Kotlin!"
  }
}
```
활용 상황
- 초기화를 생성자에서 처리할 수 없는 경우(예: Android의 onCreate 라이프사이클)
- 외부에서 값을 주입해야 하는 경우(예: 의존성 주입).

#### 4.3 초기화 여부 확인 (lateinit)

- lateinit 프로퍼티의 초기화 여부를 확인하려면 isInitialized를 사용합니다.
- 예제: isInitialized 사용

```kotlin
class MyClass {
    lateinit var name: String

    fun printName() {
        if (::name.isInitialized) {
            println("Name is $name")
        } else {
            println("Name is not initialized")
        }
    }
}

fun main() {
    val obj = MyClass()
    obj.printName() // Name is not initialized
    obj.name = "Kotlin"
    obj.printName() // Name is Kotlin
}
```

#### 4.4 추가 활용 사례

- Unit Test에서 lateinit은 객체를 나중에 초기화하는 데 자주 사용됩니다.
- 테스트 환경에서 프로퍼티를 초기화하기 전에 필요한 로직이 있을 때 유용

```kotlin
class UserTest {
    private lateinit var user: User

    @Before
    fun setUp() {
        user = User("Alice", 25)
    }

    @Test
    fun testUserName() {
        assertEquals("Alice", user.name)
    }
}

```

---

## 5. 잘못된 사용 사례

#### 5.1 by lazy 잘못된 사용

- by lazy는 초기화 블록이 복잡하거나, 자주 변경되어야 하는 값에 사용하면 적합하지 않습니다.
- 잘못된 예 : 자주 변경되는 값

```kotlin
val randomValue: Int by lazy { (1..100).random() }
println(randomValue) // 항상 같은 값 반환
```
- 수정: var로 선언하여 매번 새 값을 생성.

#### 5.2 lateinit의 잘못된 사용

- lateinit 프로퍼티는 nullable 타입이나 기본 타입(Int, Double 등)에 사용할 수 없습니다.
- 잘못된 예: 기본타입

```kotlin
lateinit var age: Int // 컴파일 에러
var age: Int = 0
```
- 수정: 기본값을 제공하거나 nullable 타입으로 선언.


---

## 6. 결론: 언제 무엇을 사용할까?

| 상황                                    | 적합한 선택      |
|---------------------------------------|-----------------|
| 읽기 전용 값                           | `by lazy`       |
| 초기화 비용이 큰 작업                    | `by lazy`       |
| 라이프사이클 후에 초기화해야 하는 값       | `lateinit var`  |
| 외부에서 값을 할당해야 하는 경우           | `lateinit var`  |
| 스레드 안전성이 필요한 경우              | `by lazy`       |
| 초기화 여부를 동적으로 확인해야 하는 경우   | `lateinit var`  |

---