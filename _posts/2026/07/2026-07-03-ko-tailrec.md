---
title: (Kotlin/코틀린) 꼬리 재귀 tailrec — 스택 오버플로우 방지
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin의 tailrec 키워드로 꼬리 재귀를 반복문으로 최적화하는 원리와 조건, 일반 재귀와의 성능 비교를 정리합니다.
---

---

## 1. 재귀의 문제 — 스택 오버플로우

재귀 함수는 호출할 때마다 **콜 스택에 프레임을 쌓습니다.**  
깊이가 깊어지면 스택 메모리가 고갈되어 `StackOverflowError`가 발생합니다.

```kotlin
fun factorial(n: Long): Long {
    if (n == 1L) return 1L
    return n * factorial(n - 1)  // 스택에 n개의 프레임 쌓임
}

factorial(100_000)  // StackOverflowError!
```

```
factorial(5)
  factorial(4)
    factorial(3)
      factorial(2)
        factorial(1) → 1
      → 2
    → 6
  → 24
→ 120
// 호출 깊이만큼 스택 프레임 유지됨
```

---

## 2. 꼬리 재귀(Tail Recursion)란?

**재귀 호출이 함수의 마지막 연산**인 경우를 꼬리 재귀라고 합니다.  
꼬리 재귀는 현재 스택 프레임을 재사용할 수 있어 이론적으로 스택이 쌓이지 않습니다.

```kotlin
// 일반 재귀: 재귀 호출 후 곱셈 연산이 남아있음 → 꼬리 재귀 아님
fun factorial(n: Long): Long {
    if (n == 1L) return 1L
    return n * factorial(n - 1)  // factorial 반환 후 * n 연산이 남음
}

// 꼬리 재귀: 재귀 호출이 마지막 연산
fun factorialTail(n: Long, acc: Long = 1L): Long {
    if (n == 1L) return acc
    return factorialTail(n - 1, acc * n)  // 이것만 하고 끝 → 꼬리 재귀
}
```

누적값(`acc`)을 파라미터로 전달해 **중간 계산 결과를 다음 호출로 넘기는** 패턴이 핵심입니다.

---

## 3. tailrec 키워드

Kotlin은 `tailrec` 키워드로 꼬리 재귀를 **반복문(while loop)으로 자동 변환**합니다.  
JVM은 꼬리 재귀 최적화를 언어 수준에서 지원하지 않기 때문에 Kotlin 컴파일러가 직접 변환합니다.

```kotlin
tailrec fun factorialTail(n: Long, acc: Long = 1L): Long {
    if (n == 1L) return acc
    return factorialTail(n - 1, acc * n)
}

factorialTail(100_000)  // StackOverflowError 없음
```

컴파일 후 실제로 생성되는 바이트코드는 아래와 동일합니다.

```kotlin
fun factorialLoop(n: Long, acc: Long = 1L): Long {
    var n = n
    var acc = acc
    while (true) {
        if (n == 1L) return acc
        acc *= n
        n -= 1
    }
}
```

---

## 4. tailrec 사용 조건

`tailrec`이 적용되려면 다음 조건을 모두 만족해야 합니다.

| 조건 | 설명 |
|------|------|
| 마지막 연산이 재귀 호출 | 재귀 호출 후 추가 연산 없어야 함 |
| 자기 자신만 호출 | 간접 재귀(A→B→A) 불가 |
| try-catch 외부 | try 블록 안의 꼬리 호출은 최적화 불가 |
| open/override 불가 | 가상 메서드는 최적화 불가 |

```kotlin
// 컴파일 경고: 꼬리 재귀가 아닌 경우 tailrec 무시됨
tailrec fun bad(n: Int): Int {
    if (n == 0) return 0
    return 1 + bad(n - 1)  // bad() 호출 후 + 1 연산이 남음 → 꼬리 재귀 아님
}
// 경고: A function is marked as tail-recursive but no tail calls are found
```

---

## 5. 실전 예제

### 피보나치 수열

```kotlin
// 일반 재귀: 지수 시간 복잡도 + 스택 오버플로우
fun fib(n: Int): Long {
    if (n <= 1) return n.toLong()
    return fib(n - 1) + fib(n - 2)  // 꼬리 재귀 아님
}

// tailrec 버전
tailrec fun fib(n: Int, a: Long = 0L, b: Long = 1L): Long {
    if (n == 0) return a
    return fib(n - 1, b, a + b)
}

fib(100)   // 354224848179261915075
fib(1_000) // 스택 오버플로우 없음
```

### 링크드 리스트 순회

```kotlin
data class Node<T>(val value: T, val next: Node<T>? = null)

tailrec fun <T> find(node: Node<T>?, target: T): Node<T>? {
    if (node == null || node.value == target) return node
    return find(node.next, target)
}
```

### 이진 탐색

```kotlin
tailrec fun binarySearch(
    list: List<Int>,
    target: Int,
    low: Int = 0,
    high: Int = list.size - 1
): Int {
    if (low > high) return -1
    val mid = (low + high) / 2
    return when {
        list[mid] == target -> mid
        list[mid] < target  -> binarySearch(list, target, mid + 1, high)
        else                -> binarySearch(list, target, low, mid - 1)
    }
}
```

---

## 6. 일반 재귀 vs tailrec 비교

| 구분 | 일반 재귀 | tailrec |
|------|---------|---------|
| 스택 사용 | 깊이만큼 증가 | 상수 (O(1)) |
| StackOverflowError | 깊이 제한 존재 | 없음 |
| 성능 | 스택 프레임 생성 비용 | 반복문과 동일 |
| 가독성 | 수학적 정의에 가까움 | 누적값 파라미터 추가 |
| 컴파일 결과 | 재귀 호출 그대로 | while 루프로 변환 |

---

## 7. 정리

- `tailrec`은 꼬리 재귀를 **반복문으로 컴파일 타임에 변환**해 스택 오버플로우를 방지
- 조건: 재귀 호출이 함수의 **마지막 연산**이어야 하며, 자기 자신만 호출
- 누적값(`acc`) 파라미터 패턴으로 일반 재귀를 꼬리 재귀로 전환
- 피보나치, 팩토리얼, 리스트 순회, 이진 탐색 등에 실전 적용 가능
