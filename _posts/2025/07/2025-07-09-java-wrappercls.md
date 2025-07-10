---
title: Java Wrapper 클래스 이해하기
tags: [ Java ]
style: fill
color: dark
description: Java Wrapper 클래스 이해하기
---

## ✨ 개요

자바에서 기본형(Primitive type)은 성능 측면에서 매우 효율적이지만, 객체로 다뤄야 할 때 한계가 있습니다. 
이 한계를 극복하기 위해 등장한 것이 바로 **Wrapper Class(래퍼 클래스)**입니다.

---

## 1. 🧩 개념 및 정의

- Wrapper Class란 기본 자료형(primitive type)을 **객체(Object)**로 감싸는 클래스입니다.
- int, char, double 등 기본형 데이터를 Integer, Character, Double 등의 객체로 변환해줍니다.

| 기본형       | 래퍼 클래스      |
| --------- | ----------- |
| `byte`    | `Byte`      |
| `short`   | `Short`     |
| `int`     | `Integer`   |
| `long`    | `Long`      |
| `float`   | `Float`     |
| `double`  | `Double`    |
| `char`    | `Character` |
| `boolean` | `Boolean`   |


---

## 2. ⚙️ 사용하는 이유?

- 컬렉션 사용: List<int> 불가능 → List<Integer>로 사용해야 함
- 기본형 → 객체형으로의 자동 변환 필요
- 유틸리티 메서드 사용: 파싱, 문자열 변환 등 (Integer.parseInt(), Double.valueOf() 등)
- 객체로서 null 값 표현 가능

---

## 3. 🧪 오토박싱 (auto-boxing) & 언박싱(Unboxing)

```java
public class WrapperExample {
    public static void main(String[] args) {
        // 오토박싱
        Integer num = 100; // int → Integer
        // 언박싱
        int value = num;   // Integer → int

        // 메서드 활용
        String str = "123";
        int parsed = Integer.parseInt(str); // 문자열 → int
        System.out.println(parsed + 1); // 124

        List<Integer> scores = new ArrayList<>();
        scores.add(90); // 오토박싱으로 int → Integer
        scores.add(85);
        int total = 0;
        for (Integer score : scores) {
            total += score; // 언박싱 발생
        }
        System.out.println("총합: " + total);
    }
}

```
- 오토박싱 : 기본형 -> 객체형
- 언박싱 : 객체형 -> 기본형 

---

## 4. 💡 주요 메서드 정리

| 클래스         | 주요 메서드                                       |
| ----------- | -------------------------------------------- |
| `Integer`   | `parseInt()`, `valueOf()`, `compareTo()`     |
| `Double`    | `parseDouble()`, `isNaN()`, `valueOf()`      |
| `Boolean`   | `parseBoolean()`, `booleanValue()`           |
| `Character` | `isDigit()`, `isLetter()`, `toUpperCase()` 등 |

---

## 5. 💡 주의사항

```java
Integer a = 128;
Integer b = 128;
System.out.println(a == b);       // false
System.out.println(a.equals(b));  // true

Integer x = 127;
Integer y = 127;
System.out.println(x == y); // true (캐시 활용)
```
- == 사용 주의: Wrapper는 객체이므로, 값 비교는 equals() 사용해야 함
- 메모리 캐싱 주의: -128 ~ 127 범위 내에서는 동일 객체로 캐싱됨

---

## 6. 🧾 결론

Wrapper Class는 기본형의 한계를 보완하며 자바의 객체지향 프로그래밍을 더욱 유연하게 해줍니다. 
컬렉션, 제네릭, 유틸리티 메서드 등을 사용할 때 매우 유용하며, 
오토박싱과 언박싱 개념은 필수적으로 이해하고 있어야 합니다.