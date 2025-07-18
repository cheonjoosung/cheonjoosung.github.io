---
title: Kotlin Nothing 이란?
tags: [ Kotlin ]
style: fill
color: dark
description: 코틀린 Nothing 이란?
---

## ✨ 개요

Kotlin을 공부하다 보면 조금 낯선 타입인 **Nothing**을 마주하게 됩니다. 
이름처럼 **아무것도 없는 타입**인데, 그렇다면 대체 왜 존재할까요? 
이번 포스팅에서는 Kotlin의 Nothing 타입을 개념부터 실전 활용까지 이해하기 쉽게 정리해보겠습니다.

---

## 1️⃣ Nothing 이란?

`Nothing`은 코틀린에서 **절대 값이 존재하지 않는 타입**을 나타냄
즉, 정상적으로 리턴될 일이 없는 함수의 반환 타입으로 사용
> "이 함수는 절대 정상 종료되지 않습니다."를 컴파일러에게 알려줌

---

## 2️⃣ 주요 특징

| 특징       | 설명                                           |
| -------- | -------------------------------------------- |
| 반환 불가    | `Nothing`을 반환하는 함수는 **정상 종료되지 않음**           |
| 타입 서브 역할 | `Nothing`은 **모든 타입의 하위 타입** (Subtype of all) |
| 주요 용도    | 예외 발생, 무한 루프 등에서 사용                          |
| 의미       | **"이 지점 이후 코드는 실행되지 않는다"**                   |

---

## 3️⃣ 언제 사용할까?

### 예외를 던지는 함수
```kotlin
fun fail(message: String): Nothing {
    throw IllegalArgumentException(message)
}
```
- 정상 반환되지 않음
- 반환 타입을 `Nothing`으로 명시해 의도를 분명히 함


### 엘비스 연산자 (?:)와 함께
```kotlin
val name: String? = null
val result = name ?: fail("name is required")
// result는 String으로 추론됨 (Nothing은 모든 타입의 하위 타입이므로)
```
- `Nothing` 덕분에 정상 타입 추론이 가능
- `fail()` 호출 이후 실행될 일이 없으므로 문제 없음


### 무한 루프 함수
```kotlin
fun loopForever(): Nothing {
    while (true) {
        println("계속 돈다...")
    }
}
```

---

## 4️⃣ Unit vs Nothing 차이

| 타입        | 의미                | 예시                          |
| --------- | ----------------- | --------------------------- |
| `Unit`    | 반환 값 없음           | 정상 종료된 void 함수              |
| `Nothing` | 절대 반환 없음 (예외, 종료) | throw, exitProcess, 무한 루프 등 |

---

## 5️⃣ 결론

`Nothing`은 **"정상 흐름이 끊기는 곳"**에 사용하는 특수한 타입입니다.
예외, 무한 루프, 종료 함수에 활용되며, Kotlin의 타입 안정성과 표현력을 높여줍니다. 
실무에서는 주로 엘비스 연산자, 예외 헬퍼 함수, 타입 추론 보조에 자주 쓰입니다.