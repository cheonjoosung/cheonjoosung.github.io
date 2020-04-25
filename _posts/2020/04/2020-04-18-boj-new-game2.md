---
title: 백준 17837번 새로운 게임 2 Java/자바 - 2019 삼성 SW 역량테스트 기출문제
tags: [알고리즘, 백준]
style: fill
color: dark
description: 백준 simulation - 새로운 게임 2 2019 삼성 SW 역량테스트 기출문제
---

## [문제](https://www.acmicpc.net/problem/17837)
주어진 NxN 2차원 배열에서 체스 말판을 정해진 규칙에 따라 이동한다. 한 칸의 4개 이상의 말이 쌓이거나 이동 횟수가 1000이 넘어가면 게임이 끝난다.

## 접근
- 좌표(x, y) 마다 말판을 순서대로 저장할 자료구조와 말판의 정보를 위한 자료구조 필요
- 말판을 순서대로 하나씩 꺼내고 좌표(x,y)를 확인한다.
- 그 좌표에서 해당 말판위에 쌓인 말판을 함께 움직이는데 흰색 칸 또는 빨간색 칸에 따라 순서가 뒤바뀌는 것을 고려한다.

## 풀이
**1번**

```javascript
public class NewGame2_17837 {
	static int n, k;

	static int [][] table = new int[12][12];
	static ArrayList<Integer>[][] order = new ArrayList[12][12];
	static ArrayList<P> list = new ArrayList<>();

	static int [] dx = {0, 0, -1, 1};
	static int [] dy = {1, -1, 0, 0};

	public static void main(String[] args) {
		Scanner sc = new Scanner(System.in);

		n = sc.nextInt();
		k = sc.nextInt();

		for(int i=0 ; i<n ; i++) 
			for(int j=0 ; j<n ; j++)
				table[i][j] = sc.nextInt();

		for(int i=0 ; i<12 ; i++) 
			for(int j=0 ; j<12 ; j++)
				order[i][j] = new ArrayList<>();

		for(int i=0 ; i<k ; ++i) {
			int x = sc.nextInt();
			int y = sc.nextInt();
			int d = sc.nextInt();

			x--;
			y--;
			d--;

			list.add(new P(x, y, d));
			order[x][y].add(i);			
		}

		int time = 0;
		while(true) {

			time++;
			if (time > 1000) break;

			for (int i = 0; i < k; i++) {
				int y = list.get(i).y;
				int x = list.get(i).x;

				int ny = y + dy[list.get(i).d];
				int nx = x + dx[list.get(i).d];

				if (!(0 <= ny && ny < n && 0 <= nx && nx < n) || table[nx][ny] == 2) {
					list.get(i).d = change(list.get(i).d);

					ny = y + dy[list.get(i).d];
					nx = x + dx[list.get(i).d];
				}

				if (!(0 <= ny && ny < n && 0 <= nx && nx < n) || table[nx][ny] == 2) { 
					/* do nothing
					 * 벽을 만나고 되돌았을 경우 고 
					 */
				} else if (table[nx][ny] == 0) { //이동칸이 흰색
					int idx = -1;
					for (int j = 0; j < order[x][y].size(); j++) { //i to size 까지 모두 이동
						int cand = order[x][y].get(j);

						if (cand == i) {
							idx = j;
						}
						if (idx == -1)
							continue;

						list.get(cand).y = ny;
						list.get(cand).x = nx;
						order[nx][ny].add(cand);
						
						if (order[nx][ny].size() >= 4) {
							System.out.println(time);
							System.exit(0);
						}
					}
					
					int size = order[x][y].size(); //제거
					for (int j = idx; j < size; j++)
						order[x][y].remove(order[x][y].size() - 1);
				} else { //이동한 칸이 빨간색
					int idx = -1;
					
					for (int j = order[x][y].size() - 1; j >= 0; j--) { //역순을 위해 j==i 인덱스 찾기
						int cand = order[x][y].get(j);

						if (cand == i) {
							idx = j;
							break;
						}
					}
					
					for (int j = order[x][y].size() - 1; j >= idx; j--) { //마지막부터 j까지 역순으로 이동
						int cand = order[x][y].get(j);

						list.get(cand).y = ny;
						list.get(cand).x = nx;
						order[nx][ny].add(cand);
						if (order[nx][ny].size() >= 4) {
							System.out.println(time);
							System.exit(0);
						}

					}
					
					int size = order[x][y].size(); //제거
					for (int j = idx; j < size; j++)
						order[x][y].remove(order[x][y].size() - 1);

				}

			}
		}

		System.out.println(-1);		
		sc.close();
	}

	public static int change(int d) {
		if(d == 0) return 1;
		else if(d == 1) return 0;
		else if(d == 2) return 3;
		else return 2;
	}
}

class P {
	int x; int y; int d;

	P(int x, int y, int d){
		this.x = x; this.y = y; this.d = d;
	}
}

```

- order 은 좌표마다 말판의 순서를 기록하기 위한 자료구조 이고 list 는 말판의 정보를 담기위한 자료구조이다
- list에서 하나씩 꺼내고 말판의 이동방향에 따라 다음 칸을 결정
- (i) 파란 또는 벽 : 방향을 바꿔준 뒤 한번 더 움직여야함. 즉,, 두 번 충돌을 고려해야함
- (ii) 흰색 : 원래 있던 좌표에서 i번째의 말판 위치를 시작점을 찾고 (x,y) -> (nx,ny) 로 order에 갱신하고 이전의 기록은 제거한다.
- (iii) 빨간색 : 흰색과 비슷하다. 역순이므로 끝에서부터 j번째를 찾고 뒤에서부터 갱신하고 이전의 기록을 제거하면 된다.

이 문제는 simulation 문제인 것 같다. 자료구조의 변형을 통해 문제를 접근해야 쉽게 풀 수 있는 것 같다.

## 결론
- **삼성전자 들어가고 싶........**