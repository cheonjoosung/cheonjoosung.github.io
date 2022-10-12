---
title: 코틀린 지연계산 sequence 활용하기 | 콜렉션 연산 처리 filter, map, flatMap  
tags: [Kotlin]
style: fill
color: dark
description: 코틀린 지연계산 sequence 활용하기 | 콜렉션 연산 처리 filter, map, flatMap
---

## 람다식

람다식이란 함수선언없이 코드블록으로 실행가능한 함수 형태로 작성한 방식이다.

예를 들면,
```kotlin
val btn = findViewById(R.id.button)

// 옛날방식
btn.setOnClickListener(object : View.OnClickListener{
    override fun onClick(v: View?) {
        // click
    }
})

// 람다방식
btn.setOnClickListener { v: View ->
    // click
}
```
옛날 방식으로 버튼의 클릭 리스너를 작성하면 View.OnClickListener 익명 객체를 만들어 오버라이딩을 하고 여기에 클릭처리를 했습니다.

람다식을 사용하면 불필요한 상용코드 없이 메소드만 호출하여 클릭 이벤트 처리가 가능합니다. 버튼이 하나만 있다면 큰 차이가 없지만
우리가 만드는 애플리케이션의 뷰에 대한 이벤트 처리가 많기 때문에 람다식을 사용하는 것이 코드 길이도 줄이고 가독성도 높일 수 있습니다.

람다식을 굳이 고려하지 않아도 인텔리J 기반의 안드로이드 스튜디오가 알아서 힌트를 주기 때문에 자연스럽게 바꾸시면 됩니다.


## 콜렉션 연산 && 지연 계산

목록을 관리를 하면 필연적으로 콜렉션 연산(filter, map, all, any, count, find, groupBy, flatMap, flatten 등) 을 사용한다.
이를 통해 목록의 요소의 추가/제거/필터 등을 간편하게 처리하여 원하는 데이터만 가진 목록을 만들 수 있다.

목록의 데이터의 수가 작다면 큰 문제가 없지만 목록의 데이터 수가 늘어날 수록 성능 이슈를 고려해야 한다.

밑에 코드를 살펴보면,
```kotlin
fun main() {

    /**
     * 일반 처리
     */
    val list = mutableListOf<Int>()
    repeat(100_000_000) { list.add(Random.nextInt(100_000_000)) }

    val startTime = System.currentTimeMillis()
    val newList = list.filter { it % 2 == 0 }.map { it % 3 == 0 }.toList()
    val endTime = System.currentTimeMillis()

    val diff = endTime - startTime
    println("Collection time $diff  ${diff / 1000.0}")


    /**
     * Sequence 처리
     */
    val list2 = mutableListOf<Int>()
    repeat(100_000_000) { list2.add(Random.nextInt(100_000_000)) }

    val startTime2 = System.currentTimeMillis()
    val newList2 = list2.asSequence().filter { it % 2 == 0 }.map { it % 3 == 0 }.toList()
    val endTime2 = System.currentTimeMillis()

    val diff2 = endTime2 - startTime2
    println("Collection Lazy time $diff2  ${diff2 / 1000.0}") 
    
    /** 결과 **/
    // Collection time 4294  4.294
    // Collection Lazy time 2968  2.968
}
```

두 연산은 기존 리스트에서 filter, map 을 하여 정제한 목록을 새로운 list 에 담는다. 그리고 각 결과에 대한 시간을 체크한다.
차이점은 아래의 연산에서는 리스트를 sequence 로 변환한 후에 filter, map 연산을 수행했다. 1번의 1억건에 대한 결과는 4.294초가 걸렸고 sequence를 추가한 2번의 1억건 결과는 2.968초가 걸렸다.

비슷한 연산이지만 시간 차이가 나는 것은 sequence 로 인한 지연계산 이 발생했기 때문이다.

일반적으로 collection 연산은 매번 수행을 할 때마다 새 collection 만들어서 반환하는데 자원을 사용한다.
```kotlin
public inline fun <T> Iterable<T>.filter(predicate: (T) -> Boolean): List<T> {
    return filterTo(ArrayList<T>(), predicate)
}

public inline fun <T, R> Iterable<T>.map(transform: (T) -> R): List<R> {
    return mapTo(ArrayList<R>(collectionSizeOrDefault(10)), transform)
}
```

반면에 sequence 로 넘어온 것은 해당 타입을 그대로 사용한다.
```kotlin
public fun <T> Sequence<T>.filter(predicate: (T) -> Boolean): Sequence<T> {
    return FilteringSequence(this, true, predicate)
}

public fun <T, R> Sequence<T>.map(transform: (T) -> R): Sequence<R> {
    return TransformingSequence(this, transform)
}
```

결정적인 차이는 수행을 하는 연산 방식이 다르다.
콜렉션 연산은 filter() 모두 수행 -> 콜렉션 만들고 -> map() 모두 수행 -> 콜렉션 만들고 -> 리턴이다
시퀀스를 사용한 연산은 각 항목에 대해 순차적으로 filter() -> map 을 반복하고 최종 연산은 호출하면 연기되엇던 모든 계산이 수행된다.

아래의 코드를 보면,
```kotlin
listOf(1, 2, 3, 4, 5).asSequence()
.map { print("map[$it] "); it * it }
.filter{ print("filter[$it] "); it % 2 ==0}
.toList()

println()

listOf(1, 2, 3, 4, 5)
    .map { print("map[$it] "); it * it }
    .filter{ print("filter[$it] "); it % 2 ==0}
    .toList()

/** 결과 **/
// map[1] filter[1] map[2] filter[4] map[3] filter[9] map[4] filter[16] map[5] filter[25]
// map[1] map[2] map[3] map[4] map[5] filter[1] filter[4] filter[9] filter[16] filter[25]
``` 
collection 처리는 map 의 모든 연산 결과를 출력하지만 sequence 처리는 map -> filter 수행후 최종연산시 연기되었던 중간연산을 호출한다.
원소를 재배치하고 콜렉션을 만드는 비용이 줄어들기 때문에 지연계산을 활용한 시퀀스처리가 성능적으로 우수하다고 볼 수 있다.

결론은 콜렉션 연산을 처리할 때 시퀀스를 고려하자!!