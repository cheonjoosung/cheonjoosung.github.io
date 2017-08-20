---
layout: post
title:  "Codeground 체스판 위의 길"
date:   2017-08-18
categories: algorithm
highlight: false
image: /images/it/algorithm/algo.jpg
tag: algorithm
---

`CodeGround 의 체스판 위의 길 풀이에 관한 포스팅입니다.` 1,1 -> m,n 까지 이동하는 경우의 수를 구하는 문제. 중간에 장애물이 있는 경우를 생각해서 경우의 수를 구해야 함. 전체 경우의 수에서 장애물을 거치는 경우를 제거하면 되는 문제이다.

<br><b>1) `주어진 문제를 풀이`</b><br>
<p>=) 1) 3*3 의 체스판이라면 6C3 = 6! / (3! * 3!), m*n 이라면 m+n C m or m+n C n 이 전체의 경우의 수입니다.</p>
<p>=) 2) 하지만 문제는 MOD 1_000_000_007 이므로 분모보다 문자가 작게되어 결과값이 0이 나올 수가 있습니다.</p>
<p>=) 2-1) 페르마의 소정리를 활용하면 a^(p-1) = 1 ( MOD p ) 이므로 a * a^(p-2) = 1 ( MOD p ) 이므로 a 의 MOD p 연산에 대한 역원은 a^(p-2) 이 됩니다. 즉, MOD 연산에 대해서 역원을 a^(p-2) 가 됩니다.</p>
<p>=) 2-2) 거듭제곱을 구하는 방법은 재귀를 이용하여 풉니다. 짝수와 홀수를 나눠서 푸는데 홀수일때는 지수가 (p-2)/2 +(p-2)/2 != (p-2) 가 안됩니다. 왜냐하면 3/2 = 1.5가 아니라 1이 나옵니다. 1 + 1 != 3 이 안되므로 p 를 한번더 곱해야 합니다.</p>
<p>=) 2-3) 먼저 20만! 의 역원을 구하면 그 이하의 팩토리의 역원은 20만!의 역원 * 20만 이 됩니다. 20만!의 역원은 1 / 20만! 과 같습니다. 분모와 분자의 역원을 곱하면 분모는 1이되고 분자는 20만!의 역원이 되기 때문이다. 1/20만! * 20만! = 1/19만! 가 되므로 19만!의 역원이 됩니다.</p>
<p>=) 3) 전체 경우의 수를 구했기 때문에 장애물을 거쳐서 도착지 까지 가는 경우의 수를 구하면 됩니다.</p>
<p>=) 3-1) (15, 16), (16,15), (99,98) 의 장애물이 있다고 하면 15,16 일때 그 지점까지 가는 경우의 수를 구합니다. </p>
<p>=) 3-2) (16,15) 일때는 (15,16)과 중복되는 경우가 있습니다. (16,15) 까지 오는데 (15,16)을 거치지 않고 오는 경우의 수를 구합니다. 이런식으로 모든 장애물까지 가는 경우의 수를 구한다음. 목적지까지 가는 거리를 구하면 됩니다. 그러면 시작점부터 각 장애물까지는 중복되지 않은 경우의 수를 구했고 목적지까지 가는 경우의 수를 곱하면 됩니다.</p>
<p>=) 3-3) 더하거나 곱하게 되면 값이 초과될 수 있으몰 MOD 를 해야 합니다.</p>
<p>=) 3-4) 전체 경우의 수 - 장애물 거쳐서 가는 경우의 수를 빼면 됩니다. 음수가 나올 수 있으므로 MOD 를 더해줘야 함.</p>

<br><b>2) `이제 정리된 문제를 코딩으로 옮기기`</b><br>
<p>=) 1) factorial 값과 reverFactorial 을 먼저 계산 </p>
<p>=) 2) combination 의 값을 역원을 이용하여 경우의 수를 구한다.</p>
<p>=) 2-1) 장애물이 체스판에 안에 있는지 밖에 있는지 판단.</p>
<p>=) 2-2) 각 장애물까지 중복되지 않는 경우의 수를 구한다.</p>
<p>=) 2-3) 각 장애물부터 도착지까지의 거리를 구하면 그것이 중복되지 않고 장애물을 거쳐서 가는 경우의 수가 된다.</p>

{% highlight ruby %}

static long Answer;
static int MOD = 1_000_000_007;

static long[] fact = new long[2000005];
static long[] reverseFact = new long[2000005];

public static void main(String args[]) throws Exception	{
  Scanner sc = new Scanner(System.in);

  fact[0] = 1;
  reverseFact[0] = 1;

  for(int i = 1; i < 200001; i++)
    fact[i] = i * fact[i-1] % MOD;

  long reverse = getPow(fact[200000], MOD-2);

  for(int i = 200000 ; i > 0 ; i--) {
    reverseFact[i] = reverse;
    reverse = (reverse * i) % MOD;
  }

  int T = sc.nextInt();

  for(int test_case = 0; test_case < T; test_case++) {
    Answer = 0;

    int row = sc.nextInt();
    int col = sc.nextInt();
    int block = sc.nextInt();

    long up = fact[row+col-2];
    long down = (reverseFact[row-1] * reverseFact[col-1] ) % MOD;
    long totalCount = (up * down) % MOD;

    List<Point> list = new ArrayList<>();
    List<Long> dupList = new ArrayList<>();

    for(int i=0 ; i<block ; i++) {
      int x = sc.nextInt();
      int y = sc.nextInt();

      if(x>=1 && y>=1 && x <= row && y<= col)
        list.add(new Point(x-1, y-1));
      else { //장애물 체스판 바깥에 존재
        continue;
      }
    }

    Collections.sort(list);

    for(int i = 0 ; i < list.size() ; i++) {
      long dist = getVal2(list.get(i).x, list.get(i).y);
      long temp = 0;

      for(int j = 0 ; j < i ; j++) {
        if(list.get(i).x >= list.get(j).x && list.get(i).y >= list.get(j).y) {
          long pointDist = getVal2(list.get(i).x - list.get(j).x, list.get(i).y - list.get(j).y);
          long smallDist = dupList.get(j);

          temp = ( temp + ( (pointDist * smallDist) % MOD ) )  % MOD;
        } else {
						continue;
          }
        }
      dist = dist - temp;
      dupList.add(dist);
    }

    long sum = 0;

    for(int i=0; i< dupList.size() ; i++) {
      int x = (row-1) - list.get(i).x;
      int y = (col-1) - list.get(i).y;

      long temp2 = getVal2(x, y);

      sum = (sum + ( (dupList.get(i) * temp2) % MOD ) ) % MOD;
    }

    Answer = (totalCount - sum) % MOD;
    Answer = (MOD + Answer) % MOD;

    System.out.println("Case #"+(test_case+1));
    System.out.println(Answer);
  }
  sc.close();
}

public static long getVal2(int x, int y) {
  long distOne = (fact[x+y] *  ( (reverseFact[x] * reverseFact[y]) % MOD ) ) % MOD;
  return distOne;
}

public static long getPow(long val, long p) {
  if(p == 0) return 1;
  if(p == 1) return val;

  long half = getPow(val, p/2);

  if(p % 2 == 0) {
    return (half * half) % MOD;
  } else {
    return ( ( (half * half ) % MOD ) * val ) % MOD;
  }
}

{% endhighlight %}

`코드그라운드 문제 풀이를 공유하는 것이 문제가 되면 바로 삭제하도록 하겠습니다. 풀이에 대해 이해가 안되는 것은 메일을 통해 전달 부탁드립니다. 그게 제일 빨라요`
