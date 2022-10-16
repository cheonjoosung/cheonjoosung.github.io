---
title: 코틀린 apply, with, let, also, run 이해하기 | Scope Function  
tags: [Kotlin]
style: fill
color: dark
description: 코틀린 apply, with, let, also, run 이해하기 | Scope Function
---

## Scope Function

코틀린에는 객체의 컨텍스트 내에서 코드 블록을 실행하는 것이 목적인 여러 함수가 있다.
람다 식이 있는 개체에서 이러한 이러한 함수(apply, with, let, also, run)를 호출하면 임시 범위가 형성이 됨
이름 없이 개체에 접근이 가능함.


## 요약정리
1. let - it - Lambda Result - Extension Function
   - null 이 아닌 객채에서 람다실행
   - 로컬 범위의 변수로 표현식 소개
2. run - this - Lambda Result - Extension Function
    - 개체 구성 및 결과 계산
3. run -  - Lambda Result - No: called without the context object
   - 표현식이 필요한 실행 문
4. with - this - Lambda Result - No: takes the context object as an argument.
   - 객체에 대한 그룹화 함수 호출
5. apply - this - Context Object - Extension Function
   - 개체 구성 및 결과 계산
6. also - it - Context Object - Extension Function
   - 추가 효과


## Context Object: This Or IT

범위 함수에서는 컨텍스트 객체로 this데(람다 수신자) or it(람다 인자) 을 사용하고 둘 다 동일한 기능을 제공한다.

```kotlin
val str = "hello"
str.run { println(this) }
str.let { println(it) }
``` 

동일한 기능을 하지만 약간의 차이가 존재한다.

### this
- run, with, apply 컨텍스트 개체를 람다 수신자로 this 를 사용한다.
- this 를 생략해도 되지만 상위 블록의 this와 혼동이 될 수 있으므로 쓰는것이 좋음
- 호수를 호출하거나 속성을 할당할 때 씀

### it
- let, also 인수 이름이 없을 때 암묵적으로 it 개체에 엑세스 함
- it 대신에 별도로 람다 인자를 정하여 코드의 가독성을 높일 수 있음

### 반환 값 유무
1. apply, also : context object 반환
   - apply { }. also { } .  context object 가 반환되므로 체이닝이 가능
2. let, run, with : 람다 결과를 반환
   - context object 에 값을 할당하거나 변경하는 역할을 함

## Functions
각 기능에 맞게 잘 호출하여 사용해야 함

### let
1. it 를 람다 인자로 사용하고 하나 이상의 함수를 호출하는데 사용되어 질 수 있음
2. Not null 인 값의 코드 블록을 실행하는데 사용되어 짐
```kotlin
val numbers = mutableListOf("one", "two", "three")
with(numbers) {
   println("'with' is called with argument $this")
   println("It contains $size elements")
}
``` 

### with
1. this 람다 리시버를 사용함
2. 값을 계산하기 위한 특성/함수 도우미로써도 역할 함
```kotlin
val numbers = mutableListOf("one", "two", "three")
with(numbers) {
   println("'with' is called with argument $this")
   println("It contains $size elements")
}

val numbers = mutableListOf("one", "two", "three")
with(numbers) {
   println("'with' is called with argument $this")
   println("It contains $size elements")
}
``` 

### run
1. this 람다 리시버를 사용함
```kotlin
val service = MultiportService("https://example.kotlinlang.org", 80)

val result = service.run {
   port = 8080
   query(prepareRequest() + " to port $port")
}

// the same code written with let() function:
val letResult = service.let {
   it.port = 8080
   it.query(it.prepareRequest() + " to port ${it.port}")
}

val hexNumberRegex = run {
   val digits = "0-9"
   val hexDigits = "A-Fa-f"
   val sign = "+-"

   Regex("[$sign]?[$digits$hexDigits]+")
}

for (match in hexNumberRegex.findAll("+123 -FFFF !%*& 88 XYZ")) {
   println(match.value)
}
``` 

### apply
1. this 람다 리시버를 사용함
2. 수신자 개체의 작동하는 코드 블록을 설정하기 위해서
```kotlin
val adam = Person("Adam").apply {
   age = 32
   city = "London"
}
println(adam)
``` 

### also
1. it 를 사용
2. 추가적인 작업을 수행하는데 좋고 자기 자신을 반환 함
3. apply 는 객체의 속성을 수정하는 반면 also 는 이를 변경하지 않고 다른 작업을 수행
```kotlin
val numbers = mutableListOf("one", "two", "three")
numbers
   .also { println("The list elements before adding new one: $it") }
   .add("four")
``` 


참조 - [코틀린 공식 Scope Function](https://kotlinlang.org/docs/scope-functions.html#distinctions)