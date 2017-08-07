---
layout: post
title:  "Codeground 마라톤 경로"
date:   2017-08-06
categories: algorithm
highlight: false
image: /images/it/algorithm/algo.jpg
tag: algorithm
---

`CodeGround 의 마라톤 경로 풀이에 관한 포스팅입니다.` 처음에 착각을 했던 부분이 시작점에서 목적지 까지 가는 경로 중 가장 짧은 거리를 찾는 문제라고 생각해서 풀다보니 답이 안나왔습니다. 왜냐하면 필요한 생수 이상의 거리중 가장 짧은 거리를 찾아야 하므로 시작점부터 목적지까지 가는 거리와 생수 갯수를 모두 구해야 하는 점이 핵심인 문제였다.

<br><b>1) `주어진 문제를 풀이`</b><br>
<p>=) 1) 주어진 인풋값이 0보다 작으면 그곳을 지날 때 물을 하나 획득한다.</p>
<p>=) 2) 2개의 주어진 지점은 고도값의 절대값 차이이다.</p>
<p>=) 3) 목적지까지 필요한 생수 갯수 이상과 고도값의 차이의 합이 가장 작은 것을 추출하면 된다.</p>

<br><b>2) `이제 정리된 문제를 코딩으로 옮기기`</b><br>
<p>=) 1) 각 지점마다 물의 갯수와 누적된 고도값의 차이의 합을 저장하기 위해 3차원 배열 선언 dist[x][y][water] 이때, 물은 min(10, m*n+m+n) 이기에 10으로 두어도 되고 최소로 두어도 상관이 없다.</p>
<p>=) 1-1) dist 의 값을 0,0,0 을 제외하고 모든 값에 -1 을 할당해 준다. 99999999 와 값을 최대값을 넣어주고 최소값을 찾아도 되고 선택은 자유다. 하지만 0으로 초기화 하면 안된다. -11 과 11 의 경우 고도차는 0이기 때문에 구하는 과정에서 해당 경우가 제외가 될 수 있기 때문이다. </p>
<p>=) 1-2) 0,0 에서 오른쪽으로만 가거나 위쪽으로만 갔을 때의 값을 비교할 일은 없다. 오직 한가지의 길만 존재하기 때문에.... </p>
<p>=) 2) y의 값 i = 0 일 때 (j, 0) , k= 10(or 필요한 생수까지) to 0 반복하면서 고도차이의 합과 생수 체크 </p>
<p>=) 2-1) dist[j-1][0][k] == -1 이면 없는 값이므로 continue</p>
<p>=) 2-2) dist[j-1][0][k] != -1 이면 dist[j][0][k] = 두 점 사이의 고도차이의 합 + dist[j-1][0][k] 이전까지의 고도차이의 합을 더해준다.</p>
<p>=) 2-3) 만약, 음수라면 k != 10(물의 사이즈) dist[j][0][k+1] 에 저장, k == 10 이면 dist[j][0][k] 에 저장</p>
<p>=) 3) y의 값 j = 0 일 때 (0, i) , k=0 ~ 10(or 필요한 생수까지) 반복하면서 고도차이의 합과 생수 체크 </p>
<p>=) 3-1) dist[0][i-1][k] == -1 이면 없는 값이므로 continue</p>
<p>=) 3-2) dist[0][i-1][k] != -1 이면 dist[0][i][k] = 두 점 사이의 고도차이의 합 + dist[0][i-1][k] 이전까지의 고도차이의 합을 더해준다.</p>
<p>=) 3-3) 만약, 음수라면 k != 10(물의 사이즈) dist[0][i][k+1] 에 저장, k == 10 이면 dist[0][i][k] 에 저장</p>
<p>=) 4) j != 0 && i != 0 일 때, k= 10(or 필요한 생수까지) to 0 반복하면서 고도차이의 합과 생수 체크. 이 때에는 물의 갯수가 높은 것부터 아래로 내려가기 때문에 기존의 값가 비교해야 되는 경우가 생긴다. 이때에 최소값으로 저장한다.</p>
<p>=) 4-1) dist[0][i-1][k] == -1 && dist[j-1][0][k] == -1 이면 아래나 왼쪽에서 오는 경우(물을 k 카운트를 가지고 오는 경우)가 없으므로 continue</p>
<p>=) 4-2) dist[0][i-1][k] == -1 && dist[j-1][0][k] != -1 왼쪽에서만 오는 경우, 2-1 ~ 2-3 과 동일한 방법으로 진행하나 dist[j][i][k+1] or dist[j][i][k] 에 저장할 때 값이 -1이 아니면 그냥 저장하고 -1 이 아닐 경우에는 값이 존재하는 것이므로 Math.min 을 이용해서 최소값으로 저장한다. </p>
<p>=) 4-3) dist[0][i-1][k] != -1 && dist[j-1][0][k] == -1 아래쪽에서만 오는 경우, 3-1 ~ 3-3 과 동일한 방법으로 진행하나 dist[j][i][k+1] or dist[j][i][k] 에 저장할 때 값이 -1이 아니면 그냥 저장하고 -1 이 아닐 경우에는 값이 존재하는 것이므로 Math.min 을 이용해서 최소값으로 저장한다.</p>
<p>=) 4-3) dist[0][i-1][k] != -1 && dist[j-1][0][k] != -1 아래와 왼쪽에서 둘다 오는 경우, dist[j-1][i][k] + leftDistance 와 dist[j][i-1][k] + downDistance 중 최소를 선택한다. 이때도 최소로 선택된 값을 dist[j][i][k+1] or dist[j][i][k] 에 저장할 때 값이 -1이 아니면 그냥 저장하고 -1 이 아닐 경우에는 값이 존재하는 것이므로 Math.min 을 이용해서 두개의 값중 최소값으로 저장한다.</p>
<p>=) 5) 좌표 지점에 물의 갯수를 가지고 갈 때의 고도차이의 최소 합을 저장하는 방식으로 이중포문을 돌면서 진행한다. 마지막 지점에 도달하면 모든 경로를 이동하면서 물의 갯수와 고도차이의 최소합이 저장되어 있다. 이 문제의 핵심은 이동할 때마다 거리를 저장하는 프로그래밍 기법을 사용해야 한다는 점이다.</p>

{% highlight ruby %}

Answer = 0;

int row = sc.nextInt();
int col = sc.nextInt();
int minWater = sc.nextInt();

int[][] arr = new int[row + 1][col + 1];
int[][][] dist = new int[row + 1][col + 1][10 + 1]; // 지점을 이동할 때마다 물 필요+거리

for (int i = 0; i < (col + 1); i++)
  for (int j = 0; j < (row + 1); j++)
    arr[j][i] = sc.nextInt();

// 초기화
for (int[][] first : dist)
  for (int[] second : first)
    Arrays.fill(second, -1 );

dist[0][0][0] = 0;

for (int i = 0; i < (col + 1); i++) {
  for (int j = 0; j < (row + 1); j++) {
    if(i==0 && j==0) continue;

    int left = -1;
    int down = -1;

    if(j >= 1) {
      left = getDistance(arr[j][i], arr[j-1][i]);
    }
    if(i >= 1) {
      down = getDistance(arr[j][i], arr[j][i-1]);
    }

    if(left != -1 && down == -1) { // j=0 일때
      for(int k=10; k>=0 ; k--) {
        if(dist[j-1][i][k] == -1) continue;

        if( isNagative(arr[j][i]) ) {
          if(k != 10) {
            dist[j][i][k+1] = dist[j-1][i][k] + left;
            } else {
              dist[j][i][k] = dist[j-1][i][k] + left;
            }
          } else {
            dist[j][i][k] = dist[j-1][i][k] + left;
          }
        }
    } else if(left == -1 && down != -1) { // i=0 일때
      for(int k=10; k>=0 ; k--) {
        if(dist[j][i-1][k] == -1) continue;

        if( isNagative(arr[j][i]) ) {
          if(k != 10) {
            dist[j][i][k+1] = dist[j][i-1][k] + down;
            } else {
              dist[j][i][k] = dist[j][i-1][k] + down;
            }
        } else {
          dist[j][i][k] = dist[j][i-1][k] + down;
          }
        }
      } else { // j >= 1 && j >= 1일 때
        for(int k=10; k>=0 ; k--) {
          if(dist[j-1][i][k] == -1 && dist[j][i-1][k] == -1) continue;

          if(dist[j-1][i][k] != -1 && dist[j][i-1][k] == -1) { // 왼쪽으로만 오는 길만 존재
            if( isNagative(arr[j][i]) ) {
              if(k != 10) {
                if(dist[j][i][k+1] == -1) {
                  dist[j][i][k+1] = dist[j-1][i][k] + left;
                } else {
                  dist[j][i][k+1] = Math.min(dist[j][i][k+1], dist[j-1][i][k] + left);
                }
							} else {
                if(dist[j][i][k] == -1) {
                  dist[j][i][k] = dist[j-1][i][k] + left;
                  } else {
                    dist[j][i][k] = Math.min(dist[j][i][k], dist[j-1][i][k] + left);
                  }
                }
              } else {
                if(dist[j][i][k] == -1) {
                  dist[j][i][k] = dist[j-1][i][k] + left;
                } else {
                  dist[j][i][k] = Math.min(dist[j][i][k], dist[j-1][i][k] + left);
                  }
							}
          } else if(dist[j-1][i][k] == -1 && dist[j][i-1][k] != -1) { // 아래로 오는 길만 존재
            if( isNagative(arr[j][i]) ) {
              if(k != 10) {
                if(dist[j][i][k+1] == -1) {
                  dist[j][i][k+1] = dist[j][i-1][k] + down;
                } else {
                  dist[j][i][k+1] = Math.min(dist[j][i][k+1], dist[j][i-1][k] + down);
                }
              } else {
                if(dist[j][i][k] == -1) {
                  dist[j][i][k] = dist[j][i-1][k] + down;
                  } else {
                    dist[j][i][k] = Math.min(dist[j][i][k], dist[j][i-1][k] + down);
                  }
                }
              } else {
                if(dist[j][i][k] == -1) {
                  dist[j][i][k] = dist[j][i-1][k] + down;
                } else {
                  dist[j][i][k] = Math.min(dist[j][i][k], dist[j][i-1][k] + down);
                }
              }
            } else {
              if( isNagative(arr[j][i]) ) {
                if(k != 10) {
                  if(dist[j][i][k+1] == -1) {
                    dist[j][i][k+1] = Math.min(dist[j-1][i][k] + left, dist[j][i-1][k] + down);
                  } else {
                    int val = Math.min(dist[j-1][i][k] + left, dist[j][i-1][k] + down);
                    dist[j][i][k+1] = Math.min(dist[j][i][k+1], val);
                  }
                } else {
                  if(dist[j][i][k] == -1) {
                    dist[j][i][k] = Math.min(dist[j-1][i][k] + left, dist[j][i-1][k] + down);
                  } else {
                    int val = Math.min(dist[j-1][i][k] + left, dist[j][i-1][k] + down);
                    dist[j][i][k] = Math.min(dist[j][i][k], val);
                  }
                }
              } else {
                if(dist[j][i][k] == -1) {
                  dist[j][i][k] = Math.min(dist[j-1][i][k] + left, dist[j][i-1][k] + down);
                } else {
                  int val = Math.min(dist[j-1][i][k] + left, dist[j][i-1][k] + down);
                  dist[j][i][k] = Math.min(dist[j][i][k], val);
                }
              }
            }
          }
        }
      }
    }

    for(int k=minWater ; k <= 10 ; k++) {
      if(dist[row][col][k] == -1) continue;

      if(Answer == 0) {
        Answer = dist[row][col][k];
        continue;
      }

      Answer = Math.min(Answer, dist[row][col][k]);
    }

    System.out.println("Case #" + (test_case + 1));
    System.out.println(Answer);
  }

{% endhighlight %}

{% highlight ruby %}
public static int getDistance(int a, int b) {
  a = Math.abs(a);
  b = Math.abs(b);

  if (a > b) {
    return (a - b);
  } else {
    return (b - a);
  }
}

public static boolean isNagative(int a) {
  if(a < 0) return true;
  else return false;
}
{% endhighlight %}

`코드그라운드 문제 풀이를 공유하는 것이 문제가 되면 바로 삭제하도록 하겠습니다. 풀이에 대해 이해가 안되는 것은 메일을 통해 전달 부탁드립니다. 그게 제일 빨라요`
