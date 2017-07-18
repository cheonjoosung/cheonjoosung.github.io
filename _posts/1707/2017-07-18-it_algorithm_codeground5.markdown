---
layout: post
title:  "Codeground 이항계수의 합"
date:   2017-07-18
categories: algorithm
highlight: false
image: /images/it/algorithm/algo.jpg
tag: algorithm
---

 `CodeGround 의 이항계수의 합에 관한 포스팅입니다.` 확률에서 쓰이는 Combination 을 코딩을 이용하여 컴퓨터 상에서 구현해보니 뭔가 신기하다라는 느낌? 수학을 내가 옮기다니.... 이 문제는 수학을 모르면 풀기 어렵다라는 생각이 너무 많이 든다..... 괜히 똑똑한 사람들이 수학을 잘하는게 아닌듯....... 자 그럼 시작해봅시다!!!

<br><b>1) `1 주어진 문제를 풀이`</b><br>
<p>=) n C r + n C r+1 = n+1 C r+1 의 공식을 활용하면 2개의 시그마를 쉽게 벗길수 있어요.</p>
<p>=) 제일 시작하는 항이 없을 때는 n C 0 를 더해주고 나중에 -1 을 해주면 되요</p>
<p>=) 위의 내용은 이항계수의 기초지식을 활용하면 쉽게 해결이 가능해요. <b>암튼 결론 -> [(n+m+2) Combination (n+1) -1 ]</b></p>
<p>=) n C r = n-1 C r + n-1 C r+1 이므로 재귀를 이용한 접근도 가능하지만 숫자가 클 때 배열에 담게되면 heap size 에러가 발생합니다.</p>
<p>=) <b>즉, 다른 방법을 적용해야 한다는 겁니다.</b></p>

<br><b>1) `2. 페르마의 소정리 이용`</b><br>
<p>=) a^(p−1) ≡ 1 (mod p) => a의 (p-1) 제곱을 p 로 나눈 나머지는 1이라는 내용이에요.</p> [증명방법은](https://namu.wiki/w/%ED%8E%98%EB%A5%B4%EB%A7%88%EC%9D%98%20%EC%86%8C%EC%A0%95%EB%A6%AC) 궁금하면 보세요
<p>=) 주어진 문제에서 구해야 될 값은 <b>(n+m+2)! / ( (n+1)! * (m+1)! ) MOD P</b>  라고 합시다. 윗변 : A, 아랫변 : B 라고 가정. </p>
<p>=) 1. 페르마 정리에서 <b>B^(P-1) ≡ 1 (mod P)</b>  에서 양변에 A 를 곱합니다.</p>
<p>=) 2. <b>A * B^(P-1) ≡ A (mod P)</b>  가 되죠! 여기에서 양변을 B 로 나눠줍니다.</p>
<p>=) 3. <b>A * B^(P-1) * B^(-1) = A * B^(P-2) ≡ A/B (mod P)</b> 가 됩니다. </p>
<p>=) 4. 구하려던 값은 <b>A/B % P` = `(A * B^(P-2)) % P` -> `( (A % P) * (B^(P-2) %P) ) %P</b> 로 바뀌는 이유 아래사진 참고</p>
 ![Dev Image](/images/it/algorithm/module_cal.png)

<br><b>1) `3. 이제 정리된 문제를 코딩으로 옮기기`</b><br>
<p>=) 1) <b> A % P 를 구하는 것이므로 (n+m+2)! % P</b></p>
<p>=) 2) <b> B^(P-2) % P => [(n+1)! * (m+1)!]^(P-2) ) % P</b></p>
<p>=) 2-1) <b> 분배가 적용이 되므로 [((n+1)! % P ) * ((m+1)! %P))]^(P-2) % P 로 바꿀 수 있습니다.</b></p>
<p>=) 2-2) <b> B^(P-2) % P => [(n+1)! * (m+1)!]^(P-2) ) % P</b></p>
<p>=) 2-3) <b> a^n mod m -> ( a mod m )^n 이라는 성질을 이용하면 { [(n+1)! * (m+1)!] % P }^(P-2) ) 로 정리 가능</b></p>

{% highlight ruby %}
fact[0] = 1;

for(int i = 1; i < 2000003; i++)
  fact[i] = i * fact[i-1] % MOD;

int T = sc.nextInt();

for(int test_case = 0; test_case < T; test_case++) {
  Answer = 0;

  int n = sc.nextInt();
  int m = sc.nextInt();

  long up = fact[n+m+2];
  long down1 = fact[n+1];
  long down2 = fact[m+1];
  long down = (down1*down2) % MOD;

  long result = combination(down, MOD-2);

  result %= MOD;
  result *= up;
  result %= MOD;

  Answer = (int) result;
{% endhighlight %}

<p>=) 3) factorial 의 값을 먼저 구하는데 3*2*1 MOD P -> (3 MOD P * 2 MOD P * 1 MOD P) MOD P 이므로 fact[i] = i * fact[i-1] % MOD 를 통해 미리 선언된 스태틱 배열에 값을 담아놔요.</p>
<p>=) 4) 윗변을 UP, 아랫변 Down = down1 * down2 % P !!! 2-3 참고</p>
<p>=) 5) 결론은 down^(P-2) 를 계산하면 됩니다.</p>

<br><b>1) `4. down^(P-2) 를 계산`</b><br>
<p>=) 1) <b> P-2 = 0 이면 return 1</b></p>
<p>=) 2) <b> P-2 = 1 이면 return val</b></p>
<p>=) 3) <b> 재귀를 이용해서 푼다. p/2 로 설정해서 p=2k(짝수) 와 p=2k+1(홀수) 로 나누어 계산해야함</b></p>
<p>=) 3-1) <b> p=2k(짝수) 일 때, val^k * val^k = val^p 가 됩니다.</b></p>
<p>=) 3-2) <b> p=2k+1(홀수) 일 때, val^k * val^k = val^2k != val^p 이므로 (정수의 내림현상으로 인해) val 를 한번 곱해줘서 val^k * val^k * val = val^(2k+1) = val^p 로 맞춰주면 됩니다.</b></p>

{% highlight ruby %}
public static long combination(long val, long p) {
  if(p == 0) return 1;
  if(p == 1) return val;

  long half = combination(val, p/2);   

  if(p%2 == 0) return (half*half) % MOD;
  else return ( ( (half*half) % MOD ) * val) % MOD;  
}
{% endhighlight %}

<p>=) 4) <b> MOD 로 나눠주는 것은 제곱형태라 값이 오버된다. val^p ≡ val (mod p) -> ( val^(p/2) * val^(p/2) ) ≡ p (mod p) 가 된다.</b></p>
<p>=) 4-1) <b> 라 생각하는데,,,, MOD 를 홀수번에서 한번 더 하는 이유는 값의 오버때문인거 같은데..... 왜인지 잘 모르겠다.....???? 왜지.....</b></p>
<p>=) 5) <b> 값 리턴 후 메인에서 처리해서 % P 해주고 윗변 곱해주고 다시 % P 하면 됩니다. </b></p>


`코드그라운드 문제 풀이를 공유하는 것이 문제가 되면 바로 삭제하도록 하겠습니다. 풀이에 대해 이해가 안되는 것은 메일을 통해 전달 부탁드립니다. 그게 제일 빨라요`
