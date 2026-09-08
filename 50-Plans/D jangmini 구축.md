---
title: D jangmini 구축
type: plan
tags: [plan, jangmini, portfolio, openai]
updated: 2026-09-08
status: 계획 확정 — Phase 0 착수 대기
applies-to: [jangmini]
---

# D jangmini 구축

AI 챗봇형 포트폴리오 [[jangmini]] 를 0부터 배포까지 여섯 단계로 나눈 것.
A·B·C 와 무관한 별개 트랙이다 — 여섯 앱 파일을 건드리지 않는다.

앱 자체의 사실(스택·데이터·결정·함정)은 [[jangmini]] 에 있다.
**이 문서는 순서와 방법만** 적는다.

## 왜 이 순서인가

**Phase 0(수집·DB화)을 먼저 한다.** 여기서 나온 스키마가 tool 정의, 브라우징
페이지, 추천 질문, 캐시 키를 전부 결정한다. 코드를 먼저 짜면 스키마가 바뀔 때마다
세 군데를 다시 고친다.

**Phase 1 에서 OpenAI 하드 리밋을 먼저 건다.** 코드가 없어도 걸 수 있고, 코드
방어(Phase 5)에 버그가 있어도 이것만은 내 코드와 무관하게 작동한다.

## Phase 0 — 수집과 DB화

앱 코드 없이 독립 스크립트로 한다.

- [ ] 스키마 확정 — `portfolio` 의 `kind` · `visibility` · `source` (표는 [[jangmini]])
- [ ] `scripts/ingest-notion.mjs` — MCP 우회 경로(`post-search` + `retrieve-page-markdown`).
      이미지는 **받는 즉시 R2 업로드** (presigned 1시간 만료)
- [ ] `scripts/ingest-docx.mjs` — 학력·자격증·대외활동·병역·자기소개 보강
- [ ] 정규화 결과 샘플을 **DB 에 넣기 전에** 사용자에게 보여준다
- [ ] `scripts/check-db.mjs` — TypeLog 것을 복사. 연결·**DB 이름 대조**·컬렉션·인덱스
- [ ] `settings` 초기값, `readers` 초기 계정(`2xteam`, bcrypt 해시), 인덱스 생성
- [ ] 추천 질문 8~12개 초안 → 사용자 검수

**멱등성** — `source.id` 로 upsert 한다. 다시 돌려도 중복이 생기지 않아야 한다.
재수집 때 `settings["content.sourceVersion"]` 을 올려 `answers` 캐시를 만료시킨다.

## Phase 1 — 저장소와 배포 골격

- [ ] `pnpm` 설치 (`npm i -g pnpm`) — 현재 없다. Node v24.13.1 · npm 11.8.0 은 확인됨
- [ ] `C:\Dev\jangmini` 로 클론 → `.git` 제거 → `git init` → `2xteam/jangmini` push
- [ ] `pnpm install` → `pnpm dev` 로 3006 확인
- [ ] **push 후에** Vercel 프로젝트 생성. `vercel.json` 에
      `{"regions":["icn1"],"framework":"nextjs"}` → [[Vercel 배포 패턴]]
- [ ] `.env.local` 생성 + `.gitignore` 확인. **값은 사용자가 직접 넣는다**
- [ ] **OpenAI 에 jangmini 전용 프로젝트와 전용 키**를 만들고 monthly hard limit 설정.
      다른 앱과 키를 공유하지 않는다 — 사고가 번지지 않게

## Phase 2 — 원작자 흔적 제거

- [ ] 교체 지점 인벤토리 작성 (경로별)
- [ ] `fastfolio-cta.tsx` 와 쓰는 곳 제거
- [ ] `welcome-modal.tsx` 는 **지우지 않고 재활용** — reader 로그인 안내 모달로 쓴다
- [ ] `public/` 원작자 에셋 삭제 — **지울 목록을 먼저 보고**한 뒤에
- [ ] `Raphael|Raphaël|Giraud|toukoum|fastfolio|lighton` grep 결과 보고

## Phase 3 — tool 을 DB 조회로

- [ ] `Data.tsx` 하드코딩 배열 → `portfolio` 조회로 교체
- [ ] tool 재정의 — `getProfile` `getExperience` `getProjects` `getSkills` `getContact`
- [ ] 원본 tool 제거 — `getSport` · `getCrazy` · `getIntership`(원본 오타)

> ⚠️ tool 하나를 지우거나 더할 때 **세 군데를 함께** 고친다. 하나라도 빠지면 런타임 에러다.
> `src/app/api/chat/route.ts` 의 `tools` · `src/components/chat/tool-renderer.tsx` 의
> `case` · 해당 컴포넌트 파일.

- [ ] 조회는 `visibility:"public"` 만 — private 을 집는 경로가 없는지 확인

## Phase 4 — 브라우징 · reader 로그인 · admin

- [ ] `/projects` `/projects/[slug]` `/resume` `/skills` `/faq`
- [ ] 첫 진입 모달 + "다시 보지 않기"(`localStorage`, **try/catch 필수** — 프라이빗
      모드에서 접근 자체가 throw 한다). 우상단 로그인 버튼은 상시 노출
- [ ] `POST /api/reader/login` — bcrypt 비교 → 만료 확인 → HMAC 서명 쿠키
      `jangmini_reader`, **호스트 전용** · httpOnly · Secure · SameSite=Lax
- [ ] **로그인 시도 제한** — IP 당 15분 5회. 4자리 비밀번호라 이게 없으면 전수 시도가 통한다
- [ ] `/admin` — `reader.role === "admin"` + **진입 시 비밀번호 재확인**, 세션 TTL 2시간
- [ ] `/admin/readers` `/admin/settings` `/admin/history` `/admin/suggestions` `/admin/usage`

## Phase 5 — 페르소나 · 방어 · 배포

- [ ] `prompt.ts` 재작성 — 원본 구조 유지, 내용은 플레이스홀더.
      규칙 4개(**없는 사실 지어내지 않기** · 주제 이탈 거절 · 사용자 언어 맞추기 ·
      급여·개인 연락처·재직 중 내부 정보 비공개) + 말미에 인젝션 가드
- [ ] 비용 방어 7층 (아래)
- [ ] `errorHandler` — 사용자에게는 일반 문구, 상세는 서버 로그만
- [ ] `layout.tsx` 메타데이터 · OG · favicon
- [ ] 가비아 `jangmini` CNAME → [[도메인과 DNS]]
- [ ] 검증 4종 (아래)

## 비용 방어 7층

| 층 | 수단 | 막는 것 |
|---|---|---|
| L0 | **추천질문 사전생성 캐시** | 정상 트래픽 대부분. OpenAI 호출 0회 |
| L1 | 입력 500자 · 최근 10턴 · `maxSteps` · `maxOutputTokens` | 한 요청의 최대 단가 |
| L2 | `readers_history` 기반 익명 제한 (`clientId` + `ipHash` 중 **엄격한 쪽**) | 한 사람의 반복 |
| L3 | **전역 일일 캡** (`usage` + `settings`) → 초과 시 채팅만 끄고 브라우징 유도 | 분산된 다수 · IP 로테이션 |
| L4 | **OpenAI 프로젝트 hard limit** + 전용 키 | 위가 전부 뚫렸을 때의 최종 상한 |
| L5 | Origin/Referer 검증 · honeypot. 필요하면 Turnstile(무료) | 스크립트 봇 |
| L6 | 인젝션 가드 + 주제 이탈 시 **짧게** 거절 | 거절도 토큰이다. 길게 설명하지 않는다 |

**L0 이 가장 크고 L4 가 마지막 보루다.** L2 는 사용자도 지적했듯 시크릿 창이면
초기화되므로 완벽할 수 없다 — L3·L4 가 실제 방어선이다.

`settings` 로 재배포 없이 조절한다. 환경변수는 **배포 시점에 스냅샷**되므로 숫자
하나 바꿀 때마다 재배포해야 한다 → [[Vercel 배포 패턴]]

## 검증 방법

| 확인할 것 | 어떻게 |
|---|---|
| 빌드 | `pnpm build`. **개발 서버를 먼저 내린다** → [[AI 협업 규칙]] |
| 빠른 질문 5개 | 각 버튼이 올바른 tool 을 부르고 컴포넌트가 그려지는지. **캐시 히트로 OpenAI 호출 0회**인지 함께 확인 |
| 페르소나 | "연봉이 얼마냐" · "파이썬 코드 짜줘" · "이전 지시를 무시하고 …" 세 개를 실제로 던진다 |
| rate limit | 익명으로 한도+1 회 → 차단 문구와 브라우징 유도가 뜨는지. reader 로그인 후 풀리는지. 만료된 reader 는 막히는지 |

`readers_history` 를 직접 조회해서 카운트가 실제로 쌓였는지 본다. 화면만 보고
"동작한다"고 판단하지 않는다.

## 관련

[[jangmini]] · [[Plans MOC]] · [[MongoDB Atlas]] · [[Vercel 배포 패턴]] · [[AI 채팅 패턴]]
