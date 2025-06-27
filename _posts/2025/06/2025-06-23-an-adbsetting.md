---
title: 맥북에서 ADB 설정하기
tags: [ Android ]
style: fill
color: dark
description: 맥북에서 Android 디바이스와 통신하기 위한 ADB(Android Debug Bridge)를 Android Studio 설치 유무에 따라 설정하는 두 가지 방법을 간단하고 명확하게 정리합니다.
---

## ✨ 개요

ADB(Android Debug Bridge)는 Android 디바이스와 로컬 개발 환경 간의 연결을 담당하는 핵심 도구입니다. 
맥북에서 ADB를 사용하려면 Android Studio의 유무에 따라 설치 및 설정 방식이 다릅니다.

---

## 1. ✅ Android Studio 설치 필요: 기본 ADB 경로 활용

1. 터미널 실행
2. 아래 명령어 두 줄 동시 입력
```gradle
echo 'export ANDROID_HOME=/Users/$USER/Library/Android/sdk' >> ~/.zshrc
echo 'export PATH="$PATH:$ANDROID_HOME/tools:$ANDROID_HOME/platform-tools"' >> ~/.zshrc
```
3. 명령어 적용
```gradle
source ~/.zshrc
```
4. 적용 여부 확인
```gradle
adb devices

// 아래와 같이 나오면 정상
* daemon not running; starting now at tcp:5037
* daemon started successfully
List of devices attached
```