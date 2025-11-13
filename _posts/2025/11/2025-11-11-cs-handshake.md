---
title: (CS/컴퓨터공학) TCP 3-Way Handshake 완벽 가이드
tags: [CS]
style: fill
color: dark
description: (CS/컴퓨터공학) TCP 3-Way Handshake 완벽 가이드 — SYN/SYN-ACK/ACK, 옵션(MSS/WS/SACK), 장애·보안·캡처까지
---

## ✨ 개요


TCP 연결은 **3단계 핸드셰이크(3-Way Handshake)** 로 시작합니다.  
클라이언트와 서버가 초기 **시퀀스 번호(ISN)**, **윈도 크기**, **옵션**을 합의하며 신뢰 가능한 바이트 스트림을 열죠.


---

## 1. 한 장 요약 

- 단계: **SYN → SYN-ACK → ACK**
- 목적: 초기 시퀀스 번호 동기화, 전송/흐름 제어 파라미터 합의
- 실패 포인트: 방화벽 차단, SYN-ACK 미응답(리버스 경로/보안 장비), SYN backlog 포화, RST
- 관찰: `tcpdump`/Wireshark로 flags·옵션 확인(MSS, 윈도 스케일, SACK, TS 등)

---

## 2 시퀀스: ASCII 시퀀스 다이어그램

```text
Client Server
| --- SYN, seq=x, MSS, WS, SACK, TS ---> |
| <--- SYN-ACK, seq=y, ack=x+1 ---------- |
| --- ACK, seq=x+1, ack=y+1 ------------> |
|============= ESTABLISHED ==============>
```

- **SYN**: 연결 요청 + 클라이언트 ISN(**x**) 제안, **옵션** 제시
- **SYN-ACK**: 서버 ISN(**y**) 제안 + 클라 ISN 확인(ack=x+1)
- **ACK**: 서버 ISN 확인(ack=y+1) → **ESTABLISHED**

> **ISN(Initial Sequence Number)** 는 랜덤에 가까운 값(보안·혼잡 회피).  
> 각 바이트는 시퀀스로 계산되며, SYN 자체가 **1바이트 소비**로 간주되어 `+1`이 됩니다.

---

## 3 핸드셰이크에서 교환되는 핵심 옵션

- **MSS (Maximum Segment Size)**: 한 세그먼트의 TCP 페이로드 최대 크기(보통 MTU 1500 → MSS 1460).
- **Window Scale (WS)**: 윈도 크기를 `2^n` 만큼 확장(대역×지연(BDP) 큰 환경 필수).
- **SACK Permitted**: 선택적 재전송 허용(손실 구간만 재전송).
- **Timestamps (TS)**: RTT 측정, PAWS 보호.

> 옵션은 **SYN 구간에서만 협상**됩니다. 이후 패킷엔 “합의된 설정”이 적용됩니다.

---

## 4. 왜 3단계인가? (신뢰·반사 공격 방지)

- 단순 2-Way(요청/응답)라면 **IP 스푸핑**으로 반사/자원 고갈이 쉬워집니다.
- 3단계는 양측이 서로의 **수신 가능성**을 확인(ACK 왕복)하므로 **반사/유령 연결**을 줄입니다.

---

## 5. 실패·이상 징후와 원인

| 증상 | 의미/원인 |
|---|---|
| SYN 보냄 → **응답 無(타임아웃)** | 방화벽 드롭, 라우팅/리턴패스 문제, 서버 다운 |
| SYN → **RST** 즉시 수신 | 포트 미개방, 서비스 미동작, 패킷 필터 차단 |
| SYN/SYN-ACK **여러 번 재전송** | 손실, 혼잡, MTU/MSS 불일치(단편화/ICMP 차단) |
| ESTABLISHED 전 **SYN backlog 포화** | SYN flood, 동시 접속 과다 → accept 지연/드롭 |
| 핸드셰이크 후 초기 데이터에서 지연 급증 | 혼잡/버퍼블로트, 중간 장비의 DPI/SSL 인스펙션 |

---

## 6. 보안: SYN Flood와 SYN Cookies

- **SYN Flood**: 대량의 SYN으로 서버의 **SYN backlog** 고갈 → 정당한 연결 수립 실패.
- **완화**
  + **SYN Cookies**: 백로그 포화 시, 서버가 상태 저장 없이 ISN에 쿠키를 인코딩하여 **ACK가 올 때만 상태 생성**
  + 방화벽 레이트리밋, 프록시/반사 방어, 임계값 조정(backlog, `tcp_max_syn_backlog`)

---

## 7. MTU/MSS와 초기 지연

- 경로 MTU보다 큰 세그먼트는 단편화 또는 드롭/ICMP 필요 → 초기 재전송/지연.
- 서버/클라이언트에서 MSS 클램핑(예: VPN/터널 환경)으로 문제를 줄일 수 있습니다.

---

## 8. 애플리케이션 관점 팁

- 초기 N바이트는 Nagle/Delayed ACK 상호작용으로 지연될 수 있음 → 소량 지연 민감 트래픽은 TCP_NODELAY 고려.
- 첫 요청 크기가 MTU를 초과하지 않게 설계(핸드셰이크 직후 재전송 방지).
- 장거리/고BDP에서는 윈도 스케일/TCP 옵션이 성능에 결정적.

---

## 9. 현대 대안: QUIC(UDP 위 전송)

- QUIC(HTTP/3) 는 TLS 1.3을 내장하고, 0-RTT/1-RTT로 연결 수립을 단축.
- 스트림 단위 HoL 차단 없음 → 손실 환경에서 초기 지연/재전송 부담이 작음.