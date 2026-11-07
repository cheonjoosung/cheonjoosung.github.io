---
title: (Android/OkHttp) OkHttp SSL Pinning 구현
tags: [ Android, OkHttp ]
style: fill
color: dark
description: OkHttp CertificatePinner로 SSL Pinning을 구현하는 방법과 핀 추출, 백업 핀, 인증서 갱신 대응, Network Security Config 방식을 정리합니다.
---

---

## 1. SSL Pinning이란?

기본 HTTPS는 **기기가 신뢰하는 모든 CA**가 발급한 인증서를 받아들입니다.
공격자가 사용자 기기에 악성 CA를 설치하거나 CA가 침해되면 **중간자 공격(MITM)** 이 가능해집니다.

SSL Pinning은 "이 서버는 **이 공개키(인증서)일 때만** 신뢰한다"고 앱에 고정해 이를 막습니다.

```
일반 HTTPS:  기기가 신뢰하는 CA가 서명했으면 통과
Pinning:     서버 인증서의 핀이 앱에 심어둔 값과 같아야 통과
```

---

## 2. 핀 값 추출

OkHttp는 인증서의 **공개키 SHA-256 해시**를 핀으로 사용합니다.

```bash
openssl s_client -servername api.example.com -connect api.example.com:443 </dev/null 2>/dev/null \
 | openssl x509 -pubkey -noout \
 | openssl pkey -pubin -outform der \
 | openssl dgst -sha256 -binary \
 | openssl enc -base64
# 출력 예: k7Hn...Qw=  →  "sha256/k7Hn...Qw="
```

> 인증서 전체가 아니라 **공개키**를 고정하므로, 같은 키로 인증서를 갱신(재발급)해도 핀이 유지됩니다.

---

## 3. CertificatePinner 적용

```kotlin
val pinner = CertificatePinner.Builder()
    .add("api.example.com", "sha256/k7Hn...현재서버키핀...=")
    .add("api.example.com", "sha256/Zq9P...백업키핀...=")      // 백업 핀
    .build()

val client = OkHttpClient.Builder()
    .certificatePinner(pinner)
    .build()
```

핀이 맞지 않으면 `SSLPeerUnverifiedException`이 발생하며 요청이 차단됩니다.

```kotlin
try {
    api.getUser()
} catch (e: SSLPeerUnverifiedException) {
    // 인증서 불일치 — 공격 가능성 또는 핀 만료. 사용자에게 안전하지 않은 연결을 알림
}
```

---

## 4. 가장 중요한 주의 — 백업 핀과 갱신 계획

핀을 고정하면 **서버 인증서의 키가 바뀌는 순간 앱이 서버에 접속하지 못합니다.** 앱 업데이트 전에는 복구할 수 없습니다.

```
❌ 현재 서버 키 하나만 핀 → 키 교체 시 모든 사용자의 앱이 장애
✅ 현재 키 + 백업 키(미리 준비한 다음 키)를 함께 핀
✅ 또는 서버 리프 인증서 대신 중간(Intermediate) CA 키를 핀 (갱신 영향 감소)
```

| 대상 | 장점 | 단점 |
|------|------|------|
| 리프(서버) 인증서 키 | 가장 엄격 | 갱신 시 키가 바뀌면 장애 위험 |
| 중간 CA 키 | 서버 인증서 갱신에 안전 | 해당 CA가 발급한 다른 인증서도 허용 |

- 핀 변경은 **앱 업데이트를 먼저 배포한 뒤** 서버 키를 교체하세요
- 만료 시점을 달력에 관리하고, 핀 실패를 **원격 로깅**해 장애를 조기에 발견하세요

---

## 5. Network Security Config 방식 (선언형)

코드 없이 XML로도 설정할 수 있습니다. OkHttp뿐 아니라 시스템 전체 HTTPS에 적용됩니다.

```xml
<!-- res/xml/network_security_config.xml -->
<network-security-config>
    <domain-config>
        <domain includeSubdomains="true">api.example.com</domain>
        <pin-set expiration="2027-06-01">
            <pin digest="SHA-256">k7Hn...현재서버키핀...=</pin>
            <pin digest="SHA-256">Zq9P...백업키핀...=</pin>
        </pin-set>
    </domain-config>
</network-security-config>
```

```xml
<application android:networkSecurityConfig="@xml/network_security_config" ... />
```

`expiration`을 지정하면 그 날짜 이후에는 핀 검증을 **건너뛰어** 갱신 누락 시의 전면 장애를 완화합니다.

---

## 6. 테스트와 디버그 빌드

- 디버그에서 프록시(Charles, mitmproxy)로 트래픽을 보려면 핀이 막습니다. **디버그 빌드에서만 핀을 끄도록** 분리하세요

```kotlin
val client = OkHttpClient.Builder().apply {
    if (!BuildConfig.DEBUG) certificatePinner(pinner)
}.build()
```

- 단, 릴리스에서 핀이 켜진 상태로 반드시 한 번 검증하세요 (잘못된 핀은 릴리스에서만 장애로 드러납니다)

---

## 7. SSL Pinning의 한계

- 루팅/후킹 도구(Frida 등)로 **우회될 수 있습니다.** 완벽한 방어가 아니라 공격 난이도를 높이는 계층입니다
- 서버 인증서 관리 부담이 생기므로, 민감 정보(금융, 개인정보)를 다루는 서비스에 적합합니다
- 앱 무결성 검증, 루팅 탐지, 토큰 보호 등 **다른 보안과 함께** 사용하세요

---

## 8. 정리

- Pinning은 신뢰 대상을 **지정한 서버 키로 한정**해 MITM을 방어
- OkHttp는 `CertificatePinner`에 공개키 SHA-256(`sha256/...`)을 등록
- **백업 핀 필수**, 갱신 순서는 앱 업데이트 먼저 → 서버 키 교체
- 선언형으로는 Network Security Config의 `pin-set`(`expiration` 활용)
- 디버그 프록시 분리, 우회 가능성을 인지하고 다른 보안과 병행
