---
title: (코틀린/Kotlin) SharedFlow vs Channel 완전 비교 — 일회성 이벤트 선택 기준
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin의 SharedFlow와 Channel을 비교하여 일회성 이벤트(토스트, 네비게이션) 처리에 어떤 것을 선택해야 하는지 기준을 정리합니다.
---

---

## 1. 일회성 이벤트란?

화면 상태(State)와 달리 **한 번만 소비되어야 하는 이벤트**가 있습니다.

```
- 토스트 메시지 표시
- 화면 이동 (Navigation)
- 스낵바 표시
- 다이얼로그 1회 표시
```

이런 이벤트는 StateFlow처럼 "최신 값을 항상 유지"하면 문제가 생깁니다.  
화면 회전 시 재구독하면 **이미 처리한 이벤트가 다시 발생**할 수 있기 때문입니다.

```kotlin
// 문제 상황: StateFlow로 이벤트 처리
private val _toastEvent = MutableStateFlow<String?>(null)
val toastEvent: StateFlow<String?> = _toastEvent.asStateFlow()

fun showToast(message: String) {
    _toastEvent.value = message
}

// 화면 회전 → 재구독 → 이미 본 토스트가 다시 표시됨!
```

---

## 2. SharedFlow로 해결

```kotlin
private val _toastEvent = MutableSharedFlow<String>()
val toastEvent: SharedFlow<String> = _toastEvent.asSharedFlow()

fun showToast(message: String) {
    viewModelScope.launch {
        _toastEvent.emit(message)
    }
}
```

```kotlin
// Fragment에서 수집
viewLifecycleOwner.lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.toastEvent.collect { message ->
            Toast.makeText(context, message, Toast.LENGTH_SHORT).show()
        }
    }
}
```

`replay = 0`(기본값)이므로 구독 이전에 발생한 이벤트는 새 구독자에게 전달되지 않습니다.

---

## 3. Channel이란?

`Channel`은 코루틴 간 **값을 전달하는 파이프**입니다. 큐(Queue)와 유사하게 동작합니다.

```kotlin
val channel = Channel<String>()

// 송신
launch {
    channel.send("이벤트1")
    channel.send("이벤트2")
}

// 수신
launch {
    for (event in channel) {
        println("수신: $event")
    }
}
```

### ViewModel에서 Channel 사용

```kotlin
private val _toastEvent = Channel<String>(Channel.BUFFERED)
val toastEvent = _toastEvent.receiveAsFlow()

fun showToast(message: String) {
    viewModelScope.launch {
        _toastEvent.send(message)
    }
}
```

---

## 4. SharedFlow vs Channel 핵심 차이

| 구분 | SharedFlow | Channel |
|------|-----------|---------|
| 구독자 모델 | 다중 구독자 (broadcast) | 단일 소비 (1개 구독자가 값을 가져가면 사라짐) |
| 구독자 없을 때 | 값 손실 (replay=0 기준) | 버퍼에 쌓임 (용량 내) |
| 구독자 여러 명 | 모두에게 전달 | 단 하나만 받음 |
| API 스타일 | Flow 기반 (`collect`) | Channel 기반 (`receive`, `for`) |
| Flow 변환 | 기본 | `receiveAsFlow()`/`consumeAsFlow()` |

### 다중 구독자 차이 예시

```kotlin
// SharedFlow: 모든 구독자가 동일 이벤트를 받음
val sharedFlow = MutableSharedFlow<Int>()

launch { sharedFlow.collect { println("구독자A: $it") } }
launch { sharedFlow.collect { println("구독자B: $it") } }

sharedFlow.emit(1)
// 출력: 구독자A: 1, 구독자B: 1 (둘 다 받음)
```

```kotlin
// Channel: 먼저 받아간 구독자만 값을 가져감
val channel = Channel<Int>()

launch { for (v in channel) println("구독자A: $v") }
launch { for (v in channel) println("구독자B: $v") }

channel.send(1)
// 출력: 구독자A: 1 (또는 구독자B: 1) — 둘 중 하나만 받음
```

---

## 5. 구독자 없을 때 동작 차이

```kotlin
// SharedFlow (replay=0): 구독자 없으면 이벤트 그냥 사라짐
val sharedFlow = MutableSharedFlow<String>()
sharedFlow.emit("이벤트")  // 구독자 없으면 손실

// Channel: 버퍼에 쌓여서 나중에 구독해도 받을 수 있음
val channel = Channel<String>(capacity = 10)
channel.send("이벤트")  // 버퍼에 저장
// 나중에 receive() 호출 시 받을 수 있음
```

이 차이 때문에 **Android에서 ViewModel → UI 이벤트 전달**에는 둘 다 사용 가능하지만 동작이 다릅니다.

---

## 6. Android에서 어떤 걸 써야 할까

### SharedFlow 권장 — 일반적인 선택

```kotlin
class MyViewModel : ViewModel() {
    private val _events = MutableSharedFlow<UiEvent>(
        extraBufferCapacity = 1,                    // 약간의 버퍼 허용
        onBufferOverflow = BufferOverflow.DROP_OLDEST
    )
    val events = _events.asSharedFlow()

    fun navigateToDetail(id: String) {
        viewModelScope.launch {
            _events.emit(UiEvent.NavigateToDetail(id))
        }
    }
}
```

- Compose와의 호환성이 좋음 (`collectAsStateWithLifecycle` 등)
- Flow 생태계의 다양한 연산자 활용 가능 (`map`, `filter` 등)
- 공식 Android 가이드에서 권장하는 방식

### Channel — 정확히 한 번 소비를 보장해야 할 때

```kotlin
class MyViewModel : ViewModel() {
    private val _events = Channel<UiEvent>(Channel.BUFFERED)
    val events = _events.receiveAsFlow()

    fun navigateToDetail(id: String) {
        viewModelScope.launch {
            _events.send(UiEvent.NavigateToDetail(id))
        }
    }
}
```

- 구독자가 일시적으로 없어도 이벤트가 버퍼에 남아 손실 없음
- 단, **구독자가 여러 개면 의도와 다르게 동작**할 수 있음 (하나만 받아감)

---

## 7. 실전 비교 표

| 상황 | 권장 |
|------|------|
| Compose에서 단일 화면 이벤트 처리 | `SharedFlow` |
| 여러 컴포넌트가 동일 이벤트를 구독해야 함 | `SharedFlow` |
| 이벤트 손실을 절대 허용 안 함 | `Channel` (버퍼 설정) |
| Flow 연산자(`map`, `filter`)와 자연스럽게 결합 | `SharedFlow` |
| 생산자-소비자 큐 패턴 (작업 분배) | `Channel` |

---

## 8. 정리

- **`SharedFlow`**: 다중 구독자에게 동일 이벤트 broadcast, Flow 생태계와 통합 우수, Android UI 이벤트의 표준 선택
- **`Channel`**: 단일 소비 보장, 버퍼링으로 손실 방지, 생산자-소비자 큐 패턴에 적합
- Android 일회성 UI 이벤트(토스트, 네비게이션)는 **`SharedFlow` + `extraBufferCapacity`** 조합이 가장 널리 쓰이는 패턴
- 정확히 한 명에게만, 정확히 한 번만 전달해야 하는 작업 분배 구조라면 `Channel`이 더 적합
