---
title: Kotlin 타입 캐스팅 as - 안전한 캐스팅부터 스마트 캐스트까지
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin Sequence를 활용한 지연 연산과 효율적인 컬렉션 처리 방식. filter, map, takeWhile 등 실전 예제로 배우는 Sequence의 모든 것.
---

## ✨ 개요

Kotlin에서는 타입 안정성(type safety) 을 강조하면서도 유연한 타입 캐스팅 기능을 제공합니다. 

Java에서 흔히 사용하던 강제 형변환을 Kotlin에서는 보다 안전한 방식으로 다루며, 스마트 캐스트(smart cast) 기능까지 기본으로 제공합니다.

---

## 1. ✅ Kotlin에서 타입 캐스팅이 필요한 이유?

- Any 또는 플랫폼 타입에서 명확한 타입으로 변환 필요할 때
- Intent에서 extra 값 캐스팅 시
- 다양한 타입을 처리하는 함수에서 공통 동작 정의를 위해 필요

---

## 2. ✅ as vs as? 차이점

- as
  + 강제 캐스팅 ClassCastException
- as?
  + 안전한 캐스팅 실패 시 null 반환
- 코드
  ```kotlin
  val value: Any = "Hello Kotlin"
  val str1: String = value as String       // 성공
  val str2: String? = value as? String     // 성공 (nullable)
  val num: Int? = value as? Int            // 실패 → null 반환
  ```

---

## 3 ✅. 스마트 캐스트 (Smart Cast)

- 스마트 캐스트는 조건문 내 타입 확인 후 자동으로 타입을 추론하는 기능입니다.
```kotlin
fun printLength(obj: Any) {
    if (obj is String) {
        // 스마트 캐스트: obj는 이 블록 내에서 String으로 간주됨
        println("Length: ${obj.length}")
    }
}
```
> 주의: var, 커스텀 getter, nullable 변수에는 스마트 캐스트가 적용되지 않을 수 있습니다.

---

## 4 ✅. 실전 예제: Intent extra 캐스팅

```kotlin
val name = intent.getStringExtra("name")
val age = intent.extras?.get("age") as? Int ?: 0
```
- 플랫폼 타입 Intent에서 안전한 캐스팅 사용 권장
- as?로 null 안전성을 확보하고, 기본값 설정으로 안정성 강화

---

## 5.🧠 **결론**

- Kotlin에서는 as?와 스마트 캐스트를 적극 활용해 예외 없는 타입 캐스팅을 추구함
- 조건문과 함께 is, as?를 조합하면 코드 안정성과 가독성을 모두 확보 가능
- Java의 단순 캐스팅 습관을 버리고, Kotlin의 안전한 방식으로 전환하는 것이 핵심