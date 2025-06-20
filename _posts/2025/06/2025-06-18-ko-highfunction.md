---
title: Kotlin 고차 함수(Higher-Order Functions) 완벽 가이드
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin에서 고차 함수가 무엇인지, 어떻게 사용하는지, 그리고 람다와 함께 어떤 방식으로 활용되는지 실전 예제로 정리합니다.
---

## ✨ 개요

Kotlin에서 고차 함수(Higher-Order Function) 란 다른 함수를 인자로 받거나, 함수를 반환하는 함수를 의미합니다. 
함수 자체를 일급 객체로 다룰 수 있기 때문에 함수형 프로그래밍 스타일을 자연스럽게 지원합니다.

---

## 1. 🔧 기본 사용법

```kotlin
fun calculate(x: Int, y: Int, op: (Int, Int) -> Int): Int {
  return op(x, y)
}

val result = calculate(3, 5) { a, b -> a + b } // 8
```

- (Int, Int) -> Int는 함수 타입이며, 람다 표현식을 인자로 전달
- Kotlin에서는 함수도 값처럼 취급 가능

---

## 2. ✅ 자주 쓰이는 예시 함수들

- let, run, apply, also, with
- map, filter, forEach, reduce 등 컬렉션 관련 함수
- ```kotlin
  val names = listOf("Kotlin", "Java", "Swift")
  names.filter { it.startsWith("K") }.forEach { println(it)
  ```

---

## 3. ✅ 람다, 스코프, 코루틴 연계

### 3.1 람다와 고차함수

```kotlin
fun operate(a: Int, b: Int, action: (Int, Int) -> Int): Int {
    return action(a, b)
}

val result = operate(3, 5) { x, y -> x + y }  // 람다 전달
```
- 여기서 { x, y -> x + y }는 익명 함수(람다) 이고, 고차함수 operate는 이를 인자로 받는다.

### 3.2 스코프 함수와 고차함수 – DSL 스타일 코드

```kotlin
val user = User().apply {
    name = "Jane"
    age = 25
}

// 직접 구현
inline fun <T> T.myApply(block: T.() -> Unit): T {
  block() // 수신 객체로 실행
  return this
}
```
- apply는 고차함수이며, this를 수신 객체로 전달하는 람다를 실행한다.

### 3.3 코루틴과 고차함수 – 비동기 로직 최적화

```kotlin
suspend fun loadData(action: suspend () -> Unit) {
    println("Start loading")
    action() // 고차함수 실행
    println("End loading")
}

loadData {
  delay(1000)
  println("Data fetched!")
}
```
- 코루틴 덕분에 고차함수 안에서도 delay, API 호출, DB 작업 가능

---

## 4. ⚠️ 고차 함수의 장점

- 중복 코드 제거 및 재사용성 향상
- 간결하고 선언적인 코드 작성 가능
- 람다와 결합 시 높은 표현력

```kotlin
fun repeatTask(times: Int, task: () -> Unit) {
    repeat(times) { task() }
}
```

---

## 5. 🧾 결론

Kotlin에서 고차 함수는 함수형 프로그래밍을 지원하고, 람다와 함께 코드의 유연성과 가독성을 높이는 강력한 도구입니다. 
실제 프로젝트에서는 콜백 처리, 컬렉션 조작, DSL 작성 등 다양한 상황에서 광범위하게 사용되며, 
Kotlin의 핵심 기능 중 하나로 익혀두면 큰 도움이 됩니다.