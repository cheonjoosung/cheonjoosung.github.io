---
title: Java Heap&Stack 변수저장 이해
tags: [Java]
style: fill
color: dark
description: Java Heap&Stack 변수저장 이해
---

## 변수(기본형 & 참조형) 어떻게 저장되는가?
자바의 자료형에는 기본형과 참조형이 있다.

기본형은 boolean, char, byte, short, int ,long, float, double이 있고 이외에는 참조형 타입니다. 기본형에는 값이 저장되고 참조형에는 주소값이 저장된다고 알고 있다.

**그렇다면 기본형/참조형 타입의 변수는 JVM의 메모리 구조에 어떻게 동작을 할까?**
![preview](https://user-images.githubusercontent.com/13310269/158181516-04ddb606-cf8f-4114-9855-eb658cab8663.png)
Heap에는 변수들이 저장이 되고 이를 연결하는 선이 보면 Java Stack, Method Area, JNI, Heap(Object → Object) 4가지가 있다. 각 선들은 객체를 가르키고 있다. 이는 각 영역에서 호출할 때 주소값를 바탕으로 Heap 영역에 변수를 참조한다고 생각하면 된다.

참고사항으로 연결이 안 된(UnReachable) Object 들은 사용을 안하는 것들로 추후에 GC에 의해 메모리에서 제거가 된다.

**Stack & Heap & Method Area**
```javascript
public class Test {

    static int methodValue = 4;
    static Object methodObject = new Object();

    public static void main(String[] args) {

        int stackValue = 4;

        int [] heapValue = new int[3];
        heapValue[0] = 0;
        heapValue[1] = 1;
        heapValue[2] = 2;

				System.out.printlnt(heapValue) // 주소값 [I@6f94fa3e
    }
}
```
![preview](https://user-images.githubusercontent.com/13310269/158181495-91d5c9c4-f97a-439f-91bf-5c021274c81c.png)
기본형 타입으로 선언된 변수들은 바로 값을 가지만 참조형으로 선언된 것들은 값을 바로 가지지 않는다. System.out.println()으로 찍어보면 주소값이 찍힌다.

![preview](https://user-images.githubusercontent.com/13310269/158181534-80b8b632-b8f3-4d15-8119-90d3e6f110f3.png)
기본형 타입으로 선언된 변수들이나 지역변수로 선언된 변수들은 Stack에 값을 가지고 저장이 된다. 메서드 실행에 따라 자동으로 할당/해제가 된다. 재귀로 동일한 메소드를 호출하면 stackOverFlowError가 나는 것을 보면 Stack영역에 계속해서 method() Call을 쌓다보니 저장할 공간이 없어서 터진다.

참조형 타입으로 선언된 변수가 클래스 변수이면 Method Area에 공간이 할당이 되고 일반 변수일 경우 Stack Area에 할당이 된다.

바로 할당이 되는 것은 아니고 new 연산자로 할당을 하면 Heap영역에 값을 저장한 공간을 만든다. 그 후 stack or method Area에 값을 저장한 주소값을 매핑시킨다.
