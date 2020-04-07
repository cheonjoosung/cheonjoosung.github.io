---
title: 프로그래머스 크레인 인형뽑기 게임 Java - 2019 카카오 개발자 겨울 인턴십
tags: [알고리즘, 프로그래머스]
style: fill
color: dark
description: 프로그래머스 Level 1 - 크레인 인형뽑기 게임 2019 카카오 개발자 겨울 인턴십
---

## [문제](https://programmers.co.kr/learn/courses/30/lessons/64061)
주어진 맵에서 인형뽑기의 움직임(moves)에 따라 인형을 뽑고 바구니에 옮겨서 저장한다. 바구니에서 같은 인형이 만나서 터진 인형의 갯수를 반환하는 문제이다.


## 접근
- 인형을 저장한 자료구조 필요. 바구니의 구조를 봤을 때 스택이 적절해 보임
- 뽑기의 움직임에 따라 주어진 board를 탐색하고 스택에 넣고 빼고의 요구사항 처리

## 풀이
**1번**
```javascript
int answer = 0;

int row = board.length;
Stack<Integer> stack = new Stack<>();

for(int pos : moves) {
    pos = pos-1;

    for(int i=0 ; i<row ; i++) {
        if(board[i][pos] != 0) {
            int val = board[i][pos];

            if(!stack.isEmpty() && stack.peek() == val) {
                stack.pop();
                answer += 2;
            } else {
                stack.push(val);
            }

            board[i][pos] = 0;					
            break;
        }
    }
}		

return answer;
```
- 주어진 board의 row는 0부터 시작한다. 즉, 뽑기가 들어가는 위치의 시작이 row-1이 아니라 0부터 1,2,3,4,... row-1 까지 올라간다.
- 뽑기의 움직임의 위치에 따라 board를 탐색하여 끝까지 가거나 0이 아닌 수를 만날때까지 탐색
- 0이 아닌 수를 만나면 스택이 비었는지 체크한다. 비어있지 않고 마지막에 넣은 인형의 값과 새로 뽑은 인형의 값이 같으면 스택에서 pop()을 호출하고 터진 인형의 수를 answer 에 더한다.
- 그 외의 경우에는 스택에 push() 를 호출하여 값을 넣는다.

이 문제에서 Stack 메소드인 isEmpty() : 비어있는지, peek() - 꼭대기 값, pop() - 마지막에 넣은 값 빼기, push() - 값 넣기에 대해 학습할 수 있다. 스택은 LIFO(Last In First Out)으로 가장 마지막에 넣은 값이 빨리 나오는 자료구조이다.

## 결론
- **마지막에 넣은 값이 필요한 문제의 경우 스택 활용**