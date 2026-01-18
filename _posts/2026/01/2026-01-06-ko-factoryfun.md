---
title: (Kotlin/코틀린) 팩토리 함수는 무엇인가?
tags: [Android,Kotlin]
style: fill
color: dark
description: (Kotlin/코틀린) 팩토리 함수는 무엇인가? - 생성자를 숨기고 객체 생성을 통제하는 방법
---

## 개요

- 코틀린을 쓰다 보면 다음과 같은 요구가 자연스럽게 생깁니다.
  - 객체 생성 로직이 복잡해졌다
  - 생성 시 검증/가공이 필요하다
  - 생성 방법을 여러 개 제공하고 싶다
  - `new` 같은 생성자 호출을 숨기고 싶다
- 이때 가장 코틀린스러운 해법이 바로 팩토리 함수(Factory Function) 입니다.
- 요약
  - 팩토리 함수 = 객체 생성을 담당하는 함수
  - 생성자를 직접 호출하지 않게 만들 수 있음
  - 이름으로 의도를 표현 가능
  - 캐싱 / 검증 / 조건 분기 / 서브타입 반환에 유리
  - 보통 `companion object` 또는 top-level 함수로 구현


---

## 1. 팩토리 함수란?

객체를 생성해서 반환하는 함수(반드시 클래스 내부일 필요는 없음)

```kotlin
fun createUser(name: String): User {
    return User(name)
}
```
이 자체가 이미 팩토리 함수이지만 코틀린에서는 이를 더 의미있고 의도적으로 사용합니다

---

## 2. 왜 생성자 대신 팩토리 함수를 쓰는가?

❌ 생성자만 사용할 때의 한계

```kotlin
class User(val name: String)
```

- 생성자 이름은 항상 클래스명
- 생성 로직을 숨길 수 없음
- 생성 실패/검증 표현이 애매함
- 캐싱/재사용이 어려움

---

## 3. 팩토리 함수의 장점

| 장점             | 설명                                   |
| -------------- | ------------------------------------ |
| 이름 부여          | `from()`, `of()`, `create()` 등 의도 표현 |
| 생성 로직 은닉       | 생성자 private 가능                       |
| 검증/가공          | 입력값 검사, 변환                           |
| 캐싱 가능          | 같은 인스턴스 재사용                          |
| 서브타입 반환        | 인터페이스/추상 타입 반환                       |
| null/Result 반환 | 실패 표현이 명확                            |

---

## 4. 가장 기본적인 패턴: companion object 팩토리

```kotlin
// 생성자를 숨기고 팩토리만 노출
class User private constructor(
    val id: Int,
    val name: String
) {
    companion object {
        fun create(id: Int, name: String): User {
            require(name.isNotBlank())
            return User(id, name)
        }
    }
}

// 호출
val user = User.create(1, "Alice")
```

---

## 5. 네이밍이 핵심이다 (팩토리 함수의 미학)

- 팩토리 함수는 이름이 곧 문서입니다.
- 자주 쓰는 네이밍 컨벤션

```text
from()      // 다른 타입에서 변환
of()        // 여러 파라미터를 조합
create()   // 일반적인 생성
newInstance() // Android에서 흔함
parse()    // 문자열/외부 데이터 파싱

User.from(entity)
Color.of(255, 0, 0)
Token.parse(jwt)
```

---

## 6. 조건에 따라 다른 객체를 반환하는 팩토리

```kotlin
// 인터페이스 + 구현체 숨기기
interface Logger {
    fun log(msg: String)
}

class DebugLogger : Logger {
    override fun log(msg: String) = println("DEBUG: $msg")
}

class ReleaseLogger : Logger {
    override fun log(msg: String) {}
}

object LoggerFactory {
    fun create(debug: Boolean): Logger {
        return if (debug) DebugLogger() else ReleaseLogger()
    }
}
```