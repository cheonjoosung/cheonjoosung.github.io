---
title: (CS/컴퓨터공학) 배열 vs 연결리스트 완전 비교
tags: [CS]
style: fill
color: dark
description: (CS/컴퓨터공학) 배열(Array) vs 연결리스트(Linked List) 완전 비교 — 성능, 메모리, 캐시 지역성, 언제 무엇을 쓸까
---

## ✨ 개요

배열과 연결리스트는 컬렉션 설계의 출발점입니다.  
둘의 **시간/공간 특성**과 **CPU 캐시 관점**을 이해하면, 실제 서비스에서 어떤 구조를 선택해야 할지 명확해집니다.

---

## 1. 한 장 요약 

| 항목 | 배열 (동적 배열: ArrayList 등) | 연결리스트 (Singly/Doubly) |
|---|---|---|
| 인덱스 접근 `a[i]` | **O(1)** (진짜 상수) | **O(n)** (앞/뒤에서 선형 탐색) |
| 맨 끝 추가 | **평균 O(1)** (가끔 리사이즈 O(n)) | **O(1)** (테일 포인터가 있으면) |
| 중간 삽입/삭제 | **O(n)** (이동/시프트 필요) | **O(1)** (해당 노드 참조 이미 있으면) / 찾기는 O(n) |
| 메모리 오버헤드 | 낮음 (연속 영역) | 높음 (노드 포인터 1~2개 + 객체 헤더) |
| 캐시 지역성 | **아주 좋음** (연속 배치) | **나쁨** (포인터 점프, 캐시 미스↑) |
| 랜덤 접근 | 최적 | 부적합 |
| 안정적 반복(iteration) | 빠름 (순차 스캔) | 느림 (포인터 따라가며 캐시 미스) |
| 대표 구현 | Java `ArrayList`, Kotlin `MutableList`(Array 기반) | Java `LinkedList` (Doubly) |

> **핵심**: “**랜덤 접근/순차 스캔/메모리 효율**”이면 배열, “**중간 삽입/삭제가 매우 빈번 + 노드 참조를 이미 갖고 있음**”이면 연결리스트.

---

## 2 구조(Structure)

### 2.1 배열

```text
index: 0 1 2 3 4 5
memory: [A] [B] [C] [D] [E] [F]
↑연속 메모리 블록 → CPU 캐시에 잘 올라감
```

### 2.2 단일 연결리스트(Singly)

```text
[A]→[B]→[C]→[D]→[E]
next next next ...
(메모리 여기저기 흩어짐 → 포인터 따라가며 캐시 미스 가능)
```

### 2.3 Signature

```text
signature = Sign( base64url(header) + "." + base64url(payload), key, alg )
```
- JWS(서명) 규격을 따릅니다. (암호화가 필요하면 JWE)

---

## 3 동적 배열의 리사이즈와 ‘평균 O(1)’

동적 배열(ArrayList 등)은 용량이 꽉 차면 **더 큰 배열로 복사**합니다(보통 x1.5~x2).  
개별 `add`는 가끔 O(n) 복사를 겪지만, **아몰타이즈드(평균) O(1)** 로 간주합니다.

```kotlin
val list = ArrayList<Int>()
repeat(1_000_000) { list.add(it) } // 평균 O(1) 삽입
```

---

## 4. 연결리스트가 진짜 빛나는 순간

노드 참조를 이미 갖고 있는 상태에서 그 자리에서 삽입/삭제해야 할 때
(예: LRU 캐시의 중간 노드 이동, 스케줄러의 ready 큐에서 특정 노드 O(1) 제거)

양방향 리스트는 앞/뒤 이동 및 삭제가 편하지만, 포인터가 늘어 오버헤드↑

```kotlin
// 단일 연결리스트 노드 기준: 현재 노드 뒤에 삽입 O(1)
class Node<T>(var value: T, var next: Node<T>? = null)
fun <T> insertAfter(cur: Node<T>, newNode: Node<T>) {
    newNode.next = cur.next
    cur.next = newNode
}
```

---

## 5. 시간 복잡도 & 상수 비용 감각

| 연산             | 배열                 | 단일 연결리스트                |
| -------------- | ------------------ | ----------------------- |
| `get(i)`       | **O(1)**           | **O(n)**                |
| `push_back`    | **Amortized O(1)** | **O(1)**(tail 보유 시)     |
| `insert(i, x)` | **O(n)**           | **O(n)**(찾기) + O(1)(삽입) |
| `remove(i)`    | **O(n)**           | **O(n)**(찾기) + O(1)(삭제) |
| 순차 반복          | **빠름 (캐시 우수)**     | **느림 (포인터 점프)**         |

---

## 6. 메모리/GC 관점

- 배열: 원소가 프리미티브면 진짜로 연속(JVM에서는 int[] 등). 객체 레퍼런스 배열인 경우 레퍼런스만 연속.
- 연결리스트: 노드마다 객체 + 포인터(1~2개) → 힙 조각화/GC 압박.
- 대규모 데이터 처리에서 연결리스트는 메모리 낭비 + 느린 반복으로 선호되지 않음.

---

## 7. 자주 하는 실수/오해

- “중간 삽입/삭제가 있으니 무조건 연결리스트” → ❌ 
  + 대부분의 실무 패턴은 배열로 충분히 빠르고 단순.
- “LinkedList가 큐/덱에 최적” → ◑
  + Java/Kotlin에서 큐/덱은 ArrayDeque(원형 버퍼)가 더 빠르고 메모리 효율적.
- “배열은 늘리기 어렵다” → ◑ 동적 배열은 자동 리사이즈. 예측 가능하면 초기 용량을 넉넉하게 잡으면 끝.

---

## 8. Kotlin/Java 실용 예시

### 8.1 중간 삽입이 있지만 “한 번에 모아서” 처리 (배열이 유리)

```kotlin
val list = ArrayList<Int>(10_000)
list.addAll(0..9999)
// 중간에 1000개 삽입이 필요하다면 → 새 리스트에 병합 후 한 번에 교체 (복사 비용 한번)
val add = (5000..5999).toList()
val merged = ArrayList<Int>(list.size + add.size).apply {
    addAll(list.subList(0, 5000))
    addAll(add)
    addAll(list.subList(5000, list.size))
}
```

### 8.2 LRU 캐시 같은 “노드 참조 기반 이동/삭제” (리스트가 유리)

```kotlin
class DNode<K, V>(var k: K, var v: V) {
    var prev: DNode<K, V>? = null
    var next: DNode<K, V>? = null
}
class DLinkedList<K, V> {
    private val head = DNode<K, V>(k = Any() as K, v = Any() as V)
    private val tail = DNode<K, V>(k = Any() as K, v = Any() as V)
    init { head.next = tail; tail.prev = head }
    fun addFront(n: DNode<K, V>) { n.next = head.next; n.prev = head; head.next!!.prev = n; head.next = n }
    fun remove(n: DNode<K, V>) { n.prev!!.next = n.next; n.next!!.prev = n.prev }
}
```

해시맵(Key→Node)과 양방향 리스트를 결합하면 이동/삭제 O(1) LRU를 만들 수 있습니다.

---

## 9. 선택 가이드 (실무 기준)

- 대부분의 앱/서비스 컬렉션: ArrayList / ArrayDeque (연산/메모리/캐시 모두 우수)
- LRU/스케줄러/플레이리스트 편집기 등 노드 참조 기반 조작: 연결리스트(보통 양방향)
- 초대형 데이터 스트리밍/분석: 배열/버퍼 + 벡터화(연속 메모리)
- 빈번한 앞삽입: 단순 연결리스트 or ArrayDeque의 addFirst (원형 버퍼가 사실 더 빠른 편)

---

## 결론 

- 현대 CPU/VM 환경에서 배열(동적 배열/원형 버퍼) 은 캐시 지역성 + 낮은 오버헤드 덕분에 대부분의 일반적 시나리오에서 더 빠르고 간단합니다.
- 연결리스트는 노드 참조를 쥔 상태에서의 O(1) 삽입/삭제가 필요할 때 빛나며, 그 외에는 비용(메모리/캐시/GC)이 크게 작용합니다.
- 즉, 접근 패턴과 데이터 크기를 먼저 정의하고, 그에 맞춰 구조를 고르는 것이 정답입니다.