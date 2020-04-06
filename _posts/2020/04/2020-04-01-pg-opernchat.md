---
title: 프로그래머스 오픈채팅방 Java - 2019 카카오 블라인드 채용
tags: [알고리즘, 프로그래머스]
style: fill
color: dark
description: 프로그래머스 Level 2 - 오픈채팅방 2019 카카오 블라인드 채용
---

## [문제](https://programmers.co.kr/learn/courses/30/lessons/42888)
오픈 채팅방에서 사용자의 출입 & 닉네임 변경에 대한 기록을 처리하는 문제이다.

명령어는 Enter, Leave, Change / 유저 아이디와 닉네임은 대소문자 구별

## 접근
- 유저의 아이디와 닉네임을 연결하는 자료구조 필요
- 닉네임이 바뀔때마다 처리하는 것은 비효율적임. 마지막의 유저 아이디의 닉네임으로 변경되므로 한번에 처리하는 것이 효율적임

## 풀이
**1번**
```javascript
ArrayList<String> list = new ArrayList<>();
HashMap<String, String> hs = new HashMap<>();

for(String temp : record) {
    String [] arr = temp.split(" ");

    if(!arr[0].equals("Leave")) 
        hs.put(arr[1], arr[2]);	
}

for(String temp : record) {
    String [] arr = temp.split(" ");

    if(arr[0].equals("Enter")) {
        list.add(hs.get(arr[1]) + "님이 들어왔습니다.");
    } else if(arr[0].equals("Leave")) {
        list.add(hs.get(arr[1]) + "님이 나갔습니다.");
    }
}


String [] answer = new String[list.size()];

for(int i=0 ; i<list.size() ; i++)
    answer[i] = list.get(i);
```
- list - 출입기록, hashmap - 아이디와 닉네임 매칭
- 주어진 배열을 서칭하면서 유저의 아이디 - 닉네임을 매핑시켜 해쉬맵에 저장. put(key, value) 는 기존의 키가 존재해도 마지막에 들어온 key-value로 변경이 된다.
- 해쉬 맵에 아이디-닉네임이 매핑을 한 후 주어진 배열의 id에 맞는 닉네임을 넣고 list 문자열에 추가한다.

이 문제에서 HashMap<String, String> 에 대한 이해도, 문자열 처리 방법(split - 자르기)에 대한 활용을 학습할 수 있다.

## 결론
- **HashMap은 중복이 안되는 자료구조이고 가장 마지막에 넣는 키의 값으로 변경이 된다.**