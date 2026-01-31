---
title: (Kotlin/코틀린) SimpleDateFormat 사용법 정리
tags: [Android,Kotlin]
style: fill
color: dark
description: (Kotlin/코틀린) SimpleDateFormat 사용법 정리 - 날짜 포맷팅의 기본과 실무 주의사항
---

## 개요

- 안드로이드/코틀린에서 날짜 문자열을 다룰 때 가장 먼저 접하게 되는 클래스가 바로 `SimpleDateFormat` 입니다.
- ```kotlin
  SimpleDateFormat("yyyy-MM-dd HH:mm:ss", Locale.getDefault())
  ```
- 하지만 사용은 간단한 반면, 스레드 안전성과 Locale/TimeZone 이슈로 인해 주의하지 않으면 버그로 이어지기 쉽습니다.

- 요약 
  - SimpleDateFormat은 날짜 ↔ 문자열 변환 클래스
  - 스레드 세이프하지 않음
  - 반드시 Locale 지정
  - Android/코루틴 환경에서는 매번 새로 생성하거나 대안 사용 권장

---

## 1. 기본 사용법

### 1.1 Date → String

```kotlin
val sdf = SimpleDateFormat("yyyy-MM-dd HH:mm:ss", Locale.getDefault())
val formatted = sdf.format(Date())

// 2026-01-31 09:20:00
```

### 1.1 String → Date

```kotlin
val sdf = SimpleDateFormat("yyyy-MM-dd", Locale.getDefault())
val date = sdf.parse("2026-01-31")

// parse()는 실패 시 ParseException을 던질 수 있음
```

---

## 2. 자주 쓰는 날짜 포맷 패턴

| 패턴   | 의미       | 예시   |
| ---- | -------- | ---- |
| yyyy | 연도       | 2026 |
| MM   | 월(01~12) | 01   |
| dd   | 일        | 31   |
| HH   | 시(24h)   | 09   |
| mm   | 분        | 20   |
| ss   | 초        | 00   |
| E    | 요일       | Sat  |
| a    | 오전/오후    | AM   |

---

## 3. Locale을 꼭 지정해야 하는 이유

```kotlin
// ❌ 잘못된 예:
SimpleDateFormat("yyyy-MM-dd")

// ✅ 권장:
SimpleDateFormat("yyyy-MM-dd", Locale.KOREA)
SimpleDateFormat("yyyy-MM-dd", Locale.US)
```

- 디바이스 Locale에 따라 결과가 달라질 수 있음
- 특히 E, MMM, a 같은 문자열 포맷에서 문제 발생

---

## 4. TimeZone 설정 (서버/UTC 처리 시 필수)

- 서버 시간이 UTC로 내려오는 경우:

```kotlin
val sdf = SimpleDateFormat("yyyy-MM-dd'T'HH:mm:ss", Locale.US)
sdf.timeZone = TimeZone.getTimeZone("UTC")

val date = sdf.parse("2026-01-31T00:00:00")

// 한국 시간으로 변환:
sdf.timeZone = TimeZone.getTimeZone("Asia/Seoul")
```

---

## 5. 실무 권장 패턴

### 5-1. 매번 새로 생성

```kotlin
fun formatDate(date: Date): String {
    return SimpleDateFormat("yyyy-MM-dd", Locale.KOREA)
        .format(date)
}
```

- 가장 안전
- 대부분의 앱에서 성능 문제 없음

### 5-2. `runCatching과` 함께 쓰기 (파싱 실패 대비)

```kotlin
fun parseDate(text: String): Date? {
    return runCatching {
        SimpleDateFormat("yyyy-MM-dd", Locale.KOREA).parse(text)
    }.getOrNull()
}
```

- 예외를 밖으로 던지지 않음
- 실무에서 매우 자주 쓰이는 패턴

---

## 6. Android 26+ / Java 8+ 권장 대안

```kotlin
val formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd")
val date = LocalDate.parse("2026-01-31", formatter)
```

- 장점
  - Immutable
  - Thread-safe
  - API 설계가 훨씬 명확
  - 신규 코드라면 java.time 패키지 적극 권장