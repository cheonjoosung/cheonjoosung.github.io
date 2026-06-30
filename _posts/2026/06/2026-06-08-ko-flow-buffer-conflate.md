---
title: (Kotlin/코틀린) Flow buffer & conflate 완전 정리 — 배압(Backpressure) 처리
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin Flow의 배압(Backpressure) 개념과 buffer, conflate, collectLatest 연산자로 생산자와 소비자 속도 불균형을 처리하는 방법을 정리합니다.
---

---

## 1. 배압(Backpressure)이란?

**생산자(Producer)가 소비자(Consumer)보다 빠르게 값을 방출**할 때 발생하는 문제입니다.

```
생산자: 100ms마다 값 방출
소비자: 값 하나 처리에 500ms 소요

→ 소비자가 처리를 못 따라감 → 배압 발생
```

기본 Flow는 **순차 실행**이라 소비자가 처리를 마칠 때까지 생산자가 기다립니다.

```kotlin
// 기본 Flow: 소비자 속도에 맞춰 대기
flow {
    repeat(5) { i ->
        println("방출: $i (${System.currentTimeMillis()}ms)")
        emit(i)
    }
}.collect { value ->
    delay(500)  // 처리에 500ms
    println("수집: $value")
}

// 출력:
// 방출: 0 → 수집: 0 (500ms) → 방출: 1 → 수집: 1 (500ms) → ...
// 총 소요 시간: 5 * 500ms = 2500ms
```

---

## 2. buffer

### 동작 원리

생산자와 소비자를 **별도 코루틴에서 실행**합니다.  
생산자는 소비자를 기다리지 않고 버퍼에 값을 쌓습니다.

```kotlin
flow {
    repeat(5) { i ->
        println("방출: $i")
        emit(i)
        delay(100)
    }
}
.buffer()  // 생산자-소비자 분리
.collect { value ->
    delay(500)
    println("수집: $value")
}

// 방출과 수집이 동시에 진행됨
// 생산자는 버퍼에 쌓고, 소비자는 버퍼에서 꺼내 처리
```

### buffer 파라미터

```kotlin
// capacity: 버퍼 크기 (기본값: Channel.BUFFERED = 64)
.buffer(capacity = 10)

// 특수 값
.buffer(Channel.UNLIMITED)  // 무제한 버퍼 (OOM 주의)
.buffer(Channel.RENDEZVOUS)  // 버퍼 없음 (기본 동작과 동일)
.buffer(Channel.CONFLATED)  // conflate와 동일
```

### onBufferOverflow 옵션

```kotlin
.buffer(
    capacity = 5,
    onBufferOverflow = BufferOverflow.DROP_OLDEST   // 오래된 값 삭제
    // onBufferOverflow = BufferOverflow.DROP_LATEST // 새 값 버림
    // onBufferOverflow = BufferOverflow.SUSPEND     // 기본값: 생산자 대기
)
```

---

## 3. conflate

### 동작 원리

소비자가 처리 중일 때 쌓인 값들은 **버리고 가장 최신 값만 유지**합니다.  
소비자가 다음 값을 가져갈 때는 항상 **가장 최근 값**을 받습니다.

```kotlin
flow {
    repeat(5) { i ->
        println("방출: $i")
        emit(i)
        delay(100)
    }
}
.conflate()  // 최신 값만 유지
.collect { value ->
    delay(300)
    println("수집: $value")
}

// 출력 예시:
// 방출: 0, 1, 2, 3, 4
// 수집: 0  (처음 값)
// 수집: 3  (그사이 1,2는 버려짐, 3,4 중 최신)
// 수집: 4  (마지막 값)
```

### buffer vs conflate

```
buffer:   [0] [1] [2] [3] [4]  → 모두 처리, 생산자-소비자 병렬화
conflate: [0]       [3] [4]    → 중간값 버림, 항상 최신 값
```

---

## 4. collectLatest

### 동작 원리

새 값이 오면 **이전 collect 블록을 취소하고 새 값으로 재시작**합니다.

```kotlin
flow {
    emit(1)
    delay(100)
    emit(2)
    delay(100)
    emit(3)
}
.collectLatest { value ->
    println("시작: $value")
    delay(200)  // 2가 오면 1의 처리 취소
    println("완료: $value")
}

// 출력:
// 시작: 1
// 시작: 2  (1 취소)
// 시작: 3  (2 취소)
// 완료: 3  (3만 완료)
```

### conflate vs collectLatest

| 구분 | conflate | collectLatest |
|------|----------|---------------|
| 취소 | 소비자 실행 완료 후 최신 값 | 소비자 실행 중 새 값 오면 즉시 취소 |
| 소비 완료 보장 | 각 값 완료 보장 | 마지막 값만 완료 |
| 주요 용도 | UI 업데이트 최적화 | 검색, 무거운 작업 |

---

## 5. 세 가지 방법 종합 비교

| 구분 | buffer | conflate | collectLatest |
|------|--------|----------|---------------|
| 중간값 처리 | 모두 처리 | 버림 | 취소 |
| 순서 보장 | 보장 | 보장 | 마지막만 |
| 메모리 | 버퍼 사용 | 최소 | 최소 |
| 생산자 대기 | 버퍼 꽉 차면 대기 | 없음 | 없음 |
| 주요 용도 | 이벤트 손실 불가 | UI 상태 업데이트 | 입력 기반 재계산 |

---

## 6. 실전 사용 패턴

### UI 업데이트 — conflate

```kotlin
// 빠르게 변하는 UI 상태: 중간 프레임 스킵해도 무방
sensorDataFlow
    .conflate()
    .collect { data ->
        updateUI(data)  // 처리 중 새 값 오면 건너뜀
    }
```

### 로그 수집 — buffer

```kotlin
// 모든 로그를 손실 없이 처리
logEventFlow
    .buffer(capacity = 100)
    .collect { event ->
        database.insertLog(event)  // 순서대로 모두 저장
    }
```

### 자동완성 — collectLatest

```kotlin
searchQueryFlow
    .debounce(300)
    .collectLatest { query ->
        val results = repository.search(query)  // 새 입력 오면 취소
        _results.value = results
    }
```

### 대용량 데이터 파이프라인 — buffer + 병렬처리

```kotlin
dataSourceFlow
    .buffer(capacity = 50)
    .flatMapMerge(concurrency = 4) { item ->
        flow { emit(processItem(item)) }
    }
    .buffer(capacity = 20)
    .collect { result ->
        saveToDatabase(result)
    }
```

---

## 7. 주의 사항

```kotlin
// buffer(UNLIMITED) 주의: OOM 발생 가능
// 생산자가 매우 빠르고 소비자가 매우 느리면 메모리 고갈
fastProducer()
    .buffer(Channel.UNLIMITED)  // 주의!
    .collect { slowConsumer(it) }

// 권장: 적절한 용량과 오버플로우 정책 설정
fastProducer()
    .buffer(capacity = 100, onBufferOverflow = BufferOverflow.DROP_OLDEST)
    .collect { slowConsumer(it) }
```

```kotlin
// collectLatest 안의 취소 불가능 작업 주의
flow.collectLatest { value ->
    withContext(NonCancellable) {
        // 취소되면 안 되는 DB 저장 등은 NonCancellable로 보호
        database.save(value)
    }
}
```

---

## 8. 정리

- **`buffer()`**: 생산자-소비자 병렬화, 모든 값 처리, 이벤트 손실 불가 상황에 사용
- **`conflate()`**: 최신 값만 유지, UI 상태 업데이트처럼 중간값 스킵해도 될 때 사용
- **`collectLatest`**: 새 값 오면 이전 처리 취소, 입력 기반 재계산에 사용

배압 전략 선택 기준: **모든 값 처리 필요** → `buffer`, **항상 최신 상태** → `conflate`, **최신 입력만 처리** → `collectLatest`
