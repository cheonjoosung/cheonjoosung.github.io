---
title: 프로그래머스 예산 자바 - Summer/Winter Coding(~2018)
tags: [알고리즘, 프로그래머스]
style: fill
color: dark
description: 프로그래머스 Level 1 - 예산(Summer/Winter Coding(~2018))
---

## [문제](https://programmers.co.kr/learn/courses/30/lessons/12982)
주어진 배열에는 부서별로 신청한 금액이 들어있고 총 예산에 맞춰서 지원할 수 있는 부서의 수를 구하기

[1, 3, 2, 5, 4] & 예산 9 일때, 1,3,2를 선택하고 3를 리턴

## 접근
- 부서의 신청금을 하나씩 더하고 예산을 넘는지 비교
- 예산을 넘지 않는 범위에서 최대로 지원할 수 있는 부서의 수 찾기


## 풀이
정렬의 초점에 맞춰서 2개의 풀이를 제시하겠습니다.

**1번** 
```javascript
int ans = 0
int sum = 0

int [] d = [...]

for(int i=0 ; i<d.length-1 ; i++){
    for(int j=0 ; j<d.length-i-1 ; j++){
        if(d[j] > d[j+1]){
            int temp = d[j];
            d[j] = d[j+1];
            d[j+1] = temp;
            }
    }
}
      
for(int i=0;i<d.length;i++){
    if(budget >= d[i]){
        budget -= d[i];
        answer++;
    }
}
```
- 주어진 부서의 신청금액을 버블 소팅을 통해 방식을 활용하여 정렬
- 작은 신청 금액부터 빼면서 예산보다 작은지 비교하면서 지원 가능한 부서수를 하나씩 올리기

**2번**
```javascript
int ans = 0
int sum = 0

int [] d = [...]
Arrays.sort(d)

for(int i=0 ; i<d.length ; i++) {}
    if(d[i] + sum <= budget) {
        sum += d[i]
        ans++;
    }
}
```
- 주어진 부서의 신청금액을 오름차순으로 정렬
- 작은 신청 금액부터 더하면서 총액을 넘는지 비교하면서 지원 가능한 부서수를 하나씩 올리기

**1번과 2번의 차이는 정렬 방식의 차이**입니다. 1번의 해답의 경우 버블 소팅을 직접 구현했다면 2번은 Arrays.sort()를 통해 Array 클래스에서 제공하는 정렬 메소드를 활용했습니다. 버블 소팅의 경우 시간 복잡도가 n*n이 걸리고 내부 클래스의 정렬 방식은 Dual-Pivot Quick Sort로 시간의 복잡도가 버블 소팅보다 작다. 정렬을 직접 구현하기 보다는 배열 클래스의 메소드를 활용하는 것이 코딩 작성 시간도 줄이고 실행시간도 줄일 수 있다.

## 결론
- **배열을 이용하는 알고리즘 문제가 나오면 정렬의 사용 가능성부터 고려해보자**