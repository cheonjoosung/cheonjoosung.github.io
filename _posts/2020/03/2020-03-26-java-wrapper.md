---
title: 자바 Wrapper 클래스 개념과 사용하는 이유
tags: [Java]
style: fill
color: dark
description: 자바 Wrapper 클래스의 개념과 사용이유
---

## Wrapper 클래스
Wrapper클래스는 자바의 primitive type의 객체화 버전이다. 

| 비교   | Primitive | Wrapper   |
|:-----:|:---------:|:---------:|
| 1     | byte    | Byte      |
| 2     | short   | Short     |
| 3     | int     | Integer   |
| 4     | long    | Integer   |
| 5     | float   | Integer   |
| 6     | double  | Integer   |
| 7     | char    | Character |
| 8     | boolean | Boolean   |

- 위의 표는 자바의 프리미티브 타입 8개에 대한 객체로 감싼 결과이다.
- **박싱** : 기본 타입 -> 래퍼 클래스의 형태로 변환
- **언박싱** : 래퍼 클래스 -> 기본 타입의 형태로 변환
- 자바 1.5 버전 이상부터는 박싱/언박싱이 자동으로 되므로 크게 신경쓸 필요는 없다.

## Wrapper 클래스를 사용하는 이유?
프리미티브 타입을 사용할 수 없는 경우를 아래의 코드릍 통해 살펴보자.

```javascript
int [] arr = new int[4];
ArrayList<int> list = new ArrayList<>() -----> Error

ArrayList<Integer> list = new ArrayList<>()
```

- 배열은 크기가 정해져있기 때문에 크기가 변하는 상황에서는 사용이 불가능하다.
- ArrayList, LinkedList 등의 자료구조를 사용해야 하고 여기에서는 wrapper 클래스 데이터 타입만 사용이 가능하다.

## 활용법
- Integer.MAX_VALUE, Integer.MIN_VALUE 를 통해 int 범위의 최대 최소값 접근
- Integer.toBinaryString(), Integer.toHexString(i), Integer.toOctalString(i)를 통해 2진수, 8진수, 16진수의 문자열로 변환가능
- 각 래퍼 클래스의 에서 longValue(), doubleValue(), byteValue() 등을 통해 기본형 타입의 값으로 바꿀 수 있다.

## 결론
- **활용법에 대해서 알고 있으면 유용하게 써먹을 수 있음**