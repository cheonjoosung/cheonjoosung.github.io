---
layout: post
title:  "Codeground 바이러스"
date:   2017-08-16
categories: algorithm
highlight: false
image: /images/it/algorithm/algo.jpg
tag: algorithm
---

`CodeGround 의 바이러스 풀이에 관한 포스팅입니다.` 바이러스가 걸리면 연결된 간선을 끊고 주어진 조건을 만족하는 지를 판단하는 문제입니다.

<br><b>1) `주어진 문제를 풀이`</b><br>
<p>=) 1) 1번 조건과 2번 조건은 그래프의 부분 그래프이므로 당연한 조건입니다.</p>
<p>=) 2) 3번 조건이 이 문제를 해결하는데 가장 핵심입니다.</p>
<p>=) 2-1) 정점의 차수는 최소 k 이면서 최대 |V'| - L - 1 이어야 하므로 예시에서는 k 가 1 이고 L 이 2 V' 은 제일 처음은 1~5까지 다 있는 그래프라고 가정한다면, 1번 정점은 차수가 최대치를 넘어가므로 제거되어야 합니다.</p>
<p>=) 2-2) 1번은 제거하고 2~5번 부터 3번 조건이 부합되는지를 점검하고 다 만족하면 제거된 정점의 합을 출력하면 됩니다.</p>

<br><b>2) `이제 정리된 문제를 코딩으로 옮기기`</b><br>
<p>=) 1) 주어진 간선을 입력받을 때 a[first][second] = 1 , a[second][first] = 1 양쪽에 다 넣습니다.</p>
<p>=) 2) V' 의 정점의 갯수에 따라 최소 간선과 최대 간선을 구합니다.</p>
<p>=) 2-1) 조건을 만족하지 않으면 정점에서 제거함</p>
<p>=) 2-2) 제거하고 반복하면서 조건을 다 만족한다면 break;</p>

{% highlight ruby %}

Answer = 0;

int k = sc.nextInt(); //1개 이상의 로봇과 통신해야 되는 수
int l = sc.nextInt(); //감영된 후 L 개 이상의 로봇이 살아남도

//각 정점의 차수 G` 은 최소 k 이면서 최대 |V`| - L - 1

int n = sc.nextInt(); //그래프 정점 수
int line = sc.nextInt(); //총 간선의 갯수

int [][] a = new int[n+1][n+1];

ArrayList<Integer> removedList = new ArrayList<>();
boolean isFound = false;

for(int [] temp : a)
	Arrays.fill(temp, -1);

for(int i=0; i<line ; i++) {
  int first = sc.nextInt();
  int second = sc.nextInt();

  a[first][second] = 1;
  a[second][first] = 1;
}

int size = n;

while( size >= l) {
  size = n - removedList.size()
  int minLine = k;

  int maxLine = size - l -1;
  int totalCount = 0;

  for(int i=1 ; i<n+1 ; i++) {
    if(removedList.contains(i)) continue;

    int lineCount = 0;

    for(int j=1 ; j<n+1 ; j++) {
      if(removedList.contains(i) || removedList.contains(j)) continue;
      if(a[i][j] == 1) lineCount++;
    }

    if(lineCount >= minLine && lineCount <= maxLine) {
      totalCount++;
    } else {
      removedList.add(i);
      break;
    }
  }

  if(size == totalCount) break;
}

for(int val : removedList) {
  Answer += val;
}

if(size < l) {
  Answer = 0;
  for(int i=1; i<=n ; i++) {
    Answer += i;
  }
}


{% endhighlight %}

`코드그라운드 문제 풀이를 공유하는 것이 문제가 되면 바로 삭제하도록 하겠습니다. 풀이에 대해 이해가 안되는 것은 메일을 통해 전달 부탁드립니다. 그게 제일 빨라요`
