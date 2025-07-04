---
title: Kotlin 코루틴 Dispatcher 완벽 가이드 - IO, Main, Default, Immediate
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin 코루틴의 Dispatcher(IO, Main, Default, Immediate)의 차이와 활용법을 실전 예제 중심으로 설명합니다.
---

## ✨ 개요

코틀린 코루틴의 `Dispatcher`는 어떤 스레드나 스레드 풀에서 코루틴이 실행될지는 결정하는
핵심 요소입니다. 올바른 Dispatcher 선택은 앱의 성능과 안정성에 매우 중요합니다.

---

## 1. 🧩 Dispatcher 종류와 특징

| Dispatcher              | 특징                                                             |
|-------------------------|------------------------------------------------------------------|
| `Dispatchers.Default`   | CPU 집중 연산 (정렬, 계산 등) 에 최적화, 공유 풀 사용                   |
| `Dispatchers.IO`        | 파일/네트워크 I/O에 최적화, 많은 스레드 풀 보유                        |
| `Dispatchers.Main`      | 안드로이드 메인(UI) 스레드에서 동작, 화면 업데이트나 UI 이벤트 처리에 적합 |
| `Dispatchers.Main.immediate` | 코루틴이 이미 메인스레드에 있으면 즉시 실행, 그렇지 않으면 Main으로 디스패치 |

---

## 2. ⚙️ Dispatcher 사용 예제


```kotlin
suspend fun loadData() = withContext(Dispatchers.IO) {
    // 네트워크 DB I/O 처리
}

suspend fun calculate() = withContext(Dispatchers.Default) {
    // 무거운 CPU 연산
}

launch(Dispathcers.Main) {
    // UI 업데이트
}
```

---

## 3. 🧪 Dispatcher 선택 시 주의사항

- UI 업데이트 → Dispatchers.Main
- 네트워크/DB → Dispatchers.IO 권장
- 무거운 로직 → Dispatchers.Default
- 즉시 UI 콜백 → Dispatchers.Main>immediate (메인스레드일 때만 즉시 실행)

---

## 4. 🧾 결론

Dispatcher는 코루틴의 실행 컨텍스트를 정의하는 핵심 구성요소입니다. 적절한
Dispatcher를 선택해 스레드 효율성과 성능을 높이고, UI와 백그라운드 작업을 안전하게
분리해 최적의 앱 구조를 설계해 보세요.