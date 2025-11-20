---
title: (CS/컴퓨터공학) 스택·큐·덱(Deque) 개념 및 비교
tags: [CS]
style: fill
color: dark
description: (CS/컴퓨터공학) 택·큐·덱(Deque) 개념 및 비교 — 연산, 복잡도, 언제 무엇을 쓸까
---

## ✨ 개요

스택·큐·덱(Deque) 개념 및 비교 — 연산, 복잡도, 언제 무엇을 쓸까

스택(Stack), 큐(Queue), 덱(Deque)은 **순서가 있는 선형 자료구조**의 기본 형태입니다.  
실무에서는 컬렉션 선택만 잘해도 성능과 코드 단순성이 크게 개선됩니다.

---

## 1. 한 장 요약 

| 구조 | 핵심 개념 | 출입 방향 | 주요 연산 | 시간복잡도(배열/덱 기반) | 대표 구현 |
|---|---|---|---|---|---|
| **스택** | 마지막에 넣은 것을 먼저 꺼냄 (**LIFO**) | **한쪽** | `push`, `pop`, `peek`, `isEmpty` | 평균 O(1) | `ArrayDeque`(권장), `Stack`(구식) |
| **큐** | 먼저 넣은 것을 먼저 꺼냄 (**FIFO**) | **양쪽(뒤로 넣고 앞에서 뺌)** | `offer`/`add`, `poll`/`remove`, `peek` | 평균 O(1) | `ArrayDeque`, `LinkedList`(권장 X) |
| **덱** | 양 끝에서 넣고 빼기(**Double-Ended Queue**) | **양쪽** | `addFirst/Last`, `pollFirst/Last`, `peekFirst/Last` | 평균 O(1) | `ArrayDeque` |

> JVM/코틀린에서는 **`ArrayDeque`** 가 스택/큐/덱 용도로 **가장 빠르고 메모리 효율**이 좋습니다.

---

## 2 개념 그림(ASCII)

### 스택 (LIFO)

```text
push → [ a | b | c(top) ]
pop ← ^ (c가 먼저 나옴)
```

### 2.2 큐 (Queue)

```text
offer → [ a | b | c ] → poll
^front ^back
```

### 2.3 덱 (Double-Ended)

```text
addFirst → [ x | a | b | c ] ← addLast
pollFirst ← → pollLast
```

---

## 3 언제 무엇을 쓰나?

- **스택**: 되돌리기(undo), 괄호/태그 검증, DFS, 재귀 제거, 파서
- **큐**: 작업 대기열, BFS, 생산자–소비자(순서 보장), 비동기 처리
- **덱**: 양끝 처리(슬라이딩 윈도우, 모노토닉 큐/스택), 편집기 버퍼, 양방향 탐색

---

## 4. 코틀린 사용 예제 (JVM 표준 라이브러리 기반)

### 4.1 스택: 괄호 유효성 검사

```kotlin
fun isValidBrackets(s: String): Boolean {
    val stack = ArrayDeque<Char>() // 스택으로 사용
    val pair = mapOf(')' to '(', ']' to '[', '}' to '{')
    for (ch in s) {
        when (ch) {
            '(', '[', '{' -> stack.addLast(ch)              // push
            ')', ']', '}' -> if (stack.isEmpty() || stack.removeLast() != pair[ch]) return false // pop
        }
    }
    return stack.isEmpty()
}
```

### 4.2 큐: BFS(최단거리/레벨 순회)

```kotlin
data class Node(val id: Int, val next: List<Int>)

fun bfs(start: Int, graph: Map<Int, Node>): List<Int> {
    val q = ArrayDeque<Int>() // 큐로 사용
    val visited = BooleanArray(graph.size)
    val order = mutableListOf<Int>()
    q.addLast(start); visited[start] = true

    while (q.isNotEmpty()) {
        val cur = q.removeFirst() // poll
        order += cur
        for (n in graph[cur]!!.next) {
            if (!visited[n]) { visited[n] = true; q.addLast(n) } // offer
        }
    }
    return order
}
```

### 4.3 덱: 슬라이딩 윈도우 최대값(O(n), 모노토닉 큐)

```kotlin
fun maxSlidingWindow(nums: IntArray, k: Int): IntArray {
    val dq = ArrayDeque<Int>() // 인덱스를 저장, 값은 단조 내림 유지
    val res = IntArray(nums.size - k + 1)
    for (i in nums.indices) {
        // 윈도 범위 밖 제거
        if (dq.isNotEmpty() && dq.first() == i - k) dq.removeFirst()
        // 뒤에서부터 현재 값보다 작은 요소 제거 (단조성 유지)
        while (dq.isNotEmpty() && nums[dq.last()] <= nums[i]) dq.removeLast()
        dq.addLast(i)
        if (i >= k - 1) res[i - k + 1] = nums[dq.first()]
    }
    return res
}

```

---

## 5. 복잡도와 상수 시간의 함정

- 이상적 연산은 모두 평균 O(1) 이지만, 구현에 따라 상수 비용이 큽니다.
- 배열 기반 원형 버퍼(ArrayDeque) 는 연속 메모리 덕에 캐시 지역성이 좋아 빠릅니다.
- LinkedList 기반 큐/덱은 포인터 추적·할당 비용 때문에 대부분 느립니다.

---

## 6. 스택/큐/덱 선택 가이드 (실무 기준)

- 스택이 필요하다 → ArrayDeque<T> 사용, addLast/removeLast를 push/pop처럼 사용
- 큐가 필요하다 → ArrayDeque<T>, addLast/removeFirst
- 양끝 입출력 → ArrayDeque<T> 의 addFirst/Last, removeFirst/Last
- 동시성 필요 시 → ConcurrentLinkedQueue, LinkedBlockingDeque 등 JUC 컬렉션(정말 필요한 경우에만)

---

## 7. 흔한 실수/오해

- Stack 클래스 사용: 레거시 동기화 오버헤드, API 불편 → ArrayDeque 권장
- LinkedList가 큐에 최적: 대부분의 일반 시나리오에서 ArrayDeque가 더 빠르고 메모리 효율적
- 덱을 리스트처럼 인덱스로 접근: 덱은 양끝 입출력 특화, 중간 접근/삽입은 다른 구조가 적합
- 큐에서 add/remove 사용 시 예외 발생 가능 → 안전하게 offer/poll/peek 사용 습관화

---

## 8. 요약 체크리스트

- LIFO → 스택 (ArrayDeque + addLast/removeLast)
- FIFO → 큐 (ArrayDeque + addLast/removeFirst)
- 양끝 필요 → 덱 (ArrayDeque + add/peek/poll First/Last)
- 성능/메모리 → ArrayDeque 우선, 링크드 구조는 특수 케이스
- 동시성 → JUC 큐/덱 고려(락·백프레셔 등 별도 설계 포함)

---

##  결론 

- 스택·큐·덱은 단순하지만 알맞은 선택만으로도 코드가 간결해지고 성능이 선형적으로 좋아집니다.
- JVM/코틀린 환경에선 ArrayDeque를 기본값으로 기억하세요.
- 특수한 동시성/락프리 요구가 없다면, 이것 하나로 스택/큐/덱 대부분의 요구를 깔끔하게 해결할 수 있습니다.