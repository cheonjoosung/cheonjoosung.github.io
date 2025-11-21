---
title: (CS/컴퓨터공학) 해시테이블 동작 원리와 충돌 해결 - 체이닝 vs 오픈 어드레싱
tags: [CS]
style: fill
color: dark
description: (CS/컴퓨터공학) 해시테이블 동작 원리와 충돌 해결 - 체이닝 vs 오픈 어드레싱 완벽 비교
---

## ✨ 개요

해시테이블(Hash Table)은 키를 해시함수로 **버킷 인덱스**에 매핑해 평균 `O(1)`로 조회/삽입/삭제를 수행하는 자료구조입니다.  
핵심은 **충돌(collision) 관리**와 **부하율(load factor)**, 그리고 **재해시(rehash)** 전략입니다.

---

## 1. 한 장 요약

1. **해시**: `h = H(key)`
2. **인덱싱**: `i = h mod capacity`
3. **충돌**: 다른 키가 같은 인덱스에 매핑되면 **해결 전략** 적용
4. **부하율**: `α = size / capacity`가 임계치(예: 0.75) 넘으면 **리사이즈 + 재해시**

> 좋은 해시함수(균등분포) + 적정 부하율 유지가 성능의 90%를 좌우합니다.

---

## 2 충돌 해결 전략

### 2.1 체이닝(Chaining)

**아이디어**: 각 버킷이 **리스트/트리**를 가짐. 같은 인덱스의 항목을 **연결**해 저장.

```text
index: 0 1 2 3
bucket: [ ] -> [K3] [K1]->[K8] [ ]
| |
val val
```

- **삽입**: 버킷 헤드에 노드 추가(보통 O(1))
- **조회/삭제**: 해당 버킷의 리스트(또는 트리)를 선형 탐색
- **시간 복잡도**:
    - 평균 `O(1)` (균등 해시 + α 적절)
    - 최악 `O(n)` (모든 키가 한 버킷)
- **변형**: Java 8+ `HashMap`은 **버킷 길이 임계치** 초과 시 **트리화(레드-블랙 트리)** → 최악 `O(log n)`

**장점**
- 단순/구현 쉬움, 삭제가 쉽다
- 테이블이 꽉 차도 삽입 가능(연결리스트가 늘어남)

**단점**
- 포인터 오버헤드, 캐시 지역성 낮음
- 매우 큰 α에서 성능 급락


### 2.2 오픈 어드레싱(Open Addressing)

**아이디어**: 버킷 하나에 **하나의 원소만** 저장. 충돌 시 **빈 슬롯**을 **탐사(probing)** 로 찾아 **테이블 내부**에 저장.

```text
table: [K1] [ ] [K8] [K3] [ ] [ ]
probe: i i+1 i+2 ...
```

- **삽입**: 충돌하면 규칙에 따라 다음 인덱스를 **계속 탐사**
- **조회/삭제**: 같은 규칙으로 탐사하며 키를 찾음
- **탐사 방식**
    - **선형 탐사**: `i, i+1, i+2, ...` (간단, 1차 클러스터링↑)
    - **제곱 탐사**: `i, i+1^2, i+2^2, ...` (1차↓, 2차 클러스터링)
    - **이중 해싱**: `i, i + j*H2(key)` (클러스터링↓, 구현 복잡)
- **삭제 이슈**: 단순 삭제는 **탐사 사슬**을 끊음 → **삭제 마커(tombstone)** 필요
- **시간 복잡도**:
    - 평균 `O(1)` (α가 낮을 때)
    - 최악 `O(n)` (α 높고 클러스터링 심함)

**장점**
- 포인터 없음 → **메모리 효율↑**, **캐시 지역성↑**
- 작은 키-값 저장에서 매우 빠를 수 있음

**단점**
- **삭제 복잡**(tombstone 관리)
- **α가 낮아야** 성능 유지(보통 0.5~0.7 권장)

---

## 3 체이닝 vs 오픈 어드레싱 비교표

| 항목 | 체이닝(Chaining) | 오픈 어드레싱(Open Addressing) |
|---|---|---|
| 저장 방식 | 버킷마다 리스트/트리 | 테이블 내부 빈 슬롯 탐사 |
| 메모리 | 포인터/노드 오버헤드↑ | 테이블만 사용(효율↑) |
| 캐시 지역성 | 낮음 | 높음(연속 메모리) |
| 삭제 | 쉬움 | tombstone 필요, 재해시로 청소 필요 |
| 부하율 민감도 | 낮음(α ↑ 가능) | 매우 높음(α 낮게 유지) |
| 최악 보장 | 리스트면 O(n), 트리화 시 O(log n) | O(n) 가능 |
| 구현 복잡도 | 낮음(트리화 시 중간↑) | 중간~높음(이중 해싱/삭제 관리) |
| 실전 사용 | Java/Kotlin `HashMap` 등 | 일부 언어 런타임/라이브러리, 고성능 집약 구조 |

---

## 4. 부하율(Load Factor)과 리사이즈/재해시(Rehash)

- **부하율 α = size / capacity**
    - 체이닝: `α ≈ 0.75~1.0+` 도 운용 가능(버킷 길이 주의)
    - 오픈 어드레싱: `α`가 0.7 넘으면 **탐사 길이 급증** → 삽입/조회 지연
- **리사이즈**: 용량 2배(또는 소수 계열)로 늘리고 전체 항목 **재해시**
- 오픈 어드레싱은 리사이즈 시 텀브스톤이 사라져 **성능 회복** 효과

---

## 5. 해시함수와 인덱싱 팁

- **균등 분포**가 가장 중요(클러스터링 방지).
- 정수 키는 **비트 혼합(mix)** 후 `& (capacity-1)` (capacity를 2의 거듭제곱으로)
- 문자열은 **Murmur/XXHash** 계열(비암호학 고속) → 보안 요구 시 **SipHash**(DoS 완화)

---

## 6. kotlin/Java 관점 실무 팁

- `HashMap/HashSet` 사용 시 **`equals`/`hashCode` 계약** 준수
  + `a == b` 이면 `a.hashCode() == b.hashCode()` 반드시 성립
  + 뮤터블 키는 **변경 금지**(버킷 불일치)
- 자주 조회/삽입: **초기 용량**과 **loadFactor(예: 0.75)** 를 업무 데이터에 맞춰 튜닝
- 동시성 필요: `ConcurrentHashMap` (세그먼트/노드 수준 동시성 관리)

---

## 7. 간단 예제: 체이닝 기반(싱글 버킷 리스트)

```kotlin
class ChainedHashMap<K, V>(initCap: Int = 16) {
    data class Node<K, V>(val k: K, var v: V, var next: Node<K, V>? = null)
    private var cap = Integer.highestOneBit(maxOf(2, initCap - 1) shl 1)
    private var size = 0
    private var table = arrayOfNulls<Node<K, V>>(cap)
    private val loadFactor = 0.75f

    private fun index(key: K) = (key.hashCode() * -0x7f4a7c15 /*mix*/).ushr(1) and (cap - 1)

    fun put(key: K, value: V): V? {
        if (size + 1 > cap * loadFactor) resize()
        val i = index(key)
        var n = table[i]
        while (n != null) {
            if (n.k == key) { val old = n.v; n.v = value; return old }
            n = n.next
        }
        table[i] = Node(key, value, table[i]); size++
        return null
    }

    fun get(key: K): V? {
        var n = table[index(key)]
        while (n != null) { if (n.k == key) return n.v; n = n.next }
        return null
    }

    private fun resize() {
        cap = cap shl 1
        val newTab = arrayOfNulls<Node<K, V>>(cap)
        for (head in table) {
            var n = head
            while (n != null) {
                val next = n.next
                val i = index(n.k)
                n.next = newTab[i]; newTab[i] = n
                n = next
            }
        }
        table = newTab
    }
}
```

---

## 8. 간단 예제: 오픈 어드레싱(선형 탐사 + tombstone)

```kotlin
class LinearProbingMap<K, V>(initCap: Int = 16) {
private data class Entry<K, V>(var k: K?, var v: V?, var tomb: Boolean = false)
private var cap = Integer.highestOneBit(maxOf(2, initCap - 1) shl 1)
private var size = 0
private var table = Array(cap) { Entry<K, V>(null, null, false) }
private val loadFactor = 0.6f

    private fun idx(key: K) = (key.hashCode() * 0x9e3779b9).ushr(1) and (cap - 1)

    fun put(key: K, value: V): V? {
        if ((size + 1).toFloat() / cap > loadFactor) resize()
        var i = idx(key)
        var firstTomb = -1
        while (true) {
            val e = table[i]
            when {
                e.k == null && !e.tomb -> {
                    val slot = if (firstTomb >= 0) firstTomb else i
                    table[slot].k = key; table[slot].v = value; table[slot].tomb = false; size++
                    return null
                }
                e.k == null && e.tomb && firstTomb < 0 -> firstTomb = i
                e.k == key -> { val old = e.v; e.v = value; return old }
            }
            i = (i + 1) and (cap - 1)
        }
    }

    fun get(key: K): V? {
        var i = idx(key)
        while (true) {
            val e = table[i]
            if (e.k == null && !e.tomb) return null
            if (e.k == key) return e.v
            i = (i + 1) and (cap - 1)
        }
    }

    fun remove(key: K): V? {
        var i = idx(key)
        while (true) {
            val e = table[i]
            if (e.k == null && !e.tomb) return null
            if (e.k == key) {
                val old = e.v
                e.k = null; e.v = null; e.tomb = true
                size--; return old
            }
            i = (i + 1) and (cap - 1)
        }
    }

    private fun resize() {
        val old = table
        cap = cap shl 1
        table = Array(cap) { Entry<K, V>(null, null, false) }
        size = 0
        for (e in old) if (e.k != null && !e.tomb) put(e.k as K, e.v as V)
    }
}
```

---

## 결론

- 해시테이블의 O(1) 성능은 충돌을 어떻게 다루는지에 달려 있습니다.
- 체이닝은 단순하고 삭제 친화적, 오픈 어드레싱은 메모리/캐시 효율이 뛰어납니다.
- 워크로드(삽입/삭제/조회 패턴, 메모리 한계, α 유지 가능성)에 맞춰 전략을 선택하면, 해시테이블은 기대한 대로 빠르고 안정적으로 동작합니다.