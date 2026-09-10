---
title: MyJane
type: project
status: 운영중
domain: www.myjane.co.kr
repo: https://github.com/2xteam/myjane
local: C:\Dev\myjane
branch: main
tags: [project, myjane]
updated: 2026-09-10
---

# MyJane

myjane.co.kr 포털. 통합 로그인과 서비스 안내를 맡는다.

랜딩은 **카테고리별로 시트를 나눈다** — 공부 기록(SnapWord · SnapNote) /
건강 기록(FitLog) / 습관 기록([[2hbk]]). 한 덩어리 제품처럼 소개하지 않는다
→ [[서비스 카테고리와 카피 원칙]]

## 성격

원래 의존성 없는 정적 사이트였으나, **2026-09-02 통합 로그인이 들어오면서 DB에 접속한다.**
랜딩(`/`)은 여전히 정적이고, 인증 라우트만 서버에서 돈다.

필요한 환경 변수: `MONGO_URI` `MONGO_USER_DB` `NEXT_PUBLIC_COOKIE_DOMAIN`
`NEXT_PUBLIC_BASE_URL` `SMTP_*` `SESSION_SECRET` `ADMIN_API_SECRET` `CRON_SECRET` `APP_*_ORIGIN`
`GOOGLE_CLIENT_ID/SECRET` `KAKAO_CLIENT_ID/SECRET` `NAVER_CLIENT_ID/SECRET`(소셜 로그인, 2026-09-10) — `NEXT_PUBLIC_*` 은
**Config 타입**이어야 한다.

`SESSION_SECRET`은 세션 서명 키다. **2hbk 배포와 같은 값**이어야 한다
→ [[인증과 세션 공유]]

## 브랜딩

**라이트 전용**이다. 2026-09-06부터 **먹청 톤**을 쓴다 —
청백 `#f7fbfb` / 먹청 `#116271` / 골드 `#c9a84c` → [[먹청 톤 팔레트]]
그 전에는 결쩜사 팔레트(`#fdfbff` / `#7c3aed`)를 그대로 썼는데 색까지 같아서 너무 닮았다.
**구조는 결쩜사를 그대로 쓴다** — 시트 쌓기, 짙은 시트로 히어로를 잡고 어두운 푸터로 닫기.
다크 모드를 얹지 않는다. 밝은 바탕에 짙은 시트를 얹는 대조가 인상이라 그게 사라진다.
서체는 본문 Pretendard + 헤드라인 Gowun Batang 700.

먹청은 어두워서 **면적·글자·테두리에 같은 값을 쓸 수 있다**(흰 시트 위 6.98:1).
그래서 `--accent` = `--accent-ink` 이고 버튼 글자는 흰색이다.
금색만 예외로 좁은 면적에만 쓴다(흰 시트 위 2.29:1).

같은 자리에 **살구 `#ff805d`** 를 한 번 넣었다가 되돌렸다. 밝은 강조색은 글자로
못 써서 토큰을 셋으로 쪼개야 했고, 결쩜사의 짙은 히어로·어두운 푸터도 무거워
보여 뺐다가 결국 다 되돌렸다. 그 대가가 [[먹청 톤 팔레트]]에 적혀 있다.

상단·푸터 워드마크는 `my`(본문색) + `jane`(먹청). 푸터는 어두우니 `my`가 청백,
`jane`이 `#5fb8c9` 다 → [[앱 공통 UI와 아이콘]]

아이콘은 `public/app-icon.svg`(직접 그린 소문자 mj)가 원본이고 `npm run icons`로
`myjane-icon.png`(512px)·`icon-192`·`apple-touch-icon`·`favicon-32`를 만든다.
2026-09-04 이전에는 사용자가 준 사진(`app-icon-source.jpg`)에서 파생했다.

**아이콘 타일은 아직 짙은 보라다.** 이 기하학을 여섯 개(myjane + 다섯 앱)가
공유하므로 하나만 바꾸면 가족이 어긋난다. 함께 바꿔야 한다.

## 랜딩 구조

`app/page.tsx` 한 파일이다. 카드를 고칠 때는 `STUDY_APPS` / `HEALTH_APPS` /
`HABIT_APPS` 배열을 만진다.

| 시트 | 내용 |
|---|---|
| 히어로(dark) | "필요한 기록만, 골라서 쌓아요" |
| STUDY(white) | 공부 기록 — SnapWord · SnapNote 2단 카드 |
| HEALTH(tint) | 건강 기록 — FitLog 단독 카드(`.apps--solo`, 620px 중앙) |
| HABIT(white) | 습관 기록 — 2hbk 단독 카드 |
| ABOUT(tint) | "묶어둔 건 계정뿐이에요" 4개 항목 |
| START(white) | 회원가입 CTA — **로그인 상태에서는 감춤** |

로그인 상태에 따라 갈리는 조각(`components/LandingAuth.tsx`) —
우측 상단은 `로그아웃`만, 회원가입 버튼·히어로 버튼·마지막 CTA 시트는 감춘다
→ [[인증과 세션 공유]]

## 통합 로그인

**2026-09-10 부터 소셜 로그인(구글 · 카카오 · 네이버)이 붙었다.** 로그인 화면 맨 위에 버튼 셋, 그 아래 이메일 폼.
첫 가입은 `/signup/social` 동의 화면 한 번. 매칭·콘솔 절차·함정은 [[G 소셜 로그인]], 수단 표는 [[인증과 세션 공유]].

네 앱의 로그인·회원가입을 여기로 모았다.

```
snapword.myjane.co.kr  →  www.myjane.co.kr/login?from=snapword&next=/home
                       ←  .myjane.co.kr 쿠키 저장 후 원래 앱으로 복귀
```

- `lib/apps.ts` — 앱 목록과 복귀 URL 생성. `next`는 **경로만** 허용해 오픈 리다이렉트를 막는다
- **입력칸은 하나다.** 이메일이든 전화번호든, 비밀번호든 PIN이든 받는다
  (`lib/identifier.ts`). 2hbk 전용이던 이메일 화면은 이 화면이 대신한다
- `requiresSessionToken` 인 앱(2hbk)은 세션에 서명 토큰이 없으면 되돌려보내지 않는다
- `particle`은 이름 뒤 조사 — `2hbk`는 "케이"로 끝나 `으로`가 아니라 `로`다
- `users.signupFrom` 에 가입 출처를 기록
- `from=fitlog` 이면 회원가입에서 신체 프로필(키·성별·출생연도)을 함께 받는다
- 각 앱은 운영 도메인일 때만 포털로 보낸다. 로컬 개발에서는 자기 로그인 화면을 쓴다
  (localhost에는 쿠키 도메인이 적용되지 않기 때문)
- 화면 구성은 [[결쩜사 페이지 패턴]]을 따르고 색만 myjane 팔레트를 쓴다

## 예정 작업

- [x] **통합 admin** (2026-09-04) — 앱별 탭. 회원·공지·문의·요약 → [[통합 admin]]
- [x] **앱 아이콘 교체** (2026-09-04) — 사용자가 준 사진에서 파생하던 것을
      직접 그린 소문자 `mj` SVG 로 바꿨다. j 의 꼬리가 게이지 호에 내접해 이어진다.
      원본 `public/app-icon.svg` · `npm run icons` → [[앱 공통 UI와 아이콘]]
- [ ] 토큰 사용 로그(`token_logs`) — 어느 앱의 어느 기능에서 썼는지
- [ ] 각 앱에 남은 회원가입·PIN 재설정 화면 정리
- [x] 2hbk 이메일 로그인 경로 · 세션 서명 토큰 (2026-09-03) → [[2hbk]]
- [x] 세션 쿠키를 HttpOnly 서버 쿠키로 전환 (2026-09-09) — `snap_session` · `snap_auth` · 표시용 `snap_user` → [[인증과 세션 공유]] · [[E 개인정보 보호 보강]]
- [x] FitLog 카드 추가 · 카테고리 구조로 랜딩 재구성 (2026-09-02)

## `/link` — SNS 프로필 링크 모음 (2026-09-09)

인스타그램 본문에는 링크가 걸리지 않아 프로필 링크 하나를 `www.myjane.co.kr/link` 로 둔다.
여섯 서비스의 **소개 페이지**(로그인 전 화면)로 가는 카드 목록이고, 그때 홍보하는 서비스를 맨 위에 올린다.
검색 색인은 막았다(`robots: noindex`) → [[SNS 소개 카드와 게시글]]

## `/share` — 공유형 링크 모음 (2026-09-10)

`/link` 가 서비스 소개 페이지 목록이라면, `/share` 는 **앱 안에서 만들어진 공유 링크**(예: TypeLog 결과 `/r/<token>` —
친구가 나눈 타입을 보고 로그인 없이 바로 참여)를 운영자가 골라 나열하는 페이지다. 첫 항목은 TypeLog 러닝 성향.

- 데이터는 포털 자신의 것 → `user` DB `share_links` (`models/ShareLink.ts`). 제목 · 설명 · 주소 · 서비스 · 이모지 · 순서 · 켜짐
- **주소는 myjane 서비스 도메인의 https 만** 받는다(`shareLinkUrlProblem`) — 남의 주소를 우리 이름으로 내보내지 않는다
- 관리는 통합 admin **`/admin/share`**(운영자·마스터) — 등록 · 수정 · 위/아래 · 끄기(숨김) · 삭제. API 는 `/api/admin/share`
- 공개 화면은 서버 컴포넌트가 켜진 것만 읽는다(캐시 60초). 검색 색인은 막았다
- 첫 건은 `scripts/seed-share-link.mjs` 로 넣었다. 이후는 admin 에서
- SNS 프로필 링크는 `/link` 하나로 두고, `/link` 하단에서 `/share` 로 이어 준다 → [[SNS 소개 카드와 게시글]]

## 팔레트 원본이 여기 있다

여섯 앱이 쓰는 색의 단일 원본이 이 저장소에 있다.

```
design/palette.json          ← 여기만 고친다
npm run palette -- --write   ← 여섯 앱의 app/palette.css 가 다시 만들어진다
```

쓰기 전에 대비 31건을 검사하고 하나라도 미달이면 아무 파일도 쓰지 않는다.
`app/palette.css` 는 여섯 앱 모두 생성 파일이다 → [[먹청 톤 팔레트]]

## 소개 페이지의 카테고리 순서

2026-09-07에 사용자가 정했다. 카테고리 시트를 이 순서로 쌓는다.

```
HEALTH  건강 기록   FitLog
TYPE    성향 기록   TypeLog
STUDY   공부 기록   SnapWord · SnapNote
HABIT   습관 기록   2hbk
ABOUT MYJANE
```

시트 배경은 흰색 / 연청록을 **번갈아** 쓴다(`.sheet` / `.sheet--tint`).
순서를 바꿀 때 `sheet--tint` 가 연속되지 않는지 확인할 것 → [[결쩜사 페이지 패턴]]

푸터 저작권 표기는 `myjane` 으로 쓴다(`MyJane` 아님).
