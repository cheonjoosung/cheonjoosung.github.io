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
<p>=) 2-1) 공차의 후보가 되려면 인접된 두 수의 차이의 최대 공약수를 구하면 된다.</p>
<p>=) 2-2) 그 값의 모든 약수는 다시 공차의 후보가 되므로 약수를 구하면 된다.</p>

<br><b>2) `이제 정리된 문제를 코딩으로 옮기기`</b><br>
<p>=) 1) 공차가 0이라면 다 0이어야 한다. </p>
<p>=) 2) 그게 아니라면 인접된 수의 차이의 최대 공약수(유클리드 호제법 활용)를 구한다. </p>
<p>=) 2-1) 그 값의 약수가 모두 공차가 된다. 시간을 줄이기 위해 temp/2 부터 시작하고 count를 1에서 시작</p>

{% highlight ruby %}

static int Answer;

public static void main(String args[]) throws Exception	{
  Scanner sc = new Scanner(System.in);

  int T = sc.nextInt();

  for(int test_case = 0; test_case < T; test_case++) {
    Answer = 0;

    int seqSize = sc.nextInt();
    long [] seq = new long[seqSize];

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
      //long minVal = 99999999;
      long temp = d;

      for(int i=1; i< seqSize-1 ; i++) {
        if(temp == 1) break;

        if(temp != seq[i+1] - seq[i])
						temp = gcd(temp, seq[i+1] - seq[i]);
				}

        int result = 1;
        for(int i=1; i <= temp/2 ; i++) {
          if(temp % i == 0) {
            result++;
          }
        }
        Answer = result;
    }
  }

public static long gcd(long a, long b) {
  while (b != 0) {
    long temp = a % b;
    a = b;
    b = temp;
  }
  return Math.abs(a);
}

{% endhighlight %}

`코드그라운드 문제 풀이를 공유하는 것이 문제가 되면 바로 삭제하도록 하겠습니다. 풀이에 대해 이해가 안되는 것은 메일을 통해 전달 부탁드립니다. 그게 제일 빨라요`
