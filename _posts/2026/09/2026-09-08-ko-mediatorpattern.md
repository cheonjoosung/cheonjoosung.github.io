---
title: (Kotlin/코틀린) 행동 패턴 — 중재자 패턴 (Mediator)
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin으로 중재자 패턴을 구현하는 방법과 채팅방, UI 컴포넌트 조율, 항공 관제 시스템 실전 예제를 정리합니다.
---

---

## 1. 중재자 패턴이란?

**객체 간의 직접 통신을 없애고 중재자를 통해 소통**하게 하는 패턴입니다.
복잡한 N:N 관계를 N:1 (컴포넌트:중재자) 관계로 단순화합니다.

```
Without Mediator:  A ↔ B ↔ C ↔ D  (모든 객체가 서로를 직접 참조 → 복잡)
With Mediator:         M             (모든 객체가 중재자만 참조)
                      /|\ 
                     A B C D
```

컴포넌트들은 서로를 모르고 오직 중재자만 알면 됩니다. 이는 에어트래픽 관제탑과 비슷합니다—비행기들이 서로 직접 통신하지 않고 관제탑을 통해 소통합니다.

---

## 2. 기본 구현 — 채팅방

채팅방은 중재자 패턴의 가장 대표적인 예입니다.
사용자들이 서로를 직접 알지 않아도, 채팅방(중재자)을 통해 메시지를 주고받습니다.

```kotlin
// 중재자 인터페이스
interface ChatMediator {
    fun sendMessage(sender: User, message: String)
    fun addUser(user: User)
}

// 컴포넌트: 각 User는 중재자(ChatMediator)만 참조, 다른 User는 모름
class User(val name: String, private val mediator: ChatMediator) {
    // 메시지 전송 시 중재자에게 위임 (다른 User에게 직접 보내지 않음)
    fun send(message: String) = mediator.sendMessage(this, message)
    fun receive(sender: String, message: String) = println("[$name] $sender: $message")
}

// 구체 중재자: 메시지를 받아 나머지 모든 사용자에게 전달
class ChatRoom : ChatMediator {
    private val users = mutableListOf<User>()

    override fun addUser(user: User) = users.add(user).let { }

    override fun sendMessage(sender: User, message: String) {
        // sender를 제외한 모든 사용자에게 메시지 전달
        users.filter { it != sender }
             .forEach { it.receive(sender.name, message) }
    }
}

val room = ChatRoom()
val alice = User("Alice", room)
val bob   = User("Bob",   room)
val carol = User("Carol", room)

room.addUser(alice); room.addUser(bob); room.addUser(carol)

alice.send("안녕하세요!")
// [Bob] Alice: 안녕하세요!
// [Carol] Alice: 안녕하세요!
// (Alice는 자신이 보낸 메시지를 받지 않음)
```

---

## 3. 실전 예제 — UI 컴포넌트 조율

체크박스 선택 시 입력 필드와 버튼이 활성화되는 UI 상호작용을 중재자로 구현합니다.
각 컴포넌트가 직접 서로를 참조하면 결합도가 높아지지만, 중재자를 통하면 독립적으로 교체할 수 있습니다.

```kotlin
interface UIMediator {
    fun componentChanged(component: UIComponent)
}

// 모든 UI 컴포넌트의 공통 기반 — 중재자만 알고 있음
abstract class UIComponent(protected val mediator: UIMediator) {
    abstract val name: String
}

class Checkbox(mediator: UIMediator, private var checked: Boolean = false) : UIComponent(mediator) {
    override val name = "Checkbox"

    fun toggle() {
        checked = !checked
        println("Checkbox: ${if (checked) "체크됨" else "해제됨"}")
        // 상태가 변경되었음을 중재자에게 알림
        mediator.componentChanged(this)
    }

    fun isChecked() = checked
}

class TextField(mediator: UIMediator, private var enabled: Boolean = false) : UIComponent(mediator) {
    override val name = "TextField"

    fun setEnabled(enabled: Boolean) {
        this.enabled = enabled
        println("TextField: ${if (enabled) "활성화" else "비활성화"}")
    }
}

class SubmitButton(mediator: UIMediator, private var enabled: Boolean = false) : UIComponent(mediator) {
    override val name = "SubmitButton"

    fun setEnabled(enabled: Boolean) {
        this.enabled = enabled
        println("SubmitButton: ${if (enabled) "활성화" else "비활성화"}")
    }
}

// 구체 중재자: 컴포넌트 변경 시 다른 컴포넌트에게 영향을 조율
class FormMediator : UIMediator {
    // 중재자가 모든 컴포넌트를 알고 있음 (컴포넌트들끼리는 서로 모름)
    lateinit var checkbox: Checkbox
    lateinit var textField: TextField
    lateinit var submitButton: SubmitButton

    override fun componentChanged(component: UIComponent) {
        when (component) {
            // Checkbox가 바뀌면 → TextField와 SubmitButton 활성화/비활성화
            is Checkbox -> {
                textField.setEnabled(component.isChecked())
                submitButton.setEnabled(component.isChecked())
            }
            // 다른 컴포넌트 변경 시 추가 로직...
        }
    }
}

val mediator = FormMediator()
val checkbox = Checkbox(mediator)
val textField = TextField(mediator)
val submit = SubmitButton(mediator)

// 중재자에게 각 컴포넌트 등록
mediator.checkbox = checkbox
mediator.textField = textField
mediator.submitButton = submit

checkbox.toggle()
// Checkbox: 체크됨
// TextField: 활성화
// SubmitButton: 활성화

checkbox.toggle()
// Checkbox: 해제됨
// TextField: 비활성화
// SubmitButton: 비활성화
```

---

## 4. 실전 예제 — 항공 관제탑

비행기들이 서로 직접 통신하지 않고 관제탑을 통해 활주로 사용을 조율합니다.

```kotlin
interface AirTrafficControl {
    fun requestLanding(aircraft: Aircraft)
    fun notifyDeparture(aircraft: Aircraft)
    val runway: String
}

// 각 Aircraft는 관제탑(AirTrafficControl)만 알면 됨
abstract class Aircraft(val id: String, protected val atc: AirTrafficControl) {
    fun land() = atc.requestLanding(this)
    fun depart() = atc.notifyDeparture(this)
}

class ControlTower(override val runway: String) : AirTrafficControl {
    private var runwayBusy = false  // 활주로 사용 현황 (중재자가 공유 상태 관리)

    override fun requestLanding(aircraft: Aircraft) {
        if (runwayBusy) {
            // 활주로 사용 중이면 대기 요청
            println("[관제탑] ${aircraft.id}: 대기하세요. 활주로 사용 중")
        } else {
            runwayBusy = true
            println("[관제탑] ${aircraft.id}: 착륙 허가. 활주로 $runway")
        }
    }

    override fun notifyDeparture(aircraft: Aircraft) {
        println("[관제탑] ${aircraft.id}: 이륙 확인")
        runwayBusy = false  // 이륙 완료 → 활주로 해제
    }
}

class Airplane(id: String, atc: AirTrafficControl) : Aircraft(id, atc)

val tower = ControlTower("28L")
val ka001 = Airplane("KA001", tower)
val oz002 = Airplane("OZ002", tower)

ka001.land()    // [관제탑] KA001: 착륙 허가. 활주로 28L
oz002.land()    // [관제탑] OZ002: 대기하세요. 활주로 사용 중
ka001.depart()  // [관제탑] KA001: 이륙 확인 → 활주로 해제
oz002.land()    // [관제탑] OZ002: 착륙 허가. 활주로 28L
```

---

## 5. 이벤트 버스로 중재자 구현

더 느슨한 결합을 원한다면 이벤트 버스 방식을 사용합니다.
구독자가 서로를 전혀 몰라도 이벤트로 통신할 수 있습니다.

```kotlin
class EventBusMediator {
    // 이벤트 이름 → 구독자 목록 (Map으로 관리)
    private val listeners = mutableMapOf<String, MutableList<(Any) -> Unit>>()

    // 이벤트 구독
    fun subscribe(event: String, listener: (Any) -> Unit) {
        // getOrPut: 없으면 빈 리스트 생성 후 추가
        listeners.getOrPut(event) { mutableListOf() }.add(listener)
    }

    // 이벤트 발행: 해당 이벤트의 모든 구독자에게 데이터 전달
    fun publish(event: String, data: Any) {
        listeners[event]?.forEach { it(data) }
    }
}

val bus = EventBusMediator()

// 같은 이벤트에 여러 구독자 등록 가능 — 서로를 모름
bus.subscribe("user.login")  { data -> println("Analytics: 로그인 - $data") }
bus.subscribe("user.login")  { data -> println("Session: 세션 생성 - $data") }
bus.subscribe("user.logout") { data -> println("Session: 세션 삭제 - $data") }

bus.publish("user.login",  "user123")
// Analytics: 로그인 - user123
// Session: 세션 생성 - user123

bus.publish("user.logout", "user123")
// Session: 세션 삭제 - user123
```

---

## 6. 정리

| 방식 | 결합도 | 특징 |
|------|--------|------|
| 직접 중재자 | 낮음 | 컴포넌트가 중재자 인터페이스를 참조 |
| 이벤트 버스 | 매우 낮음 | 발행자와 구독자가 서로 전혀 모름 |

- 중재자 패턴: 컴포넌트 간 직접 참조를 제거해 결합도를 낮춤
- **주의**: 중재자가 너무 많은 로직을 담으면 God Object가 됨 → 도메인별 중재자 분리 권장
- 실전 활용: 채팅방, UI 폼 조율, 항공 관제탑, 게임 이벤트 시스템
