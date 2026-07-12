---
title: (Kotlin/코틀린) operator fun invoke — 객체를 함수처럼 호출
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin의 operator fun invoke를 활용해 객체를 함수처럼 호출하는 패턴과 UseCase, 커맨드 패턴, 팩토리 패턴 실전 적용 사례를 정리합니다.
---

---

## 1. operator fun invoke란?

`invoke`는 Kotlin에서 **객체를 함수처럼 호출할 수 있게 해주는 연산자**입니다.  
`obj(args)` 형태의 호출이 내부적으로 `obj.invoke(args)`로 변환됩니다.

```kotlin
class Greeter {
    operator fun invoke(name: String): String {
        return "안녕하세요, $name!"
    }
}

val greeter = Greeter()
println(greeter("Alice"))          // 안녕하세요, Alice!
println(greeter.invoke("Alice"))   // 동일한 결과
```

---

## 2. 람다와의 관계

Kotlin의 람다와 함수 타입(`(A) -> B`)은 내부적으로 `invoke`를 가진 객체입니다.

```kotlin
val double: (Int) -> Int = { it * 2 }
println(double(5))         // 10
println(double.invoke(5))  // 10 — 동일
```

`operator fun invoke`를 정의하면 클래스 인스턴스가 람다처럼 동작합니다.

---

## 3. Android UseCase 패턴

Clean Architecture에서 UseCase를 `invoke`로 정의하면 호출 코드가 간결해집니다.

```kotlin
// invoke 없는 방식
class GetUserUseCase(private val repository: UserRepository) {
    fun execute(userId: String): User = repository.getUser(userId)
}

// invoke 방식
class GetUserUseCase(private val repository: UserRepository) {
    operator fun invoke(userId: String): User = repository.getUser(userId)
}
```

```kotlin
// ViewModel에서 호출 비교
class UserViewModel(private val getUser: GetUserUseCase) : ViewModel() {

    // invoke 없음: getUser.execute(id)
    fun loadUser(id: String) {
        val user = getUser.execute(id)
    }

    // invoke 있음: getUser(id) — 함수처럼 자연스럽게 호출
    fun loadUser(id: String) {
        val user = getUser(id)
    }
}
```

### suspend fun invoke — 코루틴 연동

```kotlin
class FetchProductsUseCase(
    private val repository: ProductRepository
) {
    suspend operator fun invoke(category: String): List<Product> =
        repository.getProducts(category)
}

// ViewModel에서
viewModelScope.launch {
    val products = fetchProducts("electronics")  // 함수처럼 호출
    _uiState.value = UiState.Success(products)
}
```

---

## 4. companion object invoke — 팩토리 패턴

`companion object`에 `invoke`를 정의하면 생성자처럼 보이지만 더 유연한 팩토리를 만들 수 있습니다.

```kotlin
class HttpClient private constructor(
    val baseUrl: String,
    val timeout: Int,
    val headers: Map<String, String>
) {
    companion object {
        operator fun invoke(
            baseUrl: String,
            timeout: Int = 30,
            headers: Map<String, String> = emptyMap()
        ): HttpClient {
            require(baseUrl.startsWith("http")) { "URL은 http로 시작해야 합니다" }
            return HttpClient(baseUrl, timeout, headers)
        }
    }
}

// 사용: 생성자처럼 보이지만 검증 로직이 내부에 있음
val client = HttpClient("https://api.example.com", timeout = 60)
```

---

## 5. 커맨드 패턴 구현

```kotlin
interface Command {
    operator fun invoke()
}

class SaveCommand(private val data: String, private val repo: Repository) : Command {
    override operator fun invoke() {
        repo.save(data)
        println("저장 완료: $data")
    }
}

class DeleteCommand(private val id: String, private val repo: Repository) : Command {
    override operator fun invoke() {
        repo.delete(id)
        println("삭제 완료: $id")
    }
}

// 커맨드 큐
val commands: List<Command> = listOf(
    SaveCommand("데이터1", repo),
    SaveCommand("데이터2", repo),
    DeleteCommand("old_data", repo)
)

commands.forEach { it() }  // 각 커맨드 실행
```

---

## 6. 상태 기반 invoke

```kotlin
class RateLimiter(private val maxCallsPerSecond: Int) {
    private var callCount = 0
    private var lastResetTime = System.currentTimeMillis()

    operator fun invoke(action: () -> Unit): Boolean {
        val now = System.currentTimeMillis()
        if (now - lastResetTime >= 1000) {
            callCount = 0
            lastResetTime = now
        }
        return if (callCount < maxCallsPerSecond) {
            callCount++
            action()
            true
        } else {
            false
        }
    }
}

val limiter = RateLimiter(3)

repeat(5) { i ->
    val executed = limiter { println("요청 $i 처리") }
    if (!executed) println("요청 $i 차단됨")
}
// 요청 0, 1, 2 처리
// 요청 3, 4 차단됨
```

---

## 7. operator invoke vs 일반 메서드 비교

| 구분 | 일반 메서드 | operator invoke |
|------|-----------|----------------|
| 호출 문법 | `obj.method(args)` | `obj(args)` |
| 람다 대체 | 어색함 | 자연스러움 |
| 고차 함수 전달 | 메서드 참조 필요 (`::method`) | 객체 자체 전달 가능 |
| 상태 보관 | 가능 | 가능 |
| 주요 용도 | 일반 객체 메서드 | UseCase, 커맨드, 팩토리 |

```kotlin
// 고차 함수에 전달 시 차이
class Validator {
    fun validate(s: String) = s.isNotEmpty()
    // operator fun invoke(s: String) = s.isNotEmpty()
}

val validator = Validator()

// 일반 메서드: 메서드 참조 필요
listOf("a", "", "b").filter(validator::validate)

// invoke: 객체 자체를 전달 가능
listOf("a", "", "b").filter(validator)  // operator invoke 있으면 가능
```

---

## 8. 정리

- `operator fun invoke`로 객체를 `obj(args)` 형태로 **함수처럼 호출** 가능
- Clean Architecture의 **UseCase** 패턴에서 가장 많이 활용 (`getUser(id)`)
- `companion object invoke`로 **팩토리 패턴** 구현 시 생성자처럼 읽히면서 검증 로직 포함 가능
- 커맨드 패턴, 전략 패턴 등에서 인터페이스를 더 간결하게 표현
- 상태를 가진 객체를 **람다처럼** 고차 함수에 전달할 수 있음
