---
title: 백준 17822번 원판 돌리기 Java/자바 - 2019 삼성 SW 역량테스트 기출문제
tags: [알고리즘, 백준]
style: fill
color: dark
description: 백준 simulation - 원판 돌리기 2019 삼성 SW 역량테스트 기출문제
---

## [문제](https://www.acmicpc.net/problem/17822)
숫자가 적혀진 원판은 회전시키면서 인접한 숫자가 존재하면 지우고 그게 없다면 전체의 평균보다 큰 값/작은 값에 -1/+1 을 해준뒤에 원판에 남아있는 수의 합을 리턴하면 된다.

## 접근
- 원판의 값을 저장할 2차원 배열 필요
- 원판의 회전은 특정 행의 배열을 복사하여 이동을 구현
- 이동한 뒤 인접한 값이 있으면 0으로 변경하고 인접한 값이 없는 경우는 0이 아닌 수의 평균을 구하고 평균보다 큰값은 -1 작은값은 +1

## 풀이
**1번**
```javascript
public class CirclRotate_17822 {
	static int n, m, t, ans;

	public static void main(String[] args) {
		Scanner sc = new Scanner(System.in);

		n = sc.nextInt();
		m = sc.nextInt();
		t = sc.nextInt();

		int [][] arr = new int[n+2][m]; //0번째와 n+1번째는 0으로 채워진 원판

		ans = 0;

		for(int i=1 ; i<n+1 ; i++)
			for(int j=0 ; j<m ; j++)
				arr[i][j] = sc.nextInt();  //위가 0으로 판단			

		for(int i=0 ; i<t ; i++) {
			int idx = sc.nextInt(); //몇번째
			int dir = sc.nextInt(); //방향 0이면 시계 1 반시계
			int cnt = sc.nextInt(); //몇칸 돌릴지
			
			move(arr, idx, dir, cnt);
			
			int [][] temp = new int[n+2][m];
			for(int x=0 ; x<n+2 ; x++) 
				for(int y=0 ; y<m ; y++) 
					temp[x][y] = arr[x][y];
			
			change(arr, temp);			
		}

		for(int i=1 ; i<=n ; i++)
			for(int j=0 ; j<m ; j++) 
				ans += arr[i][j];

		System.out.println(ans);
		sc.close();
	}

	public static void move(int [][] arr, int idx, int dir, int cnt) {
		int [] temp = new int[m];

		for(int t=1 ; idx*t <= n ; t++) {			
			for(int i=0 ; i<m ; i++) {
				if(dir == 0) temp[(i+cnt) % m] = arr[idx*t][i]; //시계
				else temp[(i-cnt+m) % m] = arr[idx*t][i]; //반시계
			}

			arr[idx*t] = temp.clone();
		}
	}

	//change () 아래의 코드에서 설명
}
```

- 배열을 생성할 때 인접한 값의 처리를 쉽게 하기 위해서 n+2 만큼의 크기 선언.. 0번째와 n+1번째는 모두 0이 들어가있다.
- 정해진 수의 배수가 되는 모든 원판을 회전해야 한다. 시계방향은 (i+cnt)%m 으로 그 반대는 (i-cnt)%m 표현하고 각 행의 값을 다시 원래 배열의 행에 맞게 복사한다.
- 인접한 값의 change()를 통해 구현했고 기존의 2차원 배열의 값을 복사한 2차원 배열이 필요.


```javascript
public class CirclRotate_17822 {
	static int n, m, t, ans;

	//중간생략

	public static void change(int [][] arr, int [][] temp) {
		boolean isFound = false;
		for(int i=1 ; i<=n ; i++) {			
			for(int j=0 ; j<m ; j++) {
				if(temp[i][j] == temp[i][(j+1) % m]) {              
					arr[i][j] = 0; arr[i][(j+1) % m] = 0;
					
					if(temp[i][j] != 0) isFound = true;
				}

				if(temp[i][j] == temp[i][(j-1+m) % m]) {
					arr[i][j] = 0; arr[i][(j-1+m) % m] = 0;
					
					if(temp[i][j] != 0) isFound = true;
				}

				if(temp[i][j] == temp[i-1][j]) {
					arr[i][j] = 0; arr[i-1][j] = 0;
					
					if(temp[i][j] != 0) isFound = true;
				}

				if(temp[i][j] == temp[i+1][j]) {
					arr[i][j] = 0; arr[i+1][j] = 0;
					
					if(temp[i][j] != 0) isFound = true;
				}
			}			
		}


		if(!isFound) {
			int sum = 0;
			int cnt = 0;
			for(int i=1 ; i<=n ; i++) {			
				for(int j=0 ; j<m ; j++) {
					if(arr[i][j] != 0) {
						sum += arr[i][j];
						cnt++;
					}
				}
			}

			if(cnt != 0) {
				double avg = sum / (double) cnt;
				
				for(int i=1 ; i<=n ; i++) {
					for(int j=0 ; j<m ; j++) {
						if(arr[i][j] != 0) {
							if(arr[i][j] > avg) arr[i][j] -= 1;
							else if(arr[i][j] < avg) arr[i][j] += 1;
						}
					}
				}
			}
		}

	}
}
```

**change()**
- (i,j)의 인접한 값은 (i-1, j), (i+1, j), (i, j+1), (i, j-1) 총 4가지다. 열의 값에 +1 or -1로 인해 익셉션이 발생할 수 있으므로 나머지 연산자를 활용해서 처리해준다.
- 인접한 값의 처리가 되지 않은 경우에는 평균을 구해서 +1 or -1을 해준다.

이 문제는 simulation의 조합이다. 원판으로 꼬아버린 것처럼 보이지만 배열로 그려놓고 인접한 값 처리하는게 눈에 잘 들어온다. **주의사항은 원판의 배수만큼 회전, 원판을 다 돌린 후 인접한 값이 있으면 0으로 없으면 평균처리, 평균을 구할 때 int/int 는 int 가 되기에 더블형 타입 캐스팅, 0이 아닌 수만의 평균**을 구해야 함 이 정도인것 같다.

## 결론
- **삼성전자 들어가고 싶........**