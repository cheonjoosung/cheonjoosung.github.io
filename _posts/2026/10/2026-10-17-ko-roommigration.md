---
title: (Android/Room) Room Migration 전략 — 안전한 스키마 변경
tags: [ Android, Room ]
style: fill
color: dark
description: Room 스키마 변경 시 데이터를 보존하는 Migration 작성법과 AutoMigration, 스키마 export, Migration 테스트 방법을 정리합니다.
---

---

## 1. Migration이 필요한 이유

앱을 업데이트하면서 테이블 구조가 바뀌면, 기존 사용자 기기에 있는 DB를 **데이터 손실 없이 새 스키마로 변환**해야 합니다.
`version`만 올리고 Migration을 제공하지 않으면 앱이 크래시합니다.

```
IllegalStateException: A migration from 1 to 2 was required but not found.
```

> `fallbackToDestructiveMigration()`은 Migration이 없을 때 **DB를 통째로 삭제하고 재생성**합니다. 캐시용 DB가 아니라면 운영 앱에서는 사용하지 마세요.

---

## 2. 스키마 export 설정

버전별 스키마 JSON을 남겨두면 Migration 검증과 AutoMigration에 사용됩니다.

```kotlin
// build.gradle.kts
ksp {
    arg("room.schemaLocation", "$projectDir/schemas")
}
```

```kotlin
@Database(entities = [User::class], version = 2, exportSchema = true)
abstract class AppDatabase : RoomDatabase()
```

`schemas/` 폴더(`1.json`, `2.json` ...)는 **버전 관리(Git)에 포함**하세요.

---

## 3. 수동 Migration

SQL로 직접 변환합니다. 가장 유연하고 모든 경우를 처리할 수 있습니다.

```kotlin
// v1 → v2: users 테이블에 age 컬럼 추가
val MIGRATION_1_2 = object : Migration(1, 2) {
    override fun migrate(db: SupportSQLiteDatabase) {
        // NOT NULL 컬럼은 반드시 DEFAULT 지정 (기존 행이 있으므로)
        db.execSQL("ALTER TABLE users ADD COLUMN age INTEGER NOT NULL DEFAULT 0")
    }
}

Room.databaseBuilder(context, AppDatabase::class.java, "app.db")
    .addMigrations(MIGRATION_1_2)
    .build()
```

---

## 4. 컬럼 삭제/변경 — 테이블 재생성

SQLite는 컬럼 삭제·타입 변경을 직접 지원하지 않아 **새 테이블을 만들고 데이터를 옮기는** 방식을 씁니다.

```kotlin
// v2 → v3: users에서 nickname 컬럼 삭제
val MIGRATION_2_3 = object : Migration(2, 3) {
    override fun migrate(db: SupportSQLiteDatabase) {
        // 1. 새 스키마로 임시 테이블 생성
        db.execSQL("""
            CREATE TABLE users_new (
                id INTEGER NOT NULL PRIMARY KEY,
                name TEXT NOT NULL,
                age INTEGER NOT NULL DEFAULT 0
            )
        """)
        // 2. 기존 데이터 복사 (필요한 컬럼만)
        db.execSQL("INSERT INTO users_new (id, name, age) SELECT id, name, age FROM users")
        // 3. 기존 테이블 삭제 후 이름 변경
        db.execSQL("DROP TABLE users")
        db.execSQL("ALTER TABLE users_new RENAME TO users")
    }
}
```

> 새 테이블 정의는 Room이 생성한 `schemas/3.json`의 `createSql`과 **정확히 일치**해야 합니다. 다르면 검증 단계에서 예외가 발생합니다.

---

## 5. AutoMigration — 자동 생성

단순한 변경은 Room이 스키마 JSON을 비교해 Migration을 자동 생성합니다.

```kotlin
@Database(
    entities = [User::class],
    version = 3,
    autoMigrations = [
        AutoMigration(from = 1, to = 2),                       // 컬럼 추가 등 단순 변경
        AutoMigration(from = 2, to = 3, spec = Migration2To3::class)  // 이름 변경/삭제는 spec 필요
    ],
    exportSchema = true
)
abstract class AppDatabase : RoomDatabase()

// 이름 변경/삭제는 의도를 알려줘야 함
@RenameColumn(tableName = "users", fromColumnName = "nickname", toColumnName = "displayName")
@DeleteColumn(tableName = "users", columnName = "legacyField")
class Migration2To3 : AutoMigrationSpec
```

| 자동 가능 | spec 필요 | 불가 (수동) |
|----------|----------|------------|
| 컬럼/테이블 추가 | 컬럼/테이블 이름 변경·삭제 | 타입 변경, 데이터 변환 |

---

## 6. 여러 버전 건너뛰기

사용자가 v1에서 v4로 한 번에 업데이트할 수 있습니다. Room은 **Migration을 체인으로 연결**해 순차 실행합니다.

```kotlin
.addMigrations(MIGRATION_1_2, MIGRATION_2_3, MIGRATION_3_4)
// v1 → v4: 1→2, 2→3, 3→4 순서로 자동 실행
```

각 Migration은 **이전 버전 DB 상태만 가정**하고 작성하면 됩니다.

---

## 7. Migration 테스트

```kotlin
@RunWith(AndroidJUnit4::class)
class MigrationTest {
    @get:Rule
    val helper = MigrationTestHelper(
        InstrumentationRegistry.getInstrumentation(),
        AppDatabase::class.java
    )

    @Test
    fun migrate1To2() {
        // 1. v1 스키마로 DB 생성 후 데이터 삽입
        helper.createDatabase("test.db", 1).apply {
            execSQL("INSERT INTO users (id, name) VALUES (1, '홍길동')")
            close()
        }
        // 2. v2로 Migration 실행 + 스키마 검증
        val db = helper.runMigrationsAndValidate("test.db", 2, true, MIGRATION_1_2)
        // 3. 데이터 보존 확인
        val cursor = db.query("SELECT age FROM users WHERE id = 1")
        cursor.moveToFirst()
        assertEquals(0, cursor.getInt(0))   // 기본값 적용
    }
}
```

---

## 8. 정리

- 스키마가 바뀌면 `version`을 올리고 **Migration을 반드시 제공**
- `exportSchema = true` + `schemas/` Git 관리 (검증·AutoMigration의 기반)
- 단순 변경은 `AutoMigration`, 데이터 변환·타입 변경은 수동 Migration
- 컬럼 삭제/변경은 새 테이블 생성 → 복사 → 교체
- NOT NULL 컬럼 추가는 `DEFAULT` 필수
- `MigrationTestHelper`로 **데이터 보존까지** 테스트
