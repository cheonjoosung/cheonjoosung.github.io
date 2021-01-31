---
name: realm + rxAndroid 을 이용한 안드로이드 오늘 할일 앱
tools: [Java, Android, Friebase, Realm]
image: https://i.imgur.com/1wP5frH.png
description: 오답노트의 온라인 버전이고 모바일 또는 웹을 통해 다양한 서비스를 이용할 수 있습니다.
---

## 오늘 할일
- 오늘의 해야할 일을 포스트 잇처럼 메모로 적고 관리하는 앱
- [구글 플레이스토어 주소](https://play.google.com/store/apps/details?id=com.js.memo&hl=ko)


## Period
- 1인 프로젝트
- 2019.07 ~ 2019.09

## Dev.
**개발 환경**
- Android Studio 3.4.2(target 28 min 19)
- LG v40
- Firebase 5.3.0(Crashlytics, Ads)

**구현된 기능**
- 메모에 대한 관리를 realm db 구성 + rx개념을 적용하여 메모 모델-뷰를 연결하여 메모 DB가 변경되는 경우 뷰가 자동으로 갱신되도록 구현
- 메모 추가, 삭제, 수정
- 메모 설정 : 글씨 크기, 정렬
- 리뷰 남기기 & 공유 기능
- 위젯 추가
- 구글 스토어 출시

**흐름도**
![preview](https://i.imgur.com/NqtYKvF.png)

**스크린샷(슬라이드)**
![preview](https://i.imgur.com/1wP5frH.png)


## Result & Learned
- RxAndroid, realm 등의 모바일과 관련된 트렌드 기술을 활용하여 서비스를 만들면서 이해도를 높임
- 구글 스토어에 직접 앱을 출시하면서 과정을 익혔고 자동 배포, 디자인 등 다양하게 고려해야 할 것을 알게 되었음
- 다른 앱을 더 만들어 보고 싶은 욕심이 생겼다..