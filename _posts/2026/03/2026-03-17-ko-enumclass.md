---
title: (Kotlin/코틀린) enum class 완전 정리
tags: [ Android,Kotlin ]
style: fill
color: dark
description: (Kotlin/코틀린) enum class의 기본 개념부터 property, method, when 활용, Gson/JSON 변환, Android 실무 활용 패턴까지 정리합니다.
---

## 개요

- Kotlin의 enum class는 고정된 상수 집합을 타입으로 표현할 때 사용하는 기능입니다.
- ```kotlin
  enum class Status {
    READY,
    RUNNING,
    DONE
  }
  ```
- 단순 상수처럼 보이지만, 실제로는 다음과 같은 특징을 가집니다.
  - 각각이 객체 (instance)
  - 프로퍼티 / 함수 포함 가능
  - when과 함께 강력한 타입 안정성 제공

---

## 1. enum class 기본 사용

```kotlin
enum class Status {
    READY,
    RUNNING,
    DONE
}

val status = Status.READY
```

---

## 2. enum은 사실 객체다

각 enum 값은 단순 값이 아니라 객체입니다.

```kotlin
println(Status.READY.name)   // READY
println(Status.READY.ordinal) // 0
```

| 프로퍼티    | 설명          |
| ------- | ----------- |
| name    | enum 이름     |
| ordinal | 순서 (0부터 시작) |

---

### 3. enum에 프로퍼티 추가

```kotlin
enum class Status(val code: Int) {
    READY(0),
    RUNNING(1),
    DONE(2)
}

println(Status.READY.code) // 0
```

---

## 4. enum에 함수 추가

```kotlin
enum class Status {
    READY,
    RUNNING,
    DONE;

    fun isFinished(): Boolean {
        return this == DONE
    }
}
```

---

## 5. enum별로 다른 로직 정의

```kotlin
enum class Operation {
    ADD {
        override fun apply(a: Int, b: Int) = a + b
    },
    SUB {
        override fun apply(a: Int, b: Int) = a - b
    };

    abstract fun apply(a: Int, b: Int): Int
}
```

---

## 6. when 과 함께 사용하는 것이 핵심

```kotlin
fun handle(status: Status) {
    when (status) {
        Status.READY -> {}
        Status.RUNNING -> {}
        Status.DONE -> {}
    }
}
```

- enum은 모든 경우를 컴파일러가 체크

---

## 7. enum vs sealed class

| 항목     | enum  | sealed class |
| ------ | ----- | ------------ |
| 상태 개수  | 고정    | 확장 가능        |
| 데이터 포함 | 제한적   | 자유           |
| 타입 계층  | 단일    | 상속 구조        |
| 사용 목적  | 단순 상태 | 복잡한 상태       |

- 예시

```kotlin
sealed class Result {
    object Loading : Result()
    data class Success(val data: String) : Result()
    data class Error(val message: String) : Result()
}
```

- 데이터가 필요하면 sealed class가 더 적합