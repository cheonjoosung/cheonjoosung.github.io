---
layout: post
title:  "Codeground 화학자의 문장"
date:   2017-08-05
categories: algorithm
highlight: false
image: /images/it/algorithm/algo.jpg
tag: algorithm
---

 `CodeGround 의 부분배열 풀이에 관한 포스팅입니다.` 제가 생각한 부분배열은 수학에서 배운 그 개념을 생각했는데 코드 그라운드의 부분배열의 의미는 연속된 배열의 의미입니다. 처음에는 정렬하여 제일 큰 숫자부터 더해서 합이 커지는 경우를 이용해서 구했지만 0점이 나오더라구요. 문제의 혼동이 없게끔 잘 써줬으면 하는 작은 바램을 가져봅니다. 처음에 쓴 알고리즘은 n*n*n 이었고 두번째는 n*n 그리고 마지막 수정을 거쳐서 n*n 보다 작은 시간으로 구하는 알고리즘을 찾았습니다.

<br><b>1) `주어진 문제를 풀이`</b><br>
<p>=) 1) 제일 중요한 것은 자연수의 합. 여기에서 저는 처음부터 누적해서 더하면 커진다라는 힌트를 얻었습니다.</p>
<p>=) 2) 사이즈는 10, 합은 35로 주어지고 5 1 3 5 10 7 4 9 2 8 의 배열이 있으면 누적한 배열은 5 6 9 14 24 31 35 44 46 54 가 됩니다.</p>
<p>=) 2-1) 35 이상이려면 5 부터 4까지 다 더해야합니다. 여기서 시작 인덱스를 0이 아닌 35가 넘는 6번째 부터 하면 된다는 것을 알 수 있습니다. 6번째 이상부터 합이 35이상이 나오므로 그 이전까지 더한값은 아무런 의미가 없습니다.</b></p>
<p>=) 2-2) 누적[6] - 누적[0] 을 하면 30 이 되고 이 값은 1번째부터 6번째가지 더한 값입니다. 합이 넘지 않으므로 더 돌릴필요가 없겠죠?</p>
<p>=) 2-3) 그다음 누적[7]-누적[0]=44-5= 39 이므로 합이 35을 넘습니다. 즉, 1번째부터 7번째까지의 합이므로 부분배열의 합이 넘는 배열 카운트는 6를 넘지 않는 다는 것을 알 수 있습니다. 그다음은 1을 증가시켜 누적[7] - 누적[1] = 38 이므로 역시 넘습니다. 2번째부터 7번째까지 합이므로 배열 카운트는 5가 됩니다. </p>
<p>=) 2-4) 이후 부터는 누적[x] - 누적[x-5(발견된 카운트)] 부터 체크하여 카운트를 하나씩 내려서 가능 여부를 파악하면서 최소 카운트를 찾으면 됩니다.</p>

<br><b>2) `이제 정리된 문제를 코딩으로 옮기기`</b><br>
<p>=) 1) 누적된 합을 더할 수 있는 partSumArr 배열을 선언합니다. </p>
<p>=) 2) 누적값을 구하면서 주어진 합 SUM 을 넘는 최소 인덱스를 찾습니다.</p>
<p>=) 3) 0 ~ 최소 인덱스까지의 합이 주어진 SUM 을 넘는 카운트를 구합니다.</p>
<p>=) 4) 넘지 않으면 3번 과정 반복. 넘었다면 그보다 작은 카운트로 다시 합을 측정하면서 내려온다.</p>

{% highlight ruby %}

for(int test_case = 0; test_case < T; test_case++) {
  int size = sc.nextInt();
  Sum = sc.nextInt();

  int [] arr = new int[size];
  int [] partSumArr = new int[size];

  Answer = 0;

  for(int i=0 ; i<size ; i++) arr[i] = sc.nextInt();


  partSumArr[0] = arr[0]; //partSum 이랑 arr 이의 누적값을 저장하기 위한 변수, part 1 = arr0+arr1, part2 = arr0+arr1+arr2 이런식으로
  int indexOverSum = 0;

  for(int i=1; i<size ; i++) {
    partSumArr[i] = partSumArr[i-1] + arr[i];
    if(partSumArr[i] >= Sum && indexOverSum == 0) {
      indexOverSum = i;   // 누적값중에 Sum 을 넘어가는 index 를 찾는다. 범위를 줄이기 위해서 시작지점을 0이 아닌 넘어가는 시점부터 계산
    }
  }

  int k = 0;

  for(int i=indexOverSum ; i < size ; i++) {
    /*
    * Count 가 발견이 되었다면 누적값의 차를 구할 때 인덱스를 0부터 시작하는 것이 아니라 i에서(Count+1) 만큼 뺀 숫자부터 시작하면 된다.
    * 최소 카운트를 찾는 것이기 때문에 3보다 큰 값을 구할 필요는 없기 때문이다.
    */

    if(Answer == 0) k = 0;
    else k = i-Answer-1;

    for(int j=k ; j <= i ; j++) {
      /*
      * partSumArr[5] - partSumarr[2] >= sum 이라면 3,4,5 배열값의 합은 Sum 을 넘으므로 Count 는 3이 된다.
      */

      if(partSumArr[i] - partSumArr[j] >= Sum) {
        if(Answer == 0) Answer = (i-j);
        if(Answer > (i-j) ) Answer = (i-j);
        else continue;
        } else {
          break;
        }
      }
    }

    System.out.println("#testcase"+(test_case+1));
    System.out.println(Answer);
}

{% endhighlight %}

<br><b>3) `다른 정리`</b><br>
<p>=) 1) j 의 시작점을 0부터 계속 반복하면 시간이 초과되지만 합을 넘는 카운트를 구하고 그 숫자만큼부터 누적값까지 하면 시간 단축이 가능</p>

`코드그라운드 문제 풀이를 공유하는 것이 문제가 되면 바로 삭제하도록 하겠습니다. 풀이에 대해 이해가 안되는 것은 메일을 통해 전달 부탁드립니다. 그게 제일 빨라요`
