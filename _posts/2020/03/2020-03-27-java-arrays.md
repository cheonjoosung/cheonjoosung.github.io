---
title: 자바 Arrays 클래스의 메소드 활용
tags: [Java]
style: fill
color: dark
description: 자바 Arrays 클래스와 관련 메소드를 알아보자
---

## Arrays 클래스
java.lang 패키지에 속해있고 말 그래도 배열과 관련하여 개발을 할 때 아주 유용하게 쓸 수 있는 메소드를 제공하고 있다.

**사용하기 위해서는 java.util.Arrays 를 import해야 한다.**

## Arrays 클래스의 메소드
**1.sort()**;
```javascript
int [] arr = {5, 3, 4, 1, 2}
Arrays.sort(arr); //오름차순
```
- 배열을 정렬하시오 하면 삽입, 선택, 머지, 퀵 소트를 고민하는 것보다 Arrays.sort()를 호출하는게 더 빠르다.
- default가 오름차순 정렬이다.

**2.fill()**
```javascript
int n = 5;

int [] arr = new int[n];

Arrays.fill(arr, 100);

for(int i=0 ; i<arr.length ; i++)  //라인수 낭비
    arr[i] = 100;

int [][] arr2 = new int[n][n];
for(int [] temp : arr2) {
    Arrays.fill(temp, 1001);
}
```
- fill()은 배열의 값을 채워주는 유용한 메소드
- 1차원 또는 2차원의 배열에 값을 0이외의 다른 값으로 초기화 할때 아주 유용하게 사용할 수 있다. 

**3.binarySearch()**;
```javascript
int [] arr = new int[100];

for(int i=0 ; i<arr.length ; i++) 
    arr[i] = i;

int val = Arrays.binarySearch(arr, 55);
System.out.println(" " + val);
```
- 이진검색 기능을 제공해주고 값이 존재하면 해당 인덱스를 반환하고 없다면 -1를 반환
- **단, 반드시 정렬되어 있어야 함**

**4.clone()**;
```javascript
int [] arr = new int[100];
int [] arrCopy = new int[100];

for(int i=0 ; i<arr.length ; i++) 
    arr[i] = i;

arrCopy = arr.clone();

int [][] arr2 = new int[10][10];
int [][] arr2Copy = new int[10][10];

for(int [] temp : arr2)
    Arrays.fill(temp, 99);

for(int i=0 ; i<arr2.length ; i++)
    arr2Copy[i] = arr2[i].clone();


```
- 배열의 값 복사는 알고리즘 문제에서 2차원의 맵을 복사하여 사용해야 할 경우 주로 사용
- 1차원의 경우 배열.clone() 을 호출하면 끝. 2차원 배열의 경우 각 i번째 행마다 복사를 해주면 됨
- 위의 방법을 사용하지 않는 다면 for문을 2번 사용해서 하나씩 넣어야 하기에 clone()을 통해 효율적으로 코드구현

## 결론
- **알고리즘 문제나 코딩 테스트에서 많이 사용하므로 외워두고 써먹는 것이 필수**