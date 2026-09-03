---
title: (Kotlin/코틀린) 행동 패턴 — 전략 패턴 (Strategy)
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin으로 전략 패턴을 구현하는 방법과 함수 타입을 활용한 간결한 전략 교체, 실전 정렬/결제 전략 예제를 정리합니다.
---

---

## 1. 전략 패턴이란?

**알고리즘을 캡슐화하고 교체 가능하게 만드는** 패턴입니다.
Context(사용자)는 전략 인터페이스만 알고, 구체적인 알고리즘은 ConcreteStrategy가 담당합니다.
런타임에 전략을 바꿔 끼울 수 있어 조건문(`if/when`) 덩어리를 없앨 수 있습니다.

```
Context (컨텍스트)
    → Strategy (전략 인터페이스)
        → ConcreteStrategyA  ← 런타임에 선택
        → ConcreteStrategyB  ← 런타임에 선택
        → ConcreteStrategyC  ← 런타임에 선택
```

---

## 2. 인터페이스 기반 구현

가장 전통적인 방법입니다. 각 정렬 알고리즘을 별도 클래스로 구현하고, Context는 인터페이스만 바라봅니다.

```kotlin
// 전략 인터페이스 — 모든 정렬 알고리즘이 이 계약을 따름
interface SortStrategy {
    fun sort(data: MutableList<Int>): List<Int>
}

// 전략 구현체 A: 버블 정렬
class BubbleSort : SortStrategy {
    override fun sort(data: MutableList<Int>): List<Int> {
        println("버블 정렬 수행")
        // 인접한 두 원소를 비교해 더 큰 쪽을 뒤로 이동 (O(n²))
        for (i in data.indices) {
            for (j in 0 until data.size - i - 1) {
                if (data[j] > data[j + 1]) {
                    val tmp = data[j]; data[j] = data[j + 1]; data[j + 1] = tmp
                }
            }
        }
        return data
    }
}

// 전략 구현체 B: 퀵 정렬 (여기선 표준 라이브러리 사용)
class QuickSort : SortStrategy {
    override fun sort(data: MutableList<Int>): List<Int> {
        println("퀵 정렬 수행")
        return data.sorted()  // 실제로는 Timsort (O(n log n))
    }
}

// 컨텍스트: 전략 인터페이스만 알고, 구체 구현에 의존하지 않음
class Sorter(private var strategy: SortStrategy) {
    // 런타임에 전략 교체 가능
    fun setStrategy(strategy: SortStrategy) { this.strategy = strategy }
    fun sort(data: MutableList<Int>) = strategy.sort(data)
}

// 사용
val sorter = Sorter(BubbleSort())
sorter.sort(mutableListOf(3, 1, 4, 1, 5))  // 버블 정렬 수행

// 전략만 교체 — Sorter 코드는 전혀 바뀌지 않음
sorter.setStrategy(QuickSort())
sorter.sort(mutableListOf(3, 1, 4, 1, 5))  // 퀵 정렬 수행
```

---

## 3. 함수 타입으로 더 간결하게 (Kotlin 스타일)

Kotlin에서는 전략 인터페이스 대신 **함수 타입**을 사용하면 클래스 정의 없이도 전략을 교체할 수 있습니다. 단순한 알고리즘 교체라면 이 방식이 더 실용적입니다.

```kotlin
// 전략을 함수 타입 (MutableList<Int>) -> List<Int> 으로 표현
// → 인터페이스와 클래스 정의 없이 람다만으로 전략 구현 가능
class Sorter(private var strategy: (MutableList<Int>) -> List<Int>) {
    fun setStrategy(strategy: (MutableList<Int>) -> List<Int>) {
        this.strategy = strategy
    }
    fun sort(data: MutableList<Int>) = strategy(data)
}

// 람다로 전략 주입 — 별도 클래스 필요 없음
val sorter = Sorter { data -> data.also { it.sort() } }
sorter.sort(mutableListOf(5, 3, 1, 4, 2))  // 오름차순

// 람다로 전략 교체 — 내림차순으로 변경
sorter.setStrategy { data -> data.sortedDescending() }
sorter.sort(mutableListOf(5, 3, 1, 4, 2))
```

> **인터페이스 vs 함수 타입**: 전략이 단순하고 상태가 없다면 함수 타입이 간결합니다. 전략 자체가 상태(멤버 변수)를 가진다면 인터페이스 + 클래스가 적합합니다.

---

## 4. 실전 예제 — 결제 전략

전략 패턴이 가장 자주 쓰이는 실전 사례입니다. 결제 수단(신용카드, 카카오페이 등)을 전략으로 분리하면, 새 결제 수단 추가 시 기존 코드를 수정할 필요 없습니다(OCP).

```kotlin
// 결제 전략 인터페이스
interface PaymentStrategy {
    fun pay(amount: Int): String
}

// 전략 구현체들 — 결제 수단마다 별도 클래스
class CreditCard(private val cardNumber: String) : PaymentStrategy {
    override fun pay(amount: Int) = "신용카드(${cardNumber.takeLast(4)})로 ${amount}원 결제"
    // takeLast(4): 카드번호 마지막 4자리만 표시 (보안)
}

class KakaoPay(private val userId: String) : PaymentStrategy {
    override fun pay(amount: Int) = "카카오페이($userId)로 ${amount}원 결제"
}

class NaverPay : PaymentStrategy {
    override fun pay(amount: Int) = "네이버페이로 ${amount}원 결제"
}

class ShoppingCart {
    private val items = mutableListOf<Pair<String, Int>>()
    private var paymentStrategy: PaymentStrategy? = null

    fun addItem(name: String, price: Int) = items.add(name to price)

    // 결제 수단을 런타임에 주입
    fun setPayment(strategy: PaymentStrategy) { paymentStrategy = strategy }

    fun checkout(): String {
        val total = items.sumOf { it.second }
        // 결제 수단이 없으면 안내 메시지, 있으면 실제 결제
        return paymentStrategy?.pay(total) ?: "결제 수단을 선택하세요"
    }
}

val cart = ShoppingCart()
cart.addItem("키보드", 80000)
cart.addItem("마우스", 30000)

// 신용카드로 결제
cart.setPayment(CreditCard("1234-5678-9012-3456"))
println(cart.checkout())  // 신용카드(3456)로 110000원 결제

// 결제 수단만 바꿔치기 — ShoppingCart 코드 수정 없음
cart.setPayment(KakaoPay("user@kakao"))
println(cart.checkout())  // 카카오페이(user@kakao)로 110000원 결제
```

---

## 5. sealed class + when으로 전략 열거

가능한 전략이 미리 정해져 있고 타입 안전성이 중요할 때 `sealed class`를 사용합니다.
새 전략을 추가하면 `when`에서 컴파일 오류가 발생해 누락을 방지합니다.

```kotlin
// 할인 전략을 sealed class로 열거 — 외부에서 임의 추가 불가
sealed class DiscountStrategy {
    abstract fun calculate(price: Int): Int

    // 할인 없음
    object None : DiscountStrategy() {
        override fun calculate(price: Int) = price
    }

    // 퍼센트 할인 (예: 10% → 90% 가격)
    data class Percentage(val percent: Int) : DiscountStrategy() {
        override fun calculate(price: Int) = price * (100 - percent) / 100
    }

    // 정액 할인 (예: 5000원 할인, 최소 0원)
    data class FixedAmount(val amount: Int) : DiscountStrategy() {
        override fun calculate(price: Int) = maxOf(0, price - amount)
    }

    // 회원 등급별 차등 할인
    object MemberDiscount : DiscountStrategy() {
        override fun calculate(price: Int) = when {
            price >= 100000 -> price * 80 / 100  // 10만원 이상: 20% 할인
            price >= 50000  -> price * 90 / 100  // 5만원 이상: 10% 할인
            else            -> price              // 미만: 할인 없음
        }
    }
}

fun applyDiscount(price: Int, strategy: DiscountStrategy): Int =
    strategy.calculate(price)

println(applyDiscount(100000, DiscountStrategy.Percentage(10)))    // 90000
println(applyDiscount(100000, DiscountStrategy.FixedAmount(5000))) // 95000
println(applyDiscount(100000, DiscountStrategy.MemberDiscount))    // 80000
println(applyDiscount(100000, DiscountStrategy.None))              // 100000
```

---

## 6. 정리

- 전략 패턴: `if/when` 분기 덩어리 → 각 전략 클래스로 분리
- Kotlin에서는 **함수 타입**으로 전략 인터페이스를 대체해 코드량 절감 가능
- `sealed class`: 전략 집합이 고정된 경우 `when`의 컴파일러 검증까지 활용
- OCP(개방-폐쇄 원칙): 새 전략 추가 시 기존 코드 수정 없이 클래스만 추가
- 결제, 정렬, 압축, 검증, 렌더링 등 알고리즘이 교체될 가능성이 있는 모든 곳에 적용
