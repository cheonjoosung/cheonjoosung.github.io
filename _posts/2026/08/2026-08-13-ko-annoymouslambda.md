---
title: (Kotlin/코틀린) object expression (익명 객체) vs lambda 비교
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin의 object expression(익명 객체)과 lambda의 차이점, SAM 변환, 여러 메서드를 가진 인터페이스 처리 방법과 올바른 선택 기준을 정리합니다.
---

---

## 1. object expression — 익명 객체

이름 없이 클래스를 즉석에서 정의하고 인스턴스를 생성합니다.

```kotlin
// 인터페이스를 익명 객체로 구현
interface ClickListener {
    fun onClick(view: View)
    fun onLongClick(view: View): Boolean
}

val listener = object : ClickListener {
    override fun onClick(view: View) {
        println("클릭: ${view.id}")
    }
    override fun onLongClick(view: View): Boolean {
        println("롱클릭: ${view.id}")
        return true
    }
}

button.setClickListener(listener)
```

---

## 2. lambda — 단일 메서드 함수

단일 메서드 인터페이스(함수형 인터페이스)를 간결하게 표현합니다.

```kotlin
// Java의 단일 메서드 인터페이스
button.setOnClickListener { view ->
    println("클릭: ${view.id}")
}

// Kotlin의 함수형 인터페이스 (fun interface)
fun interface Validator {
    fun validate(input: String): Boolean
}

val notBlank: Validator = Validator { it.isNotBlank() }
val minLength: Validator = Validator { it.length >= 8 }
```

---

## 3. SAM 변환 — Java 인터페이스와 lambda

Java의 단일 추상 메서드(SAM) 인터페이스는 Kotlin에서 lambda로 사용할 수 있습니다.

```kotlin
// Java: Runnable, Comparator, OnClickListener 등
val thread = Thread { println("백그라운드 실행") }
thread.start()

// Comparator (Java SAM)
val comparator = Comparator<String> { a, b -> a.length - b.length }

// Java OnClickListener
button.setOnClickListener { view ->
    // OnClickListener.onClick(view) 구현
}
```

**Kotlin 인터페이스는 SAM 변환이 기본 지원되지 않습니다.**  
`fun interface`로 선언해야 lambda 사용이 가능합니다.

```kotlin
// Kotlin 일반 interface — SAM 변환 안 됨
interface KotlinCallback {
    fun onResult(result: String)
}
// val cb: KotlinCallback = { println(it) }  // ❌ 컴파일 에러

// fun interface — SAM 변환 가능
fun interface KotlinCallback {
    fun onResult(result: String)
}
val cb: KotlinCallback = { println(it) }  // ✅
```

---

## 4. 메서드 수에 따른 선택

```kotlin
// 메서드 1개 → lambda (또는 fun interface)
fun interface SingleMethod {
    fun execute(): String
}
val single: SingleMethod = { "결과" }

// 메서드 2개 이상 → object expression 필수
interface MultiMethod {
    fun onStart()
    fun onEnd()
    fun onError(e: Exception)
}
val handler = object : MultiMethod {
    override fun onStart() = println("시작")
    override fun onEnd() = println("종료")
    override fun onError(e: Exception) = println("에러: ${e.message}")
}
```

---

## 5. 외부 변수 캡처 비교

```kotlin
var count = 0

// lambda: 외부 var 캡처 가능
val incrementer = Runnable { count++ }
incrementer.run()
println(count)  // 1

// object expression: 외부 var 캡처 가능
val counter = object : Runnable {
    override fun run() { count++ }
}
counter.run()
println(count)  // 2

// object expression만의 특권: 자기 자신 참조
val selfReferencing = object : Runnable {
    var runCount = 0  // 자체 상태 보유
    override fun run() {
        runCount++
        println("${runCount}번째 실행")
    }
}
selfReferencing.run()  // 1번째 실행
selfReferencing.run()  // 2번째 실행
```

---

## 6. 실전 패턴 — Android Listener

```kotlin
// 메서드 1개: lambda 사용
binding.button.setOnClickListener {
    viewModel.onButtonClicked()
}

// 메서드 여러 개: object expression
recyclerView.addOnScrollListener(object : RecyclerView.OnScrollListener() {
    override fun onScrollStateChanged(recyclerView: RecyclerView, newState: Int) {
        if (newState == RecyclerView.SCROLL_STATE_IDLE) {
            viewModel.onScrollStopped()
        }
    }
    override fun onScrolled(recyclerView: RecyclerView, dx: Int, dy: Int) {
        if (!recyclerView.canScrollVertically(1)) {
            viewModel.loadNextPage()
        }
    }
})

// TextWatcher: 메서드 3개
binding.editText.addTextChangedListener(object : TextWatcher {
    override fun beforeTextChanged(s: CharSequence?, start: Int, count: Int, after: Int) {}
    override fun onTextChanged(s: CharSequence?, start: Int, before: Int, count: Int) {
        viewModel.onQueryChanged(s?.toString() ?: "")
    }
    override fun afterTextChanged(s: Editable?) {}
})
```

---

## 7. 실전 패턴 — 전략 패턴에서 선택

```kotlin
fun interface SortStrategy {
    fun sort(list: MutableList<Int>)
}

class DataProcessor(private val strategy: SortStrategy) {
    fun process(data: MutableList<Int>) {
        strategy.sort(data)
    }
}

// lambda로 전략 주입
val bubbleSort = DataProcessor { list ->
    for (i in list.indices) {
        for (j in 0 until list.size - i - 1) {
            if (list[j] > list[j + 1]) {
                val temp = list[j]; list[j] = list[j + 1]; list[j + 1] = temp
            }
        }
    }
}

// object expression으로 상태 있는 전략
val trackedSort = DataProcessor(object : SortStrategy {
    var sortCount = 0
    override fun sort(list: MutableList<Int>) {
        sortCount++
        println("${sortCount}번째 정렬")
        list.sort()
    }
})
```

---

## 8. object expression vs object 선언

```kotlin
// object expression: 매번 새 인스턴스 생성
val a = object : Runnable { override fun run() = println("A") }
val b = object : Runnable { override fun run() = println("A") }
println(a === b)  // false (다른 인스턴스)

// object 선언: 싱글턴
object Singleton : Runnable {
    override fun run() = println("Singleton")
}
println(Singleton === Singleton)  // true
```

---

## 9. 선택 가이드

```
메서드가 1개인가?
    ↓ Yes → lambda 또는 fun interface
    ↓ No  → object expression

자체 상태(필드)가 필요한가?
    ↓ Yes → object expression
    ↓ No  → lambda

Java 인터페이스인가?
    ↓ Yes → SAM 변환으로 lambda 사용
    ↓ No (Kotlin) → fun interface 여부 확인

여러 곳에서 재사용하는가?
    ↓ Yes → 별도 클래스 또는 fun interface 인스턴스
    ↓ No  → 인라인 lambda / object expression
```

---

## 10. 정리

- **lambda**: 단일 메서드 함수형 인터페이스에 사용, 가장 간결
- **`fun interface`**: Kotlin 인터페이스를 SAM 변환 가능하게 선언
- **object expression**: 메서드가 여러 개이거나 자체 상태가 필요할 때
- Java SAM 인터페이스는 Kotlin에서 자동으로 lambda로 사용 가능
- Kotlin 일반 interface는 `fun interface`로 선언해야 lambda 사용 가능
