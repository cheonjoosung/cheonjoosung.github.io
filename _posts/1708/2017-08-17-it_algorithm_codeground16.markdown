---
layout: post
title:  "Codeground 등차수열"
date:   2017-08-16
categories: algorithm
highlight: false
image: /images/it/algorithm/algo.jpg
tag: algorithm
---

`CodeGround 의 등차수열 풀이에 관한 포스팅입니다.` 수열이 주어질 때 그안에서 공차가 될 수 있는 모든 수를 구하는 문제이다.

<br><b>1) `주어진 문제를 풀이`</b><br>
<p>=) 1) a - b - c - d - e 가 주어지면 b = a + (n * d) 나머지 숫자도 동일한 형태로 나타낼 수 있다. n 이 반드시 1일 필요는 없다. </p>
<p>=) 2) 주어진 문제에서 공차가 될 수 있는 후보를 구해야 한다. </p>
<p>=) 2-1) 공차의 후보가 되려면 두 수의 차이보다는 작아야 하기에 각 수열사이에서 가장작은 차가 공차의 후보 중 가장 큰 값이 될 수 있다. </p>
<p>=) 2-2) 공차의 후보 중 가장 큰 값의 약수는 다시 공차의 후보가 되므로 약수를 구해서 계산을 했을 때 등차수열이 되는지 확인하면 된다.</p>

<br><b>2) `이제 정리된 문제를 코딩으로 옮기기`</b><br>
<p>=) 1) 공차가 0이라면 다 0이어야 한다. </p>
<p>=) 2) 그게 아니라면 인접된 수 중에서 차이가 가장 작은 공차 후보를 선정한다.</p>
<p>=) 2-1) 그 공차의 약수를 주어진 수열에 적용하여 성립이 되는지 확인한다.</p>

{% highlight ruby %}

Answer = 0;

int seqSize = sc.nextInt();
long [] seq = new long[seqSize];

ArrayList<Integer> list = new ArrayList<>();

for(int i=0 ; i<seqSize ; i++)
  seq[i] = sc.nextLong();
  long d = seq[1] - seq[0];

  if(d == 0) {
    for(int i= 1; i< seqSize ; i++) {
      if(seq[i-1] + d == seq[i]) {
        Answer = 1;
      } else {
        Answer = 0;
        break;
      }
    }
  } else {
    //약수를 구해야 함
    //마지막까지 검증

    long minVal = 99999999;

    for(int i=1; i< seqSize ; i++) {
      if(seq[i] - seq[i-1] < minVal) {
        minVal = seq[i] - seq[i-1];
      }
    }

    for(int i=1; i <= minVal ; i++) {
      if(minVal % i == 0)
      list.add(i);
    }
    Answer = list.size();
  }
{% endhighlight %}

`코드그라운드 문제 풀이를 공유하는 것이 문제가 되면 바로 삭제하도록 하겠습니다. 풀이에 대해 이해가 안되는 것은 메일을 통해 전달 부탁드립니다. 그게 제일 빨라요`
