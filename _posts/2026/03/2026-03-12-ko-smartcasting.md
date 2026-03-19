---
title: (Kotlin/코틀린) 스마트 캐스팅(Smart Casting)과 메인 스레드(Thread) 관련 주의사항 정리
tags: [ Android,Kotlin ]
style: fill
color: dark
description: (Kotlin/코틀린) Kotlin Smart Casting이 멀티 스레드 환경에서 왜 동작하지 않는지, 특히 Android 메인 스레드와 비동기 처리에서 발생하는 SmartCast 오류 원인과 해결 방법 정리
---

## 개요

- Kotlin의 Smart Cast(스마트 캐스팅) 기능은 null 체크나 타입 체크 이후에 자동으로 타입을 캐스팅해주는 매우 편리한 기능입니다.
- 하지만 Android 개발 중 다음과 같은 오류를 만나는 경우가 있습니다.
- ```text
  Smart cast to 'Type' is impossible
  because 'variable' is a mutable property
  that could have been changed by this time
  ```
- 특히 코루틴 / 스레드 / 비동기 코드에서 자주 발생합니다.

---

## 1. Kotlin Smart Cast란?

Smart Cast는 Kotlin 컴파일러가 타입을 자동으로 캐스팅하는 기능입니다.

- ```kotlin
  var name: String? = "Android"
  
  if (name != null) {
    println(name.length)
  }
  ```
  - 검사를 했기 때문에 name을 String으로 간주합니다.

---

## 2. Smart Cast가 실패하는 경우

다음 코드에서는 Smart Cast가 동작하지 않습니다.

```kotlin
var name: String? = "Android"

if (name != null) {
    doSomething()
    println(name.length)
}
```

- Smart cast to 'String' is impossible

---

### 3. 왜 컴파일러가 Smart Cast를 막는가?

Kotlin은 다음 상황을 고려합니다.
- ```kotlin
  var name: String? = "Android"

  if (name != null) {
    Thread {
        name = null
    }.start()

    println(name.length)
  }
  ```
  - NullPointerException 발생할 수 있음
  - 즉 Kotlin은 스마트 캐스팅이 안전하지 않다고 판단하면 컴파일 단계에서 차단합니다.

---

## 4. Android 메인 스레드에서도 발생하는 이유

Android 코드에서도 이런 문제가 발생합니다.

- ```kotlin
  var user: User? = getUser()

  if (user != null) {
    button.setOnClickListener {
        println(user.name)
    }
  }  
  ```
  - 컴파일러 입장에서는 OnClickListener가 언제 실행될지 모름
  - 메인 스레드 이후에 값이 변경될 수 있을것이라 판단

---

## 5. Smart Cast 가능 조건

| 조건           | 가능 여부 |
| ------------ | ----- |
| val 변수       | 가능    |
| var 변수       | 제한    |
| 지역 변수        | 가능    |
| 클래스 property | 제한    |

---

## 6. 가장 많이 사용하는 해결 방법

- 지역 변수로 복사
  - ```kotlin
    val localUser = user

    if (localUser != null) {
      println(localUser.name)
    }
    ```
- let 사용
  - ```kotlin
    user?.let {
        println(it.name)
    }
    ```
- requireNotNull
  - ```kotlin
    val safeUser = requireNotNull(user)
    println(safeUser.name)
    ```

---

## 7. Coroutine 환경에서 Smart Cast 문제

코루틴에서는 더 자주 발생합니다.

```kotlin
var user: User? = repository.getUser()

if (user != null) {

launch {
    println(user.name)
  }
}
```
- 컴파일러는 다음을 의심 -> 코루틴 실행 전에 값 변경 가능

- 해결 방법

```kotlin
val safeUser = user

if (safeUser != null) {
    launch {
        println(safeUser.name)
    }
}
```

---

## 8. Smart Cast 내부 동작

- Kotlin Smart Cast는 **컴파일러 데이터 흐름 분석(Data Flow Analysis)**으로 동작합니다.
- ```text
  null check
    ↓
  변수 변경 가능성 검사
    ↓
  가능하면 Smart Cast
  ```
- 다음 조건 중 하나라도 걸리면 Smart Cast가 차단됩니다.
  - mutable property
  - open property
  - custom getter
  - multi-thread risk