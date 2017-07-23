---
layout: post
title:  "Codeground 방속의거울"
date:   2017-07-19
categories: algorithm
highlight: false
image: /images/it/algorithm/algo.jpg
tag: algorithm
---

 `CodeGround 의 방속의 거울에 관한 포스팅입니다.` 코드 그라운드 문제의 난이도는 Basic 인 것 같아요. 문제를 읽고 어떻게 풀어야 될지 딱 나오는 쉬운 문제였던 것 같아요. 조심해야할 점은 다른 것은 input data 0 1 2 식으로 한칸 띄어서 주는데 여기서는 012 붙여서 주기에 scanner.nextLine() 을 통해서 받은 후 한글자씩 받아서 데이터를 넣어줘야 합니다.

<br><b>1) `주어진 문제를 풀이`</b><br>
<p>=) 1) 먼저 빛의 시작점은 0 , n-1 입니다. </p>
<p>=) 2) 1 번은 / 모양의 거울 , 2번은 \ 모양의 거울입니다.</p>
<p>=) 2-1) 1번 거울 일 때, 빛 좌->우 : 위로, 상->하 : 좌측으로, 우->좌 : 아래로, 하->상 : 우측으로</b></p>
<p>=) 2-2) 2번 거울 일 때, 빛 좌->우 : 아래로, 상->하 : 우측으로, 우->좌 : 위로, 하->상 : 좌측으로</p>
<p>=) 2-3) 거울이 없을 때는 빛의 진행방향은 온 그대로 진행</p>
<p>=) 2-4) 어떤 구역을 지나가면서 빛이 방을 나갈 때 까지 총 지나는 거울의 수를 체크(여러개가 한 거울을 반사할 때 한번만 체크)</p>

<br><b>2) `이제 정리된 문제를 코딩으로 옮기기`</b><br>
<p>=) 1) temp(위에서 a) = number % b 라고 놓습니다.</p>
<p>=) 2) number /= b 와 number % b 를 반복하면서 나머지를 temp 와 비교를 합니다.</p>
<p>=) 3) 모두 통과하면 해당 b 는 균일수가 됩니다. 3자리 이상에서만요. b의 범위는 2-3) 참고 </p>
<p>=) 4) 만약에 2자리 수라면 위의 로직을 안타고 하위 로직을 타야 됩니다. 2-2) b = N/a -1 을 찾으면 됩니다. </p>
<p>=) 5) N = a * X 의 형태이므로 N은 a를 약수로 가집니다. 즉, N % a == 0 이 됩니다. 그런 a 를 찾아서 계산을 하면 끝납니다. </p>

{% highlight ruby %}

Answer = 0;
count = 0;

n = sc.nextInt();

int [][] arr = new int[n][n];
int [][] arrMirror = new int[n][n];

int currentX = 0;
int currentY = n-1;
int direction = 1;

sc.nextLine();

for(int j=n-1 ; j>=0 ; j--) {
  String s = sc.nextLine();

  for(int i=0 ; i<n ; i++) {
    arr[i][j] = s.charAt(i) - '0';
    arrMirror[i][j] = 0;
  }
}

while(true) {

if(currentX < 0 || currentX >= n || currentY < 0 || currentY >= n) {
  break;
}

if(arr[currentX][currentY] == 1){
  if(arrMirror[currentX][currentY] == 0) {
    count++;
    arrMirror[currentX][currentY] = 1;
  }

  switch(direction) {
    case 0:
    currentX++;
    direction = 1;
    break;

    case 1:
    currentY++;
    direction = 0;
    break;

    case 2:
    currentX--;
    direction = 3;
    break;

    case 3:
    currentY--;
    direction = 2;
    break;
    }

} else if(arr[currentX][currentY] == 2) {
  if(arrMirror[currentX][currentY] == 0) {
    count++;
    arrMirror[currentX][currentY] = 1;
  }

  switch(direction) {
    case 0:
    currentX--;
    direction = 3;
    break;

    case 1:
    currentY--;
    direction = 2;
    break;

    case 2:
    currentX++;
    direction = 1;
    break;

    case 3:
    currentY++;
    direction = 0;
    break;
    }

  } else {
    switch(direction) {
      case 0:
      currentY++;
      break;

      case 1:
      currentX++;
      break;

      case 2:
      currentY--;
      break;

      case 3:
      currentX--;
      break;
    }
  }
}


Answer = count;

System.out.println("Case #"+(test_case+1));
System.out.println(Answer);

{% endhighlight %}

<br><b>3) `다른 정리`</b><br>
<p>=) 1) 재귀를 이용하면 더 쉽게 구현을 할 수 있으나, 스택오버가 나면서 뻗어버립니다.</p>
<p>=) 2) 거울수를 체크하기 위해 arrWater 이중배열을 사용했지만 다른방법을 사용해도 될 것 같습니다.</p>

`코드그라운드 문제 풀이를 공유하는 것이 문제가 되면 바로 삭제하도록 하겠습니다. 풀이에 대해 이해가 안되는 것은 메일을 통해 전달 부탁드립니다. 그게 제일 빨라요`
