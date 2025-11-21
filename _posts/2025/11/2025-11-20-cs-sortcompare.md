---
title: (CS/컴퓨터공학) 정렬 알고리즘 핵심만 보기 — 퀵/머지/힙 정렬 복잡도 비교
tags: [CS]
style: fill
color: dark
description: (CS/컴퓨터공학) 정렬 알고리즘 핵심만 보기 — 퀵/머지/힙 정렬 시간·공간 복잡도 비교
---

## ✨ 개요

정렬 알고리즘 핵심만 보기 — 퀵/머지/힙 정렬 시간·공간 복잡도 비교

상위 3종 정렬 **퀵(Quick)** · **머지(Merge)** · **힙(Heap)** 을 필드에서 자주 묻는 포인트만 압축 정리합니다.

---

## 1. 한 장 요약

| 알고리즘 | 평균 시간 | 최악 시간 | 최선 시간 | 공간 복잡도 | 안정성 | 제자리(In-place) |
|---|---|---|---|---|---|---|
| **퀵정렬** | `O(n log n)` | `O(n^2)` | `O(n log n)` | `O(log n)` (재귀 스택) | ❌ | ✅(파티션 in-place) |
| **머지정렬** | `O(n log n)` | `O(n log n)` | `O(n log n)` | `O(n)` (배열), `O(1)` (연결리스트) | ✅ | ❌(배열) / ✅(연결리스트) |
| **힙정렬** | `O(n log n)` | `O(n log n)` | `O(n log n)` | `O(1)` | ❌ | ✅ |

> 안정성(Stable): 같은 키의 상대 순서가 보존되는가?  
> 제자리(In-place): 추가 배열 없이(상수 공간) 내부에서 정렬하는가?

---

## 2 핵심 특징과 언제 쓰나

### 🔸 퀵정렬 (Quick Sort)
- **아이디어**: 피벗을 기준으로 좌/우 파티션 후 재귀 정렬.
- **장점**: 평균적으로 **가장 빠름**(캐시 친화적, 상수↓).
- **단점**: 최악 `O(n^2)`(나쁜 피벗), **안정성 X**.
- **포인트**: **피벗 선택이 생명** → median-of-three, 랜덤 피벗, **작은 구간은 삽입정렬로 전환**(하이브리드)로 실무 성능↑.
- **언제**: 메모리 여유가 적고 평균 성능이 중요한 **일반 내장 정렬**(많은 언어들이 내부적으로 변형 사용).

### 🔸 머지정렬 (Merge Sort)
- **아이디어**: 반으로 나누고 정렬된 두 배열을 **합병**.
- **장점**: **항상 `O(n log n)` + 안정적**, 외부정렬에 적합.
- **단점**: 배열 기준 **추가 메모리 `O(n)`**.
- **포인트**: 연결리스트는 **포인터 재배치만으로 `O(1)` 공간** 가능.
- **언제**: **안정성**이 필수거나, **대용량/외부 정렬**(파일·스트리밍)에서 선호. Python/Java의 **Timsort**(실전 하이브리드)는 안정적이며 부분 정렬(run) 최적화.

### 🔸 힙정렬 (Heap Sort)
- **아이디어**: 최대힙 구성 후, 루트를 끝으로 보내고 힙 다운힙을 반복.
- **장점**: 최악도 **`O(n log n)` + `O(1)` 공간**.
- **단점**: 캐시 지역성 낮아 상수 비용 큼, **안정성 X**, 실측 속도는 보통 퀵/팀소트보다 느림.
- **언제**: **상수 공간 보장**이 중요하고 최악 시간도 보수적으로 묶어야 할 때.

---

## 3 미니 의사코드 (Kotlin 유사)

```kotlin
fun mergeSort(a: IntArray) {
    val buf = IntArray(a.size)
    fun sort(l: Int, r: Int) {
        if (l >= r) return
        val m = (l + r) ushr 1
        sort(l, m); sort(m + 1, r)
        // merge
        var i = l; var j = m + 1; var k = l
        while (i <= m && j <= r) buf[k++] = if (a[i] <= a[j]) a[i++] else a[j++]
        while (i <= m) buf[k++] = a[i++]
        while (j <= r) buf[k++] = a[j++]
        for (t in l..r) a[t] = buf[t]
    }
    sort(0, a.lastIndex)
}

fun heapSort(a: IntArray) {
    fun down(i: Int, n: Int) {
        var p = i
        while (true) {
            val l = p * 2 + 1; val r = l + 1
            var largest = p
            if (l < n && a[l] > a[largest]) largest = l
            if (r < n && a[r] > a[largest]) largest = r
            if (largest == p) break
            a[p] = a[largest].also { a[largest] = a[p] }; p = largest
        }
    }
    // build max-heap
    for (i in a.size / 2 - 1 downTo 0) down(i, a.size)
    // extract
    for (end in a.lastIndex downTo 1) {
        a[0] = a[end].also { a[end] = a[0] }
        down(0, end)
    }
}
```

---

## 4. 택 가이드 (실무 감각)

- 안정 정렬 필요 → 머지 계열(Timsort 권장)
- 메모리 타이트 + 평균 성능 → 퀵(in-place, 하이브리드)
- 최악 성능 보수적 + 상수 공간 보장 → 힙
- 부분 정렬·이미 거의 정렬 → 삽입/팀소트가 매우 강함
- 탑-K/중앙값 → 전체 정렬 대신 힙/퀵셀렉트(O(n))

---

## 5. 추가 팁

- 퀵정렬 최악 회피: 랜덤 피벗, median-of-three, 재귀 깊이 제한 + 힙정렬로 전환(Introsort).
- 머지 안정성: 멀티키 정렬(예: 날짜→이름)에서 앞선 키의 순서 보존에 유리.
- 힙정렬 사용 시: 캐시 지역성 낮음 → 매우 큰 데이터에서 성능 저하 가능.
- 외부정렬(디스크/스트림): 머지(다중 파일 k-way merge)가 표준.

---

## 6. 결론

- 세 알고리즘 모두 평균/최악/공간이 다른 장단점을 가진다.
- “안정성·메모리·최악시간 중 무엇을 우선?”을 먼저 정하고 고르면 된다.
- 일반 애플리케이션은 팀소트(머지+런 탐지) 또는 인트로소트(퀵+힙 전환) 가 실무 최적해에 가깝다.