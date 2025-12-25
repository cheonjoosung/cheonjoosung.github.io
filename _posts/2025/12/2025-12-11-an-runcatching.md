---
title: (Android/안드로이드) runCatching vs try-catch 비교 분석
tags: [Android,Kotlin]
style: fill
color: dark
description: (Android/안드로이드) runCatching vs try-catch 비교 분석
---

## ✨ 개요

실무에서 언제 무엇을 쓰는 게 맞을까?
- Kotlin을 쓰다 보면 예외 처리를 try-catch로 할지, runCatching { }로 할지 고민하는 순간이 많습니다. 
- 둘 다 “예외를 잡는다”는 목적은 같지만, 표현력/흐름 제어/가독성/실수 가능성에서 차이가 큽니다.

---

## 1.  요약

- 명시적 흐름(리턴/브레이크/컨티뉴/throw) 제어가 필요하면 try-catch가 더 안전
- “실패 가능 연산”을 값(Result)으로 다루고 체이닝하고 싶다면 runCatching
- runCatching은 편하지만, 예외를 “삼켜버리는” 코드가 되기 쉬우므로 로깅/정책이 중요
- 코루틴/취소(CancellationException)에서는 특히 주의: 무심코 잡으면 취소가 무력화될 수 있음

---

## 2. 개념 차이: “문(Statement)” vs “값(Expression)”

### 2-1. try-catch 제어문에 가까움(명시적 흐름)

```kotlin
try {
    risky()
    doNext()
} catch (e: Exception) {
    handle(e)
}
```

- 성공/실패 분기가 눈에 잘 보임
- return, break, continue, throw 같은 흐름 제어가 자연스럽다

### 2-2. runCatching: 결과를 값(Result)으로 감쌈(함수형 스타일)

```kotlin
val result = runCatching { risky() }
```

- 성공/실패를 **Result<T>**로 표현
- 이후 onSuccess, onFailure, getOrNull, getOrElse, fold 등으로 처리

---

## 3. runCatching이 빛나는 포인트: 체이닝과 조합

### 3-1. 성공/실패 후속 처리 체이닝

```kotlin
runCatching { api.getUser() }
    .onSuccess { user -> cache.save(user) }
    .onFailure { e -> logger(e) }
```

### 3.2 기본값 제공

```kotlin
val port = runCatching { config.readPort() }
    .getOrElse { 8080 }
```

### 3.3 fold로 한 번에 분기

```kotlin
val text = runCatching { readFile() }.fold(
    onSuccess = { it },
    onFailure = { "fallback" }
)
```
이런 “값 기반 흐름”은 try-catch보다 깔끔한 경우가 많습니다.

---

## 4. try-catch가 더 안전한 포인트: 흐름 제어/가독성/정책

### 4-1. 조기 return, break/continue 필요할 때

```kotlin
fun load(): Data {
    return try {
        repo.load()
    } catch (e: IOException) {
        return Data.Empty // 명시적
    }
}
```

- 쿼리 파라미터/인코딩 이슈에 강함
- 슬래시 유무(test://host-action vs test:host-action) 같은 변형에도 안전
- 라우팅 구조가 명확해짐

### 4-2. "어떤 예외를 잡을지"가 중요한 경우

```kotlin
try {
    parser.parse(input)
} catch (e: JsonDataException) {
    // JSON 구조 문제만 처리
} catch (e: IOException) {
    // 네트워크/파일 IO 문제만 처리
}
```
- runCatching은 기본적으로 “던져진 예외를 Result로 감쌈”이라서 정교한 예외 분기에는 try-catch가 더 명확합니다.

---

## 5. 가장 중요한 실무 리스크: runCatching은 “예외 삼키기”가 쉽다

```kotlin
runCatching { risky() } /// XXX

runCatching { risky() }
    .onFailure { e -> Log.e("TAG", "failed", e) } /// OOO
```

- 첫 번쨰 문장처럼 쓰면 실패해도 아무 일도 안 일어나며, 디버깅이 어려워집니다. 
- 따라서 실무에서는 최소한 아래 중 하나는 붙이는 편이 좋습니다.
  - onFailure { log }
  - getOrElse { fallback }
  - getOrThrow()로 다시 던지기

---

## 6. Coroutine에서 특히 주의: CancellationException

- 코루틴 취소는 CancellationException을 통해 전파됩니다.
- 그런데 runCatching이나 광범위한 catch (Throwable)을 쓰면 취소가 잡혀서 무력화될 수 있습니다.

```kotlin
// 위험 예시
lifecycleScope.launch {
    runCatching { delay(1000) } // 취소 예외를 잡아버릴 수 있음
}

// 권장 패턴
runCatching { delay(1000) }
    .onFailure { e ->
        if (e is CancellationException) throw e
        Log.e("TAG", "error", e)
    }

// try-catch
try {
    delay(1000)
} catch (e: CancellationException) {
    throw e
} catch (e: Exception) {
    Log.e("TAG", "error", e)
}
```