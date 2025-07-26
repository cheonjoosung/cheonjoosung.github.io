---
title: Java 다중 상속 (Multiple Inheritance) 완전 정복
tags: [ Java ]
style: fill
color: dark
description: Java 다중 상속 (Multiple Inheritance) 완전 정복
---

## ✨ 개요

Java를 공부하다 보면 자주 듣는 말이 있습니다.
> "Java는 클래스 다중 상속을 허용하지 않는다."
> 
도대체 왜 그런 걸까요? 그리고 정말 완전히 불가능할까요? 
이번 포스팅에서는 Java 다중 상속의 개념부터, 허용되지 않는 이유, 해결 방법까지 명확하게 정리해드립니다.



---

## 1️⃣ 다중 상속(Multiple Inheritance)란?

다중 상속이란 하나의 클래스가 둘 이상의 부모 클래스로부터 상속받는 것을 의미합니다.
하지만, Java는 클래스에서 이런 방식의 다중 상속을 금지합니다.

```java
class A { ... }
class B { ... }
class C : public A, public B { ... }
```

---

## 2️⃣ Java에서 클래스 다중 상속이 불가능한 이유

⚠️ Diamond Problem (다이아몬드 문제)
상속 구조가 아래처럼 엮일 경우, 어떤 부모의 메서드를 상속받는지 모호해지는 문제가 발생합니다.

```java
    A
   / \
  B   C
   \ /
    D
```
- A 클래스: void hello()
- B, C 클래스: A 상속
- D 클래스: B, C 모두 상속 (Java에서는 불가능)
→ D에서 hello()를 호출할 때, B 것인지 C 것인지 모호

---

## 3️⃣ Java에서 다중 상속을 대신하는 방법

### 3-1. 인터페이스를 통한 다중 상속 허용

```java
interface Flyable {
    void fly();
}

interface Swimmable {
    void swim();
}

class Duck implements Flyable, Swimmable {
    @Override
    public void fly() { System.out.println("오리 날다"); }
    @Override
    public void swim() { System.out.println("오리 헤엄친다"); }
}

```
> Duck 클래스는 Flyable과 Swimmable 인터페이스를 모두 상속받아 다중 상속 효과를 누림.

### 3-2. 디폴트 메서드와 충돌 해결

```java
interface A {
    default void hello() { System.out.println("A"); }
}
interface B {
    default void hello() { System.out.println("B"); }
}
class C implements A, B {
    @Override
    public void hello() {
        A.super.hello(); // 명시적으로 선택
    }
}
```
> Java 8 이후 인터페이스에 default 메서드가 추가되면서 다이아몬드 문제가 다시 발생할 수 있음. 하지만 명시적으로 해결해야 함.

---

## 4️⃣ 상속과 인터페이스 정리 비교

| 항목     | 클래스 상속 | 인터페이스 상속             |
| ------ | ------ | -------------------- |
| 다중 상속  | 불가     | 가능                   |
| 메서드 구현 | 가능     | Java 8 이후 default 가능 |
| 목적     | 재사용/확장 | 역할 명세                |

---

## 5️⃣ 결론

Java에서 클래스 다중 상속이 금지된 것은 복잡성과 모호성(Diamond Problem)을 피하기 위함입니다.
대신 인터페이스를 통해 역할 기반의 다중 상속을 유연하게 제공하고 있으며, Java 8 이후에는 default 메서드로 약간의 코드 재사용까지 가능해졌습니다.