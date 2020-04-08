---
title: Java 예외처리 try catch finally 사용법
tags: [Java]
style: fill
color: dark
description: 자바의 예외처리 방법 중의 하나인 try-catch 문을 알아보자
---

## 에외 vs 오류
자바에서 **오류**는 실행중에 발생하는 문제로 처리가 불가능한 문제입니다. **예외**는 실행중에 발생하는 문제이지만 개발자가 예측을 하여 처리를 할 수 있습니다. 이를 예외처리라 부르고 try-cath문 기능을 제공합니다.


## try-catch-finally

아래의 코드를 살펴보자.
```javascript
try {
    int num = 4/0;
} catch(ArithmeticException e) {
    e.printStackTrace();
} catch(Exception e) {
    e.printStackTrace();
} finally {
    System.out.println("Finally Called");
}
```
- try 안의 코드를 실행하고 예외가 발생하면 catch() exception에 맞는 코드가 실행이 된다. 여기에서는 4/0으로 나누었기에 ArithmeticException 이 발생한다.
- catch 문을 사용할 떄 주의할 점은 가장 하위의 클래스부터 사용해야 하는점이다.Exception 은 가장 상위의 클래스이므로 위의 놔두면 매번 해당 Exception 만 호출되기에 주의해야 한다.
- finally 는 try-catch 가 실행되면 반드시 실행되는 구문이다. 예외가 발생하더라도 코드가 실행되고 관련 문제가 여러 IT 필기시험에 출제된다.

```javascript
try {
    System.exit(0);
} catch(ArithmeticException e) {
    e.printStackTrace();
} finally {
    System.out.println("Finally Called 2");
}
```
- 여기에서 finally 구문은 실행이 될까? System.exit() 가 불릴 때는 finally가 호출되지 않고 시스템이 아예 꺼진다. 이때 빼고는 다 finally 구문은 반드시 실행이 된다.


```javascript
try {
    db.open()
    network.connect();
} catch(ArithmeticException | ArrayIndexOutOfBoundsException e) {

} finally() {
    db.close()
    network.disconnect();
}
```
- 사실 try-catch 는 db 연결, 네트워크 연결, 스트림 등에서 자주 사용한다. DB 열고 닫고 네트워크 연결하고 끊어주고를 try-catch문을 사용하라고 자바 컴파일러가 친절하게 알려준다.
- 1.7버전부터 catch 문에 '|' 를 사용하여 여러 예외처리를 한번에 할 수 있다.

## 결론
- **전공 필기시험에 try-catch 실행 결과에 대한 문제가 자주 출제된다.**