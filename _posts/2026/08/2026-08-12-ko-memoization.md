---
title: (Kotlin/코틀린) 메모이제이션(Memoization) 패턴
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin에서 메모이제이션을 구현하는 방법과 lazy, Map 캐시, 재귀 함수 최적화, 커스텀 memoize 확장 함수 패턴을 정리합니다.
---

---

## 1. 메모이제이션이란?

**이전에 계산한 결과를 저장(캐시)해 같은 입력에 대해 재계산을 피하는** 최적화 기법입니다.

```kotlin
// 메모이제이션 없음: 같은 값을 반복 계산
fun fibonacci(n: Int): Long {
    if (n <= 1) return n.toLong()
    return fibonacci(n - 1) + fibonacci(n - 2)
}
// fibonacci(40) → 약 1억 번 이상의 재귀 호출

// 메모이제이션 적용: 계산한 값을 저장
val cache = mutableMapOf<Int, Long>()
fun fibMemo(n: Int): Long {
    cache[n]?.let { return it }
    if (n <= 1) return n.toLong()
    return (fibMemo(n - 1) + fibMemo(n - 2)).also { cache[n] = it }
}
// fibonacci(40) → 40번의 고유 계산만 수행
```

---

## 2. lazy — 프로퍼티 레벨 메모이제이션

처음 접근할 때만 계산하고 이후 캐시된 값을 반환합니다.

```kotlin
class DataProcessor {
    // 처음 접근 시 한 번만 계산, 이후 캐시
    val processedData: List<Int> by lazy {
        println("처리 중...")
        (1..1000000).filter { it % 2 == 0 }.map { it * 2 }
    }

    val summary: String by lazy {
        "총 ${processedData.size}개, 합계: ${processedData.sum()}"
    }
}

val processor = DataProcessor()
println(processor.summary)  // "처리 중..." 출력 후 결과
println(processor.summary)  // 캐시 반환 (재계산 없음)
```

---

## 3. Map을 이용한 함수 캐시

```kotlin
// 무거운 연산 캐시
class PrimeChecker {
    private val cache = mutableMapOf<Int, Boolean>()

    fun isPrime(n: Int): Boolean {
        return cache.getOrPut(n) {
            // 캐시 미스 시에만 실행
            if (n < 2) false
            else (2..Math.sqrt(n.toDouble()).toInt()).none { n % it == 0 }
        }
    }
}

val checker = PrimeChecker()
println(checker.isPrime(97))   // 계산 후 캐시
println(checker.isPrime(97))   // 캐시 반환
println(checker.isPrime(100))  // 계산 후 캐시
```

---

## 4. 커스텀 memoize 확장 함수

함수 자체를 메모이즈하는 범용 확장 함수를 만들 수 있습니다.

```kotlin
// 1인자 함수 메모이즈
fun <A, R> ((A) -> R).memoize(): (A) -> R {
    val cache = mutableMapOf<A, R>()
    return { a -> cache.getOrPut(a) { this(a) } }
}

// 2인자 함수 메모이즈
fun <A, B, R> ((A, B) -> R).memoize(): (A, B) -> R {
    val cache = mutableMapOf<Pair<A, B>, R>()
    return { a, b -> cache.getOrPut(a to b) { this(a, b) } }
}
```

```kotlin
// 사용
val expensiveComputation = { n: Int ->
    println("계산 중: $n")
    n * n + n
}

val memoized = expensiveComputation.memoize()

println(memoized(5))   // 계산 중: 5 → 30
println(memoized(5))   // 캐시: 30 (재계산 없음)
println(memoized(10))  // 계산 중: 10 → 110
println(memoized(5))   // 캐시: 30
```

---

## 5. 재귀 함수 메모이제이션

```kotlin
// 메모이즈된 피보나치
val fibCache = mutableMapOf<Long, Long>()

fun fib(n: Long): Long = fibCache.getOrPut(n) {
    if (n <= 1) n else fib(n - 1) + fib(n - 2)
}

println(fib(100))  // 354224848179261915075 (빠르게 계산)

// Y-Combinator 스타일의 재귀 메모이즈
fun <T, R> memoizeRecursive(f: (recurse: (T) -> R, arg: T) -> R): (T) -> R {
    val cache = mutableMapOf<T, R>()
    lateinit var memoized: (T) -> R
    memoized = { arg -> cache.getOrPut(arg) { f(memoized, arg) } }
    return memoized
}

val fibMemo = memoizeRecursive<Long, Long> { fib, n ->
    if (n <= 1) n else fib(n - 1) + fib(n - 2)
}
println(fibMemo(50))  // 12586269025
```

---

## 6. 실전 패턴 — API 응답 캐시

```kotlin
class ProductRepository(private val api: ProductApi) {
    private val cache = mutableMapOf<Int, Product>()

    suspend fun getProduct(id: Int): Product {
        return cache.getOrPut(id) {
            api.fetchProduct(id)  // 캐시 미스 시 API 호출
        }
    }
}

// TTL 기반 캐시 (만료 시간 포함)
class TimedCache<K, V>(private val ttlMs: Long) {
    private data class Entry<V>(val value: V, val expiresAt: Long)
    private val store = mutableMapOf<K, Entry<V>>()

    fun get(key: K): V? {
        val entry = store[key] ?: return null
        return if (System.currentTimeMillis() < entry.expiresAt) entry.value
        else { store.remove(key); null }
    }

    fun put(key: K, value: V) {
        store[key] = Entry(value, System.currentTimeMillis() + ttlMs)
    }

    fun getOrPut(key: K, compute: () -> V): V {
        return get(key) ?: compute().also { put(key, it) }
    }
}

val userCache = TimedCache<Int, User>(ttlMs = 60_000)  // 1분 TTL
```

---

## 7. 동시성 안전한 캐시

```kotlin
import java.util.concurrent.ConcurrentHashMap

class ThreadSafeMemoizer<A, R>(private val compute: (A) -> R) {
    private val cache = ConcurrentHashMap<A, R>()

    fun invoke(arg: A): R = cache.computeIfAbsent(arg, compute)
}

// 멀티스레드 환경에서 안전
val safeFib = ThreadSafeMemoizer<Int, Long> { n ->
    if (n <= 1) n.toLong() else safeFib.invoke(n - 1) + safeFib.invoke(n - 2)
}
```

---

## 8. 언제 쓰고 언제 피할까

```
메모이제이션 적합:
✓ 동일 입력 → 항상 동일 출력 (순수 함수)
✓ 계산 비용이 높음 (소수 판별, 피보나치 등)
✓ 같은 값을 반복 요청하는 패턴
✓ 재귀 호출에서 중복 부분 문제

메모이제이션 부적합:
✗ 부작용이 있는 함수 (I/O, 랜덤, 시간 의존)
✗ 결과가 자주 바뀌는 경우
✗ 입력 범위가 매우 커서 캐시 메모리 부담이 큰 경우
✗ 한 번만 호출되는 함수
```

---

## 9. 정리

- **`lazy { }`**: 프로퍼티의 1회 초기화 캐시 — 가장 간단한 메모이제이션
- **`Map.getOrPut`**: 함수 결과를 Map에 캐시하는 기본 패턴
- **`memoize()` 확장 함수**: 함수 자체를 래핑해 재사용 가능한 캐시 함수 생성
- 재귀 함수에 적용하면 지수 시간 → 선형 시간으로 단축
- 멀티스레드 환경에서는 `ConcurrentHashMap.computeIfAbsent` 사용
- 순수 함수에만 적용 (부작용 있으면 캐시 오염)
