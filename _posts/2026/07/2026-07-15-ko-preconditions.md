---
title: (Kotlin/코틀린) preconditions — require, check, assert 차이
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin의 precondition 함수인 require, check, assert의 차이와 올바른 사용 시점, error와의 관계를 정리합니다.
---

---

## 1. Precondition 함수 한눈에 비교

| 함수 | 검증 대상 | 실패 시 예외 | 비활성화 |
|------|----------|------------|---------|
| `require(condition)` | **함수 인자** | `IllegalArgumentException` | 불가 |
| `check(condition)` | **객체 상태** | `IllegalStateException` | 불가 |
| `assert(condition)` | **내부 로직 디버깅** | `AssertionError` | JVM `-ea` 플래그로 비활성화 가능 |
| `error(message)` | **도달 불가 분기** | `IllegalStateException` | 불가 |

---

## 2. require — 인자 검증

**호출자가 잘못된 값을 넘겼을 때** 사용합니다. 조건이 `false`면 `IllegalArgumentException`.

```kotlin
fun createUser(name: String, age: Int) {
    require(name.isNotBlank()) { "이름은 빈 문자열이 될 수 없습니다" }
    require(age in 0..150) { "나이는 0~150 사이여야 합니다. 입력값: $age" }
    // ...
}

createUser("", 25)   // IllegalArgumentException: 이름은 빈 문자열이 될 수 없습니다
createUser("Alice", 200)  // IllegalArgumentException: 나이는 0~150 사이여야 합니다. 입력값: 200
```

```kotlin
// 람다 메시지: 실패할 때만 평가 → 문자열 생성 비용 절약
require(items.size <= MAX_SIZE) {
    "최대 ${MAX_SIZE}개까지 가능합니다. 현재: ${items.size}개"
}
```

---

## 3. check — 상태 검증

**객체의 현재 상태가 올바르지 않을 때** 사용합니다. 조건이 `false`면 `IllegalStateException`.

```kotlin
class FileWriter(private val path: String) {
    private var isOpen = false

    fun open() { isOpen = true }

    fun write(content: String) {
        check(isOpen) { "파일이 열려있지 않습니다. open()을 먼저 호출하세요" }
        // ...
    }

    fun close() {
        check(isOpen) { "이미 닫혀 있습니다" }
        isOpen = false
    }
}
```

```kotlin
// require vs check 사용 시점 구분
fun processOrder(order: Order?, userId: String) {
    requireNotNull(order) { "주문 정보 누락" }        // require: 잘못된 인자
    require(order.items.isNotEmpty()) { "주문 항목 없음" }  // require: 잘못된 인자

    check(userSession.isLoggedIn) { "로그인 필요" }   // check: 잘못된 상태
    check(paymentService.isAvailable) { "결제 서비스 불가" }  // check: 잘못된 상태
}
```

---

## 4. assert — 디버깅 전용

**내부 로직의 불변식(invariant)을 검증**할 때 사용합니다.  
JVM 실행 시 `-ea`(enable assertions) 플래그 없이는 **실행되지 않습니다.**

```kotlin
fun binarySearch(list: List<Int>, target: Int): Int {
    assert(list == list.sorted()) { "이진 탐색은 정렬된 리스트에서만 동작합니다" }
    // 실제 탐색 로직...
}
```

```bash
# 기본 실행: assert 검사 안 함
java -jar app.jar

# 디버그 실행: assert 검사 활성화
java -ea -jar app.jar
```

### assert를 거의 사용하지 않는 이유

```kotlin
// Kotlin/Android에서 assert는 사실상 거의 쓰지 않음
// 이유 1: -ea 없으면 무시됨 → 프로덕션에서 효과 없음
// 이유 2: require/check가 더 명확한 의도를 전달
// 이유 3: 단위 테스트(JUnit)가 디버깅 역할을 대체

// 실전에서는 require/check를 쓰고 테스트로 검증
```

---

## 5. error — 실행 불가 분기

조건 없이 **항상 예외를 던집니다.** 코드 흐름상 도달하면 안 되는 위치에 사용합니다.

```kotlin
fun getDiscount(memberType: MemberType): Double = when (memberType) {
    MemberType.BRONZE   -> 0.05
    MemberType.SILVER   -> 0.10
    MemberType.GOLD     -> 0.15
    MemberType.PLATINUM -> 0.20
    else -> error("처리되지 않은 회원 등급: $memberType")
}
```

`Nothing` 타입을 반환하므로 컴파일러가 이후 코드를 unreachable로 처리합니다.

```kotlin
// error()의 반환 타입은 Nothing → 타입 추론에 활용
val value: String = possiblyNull ?: error("값이 없습니다")
// ?: error() 패턴은 requireNotNull과 동일한 효과
```

---

## 6. 실전 — 메시지 작성 가이드

좋은 에러 메시지는 **무엇이 잘못됐는지 + 올바른 값/상태가 무엇인지**를 포함해야 합니다.

```kotlin
// 나쁜 메시지
require(age > 0) { "나이 오류" }

// 좋은 메시지: 입력값과 기대값 명시
require(age > 0) { "나이는 양수여야 합니다. 입력값: $age" }
require(password.length >= 8) {
    "비밀번호는 최소 8자 이상이어야 합니다. 현재 ${password.length}자"
}
check(connectionPool.available > 0) {
    "DB 커넥션 풀 고갈 (최대: ${connectionPool.maxSize})"
}
```

---

## 7. 정리

- **`require`**: 잘못된 **인자** → `IllegalArgumentException`, 항상 활성
- **`check`**: 잘못된 **상태** → `IllegalStateException`, 항상 활성
- **`assert`**: 내부 **불변식 디버깅** → `AssertionError`, `-ea` 없으면 비활성
- **`error`**: 도달 불가 **분기** → `IllegalStateException`, 반환 타입 `Nothing`
- 실전에서는 `assert` 대신 `require`/`check` + 단위 테스트 조합 권장
