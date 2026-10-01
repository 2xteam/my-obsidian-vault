---
title: Ignite
type: project
status: 운영중
domain: www.ignitearch.co.kr
repo: https://github.com/Seuyup/ignite
local: C:\Dev\ignite
branch: main
tags: [project]
updated: 2026-10-01
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
- [x] 환경 변수 15개 등록 + 재배포 (2026-09-28) — **이게 빠져서 DB·관리자 로그인이 모두 죽어 있었다**
- [x] EC2 배포 워크플로·문서 삭제 (2026-09-28)
- [ ] GitHub Secrets `EC2_HOST` · `EC2_USER` · `EC2_SSH_KEY` 삭제
- [ ] **AWS Elastic IP 해제** (인스턴스만 종료하면 계속 과금)
- [ ] Atlas Network Access 에서 EC2 IP 항목 제거

> 이관 직후 증상은 "데이터가 안 보인다" 였지만 원인은 **환경 변수가 통째로 비어 있던 것**이다.
> 관리자 로그인 화면의 `ADMIN_PASSWORD가 설정되어 있지 않습니다` 가 결정적 단서였다 —
> DB만 보고 있었으면 더 돌아갔을 것이다. 한 변수가 비면 나머지도 의심한다.
>
> 로컬 `.env.local` 은 표준 URI 였고 Vercel 에는 **SRV**(`cluster0.dvwxz9v.mongodb.net`)를 넣었다.

절차: 저장소 `docs/vercel-migration.md`

## 실패가 보이지 않는 구조 (2026-09-28)

조회 함수(`project-queries` · `ignite-data`)는 **실패를 전부 삼키고 빈 배열**을 돌려준다.
화면이 통째로 죽지는 않지만, 그 대가로 "데이터 없음"과 "DB 연결 실패"가 같아 보인다.
이관 직후 목록이 비었을 때 밖에서는 원인을 알 수 없었다.

- `logDbFailure()` 로 삼킨 실패를 함수 이름과 함께 남긴다 → Vercel Logs 의 `[db]` 줄
- `GET /api/admin/health/db` — 환경 변수 유무·URI 의 DB 이름·연결 결과·컬렉션 건수.
  관리자 쿠키로 막지만 로그인은 `ADMIN_SECRET` HMAC 이라 **DB 없이도** 통과한다

같은 함정이 다른 앱에도 있다 → [[Vercel 배포 패턴]]

## 이관 후 두 번째 사고 — 세 라우트만 500 (2026-10-01)

`/studio` · `/contact` · `/p/[slug]` 만 500. 공통점은 `sanitizeRichHtml` 하나였고,
그 안의 `isomorphic-dompurify` 가 서버에서 jsdom 을 끌어온다. Vercel 에서
jsdom 의 의존성이 ESM/CJS 로 충돌해 **import 단계**에서 터졌다.

- 정제기를 `sanitize-html`(순수 JS) 로 교체. 결과는 실제 콘텐츠 8건으로 대조해 동일
- Node 버전 문제도 아니었고(v22.23.2) `serverExternalPackages` 로도 안 됐다
- `package.json` 에 `engines.node = 22.x` 를 남겼다 (`.nvmrc` 와 같은 메이저)
- `/api/build-info` 로 지금 떠 있는 커밋과 런타임 Node 버전을 확인한다

> `/privacy` 는 404 가 맞다. 관리자 `/admin/pages` 에서 만든 개별 페이지는
> 실제 경로가 **`/p/<slug>`** 다. 사이트 안의 링크도 `/p/privacy` 로 걸려 있다.

자세한 좁히는 방법 → [[Vercel 배포 패턴]]
