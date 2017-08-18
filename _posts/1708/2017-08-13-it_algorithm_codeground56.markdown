---
layout: post
title:  "Codeground 극단적인 수"
date:   2017-08-13
categories: algorithm
highlight: false
image: /images/it/algorithm/algo.jpg
tag: algorithm
---

`CodeGround 의 극단적인 수 풀이에 관한 포스팅입니다.` 4 와 7 을 이용하여 몇 번째 있는 숫자를 출력하는 문제입니다.

<br><b>1) `주어진 문제를 풀이`</b><br>
<p>=) 1) 4 -> 7 -> 44 -> 47 -> 74 -> 77 -> 444 ->  ..... 의 식으로 진행이 되겠죠?</p>
<p>=) 2) 1 -> 2 -> 3  -> 4  -> 5  -> 6  -> 7   -> 순서의 인덱스라고 생각하면</p>
<p>=) 3) 시작이 1 이면 4에서 시작, 1이면 7에서 시작, 3 [44] = 2 * 1 [4] + 1 => 4가 추가, 4 [47] = 2 * 1 [4] + 2 => 7이 추가 의 형태이다.</p>
<p>=) 4) 시작이 2 이면 7에서 시작, 1이면 7에서 시작, 5 [74] = 2 * 2 [7] + 1 => 4가 추가, 4 [77] = 2 * 2 [7] + 2 => 7이 추가의 형태이다.</p>
<p>=) 5) 반대로 생각하면 (7-1) / 2 = 3 이므로 44 에서 4가 붙었다는 것을 알수 있다. (8-1) / 2 = 3.5 -> 3 이므로 44 에서 7이 붙었다는 것을 알 수 있다.</p>

<br><b>2) `이제 정리된 문제를 코딩으로 옮기기`</b><br>
<p>=) 1) 입력된 값이 0 이 될 때까지 (temp-1) / 2 의 값을 list 에 저장</p>
<p>=) 2) list 을 오름차순으로 정렬</p>
<p>=) 2-1) 입력된 값을 꺼내서 (시작점 * 2) + 1 과 같으면 4를 추가</p>
<p>=) 2-2) 입력된 값을 꺼내서 (시작점 * 2) + 2 과 같으면 7를 추가</p>

{% highlight ruby %}

Answer = 0;

int exNum = sc.nextInt();

ArrayList<Integer> list = new ArrayList<>();

int temp = exNum;

while(temp != 0) {
  list.add(temp);
  temp = (temp-1) / 2;
}

Collections.sort(list);

String result = "";

temp = 0;

for(int k : list) {
  if(k == temp*2 +1) {
    result += "4";
    temp = temp*2 + 1;
  } else {
    result += "7";
    temp = temp*2 + 2;
  }
}

{% endhighlight %}

`코드그라운드 문제 풀이를 공유하는 것이 문제가 되면 바로 삭제하도록 하겠습니다. 풀이에 대해 이해가 안되는 것은 메일을 통해 전달 부탁드립니다. 그게 제일 빨라요`
