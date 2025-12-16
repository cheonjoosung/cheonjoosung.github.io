---
title: (Android/안드로이드) SharedPreferences에 ArrayList/List 저장하고 깨너 쓰는 방법
tags: [Android,Kotlin]
style: fill
color: dark
description: (Android/안드로이드) Thread 사용 시 꼭 알아야 할 유용한 메서드 정리
---

## ✨ 개요

— Kotlin 기준, JSON 직렬화로 가장 실무적인 패턴 정리

앱 개발을 하다 보면 “최근 검색어”, “즐겨찾기 목록”, “간단한 캐시 데이터”처럼 작은 리스트 데이터를 로컬에 저장해야 하는 경우가 많습니다. 이때 가장 접근하기 쉬운 저장소가 SharedPreferences(또는 DataStore Preferences)인데, 문제는 Preferences는 기본적으로 String/Int/Boolean 같은 원시 타입만 저장한다는 점입니다.

> List를 JSON 문자열로 변환(직렬화)해서 저장하고, 다시 JSON을 List로 복원(역직렬화)하면 됩니다.
- String List 저장/조회
- 객체(List<MyModel>) 저장/조회
- 중복 제거/개수 제한(최근 검색어)

---

## 1.  요약

- `SharedPreferences`는 List를 직접 저장할 수 없다 → JSON 문자열로 저장
- `Gson` 또는 `Kotlinx Serialization`로 직렬화/역직렬화
- 데이터가 커지거나 타입 안정성이 중요하면 `DataStore`나 `Room`을 고려

---

## 2. String List 저장/조회 (가장 흔한 케이스)

### 2.1 의존성 (Gson)

```kotlin
implementation("com.google.code.gson:gson:2.10.1")
```

### 2.2 SharedPreferences 확장 함수로 구현

```kotlin
import android.content.SharedPreferences
import com.google.gson.Gson
import com.google.gson.reflect.TypeToken

private val gson = Gson()

fun SharedPreferences.putStringList(key: String, list: List<String>) {
    val json = gson.toJson(list)
    edit().putString(key, json).apply()
}

fun SharedPreferences.getStringList(key: String): List<String> {
    val json = getString(key, null) ?: return emptyList()
    val type = object : TypeToken<List<String>>() {}.type
    return gson.fromJson(json, type) ?: emptyList()
}
```

### 2.3 사용 예시

```kotlin
val prefs = getSharedPreferences("app_prefs", MODE_PRIVATE)

prefs.putStringList("recent_keywords", listOf("Kotlin", "RecyclerView", "Retrofit"))
val recent = prefs.getStringList("recent_keywords")
```

---

## 3. 객체 리스트 저장/조회 (List<MyModel>)

### 3.1 모델 예시

```kotlin
data class RecentItem(
    val id: String,
    val title: String,
    val timestamp: Long
)
```

### 3.2 모델 예시

```kotlin
fun SharedPreferences.putRecentItemList(key: String, list: List<RecentItem>) {
    val json = gson.toJson(list)
    edit().putString(key, json).apply()
}

fun SharedPreferences.getRecentItemList(key: String): List<RecentItem> {
    val json = getString(key, null) ?: return emptyList()
    val type = object : TypeToken<List<RecentItem>>() {}.type
    return gson.fromJson(json, type) ?: emptyList()
}
```

### 3.3 모델 예시

```kotlin
val list = listOf(
    RecentItem("1", "첫 항목", System.currentTimeMillis()),
    RecentItem("2", "두 번째 항목", System.currentTimeMillis())
)

prefs.putRecentItemList("recent_items", list)

val restored = prefs.getRecentItemList("recent_items")
```

---

## 4. 실무 패턴: 최근 검색어(중복 제거 + 개수 제한)

최근 검색어는 “중복 제거 + 최신이 위로 + 최대 N개 유지”가 기본입니다.

```kotlin
fun saveRecentKeyword(prefs: SharedPreferences, key: String, keyword: String, maxSize: Int = 10) {
    val current = prefs.getStringList(key).toMutableList()

    // 중복 제거 후 맨 앞에 추가
    current.remove(keyword)
    current.add(0, keyword)

    // 최대 개수 제한
    val trimmed = current.take(maxSize)

    prefs.putStringList(key, trimmed)
}

// 사용
saveRecentKeyword(prefs, "recent_keywords", "Kotlin", maxSize = 10)
```

---

## 5. 삭제/초기화

```kotlin
fun SharedPreferences.removeKey(key: String) {
    edit().remove(key).apply()
}

fun SharedPreferences.clearAll() {
    edit().clear().apply()
}
```

---

## 6. 주의사항 (중요)

- SharedPreferences는 “작은 데이터”에만 적합
  + 리스트가 커지거나
  + 저장/조회가 잦거나
  + 구조가 복잡해지면
  + → Room 또는 DataStore로 옮기는 게 맞습니다.
- 타입 변경에 취약
  + JSON 구조가 바뀌면 역직렬화가 실패할 수 있음
  + 모델 변경이 잦다면 kotlinx.serialization + 버전 관리 또는 Room을 고려
- 동기/비동기
  + .apply()는 비동기 저장(권장)
  + .commit()은 동기 저장(즉시 결과가 필요할 때만)