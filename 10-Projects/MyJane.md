---
title: MyJane
type: project
status: 운영중
domain: www.myjane.co.kr
repo: https://github.com/2xteam/myjane
local: C:\Dev\myjane
branch: main
tags: [project, myjane]
updated: 2026-09-04
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
`NEXT_PUBLIC_BASE_URL` `SMTP_*` `SESSION_SECRET` `ADMIN_API_SECRET` — `NEXT_PUBLIC_*` 은
**Config 타입**이어야 한다.

`SESSION_SECRET`은 세션 서명 키다. **2hbk 배포와 같은 값**이어야 한다
→ [[인증과 세션 공유]]

## 브랜딩

**라이트 전용**이다. 결쩜사 팔레트를 따른다 — `#fdfbff` / `#7c3aed` / 골드 `#c9a84c`.
참고 사이트가 라이트 전용이어서 다크를 얹으면 대조가 어긋난다.
랜딩도 결쩜사처럼 **시트를 쌓는 구조**이며 어두운 푸터로 닫는다.
서체는 본문 Pretendard + 헤드라인 Gowun Batang 700.

상단·푸터 워드마크는 `my`(본문색) + `jane`(보라). 푸터는 어두우니 `my`가 흰색이다
→ [[앱 공통 UI와 아이콘]]

아이콘은 **사용자가 준 MJ 원본 이미지**를 쓴다. 원본은 `public/app-icon-source.jpg`,
파생본이 `public/myjane-icon.png`(512px) 및 각 크기.

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
- [ ] 세션 쿠키를 HttpOnly 서버 쿠키로 전환 검토 (현재는 클라이언트가 읽는 쿠키)
- [x] FitLog 카드 추가 · 카테고리 구조로 랜딩 재구성 (2026-09-02)
