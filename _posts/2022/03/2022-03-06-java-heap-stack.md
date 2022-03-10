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
![preview](https://s3.us-west-2.amazonaws.com/secure.notion-static.com/eb4c9c99-77b5-470d-bfa0-1c8f548ad1b6/Untitled.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45EIPT3X45%2F20220310%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20220310T131620Z&X-Amz-Expires=86400&X-Amz-Signature=c509653937d3faa3654edeb90a84fe6bd982eaa1c8493c52a0a77802ae6d3e2b&X-Amz-SignedHeaders=host&response-content-disposition=filename%20%3D%22Untitled.png%22&x-id=GetObject)
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
![preview](https://s3.us-west-2.amazonaws.com/secure.notion-static.com/7591f3dd-61d2-40d6-99a0-6e0be9498c1e/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2022-03-06_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_11.17.58.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45EIPT3X45%2F20220310%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20220310T131625Z&X-Amz-Expires=86400&X-Amz-Signature=e9d1b29af0c124f6ddd2b3e6a87d21c99c9353499bde703601332295acb32fcf&X-Amz-SignedHeaders=host&response-content-disposition=filename%20%3D%22%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA%25202022-03-06%2520%25E1%2584%258B%25E1%2585%25A9%25E1%2584%258C%25E1%2585%25A5%25E1%2586%25AB%252011.17.58.png%22&x-id=GetObject)
기본형 타입으로 선언된 변수들은 바로 값을 가지만 참조형으로 선언된 것들은 값을 바로 가지지 않는다. System.out.println()으로 찍어보면 주소값이 찍힌다.

![preview](https://s3.us-west-2.amazonaws.com/secure.notion-static.com/b95285e5-f492-44d9-aa01-465b85b3bea6/Untitled.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45EIPT3X45%2F20220310%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20220310T131628Z&X-Amz-Expires=86400&X-Amz-Signature=101c9914e459e80b29980137842b4df2f85e57cc1267eff9b99f1a1532b827b7&X-Amz-SignedHeaders=host&response-content-disposition=filename%20%3D%22Untitled.png%22&x-id=GetObject)
기본형 타입으로 선언된 변수들이나 지역변수로 선언된 변수들은 Stack에 값을 가지고 저장이 된다. 메서드 실행에 따라 자동으로 할당/해제가 된다. 재귀로 동일한 메소드를 호출하면 stackOverFlowError가 나는 것을 보면 Stack영역에 계속해서 method() Call을 쌓다보니 저장할 공간이 없어서 터진다.

참조형 타입으로 선언된 변수가 클래스 변수이면 Method Area에 공간이 할당이 되고 일반 변수일 경우 Stack Area에 할당이 된다.

바로 할당이 되는 것은 아니고 new 연산자로 할당을 하면 Heap영역에 값을 저장한 공간을 만든다. 그 후 stack or method Area에 값을 저장한 주소값을 매핑시킨다.
