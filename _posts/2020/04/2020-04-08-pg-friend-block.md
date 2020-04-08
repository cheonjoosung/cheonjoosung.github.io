---
title: 프로그래머스 프렌즈4블록 Java - 2018 카카오 블라인드 채용
tags: [알고리즘, 프로그래머스]
style: fill
color: dark
description: 프로그래머스 Level 2 - 프렌즈4블록 2018 카카오 블라인드 채용
---

## [문제](https://programmers.co.kr/learn/courses/30/lessons/17679)
주어진 문자열을 2차원 배열로 옮긴 후에 2x2 의 정사각형안에 모든 문자가 동일한 모든 블록이 한번에 터지고 그 위에 블록들이 아래로 이동한다. 이 때 터진 블록의 총 갯수를 구하라

## 접근
- 2x2 의 안에 같은 모양을 가진 블록 찾고 터트리기
- 빈 공간 메꾸기
- 터질 블록이 없을 때 까지 위의 2가지 반복

## 풀이
**1번**
```javascript
public int solution(int m, int n, String[] board) {
    int answer = 0;
    char [][] map = new char[m][n];

    for(int i=0 ; i<m ; i++) {
        map[i] = board[i].toCharArray();
    }

    while(true) {
        int cnt = 0;

        boolean [][] isVisited = new boolean[m][n];

        for(int i=0 ; i<=m-2 ; i++) {
            for(int j=0 ; j<=n-2 ; j++) {
                if(map[i][j] == '0') continue;
                
                if(map[i][j] == map[i][j+1] && map[i][j+1] == map[i+1][j+1] && map[i+1][j+1] == map[i+1][j]) {
                    cnt += check(i,j, map, isVisited);
                }
            }
        }

        if(cnt == 0) {
            break;
        } else {
            answer += cnt;
            move(isVisited, map, m, n);
        }
    }

    return answer;
}

public int check(int x, int y, char [][] map, boolean [][] isVisited) {
    int res = 4;
    
    if(isVisited[x][y]) res--;
    if(isVisited[x+1][y]) res--;
    if(isVisited[x][y+1]) res--;
    if(isVisited[x+1][y+1]) res--;
    
    isVisited[x][y] = true;
    isVisited[x+1][y] = true;
    isVisited[x][y+1] = true;
    isVisited[x+1][y+1] = true;

    return res;
}

public void move(boolean [][] isVisited, char [][] map, int m, int n) {
    for(int j=0 ; j<n ; j++) {
        for(int i=m-1 ; i>=0 ; i--) {
            if(isVisited[i][j] == true) {
                map[i][j] = '0';
                
                for(int k=i-1 ; k>=0 ; k--) {
                    if(isVisited[k][j] == false) {
                        char temp = map[i][j];
                        map[i][j] = map[k][j];
                        map[k][j] = temp;
                        
                        boolean isTemp = isVisited[i][j];
                        isVisited[i][j] = isVisited[k][j];
                        isVisited[k][j] = isTemp;
                        break;
                    }
                }
            }
        }
        
    }
}	
```
- 주어진 board를 toCharArray() 메소드를 활용하여 2차원 배열로 값 복사
- 행은 0~m-2 까지, 열은 0~n-2 까지 시작한 위치에서 우->아래->좌->위 이므로 m-1,n-2까지 할필요가 없음
- 2x2 안에 모든 문자가 같으면 isVisited[][] 를 true 로 변경 (터진 곳을 표시하기 위함). 2x2 크기 이기에 기본 값이 4이지만 중복으로 겹치는 경우가 있으므로 이미 터진경우는 제외한다.
- 터진 곳을 다 체크했으면 move() 를 통해 빈곳을 메꾼다. **i)** 각 열마다 m-1 부터 0까지 이동하면서 터진 곳을 찾는다 (isVisited == true) 그 위치의 값을 '0'으로 바꿔준다, **ii)** 터진 그 위치보다 작은 칸에서부터 안 터진 곳을 찾아서 두 개의 값을 바꿔준다.(visited 도 포함)

이 문제는 요구사항을 코드로 옮길 수 있는가를 물어보는 문제이다. 이러한 문제형태를 백준에서는 시뮬레이션이라고 분류한다. 예외사항과 정확한 구현을 필요로하기에 알고리즘 문재애 있어서 기본기를 닦을 수 있는 문제라 생각한다.

## 결론
- **시뮬레이션 메모**