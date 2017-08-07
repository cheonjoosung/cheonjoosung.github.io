---
layout: post
title:  "Codeground 프로그래밍 경진대회"
date:   2017-08-04
categories: algorithm
highlight: false
image: /images/it/algorithm/algo.jpg
tag: algorithm
---

 `CodeGround 의 프로그래밍 경진대회 풀이에 관한 포스팅입니다.` 문제를 제대로 읽지 않으면 원하는 정답을 못 구할 수 있다. 그래서인지 Q&A 가 상당히 많은 문제였다.

<br><b>1) `주어진 문제를 풀이`</b><br>
<p>=) 1) N 사이즈가 주어지면 N, N-1, ... , 1점 까지 점수가 주어진다. 라운드별 동점자는 없지만 종합점수는 동점자가 존재 !!!! 가 가장 큰 힌트</p>
<p>=) 2) 그래서 구하는 것은 우승 가능성이 있는 응시자 수... 4 4 4 가 테스트 일 경우 3, 2, 1 점을 받으므로 7,6,5 가 된다. 4점의 종합점수를 가진 학생이 3명이다. 즉, 이 3명모두 우승 가능성이 있다. 제가 실수한 부분은 동점자를 한 사람으로 취급해서 제대로 된 결과가 안나왔습니다.</p>
<p>=) 3) 마지막 점수에서는 동점자가 나와서 공동우승 되는 경우도 체크가 필요 && 우승 가능성이 있는 응시자수임을 유의해야 함</b></p>

<br><b>2) `이제 정리된 문제를 코딩으로 옮기기`</b><br>
<p>=) 1) 먼저 Arrays.sort 를 통해 주어진 인풋값을 오름차순으로 정렬</b></p>
<p>=) 2) arr[index] + maxScore < arr[lastIndex] + 1 처음 배열의 값과 최대점수를 받았을 때 이전 종합점수에서 가장 높은 합을 받은 사람이 최저점을 받은 것보다 작으면 이 친구는 우승할 가능성이 제로이므로 배제해야 함</p>
<p>=) 3) 만약 크다면, 마지막 인덱스에서 해당 인덱스까지 1점, 2점, 3점 ...... N-1 점까지 다 더한게 arr[index] + maxScore 보다 크면 True 를 리턴, 그렇지 않다면 Flase 를 리턴하고 break. </p>
<p>=) 4) Answer = ArraySize - foundIndex 가 된다 </p>

{% highlight ruby %}

Answer = 0;

int num = sc.nextInt();
int [] arr = new int[num];

for(int i=0 ; i < num ; i++ )
  arr[i] = sc.nextInt();

  Arrays.sort(arr);

  int foundIndex = -1;

  for(int i = 0 ; i < num ; i++) {
    if(arr[i] + num < arr[num-1] + 1) continue;

    boolean result = check(arr, i, num);

    if(result) {
      foundIndex = i;
      break;
    }
  }

Answer = num - foundIndex;

System.out.println("Case #"+(test_case+1));
System.out.println(Answer);

{% endhighlight %}


{% highlight ruby %}

public static boolean check(int [] arr, int index, int size) {
  int temp = 1;
  boolean isCheck = false;

  for(int j = size-1 ; j >= index ; j--) {
    int last = arr[j] + temp++;
    int first = arr[index] + size;

    if( first >= last ) {
      isCheck = true;
    } else {
      isCheck = false;
      break;
    }
  }

  return isCheck;
}

{% endhighlight %}


`코드그라운드 문제 풀이를 공유하는 것이 문제가 되면 바로 삭제하도록 하겠습니다. 풀이에 대해 이해가 안되는 것은 메일을 통해 전달 부탁드립니다. 그게 제일 빨라요`
