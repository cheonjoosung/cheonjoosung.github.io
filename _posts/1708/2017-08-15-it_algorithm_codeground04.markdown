---
layout: post
title:  "Codeground 다트 게임"
date:   2017-08-15
categories: algorithm
highlight: false
image: /images/it/algorithm/algo.jpg
tag: algorithm
---

`CodeGround 의 다트 게임 풀이에 관한 포스팅입니다.` 학점이 주어질 때 들을 수 있는 최대의 학점을 구하는 문제이다.

<br><b>1) `주어진 문제를 풀이`</b><br>
<p>=) 1) 제일 중요한 것은 주어진 최대 학점이 있다면 x, y , z ...... etc 을 더해서 같거나 작은 최대의 값을 구하면 된다.</p>
<p>=) 2) 열리는 과목의 학점은 주어진 최대 학점보다 같거나 작은 수업이 존재할 수 있다.</p>

<br><b>2) `이제 정리된 문제를 코딩으로 옮기기`</b><br>
<p>=) 1) Arrays.sort 를 이용해서 주어진 배열을 오름차순으로 정리 </p>
<p>=) 2) sum = arr[lastIndex] 을 할당하고 MAX 랑 같은지 비교</p>
<p>=) 2-1) lastIndex 부터 0 번째까지 차례대로 더하면서 MAX 랑 같으면 반복문 종료. Answer 출력</p>
<p>=) 2-2) lastIndex 부터 0 번째까지 차례대로 더하면서 MAX 보다 작으면 sum += arr[index] 더한다.</p>
<p>=) 2-3) lastIndex 부터 0 번째까지 차례대로 더하면서 MAX 보다 크면 break 시키고 Answer 할당.</p>
<p>=) 2-4) Answer = Math.max(Answer, sum) 을 저장한다. </p>

{% highlight ruby %}

Answer = 0;

int num = sc.nextInt();
int sum = sc.nextInt();

int [] arr = new int[num];

for(int i=0 ; i<num ; i++){
  arr[i] = sc.nextInt();
}

Arrays.sort(arr);

for(int i = num-1 ; i >= 0 ; i--) {
  int max = 0;
  max += arr[i];

  if(max == sum) {
    Answer = max;
    break;
  }

  for(int j = i-1 ; j >= 0 ; j--) {
    if(max + arr[j] == sum) {
      Answer = max+arr[j];
      break;
    } else if(max + arr[j] > sum) {
      continue;
      } else {
        max += arr[j];
      }
    }

    if(Answer == sum) break;
    else {
      Answer = Math.max(Answer, max);
    }
}

System.out.println("Case #"+(test_case+1));
System.out.println(Answer);

{% endhighlight %}

`코드그라운드 문제 풀이를 공유하는 것이 문제가 되면 바로 삭제하도록 하겠습니다. 풀이에 대해 이해가 안되는 것은 메일을 통해 전달 부탁드립니다. 그게 제일 빨라요`
