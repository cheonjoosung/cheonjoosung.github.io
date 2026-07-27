---
title: (Kotlin/코틀린) 문자열 포매팅 — format, padStart, padEnd
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin의 String format, padStart, padEnd, trimIndent, multiline string 등 문자열 포매팅과 처리 함수를 실전 예제와 함께 정리합니다.
---

---

## 1. String 템플릿 (기본)

Kotlin의 `${}` 문자열 템플릿은 가장 기본적인 포매팅 방법입니다.

```kotlin
val name = "Alice"
val age = 30
println("이름: $name, 나이: ${age}세")
// 이름: Alice, 나이: 30세

println("내년 나이: ${age + 1}세")
// 내년 나이: 31세
```

---

## 2. String.format

Java의 `String.format`과 동일합니다. 숫자, 날짜, 정렬 등 세밀한 포매팅에 사용합니다.

```kotlin
// 정수 포매팅
println(String.format("%d", 42))      // 42
println(String.format("%05d", 42))    // 00042 (5자리, 앞에 0 채움)
println(String.format("%,d", 1000000)) // 1,000,000 (천 단위 구분)

// 실수 포매팅
println(String.format("%.2f", 3.14159))   // 3.14 (소수점 2자리)
println(String.format("%8.2f", 3.14159))  //     3.14 (8자리, 오른쪽 정렬)
println(String.format("%-8.2f", 3.14159)) // 3.14     (8자리, 왼쪽 정렬)

// 문자열 포매팅
println(String.format("%-10s|%10s", "왼쪽", "오른쪽"))
// 왼쪽        |        오른쪽

// 여러 인자
println(String.format("이름: %s, 점수: %d점 (%.1f%%)", "Alice", 95, 95.0))
// 이름: Alice, 점수: 95점 (95.0%)
```

### format의 주요 포맷 지정자

| 지정자 | 설명 | 예시 |
|--------|------|------|
| `%d` | 정수 | `42` |
| `%f` | 실수 | `3.140000` |
| `%.2f` | 소수점 N자리 | `3.14` |
| `%s` | 문자열 | `"hello"` |
| `%05d` | 0 채움 N자리 | `00042` |
| `%-10s` | 왼쪽 정렬 N자리 | `"hello     "` |
| `%,d` | 천 단위 구분 | `1,000,000` |
| `%%` | % 리터럴 | `%` |

---

## 3. padStart / padEnd

문자열 앞/뒤에 문자를 채워 원하는 길이로 만듭니다.

```kotlin
// padStart: 앞에 채움 (오른쪽 정렬 효과)
println("42".padStart(5))       // "   42" (기본: 공백)
println("42".padStart(5, '0'))  // "00042"

// padEnd: 뒤에 채움 (왼쪽 정렬 효과)
println("42".padEnd(5))        // "42   "
println("42".padEnd(5, '-'))   // "42---"

// 이미 길이가 충분하면 변경 없음
println("12345".padStart(3))   // "12345" (5 >= 3이므로 그대로)
```

### 실전 활용

```kotlin
// 주문번호, 학번 등 고정 자리수 표현
fun formatOrderId(id: Int): String = id.toString().padStart(8, '0')
println(formatOrderId(42))       // 00000042
println(formatOrderId(1234567))  // 01234567

// 표 형태 출력
val items = listOf("사과" to 3, "바나나" to 12, "포도" to 5)
for ((name, count) in items) {
    println("${name.padEnd(6)}${count.toString().padStart(4)}개")
}
// 사과      3개
// 바나나    12개
// 포도       5개
```

---

## 4. trimIndent / trimMargin

여러 줄 문자열의 들여쓰기를 정리합니다.

```kotlin
// trimIndent: 공통 들여쓰기 제거
val json = """
    {
        "name": "Alice",
        "age": 30
    }
""".trimIndent()
// 앞의 공통 4칸 들여쓰기 제거

// trimMargin: | 기준으로 앞 공백 제거
val sql = """
    |SELECT *
    |FROM users
    |WHERE age > 18
    """.trimMargin()
println(sql)
// SELECT *
// FROM users
// WHERE age > 18
```

---

## 5. 유용한 문자열 처리 함수

```kotlin
val s = "  Hello, World!  "

// 공백 처리
println(s.trim())        // "Hello, World!"
println(s.trimStart())   // "Hello, World!  "
println(s.trimEnd())     // "  Hello, World!"

// 대소문자
println(s.trim().uppercase())  // "HELLO, WORLD!"
println(s.trim().lowercase())  // "hello, world!"
println(s.trim().capitalize())  // deprecated → replaceFirstChar 사용

// 반복
println("ab".repeat(3))  // ababab

// 분리 / 결합
val csv = "a,b,c,d"
val parts = csv.split(",")  // ["a", "b", "c", "d"]
println(parts.joinToString(separator = " | "))  // "a | b | c | d"
```

---

## 6. 숫자를 문자열로 — 진법 변환

```kotlin
println(255.toString(16))   // ff    (16진수)
println(255.toString(2))    // 11111111 (2진수)
println(255.toString(8))    // 377   (8진수)

// 역방향
println("ff".toInt(16))     // 255
println("11111111".toInt(2)) // 255
```

---

## 7. 정리

- **String 템플릿 (`${}`)**: 가장 간결, 일반 포매팅에 기본 사용
- **`String.format`**: 자리수, 소수점, 정렬 등 세밀한 포매팅
- **`padStart`/`padEnd`**: 고정 자리수 표현, 표 출력
- **`trimIndent`/`trimMargin`**: 멀티라인 문자열 들여쓰기 정리
- **`joinToString`**: 컬렉션을 구분자와 함께 하나의 문자열로 결합
