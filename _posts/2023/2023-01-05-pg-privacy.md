---
title: 프로그래머스 lv1 개인정보 수집 유효기간 코틀린 (kotlin) 2023 KAKAO BLIND RECRUITMENT
tags: [알고리즘, 프로그래머스]
style: fill
color: dark
description: 프로그래머스 lv1 개인정보 수집 유효기간 코틀린 (kotlin) 2023 KAKAO BLIND RECRUITMENT
---

## [문제](https://school.programmers.co.kr/learn/courses/30/lessons/150370)
약관 종류마다 동의한 날짜와 오늘 날짜를 비교하여 파기해야 되는지를 판단하면 된다.

**주의사항**
- day 28 고정
- 주어진 날짜를 년/월/일 나누고 총 일자수로 계산하면 편하다.


## 풀이
```kotlin
fun solution(today: String, terms: Array<String>, privacies: Array<String>): IntArray {
    val answer = mutableListOf<Int>()

    val todayTotal = totalDay(today)
    val map = hashMapOf<String, Int>()

    for (i in terms.indices) {
        val token = terms[i].split(" ")
        map[token[0]] = token[1].toInt()
    }

    for (i in privacies.indices) {
        val token = privacies[i].split(" ")

        val date = token[0]
        val type = token[1]

        val dateTotal = totalDay(date)
        val month = map[type] ?: 0
        val changedDay = month * 28

        if (todayTotal >= (dateTotal + changedDay)) {
            answer.add(i+1)
        }
    }

    return answer.toIntArray()
}

private fun totalDay(todayTotal: String): Int {

    var sum = 0
    val token = todayTotal.split(".")
    val y = token[0]
    val m = token[1]
    val d = token[2]

    sum += y.toInt() * 12 * 28
    sum += (m.toInt() - 1) * 28
    sum += d.toInt()

    return sum
}
```
- 주어진 날짜를 총 일자수로 바꿔주는 함수를 만든다.
- 약관 동의 받은 타입마다 날짜를 비교해주면 끝난다.

문자열을 분리하는 것과 날짜 계산을 총 일자수로 변환만 했다면 이 문제는 쉽게 풀렸을 듯...