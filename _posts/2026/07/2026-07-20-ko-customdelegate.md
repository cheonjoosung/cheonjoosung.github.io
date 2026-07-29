---
title: (Kotlin/코틀린) 커스텀 프로퍼티 델리게이트 만들기
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin에서 getValue/setValue를 구현해 커스텀 프로퍼티 델리게이트를 만드는 방법과 lazy, Delegates.observable 등 내장 델리게이트 원리를 정리합니다.
---

---

## 1. 프로퍼티 델리게이트란?

`by` 키워드 오른쪽에 오는 객체가 프로퍼티의 **get/set 로직을 위임** 받습니다.

```kotlin
class Example {
    val lazyValue: String by lazy { "처음 접근 시 계산됨" }
    var observedValue: String by Delegates.observable("초기값") { _, old, new ->
        println("$old → $new")
    }
}
```

`lazy`, `Delegates.observable`은 모두 `getValue`/`setValue` 규약을 구현한 객체입니다.

---

## 2. 델리게이트 규약 — getValue / setValue

### 읽기 전용 프로퍼티 (val)

```kotlin
import kotlin.reflect.KProperty

class ReadOnlyDelegate<T>(private val value: T) {
    operator fun getValue(thisRef: Any?, property: KProperty<*>): T {
        println("${property.name} 프로퍼티 읽기")
        return value
    }
}

class Example {
    val name: String by ReadOnlyDelegate("Kotlin")
}

val ex = Example()
println(ex.name)  // name 프로퍼티 읽기 → Kotlin
```

### 읽기/쓰기 프로퍼티 (var)

```kotlin
class ReadWriteDelegate<T>(private var value: T) {
    operator fun getValue(thisRef: Any?, property: KProperty<*>): T = value

    operator fun setValue(thisRef: Any?, property: KProperty<*>, newValue: T) {
        println("${property.name}: $value → $newValue")
        value = newValue
    }
}

class Example {
    var count: Int by ReadWriteDelegate(0)
}

val ex = Example()
ex.count = 5   // count: 0 → 5
println(ex.count)  // 5
```

| 파라미터 | 설명 |
|---------|------|
| `thisRef` | 프로퍼티를 소유한 객체 (클래스 인스턴스) |
| `property` | 프로퍼티 메타데이터 (이름, 타입 등) |
| `newValue` | 새로 설정할 값 (setValue만) |

---

## 3. 실전 델리게이트 — SharedPreferences 자동 저장

```kotlin
class SharedPreferenceDelegate<T>(
    private val prefs: SharedPreferences,
    private val key: String,
    private val defaultValue: T
) {
    @Suppress("UNCHECKED_CAST")
    operator fun getValue(thisRef: Any?, property: KProperty<*>): T {
        return when (defaultValue) {
            is String  -> prefs.getString(key, defaultValue) as T
            is Int     -> prefs.getInt(key, defaultValue) as T
            is Boolean -> prefs.getBoolean(key, defaultValue) as T
            is Long    -> prefs.getLong(key, defaultValue) as T
            else       -> throw IllegalArgumentException("지원하지 않는 타입")
        }
    }

    operator fun setValue(thisRef: Any?, property: KProperty<*>, value: T) {
        prefs.edit().apply {
            when (value) {
                is String  -> putString(key, value)
                is Int     -> putInt(key, value)
                is Boolean -> putBoolean(key, value)
                is Long    -> putLong(key, value)
            }
        }.apply()
    }
}

// 편의 함수
fun <T> SharedPreferences.delegate(key: String, defaultValue: T) =
    SharedPreferenceDelegate(this, key, defaultValue)
```

```kotlin
class UserPreferences(prefs: SharedPreferences) {
    var username: String by prefs.delegate("username", "")
    var isDarkMode: Boolean by prefs.delegate("dark_mode", false)
    var loginCount: Int by prefs.delegate("login_count", 0)
}

// 사용: 일반 프로퍼티처럼 읽고 쓰면 자동으로 SharedPreferences에 저장
val userPrefs = UserPreferences(getSharedPreferences("app", Context.MODE_PRIVATE))
userPrefs.username = "Alice"   // 자동 저장
userPrefs.isDarkMode = true    // 자동 저장
println(userPrefs.loginCount)  // SharedPreferences에서 자동 로드
```

---

## 4. 실전 델리게이트 — null 안전 초기화

```kotlin
class NotNullDelegate<T : Any> {
    private var value: T? = null

    operator fun getValue(thisRef: Any?, property: KProperty<*>): T =
        value ?: throw IllegalStateException(
            "${property.name}이 초기화되지 않았습니다. set()을 먼저 호출하세요."
        )

    operator fun setValue(thisRef: Any?, property: KProperty<*>, newValue: T) {
        value = newValue
    }
}

fun <T : Any> notNull() = NotNullDelegate<T>()

class Fragment {
    private var binding: ViewBinding by notNull()

    fun onCreateView() {
        binding = inflateLayout()  // 여기서 초기화
    }

    fun onViewCreated() {
        binding.textView.text = "Hello"  // 안전하게 사용
    }
}
```

이는 사실 Kotlin 표준 라이브러리의 `Delegates.notNull()`과 동일하게 동작합니다.

---

## 5. 실전 델리게이트 — 유효성 검사 포함

```kotlin
class ValidatedDelegate<T>(
    private var value: T,
    private val validate: (T) -> Boolean,
    private val errorMessage: (T) -> String
) {
    operator fun getValue(thisRef: Any?, property: KProperty<*>): T = value

    operator fun setValue(thisRef: Any?, property: KProperty<*>, newValue: T) {
        require(validate(newValue)) { errorMessage(newValue) }
        value = newValue
    }
}

class User {
    var name: String by ValidatedDelegate(
        value = "",
        validate = { it.length in 2..20 },
        errorMessage = { "이름은 2~20자여야 합니다. 입력: '$it'" }
    )

    var age: Int by ValidatedDelegate(
        value = 0,
        validate = { it in 0..150 },
        errorMessage = { "나이는 0~150 사이여야 합니다. 입력: $it" }
    )
}

val user = User()
user.name = "Alice"   // OK
user.age = 25         // OK
user.name = "A"       // IllegalArgumentException: 이름은 2~20자여야 합니다.
```

---

## 6. provideDelegate — 초기화 시점 검증

`provideDelegate`를 구현하면 델리게이트 생성 시점에 추가 로직을 실행할 수 있습니다.

```kotlin
class PositiveIntDelegate(private var value: Int) {
    operator fun getValue(thisRef: Any?, property: KProperty<*>): Int = value
    operator fun setValue(thisRef: Any?, property: KProperty<*>, newValue: Int) {
        value = newValue
    }
}

class PositiveIntProvider(private val initialValue: Int) {
    operator fun provideDelegate(thisRef: Any?, property: KProperty<*>): PositiveIntDelegate {
        require(initialValue > 0) {
            "${property.name} 초기값은 양수여야 합니다. 입력: $initialValue"
        }
        return PositiveIntDelegate(initialValue)
    }
}

class Config {
    var maxConnections: Int by PositiveIntProvider(10)   // OK
    // var timeout: Int by PositiveIntProvider(-1)        // 선언 시점에 즉시 예외 발생
}
```

---

## 7. 내장 델리게이트 정리

| 델리게이트 | 사용 | 특징 |
|-----------|------|------|
| `lazy { }` | `val` | 첫 접근 시 초기화, thread-safe 옵션 |
| `Delegates.observable` | `var` | 값 변경 후 콜백 |
| `Delegates.vetoable` | `var` | 변경 전 조건 확인, false 반환 시 거부 |
| `Delegates.notNull()` | `var` | 초기화 전 접근 시 예외 |
| `map.delegate(key, default)` | `var` | Map에서 값 읽기 (다음 편 주제) |

---

## 8. 정리

- 델리게이트 규약: `getValue(thisRef, property)` + `setValue(thisRef, property, value)` 구현
- `val`은 `getValue`만, `var`은 `setValue`도 필요
- `provideDelegate`로 선언 시점 검증 가능
- SharedPreferences, 유효성 검사, null 안전 초기화 등 반복적인 프로퍼티 로직을 재사용 가능한 델리게이트로 추출
