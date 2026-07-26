---
title: (Kotlin/코틀린) 수 체계 완전 정리 — Int, Long, UInt, ULong 범위와 변환
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin의 정수 타입 Int, Long, Short, Byte, UInt, ULong의 범위와 언더플로우/오버플로우, 타입 변환, 부호 없는 정수 활용법을 정리합니다.
---

---

## 1. 정수 타입 범위

| 타입 | 크기 | 최솟값 | 최댓값 |
|------|------|--------|--------|
| `Byte` | 8bit | -128 | 127 |
| `Short` | 16bit | -32,768 | 32,767 |
| `Int` | 32bit | -2,147,483,648 | 2,147,483,647 (약 21억) |
| `Long` | 64bit | -9,223,372,036,854,775,808 | 9,223,372,036,854,775,807 |
| `UByte` | 8bit | 0 | 255 |
| `UShort` | 16bit | 0 | 65,535 |
| `UInt` | 32bit | 0 | 4,294,967,295 (약 42억) |
| `ULong` | 64bit | 0 | 18,446,744,073,709,551,615 |

---

## 2. 오버플로우 — 조용히 발생

Kotlin은 정수 오버플로우 시 **예외 없이 값이 순환**합니다.

```kotlin
val max = Int.MAX_VALUE  // 2,147,483,647
println(max + 1)         // -2,147,483,648 ← 오버플로우!

val min = Int.MIN_VALUE  // -2,147,483,648
println(min - 1)         // 2,147,483,647 ← 언더플로우!
```

```kotlin
// 오버플로우 방지: Long으로 계산
val bigNum = Int.MAX_VALUE.toLong() + 1
println(bigNum)  // 2,147,483,648 ← 올바른 값

// 또는 리터럴에 L 접미사
val correct = 2_147_483_648L
```

---

## 3. 타입 리터럴

```kotlin
val i: Int   = 100
val l: Long  = 100L       // L 접미사
val u: UInt  = 100u       // u 접미사
val ul: ULong = 100uL     // uL 접미사

val hex = 0xFF            // 16진수 → Int 255
val bin = 0b1010          // 2진수  → Int 10
val big = 1_000_000       // 밑줄 구분자 → 1000000
```

---

## 4. 명시적 타입 변환

Kotlin은 암묵적 형 변환이 없습니다. **명시적 변환 함수**를 사용해야 합니다.

```kotlin
val i: Int = 42
val l: Long = i.toLong()    // Int → Long
val d: Double = i.toDouble() // Int → Double
val b: Byte = i.toByte()    // Int → Byte (범위 초과 시 잘림)
val s: Short = i.toShort()

// 주의: 범위 초과 시 데이터 손실
val big: Int = 300
println(big.toByte())  // 44 ← 300 - 256 = 44 (잘림)
```

```kotlin
// Long → Int 안전한 변환
val longVal = 5_000_000_000L
println(longVal.toInt())            // -705032704 ← 오버플로우!
println(longVal.coerceIn(Int.MIN_VALUE.toLong(), Int.MAX_VALUE.toLong()).toInt())  // 안전
```

---

## 5. 부호 없는 정수 (UInt, ULong)

양수만 다루고 범위를 최대화해야 할 때 사용합니다.

```kotlin
val maxUInt: UInt = UInt.MAX_VALUE
println(maxUInt)  // 4294967295 (약 42억 — Int 최대의 2배)

val maxULong: ULong = ULong.MAX_VALUE
println(maxULong)  // 18446744073709551615

// 비트 조작 — 부호 없는 정수가 더 직관적
val colorRaw: UInt = 0xFFAABBCCu
println(colorRaw)  // 4289003468
```

### 부호 있는 정수 ↔ 부호 없는 정수 변환

```kotlin
val signed: Int = -1
val unsigned: UInt = signed.toUInt()
println(unsigned)  // 4294967295 (비트 패턴 그대로 해석)

val u: UInt = 4294967295u
val s: Int = u.toInt()
println(s)  // -1
```

---

## 6. 연산 시 타입 규칙

```kotlin
// 더 큰 타입으로 자동 승격
val i: Int = 10
val l: Long = 20L
// val sum = i + l  // 컴파일 에러: 명시적 변환 필요
val sum = i.toLong() + l  // 30L

// 같은 타입끼리는 자동 처리
val a = 10   // Int
val b = 20   // Int
val c = a + b  // Int 30

// 나눗셈 주의: 정수 / 정수 = 정수
println(7 / 2)    // 3 (소수점 버림)
println(7.0 / 2)  // 3.5
println(7 / 2.0)  // 3.5
```

---

## 7. 실전 — 타입 선택 가이드

| 상황 | 권장 타입 |
|------|----------|
| 일반 정수 (21억 이하) | `Int` |
| 타임스탬프, 파일 크기 | `Long` |
| 비트 플래그, 색상값 | `UInt` 또는 `Int` (비트 연산) |
| ID 값 (DB auto increment) | `Long` |
| 루프 카운터 | `Int` |
| 메모리 절약 필요 (소규모 값) | `Byte` / `Short` |

---

## 8. 정리

- `Int` 범위: 약 ±21억, 초과 시 `Long` 사용
- 오버플로우는 예외 없이 조용히 발생 — 큰 값 계산 시 주의
- Kotlin은 암묵적 변환 없음 → `toLong()`, `toInt()` 등 명시적 변환 필수
- `UInt`/`ULong`: 양수 전용, 범위 약 2배, 비트 조작에 직관적
- 정수 나눗셈은 소수점 버림 → 실수 계산 필요 시 하나를 Double로 변환
