---
title: (Kotlin/코틀린) 비트 연산자 — and, or, xor, shl, shr
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin의 비트 연산자 and, or, xor, inv, shl, shr, ushr의 동작 원리와 권한 플래그, 색상 처리, 최적화 등 실전 활용 패턴을 정리합니다.
---

---

## 1. Kotlin 비트 연산자 목록

Kotlin은 Java의 `&`, `|`, `^`, `~`, `<<`, `>>`, `>>>` 대신 **함수 이름** 형태를 사용합니다.

| Kotlin | Java | 설명 |
|--------|------|------|
| `a and b` | `a & b` | AND: 둘 다 1이면 1 |
| `a or b` | `a \| b` | OR: 하나라도 1이면 1 |
| `a xor b` | `a ^ b` | XOR: 다르면 1 |
| `a.inv()` | `~a` | NOT: 비트 반전 |
| `a shl n` | `a << n` | 왼쪽 시프트 |
| `a shr n` | `a >> n` | 오른쪽 시프트 (부호 유지) |
| `a ushr n` | `a >>> n` | 오른쪽 시프트 (부호 무시) |

---

## 2. 기본 동작

```kotlin
val a = 0b1010  // 10
val b = 0b1100  // 12

println(a and b)   // 0b1000 = 8   (둘 다 1인 비트만)
println(a or b)    // 0b1110 = 14  (하나라도 1인 비트)
println(a xor b)   // 0b0110 = 6   (서로 다른 비트)
println(a.inv())   // -11           (모든 비트 반전)
println(a shl 1)   // 0b10100 = 20 (1비트 왼쪽 = ×2)
println(a shr 1)   // 0b0101 = 5   (1비트 오른쪽 = ÷2)
```

---

## 3. 실전 패턴 — 권한 플래그

비트 플래그로 여러 권한/옵션을 하나의 Int에 효율적으로 저장합니다.

```kotlin
object Permission {
    const val NONE   = 0b0000  // 0
    const val READ   = 0b0001  // 1
    const val WRITE  = 0b0010  // 2
    const val DELETE = 0b0100  // 4
    const val ADMIN  = 0b1000  // 8
    const val ALL    = READ or WRITE or DELETE or ADMIN  // 15
}

// 권한 부여
var userPermission = Permission.NONE
userPermission = userPermission or Permission.READ
userPermission = userPermission or Permission.WRITE
// userPermission = 0b0011 = 3

// 권한 확인
fun hasPermission(permission: Int, flag: Int): Boolean = (permission and flag) != 0
println(hasPermission(userPermission, Permission.READ))    // true
println(hasPermission(userPermission, Permission.DELETE))  // false

// 권한 제거
userPermission = userPermission and Permission.WRITE.inv()
// WRITE 비트만 0으로 → READ만 남음
println(hasPermission(userPermission, Permission.WRITE))   // false
```

---

## 4. 실전 패턴 — 색상 처리 (ARGB)

Android의 색상값(ARGB)은 32비트 정수로 표현됩니다.

```kotlin
val color = 0xFF_AA_BB_CC.toInt()  // 0xAA: Alpha, 0xBB: Red, 0xCC: Green (예시 형식)

// 각 채널 추출
val alpha = (color ushr 24) and 0xFF
val red   = (color ushr 16) and 0xFF
val green = (color ushr  8) and 0xFF
val blue  =  color          and 0xFF

println("A=$alpha R=$red G=$green B=$blue")

// 색상 조합
fun argb(a: Int, r: Int, g: Int, b: Int): Int =
    (a shl 24) or (r shl 16) or (g shl 8) or b

val white = argb(255, 255, 255, 255)
val red50 = argb(128, 255, 0, 0)  // 50% 투명 빨강
```

---

## 5. 실전 패턴 — 2의 거듭제곱 최적화

```kotlin
// 2의 거듭제곱 나누기/곱하기
val n = 1024
println(n shr 1)   // 512  (÷2)
println(n shr 2)   // 256  (÷4)
println(n shl 1)   // 2048 (×2)
println(n shl 3)   // 8192 (×8)

// 짝수 홀수 판별 (% 2보다 빠름)
fun isEven(n: Int) = (n and 1) == 0
fun isOdd(n: Int) = (n and 1) != 0

// 2의 거듭제곱 여부 확인
fun isPowerOfTwo(n: Int) = n > 0 && (n and (n - 1)) == 0
println(isPowerOfTwo(16))  // true
println(isPowerOfTwo(15))  // false
```

---

## 6. shr vs ushr

```kotlin
val negative = -8  // 부호 있는 음수

println(negative shr 1)   // -4  (부호 유지: 최상위 비트 1로 채움)
println(negative ushr 1)  // 2147483644  (부호 무시: 최상위 비트 0으로 채움)
```

| 연산자 | 용도 |
|--------|------|
| `shr` | 일반적인 나눗셈 (부호 보존) |
| `ushr` | 비트 조작 (부호 없는 처리, 색상값 등) |

---

## 7. 정리

- Kotlin 비트 연산자는 기호(`&`, `|`) 대신 **이름 형태** (`and`, `or`, `xor`, `inv`, `shl`, `shr`, `ushr`)
- 권한 플래그: `or`로 부여, `and`로 확인, `and inv()`로 제거
- 색상 처리: `ushr`로 채널 분리, `shl or`로 채널 조합
- `shr`는 부호 유지, `ushr`는 논리 시프트 (색상·비트 조작에 사용)
- 2의 거듭제곱 연산은 시프트 연산이 나눗셈보다 빠름
