---
title: Kotlin 예외 처리 완벽 가이드 - Best Practice 포함
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin에서 안전하고 효과적인 예외 처리를 위한 try-catch, throw, runCatching 등의 기법과 Best Practice를 쉽게 설명합니다.
---

## ✨ 개요

코틀린 java의 예외 처리 모델을 계승하면서도 null 안전성과 함께 더 깔끔한 방식의 예외 처리를 제공합니다. 
`try-catch`, `try-finally`, `throw` 외에도 `runCatching`의 
sealed class 기반 Result 패턴 등 다양한 예외 처리 방법을 제공합니다.

---

## 1. 🧩 기본 예외 처리 방식

```kotlin
try {
    val result = riskyOperation()
} catch (e: IOException) {
    e.printStackTrace()    
} finally {
    println("항상 실행")
}
```
- `try` 블록으로 예외 발생 코드 감싸기
- `catch`에서 예외 핸들링
- `finally`는 항상 실행 됨

---

## 2. ⚙️ runCatching 과 Result 활용

- 코틀린 표준 라이브러리에서 제공하는 `runCatching`은 함수형 방식의 예외 처리를 제공합니다.
- 코드의 가독성을 높이고 예외 흐름을 함수형 체이닝으로 표현 가능

```kotlin
val result = runCatching {
    riskyOperation()
}.onSuccess {
    println("성공: $it")
}.onFailure {
    println("실패: ${it.message}")
}
```

---

## 3. 🧪 Best Practice

- 구체적인 Exception 타입을 캐치 (Exception 대신 IOException, IllegalArgumentException 등)
- 로직에 따라 throw 대신 sealed class Result 패턴 고려
- 로그를 남기되 사용자 메시지는 가공해서 전달
- checked exception 을 wrapping 해 custom exception 정의

---

## 4. 🧾 결론

코틀린의 예외 처리는 가독성, 함수형 체이닝, null 안정ㅅ겅을 함께 살린 강력한 설계가 가능합니다.
기존 Java 방식의 `try-catch` 뿐 아니라 `runCatching` 이나 `sealed class Result` 패턴까지
상황에 맞게 적용해 보다 견고한 코드를 작성해 보세요.