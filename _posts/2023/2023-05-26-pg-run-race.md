---
title: 프로그래머스 lv1 달리기 경주 kotlin
tags: [알고리즘, 프로그래머스]
style: fill
color: dark
description: 프로그래머스 lv1 달리기 경주
---

## [문제](https://school.programmers.co.kr/learn/courses/30/lessons/178871)
배열에 달리기 순서가 있는데 특정선수가 호명될 때 앞에 선수와 바꿔주면 되는 문제이다.

## 풀이 (실패 - 시간초과)
```kotlin
fun solution(players: Array<String>, callings: Array<String>): Array<String> {        
    callings.forEach {
        when (val index = players.indexOf(it)) {
            0, -1 -> {}
            else -> {
                val temp = players[index - 1]
                players[index - 1] = players[index]
                players[index] = temp
            }
        }
    }
    
    return players
}
```

## 풀이 (성공)
```kotlin

fun solution(players: Array<String>, callings: Array<String>): Array<String> {

    val rankMap = mutableMapOf<String, Int>()
    players.forEachIndexed { index, s -> rankMap[s] = index }

    callings.forEachIndexed { index, s ->
        val callRank = rankMap[s] ?: 0
        val frontPlayer = players[callRank - 1]

        players[callRank - 1] = s
        players[callRank] = frontPlayer

        rankMap[s] = callRank - 1
        rankMap[frontPlayer] = callRank
    }


    return players
}
```
- 무지성으로 배열 탐색을 매번 indexOf() 하다보니 타임아웃이 발생
- hashMap() 을 통해 검색 속도를 빠르게...