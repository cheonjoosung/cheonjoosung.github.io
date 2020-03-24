---
title: 자바 하노이의 탑 코드 재귀
tags: [자바, 손코딩]
style: fill
color: dark
description: 자바 하노이의 탑 코드 재귀 방식을 활용
---

## 하노이의 탑
3개의 축이 존재하고 n 개의 원반이 주어지고 각각의 원반의 크기는 다르다. A, B, C 3개의 축이 있다고 존재하면 A에 있는 모든 원반을 다른 축으로 옮기면 된다

**규칙**
1. 한 번에 하나의 원반만 움직일 수 있다.
2. 자신보다 작은 원반이 아래에 놓일 수 없습니다. 즉, 밑에 있는 원반은 위에 있는 원반보다 작아야 합니다.
3. 원반이 n 개라면 최소이동횟수 = 2의 n 거듭제곱 - 1

## 하노이의 탑 - 재귀
재귀를 활용하여 하노이를 작성하는 것이 일방적이다.

```javascript
void hanoi(int n, int from, int by, int to) {
  if(n == 1) move(from, to)
  else {
    hanoi(n-1, from, to, by)
    move(from, to)
    hanoi(n-1, by, from, to)
  }
}

move(int a, int b) {
  print(a + " -> " + b)
}
```
**코드 설명**
1. 3개의 각 축이 from, by, to
2. from -> to 로 옮기는 것이 목표이기에 n-1 개를 from->by로 옮기는데 to를 활용한다
3. n-1개를 from->by로 옮겼다면 from에 있는 남은 원반을 to로 이동시킨다.
4. n-1개를 by->to로 옮기는데 from 을 활용하여 이동

원반이 총 3개가 있다면 위의 2개의 원반을 어떻게 해서든 중간에 있는 축으로 옮기면 된다. 그리고 왼쪽에 남아있는 가장 큰 원반을 오른쪽에 옮긴다. 즉, n=1이 될때 까지 원반을 분할하면서 이동시킬 원반을 선택하는 것이 이 코드의 작동원리이다.

## 결론
- 하노이의 규칙을 활용하여 코드를 작성하는 것이 핵심
- IT 면접/손코딩 때 가끔씩 나오는 듯