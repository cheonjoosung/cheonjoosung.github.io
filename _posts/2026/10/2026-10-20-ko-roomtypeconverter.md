---
title: (Android/Room) Room TypeConverter — 커스텀 타입 저장
tags: [ Android, Room ]
style: fill
color: dark
description: Room이 지원하지 않는 Date, List, Enum, 객체 같은 커스텀 타입을 TypeConverter로 저장하는 방법과 JSON 변환, 주의사항을 정리합니다.
---

---

## 1. TypeConverter가 필요한 이유

Room(SQLite)이 저장할 수 있는 타입은 `Int`, `Long`, `String`, `Double`, `ByteArray` 등 **기본 타입뿐**입니다.
`Date`, `List<String>`, 직접 만든 클래스는 변환 규칙을 알려줘야 합니다.

```
커스텀 타입  ──(변환)──▶  저장 가능한 기본 타입  ──▶ DB
   Date                     Long (timestamp)
   List<String>             String (JSON / 구분자)
```

---

## 2. 기본 사용법

```kotlin
class Converters {

    @TypeConverter
    fun fromTimestamp(value: Long?): Date? = value?.let { Date(it) }

    @TypeConverter
    fun dateToTimestamp(date: Date?): Long? = date?.time
}

@Database(entities = [Event::class], version = 1)
@TypeConverters(Converters::class)         // DB 전체에 적용
abstract class AppDatabase : RoomDatabase()

@Entity
data class Event(
    @PrimaryKey val id: Long,
    val name: String,
    val startAt: Date                      // 이제 그대로 사용 가능
)
```

변환은 **양방향 한 쌍**(저장용 / 복원용)이 필요하며, 함수 시그니처(입력→출력 타입)로 Room이 자동 매칭합니다.

---

## 3. 적용 범위

```kotlin
@TypeConverters(Converters::class)   // 위치에 따라 범위가 달라짐
```

| 위치 | 적용 범위 |
|------|----------|
| `@Database` | DB 전체 (가장 일반적) |
| `@Entity` / `@Dao` | 해당 클래스 |
| 필드 / 파라미터 | 해당 항목 하나 |

---

## 4. Enum

Room은 enum을 기본으로 **이름(String)** 으로 저장합니다. 원하는 값으로 저장하려면 변환기를 만듭니다.

```kotlin
enum class Status(val code: Int) {
    PENDING(0), DONE(1), FAILED(2);

    companion object {
        fun fromCode(code: Int) = entries.first { it.code == code }
    }
}

class StatusConverter {
    @TypeConverter fun toCode(status: Status): Int = status.code
    @TypeConverter fun fromCode(code: Int): Status = Status.fromCode(code)
}
```

> enum **이름**으로 저장하면 이름을 리팩터링할 때 기존 데이터를 읽지 못합니다. 코드(숫자)로 저장하면 이름 변경에 안전합니다.

---

## 5. List / 객체 — JSON 변환

```kotlin
class JsonConverters {
    private val json = Json { ignoreUnknownKeys = true }   // kotlinx.serialization

    @TypeConverter
    fun fromStringList(list: List<String>): String = json.encodeToString(list)

    @TypeConverter
    fun toStringList(value: String): List<String> = json.decodeFromString(value)

    @TypeConverter
    fun fromAddress(address: Address?): String? =
        address?.let { json.encodeToString(it) }

    @TypeConverter
    fun toAddress(value: String?): Address? =
        value?.let { json.decodeFromString<Address>(it) }
}

@Serializable
data class Address(val city: String, val street: String)
```

```kotlin
@Entity
data class User(
    @PrimaryKey val id: Long,
    val tags: List<String>,       // JSON 문자열로 저장
    val address: Address?
)
```

---

## 6. 주의사항 — JSON 컬럼의 한계

```
JSON으로 저장한 컬럼은 SQL로 내부 값을 검색/정렬/조인할 수 없다.
```

```sql
-- tags JSON 안에서 'kotlin' 검색 → 인덱스 불가, LIKE 스캔만 가능
SELECT * FROM users WHERE tags LIKE '%kotlin%'
```

- 단순히 **저장했다가 그대로 읽는** 용도에만 적합합니다
- 내부 값을 **검색하거나 조인**해야 한다면 TypeConverter 대신 **별도 테이블(Relation)** 로 정규화하세요
- 변환 로직에 비용이 큰 처리(암호화, 대용량 파싱)를 넣으면 쿼리마다 실행되어 느려집니다

---

## 7. 기본 타입 체크 — 자주 하는 실수

```kotlin
// ❌ 같은 타입 쌍으로 변환기가 두 개 있으면 Room이 모호해서 빌드 에러
@TypeConverter fun a(list: List<String>): String = ...
@TypeConverter fun b(list: List<String>): String = ...

// ❌ null 처리 누락: DB 값이 null일 때 크래시
@TypeConverter fun toDate(value: Long): Date = Date(value)      // Long? 로 받아야 함

// ✅ nullable 컬럼은 변환기도 nullable로
@TypeConverter fun toDate(value: Long?): Date? = value?.let { Date(it) }
```

---

## 8. 정리

- Room은 기본 타입만 저장 → 나머지는 `@TypeConverter` 쌍(저장/복원)으로 변환
- `@TypeConverters`는 `@Database`에 붙여 전체 적용하는 것이 일반적
- Date는 `Long`, enum은 안정적인 **코드 값**, List/객체는 **JSON 문자열**
- nullable 타입은 변환기도 nullable로 작성
- JSON 컬럼은 내부 검색이 불가 → 검색이 필요하면 **별도 테이블로 정규화**
