---
title: Cloudflare R2
type: infra
tags: [infra, r2, storage]
updated: 2026-09-02
---

# Cloudflare R2

이미지 저장소. S3 호환 API를 쓴다.

## 버킷

| 버킷 | 용도 |
|---|---|
| `snapnote-uploads` | SnapNote 오답노트 이미지 |
| `fitlog` | FitLog 인바디 결과지 원본 |
| `ignite` | Ignite 프로젝트 이미지 |
| `templete` | klead 등 공유 버킷 (앱별 프리픽스로 분리) |

계정이 서비스별로 다를 수 있으니 `R2_ACCOUNT_ID`를 확인한다.

## CORS

브라우저에서 R2로 **직접 PUT**(사전 서명 업로드)할 때만 필요하다.
서버 경유 업로드는 CORS가 필요 없다.

```json
[{
  "AllowedOrigins": ["http://localhost:3000", "https://<운영도메인>"],
  "AllowedMethods": ["PUT", "GET", "HEAD"],
  "AllowedHeaders": ["*"],
  "ExposeHeaders": ["ETag"],
  "MaxAgeSeconds": 3600
}]
```

**토큰 권한 주의**: 오브젝트 읽기/쓰기 전용 토큰으로는 CORS를 설정할 수 없다(`AccessDenied`).
Admin Read & Write 토큰을 쓰거나 Cloudflare 대시보드에서 직접 넣는다.

## 원본 보관 정책

FitLog는 인바디 결과지 원본을 보관한다. 이유는 세 가지.

1. 추출 오류를 나중에 발견했을 때 확인할 근거
2. `etc`에 있던 항목을 정식 필드로 승격할 때 **과거 데이터 재추출**
3. 종이 결과지를 버릴 수 있게 함

용지 1장이 200KB 남짓이라 비용은 미미하다.

## 토큰 권한 확인하는 법

버킷을 만들고 환경 변수를 채워도 **토큰이 그 버킷을 못 보면 전부 403**이다.
증상은 조용하다 — FitLog는 업로드 실패를 삼켜서 값만 저장되고 원본이 비었다.

```bash
# PutObject / HeadBucket / ListObjects 를 차례로 호출해 본다
node -e "..."   # 셋 다 AccessDenied 면 토큰 문제다
```

판단 기준 —

| 결과 | 뜻 |
|---|---|
| ListBuckets만 403 | 정상. 버킷 범위 토큰은 원래 계정 전체를 못 본다 |
| HeadBucket · ListObjects · PutObject 모두 403 | 토큰이 **그 버킷 범위가 아니거나 다른 계정**의 토큰이다 |
| PutObject만 403 | 읽기 전용 토큰이다. Object Read & Write 로 다시 만든다 |

키 형식이 맞는지도 먼저 본다(계정 ID 32자 hex · Access Key ID 32자 hex ·
Secret 64자 hex). 형식이 맞는데 403이면 값이 잘린 게 아니라 **권한 문제**다.

2026-09-02 FitLog 로컬이 이 경우였다 — 값 형식은 정상인데 세 호출 모두 403.
