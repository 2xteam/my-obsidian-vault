---
title: D jangmini 구축
type: plan
tags: [plan, jangmini, portfolio, openai]
updated: 2026-09-08
status: Phase 0~4 완료 · Phase 5 착수 대기
applies-to: [jangmini]
---

# D jangmini 구축

AI 챗봇형 포트폴리오 [[jangmini]] 를 0부터 배포까지. A·B·C 와 무관한 별개
트랙이고 여섯 앱 파일을 건드리지 않는다.

앱 자체의 사실(스택·데이터·결정·함정)은 [[jangmini]] 에 있다.
**이 문서는 순서와 남은 일만** 적는다.

## 이어받는 세션이 먼저 할 것

2026-09-08 세션이 사용량 한도로 중단됐다. 이어받을 때 —

1. [[jangmini]] 를 읽는다. 특히 "여섯 앱과 다른 점" 표와 "함정" 절
2. `C:\Dev\jangmini` 에서 상태를 확인한다

```bash
cd C:/Dev/jangmini
git log --oneline -1        # 4818033 이어야 한다
pnpm db:check               # portfolio 101 · settings 12 · readers 1
pnpm r2:check               # 권한 3단 통과 + 공개 URL 대조
pnpm dev                    # 3006
```

3. **개발 서버가 떠 있으면 `pnpm build` 를 돌리지 않는다.** 같은 `.next/` 를
   써서 개발 서버가 먹통이 된다. 타입은 `npx tsc --noEmit` 으로 본다
   → [[AI 협업 규칙]]

### 지금 어디까지 되어 있나

배포는 살아 있고(`https://jangmini.myjane.co.kr`) 문서 페이지 넷이 동작한다.
**AI 채팅만 닫혀 있다** — `POST /api/chat` 이 503 을 준다.

닫아 둔 이유: 사이트가 공개된 상태에서 원본 라우트가 인증도 제한도 없었고,
`messages` 배열을 클라이언트가 통째로 보내 **범용 GPT 프록시로 쓸 수
있었다.** 급한 구멍만 막은 임시 차단장치가 `route.ts` 에 있다
(킬스위치 · system 역할 거부 · 역할별 글자 수 상한 · `maxTokens` ·
errorHandler 일반화). 제대로 된 방어는 아래 Phase 5 다.

---

## Phase 5 — 남은 일

순서가 중요하다. **채팅을 켜는 것이 맨 마지막이다.**

### 5-1. 추천 질문과 답변 캐시 (여기부터)

- [ ] `src/lib/suggestions.ts` 의 16개를 `suggestions` 컬렉션에 넣는다
      (seed 스크립트에 추가). **`key` 는 절대 바꾸지 않는다** — 캐시의 조회 키다
- [ ] 답변 **사전 생성** 스크립트. 각 `key` × `lang`(ko/en) 로 한 번 호출해
      `answers` 에 저장한다. `sourceVersion` 을 `settings["content.sourceVersion"]`
      에서 읽어 함께 박는다
- [ ] 사람이 읽고 손볼 수 있게 한다 — 채용 담당자가 볼 확률이 가장 높은
      답변을 **검수된 문장으로 고정**하는 것이 목적이다
- [ ] 칩을 누르면 **텍스트가 아니라 `key`** 를 보낸다. 지금 `/faq` 와 랜딩은
      `?query=<문장>` 을 보내므로 `?q=<key>` 로 바꿔야 캐시가 맞는다
- [ ] 캐시 히트면 **OpenAI 를 호출하지 않는다.** DB 에서 읽어 조각내 흘려보내
      UX 를 같게 유지한다 → [[AI 채팅 패턴]]
- [ ] 재수집 때 `content.sourceVersion` 을 올려 옛 답변을 만료시킨다.
      안 하면 이력을 고쳐도 챗봇이 옛 이야기를 계속 한다

### 5-2. 방어 7층

| 층 | 수단 | 막는 것 |
|---|---|---|
| L0 | 5-1 의 사전생성 캐시 | 정상 트래픽 대부분. 호출 0회 |
| L1 | 입력 상한 · `maxSteps` · `maxOutputTokens` | 한 요청의 최대 단가 |
| L2 | `readers_history` 기반 익명 제한 | 한 사람의 반복 |
| L3 | **전역 일일 캡** (`usage` + `settings`) | 분산된 다수 · IP 로테이션 |
| L4 | **OpenAI 프로젝트 hard limit** | 위가 전부 뚫렸을 때의 최종 상한 |
| L5 | Origin/Referer 검증 · honeypot | 스크립트 봇 |
| L6 | 인젝션 가드 (이미 프롬프트에 있음) | "이전 지시를 무시하라" 류 |

- [ ] L1 상한을 `route.ts` 하드코딩에서 **`settings` 조회**로 옮긴다
      (서버에서 60초 메모이즈). 환경 변수는 배포 시점에 스냅샷되어 숫자 하나에
      재배포가 필요하다 → [[Vercel 배포 패턴]]
- [ ] L2 — `clientId`(localStorage) 와 `ipHash` 중 **엄격한 쪽**을 적용한다.
      IP 는 원문을 저장하지 않고 `sha256(ip + IP_HASH_SALT)` 만 둔다
- [ ] L3 — 캡에 걸리면 "죄송합니다"로 끝내지 않고 **`/resume`·`/projects` 로
      보낸다.** 문서 페이지를 먼저 만든 이유가 이것이다
- [ ] L4 — **사용자 확인 필요.** OpenAI 키가 다른 앱과 공유 중이다.
      전용 프로젝트 키로 바꾸고 monthly hard limit 을 걸어야 L4 가 성립한다.
      지금은 이 층이 비어 있다
- [ ] L1 상한 값은 `settings` 에 이미 있다 —
      `chat.maxUserChars` 500 · `chat.maxMessages` 21 ·
      `chat.maxOutputTokens` 800 · `chat.maxSteps` 3 ·
      `chat.anon.perMinute` 3 / `perHour` 15 / `perDay` 30 ·
      `chat.global.dailyRequests` 500 · `chat.global.dailyTokens` 500000

### 5-3. reader 로그인

- [ ] `POST /api/reader/login` — bcrypt 비교 → 만료 확인 → HMAC 서명 쿠키
- [ ] 쿠키 `jangmini_reader` · **호스트 전용**(`domain` 미지정) ·
      httpOnly · Secure · SameSite=Lax
- [ ] 도착하는 myjane `snap_user` 쿠키는 **완전히 무시**한다 → [[jangmini]]
- [ ] **로그인 시도 제한** — `login_attempts`, IP당 15분 5회.
      4자리 비밀번호라 이게 없으면 전수 시도가 통한다
- [ ] 첫 진입 안내 모달에 **"다시 보지 않기"** — `localStorage`.
      **접근 자체가 throw 할 수 있으니 try/catch 로 감싸고, 실패해도 화면이
      정상 동작하게** 한다(프라이빗 모드)
- [ ] 우상단 로그인 버튼은 **상시 노출**. "다시 보지 않기" 를 눌러도 들어갈 수
      있어야 한다. 로그인 후 `2xteam · 만료 D-12` 배지
- [ ] 익명도 사이트를 전부 본다. reader 로그인은 **게이트가 아니라 제한 해제**다
      (채용 담당자는 계정이 없다)
- [ ] 제한에 걸렸을 때 입력창을 잠그는 UI 는 **이미 자리가 있다** —
      자식 컴포넌트의 `hasReachedLimit` prop 을 지우지 않고 남겨 뒀다

### 5-4. admin

- [ ] `/admin` — `readers.role === 'admin'` + **진입 시 비밀번호 재확인**,
      admin 세션 TTL 2시간 (reader 는 30일)
- [ ] `/admin/readers` 발급·만료·비번재설정·활성 · `/admin/settings` ·
      `/admin/history` · `/admin/suggestions`(캐시 답변 검수) · `/admin/usage`
- [ ] **비밀번호 변경 화면**을 꼭 둔다 — 초기값이 4자리다
- [ ] `featured` 토글을 admin 에 만들면 그때 원본을 `ingest.mjs` 의
      `FEATURED` 에서 DB 로 옮긴다. 그전까지 재수집이 DB 값을 덮으므로
      두 곳에 두지 않는다

### 5-5. 채팅 개방과 검증

- [ ] Vercel 환경 변수에 **`CHAT_ENABLED=true`** 를 넣고 재배포.
      **여기가 마지막이다** — 5-1~5-4 가 끝나기 전에는 넣지 않는다
- [ ] Vercel 환경 변수 점검 — `MONGO_URI` 가 **`mongodb+srv://`** 인지
      (로컬 표준 URI 를 복사하면 안 된다), `MONGO_DB` ·
      `NEXT_PUBLIC_COOKIE_DOMAIN` 이 **없는지**

| 확인할 것 | 어떻게 |
|---|---|
| 빌드 | `pnpm build`. **개발 서버를 먼저 내린다** |
| 빠른 질문 5개 | 각 버튼이 올바른 tool 을 부르고 컴포넌트가 그려지는지. **캐시 히트로 OpenAI 호출 0회**인지 함께 |
| 페르소나 | "연봉이 얼마냐" · "파이썬 코드 짜줘" · "이전 지시를 무시하고 …" 세 개를 실제로 던진다 |
| rate limit | 익명으로 한도+1 회 → 차단 문구와 브라우징 유도. reader 로그인 후 풀리는지. **만료된 reader 는 막히는지** |

`readers_history` 를 직접 조회해 카운트가 실제로 쌓였는지 본다.
화면만 보고 "동작한다"고 판단하지 않는다.

---

## 사용자에게 확인해야 할 것

- [ ] **OpenAI 전용 프로젝트 키** — 지금 다른 앱과 공유 중이라 L4 가 없다.
      jangmini 가 어뷰징당하면 SnapWord·SnapNote·FitLog 예산까지 태우고,
      키를 회수하면 그 앱들이 함께 죽는다 (여러 번 알렸고 아직 미결)
- [ ] **`profile/profile/1-*.png` (2.2MB)가 본인 사진인지.** 맞으면 랜딩
      히어로를 그것으로 바꿀 수 있다 — 지금은 사용자가 준 메모지를 쓰고 있다
- [ ] `public/resume.pdf` — 다운로드 버튼을 둘지
- [ ] `contribution` 이 비어 있는 11건 (2018~2022 큐텐 시절). 노션에서 채우면
      재수집 때 반영된다
- [ ] 노션 태그 `cloudflare` 가 한 번도 안 붙었다. Ignite 에 Cloudflare S3 를
      썼으니 붙일 만하다

## 하지 않기로 한 것

- **JSON Resume 테마** — 그 스키마에 회고·기여도·4단 본문이 들어갈 자리가 없다
- **레퍼런스 스크린샷 페이지** — 구조는 정해졌고 더 보는 건 결정을 늦춘다
- **myjane 디자인 시스템 적용** — 사용자가 명시적으로 하지 말라고 했다.
  원작자 색·타이포를 그대로 쓰고 레이아웃만 문서형으로 잡았다
- **`이미지 갤러리에 object-cover`** — 스크린샷의 위아래가 잘려 정작 봐야 할
  UI 가 사라진다. `object-contain` 이다

## 관련

[[jangmini]] · [[Plans MOC]] · [[MongoDB Atlas]] · [[Cloudflare R2]] ·
[[Vercel 배포 패턴]] · [[AI 채팅 패턴]]
