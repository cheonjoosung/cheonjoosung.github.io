---
title: (Kotlin/코틀린) ObservableProperty / VetoableProperty 완전 정리
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin 표준 라이브러리의 Delegates.observable과 Delegates.vetoable을 활용해 프로퍼티 변경을 관찰하고 제어하는 패턴을 정리합니다.
---

---

## 1. Delegates.observable

값이 변경될 때마다 **변경 후** 콜백을 받습니다.

```kotlin
import kotlin.properties.Delegates

var name: String by Delegates.observable("초기값") { property, oldValue, newValue ->
    println("${property.name}: '$oldValue' → '$newValue'")
}

name = "Alice"  // name: '초기값' → 'Alice'
name = "Bob"    // name: 'Alice' → 'Bob'
```

```kotlin
// 람다 파라미터
Delegates.observable(initialValue) { property, old, new ->
    // property: KProperty<*> — 프로퍼티 메타데이터 (이름 등)
    // old: 이전 값
    // new: 새 값
    // 반환값 없음 (Unit)
}
```

---

## 2. Delegates.vetoable

변경 **전** 콜백이 호출되며, `false`를 반환하면 **변경이 거부**됩니다.

```kotlin
var positiveNumber: Int by Delegates.vetoable(0) { property, oldValue, newValue ->
    newValue > 0  // true → 변경 허용, false → 거부 (oldValue 유지)
}

positiveNumber = 10   // OK: 10 > 0
println(positiveNumber)  // 10

positiveNumber = -5   // 거부: -5 > 0 이 false
println(positiveNumber)  // 10 (변경 안 됨)
```

---

## 3. observable vs vetoable 비교

| 구분 | observable | vetoable |
|------|-----------|----------|
| 콜백 시점 | 변경 **후** | 변경 **전** |
| 반환 타입 | `Unit` | `Boolean` |
| 변경 제어 | 불가 | 가능 (false 반환 시 거부) |
| 주요 용도 | 로깅, UI 업데이트 | 유효성 검사, 비즈니스 규칙 |

---

## 4. 실전 패턴 — UI 상태 자동 갱신

```kotlin
class ProfileViewModel {
    var username: String by Delegates.observable("") { _, _, newValue ->
        updateUsernameUI(newValue)
    }

    var profileImage: String by Delegates.observable("") { _, _, newValue ->
        loadImage(newValue)
    }

    var followerCount: Int by Delegates.observable(0) { _, oldValue, newValue ->
        if (newValue > oldValue) {
            showFollowerAnimation()
        }
        updateFollowerCountUI(newValue)
    }
}
```

---

## 5. 실전 패턴 — 변경 이력 추적

```kotlin
class AuditedProperty<T>(initialValue: T) {
    private val history = mutableListOf<Pair<T, Long>>()

    var value: T by Delegates.observable(initialValue) { _, _, newValue ->
        history.add(newValue to System.currentTimeMillis())
    }

    fun getHistory(): List<Pair<T, Long>> = history.toList()
}

val price = AuditedProperty(1000)
price.value = 1200
price.value = 900
price.value = 1100

price.getHistory().forEach { (value, timestamp) ->
    println("$value (${java.util.Date(timestamp)})")
}
// 1200 (...)
// 900 (...)
// 1100 (...)
```

---

## 6. 실전 패턴 — vetoable로 비즈니스 규칙 적용

```kotlin
class BankAccount(initialBalance: Double) {
    var balance: Double by Delegates.vetoable(initialBalance) { _, _, newBalance ->
        if (newBalance < 0) {
            println("잔액 부족: 출금 거부 (현재: $balance, 요청 후: $newBalance)")
            false
        } else {
            true
        }
    }

    fun deposit(amount: Double) { balance += amount }
    fun withdraw(amount: Double) { balance -= amount }
}

val account = BankAccount(1000.0)
account.deposit(500.0)   // balance: 1500.0
account.withdraw(300.0)  // balance: 1200.0
account.withdraw(2000.0) // 잔액 부족: 출금 거부 (현재: 1200.0, 요청 후: -800.0)
println(account.balance) // 1200.0 (변경 안 됨)
```

---

## 7. 실전 패턴 — 단계 전환 제어

```kotlin
enum class OrderStatus {
    PENDING, CONFIRMED, SHIPPED, DELIVERED, CANCELLED
}

class Order {
    var status: OrderStatus by Delegates.vetoable(OrderStatus.PENDING) { _, old, new ->
        val validTransitions = mapOf(
            OrderStatus.PENDING   to setOf(OrderStatus.CONFIRMED, OrderStatus.CANCELLED),
            OrderStatus.CONFIRMED to setOf(OrderStatus.SHIPPED, OrderStatus.CANCELLED),
            OrderStatus.SHIPPED   to setOf(OrderStatus.DELIVERED),
            OrderStatus.DELIVERED to emptySet(),
            OrderStatus.CANCELLED to emptySet()
        )
        val allowed = validTransitions[old]?.contains(new) ?: false
        if (!allowed) println("유효하지 않은 상태 전환: $old → $new")
        allowed
    }
}

val order = Order()
order.status = OrderStatus.CONFIRMED  // OK
order.status = OrderStatus.DELIVERED  // 유효하지 않은 상태 전환: CONFIRMED → DELIVERED
println(order.status)                 // CONFIRMED
order.status = OrderStatus.SHIPPED    // OK
order.status = OrderStatus.DELIVERED  // OK
```

---

## 8. 커스텀 ObservableProperty 구현

표준 라이브러리 내부를 이해하면 더 유연한 변형을 만들 수 있습니다.

```kotlin
// 변경 시 여러 리스너에게 알리는 델리게이트
class MultiObservableDelegate<T>(private var value: T) {
    private val listeners = mutableListOf<(T, T) -> Unit>()

    fun addListener(listener: (old: T, new: T) -> Unit) {
        listeners.add(listener)
    }

    operator fun getValue(thisRef: Any?, property: KProperty<*>): T = value

    operator fun setValue(thisRef: Any?, property: KProperty<*>, newValue: T) {
        val old = value
        value = newValue
        listeners.forEach { it(old, newValue) }
    }
}

class Settings {
    val theme = MultiObservableDelegate("light")

    init {
        theme.addListener { old, new -> println("테마 변경: $old → $new") }
        theme.addListener { _, new -> saveToPrefs("theme", new) }
    }

    var themeValue: String by theme
}
```

---

## 9. 정리

- **`Delegates.observable`**: 값 변경 후 콜백 — 로깅, UI 업데이트, 이력 추적
- **`Delegates.vetoable`**: 값 변경 전 콜백 — `false` 반환으로 변경 거부 가능, 유효성 검사, 상태 전환 제어
- 두 델리게이트 모두 `kotlin.properties.Delegates`에서 제공
- 복잡한 요구사항은 `ObservableProperty`를 상속하거나 `getValue`/`setValue` 직접 구현으로 확장
