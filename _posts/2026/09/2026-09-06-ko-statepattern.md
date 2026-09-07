---
title: (Kotlin/코틀린) 행동 패턴 — 상태 패턴 (State)
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin으로 상태 패턴을 구현하는 방법과 sealed class를 활용한 FSM, 실전 주문 상태 관리 예제를 정리합니다.
---

---

## 1. 상태 패턴이란?

**객체의 내부 상태에 따라 행동이 달라지도록**, 각 상태를 별도 클래스로 분리하는 패턴입니다.
조건문(`when`/`if`) 분기 대신 각 상태 클래스가 해당 행동을 직접 담당합니다.
상태가 추가될 때 기존 코드를 수정하지 않고 새 클래스만 추가하면 됩니다.

```
Context ──state──→ State (interface)
                    ├── StateA → 이 상태일 때의 행동
                    ├── StateB → 이 상태일 때의 행동
                    └── StateC → 이 상태일 때의 행동
```

---

## 2. 기본 구현

신호등을 예로 들겠습니다. 각 신호등 상태(빨강/초록/노랑)가 다음 상태로의 전환을 스스로 알고 있습니다.

```kotlin
// 상태 인터페이스 — 현재 상태 표시와 다음 상태 전환 정의
interface TrafficLightState {
    fun display(): String
    fun next(): TrafficLightState  // 다음 상태를 반환
}

// 각 상태가 독립적인 클래스로 분리
class RedState : TrafficLightState {
    override fun display() = "🔴 정지"
    override fun next() = GreenState()  // 빨강 → 초록
}

class GreenState : TrafficLightState {
    override fun display() = "🟢 진행"
    override fun next() = YellowState() // 초록 → 노랑
}

class YellowState : TrafficLightState {
    override fun display() = "🟡 주의"
    override fun next() = RedState()    // 노랑 → 빨강 (순환)
}

// 컨텍스트: 현재 상태 객체를 가지고, 모든 행동을 상태에게 위임
class TrafficLight {
    private var state: TrafficLightState = RedState()  // 초기 상태: 빨강

    fun show() = println(state.display())

    // 상태 전환: 현재 상태가 다음 상태를 결정
    fun change() { state = state.next() }
}

val light = TrafficLight()
repeat(6) { light.show(); light.change() }
// 🔴 정지 → 🟢 진행 → 🟡 주의 → 🔴 정지 → 🟢 진행 → 🟡 주의
```

> **포인트**: if/when 없이도 상태에 따른 행동이 결정됩니다. 새 신호(예: 점멸 상태)를 추가해도 기존 상태 클래스는 수정하지 않아도 됩니다.

---

## 3. sealed class로 FSM (유한 상태 기계)

`sealed class`를 사용하면 가능한 상태를 컴파일 타임에 열거할 수 있어 `when`에서 누락 상태를 감지합니다.

```kotlin
sealed class OrderState {
    // 각 상태가 가능한 전환과 불가능한 전환을 스스로 정의
    abstract fun confirm(): OrderState
    abstract fun ship(): OrderState
    abstract fun deliver(): OrderState
    abstract fun cancel(): OrderState
    abstract val label: String

    // 주문 대기 상태: 확인 또는 취소만 가능
    object Pending : OrderState() {
        override val label = "주문 대기"
        override fun confirm() = Confirmed   // 확인 → Confirmed 상태로
        override fun ship()    = error("확인 전 배송 불가")  // 불가능한 전환
        override fun deliver() = error("배송 전 완료 불가")
        override fun cancel()  = Cancelled
    }

    // 확인 완료 상태: 배송 또는 취소 가능
    object Confirmed : OrderState() {
        override val label = "주문 확인"
        override fun confirm() = this        // 이미 확인된 상태 → 변화 없음
        override fun ship()    = Shipped
        override fun deliver() = error("배송 전 완료 불가")
        override fun cancel()  = Cancelled
    }

    // 배송 중 상태: 완료만 가능, 취소 불가
    object Shipped : OrderState() {
        override val label = "배송 중"
        override fun confirm() = this
        override fun ship()    = this
        override fun deliver() = Delivered
        override fun cancel()  = error("배송 중 취소 불가")
    }

    object Delivered : OrderState() {
        override val label = "배달 완료"
        override fun confirm() = this
        override fun ship()    = this
        override fun deliver() = this        // 이미 완료
        override fun cancel()  = error("완료된 주문 취소 불가")
    }

    object Cancelled : OrderState() {
        override val label = "취소됨"
        override fun confirm() = error("취소된 주문 확인 불가")
        override fun ship()    = error("취소된 주문 배송 불가")
        override fun deliver() = error("취소된 주문 완료 불가")
        override fun cancel()  = this
    }
}

class Order {
    var state: OrderState = OrderState.Pending
        private set

    fun confirm() = transition { state.confirm() }
    fun ship()    = transition { state.ship() }
    fun deliver() = transition { state.deliver() }
    fun cancel()  = transition { state.cancel() }

    // 상태 전환 시 예외를 잡아 사용자에게 친절한 메시지 제공
    private fun transition(next: () -> OrderState) {
        runCatching { state = next() }
            .onSuccess { println("상태: ${state.label}") }
            .onFailure { println("오류: ${it.message}") }
    }
}

val order = Order()
order.confirm()   // 상태: 주문 확인
order.ship()      // 상태: 배송 중
order.cancel()    // 오류: 배송 중 취소 불가  ← 불가능한 전환 차단
order.deliver()   // 상태: 배달 완료
```

---

## 4. 실전 예제 — 미디어 플레이어

상태 전환을 Context(MediaPlayer)에서 관리하는 방식입니다.

```kotlin
interface PlayerState {
    fun play(player: MediaPlayer)
    fun pause(player: MediaPlayer)
    fun stop(player: MediaPlayer)
}

class MediaPlayer {
    var state: PlayerState = StoppedState(this)

    fun play()  = state.play(this)
    fun pause() = state.pause(this)
    fun stop()  = state.stop(this)
    // 상태 전환: 각 State 구현체가 호출해 다음 상태로 변경
    fun setState(s: PlayerState) { state = s }
}

class PlayingState(private val player: MediaPlayer) : PlayerState {
    override fun play(p: MediaPlayer)  = println("이미 재생 중")
    override fun pause(p: MediaPlayer) {
        println("일시정지")
        p.setState(PausedState(player))  // Playing → Paused
    }
    override fun stop(p: MediaPlayer)  {
        println("정지")
        p.setState(StoppedState(player))  // Playing → Stopped
    }
}

class PausedState(private val player: MediaPlayer) : PlayerState {
    override fun play(p: MediaPlayer)  {
        println("재생 재개")
        p.setState(PlayingState(player))  // Paused → Playing
    }
    override fun pause(p: MediaPlayer) = println("이미 일시정지")
    override fun stop(p: MediaPlayer)  {
        println("정지")
        p.setState(StoppedState(player))  // Paused → Stopped
    }
}

class StoppedState(private val player: MediaPlayer) : PlayerState {
    override fun play(p: MediaPlayer)  {
        println("재생 시작")
        p.setState(PlayingState(player))  // Stopped → Playing
    }
    override fun pause(p: MediaPlayer) = println("재생 중이 아님")
    override fun stop(p: MediaPlayer)  = println("이미 정지됨")
}

val player = MediaPlayer()
player.play()   // 재생 시작  (Stopped → Playing)
player.pause()  // 일시정지   (Playing → Paused)
player.play()   // 재생 재개  (Paused → Playing)
player.stop()   // 정지       (Playing → Stopped)
player.pause()  // 재생 중이 아님  (Stopped 상태에서 pause는 무의미)
```

---

## 5. 정리

| 구현 방법 | 특징 |
|----------|------|
| 인터페이스 + 클래스 | 상태별 행동이 복잡할 때 적합 |
| sealed class | 가능한 상태를 컴파일 타임에 열거, `when` 누락 감지 |
| enum class | 상태별 행동이 단순하고 상태 수가 고정일 때 |

- 조건문 덩어리(`if/when`으로 상태 분기)가 반복된다면 상태 패턴 적용을 고려
- 상태 전환을 **State가 결정**하는 방식과 **Context가 결정**하는 방식 중 선택
- 실전 활용: 주문 FSM, 미디어 플레이어, 네트워크 연결 상태, UI 컴포넌트 상태
