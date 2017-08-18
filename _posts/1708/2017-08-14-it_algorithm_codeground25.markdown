---
layout: post
title:  "Codeground 3N + 1"
date:   2017-08-14
categories: algorithm
highlight: false
image: /images/it/algorithm/algo.jpg
tag: algorithm
---

`CodeGround 의 3N + 1 풀이에 관한 포스팅입니다.` 숫자가 주어질 때 짝수면 나누기 2, 홀수이면 곱하기 3 를 한 후에 더하기 1을 반복하여 최종 값이 1이 나오는 횟수의 최소수와 최대수를 구하는 문제이다.

<br><b>1) `주어진 문제를 풀이`</b><br>
<p>=) 1) 8 -> 4 -> 2 -> 1 이므로 횟수는 3이다. 10 -> 5 -> 16 -> 8 -> 4 -> 2 -> 1 이 되므로 횟수는 6이다.</p>
<p>=) 2) 주어지는 값은 횟수를 알려주고 그 횟수만큼 주어진 법칙을 반복할 때 나오는 최소와 최대의 수를 구하면 된다.</p>
<p>=) 3) 최대가 되는 값을 2^k 이 될 수 밖에 없다. 나누기 2를 하므로 그 값보다 크므로 안 되고 2의 제곱근 형태는 작을 수 밖에 없다.</p>
<p>=) 4) 최대값은 쉽게 구할 수 있으므로 최소값만 구하면 된다. 최소이므로 1부터 시작하는 것이 좋다.</p>

<br><b>2) `이제 정리된 문제를 코딩으로 옮기기`</b><br>
<p>=) 1) 1부터 차례대로 진행하면서 주어진 조건을 비교하면 된다.</p>
<p>=) 2) 짝수 일때는 N/2, 홀수 있때는 3*N + 1 을 반복하면서 카운트를 하나씩 증가한다.</p>
<p>=) 3) 결과값이 1이면 리턴을 하고 totalCount 와 주어진 input 값과 비교를 하면 된다.l</p>


{% highlight ruby %}

Answer = 0;

int k = sc.nextInt();

BigInteger max = BigInteger.valueOf(2).pow(k);

long min = 1;
long num = 0;

int totalCount = 0;

for(int i=1; ; i++) {
  num = i;

  for(int j=0 ; j<k ;j++) {
    if(num % 2 == 0) {
      num = num/2;
      totalCount++;
    } else if(num %2 !=0 && num !=1){
      num = 3*num + 1;
      totalCount++;
    } else if(num == 1) {
      break;
    }
  }

  if(totalCount == k && num == 1) {
    min = i;
    break;
  }

  totalCount = 0;
}

{% endhighlight %}

`코드그라운드 문제 풀이를 공유하는 것이 문제가 되면 바로 삭제하도록 하겠습니다. 풀이에 대해 이해가 안되는 것은 메일을 통해 전달 부탁드립니다. 그게 제일 빨라요`
