---
title: Java 쓰레드 교착상태(Deadlock) 완벽 이해 및 예제
tags: [ Java ]
style: fill
color: dark
description: Java 쓰레드 교착상태(Deadlock) 완벽 이해 및 예제
---

## ✨ 개요

멀티쓰레드 프로그래밍에서 피할 수 없는 난관 중 하나가 바로 **교착상태(Deadlock)**입니다.
오늘은 Java에서 Deadlock이 무엇인지, 발생 원인, 예제, 해결 방법까지 확실히 정리해드리겠습니다.

---

## 1️⃣ 교착상태(Deadlock)란?

Deadlock(교착상태)은 둘 이상의 쓰레드가 서로가 가지고 있는 락을 풀지 않아서 영원히 기다리는 상태를 말합니다. 
즉, 서로 자원을 기다리며 무한 대기에 빠진 상황

---

## 2️⃣ 교착상태가 발생하는 4가지 조건

| 조건         | 설명                     |
| ---------- | ---------------------- |
| **상호배제**   | 한 번에 하나의 쓰레드만 자원 사용 가능 |
| **점유와 대기** | 자원을 점유한 채로 다른 자원을 기다림  |
| **비선점**    | 자원을 강제로 뺏을 수 없음        |
| **환형 대기**  | 여러 쓰레드가 원형으로 자원을 대기    |
이 4가지 조건이 동시에 만족될 때 교착상태 발생

---

## 3️⃣ 교착상태 예제 코드 (Java)

```java
class SharedResource {
    synchronized void methodA(SharedResource other) {
        System.out.println(Thread.currentThread().getName() + " methodA 진입");
        try { Thread.sleep(100); } catch (InterruptedException e) {}
        other.methodB(this);
    }

    synchronized void methodB(SharedResource other) {
        System.out.println(Thread.currentThread().getName() + " methodB 진입");
    }
}

public class DeadlockExample {
    public static void main(String[] args) {
        SharedResource resource1 = new SharedResource();
        SharedResource resource2 = new SharedResource();

        Thread t1 = new Thread(() -> resource1.methodA(resource2), "Thread-1");
        Thread t2 = new Thread(() -> resource2.methodA(resource1), "Thread-2");

        t1.start();
        t2.start();
    }
}
```
결과
- Thread-1은 resource1 락 보유, resource2 대기
- Thread-2는 resource2 락 보유, resource1 대기

→ 서로 락을 양보하지 않고 영원히 대기 → Deadlock 발생

---

## 4️⃣ 교착상태 해결 방뻡

### 1. 락 획득 순서 일관성 유지

```java
synchronized (resource1) {
    synchronized (resource2) {
        // 작업
    }
}
```

### 2. tryLock 사용 (Timeout 활용)

```java
if (lock1.tryLock(100, TimeUnit.MILLISECONDS)) {
    try {
        if (lock2.tryLock(100, TimeUnit.MILLISECONDS)) {
            try {
                // 작업
            } finally {
                lock2.unlock();
            }
        }
    } finally {
        lock1.unlock();
    }
}
// ReentrantLock의 tryLock(timeout) 사용 -> 교착상태 방지
```

### 3. 최소한의 락 사용
- 락을 너무 많이 걸지 말고 필요한 범위 최소화

---

## 5️⃣ 교착상태 vs 일반 동기화 문제 차이

| 구분    | 교착상태 (Deadlock)    | 일반 동기화 문제              |
| ----- | ------------------ | ---------------------- |
| 원인    | 서로 락 점유 상태로 무한 대기  | 락의 범위 미흡으로 인한 동시 접근 문제 |
| 해결법   | 락 순서 통일, tryLock 등 | 락 추가 또는 범위 조정          |
| 문제 상황 | 프로그램이 멈춤           | 데이터 꼬임, 예상치 못한 동작      |

---

## 6️⃣  결론

- 교착상태는 락이 서로 기다리며 영원히 대기하는 상태
- 4가지 발생 조건(상호배제, 점유와 대기, 비선점, 환형대기)을 기억할 것
- 순서 통일, tryLock 등으로 예방 가능