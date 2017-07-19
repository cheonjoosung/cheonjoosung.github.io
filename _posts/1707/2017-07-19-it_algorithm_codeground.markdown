---
layout: post
title:  "Codeground 균일수"
date:   2017-07-19
categories: algorithm
highlight: false
image: /images/it/algorithm/algo.jpg
tag: algorithm
---

 `CodeGround 의 균일수에 관한 포스팅입니다.` 문제 그대로 균일한 수에 관한 문제로 입력 받은 수를 x 진법으로 표시했을 때 각 자리의 숫자가 동일하게 나오면 된다. 처음에 문제를 봤을 때는 그냥 나누고 나머지 비교하면 될 것 같은 생각이 들었지만 수학의 개념이 다소 많이 필요했다. 처음에 아무생각 없이 진행하면 속도가 2초이상 나올 수 밖에 없습니다. 10억까지의 숫자범위이기에 어떤 조건으로 해당 수의 범위를 줄일 수 있을지를 파악하는게 핵심인 문제입니다.

<br><b>1) `1 주어진 문제를 풀이`</b><br>
<p>=) 입력받은 숫자 Number 에 대해서 b 진법으로 표시했을 때 각 자리수가 동일한 수가 나오는 수를 균일수라 함. 이때 최소의 b 를 찾아라. 극단적인 경우, b>N 인 경우는 b진법으로 한자리 수가 되는데 이때도 균일수라 간주 !!! </p>
<p>=) 1) N = aaaaaa...aaaaaa b 진법 의 형태로 표시 되겠죠? -> N = a+ab+ab^2+ ...... 와 같은 식이겠죠?</p>
<p>=) 1-1) N = a(1+b+b^2+b^3+....) = a * B 로 쉽게 표시할께요</b></p>
<p>=) 1-2) b진법 이므로 a<b 가 성립되요. 즉, b=a+k(임의의 양수) 정도로 생각할 수 있겠죠? N = a(a+k) = a^2 + ak 가 됩니다. 만약 a 가 N^(1/2) 보다 커버리면 성립이 안되기에 a < N^(1/2) 루트 엔 보다 작다는 것을 알 수 있게되요.</p>
<p>=) 2) b 의 범위를 잡는게 약간 어려웠던 것 같아요. 3가지의 경우로 나눠야 됩니다. </p>
<p>=) 2-1) 균일수가 한자리 일 경우 N = a 가 되므로 b = N+1 이 됩니다. </p>
<p>=) 2-2) 균일수가 두자리 일 경우 N = a(1+b) 가 되므로 b = N/a - 1 a의 범위를 이용해서 구하면 쉽게 구할 수 있게됩니다. </p>
<p>=) 2-3) 균일수가 세자리 이상인 경우 N = a(1+b+b^2+...) 인데 n=3자리 가정 ! a가 1이라고 보면 N = 1+b+b^2 의 형태가 됩니다. 여기서 b가 루트 N 이라고 하면 N = 1 + N^(1/2) + N 이 되면서 우변이 좌변보다 크게 됩니다. 즉, b < N^(1/2) 루트 N 보다 작아집니다. </p>
<p>=) a 와 b 의 범위를 많이 좁힐 수 있습니다. </p>

<br><b>1) `2. 이제 정리된 문제를 코딩으로 옮기기`</b><br>
<p>=) 1) temp(위에서 a) = number % b 라고 놓습니다.</p>
<p>=) 2) number /= b 와 number % b 를 반복하면서 나머지를 temp 와 비교를 합니다.</p>
<p>=) 3) 모두 통과하면 해당 b 는 균일수가 됩니다. 3자리 이상에서만요. b의 범위는 2-3) 참고 </p>
<p>=) 4) 만약에 2자리 수라면 위의 로직을 안타고 하위 로직을 타야 됩니다. 2-2) b = N/a -1 을 찾으면 됩니다. </p>
<p>=) 5) N = a * X 의 형태이므로 N은 a를 약수로 가집니다. 즉, N % a == 0 이 됩니다. 그런 a 를 찾아서 계산을 하면 끝납니다. </p>
<p>=) 4) 만약에 1자리 수라면 N+1 을 정답으로 보여주면 끝입니다. </p>

{% highlight ruby %}
Answer = 0;
int number = sc.nextInt();

//균일수가 n=3 자리 N = a(1+b+b^2+b^3+.......)
for(int b=2 ; b*b <= number ; b++) {
  int temp = number % b;
  int newNumber = number;
  boolean isDetect = true;

  while(newNumber > 0) {
    if(newNumber % b != temp) {
      isDetect = false;
      break;
    }

    newNumber /= b;
  }

  if(isDetect) {
    Answer = b;
    break;
  }
}

//균일수가 n=2 자리 N = a(1+b)
if(Answer == 0) {
  for(int a=1; a*a<= number ; a++) {
    if(number % a != 0) continue;

    int b = number/a - 1;

    if(b > a) Answer = b;
  }
}

//균일수가 n=1 자리 N = a 이므로 한자리 큰 모든 수 가 균일
if(Answer == 0) Answer = number+1;

{% endhighlight %}

`코드그라운드 문제 풀이를 공유하는 것이 문제가 되면 바로 삭제하도록 하겠습니다. 풀이에 대해 이해가 안되는 것은 메일을 통해 전달 부탁드립니다. 그게 제일 빨라요`
