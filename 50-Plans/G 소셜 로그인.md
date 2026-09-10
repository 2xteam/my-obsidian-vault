---
title: G 소셜 로그인
type: plan
tags: [plan, auth, oauth]
updated: 2026-09-10
status: 가이드 — 착수 전 (사용자 결정 대기)
applies-to: [myjane, SnapWord, SnapNote, fitlog, 2hbk, typelog]
---

# G 소셜 로그인 — 구글 · 카카오 · 네이버

런조아(runzoa.com)처럼 로그인 모달에 **구글 · 카카오 · 네이버** 버튼을 두는 방법. 2026-09-10 사용자 요청.
먼저 [[인증과 세션 공유]] · [[E 개인정보 보호 보강]] · [[F 보호자·자녀 계정]] 을 읽을 것 — 소셜 로그인은
**로그인 수단이 하나 늘어나는 것**이고, 세션·회원·자녀 프로필은 지금 구조를 그대로 쓴다.

## 한 줄 결론

포털(`www.myjane.co.kr`)에만 OAuth 라우트 둘을 두고(`start` · `callback`), 콜백에서 **기존 `issueProfileSession()` 으로
세션을 발급**한다. 다섯 앱은 아무것도 바꾸지 않는다 — 앱은 포털로 보내고 쿠키를 받을 뿐이다.
라이브러리(Auth.js)는 쓰지 않는다. 우리 세션 모델(HMAC 토큰 · `sessionVersion` · 자녀 `gid`)이 이미 있어서
Auth.js 의 세션을 얹으면 두 겹이 되고, 세 공급자의 인가 코드 흐름은 라우트 두 개로 충분하다.

## 0. 공급자 콘솔에서 받아 올 것 (운영자 · 사람이 한다)

| 공급자 | 어디서 | 받는 값 | 리다이렉트 URI |
|---|---|---|---|
| 구글 | Google Cloud Console → API 및 서비스 → 사용자 인증 정보 → **OAuth 클라이언트 ID(웹)** | `GOOGLE_CLIENT_ID` `GOOGLE_CLIENT_SECRET` | `https://www.myjane.co.kr/api/auth/oauth/google/callback` · `http://localhost:3000/api/auth/oauth/google/callback` |
| 카카오 | Kakao Developers → 내 애플리케이션 → 앱 키(REST API 키) · 카카오 로그인 활성화 · **동의항목**(닉네임 · 이메일) | `KAKAO_CLIENT_ID`(REST 키) `KAKAO_CLIENT_SECRET`(보안 탭에서 켠다) | `…/api/auth/oauth/kakao/callback` |
| 네이버 | Naver Developers → 애플리케이션 등록 → 네이버 로그인 · **제공 정보**(이메일 · 이름/별명) | `NAVER_CLIENT_ID` `NAVER_CLIENT_SECRET` | `…/api/auth/oauth/naver/callback` |

- OAuth 동의 화면(구글)·서비스 정보(카카오·네이버)에 **개인정보처리방침 URL** 이 필요하다 → `https://www.myjane.co.kr/legal/privacy` (이미 있다)
- ⚠️ **카카오 이메일**: "필수 동의" 로 받으려면 **비즈 앱 전환**(사업자 정보)이 필요하다. 개인 운영이라 **선택 동의**로만 받을 수 있고,
  사용자가 거부하면 이메일이 없이 온다. 아래 3-③ 이 그 경우를 다룬다
- 로컬 개발은 `localhost:3000` 을 리다이렉트 URI 로 함께 등록한다. `*.vercel.app` 프리뷰는 등록하지 않는다(도메인이 매번 바뀐다)
- 여섯 배포 중 **포털에만** 환경 변수를 넣는다

## 1. 흐름

```
앱 /login 버튼 ──▶ 포털 /login?from=fitlog&next=/home        (지금과 같다)
포털 모달 [구글 로그인] ──▶ GET /api/auth/oauth/google/start?from=fitlog&next=/home
   서버: state(무작위) + from/next 를 HttpOnly 쿠키 `oauth_state` 에 10분 저장 → 공급자 인가 URL 로 302
공급자 동의 ──▶ GET /api/auth/oauth/google/callback?code=…&state=…
   서버: state 검사 → code 로 토큰 교환 → 프로필(이메일 · 공급자 id · 이름) 조회
         → 회원 찾기/만들기 (아래 3) → issueProfileSession() (자녀 있으면 pickToken 흐름 그대로)
         → from/next 로 302 (앱으로 복귀) · 첫 가입이면 /signup/social 동의 화면으로 먼저
```

- **리다이렉트 방식**으로 한다(팝업 아님). 모바일 인앱 브라우저에서 팝업은 자주 막힌다 — 런조아 캡처도 인스타 인앱 브라우저다
- `state` 는 CSRF 방지. 쿠키에 둔 값과 쿼리 값이 다르면 400. `from`·`next` 도 쿠키에 넣어 두었다가 꺼낸다(`buildReturnUrl` 로 경로만 허용 — 오픈 리다이렉트 방지, 지금 규칙 그대로)
- 구글은 `id_token` 을 받는다 — 서명 검증은 `https://oauth2.googleapis.com/tokeninfo?id_token=` 로 하거나 JWKS 로 직접. `aud` 가 우리 클라이언트 id 인지, `email_verified` 가 true 인지 본다
- 카카오·네이버는 액세스 토큰으로 프로필 API 를 부른다 — 카카오 `GET https://kapi.kakao.com/v2/user/me` · 네이버 `GET https://openapi.naver.com/v1/nid/me`
- 공급자 토큰은 **저장하지 않는다.** 프로필을 읽는 데만 쓰고 버린다. 우리가 필요한 건 세션이지 공급자 API 가 아니다

## 2. 스키마 — 여섯 `models/User.ts` 함께

```ts
providers: [{ provider: "google" | "kakao" | "naver", providerId: String, email: String|null, linkedAt: Date }]
```

- `providerId` 는 공급자가 주는 고유 id(구글 `sub` · 카카오 `id` · 네이버 `response.id`). **이메일이 아니라 이 값으로 찾는다** — 이메일은 바뀔 수 있다
- 유일 인덱스 `{ "providers.provider": 1, "providers.providerId": 1 }` **부분 인덱스**로(배열이 빈 회원이 대부분)
- `password` 가 `null` 인 회원이 생긴다 — 지금 로그인 라우트는 `[password, pin].filter(Boolean)` 으로 걸러서 이미 안전하다.
  비밀번호 찾기(`forgot-pin`)는 "이 계정은 구글로 가입했어요" 라고 알려 주는 편이 낫다(같은 답 원칙과 충돌 — 로그인한 뒤 My 에서만 보여 준다)

## 3. 회원 매칭 규칙 — 여기가 제일 중요하다

| 경우 | 처리 |
|---|---|
| ① `providers` 에 같은 (provider, providerId) 가 있다 | 그 회원으로 로그인 |
| ② 없고, 공급자가 준 이메일이 **검증된 것**(구글 `email_verified` · 네이버는 검증된 이메일 · 카카오 `is_email_verified`)이며 그 이메일 회원이 있다 | **그 회원에 연결**하고 로그인. 이메일이 곧 신원이라는 지금 원칙과 같다 |
| ③ 없고, 이메일이 없거나 미검증 | **가입 화면으로** — 이메일을 입력받아 인증 메일(기존 `pendingEmail` 흐름). 인증 뒤에 계정을 만든다. 카카오 선택 동의 거부가 이 경우 |
| ④ 없고, 검증된 이메일이고 회원도 없다 | **첫 가입** — 동의 화면(아래 4)을 거쳐 회원을 만든다. `signupFrom: "google"` 처럼 출처를 남긴다 |

- ⚠️ **미검증 이메일로 기존 계정에 연결하면 계정 탈취**가 된다. 남의 주소를 자기 카카오에 넣어 두면 그 사람 계정에 들어간다. ②의 "검증된" 조건을 빼면 안 된다
- 이미 로그인한 상태에서 My 화면 "구글 연결" 도 같은 콜백을 쓴다 — 세션이 있으면 **그 회원에 연결**만 하고 세션은 유지
- 자녀 프로필은 소셜 로그인 대상이 아니다 — 자녀는 로그인하지 않는다 → [[F 보호자·자녀 계정]]

## 4. 동의 — "간주" 로 넘기지 않는다

런조아는 "로그인을 하시면 … 동의한 것으로 간주됩니다" 라고 쓴다. 우리는 **첫 가입 때만 동의 화면을 한 번 거친다** —
가입 라우트가 이미 `termsAgreedAt · privacyAgreedAt · agreedPolicyVersion · 만 14세 확인` 을 받고 있고, 방침에
"가입 시 동의를 받는다" 고 적어 두었다. 소셜이라고 이걸 건너뛰면 방침과 코드가 어긋난다 → [[C 법적 페이지]]

```
/signup/social  (콜백이 첫 가입을 만나면 여기로 — 공급자 프로필은 서명된 임시 쿠키 10분)
  이름(공급자 값으로 채움 · 고칠 수 있음)
  ☑ 만 14세 이상입니다          필수
  ☑ 이용약관 · 개인정보 동의    필수
  [가입하고 시작하기]  → 회원 생성 → issueProfileSession → 앱으로
```

두 번째 로그인부터는 화면 없이 바로 들어간다. 재로그인 때마다 묻지 않는다.

## 5. 화면 — 포털 `/login` 모달

- 이메일·비밀번호 폼 **위**에 버튼 셋. 런조아 순서(구글 → 카카오 → 네이버)를 따르되, 각 공급자의 **버튼 규정**을 지킨다
  - 카카오: 배경 `#FEE500` · 글자 `#000000 85%` · 말풍선 심볼 · 문구 "카카오 로그인" (다른 문구·색 불가)
  - 네이버: 배경 `#03C75A` · 흰 글자 · N 로고 · "네이버 로그인"
  - 구글: 흰 배경 + 회색 테두리 · 구글 G 로고 · "Google 계정으로 로그인" (구글 브랜딩 가이드 — 로고 변형 불가)
- 버튼 아래 한 줄: "이메일로 로그인" 접기/펼치기 — 기존 폼은 남긴다(소셜 없이 가입한 회원이 있다)
- `/migrate`(전화번호+PIN 전환) 링크는 그대로

## 6. 방침·볼트

- 방침 3항(제3자 제공) 또는 수집 항목에 **소셜 로그인** 줄 추가 — "구글·카카오·네이버 계정으로 로그인하면 그 서비스에서 이메일 · 이름(별명) · 계정 고유값을 받습니다. 공급자 토큰은 저장하지 않습니다"
- 쿠키 안내에 `oauth_state`(10분, 로그인 진행 중에만) 추가
- `POLICY_VERSION` 올림 → 개정 안내 띠(PolicyPrompt)가 알린다
- [[인증과 세션 공유]] 에 "로그인 수단 — 이메일+비밀번호 · 구글 · 카카오 · 네이버" 표

## 7. 손대는 파일 (포털만 · 스키마는 여섯)

```
myjane/lib/oauth/{google,kakao,naver}.ts     인가 URL · 토큰 교환 · 프로필 조회 (공급자별 40줄 안팎)
myjane/lib/oauth/state.ts                    state 쿠키 발급·검증 (HMAC · 10분 · from/next)
myjane/app/api/auth/oauth/[provider]/start/route.ts
myjane/app/api/auth/oauth/[provider]/callback/route.ts   3장 매칭 → issueProfileSession · 자녀 있으면 pickToken
myjane/app/signup/social/page.tsx             첫 가입 동의 화면
myjane/app/api/auth/oauth/complete/route.ts   동의 뒤 회원 생성
myjane/app/login/page.tsx                     버튼 셋 + 이메일 폼 접기
myjane/app/(app)/my … 연결 관리(선택)          "연결된 계정" 목록 · 연결 해제(비밀번호가 없으면 마지막 하나는 못 뗀다)
여섯 models/User.ts                           providers[]
myjane/app/legal/privacy/page.tsx · cookies    문구
```

## 8. 순서와 검증

1. [ ] 콘솔 셋 등록 · 환경 변수 6개를 포털 Vercel 에 (운영자)
2. [ ] 스키마 `providers` 여섯 앱 + 부분 유일 인덱스
3. [ ] 구글 하나로 start → callback → 세션까지 끝낸다. 자녀 있는 계정 · `from=fitlog` 복귀 · state 불일치 400 · 미검증 이메일 → ③ 확인
4. [ ] 카카오 · 네이버 추가 (프로필 API 만 다르다)
5. [ ] 첫 가입 동의 화면 · 동의 없이 `complete` 호출 → 400
6. [ ] 로그인 화면 버튼 · 브랜드 규정 확인
7. [ ] 방침·쿠키 안내 · `POLICY_VERSION`
8. [ ] 운영에서 세 공급자 실제 로그인 한 번씩 · 인스타 인앱 브라우저에서도

## 사용자가 정할 것

- 세 공급자 모두인가, 하나부터인가 (구글 하나로 시작하는 것을 권한다 — 콘솔 등록이 가장 단순하고 이메일이 항상 검증되어 있다)
- 카카오 이메일 선택 동의 거부 시 이메일을 **꼭 받을 것인가**(③처럼 입력받기) — 우리 원칙(이메일 필수)대로면 받는다
- 기존 회원의 My 화면 "소셜 계정 연결" 을 1차에 넣을지 (없어도 ②로 자동 연결되므로 뒤로 미룰 수 있다)
