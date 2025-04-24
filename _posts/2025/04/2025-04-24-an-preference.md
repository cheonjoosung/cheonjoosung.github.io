---
title: Android SharedPreferences 확장 함수로 간단하게 사용하는 방법 (제너릭 사용)
tags: [Android]
style: fill
color: dark
description: Android SharedPreferences 확장 함수로 간단하게 사용하는 방법 (제너릭 사용)
---

## 개요

Android 앱에서 사용자의 설정이나 로그인 정보 등을 간단히 저장할 수 있는 저장소로 SharedPreferences가 자주 활용됩니다.
하지만 매번 edit() → apply() 과정을 반복하다 보면 코드가 불필요하게 길어질 수 있죠.

**확장 함수**(Extension Function)를 SharedPreferences를 간결하고 안전하게 사용하는 방법을 소개합니다.

- SharedPreferences 를 주로 사용하는 곳
  + 설정값 저장
  + 자동 로그인
  + 테마 저장 등

---

## 확장 함수로 더 간단하게 만들기
### 1-1. SharedPreference 저장: putPref(key, value) 확장 함수
```kotlin
// Extensions.kt
inline fun <reified T> Context.putPref(key: String, value: T) {
  val prefs = getSharedPreferences("user_prefs", Context.MODE_PRIVATE)
  with(prefs.edit()) {
    when (value) {
      is String -> putString(key, value)
      is Int -> putInt(key, value)
      is Boolean -> putBoolean(key, value)
      is Float -> putFloat(key, value)
      is Long -> putLong(key, value)
      else -> throw IllegalArgumentException("Unsupported Type")
    }
    apply()
  }
}
```

### 1-2. SharedPreference 꺼내오기: getPref(key, value) 확장 함수
```kotlin
// Extensions.kt
inline fun <reified T> Context.getPref(key: String, defaultValue: T): T {
    val prefs = getSharedPreferences("user_prefs", Context.MODE_PRIVATE)
    return when (T::class) {
        String::class -> prefs.getString(key, defaultValue as? String ?: "") as T
        Int::class -> prefs.getInt(key, defaultValue as? Int ?: 0) as T
        Boolean::class -> prefs.getBoolean(key, defaultValue as? Boolean ?: false) as T
        Float::class -> prefs.getFloat(key, defaultValue as? Float ?: 0f) as T
        Long::class -> prefs.getLong(key, defaultValue as? Long ?: 0L) as T
        else -> throw IllegalArgumentException("Unsupported Type")
    }
}
```


## 2. 비교하기
### 2-1. SharedPreference 기존방식

```kotlin
// MainActivity.kt
private fun before() {
  val prefs = getSharedPreferences("user_prefs", Context.MODE_PRIVATE)
  with(prefs.edit()) {
    putString("stringValue", "string")
    apply()
  }
  with(prefs.edit()) {
    putInt("intValue", 100)
    apply()
  }
  with(prefs.edit()) {
    putBoolean("booleanValue", false)
    apply()
  }

  prefs.getString("stringValue", "").also { Log.e("CJS", "it=$it") }
  prefs.getInt("intValue", 0).also { Log.e("CJS", "it=$it") }
  prefs.getBoolean("booleanValue", false).also { Log.e("CJS", "it=$it") }
}
```

### 2-2. SharedPreference 확장함수 방식

```kotlin
// MainActivity.kt
private fun after() {
  putPref("stringValue", "string")
  putPref("intValue", 100)
  putPref("booleanValue", true)

  getPref("stringValue", "")
  getPref("intValue", 0)
  getPref("booleanValue", false)
}
```

- ✅ SharedPreferences 기존 방식 vs 제네릭 확장 함수 방식 비교

| 항목               | 기존 방식 사용 예시                              | 제네릭 확장 함수 사용 예시                     |
|--------------------|--------------------------------------------------|------------------------------------------------|
| **코드 중복**      | `putString()`, `putInt()` 등 여러 메서드 호출 필요 | `putPref(key, value)` 한 줄로 처리             |
| **가독성**         | 타입마다 다른 함수 호출                          | 모든 타입을 같은 방식으로 작성 가능            |
| **유지보수성**     | 타입 추가 시 함수 추가 필요                      | 확장 함수 내부 `when` 블록만 수정하면 됨       |
| **타입 안전성**    | 컴파일 타임 보장 (단, 중복 많음)                  | `reified` 제네릭으로 안전성 확보               |
| **확장성**         | 객체 저장 등 추가 작업 복잡                      | `Gson` 활용으로 `saveObject()` 형태로 확장 가능 |
| **실행 예외 처리** | 거의 없음                                       | 미지원 타입 예외 처리 가능 (`IllegalArgumentException`) |


## 3. 보너스 팁: removePref() 및 clearPrefs() 추가 
- 로그아웃, 앱 초기화, 캐시 초기화 등에 자주 사용됩니다.

```kotlin
// Extensions.kt
fun Context.removePref(key: String) {
  getSharedPreferences("user_prefs", Context.MODE_PRIVATE).edit().remove(key).apply()
}

fun Context.clearPrefs() {
  getSharedPreferences("user_prefs", Context.MODE_PRIVATE).edit().clear().apply()
}

```

---

## 5. 결론
- SharedPreferences는 간단한 설정값 저장에 매우 적합하지만,
  매번 `edit()` → `apply()` 반복으로 인해 코드가 장황해질 수 있습니다.
- 확장 함수와 제네릭을 조합하면,
  - 중복 코드 제거
  - 모든 타입 대응
  - 객체 저장까지 확장 가능
- 프로젝트 전반에서 유지보수성과 생산성을 동시에 향상시킬 수 있는 실전 팁입니다 💡