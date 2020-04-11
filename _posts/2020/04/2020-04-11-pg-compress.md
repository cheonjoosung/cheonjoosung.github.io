---
title: 프로그래머스 문자열 압축 Java - 2020 카카오 공채
tags: [알고리즘, 프로그래머스]
style: fill
color: dark
description: 프로그래머스 Level 2 - 문자열 압축 2020 카카오 공채
---

## [문제](https://programmers.co.kr/learn/courses/30/lessons/60057)
주어진 문자열을 여러 방식으로 압축하여 만든 새로운 문자열 중 가장 짧은 길이를 반환하는 문제이다.

aabbaabbcd -> 2aabbcd 또는 2a2b2a2bcd 로 변환할 수 있지만 전자가 더 짧으므로 7을 반환하면 된다.

## 접근
- 주어진 문자열을 몇개 단위로 나눌지 정한다.
- 단위마다 문자열을 비교하면서 압축 진행. 단, 문자열의 범위를 넘어가지 않도록 주의

## 풀이
**1번**
```javascript
public int solution(String s) {
    int answer = s.length();

    for(int n=1 ; n<=s.length()/2 ; n++) {
        StringBuilder temp = new StringBuilder();
        
        for(int i=0 ; i<s.length() ; i = i+n) {
            String word = "";
            
            if(i+n >= s.length()) word = s.substring(i, s.length());
            else word = s.substring(i, i+n);
            
            int cnt = 1;
            StringBuilder sb = new StringBuilder();
            
            for(int j=i+n ; j<s.length() ; j=j+n) {
                String word2 = "";
                
                if(j+n >= s.length()) {
                    word2 = s.substring(j, s.length());
                } else {
                    word2 = s.substring(j, j+n);
                }
                
                if(word.equals(word2)) {
                    cnt++;
                    i = j;
                } else {
                    break;
                }
            }
            
            if(cnt == 1) sb.append(word);
            else sb.append(cnt).append(word);
                            
            temp.append(sb.toString());
        }

        answer = Math.min(answer, temp.toString().length());
    }

    return answer;
}
```

- 문자열을 압축할 때 문자열의 길이 / 2 가 최대로 압축할 수 있는 범위이다. 그 값을 초과할 경우 압축이 되지 않기 때문이다.
- 0 번째 인덱스부터 시작을 해야 하므로 (i, i+n) 만큼 문자열을 나누고 그 다음 위치부터 끝까지 (j = i+n, j+n) 를 반복하면서 문자열 비교
- s+n 이 문자열의 길이의 범위를 초과하는 것의 주의를 해야하므로 예외처리를 추가해야 한다.


이 문제는 요구사항을 코드로 구현할 수 있는지를 물어보는 문제이다. 

## 결론
- **카카오 문제는 문자열 활용을 참 좋아한다**