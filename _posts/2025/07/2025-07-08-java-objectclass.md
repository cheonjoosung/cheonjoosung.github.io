---
title: Java의 Object 클래스 이해하기
tags: [ Java ]
style: fill
color: dark
description: Kotlin의 모든 클래스가 상속받는 최상위 타입 Any 클래스의 역할과 주요 메서드, Java Object와의 차이를 정리합니다.
---

## ✨ 개요

Java에서 모든 클래스는 자동으로 상속받는 클래스가 있습니다. 바로 Object 클래스입니다. 
오늘은 이 Object 클래스가 어떤 역할을 하는지, 어떤 메서드들을 제공하는지 자세히 알아보겠습니다.

---

## 1. 🧩 개념 및 정의

- Object 클래스는 **모든 클래스의 최상위 부모(Superclass)**입니다.
- 자바에서 선언한 모든 클래스는 명시적으로 상속을 받지 않아도 자동으로 Object를 상속받습니다.
- 핵심 기능: **기본적인 메서드들(equals, toString, hashCode, clone 등)**을 제공하여 객체 비교, 문자열 표현, 복사, 동기화 등에서 활용됩니다 

---

## 2. ⚙️ 주요 메서드 정리

| 메서드                                 | 설명                              | 오버라이딩 권장 여부           |
| ----------------------------------- | ------------------------------- | --------------------- |
| `toString()`                        | 객체의 문자열 표현 반환                   | ✅                     |
| `equals(Object obj)`                | 객체 비교 (주소 비교 → 값 비교로 커스터마이징 가능) | ✅                     |
| `hashCode()`                        | 객체의 해시코드 반환                     | ✅ (equals 오버라이딩 시 함께) |
| `clone()`                           | 객체 복사 (얕은 복사)                   | 필요시                   |
| `finalize()`                        | 객체가 GC 되기 직전에 호출                | ❌ (Deprecated)        |
| `getClass()`                        | 런타임 클래스 정보 반환                   | ❌                     |
| `wait()`, `notify()`, `notifyAll()` | 멀티스레드 동기화 지원                    | ❌ (주의해서 사용)           |

---

## 3. 🧪 예제

```java
class Person {
    String name;
    int age;

    Person(String name, int age) {
        this.name = name;
        this.age = age;
    }

    @Override
    public String toString() {
        return "이름: " + name + ", 나이: " + age;
    }

    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;
        if (!(obj instanceof Person)) return false;
        Person other = (Person) obj;
        return this.name.equals(other.name) && this.age == other.age;
    }

    @Override
    public int hashCode() {
        return name.hashCode() + age;
    }
}

public class ObjectDemo {
    public static void main(String[] args) {
        Person p1 = new Person("Joo", 30);
        Person p2 = new Person("Joo", 30);

        System.out.println(p1.toString());          // 이름: Joo, 나이: 30
        System.out.println(p1.equals(p2));          // true
        System.out.println(p1.hashCode() == p2.hashCode()); // true
    }
}
```

---

## 4. 💡 핵심

- equals()와 hashCode()는 항상 함께 오버라이딩해야 합니다. (HashMap, HashSet에서 객체가 제대로 비교되지 않을 수 있음)
- clone()은 얕은 복사만 가능하며, Cloneable 인터페이스를 구현해야 사용 가능
- getClass()는 리플렉션이나 로깅에 유용

---

## 5. 🧾 결론

Java의 Object 클래스는 자바에서 객체지향 프로그래밍을 하는 데 있어서 뼈대와도 같은 존재입니다. 
많은 기능을 제공하며, 특히 객체 비교나 로깅, 해시 기반 자료구조를 쓸 때 꼭 알아야 할 메서드들이 포함되어 있습니다. 
실무에서도 toString, equals, hashCode는 거의 필수적으로 오버라이딩하므로, 개념과 용법을 잘 숙지해 두세요!