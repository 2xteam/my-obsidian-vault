---
title: 도메인과 DNS
type: infra
tags: [infra, dns, gabia]
updated: 2026-09-03
---

# 도메인과 DNS

## myjane.co.kr (가비아)

네임서버: `ns.gabia.co.kr` 외. **Route 53·Cloudflare DNS는 쓰지 않는다.**

| 호스트 | 타입 | 대상 |
|---|---|---|
| `@` | A | Vercel (myjane 포털) |
| `www` | CNAME | Vercel (myjane 포털) |
| `snapword` | CNAME | Vercel |
| `snapnote` | CNAME | Vercel |
| `fitlog` | CNAME | Vercel |
| `2hbk` | CNAME | Vercel |

MX·TXT 레코드 없음 → 도메인 메일 미사용. DNS를 바꿔도 메일 영향이 없다.
TTL은 600으로 유지 — 문제 시 10분 내 롤백.

## ignitearch.co.kr (가비아)

| 호스트 | 타입 | 대상 |
|---|---|---|
| `@` | A | `3.35.114.6` (AWS EC2) |
| `www` | A | `3.35.114.6` |

MX 없음. Vercel 이관 시 두 레코드를 함께 바꾼다 → [[Ignite]]

## 가비아 DNS 관리툴 주의사항

- apex(`@`)에는 **CNAME을 넣을 수 없다** → A 레코드 사용
- 같은 호스트에 A와 CNAME 공존 불가 → 기존 레코드를 **먼저 삭제**
- 호스트 칸에는 서브도메인만 (`snapword`). 전체 도메인을 넣으면 중복됨
- CNAME 값 끝에 마침표가 필요할 수 있다
- 하단 **저장**까지 눌러야 반영된다

## 이관 이력

- 2026-09-01 SnapWord · SnapNote: Cloudways(`165.22.247.25`) → Vercel
- 2026-09-02 myjane apex·www: Cloudways → Vercel, **Cloudways 서버 완전 삭제**
- 2026-09-02 fitlog 신규 연결
- 2026-09-03 `2hbk` 신규 연결 → [[2hbk]]
