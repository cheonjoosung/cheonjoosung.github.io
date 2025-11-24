---
title: (CS/컴퓨터공학) REST API 디자인 기본 규칙
tags: [CS]
style: fill
color: dark
description: (CS/컴퓨터공학)REST API 디자인 기본 규칙 — 리소스/HTTP 메서드/상태코드/버저닝/에러/보안/캐싱
---

## ✨ 개요

REST는 “HTTP를 **일관된 규약**으로 잘 쓰자”가 핵심입니다.  
아래 체크리스트를 따르면 클라이언트(Android/iOS/Web/서버) 모두가 **예측 가능**하고

---

## 1. 리소스 설계 & 경로 규칙

- **명사, 복수형, 소문자/하이픈** 사용
  + `/users`, `/users/{userId}`, `/users/{id}/orders`
- **행위는 메서드로 표현**, 경로에 동사 금지 (`/approve` 대신 `PATCH /orders/{id}`)
- **중첩은 2~3단계 이내**: 과한 중첩은 필터/관계 링크로 해결
- **컬렉션/개체 구분**: `/orders`(컬렉션), `/orders/{id}`(개체)

---

## 2 HTTP 메서드 의미(정리표)

| 메서드 | 의미 | 멱등성 | 예시 |
|---|---|---|---|
| `GET` | 조회 | ✅ | `GET /users?role=admin` |
| `POST` | 생성/행위 | ❌(보통) | `POST /orders` |
| `PUT` | 전체 교체 | ✅ | `PUT /profiles/{id}` |
| `PATCH` | 부분 수정 | ❌/준멱등 | `PATCH /profiles/{id}` |
| `DELETE` | 삭제 | ✅ | `DELETE /orders/{id}` |

> **멱등성**을 지켜야 재시도/중복요청에서도 안전합니다.

---

## 3 상태코드 사용 원칙

- 성공: `200 OK`(조회/일반), `201 Created`(+ `Location` 헤더), `204 No Content`(본문 없음)
- 클라이언트 오류: `400 Bad Request`, `401 Unauthorized`, `403 Forbidden`, `404 Not Found`, `409 Conflict`, `422 Unprocessable Entity`
- 서버 오류: `500`/`502`/`503`/`504`
- **일관된 에러 바디**를 반드시 동봉(§9 참조)

---

## 4. 표현(Representation) & 콘텐츠 협상

- `Content-Type: application/json` 기본
- `Accept` 헤더로 버전/미디어타입 협상 가능
- 숫자/날짜는 **명확한 타입/포맷**: ISO8601 `UTC`(예: `2025-11-24T07:10:00Z`)

---

## 5. 버저닝(Versioning)

- **경로 버전**이 가장 단순: `/v1/orders` → `/v2/orders`
- 장기 운영은 **헤더 버전**도 고려: `Accept: application/vnd.example.v2+json`
- **하위호환 우선**: 필드 추가는 OK, 의미 변경/삭제는 **선언→유예→폐기** 단계로

---

## 6. 페이징/정렬/필터

- 쿼리 규칙 통일:
    - `GET /orders?page=2&size=20&sort=createdAt,desc&status=PAID`
- 응답 예:
  ```json
  {
    "data": [/* ... */],
    "page": 2,
    "size": 20,
    "totalElements": 132,
    "totalPages": 7,
    "links": {
      "self": "/orders?page=2&size=20",
      "next": "/orders?page=3&size=20",
      "prev": "/orders?page=1&size=20"
    }
  }
  ```
- 커서 방식(대규모/변동성 큰 데이터):
  + GET /events?after=eyJpZCI6IjEyMyJ9&limit=50
  + 응답에 nextCursor 포함

---

## 7. 생성/업데이트 요청 바디

- `POST /roders`
    + ```json
      {
          "userId": "u_123",
          "items": [{"sku":"A12","qty":2}],
          "memo": "gift wrap"
      }
      ``` 
- `PUT` 은 전체 스냅샷, `PATCH`는 변경분
    + ```json
      { "memo": "remove gift wrap" }
      ``` 

---

## 8. 멱등키(Idempotency-Key) & 재시도

- 결제/주문 등 중복 방지가 필요한 POST 요청은:
  + Idempotency-Key: 5f6b-... 헤더 + 서버 측 키 기반 중복 차단
  + 동일 키 재요청 시 동일 응답 반환

---

## 9. 에러 응답 표준화 (RFC 7807 권장)

- `application/problem+json`
  + ```json
    {
      "type": "https://api.example.com/problems/validation-error",
      "title": "Validation failed",
      "status": 422,
      "detail": "items[0].qty must be >= 1",
      "instance": "/orders",
      "errors": {
        "items[0].qty": "must be >= 1"
      }
    }
    ```

---

## 10. 동시성 제어 & 조건부 요청

- 낙관적 락: Etag + If-Match
  + ```bash
    GET /profiles/10
    ETag: "v7"
    
    PATCH /profiles/10
    If-Match: "v7"
    ``` 
  + 버전 불일치 시 412 Precondition Failed
- 조건부 조회 캐싱: If-None-Match, If-Modified-Since

---

## 11. 캐싱 전략

- 정적/조회 API:
  + Cache-Control: public, max-age=60
  + 변경 잦으면 ETag/Last-Modified와 조건부 GET
- 민감 데이터는 Cache-Control: no-store

---

## 12. 보안(필수)

- TLS 강제(HSTS), 최신 암호 스위트
- OAuth2.1/OIDC 기반 토큰: Authorization: Bearer <JWT>
- Scope/Role로 최소 권한 적용
- 입력 검증(서버), 출력 이스케이프, 레이트 리밋/봇 차단
- 민감 응답에 PII 최소화, 감사 로그 남기기

---

## 13. 최소 예시(생성 -> 조회)

- 요청

```json
POST /v1/orders
Content-Type: application/json
Idempotency-Key: 5f6b-12ab

{
  "userId": "u_123",
  "items": [{"sku":"A12","qty":2}]
}
```

- 응답 성공

```json
201 Created
Location: /v1/orders/o_987
Content-Type: application/json
ETag: "v1"
X-Request-Id: 83f2f9f0

{
  "id": "o_987",
  "status": "PENDING",
  "userId": "u_123",
  "items": [{"sku":"A12","qty":2}],
  "createdAt": "2025-11-24T07:10:00Z",
  "links": { "self": "/v1/orders/o_987" }
}
```

- 에러 응답

```json
422 Unprocessable Entity
Content-Type: application/problem+json
X-Request-Id: 83f2f9f0

{
  "type": "https://api.example.com/problems/validation-error",
  "title": "Validation failed",
  "status": 422,
  "detail": "items[0].qty must be >= 1",
  "errors": { "items[0].qty": "must be >= 1" }
}
```