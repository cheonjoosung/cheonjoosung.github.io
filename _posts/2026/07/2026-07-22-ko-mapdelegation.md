---
title: (Kotlin/코틀린) Map delegation — Map에서 프로퍼티 읽기
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin의 Map 위임(Map delegation)으로 Map의 값을 프로퍼티처럼 접근하는 방법과 JSON 파싱, 동적 설정 처리 실전 패턴을 정리합니다.
---

---

## 1. Map delegation이란?

Kotlin은 `Map`과 `MutableMap`에 `getValue`/`setValue`가 내장되어 있어  
**Map을 프로퍼티 델리게이트**로 사용할 수 있습니다.

```kotlin
class User(val map: Map<String, Any?>) {
    val name: String by map
    val age: Int by map
    val email: String by map
}

val user = User(mapOf(
    "name" to "Alice",
    "age" to 30,
    "email" to "alice@example.com"
))

println(user.name)   // Alice
println(user.age)    // 30
println(user.email)  // alice@example.com
```

프로퍼티 이름이 Map의 키로 사용됩니다.

---

## 2. var 프로퍼티 — MutableMap 위임

```kotlin
class MutableUser(val map: MutableMap<String, Any?>) {
    var name: String by map
    var age: Int by map
}

val user = MutableUser(mutableMapOf(
    "name" to "Bob",
    "age" to 25
))

user.name = "Charlie"  // map["name"] = "Charlie"
user.age = 26          // map["age"] = 26

println(user.map)  // {name=Charlie, age=26}
```

---

## 3. 키 없을 때 동작

Map에 해당 키가 없으면 `NoSuchElementException`이 발생합니다.

```kotlin
class Config(val map: Map<String, Any?>) {
    val host: String by map
    val port: Int by map
}

val config = Config(mapOf("host" to "localhost"))
println(config.host)  // localhost
println(config.port)  // NoSuchElementException: Key port is missing in the map.
```

기본값이 필요하면 `withDefault`를 사용합니다.

```kotlin
class Config(map: Map<String, Any?>) {
    private val safeMap = map.withDefault { key ->
        when (key) {
            "host" -> "localhost"
            "port" -> 8080
            else   -> null
        }
    }

    val host: String by safeMap
    val port: Int by safeMap
}
```

---

## 4. 실전 패턴 — JSON 응답 파싱

```kotlin
// API 응답이 Map으로 들어오는 경우 (Gson, org.json 등)
class ApiResponse(private val json: Map<String, Any?>) {
    val userId: Int by json
    val userName: String by json
    val email: String by json
    val createdAt: String by json
    val isActive: Boolean by json
}

// 사용
val jsonMap = mapOf(
    "userId" to 1,
    "userName" to "Alice",
    "email" to "alice@example.com",
    "createdAt" to "2026-01-01",
    "isActive" to true
)

val response = ApiResponse(jsonMap)
println("${response.userName} (${response.email})")  // Alice (alice@example.com)
```

---

## 5. 실전 패턴 — 동적 설정 클래스

```kotlin
class AppConfig(private val properties: Map<String, String>) {
    val databaseUrl: String by properties
    val databaseUser: String by properties
    val databasePassword: String by properties
    val maxPoolSize: String by properties
    val debugMode: String by properties
}

// .properties 파일이나 환경변수로부터 로드
fun loadConfig(): AppConfig {
    val props = mapOf(
        "databaseUrl" to System.getenv("DB_URL"),
        "databaseUser" to System.getenv("DB_USER"),
        "databasePassword" to System.getenv("DB_PASSWORD"),
        "maxPoolSize" to (System.getenv("POOL_SIZE") ?: "10"),
        "debugMode" to (System.getenv("DEBUG") ?: "false")
    )
    return AppConfig(props.filterValues { it != null }.mapValues { it.value!! })
}
```

---

## 6. 실전 패턴 — 플러그인 시스템

```kotlin
// 런타임에 동적으로 설정되는 속성들
class PluginConfig(private val settings: MutableMap<String, Any?> = mutableMapOf()) {
    var name: String by settings
    var version: String by settings
    var enabled: Boolean by settings
    var priority: Int by settings

    fun toMap(): Map<String, Any?> = settings.toMap()
}

val plugin = PluginConfig()
plugin.name = "DataExporter"
plugin.version = "1.0.0"
plugin.enabled = true
plugin.priority = 10

println(plugin.toMap())
// {name=DataExporter, version=1.0.0, enabled=true, priority=10}
```

---

## 7. Map delegation vs data class

| 상황 | Map delegation | data class |
|------|---------------|------------|
| 키가 런타임에 결정됨 | 적합 | 부적합 |
| 추가 필드가 자주 생김 | 적합 | 매번 수정 필요 |
| 타입 안전성 | 약함 (캐스팅 필요) | 강함 |
| IDE 자동완성 | 프로퍼티 이름으로 지원 | 완전 지원 |
| JSON 파싱 | 편리 | data class 선호 |
| 직렬화 | 별도 처리 필요 | `@Serializable` 간단 |

**고정된 구조라면 data class**, 동적 구조나 외부 설정 로딩에는 Map delegation이 유용합니다.

---

## 8. 타입 안전성 개선

```kotlin
// 제네릭 헬퍼로 타입 캐스팅 숨기기
class TypedMap(private val map: Map<String, Any?>) {
    inline operator fun <reified T> getValue(thisRef: Any?, property: KProperty<*>): T {
        val value = map[property.name]
            ?: throw NoSuchElementException("키 없음: ${property.name}")
        return value as? T
            ?: throw ClassCastException("${property.name}: ${value::class.simpleName} → ${T::class.simpleName} 변환 실패")
    }
}

class StronglyTypedConfig(map: Map<String, Any?>) {
    private val typedMap = TypedMap(map)
    val host: String by typedMap
    val port: Int by typedMap
    val debug: Boolean by typedMap
}
```

---

## 9. 정리

- `Map`에는 `getValue`가 내장, `MutableMap`에는 `setValue`도 내장
- 프로퍼티 이름이 Map 키로 자동 매핑됨
- 키 없으면 `NoSuchElementException` — `withDefault`로 기본값 처리
- JSON 파싱 결과, 환경변수, 동적 설정처럼 키-값 구조를 타입 있는 프로퍼티로 감쌀 때 유용
- 고정 구조는 data class, 동적/외부 구조는 Map delegation이 적합
