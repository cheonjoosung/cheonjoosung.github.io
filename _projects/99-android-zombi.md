---
name: Running Dead!! 안드로이드 좀비게임
tools: [Java, Android]
image: https://i.imgur.com/TOs9HTh.png
description: 안드로이드의 GPS 기능을 활용한 좀비 게임
---

## Runnig Dead
- 워킹 데드를 착안하여 만든 GPS를 이용한 안드로이드 좀비 게임
- 게임을 목표로 했지만 운동용에 가까움
- 지도 상에 출발지/목적지를 설정한 후 지도에 생성되는 좀비를 피하며 목적지까지 이동하는 단순한 시나리오

## Period
- 대학교 프로그래밍 언어론 수업 3인 팀 프로젝트
- PP(Pair Programming)방식으로 개발했고 총 9일 소요(개발 7일 + 테스트 2일)

## Dev.
**개발 환경**
- Eclipse
- Android 4.x 삼성갤럭시 s2
- Google map v2

**구현된 기능**
- **[본인 역할]**구글 지도 API를 활용하여 맵 화면 구현, 마커 기능 구현, 좀비 이동 속도 테스트
- 좀비 클래스 구현 : 난이도(매우 쉬움, 쉬움, 보통, 어려움) 구현 + 속도(노인, 어린이, 성인) 구현
- Thread를 통해 지도에 랜덤하게 좀비 생성 및 사용자 추적
- 게임 실패/성공 시 이벤트 발생

**스크린샷**
![preview](https://i.imgur.com/TOs9HTh.png)


## Result & Learned
- 완성 후 팀 프로젝트 공유 시간에 좀비 게임 영상으로 많은 박수와 환호 그리고 재미까지
- 좋은 성적을 얻으면서 모바일에 흥미를 느꼈고 **모바일 개발자에 대한 꿈을 가지게 된 계기**

<!-- //사진 슬라이드 활용시
{% capture carousel_images %}
https://bit.ly/2BBbVhc
https://bit.ly/2DOtxXB
{% endcapture %}
{% include elements/carousel.html %} 

{% include elements/figure.html image="https://bit.ly/2N69TKO" caption="The Ocean" %}
//사진에 설명 넣을 때

</p> -->
