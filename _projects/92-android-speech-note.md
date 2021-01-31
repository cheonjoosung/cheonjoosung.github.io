---
name: 코틀린 STT(Speech To Text) 기능을 활용한 Speech Note
tools: [Kotlin, Android]
image: https://play-lh.googleusercontent.com/u0I2iQGWyw0yyZGDRcB7frx_O-utiXxWhvRlFyio2vRatcs8ID9YRFsOi1s3YP92QMk=s360-rw
description: 구글 SpeechRecognizer를 활용하여 음성을 녹음하고 실시간으로 텍스트로 변환하는 모바일 서비스
---

## Speech Note
- 음성을 인식한 후 실시간으로 텍스트로 변환하여 보여주고 녹음 기능을 가진 모바일 앱
- [구글 플레이스토어 주소](https://play.google.com/store/apps/details?id=kr.co.js.meetstt)

## Period
- 1인 프로젝트
- 2021.01.01 ~ 2021.01.31

## Dev.
**개발 환경**
- Android Studio 4.1.1(target 30 min 16)
- LG v40
- Google Speech Recognier
- Kotlin 1.4.21
- UI 라이브러리(Lottie, angads/toggle, afollestad/dialog)

**구현된 기능**
- 구글 인식 라이브러리를 활용하여 실시간으로 텍스트로 변환하고 화면에 보여줌
- 음성 녹음 기능
- 파일 정보/삭제/수정 및 공유 가능
- 텍스트 파일에 타임라인 정보 추가

**예정 기능**
- 클라우드를 활용한 화자분리 기능

**스크린샷(슬라이드)**
{% capture carousel_images %}
https://play-lh.googleusercontent.com/x8RonpXh5qinHwUY0YefmPKJJ1uxfcUfvmTIRW3xE17Aa4RR_0t5L1xVPVt_70U-KP4=w2292-h2084-rw
https://play-lh.googleusercontent.com/E3OpiEaS7N6XojE9Y9tUZQ9BtZ4O9OASuTssIGLOXnQm518wP7mNkINK8gzSKKG0UQ=w2292-h2084-rw
https://play-lh.googleusercontent.com/jRKulbetIUxRhvQ57xxzfmgMsjnz8RJsi5yyVIwf9csn1mbGhxLd0HUDOdB-eOrnig=w2292-h2084-rw
https://play-lh.googleusercontent.com/yHeWnTwrZk4yiUOyo9nKU9WgNfMfweovnsxydqo0JldfsOj888UKR27GblV1UiWtMg=w2292-h2084-rw
https://play-lh.googleusercontent.com/1n85HfQrAmLHg7dBymshJjQhMavSIwveGztgHqg5_h9nS9CYm4fJZkMBVQiN3ZJ5Q3IV=w2292-h2084-rw
https://play-lh.googleusercontent.com/X7nkzr6Umu6yDNAeXB3hj_-e7HZLyi4GSCHm3Pom5cRZ67PgNgkmXC-eCfIeIMnXDv0_=w2292-h2084-rw
{% endcapture %}
{% include elements/carousel.html %}


## Result & Learned
- 코틀린으로 개발을 해보니 자바 기반의 안드로이드보다 코드 생산성이 늘어나고 널타입 안정성이 늘어나서 좋음
- Lottie 라이브러리를 적용하는 것이 쉬웠고 앱에 생동감을 불어넣어줄 수 있는 점이 좋았음
- 코루틴을 처음 사용해보았는데 자바의 쓰레드나 비동기처리보다 훨씬 쉬워서 놀랐음.... 킹틀린 ㅇㅈ?