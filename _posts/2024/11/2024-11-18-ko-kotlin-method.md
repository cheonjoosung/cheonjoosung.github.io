---
title: Kotlin 자주 사용하는 메소드의 실용적인 사용법과 예제 (orEmpty, takeIf, filter, map 등)
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin 자주 사용하는 메소드의 실용적인 사용법과 예제(orEmpty(), takeIf(), filter(), map() 등)
---

## 개요

Kotlin은 간결하면서도 강력한 문법을 제공하며, 
다양한 메소드를 통해 개발자의 생산성을 극대화합니다.

이번 포스트에서는 Kotlin에서 자주 사용하는 메소드와 그 실용적인 사용법을 예제와 함께 소개합니다.
이 메소드들은 코드의 가독성과 효율성을 높이는 데 매우 유용합니다.
---

## 1. Null 안전성을 위한 메소드

#### 1.1 orEmpty()
- 정의: null인 문자열이나 컬렉션을 빈 값으로 대체.
- 적용 대상: String, List, Set, Map.
- 데이터를 다룰 때 null 체크 없이 안전한 처리 가능

```kotlin
val nullableList: List<Int>? = null
println(nullableList.orEmpty()) // 출력: []

val nullableString: String? = null
println(nullableString.orEmpty()) // 출력: ""

val result = nullableString ?: ""
val result = nullableString.orEmpty()
```


#### 1.2 isNullOrEmpty()
- 정의: 문자열이나 컬렉션이 null이거나 비어 있는지 확인.
- 조건문에서 null과 빈 상태를 간편하게 체크 가능

```kotlin
val list: List<Int>? = null
println(list.isNullOrEmpty()) // true

val str: String? = ""
println(str.isNullOrEmpty()) // true

if (str != null && str != "") { /**/ }
if (str.isNullOrEmpty()) { /**/ }
```
- null 과 "" 를 하나의 메소드를 호출하여 확인 가능

#### 1.3 isNullOrBlank()
- 정의: 문자열이 null, 빈 값(""), 또는 공백으로만 이루어져 있는지 확인
- **isNullOrEmpty() 와 비슷하나 공백만으로 이루어진지도 체크함**
- 사용자 입력 처리에서 공백 문자열을 쉽게 걸러냄

```kotlin
val text: String? = "  "
println(text.isNullOrBlank()) // true
```
- null 과 "  " 를 하나의 메소드를 호출하여 확인 가능

---

## 2. 데이터 처리 메소드

Kotlin은 Java를 대체하기 위해 설계된 만큼, 여러 기술적 차이를 통해 생산성과 안정성을 개선합니다.

#### 2.1 filter
- 조건에 맞는 항목만 추출.
- 데이터를 조건에 따라 선택적으로 필터링

```kotlin
val numbers = listOf(1, 2, 3, 4, 5)
println(numbers.filter { it % 2 == 0 }) // [2, 4]
```

#### 2.2 map
- 각 항목을 변환하여 새로운 컬렉션 생성.
- 데이터를 변환하거나 포맷 변경 시 사용.

```kotlin
val names = listOf("Alice", "Bob", "Charlie")
println(names.map { it.uppercase() }) // [ALICE, BOB, CHARLIE]
```

#### 2.3 reduce 와 fold
- reduce : 첫 번째 항목을 초기값으로 사용.
- fold : 초기값을 사용자가 지정
- 합계, 곱셈 등 연산을 처리할 때 유용

```kotlin
val numbers = listOf(1, 2, 3, 4)
println(numbers.reduce { acc, num -> acc + num }) // 10
println(numbers.fold(10) { acc, num -> acc + num }) // 20
```

---

## 3. 컬렉션 변환 및 그룹화 메소드

#### 3.1 groupBy 와 gropingBy
- 컬렉션의 항목을 키를 기준으로 그룹화
- 데이터를 카테고리별로 묶을 때 유용
- groupingBy 는 각 그룹별 카운팅 할 때 유리함

```kotlin
val words = listOf("apple", "banana", "apricot")
val grouped = words.groupBy { it.first() }
println(grouped) // {a=[apple, apricot], b=[banana]}
```

```kotlin
data class User(val name: String, val age: Int, val gender: Char)

fun main() {
    val users = listOf(
        User("Alice", 25, 'F'),
        User("Bob", 30, 'M'),
        User("Charlie", 35, 'M'),
        User("Diana", 28, 'F'),
        User("Eve", 22, 'F')
    )

    // 성별로 그룹화
    val groupedByGender = users.groupBy { it.gender }
    println(groupedByGender)
    //F=[User(name=Alice, age=25, gender=F), User(name=Diana, age=28, gender=F), User(name=Eve, age=22, gender=F)],
    //M=[User(name=Bob, age=30, gender=M), User(name=Charlie, age=35, gender=M)]
  
    // 나이대(20대, 30대)로 그룹화
    val groupedByAgeGroup = users.groupBy {
        when (it.age / 10) {
            2 -> "20대"
            3 -> "30대"
            else -> "기타"
        }
    }
    println(groupedByAgeGroup)
    //20대=[User(name=Alice, age=25, gender=F), User(name=Diana, age=28, gender=F), User(name=Eve, age=22, gender=F)],
    //30대=[User(name=Bob, age=30, gender=M), User(name=Charlie, age=35, gender=M)]

    // 성별로 사용자 수 세기
    val countByGender = users.groupingBy { it.gender }.eachCount()
    println(countByGender)
    // F=3, M=2
}
```

#### 3.2 chunked
- 컬렉션을 일정 크기의 청크로 나눔
- 대규모 데이터를 일정 단위로 처리

```kotlin
val numbers = (1..10).toList()
val chunks = numbers.chunked(3)
println(chunks) // [[1, 2, 3], [4, 5, 6], [7, 8, 9], [10]]
```

---

## 4. 컬렉션 확장 메소드

#### 4.1 firstOrNull 와 lastOrNull
- 조건을 만족하는 첫 번째 또는 마지막 항목 반환. 없으면 null
- 조건에 따라 안전하게 특정 항목 검색하고 싶을 때 { predicate } 사용

```kotlin
val list = (1..100).toList()

list.firstOrNull().also { println(it) } // 1
list.firstOrNull { it > 2 }.also { println(it) } // 3
list.lastOrNull().also { println(it) } // 100
list.lastOrNull { it <= 99 }.also { println(it) } // 100
```

#### 4.2 take 와 takeWhile
- take(n): 처음 n개의 항목 반환.
- takeWhile(predicate): 조건을 만족하는 항목 반환.
- 데이터를 부분적으로 가져올 때 사용

```kotlin
val numbers = listOf(1, 2, 3, 4, 5)
println(numbers.take(3)) // [1, 2, 3]
println(numbers.takeWhile { it < 4 }) // [1, 2, 3]
```

---

## 5. 성능 최적화를 위한 메소드

#### 5.1 asSequence()
- 컬렉션을 지연 평가(lazy evaluation) 가능한 Sequence로 변환
- 대규모 데이터 처리 시 불필요한 연산을 줄여 성능 최적화

```kotlin
val list = (1..100_000_000).toList()

// 짝수인 것들의 제곱인 리스트들의 합
var startTime = System.currentTimeMillis()
val sum1 = list
    .filter { it % 2 ==0 }
    .map { it * it }
    .sum()
var endTime = System.currentTimeMillis()
println("sum1 time ${(endTime-startTime)}") // 2890

startTime = System.currentTimeMillis()
val sum2 = list
    .asSequence()
    .filter { it % 2 ==0 }
    .map { it * it }
    .sum()
endTime = System.currentTimeMillis()
println("sum2 time ${(endTime-startTime)}") // 779
```
- 예제를 살펴볼 때 1억개 일 때 4배정도 차이가 난다
- 처리 단위가 클 수록 sequence() 사용이 반드시 필요해지고 낮은 데이터 단위라면 큰 차이는 없다
- Sequence가 유리한 경우:
  * 데이터 크기가 매우 크고 전체 데이터를 한꺼번에 처리할 필요가 없는 경우.
  * 필터링 후 일부 항목만 필요한 경우 (take, first, find 등).
  * 복잡한 체이닝 작업으로 인해 중간 리스트 생성이 많은 경우 (filter, map, flatMap 등).
- List가 유리한 경우:
  * 데이터 크기가 작고 연산이 단순한 경우.
  * 데이터에 여러 번 접근해야 하는 경우.
  * 데이터 전체를 메모리에 유지하는 것이 문제가 되지 않는 경우.

---

## 6. 기타 유용한 메소드

#### 6.1 sortedBy와 sortedDescending
- 항목을 오름차순 또는 내림차순으로 정렬

데이터 정렬
```kotlin
val numbers = listOf(5, 3, 8, 1)
println(numbers.sortedBy { it }) // [1, 3, 5, 8]
println(numbers.sortedDescending()) // [8, 5, 3, 1]
```

객체 리스트 정렬
```kotlin
data class User(val name: String, val age: Int, val gender: Char)

fun main() {
    
  val users = listOf(
    User("Alice", 25, 'F'),
    User("Bob", 30, 'M'),
    User("Charlie", 25, 'M'),
    User("Diana", 22, 'F'),
    User("Eve", 30, 'F')
  )

  // `sortedBy`와 `sortedByDescending` 조합
  val sortedUsers = users.sortedWith(
    compareBy<User> { it.age }
      .thenByDescending { it.gender }
      .thenBy { it.name }
  )

  println("Sorted Users (age -> gender -> name):")
  sortedUsers.forEach { println(it) }
  /**
    Sorted Users (age -> gender -> name):
    User(name=Diana, age=22, gender=F)
    User(name=Alice, age=25, gender=F)
    User(name=Charlie, age=25, gender=M)
    User(name=Eve, age=30, gender=F)
    User(name=Bob, age=30, gender=M)
   */
}
```

- sortedWith 와 compareBy의 사용법
  * compareBy: 키를 지정하여 정렬 조건을 설정.
    + compareBy { it.age } (age 기준으로 정렬)
  * thenBy / thenByDescending: 추가적인 정렬 조건을 설정.
    +t henBy { it.name } (name 기준 추가 정렬)
  * compareByDescending: 내림차순 정렬.

- 정렬 키를 조합한 방식
  * 간단한 키 1개: sortedBy 또는 sortedByDescending
  * 다중 키 조합: compareBy + thenBy 또는 sortedWith

정렬 키를 조합한 방식
간단한 키 1개: sortedBy 또는 sortedByDescending
다중 키 조합: compareBy + thenBy 또는 sortedWith



#### 6.2 shuffled()
- 컬렉션의 항목을 무작위로 섞음

```kotlin
val numbers = listOf(1, 2, 3, 4, 5)
println(numbers.shuffled()) // [예: 3, 5, 1, 4, 2]
```

---

## 7. 결론

Kotlin의 다양한 메소드는 데이터를 간결하고 효율적으로 처리할 수 있도록 설계되었습니다.

- Null Safety: orEmpty, isNullOrEmpty로 안정적인 코드 작성.
- 데이터 처리: filter, map, groupBy로 직관적인 데이터 변환 및 그룹화.
- 성능 최적화: asSequence로 대규모 데이터 처리 효율성 확보.

이 글에서 소개한 메소드는 실무에서도 자주 사용되며, Kotlin 개발의 생산성을 크게 높여줍니다.
프로젝트에서 적극 활용해 보세요! 😊

---