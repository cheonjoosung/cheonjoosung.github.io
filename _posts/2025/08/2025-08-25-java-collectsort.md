---
title: Java Comparator vs Comparable 완벽 정리
tags: [ Java ]
style: fill
color: dark
description: Java에서 객체 정렬을 할 때 가장 많이 사용하는 두 가지 인터페이스가 있습니다. 바로 `Comparable`과 `Comparator`입니다. 이 둘은 목적은 비슷하지만, 사용하는 방식과 유연성에 차이가 있습니다.
---

## ✨ 개요

멀티쓰레드 프로그래밍을 하다 보면, 여러 쓰레드가 하나의 변수를 동시에 변경하면서 동기화 문제가 발생하게 됩니다. 
이를 해결하는 대표적인 방법 중 하나가 바로 Java의 Atomic 클래스입니다. 이번 포스팅에서는 Atomic 클래스의 개념, 종류, 사용법, 장단점, 예제까지 모두 정리해드립니다.

---

## 1️⃣ `Comparable` 인터페이스란?

### 1. 개념 및 정의
- 기본 정렬 기준(natural ordering)을 정의할 때 사용
- 해당 클래스 내부에서 직접 정렬 기준을 구현
- `compareTo()` 메서드 구현 필요

### 2. 사용 예시
- ```java
  public class Person implements Comparable<Person> {
      String name;
      int age;

      public Person(String name, int age) {
        this.name = name;
        this.age = age;
      }

      @Override
      public int compareTo(Person other) {
        return this.age - other.age; // 나이 오름차순 정렬
      }
  }
  
  List<Person> list = Arrays.asList(
    new Person("Joo", 30),
    new Person("Min", 25)
  );
  
  Collections.sort(list); // compareTo 기준으로 정렬
  ```   

---

## 2️⃣ Comparator 인터페이스란?

### 1. 개념 및 정의
- 외부에서 정렬 기준을 정의할 수 있는 인터페이스
- 기존 클래스를 변경하지 않고 다양한 정렬 가능
- compare(T o1, T o2) 메서드 구현

### 2. 사용 예시
- ```java
  public class AgeComparator implements Comparator<Person> {
    @Override
    public int compare(Person p1, Person p2) {
        return p1.age - p2.age; // 나이 오름차순
    }
  }
  
  Collections.sort(list, new AgeComparator());
  
  // Java 8 이상에서는 람다로 더욱 간결하게 작성 가능:
  list.sort((p1, p2) -> p1.name.compareTo(p2.name)); // 이름 기준 오름차순
  ```

---

## 3️⃣ Comparable vs Comparator 비교
- | 항목       | `Comparable`             | `Comparator`                   |
  | -------- | ------------------------ | ------------------------------ |
  | 위치       | 클래스 내부                   | 클래스 외부                         |
  | 메서드      | `compareTo(T o)`         | `compare(T o1, T o2)`          |
  | 정렬 기준 변경 | 어렵다 (하나만 정의 가능)          | 쉬움 (여러 기준 정의 가능)               |
  | 사용 예시    | `Collections.sort(list)` | `Collections.sort(list, comp)` |
  | 자주 쓰는 곳  | TreeSet, TreeMap (기본 정렬) | 다중 기준, 동적 정렬 등                 |


---

## 4️⃣ 결론

- | 핵심 요약 ✅                          |
  | -------------------------------- |
  | `Comparable` = 기본 정렬 기준 (내부 정의)  |
  | `Comparator` = 커스텀 정렬 기준 (외부 정의) |
  | 상황에 맞게 유연하게 선택하여 사용하자            |
- 정렬 기준이 한 가지이고 클래스 내부에서 직접 정의해도 되는 경우는 Comparable, 여러 정렬 방식이 필요하거나 외부에서 제어하고 싶다면 Comparator를 선택하세요!