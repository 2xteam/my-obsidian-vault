---
title: F 보호자·자녀 계정
type: plan
tags: [plan, auth, privacy, children]
updated: 2026-09-09
status: 구현 완료 (2026-09-09) — 운영 확인 남음
applies-to: [myjane, SnapWord, SnapNote, fitlog, 2hbk, typelog]
---

# F 보호자·자녀 계정

만 14세 미만은 **보호자가 가입하고 그 계정 안에 자녀를 추가**한다 (2026-09-07 사용자 결정).
아동에게서 직접 연락처를 받지 않으니 법정대리인 동의 문제가 단순해진다.
[[C 법적 페이지]] 의 방침 6항은 지금 "만 14세 미만 가입은 받지 않습니다. 보호자 계정에 자녀를
추가하는 방식을 준비하고 있습니다" 라고 적혀 있다 — **이 계획이 끝나면 그 문장을 되돌린다.**

먼저 [[인증과 세션 공유]] · [[E 개인정보 보호 보강]] 을 읽을 것. 로그인은 이메일+비밀번호만이고,
서버는 HttpOnly 토큰의 `uid` 로 사람을 안다. 자녀는 그 위에 **프로필**로 올라간다.

## 왜 계획서가 필요한가

여섯 앱이 공유하는 `users` 와 세션 토큰의 의미가 바뀐다. 한 세션에서 끝낼 크기가 아니고,
잘못 잡으면 나중에 고치기 어렵다(E 에서 phone 키를 걷어내는 데 라우트 68개를 만졌다).

## 사용자 결정 (2026-09-09)

- 자녀는 `users` 의 **별도 row**. `parentId` 로 보호자를 가리킨다 (팔로우 때문에 `userId` 도 필요)
- 로그인할 때 **보호자인지 어느 자녀인지 골라** 들어간다
- FitLog 건강정보 동의는 **자녀를 추가할 때 체크박스**로 받는다
- 만 14세가 되면 **이메일·비밀번호를 등록해 독립**하는 절차를 둔다

## 구현 (2026-09-09, 같은 날)

```
여섯 앱  models/User.ts            parentId · independenceOnVerify · independentAt
여섯 앱  lib/sessionToken.ts       claims.gid (자녀 세션이면 보호자 _id)
다섯 앱  lib/auth.ts               Viewer.guardianId (참고용 — 앱 동작은 그대로)
포털     lib/family.ts             MAX_CHILDREN 5 · createChild · pickToken(5분) · purgeUserAcrossApps · familyFollow
         lib/profileSession.ts     프로필로 세션 발급 (child 표시 · hasEmail true)
         lib/serverSession.ts      requireSessionUser 가 자녀 세션(gid)을 **기본 403** — allowChild 옵션
         app/api/auth/login        자녀가 있으면 choose + pickToken + profiles (세션 미발급)
         app/api/auth/pick-profile 고른 프로필로 세션 (자녀면 gid)
         app/api/auth/switch       GET 목록 · POST 전환 — 자녀→보호자는 보호자 비밀번호 필요
         app/api/account/children  GET · POST(법정대리인 동의 필수 · 건강·국외 선택) · [id] PATCH · DELETE(즉시 폐기) · [id]/independence
         app/account/children      자녀 관리 화면 · app/account/switch 전환 화면 · components/ProfilePicker
         app/api/auth/verify-email independenceOnVerify → parentId 해제 · independentAt · 세션 폐기
         app/api/cron/purge        보호자 폐기 때 자녀도 함께 · purgeUserAcrossApps 공용화
         components/LandingAuth    헤더에 이름(자녀) + 전환 링크
2hbk     app/api/admin/family-follow  가족 전원 양방향 approved (ADMIN_API_SECRET)
방침     6항을 실제 동작으로 다시 썼다 (자녀 프로필 · 즉시 삭제 · 독립) · 가입 화면 문구
```

- 나이는 `올해 - 출생연도 - 1 ≥ 14` (생일을 모르므로 보수적)
- 자녀 프로필은 이메일이 없어 앱 `EmailBanner` 가 뜨지 않게 세션에 `child: true`
- 자녀 세션에서 포털 계정 API 를 부르면 403 `{ child: true }` — 동의 화면은 "보호자 프로필에서" 안내

### 배포 확인 (2026-09-09 운영 · myjane 4435fad · SnapWord b4277af · SnapNote 4b4b6a0 · fitlog 08393ff · 2hbk 063da4b · typelog ea6f2e7)

```
/account/children · /account/switch   200
/api/auth/switch · /api/account/children 무인증 401 · pick-profile 엉터리 토큰 401
2hbk /api/admin/family-follow 무인증 403 · 방침 6항 "자녀 프로필" 렌더 · 법적 페이지 헤더-탭 여백
```
아래 항목은 실제 계정으로 사람이 한 번 봐야 한다.

### 운영에서 확인할 것

- [ ] 자녀 추가 → 로그인 시 프로필 선택 → 자녀로 SnapWord 폴더 생성 → 보호자 프로필에는 안 보임
- [ ] 자녀 세션으로 /account/withdraw · /api/account/consents → 403
- [ ] 2hbk 친구 목록에 가족이 서로 보임
- [ ] 자녀 삭제 → 다섯 앱 0건 (응답 `purged`)
- [ ] 독립: 출생연도 14세 이상 → 메일 → 링크 → 자기 이메일로 로그인 · 보호자 목록에서 사라짐

## 결정해야 할 것 (원래 제안표 — 위 결정으로 확정)

| 물음 | 제안 | 이유 |
|---|---|---|
| 자녀는 별도 `users` 문서인가, 보호자 문서 안의 배열인가 | **별도 문서** (`users` 에 `guardianId` 를 가진 row) | 기록의 소유자 키(`createdBy` · `userId`)가 이미 회원 `_id` 다. 자녀도 `_id` 를 가져야 기존 라우트가 그대로 돈다 |
| 자녀 문서에 무엇을 두나 | `name`(별칭) · `guardianId` · `birthYear`(선택) · `signupFrom`. **이메일·전화번호·비밀번호 없음** | 자녀는 로그인하지 않는다. 연락처를 받지 않는 것이 이 방식의 핵심이다 |
| 로그인 뒤 "누구로 들어가나" | 포털 로그인 → 프로필 고르기(본인 / 자녀…) → 토큰의 `uid` 를 **그 프로필의 `_id`** 로 발급, `gid` 에 보호자 `_id` | 앱은 `uid` 만 보므로 **앱 코드를 안 고친다.** 프로필을 바꾸면 다시 발급 |
| 자녀 프로필에서 못 하는 것 | 탈퇴·동의 변경·이메일·비밀번호 변경·자녀 추가 | 보호자만 한다. `gid` 가 있으면 계정 설정 API 를 403 |
| FitLog 건강정보 동의 | 자녀 프로필의 건강정보·국외이전 동의는 **보호자가** 자녀별로 한다 | 민감정보 동의는 프로필별. `healthDataAgreedAt` 은 자녀 문서에 남기고 동의 화면은 보호자 세션에서 |
| 2hbk 친구 | 보호자와 자녀, 자녀끼리 **자동 팔로우** (2026-09-07 결정) | `follows` 에 양방향 생성. 자녀는 `userId`(도메인 id) 도 만들어 준다 |
| 자녀 수 | 보호자당 최대 5 | 남용 방지 |
| 자녀 프로필 삭제 | 보호자가 지우면 그 프로필의 데이터를 **즉시** 폐기 (6개월 보관 없음) | 계정 탈퇴가 아니라 보호자가 관리하는 하위 데이터다. 방침에 그렇게 적는다 |
| 자녀가 14세가 되면 | 이번 범위 밖. 필요해지면 "독립" 기능(이메일 등록 → 별도 계정) | 지금 만들지 않는다 |

## 손대는 곳

```
여섯 앱  models/User.ts        guardianId (ObjectId, null) · isChildProfile 없이 guardianId 유무로 판단
포털     lib/sessionToken.ts   claims 에 gid (보호자 _id, 선택)
         app/api/auth/login    응답에 profiles[] · 프로필 고르기 없이 본인으로 발급
         app/api/auth/switch   { profileId } → 그 프로필로 새 토큰 (보호자 세션에서만)
         app/account/children  자녀 추가·이름 변경·삭제 (법정대리인 동의 게이트 → 12번)
         app/api/account/children  CRUD · 삭제는 다섯 앱 purge-user 호출 (즉시)
다섯 앱  lib/auth.ts           getViewer 가 gid 를 함께 돌려준다 (403 판단용)
         계정 설정 라우트       gid 있으면 403 (탈퇴 링크 · 동의 · /api/me PATCH)
2hbk     자녀 추가 훅          follows 양방향 · userId 발급
포털     components/ProfileSwitcher  상단에 "본인 / 자녀" 전환
```

## 순서

1. [x] 사용자 결정표 확정 (2026-09-09)
2. [x] `guardianId` 여섯 스키마 + 토큰 `gid` + `getViewer` (앱은 아직 아무 동작 변화 없음)
3. [x] 자녀 CRUD + 법정대리인 동의 게이트(`guardianAgreedAt` — E 의 분리 동의 셋 중 하나, 게이트만 비어 있다)
4. [x] 프로필 전환 (포털 UI + `/api/auth/switch`) · 앱 상단에 "누구로 보고 있는지" 표시
5. [x] 자녀 프로필에서 계정 설정 403
6. [x] 2hbk 자동 팔로우
7. [x] 자녀 삭제 = 즉시 폐기 (purge-user 재사용)
8. [x] 방침 6항 문구 되돌리기 · 가입 화면 문구 — `POLICY_VERSION` 은 같은 날(2026-09-09) 개정이라 그대로 두었다
9. [ ] 검증 — 자녀 토큰으로 탈퇴·동의 API 403 · 자녀 기록이 보호자 프로필에 섞이지 않음 · 자녀 삭제 뒤 다섯 앱 0건

## 함정 (예상)

- **`uid` 가 자녀인 세션으로 포털 계정 API 를 부르면** 보호자 것을 바꾸는 사고. 모든 계정 설정 라우트가 `gid` 를 먼저 본다
- `users` 유일 인덱스 — 자녀는 `email`·`phone` 이 없다. `email_1` 은 비유일이라 괜찮지만 **유일 인덱스를 걸게 되면 부분 인덱스**로
- 통합 admin 회원 목록에 자녀가 섞여 보인다. `guardianId` 로 묶어 보여 준다
- 탈퇴 폐기(6개월) 가 보호자를 지울 때 **자녀 문서와 자녀 데이터도** 함께 지워야 한다 — `purge` 가 `guardianId` 로 자녀를 찾아 재귀
