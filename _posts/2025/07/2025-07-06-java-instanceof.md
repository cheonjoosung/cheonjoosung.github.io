---
title: Java instanceof 완벽 가이드 - 안전한 타입 검사와 패턴 매칭
tags: [ Java ]
style: fill
color: dark
description: Java의 instanceof 키워드를 이용한 타입 검사 방법과 Java 14부터 추가된 패턴 매칭 문법까지 쉽게 설명합니다.
---

## ✨ 개요

Java에서 `instanceof`는 객체가 특정 타입의 인스턴스인지 검사하는 키워드입니다.
상속 관계나 인터페이스 구현 여부를 런타임에 확인할 때 매우 유용합니다.

---

## 1. 🧩 기본 사용 

```java
void typeCheck(Object obj) {
    if (obj instanceof String) {
    String str = (String) obj;
    System.out.println(str.length());
}
}
```
- 타입을 검사한 뒤 안전하게 캐스팅

---

## 2. ⚙️ Java 14 패턴 매칭

```java
void typeCheck(Object obj) {
    if (obj instanceof String str) {
        System.out.println(str.length());
    }
}
```
- Java 14부터는 패턴 매칭 기능이 `instanceof`와 결합되어 더 간결한 코드 작성이 가능
- 검사 + 변수 선언 + 캐스팅을 한 줄로 처리 가능

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

- 불필요한 instanceof 남발을 피하고 다형성을 우선 고려
- 패턴 매칭이 가능한 최신 Java 버전을 사용한다면 적극 활용
- null 검사 후 사용 권장

---

## 4. 🧾 결론

`instnaceof`는 Java 타입 검사에서 필수적인 도구이며 패텅 매칭이 더해져 가독성이 크게 향상 되었음
객체지향적인 설계(다형성)를 우선 고려하되 안전한 런타입 검사 도구로써 적절한 활용 필요함