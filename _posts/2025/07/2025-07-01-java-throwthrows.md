---
title: Java throw vs throws 예외처리 가이드
tags: [ Java ]
style: fill
color: dark
description: Java 의 예외 처리에서 자주 혼동되는 throw 와 throws 차이점과 사용법을 예제 중심으로 설명합니다
---

## ✨ 개요

Java 에서 예외(Exception)는 프로그램 오류를 처리하고 복구할 수 있는 강력한 구조입니다. 
특히 `throw` 와 `throws` 는 예외를 발생시키고 선언하는 역할로 구분되며 헷갈리기 쉬운 개념입니다.

---

## 1. ✅ throws 란?

- 직접 예외 객체를 발생시키는 키워드
- 런타임 시점에서 특정 조건에 맞으면 예외를 던짐

```kotlin
if (amount < 0) {
    throw new IllegalArgumentException("음수는 허용되지 않습니다")
}
```

---

## 2. ✅ throws 란?

- 메서드 시그니처에 예외가 발생할 수 있음을 선언
- 실제 예외 처리는 메서드를 호출한 쪽에서 수행하도록 위임

```java
public void readFile(String path) throws IOException {
    FileReader reader = new FileReader(path);
}
```

---

## 3 ✅. throws vs throws 차이점 요약

| 구분       | throw                                 | throws                                 |
|------------|---------------------------------------|----------------------------------------|
| 의미       | 예외를 발생                            | 예외 발생 가능성을 선언                  |
| 위치       | 메서드, 블록 내부                       | 메서드 시그니처                          |
| 예외 처리  | 즉시 발생                              | 호출한 곳에서 처리                       |

---

## 4.🧠 🧾 결론

- throws 는 예외를 즉시 발생시키는 키워드
- throws 는 예외가 발생할 수 있음을 선언하는 키워드
- 두 개념을 명확히 구분해 예외 처리를 설계하면 코드의 안전성과 가독성을 높일 수 있음