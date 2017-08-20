---
layout: post
title:  "Codeground 김씨만 행복한 세상"
date:   2017-08-19
categories: algorithm
highlight: false
image: /images/it/algorithm/algo.jpg
tag: algorithm
---

`CodeGround 의 김씨만 행복한 세상 풀이에 관한 포스팅입니다.` 인접한 두 지점이 같은 색깔로 칠해지지 않아야 행복한 세상이 되는 문제입니다. 즉, 인접한 지점이 같은 색이면 0 을 출력 그렇지 않으면 1을 출력하는 문제입니다.

<br><b>1) `주어진 문제를 풀이`</b><br>
<p>=) 1) 주어진 인풋값을 오름차순으로 정리해야 합니다. (아래의 코딩에는 없음) </p>
<p>=) 2) 1 - 2 과 나오면 1 에는 0 을 할당, 2에는 1을 할당</p>
<p>=) 2-1) 2 - 3 이 나오면 2 에는 1 이 할당되어 있으므로 3에는 0을 할당</p>
<p>=) 2-2) 위의 과정을 반복하다가 마지막까지 인접한 지점의 값이 다르면 김씨가 행복한 세상이 된다.</p>

<br><b>2) `이제 정리된 문제를 코딩으로 옮기기`</b><br>
<p>=) 1) 주어진 인풋값을 오름차순으로 정리 </p>
<p>=) 2) 0 과 1을 할당하면서 같은 값이 나오는지를 판단</p>

{% highlight ruby %}

Scanner sc = new Scanner(System.in);

int T = sc.nextInt();

for(int test_case = 0; test_case < T; test_case++) {
  Answer = 1;

  ArrayList<Pass> list = new ArrayList<>();

  int numTown = sc.nextInt();
  int count = sc.nextInt();

  int [] arr = new int[numTown];

  Arrays.fill(arr, -1);

  for(int i=0 ; i<count ; i++) {
    int a = sc.nextInt();
    int b = sc.nextInt();

    list.add(new Pass(a, b));
  }

  for(Pass a : list) {
    int s = a.s;
    int e = a.e;

    if(arr[s-1] == -1 && arr[e-1] == -1) {
      arr[s-1] = 1;
      arr[e-1] = 0;
    } else if(arr[s-1] != -1 && arr[e-1] == -1) {
      int k = arr[s-1];
      if(k == 1) {
        arr[e-1] = 0;
      } else {
        arr[e-1] = 1;
      }
    } else if(arr[s-1] == -1 && arr[e-1] != -1) {
      int k = arr[e-1];

      if(k == 1) {
        arr[s-1] = 0;
      } else {
        arr[s-1] = 1;
      }
    } else {
      int m = arr[s-1];
      int n = arr[e-1];

      if(m == n) {
        Answer = 0;
        break;
      }
    }
  }

class Pass {
  int s;
  int e;

  public Pass(int s, int e){
    this.s = s;
    this.e = e;
  }
}


{% endhighlight %}

`코드그라운드 문제 풀이를 공유하는 것이 문제가 되면 바로 삭제하도록 하겠습니다. 풀이에 대해 이해가 안되는 것은 메일을 통해 전달 부탁드립니다. 그게 제일 빨라요`
