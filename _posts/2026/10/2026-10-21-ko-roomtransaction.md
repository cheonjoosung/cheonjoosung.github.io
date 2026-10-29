---
title: (Android/Room) Room Transaction — 원자적 DB 작업
tags: [ Android, Room ]
style: fill
color: dark
description: Room에서 @Transaction, withTransaction으로 여러 DB 작업을 원자적으로 처리하는 방법과 롤백, 코루틴 주의점, 성능 이점을 정리합니다.
---

---

## 1. 트랜잭션이란?

여러 DB 작업을 **전부 성공하거나 전부 취소**되도록 묶는 단위입니다 (원자성).

```
계좌 이체: 출금 + 입금

트랜잭션 없음:  출금 성공 → (앱 크래시) → 입금 실패  →  돈이 사라짐
트랜잭션 있음:  출금 성공 → 입금 실패 → 전체 롤백   →  원래 상태 유지
```

---

## 2. DAO의 @Transaction

여러 작업을 하나의 DAO 함수로 묶습니다.

```kotlin
@Dao
abstract class TransferDao {          // 본문이 있는 함수를 쓰려면 abstract class (또는 interface의 default)

    @Query("UPDATE accounts SET balance = balance - :amount WHERE id = :id")
    abstract suspend fun withdraw(id: Long, amount: Long)

    @Query("UPDATE accounts SET balance = balance + :amount WHERE id = :id")
    abstract suspend fun deposit(id: Long, amount: Long)

    @Transaction                       // 이 함수 전체가 하나의 트랜잭션
    open suspend fun transfer(from: Long, to: Long, amount: Long) {
        withdraw(from, amount)
        deposit(to, amount)            // 여기서 예외가 나면 withdraw도 롤백
    }
}
```

interface DAO에서는 본문 있는 함수도 선언할 수 있습니다.

```kotlin
@Dao
interface OrderDao {
    @Insert suspend fun insertOrder(order: Order): Long
    @Insert suspend fun insertItems(items: List<OrderItem>)

    @Transaction
    suspend fun placeOrder(order: Order, items: List<OrderItem>) {
        val orderId = insertOrder(order)
        insertItems(items.map { it.copy(orderId = orderId) })
    }
}
```

---

## 3. 여러 DAO를 묶는 — withTransaction

서로 다른 DAO의 작업을 하나로 묶을 때는 `withTransaction`을 사용합니다.

```kotlin
class OrderRepository @Inject constructor(
    private val db: AppDatabase
) {
    suspend fun checkout(order: Order, items: List<OrderItem>) {
        db.withTransaction {                         // room-ktx 필요
            val id = db.orderDao().insertOrder(order)
            db.orderItemDao().insertAll(items.map { it.copy(orderId = id) })
            db.stockDao().decrease(items)            // 재고 차감
            db.cartDao().clear()                     // 장바구니 비우기
        }
    }
}
```

---

## 4. 롤백 규칙

블록 안에서 **예외가 밖으로 던져지면 롤백**됩니다.

```kotlin
try {
    db.withTransaction {
        db.accountDao().withdraw(1, 1000)
        if (db.accountDao().balance(1) < 0) {
            throw InsufficientBalanceException()     // 예외 → 전체 롤백
        }
        db.accountDao().deposit(2, 1000)
    }
} catch (e: InsufficientBalanceException) {
    // 롤백 완료된 상태: DB는 변경 전
}
```

> 트랜잭션 **안에서** 예외를 `try/catch`로 삼키면 롤백되지 않고 그대로 커밋됩니다. 롤백하려면 예외를 다시 던져야 합니다.

---

## 5. 코루틴 사용 시 주의

```kotlin
db.withTransaction {
    val user = db.userDao().find(id)

    // ❌ 트랜잭션 안에서 네트워크 호출 — DB 연결을 오래 점유해 다른 쿼리가 대기
    val remote = api.fetchProfile(id)
    db.userDao().update(user.copy(name = remote.name))
}

// ✅ 네트워크 호출은 트랜잭션 밖에서 먼저
val remote = api.fetchProfile(id)
db.withTransaction {
    val user = db.userDao().find(id)
    db.userDao().update(user.copy(name = remote.name))
}
```

- 트랜잭션은 **짧게** 유지: DB 작업만 넣고 네트워크, 파일 I/O, 긴 연산은 바깥에서 처리
- 트랜잭션 안에서 다른 디스패처로 `withContext`를 전환하면 트랜잭션이 깨질 수 있으니 피하세요

---

## 6. 성능 이점 — 대량 삽입

트랜잭션 없이 1000건을 하나씩 insert하면 **매번 디스크에 커밋**합니다. 하나로 묶으면 커밋이 한 번뿐이라 훨씬 빠릅니다.

```kotlin
// 느림: insert 호출마다 커밋
items.forEach { dao.insert(it) }

// 빠름: List insert는 내부적으로 하나의 트랜잭션
dao.insertAll(items)

// 여러 종류의 쓰기를 묶을 때
db.withTransaction { items.forEach { dao.insert(it) } }
```

---

## 7. @Transaction이 필요한 곳 — 읽기도 포함

`@Relation`을 쓰는 쿼리는 **여러 쿼리가 실행**되므로 읽기에도 `@Transaction`이 필요합니다.

```kotlin
@Transaction
@Query("SELECT * FROM users")
fun observeUsersWithPosts(): Flow<List<UserWithPosts>>
```

일관된 스냅샷으로 읽지 않으면, 부모를 읽은 뒤 자식을 읽는 사이에 데이터가 바뀌어 **불일치한 결과**가 나올 수 있습니다.

---

## 8. 정리

- 트랜잭션은 여러 작업을 **전부 성공 / 전부 롤백**으로 묶는 원자적 단위
- 단일 DAO: `@Transaction` 함수, 여러 DAO: `db.withTransaction { }`
- 블록 밖으로 **예외가 던져지면 롤백**, 안에서 삼키면 커밋됨
- 트랜잭션은 짧게, 네트워크 호출은 바깥에서
- 여러 쓰기를 묶으면 커밋 횟수가 줄어 성능도 향상
- `@Relation` 조회에도 `@Transaction` 필요
