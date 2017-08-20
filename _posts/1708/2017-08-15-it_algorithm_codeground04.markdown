---
layout: post
title:  "Codeground 다트 게임"
date:   2017-08-15
categories: algorithm
highlight: false
image: /images/it/algorithm/algo.jpg
tag: algorithm
---

`CodeGround 의 다트 게임 풀이에 관한 포스팅입니다.` 다트는 정중앙 지점에 맞으면 Bull 이라고 하여 50점을 주고 1~20까지의 점수중에 X3, X2를 해주는 지점이 있고 나머지는 X1 로 해서 점수를 주죠.

<br><b>1) `주어진 문제를 풀이`</b><br>
<p>=) 1) 점수가 1 ~ 20 이므로 점수마다 이므로 18도(360/20)의 영역을 가지고 있습니다.</p>
<p>=) 2) 어느 좌표가 주어지면 0,0 에서부터 거리를 계산후 Bull or triple or double or one or out 인지 판단</p>
<p>=) 2-1) 그 후 0,0 과 a,b 를 이었을 때 각도를 체크 tan x = b/a 가 되는 x 도를 구합니다. 1~20 에서 어느 포인트에 꽂혔는지 계산합니다.</p>
<p>=) 2-2) 점수 X 추가 점수 를 계산하면 됩니다.</p>
<p>=) 3) 반지름이 1일 때 호의길이가 1이 되는 세타를 radian 이 됩니다.</p>
<p>=) 3-1) 반원은 2*파이*반지름 의 반이므로 파이 & radin = r * 180 도 이므로 radin = 180 / 파이 인걸 알 수 있습니다.</p>
<p>=) 3-2) degree = x radian X (180/파이) 를 하면 됩니다.</p>

<br><b>2) `이제 정리된 문제를 코딩으로 옮기기`</b><br>
<p>=) 1) 포인트를 입력받으면 어느 지점에 꽂힌지 판단합니다. </p>
<p>=) 2) 해당 포인트의 각도를 구합니다. rad = Math.atan2(y,x) 를 하면 rad 으로 나옵니다.</p>
<p>=) 2-1) 구하는 각도는 radian x (180/Math.PI) 를 곱해야 합니다.</p>
<p>=) 2-2) tan 의 각도는 0 ~ 180 / -180 ~ 0 사이의 값만 나옵니다.</p>

{% highlight ruby %}

static int Answer;
static int [] radious;

public static void main(String args[]) throws Exception	{
	Scanner sc = new Scanner(System.in);

	int T = sc.nextInt();

	for(int test_case = 0; test_case < T; test_case++) {
		Answer = 0;

		radious = new int[5];

		for(int i=0 ; i<5 ; i++)
			radious[i] = sc.nextInt();

		int dartCount = sc.nextInt();

		for(int i=0 ; i< dartCount ; i++) {
			int x = sc.nextInt();
			int y = sc.nextInt();

			double distance = getDistance(x, y);
			double degree = getAngle(x, y);

			int score = calScore(distance, degree);
			Answer += score;
		}

		System.out.println("Case #"+(test_case+1));
		System.out.println(Answer);
	}
	sc.close();
}

public static int calScore(double distance, double degree) {
	int point = getScore(degree);

	if(distance <= radious[0]) {
		return 50;
	} else if( (distance > radious[0] && distance < radious[1] )
			|| ( distance > radious[2] && distance < radious[3] ) ){
		return point;
	} else if(distance >= radious[1] && distance <= radious[2]) {
		return point*3;
	} else if(distance >= radious[3] && distance <= radious[4]) {
		return point*2;
	} else {
		return 0;
	}
}

public static double getAngle(int x, int y) {
	double rad = Math.atan2(y, x);
	double degree = (rad*180) / Math.PI;

	return degree;
}

public static double getDistance(int x, int y) {
	return Math.sqrt(x*x + y*y);
}

public static int getScore(double tan) {
	if( (tan >= 0 && tan < 9) || (tan >= -9 && tan <= 0) ) {
		return 6;
	} else if(tan >= 9 && tan < 27) {
		return 13;
	} else if(tan >= 27 && tan < 45) {
		return 4;
	} else if(tan >= 45 && tan < 63) {
		return 18;
	} else if(tan >= 63 && tan < 81) {
		return 1;
	} else if(tan >= 81 && tan < 99) {
		return 20;
	} else if(tan >= 99 && tan < 117) {
		return 5;
	} else if(tan >= 117 && tan < 135) {
		return 12;
	} else if(tan >= 135 && tan < 153) {
		return 9;
	} else if(tan >= 153 && tan < 171) {
		return 14;
	} else if((tan >= 171 && tan <= 180) || (tan >= -180 && tan < -171) ) {
		return 11;
	} else if(tan >= -171 && tan < -153) {
		return 8;
	} else if(tan >= -153 && tan < -135) {
			eturn 16;
	} else if(tan >= -135 && tan < -117) {
		return 7;
	} else if(tan >= -117 && tan < -99) {
		return 19;
	} else if(tan >= -99 && tan < -81) {
		return 3;
	} else if(tan >= -81 && tan < -63) {
		return 17;
	} else if(tan >= -63 && tan < -45) {
		return 2;
	} else if(tan >= -45 && tan < -27) {
		return 15;
	} else if(tan >= -27 && tan < -9) {
		return 10;
	} else {
		return -1;
	}
}

{% endhighlight %}

`코드그라운드 문제 풀이를 공유하는 것이 문제가 되면 바로 삭제하도록 하겠습니다. 풀이에 대해 이해가 안되는 것은 메일을 통해 전달 부탁드립니다. 그게 제일 빨라요`
