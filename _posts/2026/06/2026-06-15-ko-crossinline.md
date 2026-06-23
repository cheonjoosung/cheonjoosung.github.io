---
title: (Kotlin/코틀린) crossinline vs noinline 차이
tags: [ Kotlin, Android ]
style: fill
color: dark
description: Kotlin inline 함수의 동작 원리와 crossinline, noinline 키워드의 차이를 비제어 흐름(non-local return) 제약과 함께 Android 실전 예제로 정리합니다.
---

## 개요

- `inline` 함수에서 람다 매개변수를 다룰 때 쓰이는 **`crossinline`**, **`noinline`** 키워드를 다룹니다.
- 이 글에서는 다음을 설명합니다.
  - `inline` 함수와 non-local return의 기본 동작
  - `crossinline`이 필요한 상황
  - `noinline`이 필요한 상황
  - 두 키워드 비교와 Android 실전 예제

---

## 1. inline 함수와 non-local return

```kotlin
inline fun runTask(task: () -> Unit) {
    println("작업 시작")
    task()
    println("작업 종료")
}

fun process(items: List<Int>) {
    runTask {
        for (item in items) {
            if (item < 0) return  // non-local return — process() 자체를 종료
            println(item)
        }
    }
}
```

```kotlin
// inline이 아니면 컴파일 에러
fun runTaskNonInline(task: () -> Unit) {
    task()
}

fun process2(items: List<Int>) {
    runTaskNonInline {
        if (true) return  // ❌ 컴파일 에러 — non-local return은 inline 함수에서만 가능
    }
}
```

`inline` 함수는 호출부에 람다 코드를 **그대로 삽입**하기 때문에, 람다 내부의 `return`이 바깥 함수까지 종료시킬 수 있습니다.

---

## 2. crossinline — 다른 실행 컨텍스트에서 호출되는 람다

`inline` 함수 안에서 람다가 **다른 람다(다른 실행 컨텍스트) 안에서 호출**되면 non-local return이 허용되지 않습니다.

```kotlin
inline fun runAsync(crossinline task: () -> Unit) {
    val thread = Thread {
        task()  // 다른 실행 컨텍스트(Thread)에서 호출됨
    }
    thread.start()
}

fun process(items: List<Int>) {
    runAsync {
        // return  // ❌ crossinline 람다는 non-local return 금지
        items.forEach { println(it) }
    }
}
```

```kotlin
// crossinline 없이 람다를 다른 컨텍스트에서 호출하려 하면 컴파일 에러
inline fun runAsyncWrong(task: () -> Unit) {  // crossinline 누락
    val thread = Thread {
        task()  // ❌ 컴파일 에러: "can't be inlined into different execution context"
    }
    thread.start()
}
```

`crossinline`은 "이 람다는 인라인되긴 하지만, **non-local return은 금지**한다"는 제약을 추가합니다.

---

## 3. noinline — 인라인하지 않는 람다

`inline` 함수에 매개변수가 여러 개일 때, **특정 람다만 인라인하지 않도록** 지정합니다.

```kotlin
inline fun processData(
    onStart: () -> Unit,
    noinline onComplete: () -> Unit  // 인라인되지 않음 — 별도 객체로 생성
) {
    onStart()
    saveCallback(onComplete)  // 함수 참조처럼 다른 곳에 저장/전달 가능
    println("처리 완료")
}

fun saveCallback(callback: () -> Unit) {
    // 콜백을 저장해두고 나중에 호출
    pendingCallbacks.add(callback)
}
```

```kotlin
// noinline이 필요한 이유 — 인라인 람다는 "저장"하거나 "다른 곳에 전달"할 수 없음
inline fun wrong(task: () -> Unit) {
    pendingCallbacks.add(task)  // ❌ 컴파일 에러 — 인라인 람다는 값으로 취급 불가
}

inline fun correct(noinline task: () -> Unit) {
    pendingCallbacks.add(task)  // ✅ noinline이므로 일반 함수 타입 객체로 존재
}
```

---

## 4. crossinline vs noinline 비교

| 항목 | `crossinline` | `noinline` |
|------|----------------|------------|
| 인라인 여부 | 인라인됨 | 인라인 안 됨 (객체로 생성) |
| non-local return | ❌ 금지 | ❌ 금지 (인라인 안 되므로 원천적으로 불가) |
| 주 사용처 | 다른 실행 컨텍스트(Thread, 다른 람다)에서 호출 | 람다를 저장·전달·반환해야 할 때 |
| 성능 | 인라인 이점 유지 | 일반 함수 타입과 동일 (객체 생성) |

```kotlin
inline fun example(
    crossinline onCross: () -> Unit,
    noinline onNoinline: () -> Unit
) {
    Thread { onCross() }.start()      // crossinline — 다른 컨텍스트 호출 허용
    val ref: () -> Unit = onNoinline  // noinline — 변수에 저장 가능
}
```

---

## 5. Android 실전 예제

### crossinline — 콜백을 비동기 스레드에서 실행

```kotlin
inline fun <T> runOnBackground(
    crossinline action: () -> T,
    crossinline onResult: (T) -> Unit
) {
    Thread {
        val result = action()
        Handler(Looper.getMainLooper()).post {
            onResult(result)  // 다른 스레드(메인) 컨텍스트에서 호출 — crossinline 필요
        }
    }.start()
}

// 사용
runOnBackground(
    action = { repository.loadHeavyData() },
    onResult = { data -> updateUi(data) }
)
```

### noinline — 콜백을 리스트에 저장

```kotlin
class EventBus {
    private val listeners = mutableListOf<() -> Unit>()

    inline fun subscribe(
        immediate: () -> Unit,
        noinline persistent: () -> Unit
    ) {
        immediate()                 // 즉시 인라인 실행
        listeners.add(persistent)   // 저장해야 하므로 noinline 필요
    }

    fun notifyAll() = listeners.forEach { it() }
}
```

### measureTimeMillis — 표준 라이브러리의 inline 활용

```kotlin
// kotlin.system.measureTimeMillis — inline 함수
inline fun measureTimeMillis(block: () -> Unit): Long {
    val start = System.currentTimeMillis()
    block()
    return System.currentTimeMillis() - start
}

fun loadUsers(): List<User> {
    lateinit var result: List<User>
    val time = measureTimeMillis {
        result = repository.fetchUsers()
        // return result  // non-local return으로 fun loadUsers()까지 종료 가능 (가능하지만 비권장)
    }
    println("로딩 시간: ${time}ms")
    return result
}
```

---

## 6. 정리

| 항목 | 내용 |
|------|------|
| `inline` | 호출부에 함수 본문을 삽입, 람다의 non-local return 허용 |
| `crossinline` | 다른 실행 컨텍스트(Thread 등)에서 람다 호출 시 필요, non-local return 금지 |
| `noinline` | 람다를 저장·전달해야 할 때 필요, 인라인되지 않고 일반 객체로 생성 |
| 선택 기준 | 람다를 즉시 같은 흐름에서 실행 → 기본 inline / 다른 컨텍스트 호출 → crossinline / 저장·전달 → noinline |

- 두 키워드 모두 **"이 람다는 일반적인 inline 규칙을 따르지 않는다"** 는 예외를 표시하는 역할입니다.
- 매개변수가 여러 개인 inline 함수를 설계할 때, 각 람다의 사용 방식에 따라 선택합니다.

---

## 참고

- [Kotlin 공식 문서 — Inline functions](https://kotlinlang.org/docs/inline-functions.html)
- [SAM 변환 포스팅 보기](/posts/2026-06-14-ko-sam-conversion)
