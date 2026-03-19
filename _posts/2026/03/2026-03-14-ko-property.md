---
title: (Kotlin/코틀린) 프로퍼티 get/set 사용 시 주의사항 - 성능, 스레드, SmartCast까지 정리
tags: [ Android,Kotlin ]
style: fill
color: dark
description: (Kotlin/코틀린) 프로퍼티의 getter/setter 사용 시 발생할 수 있는 성능 문제, SmartCast 제한, 스레드 안전성 이슈와 실무에서의 올바른 사용 패턴 정리
---

## 개요

- Kotlin의 프로퍼티는 단순 필드처럼 보이지만 실제로는 getter/setter 함수로 동작합니다.
- ```kotlin
  var name: Stirng = "Android"
  
  // 컴파일 시 아래로 변환
  getName()
  setName("Kotlin")
  ```
- 👉 즉, 프로퍼티는 “변수”가 아니라 함수 호출 기반 구조입니다.
- 이 때문에 다음과 같은 문제가 발생할 수 있습니다.
  - Smart Cast 불가
  - 성능 문제
  - 스레드 안전성 문제
  - 예상치 못한 값 변경

---

## 1. Custom getter 사용 시 SmartCast 불가

```kotlin
// ❌ 문제 코드
val name: String?
get() = field

if (name != null) {
    println(name.length) // ❌ 컴파일 에러
}

// ✅ 해결 방법
val localName = name

if (localName != null) {
    println(localName.length)
}
```

- getter는 호출될 때마다 값이 바뀔 수 있다고 판단합니다.
- 👉 컴파일러는 Smart Cast를 허용하지 않음

---

## 2. var 프로퍼티 + SmartCast 제한

```kotlin
// ❌ 문제 코드
var user: User? = getUser()

if (user != null) {
    println(user.name) // ❌ SmartCast 불가
}

// 해결
val safeUser = user
if (safeUser != null) {
    println(safeUser.name)
}
```

- 특히 멀티 스레드 환경에서는 더 위험합니다.

---

### 3. getter에 로직 넣는 실수 (성능 문제)

❌ 안 좋은 패턴

```kotlin
val bitmap: Bitmap
get() = BitmapFactory.decodeFile(path)
```

- getter 호출마다 파일 IO 수행
- RecyclerView에서 심각한 성능 저하

✅ 권장 패턴

```kotlin
val bitmap: Bitmap by lazy {
    BitmapFactory.decodeFile(path)
}

fun loadBitmap(): Bitmap {
    return BitmapFactory.decodeFile(path)
}
```

- getter는 “가벼운 연산”만 해야 함

---

## 4. setter에서 부작용(side effect) 발생

❌ 문제 코드

```kotlin
var text: String = ""
set(value) {
    field = value
    textView.text = value
} 
```

- UI 업데이트가 숨겨져 있음
- 디버깅 어려움
- 테스트 어려움

✅ 권장

```kotlin
fun updateText(value: String) {
    text = value
    textView.text = value
}
```

- 👉 setter는 상태 변경만 담당

---

## 5. backing field(field) 사용 주의

```kotlin
var count: Int = 0
    set(value) {
        field = value * 2
    }

// 권장
var rawCount: Int = 0

val count: Int
get() = rawCount * 2
```

- 👉 외부에서 5를 넣어도 실제 값은 10 이런 패턴은 혼란을 유발합니다.
- 역할 분리

---

## 6. getter에서 상태 변경 금지


```kotlin
var count = 0
    get() {
        field++
        return field
    }
```

- getter 호출만으로 상태가 바뀜
- 예측 불가능
- 버그 발생 확률 매우 높음

---

## 7. 멀티 스레드 환경에서의 문제

```kotlin
var data: String? = null
get() = field  // 👉 getter 결과가 항상 안전하지 않음

// 아래의 방식 권장
@Volatile
var data: String? = null

synchronized(lock) {
    data
}
```

---

## 8. open 프로퍼티 오버라이딩 이슈

```kotlin
open class Base {
    open val value: Int = 10
}

class Child : Base() {
    override val value: Int
        get() = computeValue()
}
```

- 👉 부모에서는 단순 값, 자식에서는 로직 수행
- 이런 구조는 예측이 어려움.

---

## 9. 프로퍼티 vs 함수 선택 기준

| 상황             | 프로퍼티 | 함수 |
| -------------- | ---- | -- |
| 값 조회           | O    |    |
| 계산 있음          |      | O  |
| 비용 큼           |      | O  |
| side effect 있음 |      | O  |