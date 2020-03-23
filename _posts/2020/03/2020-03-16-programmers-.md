---
title: 프로그래머스 문자열 내 마음대로 정렬하기[java]
tags: [알고리즘, 프로그래머스]
style: fill
color: dark
description: 프로그래머스 Level 1 - 문자열 내 마음대로 정렬하기
---

## [문제](https://programmers.co.kr/learn/courses/30/lessons/12915?language=java)
주어진 문자열 리스트에서 특정 인덱스의 위치 n의 위치에 오름차순 정렬

n=2 & [ipad, phone, lip] -> (a, o, i) 를 기준으로 정렬되야 하므로 [ipad, lip, phone]이 됩니다.

## 접근
- 해당 인덱스의 캐릭터 값을 가져와서 비교
- 위의 비교가 다르다면 순서 비교 가능
- 위의 비교가 같다면 문자열 전체를 비교

## 풀이
2가지의 방법으로 풀어보겠습니다.

**1번** 다른 분의 풀이이고 저도 이 해답을 보면서 참신하다고 생각이 들었습니다.
```javascript
 String[] answer = {};
        ArrayList<String> arr = new ArrayList<>();
        for (int i = 0; i < strings.length; i++) {
            arr.add("" + strings[i].charAt(n) + strings[i]);
        }
        Collections.sort(arr);
        answer = new String[arr.size()];
        for (int i = 0; i < arr.size(); i++) {
            answer[i] = arr.get(i).substring(1, arr.get(i).length());
        }
        return answer;
```
- ArrayList를 만들고 문자열의 n번째 인덱스를 넣은 후, 배열을 다시 넣기
- 이 상태가 되면 기존의 정렬 기능을 사용해도 무방함 & 정렬 후 그 값을 배열에 다시 넣으면 된다

**2번**
```javascript
String [] str , int n

Arrays.sort(str, new Comparator<String>() {
	@Override
	public int compare(String o1, String o2) {
		if(o1.charAt(n) > o2.charAt(n)) return 1
		else if(o1.charAt(n) == o2.charAt(n)) return o1.compareTo(o2)
		else return -1;
	}
});
```
- 익명함수의 방식을 사용하여 오름차순 정렬. str 배열을 정렬하는데 compare() 메소드를 확장한 코드를 통해 기준 확립
- 두 문자열의 n 번째 인덱스를 비교하여 배열을 정렬, compareTo는 o1-o2 문자열의 비교 값에 따라 1, 0, -1 이 리턴된다.

**1번과 2번의 차이는 확장성의 차이**입니다. 1번의 해답의 경우 이 문제의 특수하게 적용될 수 있는 경우입니다. 여러 알고리즘의 문제를 보면 클래스 단위의 정렬이 필요할 때가 많습니다. 예를 들면 이름(문자열), 나이(정수)를 2가지 이상의 변수를 가진 객체를 정렬할 때는 **Compartor or Comparable를 활용해서 정렬**을 해야 합니다. 그러므로 Comparator & Comparable 을 연습하는 것을 추천드립니다.

## 결론
- **자바에서 Comparator, Comparable을 활용하여 대소 비교를 통해 정렬을 하는 문제들이 있다.**
- [Compator & Comparable](https://jeong-pro.tistory.com/173) 사이트를 참조하세요