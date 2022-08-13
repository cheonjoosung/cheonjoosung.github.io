---
title: (kotlin) 코틀린 리스트 정렬하기 | 프로퍼티 2개 이상일 때 (sorted, sortedWith)
tags: [Kotlin]
style: fill
color: dark
description: 코틀린 리스트 정렬, Kotlin List Sort
---

## 자료구조 내 프로퍼티가 2개 일때 정렬로

sort() 또는 sortWith() 2가지만 알면 웬만한 리스트 정렬은 다 처리할 수 있다.

#### 1.Comparable 없이

```kotlin
data class User(
    val name: String,
    val age: Int,
    val favorite: Favorite
)

enum class Favorite(val favorite: String, val number: Int) {
    BIKE("BIKE", 1), SWIM("SWIM", 2), YOUTUBE("YOUTUBE", 3)
}

fun main() {

    val list = mutableListOf<User>().apply {
        add(User("a", 15, Favorite.BIKE))
        add(User("a", 17, Favorite.BIKE))
        add(User("a", 15, Favorite.SWIM))
        add(User("ab", 15, Favorite.BIKE))
        add(User("b", 12, Favorite.SWIM))
        add(User("b", 11, Favorite.YOUTUBE))
        add(User("b", 12, Favorite.BIKE))
        add(User("bc", 15, Favorite.BIKE))
        add(User("c", 9, Favorite.BIKE))
        add(User("c", 5, Favorite.BIKE))
        add(User("c", 15, Favorite.YOUTUBE))
        add(User("a", 13, Favorite.BIKE))
    }

    list.sortWith(compareBy({it.name}, {it.age}, {it.favorite.number}))
    //list.sortWith(compareBy({it.name}, {-it.age}, {it.favorite.number}))
}
```
sortWith()를 사용하고 프로퍼티를 비교하는데 name->age->favorite.number 순으로 정렬을 진행한다.
기본적으로 오름차순으로 정렬이 된다.

만약 **일부 프로퍼티만 내림차순으로 하고 싶다면 -키워드 추가**하면 된다.

<br>
#### 2.Comparable 사용버전

```kotlin
data class User(
    val name: String,
    val age: Int,
    val favorite: Favorite
): Comparable<User> {

    override fun compareTo(other: User): Int {
        return compareValuesBy(this, other, {it.name}, {it.age}, {it.favorite.number})
        //return compareValuesBy(this, other, {it.name}, {-it.age}, {it.favorite.number})
    }

}

enum class Favorite(val favorite: String, val number: Int) {
    BIKE("BIKE", 1), SWIM("SWIM", 2), YOUTUBE("YOUTUBE", 3)
}

fun main() {

    val list = mutableListOf<User>().apply {
        add(User("a", 15, Favorite.BIKE))
        add(User("a", 17, Favorite.BIKE))
        add(User("a", 15, Favorite.SWIM))
        add(User("ab", 15, Favorite.BIKE))
        add(User("b", 12, Favorite.SWIM))
        add(User("b", 11, Favorite.YOUTUBE))
        add(User("b", 12, Favorite.BIKE))
        add(User("bc", 15, Favorite.BIKE))
        add(User("c", 9, Favorite.BIKE))
        add(User("c", 5, Favorite.BIKE))
        add(User("c", 15, Favorite.YOUTUBE))
        add(User("a", 13, Favorite.BIKE))
    }
    
    list.sort()
    list.forEach { println(it) }
}
```
Comparable 인터페이스를 구현하고 compareTo를 사용해야 하는 경우가 있다.
빈번하게 일어나는 경우 클래스에 정의를 해놓고 collection.sort() 만 호출하면 정렬이 된다.

만약 **일부 프로퍼티만 내림차순으로 하고 싶다면 -키워드 추가**하면 된다.