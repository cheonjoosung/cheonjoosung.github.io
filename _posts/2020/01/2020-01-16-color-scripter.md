---
title: 블로그에 소스코드 삽입하기 Color Scripter
tags: [꿀팁, 블로그]
style: fill
color: dark
description: 블로그에 소스코드를 삽입할 때 사용할 수 있는 방법
---

## Color Scipter
IDE에서 코드를 작성하고 CTRL C+V를 통해 블로그에 삽입하면 알아보기 힘들어집니다.

1. [Color Scipter 사이트](https://colorscripter.com/)에 들어갑시다.
2. 프로그래밍 언어 선택(C, Java, Swfit 등)하고 스타일 패키지는 서브라임 블랙 추천(어두워서 눈이 덜 아프고 간지남)
3. 실시간 하이라이팅을 체크해야 키워드 및 연산자가 하이라이트 되어 눈에 코드가 쑥 들어옵니다.
4. 코드를 작성하고 `우측 하단 아래에 클립보드에 복사하기` 클릭하고 블로그에 넣으면 끝


## 그냥 코드 VS Color Scipter 

for(int i=0; i < (row+1) ; i++) {
 	for(int j=0 ; j < (col+1) ; j++) {
 		arr[i][j] = sc.nextInt();					
 	}
 }

위의 코드가 Color Scipter를 통해 아래의 코드로 변합니다.

 <div class="colorscripter-code" style="color:#010101;font-family:Consolas, 'Liberation Mono', Menlo, Courier, monospace !important; position:relative !important;overflow:auto"><table class="colorscripter-code-table" style="margin:0;padding:0;border:none;background-color:#fafafa;border-radius:4px;" cellspacing="0" cellpadding="0"><tr><td style="padding:6px;border-right:2px solid #e5e5e5"><div style="margin:0;padding:0;word-break:normal;text-align:right;color:#666;font-family:Consolas, 'Liberation Mono', Menlo, Courier, monospace !important;line-height:130%"><div style="line-height:130%">1</div><div style="line-height:130%">2</div><div style="line-height:130%">3</div><div style="line-height:130%">4</div><div style="line-height:130%">5</div></div></td><td style="padding:6px 0;text-align:left"><div style="margin:0;padding:0;color:#010101;font-family:Consolas, 'Liberation Mono', Menlo, Courier, monospace !important;line-height:130%"><div style="padding:0 6px; white-space:pre; line-height:130%">for(int&nbsp;i=0;&nbsp;i&nbsp;&lt;&nbsp;(row+1)&nbsp;;&nbsp;i++)&nbsp;{</div><div style="padding:0 6px; white-space:pre; line-height:130%">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;for(int&nbsp;j=0&nbsp;;&nbsp;j&nbsp;&lt;&nbsp;(col+1)&nbsp;;&nbsp;j++)&nbsp;{</div><div style="padding:0 6px; white-space:pre; line-height:130%">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;arr[i][j]&nbsp;=&nbsp;sc.nextInt();&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</div><div style="padding:0 6px; white-space:pre; line-height:130%">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}</div><div style="padding:0 6px; white-space:pre; line-height:130%">&nbsp;}</div></div><div style="text-align:right;margin-top:-13px;margin-right:5px;font-size:9px;font-style:italic"><a href="http://colorscripter.com/info#e" target="_blank" style="color:#e5e5e5text-decoration:none">Colored by Color Scripter</a></div></td><td style="vertical-align:bottom;padding:0 2px 4px 0"><a href="http://colorscripter.com/info#e" target="_blank" style="text-decoration:none;color:white"><span style="font-size:9px;word-break:normal;background-color:#e5e5e5;color:white;border-radius:10px;padding:1px">cs</span></a></td></tr></table></div>