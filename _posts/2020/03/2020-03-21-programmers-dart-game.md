---
title: 프로그래머스 다트게임 자바 - 2018 카카오 블라인드 채용
tags: [알고리즘, 프로그래머스]
style: fill
color: dark
description: 프로그래머스 Level 1 - [1차] 다트게임 2018 카카오 블라이드 채용
---

## [문제](https://programmers.co.kr/learn/courses/30/lessons/17682)
주어진 문자열을 규칙에 따라 더하는 문제로 내용을 그대로 구현하면 된다.

S, D, T 에 따라 거듭제곱의 횟수가 달라지고 옵션인 *, # 에 따라 점수가 x2 또는 x(-1)을 해야한다.

[카카오 공식해설](https://tech.kakao.com/2017/09/27/kakao-blind-recruitment-round-1/)

## 접근
- 방식 1 - 정규식과 패턴을 활용을 통해 숫자 단위로 끊어주기 
- 방식 2 - 문제를 요구사항에 맞춰서 코드로 구현한다. 


## 풀이
**1번** 방식 1 어떤분이 작성하신 코드
```javascript
String reg = "([0-9]{1,2}[S|T|D][*|#]{0,1})";
...
...
...
//생략
```
- 정규식이 익숙치 않아서 이렇게 푸는 것이 어려워서 구현을 포기했습니다. ㅜㅜ

**2번** 방식 2
```javascript
public int solution(String dartResult) {
        int answer = 0;
        int [] point = new int[3];
        int idx = 0;

        String temp = "";

        for(int i=0 ; i<dartResult.length() ; i++) {
            char c = dartResult.charAt(i);

            if(c >= '0' && c <= '9') {
                temp += String.valueOf(c);
            } else if(c == 'S' || c == 'D'|| c == 'T'){
                int p = Integer.parseInt(temp);
                
                if(c == 'S') p = (int) Math.pow(p, 1);
                else if(c == 'D') p = (int) Math.pow(p, 2);
                else if(c == 'T') p = (int) Math.pow(p, 3);
                
                point[idx++] = p;
                temp = "";
            } else {
                if(c == '#') {
                    point[idx-1] *= -1;
                } else {
                    point[idx-1] *= 2;
                    if(idx-2 >= 0) point[idx-2] *= 2;
                }
            }

        }
        
        for(int val : point) {
            answer += val;
        }
        
        return answer;
    }
}
```
- 문자열을 탐색하면서 0~9사이의 숫자인지 체크
- 숫자라면 ? temp에 하나씩 담아서 숫자로 만듬(1, 0 이면 -> 10)
- S, D, T 일 경우 ? 문자에 따라 숫자의 거듭제곱의 횟수를 달리하여 점수를 구하고 배열에 저장
- #일 경우 ? 현재 점수의 x(-1) 
- *일 경우 ? 현재 점수와 이전 점수 x2를 하지만 인덱스 에러 주의

다트게임 문제는 문자열을 주어진 요구사항에 따라 예외처리를 만들어 점수를 구할 수 있는지를 물어보는 같다. 또한, 주어진 문자열을 정규식과 토크나이저를 활용하여 자르는 방법을 구현할 수 있는지를 물어보는 문제일 수도 있다.

## 결론
- 정규식.. 강의는 인프런에 있습니다
- **카카오 문제는 문자열 활용과 관련된 문제**들이 많은 것 같다.