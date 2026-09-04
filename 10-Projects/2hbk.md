---
title: 2hbk
type: project
status: 운영중
domain: 2hbk.myjane.co.kr
repo: https://github.com/2xteam/2hbk
local: C:\Dev\2hbk
branch: main
db: hamhibokka
tags: [project, myjane]
updated: 2026-09-03
---

# 2hbk

함히보까 — 목표를 정하고 해낼 때마다 스티커를 붙여 채우는 기록 도구.

**이름은 `2hbk`로 쓴다.** `함히보까`는 설명하는 자리(랜딩 푸터·메타 설명·포털 카드
소개문)에서만 쓴다. 2026-09-03 사용자 지시.

## 이관 (2026-09-03)

원래 **React Native 앱 + NestJS/GraphQL 백엔드** 두 저장소였다.

```
https://github.com/2xteam/hamhibokka-frontend   RN 0.80 · Apollo · Recoil (src 약 11,500줄)
https://github.com/2xteam/hamhibokka-backend    NestJS 11 · Apollo Server · Azure Blob · JWT
```

Next.js 15 단독 프로젝트로 옮겼다. **옛 Azure 리소스는 둘 다 이미 사라진 상태였다** —
`hamhibokka-backend-*.azurewebsites.net`과 `hamhibokkastorage.blob.core.windows.net`이
공용 DNS에서 NXDOMAIN이다. 즉 앱은 이관 시점에 이미 동작하지 않았다.

> 로컬 DNS(`10.124.0.5`)로 조회하면 NXDOMAIN이 나와도 `nslookup <host> 8.8.8.8`은
> **NXDOMAIN에도 종료 코드 0**을 준다. "resolves"로 잘못 읽었다.
> 실제 확인은 `dns.Resolver`에 `8.8.8.8`을 지정해 `resolve4()`로 해야 한다.

## 스택

Next.js 15 App Router · MongoDB(`hamhibokka` DB) · Cloudflare R2 · Vercel `icn1`
로컬 개발 포트 **3004** (SnapWord 3000 · SnapNote 3001 · myjane 3002 · FitLog 3003).

DB 이름은 URI 경로가 아니라 `lib/db.ts`에서 `hamhibokka`로 못 박는다 → [[MongoDB Atlas]]

## 로그인 — **이 앱만 이메일이다**

네 앱 중 유일하게 **이메일 + 비밀번호**로 로그인한다. 나머지 셋은 전화번호 + PIN이다.
옛 백엔드가 이메일 로그인이었고 기존 회원 17명의 bcrypt 해시를 그대로 옮겼기 때문에
**같은 비밀번호로 계속 로그인된다** → [[인증과 세션 공유]]

포털에 `usesEmailLogin` 플래그와 `/api/auth/login-email` · `/api/auth/register-email`,
`components/EmailAuth.tsx`가 이 앱을 위해 새로 들어갔다.

### API는 쿠키의 id를 믿지 않는다

`snap_user` 쿠키는 클라이언트가 고칠 수 있는 평문이다. 이 앱은 **남의 목표에 스티커를
붙이고 참가를 승인하는 동작**이 있어 그대로 둘 수 없었다. 같은 쿠키에 **HMAC 서명
토큰**(`token`)을 한 칸 덧붙이고 서버는 그것만 검증한다.

- 발급: 포털 `lib/sessionToken.ts` · 검증: 2hbk `lib/auth.ts`
- 두 배포가 **`SESSION_SECRET`을 같은 값으로** 공유해야 한다
- 다른 세 앱은 모르는 필드라 무시한다 → 기존 세션과 호환된다
- 검증: 정상 토큰 200 / 서명 변조 401 / 토큰 없음 401 / 쿠키 id만 위조 401

## 데이터

| 컬렉션 | 내용 |
|---|---|
| `hamhibokka.goals` | 목표 1건. **참가자와 스티커 적립 로그를 문서 안에 내장** |
| `hamhibokka.follows` | 팔로우 (`followerId` → `followingId`) |
| `hamhibokka.goalinvitations` | 초대(`invite`) · 참가 요청(`request`) |
| `user` DB `users` | 통합 회원 + 2hbk 전용 `userId` `nickname` `password` `profileImage` `followApprovalRequired` `emailVerified` |

> ⚠️ **`users.userId`가 도메인 식별자다.** Mongo `_id`가 아니다.
> 목표의 `createdBy`·`participants[].userId`, 팔로우 양쪽, 초대 양쪽이 모두
> `user_xxxxxxxxx` 문자열을 참조한다. 이관에서 이 값을 보존했기 때문에
> **목표·팔로우·초대 문서는 한 건도 손대지 않았다.**

`user.users`의 유일 인덱스는 **부분 인덱스**로 걸었다. 다른 앱 회원은 `userId`가 없고
스키마 기본값이 `null`이라 sparse로는 null끼리 충돌한다.

```js
{ userId: 1 }, { unique: true, partialFilterExpression: { userId: { $type: "string" } } }
```

이메일 유일 인덱스는 걸지 않았다. 네 앱이 공유하는 컬렉션이라 기존 `email_1`(비유일)을
갈아엎어야 하기 때문이다. 중복은 가입 API에서 막는다.

### 목표 방식이 공개 범위를 정한다

| 방식 | 공개 범위 | 자동 승인 | 첫 참가자 |
|---|---|---|---|
| `personal` 혼자 하기 | `private` | 예 | 만든 사람 |
| `competition` 겨루기 | `public` | 아니오 | 없음 |
| `challenger_recruitment` 챌린저 모집 | `followers` | 아니오 | 없음 |

입력으로 들어온 값이 있으면 그쪽이 이긴다(`lib/services/goals.ts`).

## 회원 이관 결과

```
hamhibokka.users 17건 → user.users
  신규 16 · 병합 1 (이메일이 겹친 계정)
user.users 7건 → 23건
```

- `_id`·`userId`·bcrypt 해시 보존. `name`은 `nickname`으로 채우고 `phone`/`pin`은 비움
- `signupFrom: "2hbk"` 기록
- 병합된 1건은 **전화번호+PIN과 이메일+비밀번호 둘 다로 로그인된다**
- 원본 `hamhibokka.users`는 지우지 않고 되돌릴 근거로 남겼다
- 백업은 `2hbk/scripts/backup/<타임스탬프>/`에 JSON으로 (`.gitignore` 등록)

검증 — 도메인 데이터가 참조하는 `userId` 10건이 모두 통합 회원에 있다.
유일한 예외가 `user_ykk3aj07x` 하나인데 **이관 이전부터 없던 계정**이다
(삭제된 계정이 만든 테스트 목표 `goal_pfxbpamtt`. 참가자·팔로우·초대에는 없다).
화면은 이런 경우 `알 수 없는 사용자`로 그린다.

## 함정 / 결정

### 스티커 지급 권한을 좁혔다

옛 백엔드의 `receiveSticker`는 **로그인만 되어 있으면 누구든 아무 목표의 아무
참가자에게** 스티커를 붙일 수 있었다(리졸버에 `JwtAuthGuard`만 있었다).
스티커를 모아 목표를 달성하는 앱에서 이건 남의 기록을 조작할 수 있다는 뜻이다.
**목표를 만든 사람만** 줄 수 있게 막았다. `personal`은 만든 사람이 곧 참가자다.

잘못 붙인 것을 뗄 수 있게 `-1`도 허용한다(가진 것보다 많이 뺄 수는 없다).

### 자동 승인을 실제로 지키게 했다

옛 백엔드는 `autoApprove`가 켜져 있어도 참가 요청 문서를 만들었다. "자동 승인"이라고
표시해 두고 생성자가 일일이 눌러야 했다. 지금은 켜져 있으면 바로 참가자가 된다.

거절당한 요청도 지우고 다시 보낼 수 있게 했다. 예전에는 한 번 거절당하면 영영
같은 목표에 요청할 수 없었다.

### 이미지는 옮길 수 없었다

프로필 3건 · 목표 8건이 Azure Blob에 있었고 **그 스토리지 계정이 이미 사라졌다.**
DB의 주소는 그대로 두고(지우는 건 선택), 화면이 실패한 이미지를 감춘다
(`components/SafeImage.tsx` · `Avatar`는 닉네임 첫 글자로 되돌아간다).
`npm run check:images`로 확인하고 `--clean`으로 정리할 수 있다.

새 업로드는 R2로 간다. → [[Cloudflare R2]]

### 로그인 무한 왕복 (2026-09-03)

배포 첫 시도에서 로그인 후 홈에 들어가지 못하고 로그인 화면과 끝없이 오갔다.
판단 기준이 화면 게이트(사용자+토큰)와 로그인 화면(사용자만)에서 갈렸던 것이다.
원인과 대응 세 가지는 [[인증과 세션 공유]]에 정리했다.

`components/AuthGate.tsx`에 왕복 횟수 카운터를 뒀다. 예상 못 한 조합에서 또
왕복하더라도 무한 이동 대신 안내 화면으로 끝난다.

### 가입에서 서명은 쓰기보다 먼저

먼저 저장하고 나중에 토큰을 서명하니, 서명이 실패했을 때(운영에서 `SESSION_SECRET`
누락) **계정만 남고 세션은 없는** 상태가 됐다. 그 뒤로는 다시 가입할 수도 없다 —
"이미 가입된 이메일입니다"만 나온다.

`_id`를 미리 만들어 서명한 뒤에 쓴다. 서명이 실패하면 아무것도 저장되지 않는다.

### ⚠️ dev 서버가 도는 중에 `npm run build` 를 돌리지 말 것

`next build`가 `.next`를 갈아엎어 **켜져 있던 dev 서버가 깨진다.** 증상은
`Cannot find module './331.js'` 같은 500이고, 코드를 아무리 고쳐도 낫지 않는다.
dev 서버를 멈추고 `.next`를 지운 뒤 다시 띄워야 한다.
2026-09-03에 이걸로 한참 헤맸다.

### N+1 조회를 없앴다

옛 백엔드는 참가자마다 사용자 조회를 한 번씩 돌렸다(`findByUserId`). 목표 12개짜리
목록에도 수십 번 왕복했다. 목록을 그리기 전에 등장하는 `userId`를 모아
`lookupUsers()`로 한 번에 가져온다.

## 디자인

**결쩜사 패턴을 따른다** → [[결쩜사 디자인 시스템]] · [[결쩜사 페이지 패턴]]
옛 RN 앱의 민트(`#4ECDC4`) 팔레트는 버렸다.

- 기본 **라이트** `#fdfbff` / `#7c3aed`, 선택으로 퍼플 다크 · 커스텀 (테마 3종)
- 화면은 `.sheet`(plain / tint / dark / gold) 단위로 쌓고 eyebrow → 헤드라인 → 콘텐츠
- 상단 메뉴는 영어 `Home` `Goals` `Friends`, 부가 기능(`Invites` · `My` · `Logout`)은
  `More ▾`로 접는다 → [[앱 공통 UI와 아이콘]]

### 스티커판이 주인공이다

목표에 필요한 만큼 칸을 깔고 받은 만큼 금색 씰을 채운다(`components/StickerBoard.tsx`).
`12/20`이라고 쓰는 것보다 **남은 칸이 눈에 보이는 것**이 훨씬 잘 통한다.
칸이 60개를 넘으면 앞쪽만 그리고 `+N`으로 적는다.

금색은 결쩜사 규칙대로 **강조점에만** — 채워진 씰과 달성 배지뿐이다.
다 채우면 시트가 `gold` 톤으로 바뀐다.

목표를 만드는 화면에서도 필요한 스티커 수를 바꾸면 미리보기 판이 바로 늘어난다.
숫자보다 칸 수를 보는 편이 "이게 얼마나 걸릴 일인지" 가늠하기 쉽다.

## 진행 상황

- [x] 두 저장소 클론 · 도메인 로직 파악
- [x] Next.js 골격 · 결쩜사 팔레트·서체·시트 구조
- [x] 회원 이관 (드라이런 → 백업 → 실행 → 검증)
- [x] 포털 이메일 로그인 경로 · 서명 토큰
- [x] REST Route Handler 전체 (목표 · 참가 · 스티커 · 팔로우 · 초대 · 프로필)
- [x] 화면 전체 (랜딩 · 홈 · 목표 · 상세 · 친구 · 초대 · 프로필)
- [x] 포털 랜딩에 **HABIT · 습관 기록** 시트 추가
- [x] 아이콘 (게이지 호 + 별 심볼) · 매니페스트
- [x] R2 `2hbk` 버킷 + 토큰 (사용자가 준비) → [[Cloudflare R2]]
- [x] GitHub 저장소 · 첫 커밋 (2026-09-03, `2xteam/2hbk` main)
- [x] Vercel 배포 · 환경 변수 11개 (2026-09-03)
- [x] `2hbk` 서브도메인 연결 → [[도메인과 DNS]]
- [x] 운영 확인 — 포털 로그인 → 2hbk 진입 → `/api/me` 200
      (두 배포의 `SESSION_SECRET`이 맞는다는 증거다)
- [ ] 옛 저장소 아카이브 · Azure 잔여 리소스 정리 확인
