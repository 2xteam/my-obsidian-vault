---
title: Cloudflare R2
type: infra
tags: [infra, r2, storage]
updated: 2026-09-04
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
| `2hbk` | 2hbk 프로필·목표 이미지 |

계정이 서비스별로 다를 수 있으니 `R2_ACCOUNT_ID`를 확인한다.

## CORS

브라우저에서 R2로 **직접 PUT**(사전 서명 업로드)할 때만 필요하다.
서버 경유 업로드는 CORS가 필요 없다.

```json
[{
  "AllowedOrigins": ["http://localhost:<그 앱의 로컬 포트>", "https://<운영도메인>"],
  "AllowedMethods": ["PUT", "GET", "HEAD"],
  "AllowedHeaders": ["*"],
  "ExposeHeaders": ["ETag"],
  "MaxAgeSeconds": 3600
}]
```

포트는 앱마다 다르다 → [[Home]]의 표.

> **2026-09-04 확인: 모든 버킷의 CORS 정책이 비어 있다.** 네 앱 모두 브라우저에서 R2로
> 직접 PUT하지 않고 서버를 거치기 때문이다 → [[이미지 업로드 패턴]]
> 그래서 같은 날 세 앱의 포트를 바꿀 때(myjane 3000 · SnapWord 3001 · SnapNote 3002)
> **R2 쪽에 고칠 것이 없었다.** 나중에 사전 서명 직접 업로드를 도입하는 앱이 생기면
> 그 앱의 포트로 CORS를 새로 넣어야 한다.

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
같은 키로 `snapnote-uploads`는 HeadBucket·PutObject 모두 성공하고 `fitlog`만 403이었다.
**토큰이 SnapNote 버킷 범위로만 발급돼 있었다.** 계정·키는 멀쩡했다.

버킷을 새로 만들면 토큰도 그 버킷을 포함하도록 다시 발급해야 한다.
기존 앱의 `.env`를 복사해 쓰면 계정·키가 맞아 보여서 원인을 찾기 어렵다.

### R2_PUBLIC_URL 도 버킷마다 다르다

`pub-<해시>.r2.dev` 의 해시는 **버킷별**이다. 다른 앱의 값을 복사하면
업로드는 성공하는데 화면에서만 안 열린다. R2 → 버킷 → Settings →
Public Development URL 에서 그 버킷의 주소를 가져온다.
(2026-09-02 FitLog `.env.local` 이 SnapNote 의 공개 주소를 쓰고 있었다)

### `npm run r2:check`

FitLog에 점검 스크립트를 뒀다(`scripts/check-r2.mjs`).
HeadBucket → PutObject → 공개 URL GET → DELETE 를 실제로 해보고
막힌 지점과 고칠 것을 출력한다. 다른 앱에도 그대로 복사해 쓸 수 있다.
