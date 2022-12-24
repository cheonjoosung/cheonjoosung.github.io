---
title: 프로그래머스 lv1 크기가 작은 부분문자열 (kotlin)
tags: [알고리즘, 프로그래머스]
style: fill
color: dark
description: 프로그래머스 lv1 크기가 작은 부분문자열 (kotlin)
---

## [문제](https://school.programmers.co.kr/learn/courses/30/lessons/147355)
문자열 t와 p가 주어질 때 p 크기 만큼의 t 에서 부분 문자열을 만들어 크기를 비교하는 문제이다.

주의 해야할 점
- 1 <= p <= 18 Int 를 넘어서므로 런타임 에러가 뜬다면 크기 비교를 long 타입으로..


## 풀이 1 
```kotlin
fun solution(t: String, p: String): Int {
    var answer: Int = 0

    for (i in 0 until t.length - p.length + 1) {
        val sub = t.substring(i, i + p.length)

        // Int 비교가 아니라 Long 비교
        if (sub.toLong() <= p.toLong()) {
            answer++
        }
    }
    
    return answer
}
```
- substring() 범위와 크기 비교시 Long 타입만 신경쓰면 됩니다.

## 풀이2 - 람다식
```kotlin
fun solution2(t: String, p: String): Int {
    return (0..t.length - p.length)
        .map { t.substring(it until it + p.length) }
        .count { it <= p }
}
```
- 람다식 연습을 합시다.
- 코틀린의 타입 추론능력이 빛을 발합니다. count { it <= p } 문자열을 알아서 정수형으로 인지해서 비교해줍니다.