---
title: Android DataStore 완벽 가이드 - SharedPreferences를 대체하는 최신 데이터 저장소
tags: [ Android ]
style: fill
color: dark
description: Jetpack DataStore를 사용하여 Android에서 앱 설정과 사용자 데이터를 안전하고 비동기적으로 저장하는 방법과 SharedPreferences와의 비교를 설명합니다.
---

## ✨ 개요

Jetpack의 DataStore는 SharedPreferences의 대체재로 등장한 최신 데이터 저장 솔루션입니다. 비동기적이고 안전하며, 코루틴(Coroutine)과 Flow 기반의 데이터 처리를 지원합니다. 설정 정보나 간단한 유저 데이터를 저장할 때 강력한 장점을 가집니다.

---

## 1. ⚙️ 개념

- Jetpack에서 제공하는 최신 데이터 저장 솔루션으로, 비동기 처리와 타입 안전성을 갖춘 Key-Value 저장 방식
- SharedPreferences의 단점을 보완하고, Flow 및 Coroutine을 기반으로 설계
- 핵심 특징
  * 코루틴 기반 비동기 저장
  * Flow 통한 실시간 데이터를 감지 및 수집
  * 두 종류 제공
    1. `Preferences DataStore`: Key-Value 저장
    2. `Proto DataStroe`: 구조화된 커스텀 객체 저장 (protobuf 기반)
  * 스레드 안전 + ANR 방지
  * JetPack 공식 권장 방식


---

## 2. ⚙️ 기본 사용 방법

- 의존성 추가
  + `implementation("androidx.datastore:datastore-preferences:1.0.0")`

```kotlin
class UserPreferencesRepository(private val context: Context) {
  // 초기 설정
  private val Context.dataStore: DataStore<Preferences> by preferencesDataStore(name = "user_settings")

  // 키 정의
  object UserPreferencesKeys {
    val USERNAME = stringPreferencesKey("userName")
    val IS_LOGIN = booleanPreferencesKey("isLogin")
  }

  val userData: Flow<Pair<String?, Boolean>> = context.dataStore.data
    .map { prefs ->
      val username = prefs[UserPreferencesKeys.USERNAME]
      val isLoggedIn = prefs[UserPreferencesKeys.IS_LOGIN] ?: false
      username to isLoggedIn
    }

  suspend fun saveUser(username: String, isLoggedIn: Boolean) {
    context.dataStore.edit { prefs ->
      prefs[UserPreferencesKeys.USERNAME] = username
      prefs[UserPreferencesKeys.IS_LOGIN] = isLoggedIn
    }
  }

  suspend fun clearUser() {
    context.dataStore.edit { prefs ->
      prefs.clear()
    }
  }
}
```
- preferencesDataStore()를 통해 설정 기반 저장소 생성
- Key는 stringPreferencesKey, intPreferencesKey 등으로 정의

---

## 3. 📊 DataStore vs SharedPreferences 비교

| 항목                | SharedPreferences               | DataStore (Preferences)           |
|---------------------|----------------------------------|-----------------------------------|
| API 스타일          | 동기(Synchronous)                | 비동기(Asynchronous, Coroutine)   |
| 트랜잭션 보장       | 불안정                            | 안정적                            |
| 타입 안전성         | 낮음 (Key-Value 문자열 기반)     | 높음 (Type-safe Preferences API) |
| 권장 여부           | 사용 가능하나 최신 앱에선 비권장  | Jetpack 권장 방식                 |
| 백업 가능성         | 있음                              | 있음                              |
| 단점                | ANR 위험, 데이터 손실 가능성       | 초기 학습 곡선 존재                |

- DataStore는 비동기, 데이터 손실 방지, 타입 안전성 측면에서 SharedPreferences보다 뛰어납니다. 
- Jetpack 개발 환경에서는 DataStore를 적극 사용하는 것이 권장됩니다.

---

## 4. 🧾 결론

- SharedPreferences의 한계를 보완한 Jetpack 기반의 최신 데이터 저장소가 DataStore입니다.
- Coroutine과 Flow를 통해 비동기적이고 안전한 접근이 가능하며, 타입 안정성도 확보할 수 있습니다.
- 최신 앱 아키텍처에서는 SharedPreferences 대신 DataStore 사용이 표준으로 자리잡고 있습니다.