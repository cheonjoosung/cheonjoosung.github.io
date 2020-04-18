---
title: 백준 17779번 게리맨더링 2 Java/자바 - 2019 삼성 SW 역량테스트 기출문제
tags: [알고리즘, 백준]
style: fill
color: dark
description: 백준 bfs + simulation - 게리맨더링 2 2019 삼성 SW 역량테스트 기출문제
---

## [문제](https://www.acmicpc.net/problem/17779)
2차원 배열을 정해진 규칙에 따라 5개의 구역으로 나누고 각 선거구마다의 최대와 최소의 차이가 작은 값을 구하는 문제이다.

## 접근
- 기준점(x, y)에 따라 모든 경우의 수를 만든다. 규칙 d1, d2 에 따라 가능한 점만
- 구역을 나눈뒤에 합을 구한뒤 최소와 최대의 차이가 작은 값을 구한다.

![priview](https://i.imgur.com/rjyat8s.png)

보라색점이 기준이고 빨간색의 다른 3개의 점이 기준점을 바탕으로 만든 기준이다.
4개의 점은 (x, y), (x+d1, y-d1), (x+d1+d2, y-d1+d2), (x+d2, y+d2) 이렇다.

## 풀이
**1번**
```javascript
public class Vote_17779 {
	static int ANS, n, total;
	static int [][] arr;
	
	public static void main(String[] args) {
		Scanner sc = new Scanner(System.in);
		
		n = sc.nextInt();
		
		ANS = Integer.MAX_VALUE;
		
		arr = new int[n+2][n+2];
		
		for(int i=1 ; i<=n ; i++) 
			for(int j=1; j<=n ; j++) {
				arr[i][j] = sc.nextInt();
				total += arr[i][j];
			}
				
		int [] d = new int[2];
		
		for(int x=1 ; x<=n ; x++) {
			for(int y=1 ; y<=n ; y++) {
				dfs(0, d, x, y);				
			}
		}	
		
		System.out.println(ANS);
		sc.close();
	}
	
	public static void dfs(int cnt, int [] d, int x, int y) { 
		if(cnt == 2) {
			if(x+d[0]+d[1] <= n && y-d[0] >= 1 && y+d[1] <=n && y>1) {
				calc(d, x, y);
			}
			return;
		}
		
		for(int i=1 ; i<=n ; i++) {
			d[cnt] = i;
			dfs(cnt+1, d, x, y);
		}
	}
}
```

- 배열의 시작이 1부터 시작이므로 크기를 +2만큼 더 크게 만들고 1 부터 n 까지만 처리한다.
- d[] 는 기준점으로부터 다른 3개의 점을 만들기 위한 배열이다.
- 문제에 주어진 규칙을 만족하는 모든 경우의 수에 대해서 계산을 진행하면 된다.


```javascript
public class CirclRotate_17822 {
	static int n, m, t, ans;

	//중간생략

	public static void calc(int [] d, int x, int y) {
	int sum1 = 0; int sum2 = 0; int sum3 = 0; int sum4 = 0; int sum5 = 0;

	int temp = y;
	for(int r=1; r<x+d[0] ; r++) {
		if(r>=x) temp=temp-1;
		for(int c=temp ; c>=1 ; c--) {
			sum1 += arr[r][c];				
		}
	}
	//System.out.println("SUM1 : " + sum1);
	
	temp = y+1;
	for(int r=1; r<=x+d[1] ; r++) {
		if(r>x) temp++;
		for(int c=temp ; c<=n ; c++) {
			sum2 += arr[r][c];
		}
	}
	//System.out.println("SUM2 : " + sum2);
	
	temp = y-d[0];
	for(int r=x+d[0]; r<=n ; r++) {
		for(int c=1 ; c<Math.min(temp, y-d[0]+d[1]) ; c++) {
			sum3 += arr[r][c];
		}
		
		if(r < x+d[0]+d[1]) temp++;
	}
	//System.out.println("SUM3 : " + sum3);
	
	temp = y+d[1];
	for(int r=x+d[1]+1; r<=n ; r++) {
		for(int c=n ; c>=Math.max(temp, y-d[0]+d[1]) ; c--) {
			sum4 += arr[r][c];
		}
		
		if(r <= x+d[0]+d[1]) temp--;
	}
	//System.out.println("SUM4 : " + sum4);
	
	sum5 = total - (sum1+sum2+sum3+sum4);
	int max = Math.max(sum1, Math.max(sum2, Math.max(sum3, Math.max(sum4, sum5))));
	int min = Math.min(sum1, Math.min(sum2, Math.min(sum3, Math.min(sum4, sum5))));
	
	ANS = Math.min(ANS, max - min);
	}
}
```

- 가장 무식한 방벙으로 구역마다 합을 구했다.
- 쉬운 방법으로는 각 구역을 직사각형으로 만들고 각 구역마다 5구역과 겹치는 부분만 제외하면 된다.

이 문제는 bfs + simulation의 조합이다. 기준점으로부터 다른 3개의 기준점을 만드는 방법과 구역에 따라 합을 쉽게 구하는 법만 잘 만들면 쉽다.

## 결론
- **삼성전자 들어가고 싶........**