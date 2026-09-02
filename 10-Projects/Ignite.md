---
title: Ignite
type: project
status: 이관대기
domain: www.ignitearch.co.kr
repo: https://github.com/Seuyup/ignite
local: C:\Dev\ignite
branch: main
tags: [project]
updated: 2026-09-02
---

# Ignite

건축사무소 웹사이트. 프로젝트 포트폴리오 + 관리자 CMS.

> MyJane 패밀리와 **별개 서비스**다. 저장소 소유 계정(`Seuyup`)도 다르다.

## 스택

Next.js 15 · React 19 · MongoDB(mongoose 9) · Cloudflare R2 · Tiptap 3 에디터 · Swiper · sharp
현재 **AWS EC2 + nginx + PM2**에서 운영 중이며 Vercel 이관 준비가 끝난 상태.

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
- [ ] **Vercel 계정 재가입 대기** — 계정 삭제 후 재가입이 막혀 지원 문의 중
- [ ] 가비아 DNS에서 `@`·`www` 함께 전환
- [ ] 컷오버 후 **AWS Elastic IP 해제** (인스턴스만 종료하면 계속 과금)

절차: 저장소 `docs/vercel-migration.md`
