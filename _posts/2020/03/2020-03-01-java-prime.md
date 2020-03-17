---
title: 자바 소수 구하기 실행시간 단축 알고리즘 Java PrimeNumber Algorithm
tags: [자바, 손코딩]
style: fill
color: dark
description: 자바의 소수를 효율적이고 최적으로 구하는 방법. 자바 손코딩 
---

## 소수란 ?
자기 자신과 약수를 1만 가지고 있는 숫자를 의미하고 1은 합성수도 아니고 소수도 아닌 수다.

## 자바 소수 구하기 버전 1
자바의 소수를 구하는 방법은 반목문 2개와 조건문을 잘 활용하면 쉽게 구현이 가능하다

1~100까지 중 소수인 수를 찾는다고 가정해보자

```javascript
int n = 100

for(int i=2 ; i<=n ; i++) {
  boolean isPrime = true;

  for(int j=2 ; j<i ; j++) {
    if(i % j == 0) {
      isPrime = false;
      break;
    }
  }
}
```
여기서 n=100 작은 숫자이기에 100*100=10,000 번을 해도 느리지가 않다. 하지만 그 숫자가 더욱 커진다면 알고리즘을 느려질 수 밖에 없기에 다른 방법을 고민해야 한다.

## 자바 소수 구하기 버전 2
소수는 합성수가 아닌 수 이기에 배수를 활용하면 반복되는 횟수를 많이 줄일 수 있다.

```javascript
booeln [] isNotPrime = new int[size + 1] //1부터 시작하기에 사이즈는 하나 더 크게
isNotPrime[0] = true
isNotPrime[1] = true

for(int i=2 ; i<=n ; i++) {
  if( isNotPrime[i] ) continue;

  for(int j=2 ; i*j<=n ; j++) {
    isNotPrime[i*j] = true;
  }
}
```

배수를 활용 i=2 일때 j=1 ... 숫자.. 하면서 i*j 는 무조건 약수를 가지는 수가 되고 isNotPrime[ixj] = true를 통해 해당 숫자는 소수가 아니라는 표시를 기록해준다. `소수 구하기 버전 1`에서 for문을 반드시 size X size 만큼 필요하다면 `소수 구하기 버전 2`는 반복 횟수를 획기적으로 줄일 수 있다.

## 결론
- 배열을 활용하여 이전의 연산을 저장하고 활용하기에 시간 단축 활용 - `메모이제이션 기법`이라고 부름
- 소수를 한번만 구하는 문제가 아니라 여러 번을 요구하는 경우 배열을 사용하기에 저장한 배열에서 바로 꺼내어 사용할 수 있음