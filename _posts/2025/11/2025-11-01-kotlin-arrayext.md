---
title: Kotlin/코틀린 List/MutableList/Set/Map 컬렉션 확장함수 모음
tags: [Kotlin]
style: fill
color: dark
description: Kotlin/코틀린 List/MutableList/Set/Map 컬렉션 확장함수 모음 – takeIf 실전 패턴 포함
---

## ✨ 개요

- 대상: `List`, `MutableList`, `Set`, `Map`, `Sequence`
- 포커스: NPE·중복·조건부 추가/수정/이동/슬라이딩 윈도우/그룹핑/집계/불변 컬렉션 편의
- 안드로이드 실무 팁: `DiffUtil`·UI 리스트 갱신과 궁합, `copy + modified` 불변 패턴

---

## 1. 표준 확장 실전 패턴

```kotlin
// 1) takeIf / takeUnless – 조건부 반환
val onlyEven = list.takeIf { it.all { n -> n % 2 == 0 } } ?: emptyList()

// 2) mapNotNull / filterNotNull – 널 제거/매핑
val names = users.mapNotNull { it.nickname?.trim()?.takeIf(String::isNotEmpty) }

// 3) ifEmpty / ifBlank (List는 ifEmpty, String은 ifBlank)
val safe = list.ifEmpty { listOf(defaultItem) }

// 4) associateBy / groupBy – 키 충돌 시 최근값 유지
val byId = users.associateBy { it.id }           // 마지막 요소가 이김
val byTeam = users.groupBy { it.teamId }         // Map<TeamId, List<User>>

// 5) distinctBy – 특정 키 기준 중복 제거(첫 번째 보존)
val unique = users.distinctBy { it.id }

// 6) chunked / windowed – 청크/슬라이딩 윈도우
val pages = items.chunked(20)                     // 20개씩 페이지
val windows = items.windowed(size = 5, step = 1)  // 5개 슬라이딩
```

---

## 2 실전 유틸 확장 40선 - `CollectionsExt.kt`

```kotlin
/* ------------------- 공통 안전/편의 ------------------- */

// 인덱스 안전 접근
fun <T> List<T>.getOrNull(index: Int): T? = if (index in indices) this[index] else null

// 비어있으면 null
fun <T> List<T>.nullIfEmpty(): List<T>? = if (isEmpty()) null else this

// 사이즈가 특정 값일 때만 (takeIf 패턴)
inline fun <T> List<T>.takeIfSize(expected: Int): List<T>? =
    takeIf { size == expected }

// 조건에 맞는 첫 요소 없으면 null
inline fun <T> Iterable<T>.firstOrNullIf(pred: (T) -> Boolean): T? =
    firstOrNull(pred)

/* ------------------- MutableList 조작 ------------------- */

// 조건부 추가
fun <T> MutableList<T>.addIf(condition: Boolean, value: T): Boolean =
    if (condition) add(value) else false

// null 아닐 때만 추가
fun <T> MutableList<T>.addIfNotNull(value: T?): Boolean =
    value?.let { add(it) } ?: false

// 조건부 교체 (첫 매칭 1개)
inline fun <T> MutableList<T>.replaceFirstIf(
    predicate: (T) -> Boolean,
    newItem: T
): Boolean {
    val idx = indexOfFirst(predicate)
    return if (idx >= 0) { this[idx] = newItem; true } else false
}

// 요소 이동 (drag & drop)
fun <T> MutableList<T>.move(from: Int, to: Int): Boolean {
    if (from !in indices || to !in 0..size) return false
    val item = removeAt(from)
    add(if (to > size) size else to, item)
    return true
}

// 스왑
fun <T> MutableList<T>.swap(i: Int, j: Int): Boolean {
    if (i !in indices || j !in indices) return false
    if (i == j) return true
    val tmp = this[i]; this[i] = this[j]; this[j] = tmp
    return true
}

// upsert (키 비교로 있으면 교체, 없으면 추가)
inline fun <T> MutableList<T>.upsert(
    newItem: T,
    crossinline sameKey: (old: T, new: T) -> Boolean
): Boolean {
    val idx = indexOfFirst { sameKey(it, newItem) }
    return if (idx >= 0) { this[idx] = newItem; true } else add(newItem)
}

// 조건 삭제
inline fun <T> MutableList<T>.removeIf2(predicate: (T) -> Boolean): Boolean =
    removeAll(predicate)

/* ------------------- Set/Map 편의 ------------------- */

// Set 대칭차집합 (토글)
operator fun <T> MutableSet<T>.xorAssign(value: T) {
    if (!add(value)) remove(value) // 있으면 제거, 없으면 추가
}

// Map putIfAbsent + 반환
fun <K, V> MutableMap<K, V>.getOrPutAndGet(key: K, default: () -> V): V =
    getOrPut(key, default)

// Key 기준 최신값 유지(덮어쓰기) – 리스트 → 맵
inline fun <T, K> Iterable<T>.toMapKeepLast(keySelector: (T) -> K): Map<K, T> =
    associateBy(keySelector)

/* ------------------- 불변(List) 편의 (copy + modified) ------------------- */

// 불변 리스트에 요소 추가해 새 리스트 반환
fun <T> List<T>.plusNew(item: T): List<T> = toMutableList().apply { add(item) }

// 요소 여러 개
fun <T> List<T>.plusAllNew(items: Iterable<T>): List<T> =
    toMutableList().apply { addAll(items) }

// 특정 인덱스 교체 (불변)
fun <T> List<T>.replaced(index: Int, newItem: T): List<T> =
    toMutableList().apply { if (index in indices) this[index] = newItem }

// 조건부 교체 (불변)
inline fun <T> List<T>.replacedIf(
    predicate: (T) -> Boolean,
    transform: (T) -> T
): List<T> = map { if (predicate(it)) transform(it) else it }

// 삭제 (불변)
fun <T> List<T>.removedAt(index: Int): List<T> =
    toMutableList().apply { if (index in indices) removeAt(index) }

// 삽입 (불변)
fun <T> List<T>.inserted(index: Int, item: T): List<T> =
    toMutableList().apply { add(index.coerceIn(0, size), item) }

/* ------------------- 중복/정렬/집계 ------------------- */

// distinctBy인데 "마지막"을 보존하고 싶을 때
inline fun <T, K> Iterable<T>.distinctByKeepLast(selector: (T) -> K): List<T> =
    associateBy(selector).values.toList()

// 정렬 안정성 보장(이미 정렬된 경우 no-op)
inline fun <T : Comparable<T>> List<T>.sortedIfNeeded(): List<T> =
    if (zipWithNext().all { (a, b) -> a <= b }) this else sorted()

// min~max 한 번에
inline fun <T, R : Comparable<R>> Iterable<T>.minMaxByOrNull(selector: (T) -> R): Pair<T, T>? {
    val itr = iterator(); if (!itr.hasNext()) return null
    var minE = itr.next(); var maxE = minE
    var minV = selector(minE); var maxV = minV
    for (e in itr) {
        val v = selector(e)
        if (v < minV) { minV = v; minE = e }
        if (v > maxV) { maxV = v; maxE = e }
    }
    return minE to maxE
}

// Long 합/평균
inline fun <T> Iterable<T>.sumOfLong(selector: (T) -> Long): Long {
    var s = 0L; for (e in this) s += selector(e); return s
}
inline fun <T> Iterable<T>.averageOf(selector: (T) -> Double): Double {
    var s = 0.0; var c = 0; for (e in this) { s += selector(e); c++ }; return if (c == 0) 0.0 else s / c
}

/* ------------------- 슬라이딩/파티션 ------------------- */

// 고정 크기 슬라이딩 윈도우(경계 포함/미포함 선택)
fun <T> List<T>.sliding(size: Int, step: Int = 1, partial: Boolean = false): List<List<T>> =
    buildList {
        var i = 0
        while (i < this@sliding.size) {
            val end = (i + size).coerceAtMost(this@sliding.size)
            if (partial || end - i == size) add(this@sliding.subList(i, end))
            i += step
        }
    }

// 조건 파티션 → (true리스트, false리스트)
inline fun <T> Iterable<T>.partitionTo(predicate: (T) -> Boolean): Pair<List<T>, List<T>> =
    partition(predicate)

/* ------------------- 시퀀스/성능 ------------------- */

// 리스트를 시퀀스로 → 체이닝 비용 절감(대량 처리)
inline fun <T, R> List<T>.seqMapNotNull(crossinline block: (T) -> R?): List<R> =
    asSequence().mapNotNull(block).toList()

/* ------------------- UI 목록 갱신(DiffUtil 친화) ------------------- */

// 키가 같은 항목의 내용만 바뀌었는지 빠르게 판단
inline fun <T, K> List<T>.contentEqualsBy(
    other: List<T>,
    crossinline key: (T) -> K,
    crossinline equals: (T, T) -> Boolean = { a, b -> a == b }
): Boolean {
    if (size != other.size) return false
    for (i in indices) {
        val a = this[i]; val b = other[i]
        if (key(a) != key(b) || !equals(a, b)) return false
    }
    return true
}
```

---

## 3 사용 예

```kotlin
// 조건부 추가/교체
val items = mutableListOf(1, 3, 5)
items.addIf(condition = true, value = 7)
items.replaceFirstIf({ it == 3 }, newItem = 4) // [1,4,5,7]

// 불변 편집
val newList = items.replaced(0, 10).inserted(1, 20).removedAt(3)

// upsert (id 동일 시 교체)
data class User(val id: Long, val name: String)
val users = mutableListOf(User(1,"A"), User(2,"B"))
users.upsert(User(2,"Bee")) { old, new -> old.id == new.id } // (2,"Bee")로 갱신

// 중복 제거(마지막 보존)
val latestById = listOf(User(1,"A"), User(1,"A2"), User(2,"B"))
    .distinctByKeepLast { it.id } // (1,"A2"),(2,"B")

// 슬라이딩
val windows = (1..7).toList().sliding(size = 3, step = 2, partial = true)
// [[1,2,3],[3,4,5],[5,6,7],[7]]

// min~max 한번에
val (minU, maxU) = users.minMaxByOrNull { it.name.length }!!

// 시퀀스 기반 대량 처리
val result = bigList.seqMapNotNull { it.takeIf { cond(it) }?.transform() }
```

---

## 4. 기타 팁

- 불변 리스트 편집(copy + modified)을 표준으로 삼으면 DiffUtil 구성이 쉬워집니다.
- takeIf는 “조건 만족 시만 유지” → 연쇄 필터링에 좋지만, null 체이닝을 염두에 두세요.
- 대량 변환은 asSequence()로 중간 리스트 생성 비용을 줄이세요.
- distinctByKeepLast는 API 응답처럼 뒤에 올수록 최신인 데이터를 정리할 때 유용합니다.
- 슬라이딩/청크는 페이징, 그래프 윈도우, 배치 업로드에 활용하세요.