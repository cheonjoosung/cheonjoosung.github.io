---
title: (Kotlin/코틀린) enum vs sealed class 차이점 비교 - 언제 무엇을 써야 할까?
tags: [ Android,Kotlin ]
style: fill
color: dark
description: (Kotlin/코틀린) enum class와 sealed class의 차이점을 비교하고, 상태 표현, 데이터 포함, when 분기, UI 상태 관리, MVI 패턴 등 실무에서 어떤 상황에 무엇을 써야 하는지 정리합니다.
---

## 개요

- Kotlin에서 상태를 표현할 때 자주 비교되는 것이 enum class와 sealed class입니다.
- 둘 다 when과 함께 쓰기 좋고, 타입 안정성이 높다는 공통점이 있지만 성격은 꽤 다릅니다.
- 실무에서 자주 나오는 고민은 보통 이런 식입니다.
    - 단순 상태값이면 enum이 맞나?
    - 성공/실패마다 데이터가 다르면 sealed가 맞나?
    - UI 상태 관리에는 무엇이 더 적합한가?
    - MVI에서는 왜 sealed class를 많이 쓰는가?

---

## 1. 요약

- enum class: 고정된 상수 집합
- sealed class: 제한된 타입 계층
- 상태가 단순하고 값 종류가 고정되어 있으면 enum 상태마다 들고 있는 데이터가 다르거나 구조가 다르면 sealed class

---

## 2. enum은 사실 객체다

```kotlin
enum class Status {
    READY,
    LOADING,
    SUCCESS,
    ERROR
}
```

- 특징
    - 미리 정의된 상수만 존재
    - 각 값은 싱글톤 객체
    - 상태 집합이 단순할 때 적합

---

### 3. sealed class란?

```kotlin
sealed class UiState {
    object Loading : UiState()
    data class Success(val data: List<String>) : UiState()
    data class Error(val message: String) : UiState()
}
```

- 특징
    - 상속 가능한 하위 타입이 제한됨
    - 각 상태가 서로 다른 데이터를 가질 수 있음
    - 단순 상수 이상의 상태 모델링 가능\

---

## 4. 가장 큰 차이: 데이터 포함 가능성

- enum
    - 이 구조는 “상태 이름”만 표현합니다.
    - 문제
        - 성공 시 데이터는 어디에 둘까?
        - 실패 시 에러 메시지는 어떻게 들고 갈까?
- sealed
    - 상태와 데이터가 하나로 묶임
    - 잘못된 조합 방지 가능
    - SUCCESS인데 data가 null인 어색한 상태를 줄일 수 있음

---

## 5. 표현력 차이

- enum이 잘 맞는 경우
    - 탭 종류
    - 정렬 기준
    - 네트워크 연결 상태
    - 요일, 월, 권한 상태
    - 단순 모드 값
- sealed class 가 맞는 경우
    - 로딩 / 성공 / 실패
    - 사용자 액션 종류
    - API 응답 결과
    - 내비게이션 이벤트
    - 에러 타입별 추가 정보

---

## 6. 비교 표

| 항목         | enum class | sealed class  |
| ---------- | ---------- | ------------- |
| 목적         | 고정된 상수 집합  | 제한된 타입 계층     |
| 상태별 데이터    | 제한적        | 자유로움          |
| 표현력        | 단순 상태에 강함  | 복잡한 상태에 강함    |
| when 분기    | 간단         | 타입별 데이터 활용 가능 |
| UI 상태 관리   | 단순한 경우 적합  | 대부분 더 적합      |
| 액션/이벤트 모델링 | 부적합        | 적합            |
| 코드 양       | 적음         | 다소 많음         |