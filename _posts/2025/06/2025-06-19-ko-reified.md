---
title: Kotlin reified 키워드 완벽 정리 - 제네릭 타입 안전하게 다루기
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin에서 제네릭 타입 정보를 런타임에 유지할 수 있게 해주는 reified 키워드의 개념, 사용법, 주의사항을 예제 중심으로 정리합니다.
---

## ✨ 개요

Kotlin에서는 `제네릭 타입은 기본적으로 런타임에 소거(type erasure)` 됩니다. 
하지만 inline 함수와 reified 키워드를 함께 사용하면 제네릭 타입 정보를 런타임에 유지하여, 타입 캐스팅이나 타입 체크가 가능해집니다.

---

## 1. 🔧 🔍 기본 문법과 개념

- reified는 inline 함수에서만 사용 가능
- 제네릭 타입 T를 실체화해 T::class 같은 연산이 가능함

```kotlin
inline fun <reified T> Gson.fromJson(json: String): T =
  this.fromJson(json, T::class.java)
```

> 위 예제는 fromJson<T>() 호출 시 T의 타입 정보를 유지하여 자동으로 파싱할 수 있도록 해줍니다.

---

## 2. ✅ 활용 사례

- 타입 체크
  + ```kotlin
    inline fun <reified T> Any.isOfType(): Boolean = this is T
    ```
- Intent Extras 추출
  + ```kotlin
    inline fun <reified T : Serializable> Intent.serializable(key: String): T? {
      return getSerializableExtra(key) as? T
    }
    ```
- GSON
  + ```kotlin
    inline fun <reified T> parseJson(json: String): T {
       return Gson().fromJson(json, T::class.java)
    }
      
    val user: User = parseJson(jsonString)
    ```

---

## 3. ⚠️ 주의사항

- `reified`는 반드시 inline과 함께 사용해야 함
- 클래스 멤버 함수나 추상 함수에서는 사용 불가
- 복잡한 제네릭 구조에서는 한계가 있음

---

## 4. 🧾 결론

`reified`는 Kotlin 제네릭에서 런타임 타입 정보를 안전하게 활용할 수 있는 강력한 도구입니다. 
복잡한 리플렉션이나 타입 캐스팅을 줄이고, 타입 안정성을 높이는 데 효과적입니다. 
단, inline과 함께 사용해야 하며, 적절한 위치에서 사용하는 것이 중요합니다.

- reified 키워드 언제 왜 사용?
  + reified는 제네릭 타입의 런타임 타입 정보를 유지하고 싶을 때 사용
  + 코틀린의 일반 제네릭은 타입 소거 때문에 런타임에 T::class, is T 같은 연산이 불가능
  + reified를 사용하면 이런 제약을 우회할 수 있음 (단, reified는 inline 함수에서만)
  + 예로 Gson 파싱, 타입 필터링, ViewModelFactory 로 뷰 모델 생성 시

- 왜 reified는 inline 함수에서만 사용?
  + reified는 타입 정보를 런타임까지 보존해야 하므로, 컴파일 시점에 해당 타입이 코드에 구체적으로 삽입
  + inline 함수는 컴파일 시점에 함수 본문이 호출된 위치에 직접 복사되기에 제네릭 타입도 실제 타입으로 치환
  + 이 구조 덕분에 T::class, is T 같은 코드가 가능해짐
  + 일반 함수는 이 치환이 일어나지 않기 때문에 reified를 사용 불가