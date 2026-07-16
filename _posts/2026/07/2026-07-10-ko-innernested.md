---
title: (Kotlin/코틀린) inner class vs nested class 완전 비교
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin의 inner class와 nested class(static nested)의 차이와 외부 클래스 참조, 메모리 누수 주의사항, 실전 사용 기준을 정리합니다.
---

---

## 1. Java vs Kotlin 기본 동작 차이

Java에서 내부 클래스(inner class)는 **기본이 non-static**, 즉 외부 클래스 인스턴스를 암묵적으로 참조합니다.  
Kotlin은 반대로 **기본이 static(nested)** 이며, 외부 참조가 필요한 경우에만 `inner` 키워드를 명시합니다.

| 구분 | Java | Kotlin |
|------|------|--------|
| 기본 내부 클래스 | non-static (외부 참조 O) | nested (외부 참조 X) |
| 외부 참조 포함 | 명시 없음 | `inner` 키워드 필요 |
| static 내부 클래스 | `static` 명시 | 기본값 (nested class) |

---

## 2. nested class (기본)

외부 클래스 인스턴스와 **독립적**입니다. 외부 클래스의 멤버에 접근할 수 없습니다.

```kotlin
class Outer {
    private val secret = "비밀"

    class Nested {
        fun greet() = "안녕!"
        // fun reveal() = secret  // 컴파일 에러: Outer 인스턴스 없음
    }
}

// Outer 인스턴스 없이 바로 생성 가능
val nested = Outer.Nested()
println(nested.greet())  // 안녕!
```

---

## 3. inner class

`inner` 키워드를 붙이면 외부 클래스 인스턴스를 **암묵적으로 참조**합니다.

```kotlin
class Outer {
    private val secret = "비밀"

    inner class Inner {
        fun reveal() = secret  // 외부 클래스 멤버 접근 가능
        fun outerRef() = this@Outer  // 외부 클래스 인스턴스 명시적 참조
    }
}

// 반드시 Outer 인스턴스를 통해 생성
val outer = Outer()
val inner = outer.Inner()
println(inner.reveal())  // 비밀
```

---

## 4. 핵심 차이 비교

| 구분 | nested class | inner class |
|------|-------------|------------|
| 키워드 | 없음 (기본) | `inner` |
| 외부 클래스 참조 | 없음 | 있음 (암묵적) |
| 생성 방법 | `Outer.Nested()` | `outer.Inner()` |
| 외부 멤버 접근 | 불가 | 가능 |
| 메모리 누수 위험 | 없음 | 있음 (주의 필요) |
| 주요 용도 | 독립 헬퍼 클래스 | 외부 상태 참조 필요 시 |

---

## 5. 메모리 누수 주의 — inner class

`inner class`는 외부 클래스 인스턴스를 강한 참조로 보유합니다.  
외부 클래스가 GC되어야 하는 상황에서도 inner class 인스턴스가 살아있으면 외부도 해제되지 않습니다.

```kotlin
class Activity {
    inner class ClickListener : View.OnClickListener {
        override fun onClick(v: View?) {
            // Activity의 멤버에 접근 가능하지만
            // 이 리스너가 View에 등록된 채로 남으면 Activity 메모리 누수!
            updateUI()
        }
    }
}
```

Android에서 `Handler`, `Runnable`, 콜백 등에 inner class를 사용할 때 특히 주의해야 합니다.  
→ `WeakReference` 사용 또는 nested class로 전환 권장

---

## 6. 실전 사용 패턴

### nested class — Builder 패턴

```kotlin
class HttpRequest private constructor(
    val url: String,
    val method: String,
    val headers: Map<String, String>
) {
    class Builder {  // nested: HttpRequest 인스턴스 불필요
        private var url = ""
        private var method = "GET"
        private val headers = mutableMapOf<String, String>()

        fun url(url: String) = apply { this.url = url }
        fun method(method: String) = apply { this.method = method }
        fun header(key: String, value: String) = apply { headers[key] = value }

        fun build() = HttpRequest(url, method, headers.toMap())
    }
}

val request = HttpRequest.Builder()
    .url("https://api.example.com")
    .header("Authorization", "Bearer token")
    .build()
```

### inner class — Iterator 구현

```kotlin
class LinkedList<T> {
    private var head: Node<T>? = null

    private data class Node<T>(val value: T, var next: Node<T>? = null)

    inner class ListIterator : Iterator<T> {  // inner: head 접근 필요
        private var current = head

        override fun hasNext() = current != null
        override fun next(): T {
            val value = current!!.value
            current = current!!.next
            return value
        }
    }
}
```

---

## 7. 익명 객체와의 관계

익명 객체(`object : Interface { }`)는 inner class처럼 **외부 스코프를 캡처**합니다.

```kotlin
class Counter {
    private var count = 0

    fun getIncrementer() = object : Runnable {
        override fun run() {
            count++  // 외부 count 캡처 — inner class와 동일한 누수 위험
        }
    }
}
```

---

## 8. 정리

- Kotlin nested class는 **기본이 static** — 외부 참조 없음, 독립적
- `inner` 키워드 추가 시 외부 인스턴스 참조 — 외부 멤버 접근 가능
- inner class는 **메모리 누수 위험** — 특히 Android 콜백/리스너 등록 시 주의
- Builder, 헬퍼 클래스 등 독립 구조는 nested, 외부 상태 필요 시 inner 사용
