---
title: 프로그래머스 lv1 옹알이(1) 코틀린 (kotlin)
tags: [알고리즘, 프로그래머스]
style: fill
color: dark
description: 프로그래머스 lv1 옹알이(1) 코틀린 (kotlin)
---

## [문제](https://school.programmers.co.kr/learn/courses/30/lessons/120956)
조카가 aya, ye, woo, ma 4가지를 조합한 발음만 사용이 가능할 때 주어진 옹알이 중
발음이 가능한 단어의 갯수를 출력하는 것이다.

문자열 매칭&변환 메소드를 사용하거나 정규식을 이용해서 풀면 된다.


## 풀이 1 (매칭 & 변환)
```kotlin
fun solution(babbling: Array<String>): Int {
    var answer: Int = 0

    val based = arrayOf("aya", "ye", "woo", "ma")

    for (i in babbling.indices) {
        var str = babbling[i]

        for (j in based.indices) {
            if (str.contains(based[j])) {
                str = str.replace(based[j], " ")
            }
        }

        //println("${babbling[i]} -> $str")
        if (str.replace(" ", "") == "") {
            answer++
        }
    }

    return answer
}
```
- 각 문자열을 반복하면서 옹알이(aya, ye, woo, ma) contains() 체크
- 매칭이 된다면 replace() 로 변환
- 남은 문자열의 empty 체크

## 풀이 (정규식)
```kotlin
fun solution(babbling: Array<String>): Int {
    val regex = "aya|ye|woo|ma".toRegex()

    return babbling
        .map { it.replace(regex, "") }
        .filter { it.isEmpty() }
        .count()
}
```
- 옹알이를 정규식으로 만들기
- 람다식 map 으로 빈 filter 를 사용하여 결과값 반환
