---
title: repeatOnLifecycle vs launchWhenStarted 완전 비교
tags: [ Kotlin, Android ]
style: fill
color: dark
description: Android Lifecycle-aware 코루틴 API인 repeatOnLifecycle과 launchWhenStarted(deprecated)의 동작 차이와 메모리 누수 문제를 비교 정리합니다.
---

---

## 1. 왜 두 API가 모두 존재하는가?

Android에서 Lifecycle을 인지하는 코루틴 수집 방식은 시간이 지나며 변화했습니다.

```
launchWhenStarted (구) → 메모리/리소스 낭비 문제 발견
       ↓
repeatOnLifecycle (신) → 문제 해결, 현재 공식 권장 방식
```

`launchWhenStarted`는 현재 **deprecated** 되었으며, `repeatOnLifecycle`이 표준입니다.

---

## 2. launchWhenStarted의 문제

### 동작 원리

```kotlin
lifecycleScope.launchWhenStarted {
    flow.collect { value ->
        updateUI(value)
    }
}
```

- STARTED 상태가 될 때까지 코루틴 본문 실행을 **suspend**
- STOPPED 상태가 되면 코루틴 실행을 다시 **suspend** (취소 아님!)
- STARTED로 복귀하면 **이어서 실행**

### 핵심 문제: 백그라운드에서도 업스트림 Flow는 계속 수집됨

```kotlin
lifecycleScope.launchWhenStarted {
    locationFlow.collect { location ->
        // Activity가 백그라운드로 가도 collect 코드 실행은 멈춤
        // 하지만 locationFlow 자체의 업스트림(GPS 콜백 등)은 계속 동작!
        updateMap(location)
    }
}
```

```
Activity STARTED  → collect 블록 실행 중
Activity STOPPED  → collect 블록 실행 일시정지 (suspend)
                     하지만 GPS 콜백, 네트워크 연결 등 업스트림은 계속 살아있음
                     → 배터리 소모, 메모리 낭비
```

---

## 3. repeatOnLifecycle로 해결

### 동작 원리

```kotlin
viewLifecycleOwner.lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        locationFlow.collect { location ->
            updateMap(location)
        }
    }
}
```

- 지정한 상태(`STARTED`)에 도달하면 **새 코루틴을 시작**해서 블록 실행
- 상태 아래로 떨어지면 (`STOPPED`) 코루틴을 **완전히 취소**
- 다시 상태에 도달하면 **처음부터 새로 시작**

```
Activity STARTED → 코루틴 A 시작 (collect 시작)
Activity STOPPED → 코루틴 A 취소 (collect 완전 종료, 업스트림도 취소됨)
Activity STARTED → 코루틴 B 시작 (collect 재시작)
```

업스트림 Flow까지 완전히 취소되므로 **백그라운드에서 리소스를 사용하지 않습니다.**

---

## 4. 핵심 차이 비교

| 구분 | launchWhenStarted | repeatOnLifecycle |
|------|-------------------|--------------------|
| 상태 미달 시 동작 | suspend (일시정지) | 취소 (cancel) |
| 업스트림 Flow | 계속 활성 상태 유지 | 완전히 취소됨 |
| 상태 재도달 시 | 이어서 실행 | 처음부터 재시작 |
| 메모리/배터리 | 낭비 가능 | 효율적 |
| 현재 상태 | Deprecated | 공식 권장 |

---

## 5. 코드 비교 — 실제 동작 확인

```kotlin
// launchWhenStarted: 백그라운드에서도 Flow 생산자가 살아있음
lifecycleScope.launchWhenStarted {
    flow {
        while (true) {
            emit(System.currentTimeMillis())
            delay(1000)
            println("Flow 생산 중...")  // STOPPED 상태에도 출력 계속됨!
        }
    }.collect { time ->
        updateUI(time)  // 이 부분만 실행이 멈춤
    }
}
```

```kotlin
// repeatOnLifecycle: STOPPED 시 생산자까지 완전히 취소
lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        flow {
            while (true) {
                emit(System.currentTimeMillis())
                delay(1000)
                println("Flow 생산 중...")  // STOPPED 시 더 이상 출력 안 됨
            }
        }.collect { time ->
            updateUI(time)
        }
    }
}
```

---

## 6. 사용할 때 주의사항

### Fragment에서는 viewLifecycleOwner 사용

```kotlin
class MyFragment : Fragment() {
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        // ⚠️ this.lifecycleScope 사용 시 Fragment의 생명주기를 따름 (View보다 길게 유지됨)
        // ✅ viewLifecycleOwner.lifecycleScope 사용 권장
        viewLifecycleOwner.lifecycleScope.launch {
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.uiState.collect { state ->
                    renderState(state)  // View가 살아있을 때만 접근
                }
            }
        }
    }
}
```

Fragment의 View는 `onDestroyView`에서 파괴되지만 Fragment 객체는 더 오래 살 수 있습니다.  
`lifecycleScope`(Fragment 기준)를 쓰면 View가 없는데도 코루틴이 View에 접근해 크래시가 날 수 있습니다.

### 여러 Flow를 동시에 수집

```kotlin
viewLifecycleOwner.lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        launch { viewModel.uiState.collect { renderState(it) } }
        launch { viewModel.events.collect { handleEvent(it) } }
        // 두 collect가 각각 독립적인 코루틴으로 동시 실행
    }
}
```

---

## 7. collectAsStateWithLifecycle (Compose)

Compose에서는 `repeatOnLifecycle`을 직접 작성할 필요 없이 라이브러리 함수로 대체됩니다.

```kotlin
@Composable
fun MyScreen(viewModel: MyViewModel) {
    // 내부적으로 repeatOnLifecycle(STARTED)와 동일하게 동작
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    Text(text = uiState.message)
}
```

```kotlin
// build.gradle.kts
implementation("androidx.lifecycle:lifecycle-runtime-compose:2.7.0")
```

---

## 8. 어떤 State를 지정해야 할까

| State | 사용 시점 |
|-------|----------|
| `STARTED` | 가장 일반적, 화면이 보일 때만 데이터 갱신 (기본 권장) |
| `RESUMED` | 화면이 완전히 포그라운드일 때만 (애니메이션, 카메라 등) |
| `CREATED` | 화면이 살아있는 동안 항상 (드물게 사용) |

```kotlin
// 화면이 완전히 활성화되어야 하는 카메라 프리뷰
repeatOnLifecycle(Lifecycle.State.RESUMED) {
    cameraFlow.collect { frame -> processFrame(frame) }
}
```

---

## 9. 정리

- **`launchWhenStarted`**: 코루틴 실행만 일시정지, 업스트림 Flow는 계속 동작 → 리소스 낭비, deprecated
- **`repeatOnLifecycle`**: 상태 벗어나면 코루틴 완전 취소, 업스트림까지 정리 → 공식 권장
- Fragment에서는 `viewLifecycleOwner.lifecycleScope` + `repeatOnLifecycle(STARTED)` 조합이 표준
- Compose에서는 `collectAsStateWithLifecycle()`로 더 간결하게 동일 효과
