---
title: (Kotlin/코틀린) 프로토타입 패턴(Prototype Pattern) 완전 정리
tags: [ Kotlin, Android ]
style: fill
color: dark
description: Kotlin data class copy()를 활용한 프로토타입 패턴, 얕은 복사와 깊은 복사의 차이, 확장 함수 deepCopy, Android MVI State 복사까지 깊이 있게 정리합니다.
---

## 개요

- 생성 패턴(Creational Pattern) 중 **프로토타입 패턴(Prototype Pattern)** 을 다룹니다.
- 프로토타입 패턴은 **기존 객체를 복사(Clone)해 새 객체를 생성** 하는 패턴입니다.
- 객체 생성 비용이 크거나, 비슷한 객체를 여럿 만들어야 할 때 유용합니다.
- 이 글에서는 다음을 설명합니다.
  - 프로토타입 패턴이 필요한 이유
  - 얕은 복사(Shallow Copy) vs 깊은 복사(Deep Copy)
  - `data class copy()` 활용과 한계
  - 깊은 복사 — 직접 구현 및 확장 함수
  - Android 실전 예제 (MVI State, 설정 템플릿, 알림)

---

## 1. 왜 프로토타입 패턴이 필요한가

### ❌ 문제 — 비용이 큰 객체를 매번 새로 생성

```kotlin
// DB에서 수만 건을 조회하고 복잡한 계산을 거쳐 Report 생성
fun createMonthlyReport(month: Int): Report {
    val rawData       = database.fetchAllTransactions(month)  // 느린 조회
    val processedData = heavyProcessing(rawData)              // 복잡한 계산
    return Report(month, processedData)
}

// 같은 데이터인데 팀별로 반복 생성 — 낭비
val hrReport      = createMonthlyReport(4)
val financeReport = createMonthlyReport(4)
val ceoReport     = createMonthlyReport(4)
```

---

### ✅ 프로토타입으로 해결 — 한 번 만들고 복사

```kotlin
// 한 번만 생성
val original = createMonthlyReport(4)

// 복사해서 팀별 커스터마이징
val hrReport      = original.copy(recipient = "인사팀")
val financeReport = original.copy(recipient = "재무팀")
val ceoReport     = original.copy(recipient = "대표이사", isConfidential = true)
```

**프로토타입이 적합한 경우:**

```
✔ 객체 생성 비용이 큰 경우 (DB 조회, 네트워크, 복잡한 계산)
✔ 비슷한 객체를 여럿 만들어야 하는 경우
✔ 객체의 초기 설정이 복잡해 재사용하고 싶은 경우
✔ 불변 상태를 유지하면서 일부 값만 변경하고 싶은 경우 (MVI State)
```

---

## 2. 얕은 복사 vs 깊은 복사

프로토타입 패턴에서 가장 중요한 개념입니다.

### 얕은 복사 (Shallow Copy)

```kotlin
data class Address(var city: String, var street: String)
data class Person(val name: String, val address: Address)

val original = Person("홍길동", Address("서울", "강남대로"))
val shallow  = original.copy()  // data class 기본 copy() — 얕은 복사

// 기본 타입 필드 — 독립적으로 복사됨
println(original.name)  // 홍길동
println(shallow.name)   // 홍길동 (독립)

// 참조 타입 필드 — 같은 객체를 공유
shallow.address.city = "부산"
println(original.address.city)  // 부산 ❌ — original도 변경됨
println(shallow.address.city)   // 부산
```

```
얕은 복사:
original ─→ [name: "홍길동"] [address ─→ Address("서울", "강남대로")]
shallow  ─→ [name: "홍길동"] [address ─┘]  ← 같은 Address 객체 참조
```

---

### 깊은 복사 (Deep Copy)

```kotlin
val original = Person("홍길동", Address("서울", "강남대로"))

// 중첩 객체까지 새로 생성
val deep = original.copy(address = original.address.copy())

deep.address.city = "부산"
println(original.address.city)  // 서울 ✅ — original 영향 없음
println(deep.address.city)      // 부산
```

```
깊은 복사:
original ─→ [name: "홍길동"] [address ─→ Address("서울", "강남대로")]
deep     ─→ [name: "홍길동"] [address ─→ Address("부산", "강남대로")]  ← 별개 객체
```

---

## 3. data class copy() — 기본 사용

`data class`는 `copy()`를 자동 생성합니다. **모든 필드가 불변(val)이라면 얕은 복사로도 안전합니다.**

```kotlin
data class UserConfig(
    val name: String,
    val theme: String = "light",
    val fontSize: Int = 14,
    val language: String = "ko",
    val notificationsEnabled: Boolean = true
)

val defaultConfig = UserConfig(name = "기본 설정")

// 일부만 변경해 새 객체 생성 — 원본 불변 유지
val darkConfig  = defaultConfig.copy(theme = "dark")
val largeConfig = defaultConfig.copy(fontSize = 18, language = "en")
val muteConfig  = defaultConfig.copy(notificationsEnabled = false)

println(defaultConfig.theme)   // light ✅ 원본 불변
println(darkConfig.theme)      // dark
println(largeConfig.fontSize)  // 18
```

---

## 4. 중첩 참조 타입 — copy()의 한계

참조 타입 필드가 **가변(var)** 이거나 **가변 컬렉션**이면 얕은 복사가 위험합니다.

```kotlin
data class Tag(val id: Int, var name: String)  // name이 가변

data class Article(
    val title: String,
    val tags: MutableList<Tag>
)

val original = Article("Kotlin 정리", mutableListOf(Tag(1, "Kotlin"), Tag(2, "Android")))

// ❌ 얕은 복사 — tags 리스트를 공유
val shallow = original.copy()
shallow.tags.add(Tag(3, "Design Pattern"))

println(original.tags.size)  // 3 ❌ — original도 변경됨
```

---

## 5. 깊은 복사 구현 방법

### 방법 1 — copy() 중첩 호출

```kotlin
data class Address(val city: String, val street: String)

data class User(
    val name: String,
    val address: Address,
    val hobbies: List<String>
)

val original = User("홍길동", Address("서울", "강남대로"), listOf("독서", "등산"))

// 참조 타입마다 copy() 중첩 호출
val deep = original.copy(
    address = original.address.copy(),
    hobbies = original.hobbies.toList()  // 새 리스트 생성
)
```

---

### 방법 2 — deepCopy 확장 함수

복잡한 중첩 구조라면 확장 함수로 의도를 명확히 표현합니다.

```kotlin
data class FilterOption(
    val category: String,
    val priceRange: IntRange,
    val selectedBrands: MutableList<String> = mutableListOf(),
    val sortOrder: String = "newest"
)

fun FilterOption.deepCopy() = FilterOption(
    category       = category,
    priceRange     = priceRange,
    selectedBrands = selectedBrands.toMutableList(),  // 새 리스트
    sortOrder      = sortOrder
)

val defaultFilter = FilterOption(
    category       = "전체",
    priceRange     = 0..100_000,
    selectedBrands = mutableListOf("Nike", "Adidas")
)

val myFilter = defaultFilter.deepCopy()
myFilter.selectedBrands.add("Puma")

println(defaultFilter.selectedBrands)  // [Nike, Adidas] ✅
println(myFilter.selectedBrands)       // [Nike, Adidas, Puma]
```

---

### 방법 3 — 불변 컬렉션으로 설계 (권장)

가장 근본적인 해결책은 **가변 타입을 쓰지 않는 것**입니다.

```kotlin
// ✅ 모든 필드를 불변으로 — copy()만으로 안전
data class FilterOption(
    val category: String,
    val priceRange: IntRange,
    val selectedBrands: List<String> = emptyList(),  // 불변 List
    val sortOrder: String = "newest"
)

val defaultFilter = FilterOption("전체", 0..100_000, listOf("Nike", "Adidas"))

// 새 브랜드 추가 — 기존 리스트 + 새 항목으로 새 리스트 생성
val myFilter = defaultFilter.copy(
    selectedBrands = defaultFilter.selectedBrands + "Puma"
)

println(defaultFilter.selectedBrands)  // [Nike, Adidas] ✅
println(myFilter.selectedBrands)       // [Nike, Adidas, Puma]
```

---

## 6. 프로토타입 레지스트리

복사할 원본 객체들을 미리 등록해두고 이름으로 꺼내 쓰는 방식입니다.

```kotlin
data class NotificationTemplate(
    val title: String,
    val body: String,
    val channelId: String,
    val priority: Int = 0
)

object NotificationRegistry {
    private val templates = mapOf(
        "order_complete" to NotificationTemplate(
            title     = "주문 완료",
            body      = "주문이 정상 처리되었습니다",
            channelId = "ORDER",
            priority  = 1
        ),
        "delivery_start" to NotificationTemplate(
            title     = "배송 시작",
            body      = "상품이 출발했습니다",
            channelId = "DELIVERY",
            priority  = 1
        ),
        "promotion" to NotificationTemplate(
            title     = "혜택 안내",
            body      = "새로운 프로모션이 시작되었습니다",
            channelId = "MARKETING",
            priority  = 0
        )
    )

    // 복사본 반환 — 원본 보호
    fun get(key: String): NotificationTemplate =
        templates[key]?.copy() ?: error("템플릿 없음: $key")
}

// 사용 — 템플릿을 복사해 커스터마이징
fun sendOrderNotification(orderId: String) {
    val notification = NotificationRegistry.get("order_complete")
        .copy(body = "주문 #$orderId 이 완료되었습니다")
    // 발송 처리
}

fun sendDeliveryNotification(trackingNumber: String) {
    val notification = NotificationRegistry.get("delivery_start")
        .copy(body = "운송장 번호: $trackingNumber")
    // 발송 처리
}
```

---

## 7. Android 실전 예제 ① — MVI State 복사

MVI 패턴의 상태 변경이 프로토타입 패턴의 가장 자연스러운 Android 활용 사례입니다.

```kotlin
data class CartState(
    val isLoading: Boolean = false,
    val items: List<CartItem> = emptyList(),
    val totalPrice: Int = 0,
    val couponCode: String? = null,
    val discountAmount: Int = 0,
    val error: String? = null
)

data class CartItem(
    val productId: Long,
    val name: String,
    val price: Int,
    val quantity: Int  // 불변
)

class CartViewModel : ViewModel() {
    private val _state = MutableStateFlow(CartState())
    val state: StateFlow<CartState> = _state.asStateFlow()

    fun addItem(newItem: CartItem) {
        _state.update { current ->
            val updatedItems = current.items + newItem           // 새 리스트
            current.copy(                                        // 프로토타입 ✅
                items      = updatedItems,
                totalPrice = updatedItems.sumOf { it.price * it.quantity }
            )
        }
    }

    fun updateQuantity(productId: Long, quantity: Int) {
        _state.update { current ->
            val updatedItems = current.items.map { item ->
                if (item.productId == productId) item.copy(quantity = quantity)  // ✅
                else item
            }
            current.copy(
                items      = updatedItems,
                totalPrice = updatedItems.sumOf { it.price * it.quantity }
            )
        }
    }

    fun removeItem(productId: Long) {
        _state.update { current ->
            val updatedItems = current.items.filter { it.productId != productId }
            current.copy(
                items      = updatedItems,
                totalPrice = updatedItems.sumOf { it.price * it.quantity }
            )
        }
    }

    fun applyCoupon(code: String, discount: Int) {
        _state.update { it.copy(couponCode = code, discountAmount = discount) }
    }
}
```

---

## 8. Android 실전 예제 ② — 알림 설정 템플릿

```kotlin
data class PushConfig(
    val channelId: String,
    val smallIcon: Int,
    val importance: Int      = NotificationManager.IMPORTANCE_DEFAULT,
    val autoCancel: Boolean  = true,
    val vibrate: Boolean     = true,
    val sound: Boolean       = true,
    val badgeCount: Int      = 0
)

object PushConfigFactory {
    private val orderBase = PushConfig(
        channelId  = "ORDER",
        smallIcon  = R.drawable.ic_order,
        importance = NotificationManager.IMPORTANCE_HIGH
    )

    private val chatBase = PushConfig(
        channelId  = "CHAT",
        smallIcon  = R.drawable.ic_chat,
        importance = NotificationManager.IMPORTANCE_HIGH
    )

    private val promoBase = PushConfig(
        channelId  = "PROMO",
        smallIcon  = R.drawable.ic_promo,
        importance = NotificationManager.IMPORTANCE_LOW,
        vibrate    = false
    )

    fun order(badgeCount: Int = 0) = orderBase.copy(badgeCount = badgeCount)
    fun chat(sound: Boolean = true) = chatBase.copy(sound = sound)
    fun promo() = promoBase.copy()
}

// 사용
val config = PushConfigFactory.order(badgeCount = 3)
buildNotification(context, config)
```

---

## 9. 얕은 복사 vs 깊은 복사 정리

| 항목 | 얕은 복사 | 깊은 복사 |
|------|-----------|-----------|
| 기본 타입 (Int, String 등) | 독립 복사 | 독립 복사 |
| 참조 타입 (객체, 컬렉션) | 같은 객체 공유 | 새 객체 생성 |
| 성능 | 빠름 | 느림 (중첩 깊이에 비례) |
| 안전성 | 가변 필드 포함 시 부작용 위험 | 완전 독립 |
| Kotlin 기본 | `data class copy()` | 직접 구현 / 확장 함수 |
| 안전한 경우 | **모든 필드가 불변(val)** 일 때 | 가변 필드 포함 시 |

---

## 10. copy() 사용 시 주의사항

### ❌ 가변 컬렉션을 val로 선언해도 내부는 가변

```kotlin
data class Bag(val items: MutableList<String>)  // val이지만 내부는 가변

val original = Bag(mutableListOf("A", "B"))
val copy     = original.copy()

copy.items.add("C")
println(original.items)  // [A, B, C] ❌ — 공유됨
```

```kotlin
// ✅ 불변 List 사용
data class Bag(val items: List<String>)

val original = Bag(listOf("A", "B"))
val copy     = original.copy(items = original.items + "C")
println(original.items)  // [A, B] ✅
```

---

### ❌ 중첩 depth가 깊을 때 copy() 연쇄

```kotlin
// 3단계 중첩 — copy() 연쇄가 장황해짐
val updated = order.copy(
    customer = order.customer.copy(
        address = order.customer.address.copy(city = "부산")
    )
)

// ✅ 중간 변수로 가독성 개선
val updatedAddress  = order.customer.address.copy(city = "부산")
val updatedCustomer = order.customer.copy(address = updatedAddress)
val updatedOrder    = order.copy(customer = updatedCustomer)
```

---

## 11. 정리

| 항목 | 내용 |
|------|------|
| 목적 | 기존 객체를 복사해 새 객체를 효율적으로 생성 |
| Kotlin 기본 도구 | `data class copy()` — 얕은 복사 |
| 깊은 복사 | `copy()` 중첩 / `deepCopy()` 확장 함수 / 불변 컬렉션 |
| 가장 안전한 설계 | 모든 필드를 **불변(val)** 으로, **불변 컬렉션** 사용 |
| Android 활용 | MVI State 변경, 알림 템플릿, 설정 복제 |
| 주의사항 | `val MutableList`는 내부가 가변 — 불변 `List` 사용 권장 |

- 프로토타입 패턴은 Kotlin에서 `data class copy()`로 가장 자연스럽게 표현됩니다.
- **불변 data class + 불변 컬렉션** 조합이면 `copy()`만으로 완전히 안전합니다.
- 가변 필드가 반드시 필요하다면 `deepCopy()` 확장 함수를 명시적으로 정의하세요.

---

## 참고

- [Kotlin data class 공식 문서](https://kotlinlang.org/docs/data-classes.html)
- [빌더 패턴 포스팅 보기](/posts/2026-05-01-builder)
- [팩토리 패턴 포스팅 보기](/posts/2026-05-02-factory)
- [싱글턴 패턴 포스팅 보기](/posts/2026-05-04-singleton)
