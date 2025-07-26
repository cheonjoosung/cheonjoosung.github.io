---
title: Java Atomic 클래스 완벽 정리
tags: [ Java ]
style: fill
color: dark
description: Java Atomic 클래스 완벽 정리
---

## ✨ 개요

멀티쓰레드 프로그래밍을 하다 보면, 여러 쓰레드가 하나의 변수를 동시에 변경하면서 동기화 문제가 발생하게 됩니다. 
이를 해결하는 대표적인 방법 중 하나가 바로 Java의 Atomic 클래스입니다. 이번 포스팅에서는 Atomic 클래스의 개념, 종류, 사용법, 장단점, 예제까지 모두 정리해드립니다.

---

## 1️⃣ Java Atomic 클래스란?

Atomic 클래스는 멀티쓰레드 환경에서 안전하게 값을 변경하기 위한 클래스입니다.
이 클래스들은 CAS (Compare-And-Swap) 연산 기반으로 동작하여, synchronized 없이도 안전하게 값을 변경할 수 있습니다.
> ✅ Atomic 클래스의 모든 연산은 원자적(Atomic)으로 수행되어 동시성 문제를 예방합니다.

---

## 2️⃣ 대표적인 Atomic 클래스 종류

| 클래스                  | 설명              |
| -------------------- | --------------- |
| `AtomicInteger`      | `int` 타입 전용     |
| `AtomicLong`         | `long` 타입 전용    |
| `AtomicBoolean`      | `boolean` 타입 전용 |
| `AtomicReference<T>` | 참조 타입 전용        |

---

## 3️⃣ 주요 메서드 예시 (AtomicInteger 기준)

| 메서드                             | 설명          |
| ------------------------------- | ----------- |
| `get()`                         | 현재 값 반환     |
| `set(value)`                    | 값 설정        |
| `getAndIncrement()`             | 값 반환 후 +1   |
| `incrementAndGet()`             | +1 후 반환     |
| `compareAndSet(expect, update)` | 기대값과 같으면 변경 |

---

## 4️⃣ 사용 예제

### 1. AtomicInteger

```java
import java.util.concurrent.atomic.AtomicInteger;

public class AtomicExample {
    private static AtomicInteger counter = new AtomicInteger(0);

    public static void main(String[] args) throws InterruptedException {
        Runnable task = () -> {
            for (int i = 0; i < 1000; i++) {
                counter.incrementAndGet(); // 동기화 필요 없음
            }
        };

        Thread t1 = new Thread(task);
        Thread t2 = new Thread(task);
        t1.start();
        t2.start();
        t1.join();
        t2.join();

        System.out.println("최종 값: " + counter.get()); // 2000 예상
    }
}
```

### 2. AtomicReference

```java
import java.util.concurrent.atomic.AtomicReference;

class User {
    String name;
    User(String name) { this.name = name; }
}

AtomicReference<User> user = new AtomicReference<>(new User("Alice"));
user.set(new User("Bob"));
System.out.println(user.get().name); // Bob
```
- 객체 참조 변경에 사용

---

## 5️⃣ Atomic 클래스가 필요한 이유

| 기존 방식                  | 문제점             |
| ---------------------- | --------------- |
| `int` + `synchronized` | 성능 저하 (락 사용)    |
| `AtomicInteger`        | 락 없이 고성능 동시성 보장 |

---

## 6️⃣  결론

- 동기화 없이 안전하게 값 변경 가능
- CAS 기반으로 동시성 처리, 성능 우수
- 단일 변수에 적합, 복합 연산은 별도 동기화 필요
- 멀티스레드 환경에서 필수적으로 알아야 할 클래스