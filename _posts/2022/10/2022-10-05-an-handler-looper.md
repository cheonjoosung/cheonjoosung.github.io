---
title: Android Looper & Handler | 안드로이드 루퍼 & 핸들러
tags: [Android]
style: fill
color: dark
description: Android Looper & Handler | 안드로이드 루퍼 & 핸들러
---

## 안드로이드 Main Thread

안드로이드에서 Looper & Handler 가 필요한 이유는 Thread 간 통신 때문이다.
기본적으로 안드로이드는 싱글 쓰레드(메인 쓰레드)를 모델을 사용하고 오직 이를 이용하여 UI를 그린다.
또한, 다른 쓰레드가 메인 쓰레드를 블럭하면 안 된다.


#### 1. Looper & Handler 가 필요한 이유
안드로이드를 이용하여 앱을 만들때 쓰레드 하나로만 쓰는 경우가 거의 없다.
네트워크 및 IO 작업을 하는 것은 메인 쓰레드가 아닌 다른 쓰레드를 이용하여 처리해야 한다.
이런 작업을 통해 나온 데이터를 메인 쓰레드로 전달하고 메인 쓰레드에서 화면에 그린다.

=> 다른 쓰레드에서 메인 쓰레드로 전달해주는 시스템이 Looper & Handler 이다


#### 2. Looper & Handler 동작 원리
![preview](https://user-images.githubusercontent.com/13310269/194057890-a6d38418-7110-4cb3-b80d-ec9ecb0c9226.png)

Looper 역할 (메인 쓰레드는 기본적으로 가짐)
1. 주기적으로 메시지 큐를 확인
2. Message or Runnable 객체가 존재하면 handler 로 보내 처리

Handler 역할
1. Looper 가 보낸 Message or Runnable 객체를 처리 
2. 다른 쓰레드에서 요청한 정보를 메인쓰레드의 메시지 큐에 담는다

Message 는 객체의 정보를 담고 있고 Runnable 은 method 나 실행한 코드를 담는다