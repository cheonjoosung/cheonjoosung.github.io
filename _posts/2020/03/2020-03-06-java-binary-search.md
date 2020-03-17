---
title: 자바 이진검색 코드 Binary Search
tags: [자바, 손코딩]
style: fill
color: dark
description: 자바로 이진검색 코드 작성
---

## 이진검색
이진검색은 탐색 알고리즘의 하나로써 주어진 자료에서 빠르게 탐색하여 원하는 값을 찾을 수 있다. 주어진 자료를 반으로 분할하고 찾고자 하는 키 값을 기준으로 정복하는 방식을 반복하여 원하는 값을 찾는 방식이다. 

**조건 : 반드시 주어진 자료는 오른차순 또는 내림차순의 정렬**

## 이진검색 버전 1 - 재귀
재귀를 활용하면 간결하게 이진검색 코드를 구현할 수 있다.

```javascript
int binarySearch(int [] arr, int key, int left, int right) {
  if(false == isSorted(arr)) 
    return // 주어진 배열이 정렬되었는지 체크

  if(key < arr[0] || key > arr[arr.lenght-1]) 
    return //키값이 주어진 자료의 범위에서 벗어났을 때

  int mid = (left + right) / 2

  if(key == arr[mid]) return mid
  else if(key < arr[mid]) {
    return binarySearch(arr, key, left, mid-1)
  } else if(key > arr[mid]) {
    return binarySearch(arr, key, mid+1, right)
  }
}
```

**전달받은 배열이 정렬되었는지 체크와 키 값의 범위를 체크하는 예외처리를 추가**했다. 찾고자 하는 값을 배열의 중간값과 비교하면서 대소여부에 따라 배열의 범위를 다르게 분할하면서 원하는 인덱스 값을 찾을 수 있다.

## 이진검색 버전 2 - 반복문
메모이제이션 기법을 활용하여 피보나치 수열의 코드를 구현해 보자.

```javascript
int binarySearch(int [] arr, int key, int low, int high) {
  if(false == isSorted(arr)) 
    return // 주어진 배열이 정렬되었는지 체크

  if(key < arr[0] || key > arr[arr.lenght-1]) 
    return //키값이 주어진 자료의 범위에서 벗어났을 때

  int mid = 0
  int res = -1

  while(low <= high) {
    mid = (low + high) / 2

    if(arr[mid] == key) {
      res = mid
      break;
    } else if(arr[mid] < key) {
      low = mid + 1
    } else {
      high = mid - 1
    }
  }

  return res
}
```

반복문을 활용해서 이진검색 기능을 구현할 수 있다. 버전 1의 경우 재귀의 스택 오버플로우만 조심하면 크게 문제될 것은 없다. 그렇기에 **버전 1보다는 버전2의 방식으로 코드를 구현하는 것을 추천**

## 결론
- 재귀가 간편하지만 반복문을 활용해서 코드를 구현하는 것이 좋다고 봄
- IT 면접/손코딩 빈출빈도 높음 + 예외처리를 함께 작성하여 추가 점수 노려볼만함