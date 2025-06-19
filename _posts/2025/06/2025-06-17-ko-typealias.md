---
title: Kotlin typealias 완벽 정리 - 타입 별칭으로 더 읽기 쉬운 코드 만들기
tags: [ Kotlin ]
style: fill
color: dark
description: OkHttp와 Retrofit2는 Android에서 가장 많이 사용되는 HTTP 통신 라이브러리입니다. 이 두 라이브러리의 특징과 공통점, 그리고 차이점을 간결하게 비교 정리합니다.
---

## ✨ 개요

Kotlin에서 typealias는 복잡한 타입 선언을 간단한 이름으로 별칭(alias) 처리할 수 있는 기능입니다. 
긴 타입 이름이나 람다 식을 단순하게 만들어 코드의 가독성과 유지보수성을 높이는 데 유용합니다.

---

## 1. 🔧 기본 사용법

```kotlin
typealias UserMap = Map<Int, String>

typealias ClickHandler = (View) -> Unit
```

- UserMap은 Map<Int, String>의 별칭
- ClickHandler는 View를 매개로 하는 함수 타입의 별칭
> 사용 시 기존 타입과 동일하게 취급되며 컴파일 시 별칭은 실제 타입으로 치환됩니다.

---

## 2. ✅ 활용 사례

- 복잡한 타입을 단순하게 표현할 때
  + ```kotlin
    typealias Callback = (Result<String>) -> Unit
    ```
- DI/Module 구조에서 명확한 의미 전달
  + ```kotlin
    typealias AppDispatchers = CoroutineDispatcherProvider
    ```
- 테스트 코드에서 Mock 타입 식별
  + ```kotlin
    typealias FakeUserRepository = UserRepositoryImpl
    ```

---

## 3. ⚠️ 주의사항

- 타입 자체를 변경하는 것이 아닌 이름만 바꾸는 것
- 타입 호환성 검사 시 실체 타입 기준으로 비교됨
- Generic 타입, 람다 타입, 중첩 타입에 특히 유용

---

## 4. 🧾 결론

`typealias`는 Kotlin에서 타입 선언의 복잡도를 줄이고 의미를 명확히 하기 위한 도구입니다. 
특히 람다나 제네릭이 많은 환경에서 코드를 깔끔하게 정리할 수 있습니다. 
다만 오용 시 실제 타입을 숨겨 혼동을 줄 수 있으므로, 명확한 의미를 전달할 수 있을 때만 사용하는 것이 좋습니다.