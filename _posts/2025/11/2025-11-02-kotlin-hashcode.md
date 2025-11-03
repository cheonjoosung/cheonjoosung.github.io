---
title: Kotlin/코틀린 일반 클래스에서 hashCode() 제대로 구현하기
tags: [Kotlin]
style: fill
color: dark
description: Kotlin/코틀린 일반 클래스에서 `hashCode()` 제대로 구현하기 — equals 계약, 충돌 최소화, 실전 레시피
---

## ✨ 개요

일반 클래스에서 `hashCode()` 제대로 구현하기 — equals 계약, 충돌 최소화, 실전 레시피

`data class`는 `equals`/`hashCode`를 자동 생성하지만, **일반 클래스(비–data)** 에서는 우리가 직접 `hashCode()`를 구현해야 할 때가 많습니다.  
이 글은 **언제/왜/어떻게** `hashCode()`를 구현해야 하는지, 실전 레시피와 주의점을 한 번에 정리합니다.

---

## 1. 요약

- **계약**: `a == b`이면 반드시 `a.hashCode() == b.hashCode()`여야 합니다. (역은 필수 아님)
- **기본값**: 아무것도 오버라이드하지 않으면 `Any.hashCode()` → **객체 식별자 기반(참조 동일성)** 입니다. 값 동등성을 원하면 반드시 재정의.
- **핵심 필드만 사용**: `equals`에 참여하는 **불변 필드**들로만 계산하세요.
- **배열/부동소수/Null** 등 **특수 타입**은 전용 처리 필요 (`contentHashCode()`, `toBits()`, null-safe).
- **해시 캐싱**은 **진짜 불변 객체**에만 고려.

---

## 2 `equals`/`hashCode` 계약 (필수 이해)

- `a == b` ⇒ `a.hashCode() == b.hashCode()` (같은 객체면 같은 해시)
- 같은 해시라고 해서 반드시 `==`는 아님(충돌 가능).
- `hashCode()`는 **실행 중 일관성**이 있어야 합니다: `equals` 결과가 변하지 않았다면 해시도 변하면 안 됩니다.  
  → **키로 쓰는 객체를 컬렉션에 넣은 후 필드를 바꾸지 마세요.**

---

## 3 언제 오버라이드해야 하나?

- 이 객체를 `HashSet`, `HashMap`의 **키**로 쓰거나,
- **값 동등성**(내용이 같으면 같다고 보고 싶을 때)을 정의한 경우.
- 그렇지 않고 **식별성(참조)** 기준이면 기본 구현(오버라이드 X)으로 충분.

> 값 동등성이 목적이라면 `data class`가 기본 해답입니다. 다만 **맞춤 동등성**(일부 필드만 비교·계산 등)이 필요하면 일반 클래스에서 수동 구현이 낫습니다.

---

## 4. 기본 레시피(가장 흔한 패턴)

```kotlin
class User(
    val id: Long,
    val email: String?,
    val name: String,
    val flags: Int
) {
    override fun equals(other: Any?): Boolean {
        if (this === other) return true
        if (other !is User) return false
        return id == other.id &&
               email == other.email &&
               name == other.name &&
               flags == other.flags
    }

    override fun hashCode(): Int {
        var result = id.hashCode()
        result = 31 * result + (email?.hashCode() ?: 0)
        result = 31 * result + name.hashCode()
        result = 31 * result + flags
        return result
    }
}
```
- 소수 배수(31) 를 곱하며 누적 → 분포 개선(전통적인 자바 관례).
- null은 ?: 0으로 안전 처리.
- equals에 포함한 필드만 해시에 포함하세요.

---

## 5. 타입별 처리 요령

| 타입                             | 해시코드 계산                                                                 |
| ------------------------------ | ----------------------------------------------------------------------- |
| `Int`, `Boolean`, `Char`       | 그대로 사용 (`true`→1, `false`→0 권장)                                         |
| `Long`                         | `id.hashCode()`(JVM: `(id xor (id ushr 32)).toInt()`)                   |
| `Float`                        | `value.toBits()` 사용(예: `31*.. + value.toBits()`)                        |
| `Double`                       | `value.toBits().hashCode()`                                             |
| `String`, `List`, `Set`, `Map` | 각 타입의 `hashCode()`는 **내용 기반**이라 그대로 사용 OK                               |
| **배열(Array, IntArray 등)**      | `contentHashCode()` / `contentEquals()` 필수 (기본 `hashCode()`는 **참조 기반**) |
| `Enum`                         | 기본 `hashCode()` 사용                                                      |
| `Nullable`                     | `x?.hashCode() ?: 0`                                                    |
| 복합 객체                          | 그 객체의 동등성/해시 구현에 신뢰 기반                                                  |

```kotlin
override fun hashCode(): Int {
    var r = 1
    r = 31 * r + items.contentHashCode()   // ★ 배열은 contentHashCode
    r = 31 * r + (tag?.hashCode() ?: 0)
    return r
}

// 부동소수
val fBits = price.toBits()
result = 31 * result + fBits
```

---

## 6. 자바 유틸을 써도 되나? (Objects.hash)

JVM 한정으로 java.util.Objects.hash(a, b, c)를 써서 간단히 만들 수 있지만,
- varargs로 배열 생성 비용(GC)이 있고,
- 배열의 내용 해시가 필요한 경우엔 또 별도 처리해야 합니다.
- 성능/명확성 관점에서 직접 31 패턴으로 합성하는 걸 권장합니다.

---

## 7. 흔한 실수 체크리스트

- equals만 오버라이드하고 hashCode를 빼먹음 → HashMap/HashSet 오동작
- equals와 hashCode에 서로 다른 필드 집합 사용
- 배열을 contentHashCode() 대신 기본 hashCode()로 사용
- Double/Float를 직접 캐스팅 → toBits()로 일관된 규칙 사용
- 키로 쓰는 객체를 컬렉션에 넣은 뒤 필드를 변경
- 가변 객체에 해시 캐싱 도입

---

## 8. 확장 유틸

```kotlin
/** null-safe hash */
inline fun <T> T?.h(): Int = this?.hashCode() ?: 0

/** 31 곱셈 합성 */
fun combineHash(vararg parts: Int): Int {
    var r = 1
    for (p in parts) r = 31 * r + p
    return r
}

// 사용 예
override fun hashCode(): Int = combineHash(
    id.hashCode(),
    email.h(),
    name.hashCode(),
    flags
)
```