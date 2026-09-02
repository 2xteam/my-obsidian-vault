---
title: MyJane
type: project
status: 운영중
domain: www.myjane.co.kr
repo: https://github.com/2xteam/myjane
local: C:\Dev\myjane
branch: main
tags: [project, myjane-family]
updated: 2026-09-02
---

# MyJane

앱 패밀리 포털. SnapWord · SnapNote · FitLog로 이동하는 링크 사이트.

## 성격

원래 의존성 없는 정적 사이트였으나, **2026-09-02 통합 로그인이 들어오면서 DB에 접속한다.**
랜딩(`/`)은 여전히 정적이고, 인증 라우트만 서버에서 돈다.

필요한 환경 변수: `MONGO_URI` `MONGO_USER_DB` `NEXT_PUBLIC_COOKIE_DOMAIN`
`NEXT_PUBLIC_BASE_URL` `SMTP_*` — `NEXT_PUBLIC_*` 은 **Config 타입**이어야 한다.

## 브랜딩

색 체계는 **결쩜사 팔레트**를 따른다 (라이트 `#fdfbff`/`#7c3aed`, 다크 `#1e0938`/`#a78bfa`).
랜딩도 결쩜사처럼 **시트를 쌓는 구조**이며 어두운 푸터로 닫는다.
MJ 아이콘(네이비+앰버)은 교체 예정이라 당분간 그대로 둔다.

이전 팔레트 메모 —
`public/myjane-icon.png`(512px)가 원본이고 나머지 크기는 여기서 파생한다.

| | 배경 | 강조 |
|---|---|---|
| 다크(기본) | `#0c2343` | `#f9c22a` |
| 라이트 | `#f4f6fc` | `#f2951a` (텍스트용 `#a2620a`) |

앰버는 흰 배경에서 대비가 부족해 텍스트 강조에는 어둡게 조정한 `--accent-ink`를 쓴다.

## 앱 카드 추가하기

`app/page.tsx`의 `APPS` 배열만 수정하면 된다.

## 통합 로그인

세 앱의 로그인·회원가입을 여기로 모았다.

```
snapword.myjane.co.kr  →  www.myjane.co.kr/login?from=snapword&next=/home
                       ←  .myjane.co.kr 쿠키 저장 후 원래 앱으로 복귀
```

- `lib/apps.ts` — 앱 목록과 복귀 URL 생성. `next`는 **경로만** 허용해 오픈 리다이렉트를 막는다
- `users.signupFrom` 에 가입 출처를 기록
- `from=fitlog` 이면 회원가입에서 신체 프로필(키·성별·출생연도)을 함께 받는다
- 각 앱은 운영 도메인일 때만 포털로 보낸다. 로컬 개발에서는 자기 로그인 화면을 쓴다
  (localhost에는 쿠키 도메인이 적용되지 않기 때문)
- 화면 구성은 [[결쩜사 페이지 패턴]]을 따르고 색만 myjane 팔레트를 쓴다

## 예정 작업

- [ ] **통합 admin** — 앱별 탭으로 회원·토큰·문의·공지·통계 관리
- [ ] 토큰 사용 로그(`token_logs`) — 어느 앱의 어느 기능에서 썼는지
- [ ] 각 앱에 남은 회원가입·PIN 재설정 화면 정리
- [ ] 세션 쿠키를 HttpOnly 서버 쿠키로 전환 검토 (현재는 클라이언트가 읽는 쿠키)
- [ ] FitLog 카드 추가
