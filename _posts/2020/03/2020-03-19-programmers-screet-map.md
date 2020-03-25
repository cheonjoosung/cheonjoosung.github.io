---
title: 프로그래머스 비밀지도 자바 - 2018 카카오 블라인드 채용
tags: [알고리즘, 프로그래머스]
style: fill
color: dark
description: 프로그래머스 Level 1 - [1차] 비밀지도 2018 카카오 블라이드 채용
---

## [문제](https://programmers.co.kr/learn/courses/30/lessons/17681)
주어진 2개의 배열을 값을 이진수로 각각 변환 후 합치게 되면 비밀지도가 나타나고 이 비밀지도를 요구사항에 맞춰서 출력하면 된다.

## 접근
- 배열의 값을 이진수로 바꾼뒤 겹치는 작업은 비트연산자의 or 연산인 ㅣ 와 같다
- 비트연산한 결과의 값을 1이면 #으로 0이면 공백으로 변환시키면 된다


## 풀이
**1번** 
```javascript
String [] ans = new String[n];

for(int i=0 ; i<n ; i++) {
    String binaryStr = Integer.toBinaryString(arr1[i] | arr2[i]);

    String temp = "";

    for(int j=0 ; j<n-binaryStr.length() ; j++) {
        temp += " ";
    }

    for(int j=0 ; j<binaryStr.length() ; j++) {
        if(binaryStr.charAt(j) == '1') temp += "#";
        else temp += " ";
    }

    ans[i] = temp;
}
```
- toBinaryString()를 통해 이진수 빠르게 구하기
- 이진수의 변환결과가 자릿수보다 작을 때 나머지 공간은 공백으로 채우기

비밀지도 문제는 문자열을 주어진 요구사항에 따라 잘 변환할 수 있는지와 이진수를 구하는 메소를 안다면 쉽게 풀 수 있습니다. 만약 이진수 변환 메소드를 모른다면 하나씩 나누면서 구해야 하는 복잡함을 거쳐야 합니다. toHexString(), toOxString() 등의 8진수 16진수 변환할 수 있는 메소드가 Integer 클래스안에 포함되어 있습니다.

## 결론
- **Integer 클래스안에는 이진수, 8진수, 16진수 변환 메소드가 있다**