---
title: (Kotlin/코틀린) null 처리 전략 — requireNotNull, checkNotNull, error
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin의 requireNotNull, checkNotNull, error, 엘비스 연산자 등 null 처리 전략을 비교하고 상황별 올바른 선택 기준을 정리합니다.
---

---

## 1. Kotlin의 null 처리 도구 한눈에 보기

| 함수/연산자 | 용도 | 실패 시 예외 |
|------------|------|------------|
| `?.` | null이면 null 반환 | 없음 |
| `?:` | null이면 대체 값 | 없음 |
| `!!` | null이면 NPE | `NullPointerException` |
| `requireNotNull` | **입력 검증** (파라미터) | `IllegalArgumentException` |
| `checkNotNull` | **상태 검증** (현재 상태) | `IllegalStateException` |
| `error` | **도달 불가 코드** 표시 | `IllegalStateException` |

---

## 2. requireNotNull — 파라미터 검증

함수에 전달된 인자가 null이 아님을 보장합니다. `null`이면 `IllegalArgumentException`을 던집니다.

```kotlin
fun processUser(user: User?) {
    val validUser = requireNotNull(user) { "user는 null이 될 수 없습니다" }
    // 이 시점부터 validUser는 non-null
    println(validUser.name)
}

processUser(null)  // IllegalArgumentException: user는 null이 될 수 없습니다
```

```kotlin
// 스마트 캐스트 활용
fun sendEmail(email: String?) {
    requireNotNull(email) { "이메일 주소 필수" }
    // email은 여기서 String으로 스마트 캐스트됨
    emailService.send(email)
}
```

---

## 3. checkNotNull — 상태 검증

현재 객체/상태가 올바른지 검증합니다. `null`이면 `IllegalStateException`을 던집니다.

```kotlin
class DatabaseConnection {
    private var connection: Connection? = null

    fun connect(url: String) {
        connection = DriverManager.getConnection(url)
    }

    fun query(sql: String): ResultSet {
        val conn = checkNotNull(connection) { "먼저 connect()를 호출하세요" }
        return conn.createStatement().executeQuery(sql)
    }
}
```

`require`는 잘못된 인자, `check`는 잘못된 상태를 나타냅니다.

---

## 4. error — 도달 불가 코드

코드 흐름상 **절대 실행되어서는 안 되는 위치**에 표시합니다. 항상 `IllegalStateException`을 던집니다.

```kotlin
fun getStatusMessage(status: Status): String = when (status) {
    Status.ACTIVE   -> "활성"
    Status.INACTIVE -> "비활성"
    Status.DELETED  -> "삭제됨"
    // 새 Status 추가 시 컴파일 에러 방지를 위해 else 대신 error 사용
    else -> error("알 수 없는 상태: $status")
}
```

```kotlin
// sealed class와 함께 — 새 타입 추가 시 컴파일러가 감지
sealed interface Result<T>
data class Success<T>(val data: T) : Result<T>
data class Failure<T>(val error: Exception) : Result<T>

fun <T> handleResult(result: Result<T>) = when (result) {
    is Success -> processData(result.data)
    is Failure -> handleError(result.error)
    // else 불필요 — sealed이므로 컴파일러가 완전성 보장
}
```

---

## 5. ?: (엘비스 연산자) 패턴

null일 때 대체 값을 제공하거나 early return/throw 할 때 사용합니다.

```kotlin
// 대체 값
val name = user?.name ?: "이름 없음"

// early return
fun process(data: Data?) {
    val validData = data ?: return
    // validData는 non-null
}

// 예외 던지기
fun getUser(id: String): User {
    return userRepository.find(id)
        ?: throw NoSuchElementException("사용자 없음: $id")
}
```

---

## 6. !! 연산자 — 최후 수단

nullable 타입을 강제로 non-null로 변환합니다. null이면 `NullPointerException`.

```kotlin
val name: String? = "Alice"
val length = name!!.length  // 확실히 null이 아닐 때만 사용
```

**남용 금지.** `!!`는 "나는 이 값이 null이 아님을 확신한다"는 의미이며,  
틀리면 NPE가 발생해 Kotlin의 null safety 이점을 잃습니다.

```kotlin
// !! 대신 더 안전한 대안
val length = name?.length ?: 0           // 대체 값
val length = requireNotNull(name).length // 명시적 에러 메시지
```

---

## 7. require vs check vs error 선택 기준

```kotlin
fun createAccount(
    username: String?,
    password: String?,
    age: Int
) {
    // 1. 파라미터 검증 → require / requireNotNull
    requireNotNull(username) { "사용자명 필수" }
    requireNotNull(password) { "비밀번호 필수" }
    require(age >= 14) { "14세 이상만 가입 가능" }

    // 2. 내부 상태 검증 → check / checkNotNull
    check(isConnected) { "네트워크 연결 필요" }
    val session = checkNotNull(currentSession) { "세션이 없습니다" }

    // 3. 논리적으로 불가능한 상황 → error
    val role = when (age) {
        in 14..17 -> "TEEN"
        in 18..Int.MAX_VALUE -> "ADULT"
        else -> error("나이 검증을 통과했는데 범위 밖: $age")
    }
}
```

---

## 8. 정리

- **`requireNotNull`**: 함수 파라미터가 null이면 안 될 때 → `IllegalArgumentException`
- **`checkNotNull`**: 현재 객체 상태가 null이면 안 될 때 → `IllegalStateException`
- **`error`**: 도달 불가능한 코드에 표시 → `IllegalStateException`
- **`?:`**: null일 때 대체 값, early return, throw 패턴으로 활용
- **`!!`**: 확실히 null이 아닌 경우에만 최후 수단으로 사용, 남용 금지
