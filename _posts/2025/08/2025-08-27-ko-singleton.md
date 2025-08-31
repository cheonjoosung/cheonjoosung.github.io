---
title: Kotlin 싱글턴 패턴 완벽 가이드 - object 키워드로 쉽게 구현하기
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin 싱글턴 패턴 완벽 가이드 - object 키워드로 쉽게 구현하기
---

## ✨ 개요

소프트웨어 개발에서 싱글턴(Singleton)은 가장 널리 사용되는 디자인 패턴 중 하나입니다.
Kotlin에서는 Java보다 훨씬 간단하고 안전하게 싱글턴을 구현할 수 있는데요, 오늘은 Kotlin의 object 키워드를 중심으로 싱글턴 패턴을 쉽고 확실하게 정리해드립니다.

---

## 1️⃣ 싱글턴(Singleton)이란?

- 프로그램 전체에서 단 하나의 인스턴스만 생성되는 객체
- 전역 상태를 관리하거나 공통 유틸리티 기능을 제공할 때 주로 사용됨
> Kotlin에서는 object 키워드를 사용해 간단하게 싱글턴을 구현할 수 있습니다.

---

## 2️⃣ Kotlin에서 싱글턴 구현 방법

- object 키워드 사용
  + ```kotlin
    object AppConfig {
       val appName = "MyApp"
       var version = "1.0"
    
       fun printInfo() {
        println("App: $appName, Version: $version")
       }
    }
    
    fun main() {
        AppConfig.printInfo()         // App: MyApp, Version: 1.0
        AppConfig.version = "1.1"
        AppConfig.printInfo()         // App: MyApp, Version: 1.1
    }
    ``` 

---

## 3️⃣ 싱글턴 활용 예제

- 로그인 세션 관리
  + ```kotlin
    object SessionManager {
    var token: String? = null
    
        fun isLoggedIn(): Boolean = token != null
    }
    
    SessionManager.token = "abc123"
    println(SessionManager.isLoggedIn()) // true
    ```
 
- 로그 유틸리티
  + ```kotlin
    object Logger {
        fun info(tag: String, message: String) {
            println("INFO [$tag]: $message")
        }

        fun error(tag: String, message: String) {
            println("ERROR [$tag]: $message")
        }
    }
    
    fun main() {
        Logger.info("MainActivity", "앱 시작")
        Logger.error("Network", "응답 실패")
    }
    ```
  + object로 선언했기 때문에 어디서든 Logger.info() 식으로 바로 사용 가능하며, 별도의 객체 생성 없이 공통 유틸 기능을 처리할 수 있습니다.    
  
- 간단한 인메모리 캐시
  + ```kotlin
    object CacheManager {
        private val cache = mutableMapOf<String, Any>()

        fun put(key: String, value: Any) {
            cache[key] = value
        }

        fun get(key: String): Any? = cache[key]

        fun clear() {
            cache.clear()
        }
    }

    CacheManager.put("userId", 1234)
    val userId = CacheManager.get("userId") as? Int
    println("User ID = $userId") // User ID = 1234
    ```
  + 자주 쓰는 데이터를 앱 메모리에 저장할 때 사용합니다. (예: API 토큰, 설정 값 등)  

---

## 4️⃣ 비교

- Java & Kotlin 싱글턴 비교
  + | 항목      | Java                      | Kotlin               |
    | ------- | ------------------------- | -------------------- |
    | 구현 방식   | private 생성자 + static 인스턴스 | `object` 키워드 한 줄     |
    | 스레드 안전성 | 개발자가 직접 처리해야 함            | **자동으로 thread-safe** |
    | 코드 복잡도  | 복잡함                       | 매우 간단함               |

- companion object 와의 차이점
  + | 항목        | `object`              | `companion object`   |
    | --------- | --------------------- | -------------------- |
    | 사용 목적     | 싱글턴 객체                | 클래스의 정적 멤버 역할        |
    | 클래스 독립 여부 | 클래스와 무관하게 단독 존재 가능    | 특정 클래스 내부에 정의되어야 함   |
    | 호출 방식     | `ObjectName.method()` | `ClassName.method()` |
    
---

## 5️⃣ 싱글턴 사용 시 주의할 점 

- 앱 전역 상태를 관리하는 데는 좋지만, 지나치게 사용하면 테스트/유지보수 어려움
- object는 앱 전체 생명주기를 가질 수 있으므로 메모리 누수에 주의
- 상태 변경이 많다면 **불변성(immutability)**을 고려할 것

---

## 6️⃣ 결론

- Kotlin의 object는 싱글턴 구현의 최적 해법
- 코드가 간결하고 thread-safe하다
- 남용은 피하고, 전역 상태 관리 등에 한정해 사용