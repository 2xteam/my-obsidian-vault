---
title: Seveny Hair
type: project
status: 개발중
local: C:\Dev\seveny-hair
tags: [project]
updated: 2026-10-02
---

# Seveny Hair

헤어샵 **seveny hair** 웹사이트. 네이버 플레이스: https://m.place.naver.com/hairshop/1767344271/home

> 저장소는 **고객 계정 `sevenyhair/sevenyhair`** (2026-10-02 이관). 이전 `2xteam/seveny-hair` 는 보관용, 로컬 원격 이름 `2xteam`.
> 인프라는 고객 계정으로 새로 만든다 — 순서는 저장소 `docs/setup-infra.md`.

> MyJane 패밀리와 **별개 서비스**다. 스택은 [[Ignite]] 를 따른다.

## 스택

Next.js 15 · React 19 · Tailwind 3(preflight 끔) · MongoDB(mongoose 9) · 포트 **3040**

## 진행 상황

- [x] 2026-10-02 Webflow 사이트 "Laurence. das Haarlokal" 디자인·움직임 클론 (7개 라우트)
- [x] DB 설계 · 시드 스크립트 (`npm run seed`)
- [x] 2026-10-02 영어화 + 네이버 플레이스 실데이터(사진 22·스타일 26·가격·영업시간·소개) 반영, 로고 제작
- [x] 인스타 피드: 코드 저장 + 이미지 프록시 + `npm run ig` 수동 등록 (자동 수집은 하지 않기로 결정, 코드 제거)
- [x] 2026-10-02 한글화 (메뉴·제목 영어 / 설명·가격·이력 한글, Pretendard · Noto Serif KR), 로고 SEV/ENY
- [x] 2026-10-02 GitHub 연결(2xteam/seveny-hair) · Vercel 배포 오류 수정 (framework 명시, next/font 제거)
- [x] 2026-10-02 SEO — 라우트 메타 · OG 이미지(next/og) · sitemap · robots · HairSalon JSON-LD
- [x] 2026-10-02 모바일 고정 배경 확대 문제 → FixedBg (1차 fixed+clip 은 iOS 에서 실패, 2차 터치 기기 sticky 로 해결)
- [ ] 이미지 R2 이전 → 타겟 CDN 허용 제거
- [ ] 관리자 CMS (Ignite 패턴)
- [x] 2026-10-02 저장소 이관 → sevenyhair/sevenyhair, Vercel 프로젝트 고객 쪽에 새로 생성 중
- [x] 2026-10-02 고객 MongoDB Atlas (`sevenyhair.8bgdaru.mongodb.net`) 생성 · 로컬 `MONGODB_URI` 교체 · `npm run seed` (35건)
- [ ] 새 Vercel 프로젝트에 `MONGODB_URI` 등록 → Redeploy
- [ ] 고객 Cloudflare R2 버킷 `sevenyhair` · API 토큰 (이미지 업로드 기능 때 사용)
- [ ] 도메인 연결 → `NEXT_PUBLIC_SITE_URL`

## DB — `seveny`

| 컬렉션 | 내용 |
|---|---|
| `shop` | 매장 정보 단일 문서 (`key: "main"`) — 로고·주소·영업시간·결제수단·내비 |
| `pages` | 페이지별 히어로·섹션 (`slug`) |
| `services` | 가격표 — 탭 → 표(열 머리) → 행(서비스명·가격들) |
| `testimonials` | 후기 |
| `staff` | 디자이너 소개 |
| `posts` | 블로그/공지 (`slug`, `publishedAt`) |

지금까지는 myjane 클러스터 인증을 빌려 썼다(시드는 안 함). 고객 Atlas 로 새로 만든다 → [[MongoDB Atlas]]. DB 이름은 코드 상수 `seveny`.

## 클론 방법

전체 절차는 [[웹사이트 클론과 콘텐츠 교체]] 로 옮겼다. 아래는 요약.

Webflow 사이트는 움직임 정의가 JS 번들 안에 통째로 들어 있다.
`webflow.schunk.*.js` 에서 `Webflow.require("ix2").init({...})` 의 객체를 꺼내면
이벤트(PAGE_START, SCROLL_INTO_VIEW, SCROLLING_IN_VIEW…)와 액션 리스트(지속시간·지연·easing·값)를
수치로 얻는다. 스크린샷으로 추정하는 것보다 정확하다.

- 연속 스크롤 효과의 `smoothing: 50` 은 매 프레임 `current += (target - current) * 0.5`
- 액션 그룹은 순차 실행 (그룹 0 이 끝나야 그룹 1)

## 데이터 출처 (스크래핑 메모)

- 네이버 플레이스: `api.place.naver.com/graphql` (User-Agent·Referer 필요)
  - `getPhotoViewerItems` cursors `[{id:"biz"}]` → 업체 사진, `clip` → 클립(영상 주소는 hdnts 토큰 만료)
  - `getBeautyStyles` → 스타일 정보
  - 페이지 HTML 의 `window.__APOLLO_STATE__` 에 소개·메뉴·영업시간·디자이너
- 인스타: `instagram.com/p/{code}/media/?size=l` → 최신 CDN 주소로 302. CDN 은 CORP same-origin → 서버 프록시 필수
- Chrome 확장은 naver.com 접근이 막혀 있다(안전 제한). curl 로 받았다.

## 함정

- **모바일 고정 배경** — `background-attachment: fixed` 를 모바일이 무시해 Milestones 배경이 크게 확대됐다. PC 를 좁혀 보면 정상이라 실기기에서만 보인다 → FixedBg. 1차 "fixed 레이어 + clip" 은 iOS 에서 늦게 그려져 실패, PC=background-attachment fixed / 터치=sticky 로 해결

- **화면 캡처가 멈춘다**: Chrome 창이 Claude 앱 창에 완전히 가려지면 Windows 창 가림 감지로
  `document.visibilityState=hidden`, rAF 정지 → 스크린샷 30초 타임아웃. JS 실행은 된다(진단에 사용).
  Chrome 154 에는 `calculate-native-win-occlusion` 플래그가 없다. **두 창을 나란히(Win+←/→) 두면 해결** (2026-10-02 확인)
  → [[화면 확인의 함정]]
- 창이 최대화돼 있으면 `resize_window` 가 안 먹는다. 같은 출처 페이지에 375px·820px iframe 을 띄워
  미디어쿼리를 확인했다.
- 원본 영업시간이 페이지마다 다르다(홈 DI–DO/SO, Kontakt DI–MI/DO). `shop.hours` 하나로 통일했다.
