---
title: Java Checked Exception & UnChecked Exception 이란?
tags: [ Java ]
style: fill
color: dark
description: Java Checked Exception & UnChecked Exception 이란?
---

## ✨ 개요

Java를 공부하다 보면 Exception이라는 단어는 귀에 딱지가 앉을 정도로 듣게 됩니다. 
그 중에서도 가장 많이 헷갈리는 개념이 바로 Checked Exception과 Unchecked Exception의 차이입니다.
오늘 이 포스팅을 통해 명확하게 정리해봅시다.

---

## 1️⃣ 예외(Exception)란?

- 프로그램 실행 중 발생할 수 있는 비정상적인 상황
- Java 는 예외를 클래스 계층 구조로 관리
- 모든 예외의 최상위는 Throwable

```java
Throwable
 ├── Error        (시스템 심각 오류, 개발자가 처리하지 않음)
 └── Exception    (개발자가 처리해야 함)
      ├── Checked Exception
      └── Unchecked Exception (RuntimeException)
```

---

## 2️⃣ Checked Exception 이란?

> "컴파일러가 확인하는 예외"로 강제 처리 필요 (`try-catch` or `trhows`)

대표적으로
- IOException
- SQLException
- FileNotFoundException

```java
public void readFile() throws IOException {
    FileReader reader = new FileReader("data.txt"); // 반드시 예외 처리 필요
}
```

| 구분 | Checked Exception |
| -- | ----------------- |
| 검증 | 컴파일러가 검증          |
| 처리 | 반드시 예외 처리         |
| 발생 | 외부 환경 (파일, DB 등)  |
| 목적 | 안정적인 예외 회복        |

---

## 3️⃣ UnChecked Exception 이란?

> 컴파일러가 검사하지 않는 예외로 런타임 중 발생하고 개발자가 원할떄만 처리

대표적으로
- NullPointerException
- ArrayIndexOutOfBoundsException
- IllegalArgumentException

```java
public void divide(int a, int b) {
    int result = a / b; // b가 0이면 ArithmeticException 발생
}
```

| 구분 | Unchecked Exception |
| -- | ------------------- |
| 검증 | 컴파일러가 검사 안 함        |
| 처리 | 선택 사항               |
| 발생 | 주로 프로그래밍 실수         |
| 목적 | 명확한 코드 작성 유도        |

---

## 4️⃣ 왜 나누었을까?

| 구분       | Checked        | Unchecked       |
| -------- | -------------- | --------------- |
| 책임       | 외부 환경 대비       | 개발자 실수 방지       |
| 코드 의도 명확 | 예외를 반드시 명시     | 예외를 선택적으로 처리 가능 |
| 예시       | 파일, DB, 네트워크 등 | Null, 잘못된 인덱스 등 |

---

## 5️⃣ 결론

| 구분    | Checked Exception | Unchecked Exception |
| ----- | ----------------- | ------------------- |
| 검사    | 컴파일러 필수 확인        | 런타임에서 발생            |
| 의도    | 예외 복구 의도          | 프로그래밍 오류 방지         |
| 사용 추천 | 외부 자원             | 논리적 실수 (null 등)     |