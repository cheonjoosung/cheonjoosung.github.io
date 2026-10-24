---
title: (Android/Room) Room FTS (Full Text Search) — 전문 검색
tags: [ Android, Room ]
style: fill
color: dark
description: Room에서 FTS4 가상 테이블로 전문 검색을 구현하는 방법과 contentEntity 연동, MATCH 쿼리, 접두사 검색, 토크나이저 선택을 정리합니다.
---

---

## 1. LIKE 검색의 한계

```sql
SELECT * FROM notes WHERE content LIKE '%코틀린%'
```

앞에 `%`가 붙으면 **인덱스를 사용하지 못하고 전체 행을 스캔**합니다. 데이터가 많아지면 느려집니다.
FTS(Full Text Search)는 텍스트를 **단어 단위로 색인**해 빠른 검색을 제공하는 SQLite 기능입니다.

---

## 2. FTS 엔티티 정의

`@Fts4`를 붙인 엔티티는 가상 테이블이 됩니다.

```kotlin
// 원본 테이블
@Entity(tableName = "notes")
data class Note(
    @PrimaryKey(autoGenerate = true) val id: Long = 0,
    val title: String,
    val content: String,
    val createdAt: Long
)

// 검색용 가상 테이블: 원본의 title, content만 색인
@Fts4(contentEntity = Note::class)
@Entity(tableName = "notes_fts")
data class NoteFts(
    val title: String,
    val content: String
)
```

> `contentEntity`를 지정하면 텍스트를 **중복 저장하지 않고** 원본 테이블을 참조합니다. Room이 원본 변경 시 FTS 테이블을 갱신하는 트리거를 자동 생성합니다.

```kotlin
@Database(entities = [Note::class, NoteFts::class], version = 1)
abstract class AppDatabase : RoomDatabase() {
    abstract fun noteDao(): NoteDao
}
```

---

## 3. MATCH 쿼리

```kotlin
@Dao
interface NoteDao {
    // FTS 테이블에서 검색해 원본 테이블과 rowid로 조인
    @Query("""
        SELECT notes.* FROM notes
        JOIN notes_fts ON notes.rowid = notes_fts.rowid
        WHERE notes_fts MATCH :query
        ORDER BY notes.createdAt DESC
    """)
    fun search(query: String): Flow<List<Note>>
}
```

---

## 4. 검색어 문법

FTS의 MATCH는 별도의 쿼리 문법을 가집니다.

| 입력 | 의미 |
|------|------|
| `kotlin` | 'kotlin' 단어 포함 |
| `kotlin room` | 두 단어 **모두** 포함 (AND) |
| `kotlin OR room` | 둘 중 하나 포함 |
| `kotlin NOT room` | kotlin 포함, room 제외 |
| `"coroutine flow"` | 정확히 이 구문 (연속) |
| `cor*` | **접두사** 검색 (cor로 시작) |
| `title:kotlin` | title 컬럼에서만 검색 |

```kotlin
// 사용자 입력을 접두사 검색으로 변환 (입력 중 실시간 검색)
fun toFtsQuery(input: String): String =
    input.trim()
        .split("\\s+".toRegex())
        .filter { it.isNotEmpty() }
        .joinToString(" ") { "${it.replace("\"", "")}*" }   // 따옴표 제거 후 * 추가

viewModel.search(toFtsQuery("코틀 코루"))   // "코틀* 코루*"
```

> **사용자 입력을 그대로 MATCH에 넣으면 안 됩니다.** `"`, `*`, `-` 등 특수문자가 문법 오류를 일으킬 수 있으니 반드시 정제하세요.

---

## 5. 토크나이저 선택

FTS는 텍스트를 단어로 쪼개는 **토크나이저**에 따라 동작이 달라집니다.

```kotlin
@Fts4(contentEntity = Note::class, tokenizer = FtsOptions.TOKENIZER_UNICODE61)
@Entity(tableName = "notes_fts")
data class NoteFts(val title: String, val content: String)
```

| 토크나이저 | 특징 |
|-----------|------|
| `SIMPLE` (기본) | 공백/기호 기준, ASCII 대소문자 구분 안 함 |
| `UNICODE61` | 유니코드 대소문자 처리, 한글 등 비ASCII 문자도 단어로 인식 |
| `PORTER` | 영어 어간 추출 (running → run) |

> 기본 토크나이저는 비ASCII 문자 처리가 약하므로 **한글이 있다면 `UNICODE61`** 을 권장합니다.

---

## 6. 한국어 검색의 한계

FTS 토크나이저는 **띄어쓰기 단위**로 색인합니다. 한국어는 조사가 붙어 단어가 달라집니다.

```
"코틀린을 배운다"  →  토큰: [코틀린을, 배운다]
검색어 "코틀린"      →  매칭 실패 (토큰이 '코틀린을')
검색어 "코틀린*"     →  접두사 검색으로 매칭 성공
```

그래서 한국어는 **접두사 검색(`*`)을 기본으로 사용**하거나, 부분 문자열 검색이 중요하면 앱 단에서 n-gram(2~3글자 조각)을 별도 컬럼에 저장해 색인하는 방법을 고려합니다.

---

## 7. FTS5 사용

Room은 FTS3/FTS4를 지원합니다. SQLite의 FTS5는 Room 어노테이션이 아직 없어 별도 처리(직접 SQL)가 필요하므로, 일반적인 앱에서는 **FTS4로 충분**합니다.

---

## 8. 정리

- LIKE `%...%`는 전체 스캔 → 데이터가 많으면 **FTS로 단어 색인 검색**
- `@Fts4(contentEntity = ...)`로 원본 테이블과 연결 (중복 저장 없이 트리거로 동기화)
- 검색은 `rowid` 조인 + `MATCH`, 접두사는 `단어*`
- 사용자 입력은 **특수문자 정제 후** MATCH에 전달
- 한글 포함 시 `UNICODE61` 토크나이저 + 접두사 검색 활용
