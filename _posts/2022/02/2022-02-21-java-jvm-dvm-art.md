---
title: (Java) Java JVM & DVM & ART 개념 및 차이점
tags: [Java]
style: fill
color: dark
description: (Java) Java JVM & DVM & ART 개념 및 차이점
---

## JVM
자바 **바이트 코드를 실행할 수 있는 주체**로 OS 운영체제 위에서 작동을 한다
→ 플랫폼 독립적 (C++은 실행파일까지 만들어버리기에 OS마다 새로 빌드)
![preview](https://user-images.githubusercontent.com/13310269/158185329-e64ff42a-6b4d-4af3-9ad8-7c4313d519bd.png)

**JVM 구조**
![preview](https://user-images.githubusercontent.com/13310269/158185339-540be758-2976-4515-b604-3d26932e191e.png)
1. 클래스 로더
    1. 로딩 - boost strap, extension, application
    2. 링크 - verify, prepare, resolve
    3. 초기화 - 모든 정적 변수 원래 값으로 지정되고 정적 블록 실행?
2. 실행엔진
    1. 인터프리터 - 바이트 코드를 한 행식 읽어서 실행
    2. JIT 컴파일러
        1. 빌드 - 소스코드를 중간언어인 바이트 코드로 변경
        2. 런타임 - 중간언어(바이트코드)를 기계어로 변환. 변환 시간이 오래 걸리기에 인터프리터로 해석하고 실행
    3. GC(Garbar Collector - 참지되지 않는 오브젝트를 수집/제거
3. 메모리(Run Time Data)
    1. PC Register
        1. 수행 명령어의 실행 주소를 가짐
    2. Native Method Stack
        1. Native method 정보 보유
        2. Java외의 언어로 작성된 네이티브 코드들을 위한 stack으로 JNI(Java Native Interface), C/C++코드를 수행하기 위한 stack
    3. Heap Area
        1. 모든 객체, 인스턴스 변수, 배열이 저장되는 곳
        2. 여러 쓰레드에 대한 메모리를 공유하나 스레드 안전하지 않음
    4. Stack Area
        1. 모든 쓰레드에 대해 별도의 런타임 스택이 만들어짐. 각각은 스택 프레임이라고 하는 스택 메모리 존재하고 3가지 하위 항목이 있음
            1. Local - 지역 변수
            2. Operand - 수행 도중 중간 작업 필요한 경우
            3. Frame - 메소드에 해당하는 모든 심볼
    5. Method Area
        1. 클래스 수준의 데이터가 정적 변수를 포함하여 저장
        2. 공유 자원...?

![preview](https://user-images.githubusercontent.com/13310269/158185348-bfbbd1d2-a511-46a4-8d2e-c7aee4856d90.png)

## DVM
안드로이드 운영체제는 PC와 달리 메모리/실행 등의 제약이 많으므로 바이트코드를 그대로 사용이 불가능했다. 바이트 코드를 묶은 .dex파일을 만들고 이에 resourse를 더해 .apk파일을 만든다.
![preview](https://user-images.githubusercontent.com/13310269/158185362-5af8f72d-2baf-4adc-9e7f-f6919c8f423a.png)
안드로이드에서 JVM으로 컴파일된 바이트코드를 실행할 수 없다. java byteCode → dalvik byteCode로 변환하는 dx툴을 android SDk에 포함시켜 두었다.여러 클래스 파일을 묶어서 .dex로 변환하면서 dalvik byte code 로 바꾼다. 압축이 일어나기에 .jar 파일 보다는 크기가 조금 줄어든다.

**JIT 컴파일**
- 자주 사용하는 부분에 대해 미리 컴파일하여 기계어로 해석
- 실행시 인터프리팅을 시작
- 인터프리터 방식의 단점을 보오나하기 위해 도입
- 인터프리터 방식으로 실행하다가 적절한 시점에 바이트코드 전체를 컴파일하여 네이티브 코드로 변경
- 네이티브 코드는 캐시에 저장하므로 빠르게 수행

> **Trace-JIT(인터프리터)?**
Threshold를 초과하면 바이트코드를 기계어로 해석. 특정 횟수 이상 반복되면 컴파일
>

> **Intepreter(인터프리터)?**
한 줄씩 실행
>

**JVM이 아닌 DVM 일까?**
- 라이센스 문제 (오라클...?)
- 모바일 환경에 대한 최적화 : 배터리, 컴퓨팅 파워, 메모리 제약 등


## ART
**ART 주요기능**
- AOT
    - AOT는 ART의 구현된 기능 중 하나로 앱 성능을 개선하는 역할을 한다. 설치 시 ART는 dex2oat도구를 사용하여 앱을 컴파일한다.
    - 이를 통해 dex파일을 입력으로 받아 대상 기기에서 실행할 수 있는 컴파일된 앱을 생성
- GC개선
    - 단일 GC 일시중지가 있는 동시 실행 설계
    - 백그라운드 메모리 사용량/조각화 줄이기 위한 동시 복사
    - 힙 크기와 무관한 GC 일시중지 길이
    - 최근 할당된 단기 객체를 정리하는 특별한 경우 GC 시간이 짧아지는 컬렉터
    - • 동시 가비지 컬렉션이 더욱 적절한 때에 실행되기 때문에 일반적인 사용 사례에서 `[GC_FOR_ALLOC](http://developer.android.com/tools/debugging/debugging-memory.html?hl=ko#LogMessages)` 이벤트가 거의 발생하지 않도록 하는 개선된 가비지 컬렉션 에르고노믹스
- 개발/디버깅 개선
    - 샘플링 프로파일러 지원
    - 디버깅 기능 추가 지원
    - 에외 및 비정상 종료 보고서의 진단 세부정보 개선

## DVM vs ART
![preview](https://user-images.githubusercontent.com/13310269/158185405-ddcc09e9-1179-4bba-98ca-c099d22a2237.png)
DVM으로 실행을 하면 코드를 Trace JIT 방식으로 기계어로 읽는 반면에 AOT는 미리 사전 컴파일을 진행하고 ART를 통해 바로 실행하는 차이가 있다.

**dexopt vs dex2oat**
내부적으로 살펴보면 dex파일을 내부적으로 처리하는 방식이 다르다.

- dvm
    - dexopt(proudec optimized dex file)
    - odex file을 생성
    - DVM → JIT → Machine Code
- art
    - dex2oat(dex transform odex & odex transform to oat)
    - oat file - 실행가능한 파일
    - ART Runtime


## 결론
- 컬렉션 사용법을 익히고 멀티쓰레드 환경에서의 구현 방법을 익혀두자
