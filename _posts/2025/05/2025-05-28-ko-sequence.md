---
title: 객체지향프로그래밍(OOP)의 핵심 특징 4가지 완벽 정리
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin Sequence를 활용한 지연 연산과 효율적인 컬렉션 처리 방식. filter, map, takeWhile 등 실전 예제로 배우는 Sequence의 모든 것.
---

## ✨ 개요

Kotlin에서 대량의 데이터를 처리하거나 체이닝 연산을 최적화하고 싶을 때, **Sequence**는 강력한 해결책이 됩니다.

컬렉션(`List`, `Set`)은 연산마다 전체 데이터가 메모리에 로드되지만,  
**`Sequence`는 지연 연산(Lazy Evaluation)**을 통해 한 번에 하나씩 요소를 처리해 메모리와 성능을 절약합니다.

---

## 1. ✅ Sequence란?

### ✅ 정의
- Kotlin의 **지연 컬렉션 처리 API**
- 중간 연산(`map`, `filter`)은 즉시 실행되지 않고, **최종 연산(`toList`, `count`, `find`) 시에만 실행**됩니다.
- Java 8 Stream과 유사한 구조

---

## 2. ✅ Sequence vs 일반 컬렉션

| 항목            | Collection (`List`, `Set`) | `Sequence`           |
|-----------------|-----------------------------|-----------------------|
| **연산 방식**     | 즉시 실행                   | 지연 실행 (lazy)       |
| **성능 (대량 처리)** | 비효율적                     | 효율적                  |
| **중간 연산 처리** | 모든 요소에 순차 적용        | 필요할 때만 적용       |
| **메모리 사용**   | 전체 로드                   | 한 항목씩 처리          |
| **예시 사용처**   | 소량의 고정된 리스트          | 대용량 or 체이닝 연산   |

- 성능 차이
  + ```kotlin
    import kotlin.system.measureTimeMillis

    fun main() {
    val list = (1..1_000_000).toList()
    
        val collectionTime = measureTimeMillis {
            list
                .filter { it % 2 == 0 }
                .map { it * 2 }
                .take(100)
                .forEach {}
        }
    
        val sequenceTime = measureTimeMillis {
            list.asSequence()
                .filter { it % 2 == 0 }
                .map { it * 2 }
                .take(100)
                .forEach {}
        }
    
        println("Collection 처리 시간: ${collectionTime}ms") // Collection 처리 시간: 80ms
        println("Sequence 처리 시간: ${sequenceTime}ms") // Sequence 처리 시간: 8ms
    }
    ```
    + 대용량 일수록 시퀀스를 사용하는 것이 성능에서 유리함 (에제코드에서 10배 차이)

---

## 3 ✅. 기본 사용법

```kotlin
val numbers = listOf(1, 2, 3, 4, 5)

// 일반 컬렉션 방식
val doubledList = numbers.map { it * 2 }.filter { it > 5 }

// Sequence 방식
val doubledSeq = numbers.asSequence()
    .map { it * 2 }
    .filter { it > 5 }
    .toList()
```
- 💡asSequence() 로 Sequence 변환 후, 연산을 체이닝합니다.

---

## 4 ✅. Sequence의 주요 함수 예제

### 4.1 map + filter + tak

```kotlin
val result = generateSequence(1) { it + 1 } // 무한 시퀀스
  .map { it * 2 }
  .filter { it % 3 == 0 }
  .take(5)
  .toList()

println(result) // [6, 12, 18, 24, 30]
```

### 4.2 takeWhile

```kotlin
val result = sequenceOf(1, 2, 3, 10, 20, 30)
    .takeWhile { it < 10 }
    .toList()

println(result) // [1, 2, 3]
```

### 4.3 zipWithNext

```kotlin
val result = sequenceOf(1, 2, 4, 7).zipWithNext { a, b -> b - a }
println(result) // [1, 2, 3]
```

---

## 5.🧠 **Sequence** 주의사항

- 중간 연산만 사용하면 아무 일도 일어나지 않음 (최종 연산 필요)
- 반복(iteration)은 한 번만 가능 (재사용 X)
- 짧은 리스트에선 일반 컬렉션이 더 빠를 수 있음

- Sequence 연산 부류
  + | 구분        | 함수 예시                                | 특징           |
    | --------- | ------------------------------------ | ------------ |
    | **중간 연산** | `map`, `filter`, `take`, `sorted` 등  | 지연됨, 체이닝 가능  |
    | **최종 연산** | `toList`, `count`, `first`, `find` 등 | 즉시 실행, 결과 리턴 |

---

## 6.🧠 **결론**

- Sequence는 지연 연산 기반의 효율적인 컬렉션 처리 방식
- 대량의 데이터를 처리하거나 filter-map-take와 같은 체이닝이 많을 때 유리
- generateSequence, asSequence, takeWhile 등 함수 조합을 잘 활용하면 성능 최적화에 매우 효과적입니다