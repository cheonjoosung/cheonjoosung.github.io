---
title: (Android/안드로이드) BadParcelableException() 해결 방법
tags: [ Android,Kotlin ]
style: fill
color: dark
description: (Android/안드로이드) BadParcelableException() 해결 방법 - 객체 대신 JSON(String)으로 Intent/Bundle 전달하기
---

## 개요

- `Intent.putExtra()`로 `@Parcelize` 객체를 전달하다가 아래 같은 크래시를 겪는 경우가 있습니다.
  - `BadParcelableException: ClassNotFoundException when unmarshalling...`
  - `ClassLoader referenced unknown class ...`
- 원인은 대개 Android가 Bundle에서 값을 복원(언마샬링)하는 시점에 classLoader를 제대로 못 잡는 케이스(특히 Fragment arguments / savedInstanceState / 프로세스 death 복원 / 모듈 분리 등)입니다.
- 그래서 “객체 자체(Parcelable)를 넘기지 말고”, Gson으로 JSON 문자열로 변환해서 String으로 전달하면, Bundle이 클래스 로딩을 할 필요가 없어져 언마샬링 크래시를 크게 줄일 수 있습니다.

---

## 1. 왜 Parcelable 대신 JSON String이 해결책이 되나?

- Parcelable 전달의 문제(핵심)
  - Parcelable은 복원 시점에 해당 클래스가 반드시 로드 가능해야 함
  - Bundle이 복원되는 순간(예: 프로세스 재시작, Fragment restore) classLoader가 엉키면:
    - `ClassNotFoundException when unmarshalling`
    - `BadParcelableException`로 이어짐
- JSON String 전달의 장점
  - Bundle에는 단순 String만 들어감 → 언마샬링 시 클래스 로딩 필요 없음
  - 복원 시점에 classLoader가 꼬여도 String은 안전
  - 모듈/동적 기능/난독화 영향도 상대적으로 적음

> 즉, “Parcelable이 깨질 수 있는 환경”에서 String 기반 전달은 구조적으로 더 튼튼한 우회로가 됩니다.

---

## 2. 구현: Gson으로 직렬화해서 putExtra로 넘기기

### 2.1 데이터 모델

```kotlin
data class People(
    val id: Int,
    val name: String
)
```

### 2.2 보내는 쪽: 객체 → JSON String

```kotlin
private const val EXTRA_PEOPLE_JSON = "extra_people_json"

fun Context.openDetail(people: People) {
    val gson = Gson()
    val json = gson.toJson(people)

    val intent = Intent(this, DetailActivity::class.java).apply {
        putExtra(EXTRA_PEOPLE_JSON, json)
    }
    startActivity(intent)
}
```

### 2.3 받는 쪽: JSON String -> 객체

```kotlin
class DetailActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        val json = intent.getStringExtra(EXTRA_PEOPLE_JSON)
        require(!json.isNullOrBlank()) { "people json is null/blank" }

        val people = Gson().fromJson(json, People::class.java)

        // use people...
    }
}
```

- 중요한 점
  - Bundle 복원 단계에서는 “String”만 다루므로 classLoader 문제가 거의 없음
  - fromJson()은 Activity가 생성된 이후 수행되므로 안정적

---

### 3. 리스트/제네릭 타입까지 다룰 때 (TypeToken)

```kotlin
val type = object : TypeToken<List<People>>() {}.type
val peopleList: List<People> = Gson().fromJson(json, type)
```

---

## 4. 실무에서 꼭 넣는 안전장치 3가지

### 4.1 JSON 버전 필드 넣기 (권장)

```kotlin
// 모델이 바뀌면 old JSON이 파싱 실패할 수 있어 “버전”을 넣으면 안정적입니다.
data class PeoplePayload(
    val version: Int = 1,
    val data: People
)
```

### 4.2 파싱 실패 대비 `runCatching`

```kotlin
val people = runCatching {
    Gson().fromJson(json, People::class.java)
}.getOrNull()
```

### 4.3 큰 데이터는 여전히 금지

- JSON은 classLoader 문제는 피하지만, 용량 문제는 남아있습니다.
  - 큰 리스트/큰 문자열 → `TransactionTooLargeException` 위험
  - 해결: 여전히 ID만 전달 + 대상 화면에서 재조회가 최종 정답

---

## 5. Parcelable vs Gson String

| 항목             | Parcelable | Gson JSON String |
| -------------- | ---------- | ---------------- |
| classLoader 의존 | 큼(복원 시 필요) | 거의 없음            |
| 성능             | 빠름         | 파싱 비용 있음         |
| 안정성(복원/모듈)     | 환경에 따라 깨짐  | 상대적으로 안정         |
| 타입 안정성         | 강함         | 런타임 파싱           |
| 대용량 전송         | 위험         | 똑같이 위험           |

- “classLoader/언마샬링 크래시”가 주된 문제면 JSON String이 좋은 우회책
- “대용량”이 문제면 둘 다 위험 → ID 전달 전략으로 전환