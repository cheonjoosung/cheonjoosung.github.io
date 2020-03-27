---
title: 프로그래머스 n진수 게임 - 2018 카카오 블라인드 채용
tags: [알고리즘, 프로그래머스]
style: fill
color: dark
description: 프로그래머스 Level 2 - [3차] n진수 게임 2018 카카오 블라이드 채용
---

## [문제](https://programmers.co.kr/learn/courses/30/lessons/17687)
0부터 숫자를 차례대로 셀 때 이를 n진법으로 변환하여 자신의 순서에 다라 숫자를 부르면 되는 게임이다.

0, 1, 2, 3, 4, 5, 6, 7, 8 을 2진법으로 변환하면 0, 1, 10, 11, 100, 101, 110, 111 이다. 이 때 사람이 두 명이고 순서가 첫 번째라면 0, 1, 1, 1 .... 변환한 숫자를 이어붙인 문자열에서 첫번쨰, 세번째, 다섯번째, 일곱번째의 번호를 부르면 된다.

[카카오 공식해설](https://tech.kakao.com/2017/11/14/kakao-blind-recruitment-round-3/)

## 접근
- 숫자 0부터 차례대로 n진법으로 변환하고 문자열로 이어붙힌다.
- 변환된 문자열의 길이가 t*m 보다 작을 때 까지 반복 (사람의 수 m x 말해야 하는 문자갯수 t = 구해야할 문자열의 최소 길이)
- 구한 문자열에서 순서 p 에 따라 원하는 문자열만 가져오기


## 풀이
**1번**
```javascript
public String solution(int n, int t, int m, int p) {
    StringBuilder sb = new StringBuilder();
    sb.append(0);
    int num = 1;

    while(sb.length() < t*m) {
        sb.append(convert(num, n));
        num++;
    }

    StringBuilder ans = new StringBuilder();

    for(int i=0 ; i<t*m ; i++)
        if(i % m == (p-1)) 
            ans.append(sb.charAt(i));

    return ans.toString();
}

public StringBuilder convert(int num, int n) {
    StringBuilder sb = new StringBuilder();
    
    while(num>0) {
        int val = num % n;
        sb.append(val >= 10 ? String.valueOf((char)('A' + val - 10)) : val);
        num /= n;
    }
    return sb.reverse();
}
```
- 숫자 1부터 시작하여 convert() 메소드 호출
- 11~16진 사이의 표현을 위해 (char)('A' + val - 10) 'A'가 기준점이 되고 (val-10)의 값에 따라 B, C, D, E, F 의 값이 나옴
- convert에서 구한 문자열에서 i % m == p-1 이 되는 값을 ans에 담아준다. 2명이라고 하면  0, 1, 0, 1, 0, 1 ... 순으로 말해야 하는 숫자의 인덱스가 반복되니 i 를 사람수로 나눠주면 된다.

n진수 게임 문제는 최소로 구해야하는 문자열의 길이와 숫자를 n진법으로 변환하는 것을 물어보는 문제 같다. **val >= 10 ? String.valueOf((char)('A' + val - 10)) : val);** 이 코드를 기억하고 있으면 다양한 곳에 활용이 가능할 것 같다.

## 결론
- **카카오 문제는 문자열 활용과 관련된 문제**들이 많은 것 같다.