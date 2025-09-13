---
title: 안드로이드 Gson → Moshi 마이그레이션 가이드 
tags: [Android]
style: fill
color: dark
description: Android Gson → Moshi 마이그레이션 가이드
---
---

## ✨ 개요

많은 Android 프로젝트에서 초기에는 Google의 Gson을 사용하지만,
점점 더 많은 팀이 Moshi로 전환하고 있습니다. 그 이유는 다음과 같습니다:
- Kotlin 친화적
- 빠른 성능 & 코드 생성 지원
- 엄격하고 예측 가능한 파싱
- Square & Retrofit 공식 지원

---

## 1. Gradle 설정

```gradle
// Moshi 라이브러리 추가
implementation("com.squareup.moshi:moshi:1.15.0")
implementation("com.squareup.moshi:moshi-kotlin:1.15.0")
kapt("com.squareup.moshi:moshi-kotlin-codegen:1.15.0")


// build.gradle에 아래 설정도 포함
kotlin {
    sourceSets.all {
        languageSettings.optIn("kotlin.RequiresOptIn")
    }
}
```
> kapt가 필요한 이유: @JsonClass(generateAdapter = true) 기반 코드 생성

---

## 2. 주요 차이점 요약

| 항목             | Gson                       | Moshi                     |
| -------------- | -------------------------- | ------------------------- |
| 어노테이션          | `@SerializedName("field")` | `@Json(name = "field")`   |
| Kotlin 지원      | 약함                         | 강력함 (`moshi-kotlin` 필요)   |
| 기본값 지원         | X (null 필요)                | O (`@JsonClass`로 코드 생성 시) |
| Custom Adapter | TypeAdapter                | JsonAdapter.Factory       |

---

## 3. 클래스 변환 예제

### 기존 Gson 클래스

```kotlin
data class User(
    @SerializedName("user_id") val id: Int,
    @SerializedName("user_name") val name: String,
    val email: String
)

```

### Moshi로 전환

```kotlin
import com.squareup.moshi.Json
import com.squareup.moshi.JsonClass

@JsonClass(generateAdapter = true)
data class User(
    @Json(name = "user_id") val id: Int,
    @Json(name = "user_name") val name: String,
    val email: String
)
```

### Moshi 

```kotlin
val json = """{"id":1, "name":"Joo", "email":"joo@example.com"}"""
val moshi = Moshi.Builder().build()
val adapter = moshi.adapter(User::class.java)
val user = adapter.fromJson(json)
println(user?.email) // joo@example.com

val jsonOutput = adapter.toJson(user)
```

---

## 4. Moshi 파싱 예제

```kotlin
val json = """{"user_id":1,"user_name":"Joo","email":"joo@example.com"}"""

val moshi = Moshi.Builder().build()
val adapter = moshi.adapter(User::class.java)

val user = adapter.fromJson(json)
println(user?.name) // Joo
```

--- 

## 5. Gson -> Moshi 변환 체크리스트

| 항목            | 해야 할 일                                             |
| ------------- | -------------------------------------------------- |
| 어노테이션         | `@SerializedName` → `@Json`                        |
| 어댑터 설정        | Gson의 `TypeAdapter` → Moshi의 `JsonAdapter.Factory` |
| 기본값 처리        | `@JsonClass(generateAdapter = true)` 사용            |
| Retrofit 연동   | MoshiConverterFactory로 교체                          |
| Gson 사용 코드 제거 | `Gson().fromJson()` → `Moshi.adapter().fromJson()` |

---

## 6. Retrofit과 함께 사용하는 경우

### 기존

```kotlin
val retrofit = Retrofit.Builder()
    .baseUrl(BASE_URL)
    .addConverterFactory(GsonConverterFactory.create())
    .build()
```

### 변경 후

```kotlin
val moshi = Moshi.Builder().build()

val retrofit = Retrofit.Builder()
    .baseUrl(BASE_URL)
    .addConverterFactory(MoshiConverterFactory.create(moshi))
    .build()
```

---

## 7. 마이그레이션 주의 사항 및 팁

### 주의사항

| 문제 상황       | 해결 방법                                               |
| ----------- | --------------------------------------------------- |
| 기본값 무시됨     | `@JsonClass(generateAdapter = true)` + `kapt` 설정 확인 |
| Nullable 오류 | Moshi는 기본적으로 `null`을 허용하지 않음 → `String?` 등으로 명시해야 함 |
| 파싱 실패       | 필드 이름이 정확히 일치해야 함, fallback 없음                      |
| 리플렉션 방식 사용  | 리플렉션이 느릴 수 있음 → 가능하면 코드 생성 방식 사용 권장                 |

### 마이그레이션 팁

- 전체 프로젝트 일괄 변경보다 모듈/패키지 단위 점진적 변경 권장
- Gson → Moshi 어댑터 Wrapper를 만들어 중간단계 유지 가능
- 테스트 커버리지가 없다면, 반드시 변경 전후 단위 테스트 추가

---

## 8. 결론

| 마이그레이션 요약                                |
| ---------------------------------------- |
| Gson을 Moshi로 전환하면 Kotlin 친화성 + 성능 향상 가능  |
| 어노테이션, 기본값 처리, Retrofit 설정 등 전체 구조 확인 필요 |
| Moshi는 더 엄격하지만 그만큼 안정성이 높음               |
| 점진적 전환과 테스트 기반 마이그레이션 전략이 핵심             |