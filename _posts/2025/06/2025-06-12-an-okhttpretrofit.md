---
title: OkHttp vs Retrofit2 - 특징, 공통점, 차이점 비교 정리
tags: [ Android ]
style: fill
color: dark
description: OkHttp와 Retrofit2는 Android에서 가장 많이 사용되는 HTTP 통신 라이브러리입니다. 이 두 라이브러리의 특징과 공통점, 그리고 차이점을 간결하게 비교 정리합니다.
---

## ✨ 개요

Android 앱에서 서버와의 통신을 구현할 때 가장 널리 쓰이는 라이브러리는 OkHttp와 Retrofit입니다.

둘 다 강력하고 효율적인 HTTP 클라이언트지만, 목적과 사용 방식에서 분명한 차이가 있습니다.

---

## 1. 🧪 주요 특징

| 항목      | OkHttp                                      | Retrofit                                      |
|-----------|----------------------------------------------|-----------------------------------------------|
| 목적      | HTTP 클라이언트 (낮은 수준)                      | REST API 클라이언트 (고수준 wrapper)              |
| 기본 기능 | 요청 생성, 연결, 응답 처리                      | OkHttp 기반으로 인터페이스 기반 API 자동 처리       |
| 확장성    | Interceptor, Cache, Auth 등 직접 구현 가능     | Gson, Moshi, Coroutine 등 통합 쉬움              |
| 사용 난이도| 중급 이상 (직접 구성 필요)                       | 쉬움 (Annotation 기반 자동화)                     |

---

## 2. ✅ 공통점

- 모두 Square에서 개발한 라이브러리
- OkHttpClient 인스턴스를 Retrofit 내부에서 사용 가능
- 비동기/동기 요청 처리 가능
- Interceptor, Logging, Timeout 등 공통 기능 활용 가능

---

## 3. 결론

- 단순한 HTTP 요청/응답 처리에는 OkHttp만으로 충분할 수 있음
- API 레이어를 정형화하고 유지보수를 쉽게 하려면 Retrofit이 더 적합
- 복잡한 앱에서는 Retrofit + OkHttp 조합으로 사용하는 것이 일반적입니다.

