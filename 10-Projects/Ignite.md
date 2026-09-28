---
title: Ignite
type: project
status: 운영중
domain: www.ignitearch.co.kr
repo: https://github.com/Seuyup/ignite
local: C:\Dev\ignite
branch: main
tags: [project]
updated: 2026-09-28
---

# Ignite

건축사무소 웹사이트. 프로젝트 포트폴리오 + 관리자 CMS.

> MyJane 패밀리와 **별개 서비스**다. 저장소 소유 계정(`Seuyup`)도 다르다.

## 스택

Next.js 15 · React 19 · MongoDB(mongoose 9) · Cloudflare R2 · Tiptap 3 에디터 · Swiper · sharp
**Vercel(`icn1`)** 에서 운영 중. 2026-09-28 도메인 컷오버까지 끝났다.

## 구조

- 공개: `/` `/projects` `/projects/[slug]` `/p/[slug]` `/studio` `/contact` `/privacy`
- 관리자: `/admin/(panel)` — home · projects(add/list/modify/trash) · pages · studio · contact
- 전 페이지 `force-dynamic`

## 이관 준비 상태 (2026-09-02)

- [x] R2 사전 서명 업로드로 전환 (30MB 업로드가 Vercel 4.5MB 제한에 걸리던 문제)
- [x] 서버 sharp 압축 → 브라우저 canvas 리사이즈 이전
- [x] mongoose 서버리스 튜닝 · `vercel.json` 리전 `icn1` · maxDuration 명시
- [x] R2 버킷 CORS 적용
- [x] EC2 배포 워크플로 비활성화
- [x] Vercel 프로젝트 생성·배포 (2026-09-28)
- [x] 가비아 DNS 전환 — `@` A `216.198.79.1` (308 → www), `www` CNAME `…vercel-dns-017.com`
- [ ] 컷오버 후 **AWS Elastic IP 해제** (인스턴스만 종료하면 계속 과금)
- [ ] Atlas Network Access 에서 EC2 IP 항목 제거
- [ ] `.github/workflows/deploy-ec2.yml` 삭제 (현재는 수동 실행만 가능하도록 비활성)

절차: 저장소 `docs/vercel-migration.md`

## 실패가 보이지 않는 구조 (2026-09-28)

조회 함수(`project-queries` · `ignite-data`)는 **실패를 전부 삼키고 빈 배열**을 돌려준다.
화면이 통째로 죽지는 않지만, 그 대가로 "데이터 없음"과 "DB 연결 실패"가 같아 보인다.
이관 직후 목록이 비었을 때 밖에서는 원인을 알 수 없었다.

- `logDbFailure()` 로 삼킨 실패를 함수 이름과 함께 남긴다 → Vercel Logs 의 `[db]` 줄
- `GET /api/admin/health/db` — 환경 변수 유무·URI 의 DB 이름·연결 결과·컬렉션 건수.
  관리자 쿠키로 막지만 로그인은 `ADMIN_SECRET` HMAC 이라 **DB 없이도** 통과한다

같은 함정이 다른 앱에도 있다 → [[Vercel 배포 패턴]]
