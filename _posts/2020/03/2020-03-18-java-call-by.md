---
title: 자바 Call by Value vs Call by Reference
tags: [Java]
style: fill
color: dark
description: 자바의 call by value 의 call by reference의 차이와 함께 데이터 타입에 대해서 이해하자
---

## 자바의 데이터 타입 : 기본형 vs 참조형
call by value와 call by reference를 알기전에 자바의 데이터 타입에 대해서 알 필요가 있습니다. 왜냐하면 값의 전달과 참조할 수 있는 주소값의 전달은 데이터 타입에 의해서 결정되기 때문입니다.

**기본형**
- 논리형 boolean
- 문자형 char
- 정수형 int, long
- 실수형 float, double

**참조형**
- 기본형을 제외한 모든 타입은 참조형이다.

## Call by value
기본형 타입은 값을 복사하여 메소드를 호출한다.

```javascript
int a = 5
int b = 100

print(a + " " + b)
swap(a, b)
print(a + " " + b)

void swap(int a, int b) {
	int temp = a;
	a = b;
	b = temp;
}
```

swap() 메소드를 호출해도 값을 복사하여 전달하기 때문에 a, b의 값은 계속해서 5,100을 출력한다. 

**중요한 또다른 예제를 살펴보자**
```javascript
String a = "HI"
String b = "BYE"

print(a + " " + b)
swap(a, b)
print(a + " " + b)

void swap(String a, String b) {
	String temp = a;
	a = b;
	b = temp;
}
```

두 번의 출력 결과는 "HI", "BYE"로 동일합니다. 기본형 타입외에 모든 타입은 참조형인데 이상하게도 String 클래스는 기본형처럼 사용합니다. **자바에서 String 클래스는 특수 포지션이라고 생각하면 되고 Call By Value 범주에 속하는 예외라고 반드시 기억해야 합니다.**

[자바 String](java-string)의 포스트를 통해 특수성을 조금 더 알아보실 수 있습니다.


## Call By Reference
메소드를 호출할 때 변수의 주소값을 전달하는 방식

```javascript
int [] arr = {0, 5, 100}

print(arr[0] + " " + arr[2])
swap(arr, 0, 2)
print(arr[0] + " " + arr[2])

swap(int [] arr, int i1, int i2){
	int temp = arr[i1];
	arr[i1] = arr[i2];
	arr[i2] = temp;
}
```

결과값은 0,100 - 100,0 이 출력이 된다. 배열은 주소값이 전달되기에 배열에서 i1, i2의 인덱스의 위치가 참조하고 있는 값을 실제로 바꾼다.

## 결론
- **IT 전공면접에서 Call By Value & Reference 물어봄 + String 예외 포함**
- 자바에서 String 클래스는 특이함