---
title: jangmini
type: project
status: 개발중
domain: jangmini.myjane.co.kr
repo: https://github.com/2xteam/jangmini
local: C:\Dev\jangmini
branch: main
db: jangmini
tags: [project, portfolio]
updated: 2026-09-09
---

# jangmini

방문자가 물으면 **AI가 내 이력서를 근거로 답하는 포트폴리오.** 이름은 "장민"에
제미나이를 얹은 것이다.

> **2026-09-09 현재 — Phase 0~5 구현 완료, 로컬 검증까지 통과.**
> 남은 것은 **Vercel 환경 변수 등록과 `CHAT_ENABLED=true`** 뿐이다.
> 운영 배포는 아직 채팅이 닫혀 있다(503). → [[D jangmini 구축]]

## 여섯 앱과 다른 점 — 먼저 읽을 것

**myjane 여섯 앱의 규칙을 따르지 않는다.** 성격이 다른 앱이다.

| 항목 | 여섯 앱 | jangmini |
|---|---|---|
| 회원 | `user` DB 공유 | **공유하지 않는다** — `jangmini` DB 의 `readers` |
| 세션 쿠키 | `snap_user` · `.myjane.co.kr` | `jangmini_reader` · **호스트 전용** |
| 로그인의 뜻 | 서비스를 쓰려면 필요 | **제한 해제일 뿐** — 익명도 전부 볼 수 있다 |
| admin | 포털 통합 admin | **이 앱 자체 라우트** `/admin` |
| 디자인 | 여섯 앱 디자인 시스템 · 결쩜사 패턴 | 포크 기반. **따르지 않는다** |
| 카피 원칙 | 공부·건강·습관·성향 기록 | 해당 없음 — 개인 포트폴리오다 |

[[서비스 카테고리와 카피 원칙]]의 "쓰지 않는 표현"도 적용 대상이 아니다.
포털 랜딩과 앱 스위처에 jangmini 를 **넣지 않는다.**

> 이 표는 다음 세션이 jangmini 를 "일곱 번째 기록 앱"으로 보고 여섯 앱 규칙에
> 맞춰 "고치는" 것을 막기 위해 있다. 다르게 만든 것이 맞다.
>
> **2026-09-08 사용자 지시: "myjane의 디자인패턴을 입히려고 하지 말아주세요.
> 일단은 원작자의 디자인 패턴 그대로 사용해주세요."**

## ⚠️ myjane 쿠키가 여기까지 온다

myjane 세션 쿠키 `snap_user` 는 `.myjane.co.kr` 에 붙는다. jangmini 는 그 하위
도메인이라 **이 쿠키가 요청마다 자동으로 도착한다.**

**완전히 무시한다.** 이걸 보고 로그인 상태를 판단하는 코드를 쓰지 않는다.
jangmini 의 로그인 여부는 `jangmini_reader` 쿠키만으로 판단한다.

같은 이유로 jangmini 에는 **`NEXT_PUBLIC_COOKIE_DOMAIN` 을 두지 않는다.**

## 스택

Next.js 15.1.9 App Router · React 19 · Vercel AI SDK(`ai@4`) · OpenAI ·
mongoose 9 · MongoDB(`jangmini`) · Cloudflare R2 · Vercel `icn1`
로컬 개발 포트 **3006**, 검증 서버 **3016** (전체 표는 [[Home]]).
패키지 매니저는 **pnpm 12** (다른 앱은 npm 이다).

원본은 [toukoum/portfolio](https://github.com/toukoum/portfolio) 포크다.
**원본과 갈라진 지점** — 콘텐츠 하드코딩을 DB 조회로 바꿨고, 문서 페이지 ·
이미지 이관 · 답변 캐시 · 방어 계층 · reader 로그인 · 자체 admin 을 더했다.

DB 이름은 URI 경로가 아니라 코드에서 **`jangmini` 로 못 박는다** → [[MongoDB Atlas]]
**`MONGO_DB` 환경변수를 만들지 않는다.**

## 환경 변수

`.env.local` 에 있는 것 (값은 적지 않는다):

```
MONGO_URI  SESSION_SECRET  IP_HASH_SALT  OPENAI_API_KEY
SEED_READER_PASSWORD  NOTION_TOKEN
R2_ACCOUNT_ID  R2_ACCESS_KEY_ID  R2_SECRET_ACCESS_KEY  R2_BUCKET_NAME  R2_PUBLIC_URL
```

- **`MONGO_URI` 는 로컬만 표준 URI** 다. 이 PC 는 Node 에서 SRV 조회가 안 된다
  → [[MongoDB Atlas]]. **Vercel 에는 `mongodb+srv://`** 를 넣는다
- `CHAT_ENABLED` — **로컬 `.env.local` 에만 `true` 다.** Vercel 에는 아직 없어서
  운영 채팅이 503 이다. Phase 5 운영 검증 때 넣는다
- `SEED_READER_PASSWORD` 는 시드가 끝났으므로 지워도 된다(DB 에는 bcrypt 해시만)
- `GITHUB_TOKEN` · `OPENAI_VISION_MODEL` · `MONGO_DB` ·
  `NEXT_PUBLIC_COOKIE_DOMAIN` 은 **일부러 두지 않는다** (`.env.example` 에 이유가 있다)

## 명령

```
pnpm dev              3006
pnpm db:check         연결·DB 이름·컬렉션·인덱스 점검 (-- --ensure 로 생성)
pnpm r2:check         R2 권한 3단 + 실제 업로드 후 공개 URL 대조
pnpm seed             settings 기본값 · readers 초기 계정 (멱등)
pnpm fetch:notion     Notion → tmp-ingest/
pnpm fetch:resume     이력서 .docx → tmp-ingest/
pnpm ingest           정규화 결과 출력 (dry-run) · -- --write 로 적재
pnpm images:migrate   Notion 이미지 → R2 (dry-run) · -- --write
pnpm warm             추천 질문 답변 사전 생성 (dry-run) · -- --write --lang=ko
pnpm icons            avatar.png → 파비콘·애플 아이콘·OG
pnpm shots:upload     실환경 캡쳐 → R2 (dir prefix 인자 2개)
```

전체 재수집은 `fetch:notion` → `fetch:resume` → `ingest -- --write` →
`images:migrate -- --write` 순서다.

## 데이터

`jangmini` DB, 컬렉션 8개. 이름은 **모델 정의에서 명시한다**(아래 함정).

| 컬렉션 | 현재 | 무엇이 있나 |
|---|---|---|
| `portfolio` | **102건** | 수집한 콘텐츠. `kind` 로 구분 |
| `settings` | **12건** | key-value. 재배포 없이 한도·모델·킬스위치를 바꾼다 |
| `readers` | **1건** | `2xteam` (role=admin · 무기한 · dailyCap 500) |
| `readers_history` | 26 | 질문 이력 **겸 카운터**. TTL 90일 |
| `login_attempts` | 7 | 로그인 실패. TTL 1시간 |
| `suggestions` | **16** | 추천 질문. 문장 원본은 `src/lib/suggestions.ts` |
| `answers` | **16** | 사전 생성 답변 캐시 (ko, sourceVersion 1) |
| `usage` | 1 | 일별 집계 — 전역 캡 판정. `_id` 가 KST 날짜 |

### portfolio 102건의 내역

```
project 46 · skill 34 · activity 8 · essay 4 · experience 4 ·
certificate 3 · education 2 · profile 1
```

`kind` — `profile` `experience` `project` `skill` `education` `certificate`
`activity` `essay`

`visibility` 는 전부 `public` 이다(2026-09-08 사용자 지시 "모두 공개").
필드 자체는 남겨 뒀고 조회는 항상 `visibility:'public'` 을 건다 —
비공개가 필요해지면 플래그만 뒤집으면 된다.

`featured: true` **7건** — `/resume` 에 싣는 대표 프로젝트.
`ignite-architecture` · `snapapps` · `tracx-ai-agent` · `inquiry-ticket-admin` ·
`wms-pda-scanner` · `sirloin-oms` · `aspnet-fe-be-split`

> **기여도로는 대표작을 고를 수 없다.** 「기여도 ≥ 0.9 또는 2024년 이후」로
> 뽑으면 28건이고 2011~2012 박물관 프로젝트가 전부 1.0 으로 들어온다 —
> 1인 작업이라 당연히 1.0 이다. 기여도는 "혼자 했는가"를 재는 값이지
> "대표작인가"를 재지 않는다. 그래서 사람이 고른 목록을 `scripts/ingest.mjs`
> 의 `FEATURED` 에 둔다. Admin 토글이 생기면 그때 DB 로 원본을 옮긴다.

### 수집 원본과 충돌 규칙

| 원본 | 무엇을 가져오나 |
|---|---|
| Notion DB `포트폴리오 이력` | 프로젝트 47행 → 실제 45건. 본문 4단(개요/역할/성과/회고) |
| Notion DB `포트폴리오 리소스` | 스킬 18행 (숙련도 있음) |
| Notion 루트 페이지 | 프로필 · 회사 4곳 경력 |
| 이력서 docx | 학력 2 · 자격증 3 · 대외활동 8 · 자기소개 4 · 스킬 16 (숙련도 없음) |

**프로젝트 본문은 Notion 이 원본**, 이력서 부가정보는 **docx 가 원본**.
스킬은 **둘을 합친다**(Notion 18 ∪ 이력서 16 = 34) — 리소스 DB 가 이력서보다
오래돼서 합치지 않으면 스킬이 실제보다 적게 보인다.

이력서 원본 경로는 `C:\Users\Tracxlogis\Downloads\newenw\` 이고
`.env.local` 의 `RESUME_DOCX` 로 바꿀 수 있다. 사본이
`tmp-ingest/resume.docx` 에도 있다 — 원본이 사라져 막힌 일을 겪었다.

### Notion 에 없는 프로젝트는 손으로 등록한다

Notion 포트폴리오 DB 에 없는 프로젝트는 `content/projects-manual.json` 에
적는다. `ingest` 가 `source.type: 'manual'` 로 적재하므로 **Notion 재수집이
이 항목을 덮지 않는다.** FitLog 이 첫 사례다 — 2026-09 에 시작해서 Notion
DB 에 항목이 없고, 그래서 재수집마다 사라질 자리였다.

넣은 뒤에는 프로젝트 `order` 를 `period.start` 내림차순으로 **전체** 다시
매긴다. 손등록 항목만 뒤에 붙이면 가장 최신 프로젝트가 목록 끝으로 간다.

리포트도 `order` 로 정렬한다. 삽입 순서로 찍으면 손등록 항목이 맨 끝에
보여서 "안 들어갔나" 하고 오해한다 — 실제로 한 번 그랬다.

### 이미지

R2 버킷 `jangmini` 에 **오브젝트 82개 · 약 42MB**.
Notion 에서 온 것은 `projects/<slug>/<n>-<hash>.<ext>` (프로필은
`profile/profile/...`), 손으로 찍은 캡쳐는 `projects/<slug>/<파일명>-<hash>.<ext>`
— 어느 화면인지 URL 로 알 수 있게 파일명을 남긴다.
공개 URL 은 `https://pub-b6ba1f5f07ef4f8c9be1a32f6beccf5c.r2.dev`.

`next.config.ts` 의 `remotePatterns` 에 이 호스트를 **적어 뒀다** —
환경 변수로 읽지 않는다(빌드 시점 평가라 값이 없으면 조용히 빈 배열이 된다).

## 라우트

```
/                  랜딩 (챗 입력 + 빠른 질문 5개 + 우상단 햄버거 메뉴)
/chat              챗 화면. `?q=<key>` 는 추천 질문(캐시 히트), `?query=<문장>` 은 자유 입력
/admin             자체 admin. 로그인 → 비밀번호 재확인 → 콘솔 4탭
/resume            문서형 이력서. 대표 8건. 01~05 번호 절
/projects          46건 아카이브. ?tech= 로 필터 (태그 32종)
/projects/[slug]   개요/역할/성과/회고 + 이미지 갤러리. 46건 정적 생성
/faq               추천 질문 16개 + 상단에 문서 진입점 3개
/api/chat          POST. 로컬은 열려 있고 **운영은 CHAT_ENABLED 미설정으로 503**
/api/reader/*      login · logout · me · stepup
/api/admin/*       settings · readers · usage · answers (requireAdmin)
```

## tool 6개

`getPresentation` `getExperience` `getProjects` `getSkills` `getResume` `getContact`

> ⚠️ tool 을 더하거나 지울 때 **세 군데를 함께** 고친다. 하나라도 빠지면
> 런타임 에러다 —
>   ① `src/app/api/chat/route.ts` 의 `tools` 객체
>   ② `src/components/chat/tool-renderer.tsx` 의 `case`
>   ③ 해당 컴포넌트 파일

**원본과 다른 점** — 원본 tool 은 `"Here are all the projects made by Raphael
(above)!"` 같은 문자열만 돌려주고 컴포넌트가 하드코딩 배열을 그렸다. 그래서
**모델이 프로젝트를 전혀 몰랐다.** 지금은 tool 이 DB 를 조회해 데이터를
돌려주므로 "React Native 쓴 프로젝트가 뭐야" 에 답할 수 있다.

다만 **본문은 담지 않는다** — tool 반환값이 모델 컨텍스트로 들어가고 45건
본문 전체는 수만 토큰이다. `getProjects` 는 기본이 대표 7건이고
`tech`/`company`/`all` 로 좁힌다.

## 비용 방어 — 순서가 중요하다

`src/app/api/chat/route.ts` 의 처리 순서다.

| | 층 | 무엇 |
|---|---|---|
| ① | 킬스위치 | `CHAT_ENABLED` **환경 변수**. DB 와 무관하게 닫힌다 |
| ② | L5 | Origin 검증. 스크립트 직격을 걸러낸다 |
| ③ | L1 | system 역할 거부 · 글자 수 · 메시지 개수 |
| ④ | **L0** | **사전 생성 답변 캐시.** 맞으면 여기서 끝 |
| ⑤ | L2·L3 | 익명 제한 · 전역 일일 캡 |
| ⑥ | | OpenAI |
| ⑦ | | `readers_history` · `usage` 기록 |

**④가 ⑤보다 먼저다.** 캐시 히트는 OpenAI 를 부르지 않으므로 비용이 0이다.
거기에 제한을 걸면 돈이 들지 않는 요청을 막는 셈이고, 방문자는 문서를 보러
갈 이유가 없는데도 막힌다. 검증에서 실제로 이 순서를 확인했다 — 추천 질문은
전역 캡을 1로 내려도 통과하고, 자유 질문은 429 가 된다.

**L4(OpenAI 프로젝트 하드 리밋)는 아직 비어 있다** — 키를 다른 앱과 공유하고
있다. 익명 제한(L2)은 시크릿 창이면 초기화되므로, 실제 방어선은 L3 과 L4 다.

한도 값은 `settings` 에서 읽고 60초 메모이즈한다. Admin 에서 바꾸면
`invalidateLimits()` 로 **즉시** 반영된다(검증됨).

## 로그인과 admin

reader 로그인은 **게이트가 아니라 제한 해제**다. 익명도 사이트를 전부 본다.

- 쿠키 `jangmini_reader` — HMAC-SHA256 서명, **호스트 전용**, HttpOnly,
  admin TTL 2시간 / reader 30일
- 실패 문구를 아이디 유무로 나누지 않고, 계정이 없어도 bcrypt 를 한 번 돌린다
  (응답 시간 차이로 아이디 존재를 알 수 있다)
- 로그인 시도 IP 당 15분 5회 — 초기 비밀번호가 4자리다
- `/admin` 은 **로그인 + 10분 내 비밀번호 재확인**이 있어야 열린다.
  admin 을 `readers.role` 로 구분하므로 reader 비밀번호가 곧 admin 권한이다
- `requireAdmin()` 을 `src/lib/admin-auth.ts` 한 곳에 모았다 — 라우트마다
  각자 검사하면 새 라우트에서 빠뜨려도 아무 에러가 없다
- 계정 API 는 `passwordHash` 를 내보내지 않고, 사용량 API 는 `clientId`·
  `ipHash` 를 내보내지 않는다(화면에 뜨면 그게 곧 로그다)
- **새 계정은 비밀번호 8자 이상**을 요구한다. 초기 계정이 4자리인 것은 사용자
  지시였지만 그 값을 새 계정에 물려주지 않는다

## 결정 사항

- **AI 채팅은 부가 기능이고 DB 브라우징이 본체다.** 원본은 채팅만 있어서
  채팅이 막히면 사이트가 죽는다. 채팅 답변은 훑을 수도, Ctrl-F 할 수도,
  검색엔진이 색인할 수도 없다
- **`/resume` 에는 대표만 싣는다.** 45건을 한 스크롤에 넣으면 TracX AI
  Agent(2026)가 게시판 플렛폼(2014) 옆에 묻힌다
- **`/projects` 필터는 URL 쿼리다.** 클라이언트 상태로 두면 필터를 걸어 놓은
  화면을 링크로 보낼 수 없고 검색엔진도 필터된 목록을 보지 못한다
- **기술 스택 연대기가 이 데이터의 가장 강한 패다.** 45건 전부에 기간과 태그가
  있어서 "2012~2022 ASP.NET 풀스택 → 2023 프론트엔드 전환 → 2026 AI·Next.js"
  라는 14년 서사가 문장 없이 한 표에 보인다. 조사한 레퍼런스 12곳 중 이걸
  하는 곳은 없었다. `/resume` 의 04 절에 목록보다 **먼저** 둔다
- **회고가 47/47 에 있다.** "무엇을 만들었다" 가 아니라 "무엇을 배웠다" 라서
  판단력을 보여준다. 프로젝트 상세에서 접지 않고 펼쳐 둔다
- **프롬프트에 사실을 적지 않는다.** 박아 두면 노션을 고쳐도 옛 이야기를 계속
  한다. 말투와 규칙만 담고 사실은 tool 로 가져온다
- **인젝션 가드는 프롬프트 맨 뒤에** 둔다. 앞에 두면 tool 이 돌려준 수천 자에
  묻힌다 → [[AI 채팅 패턴]]
- **캐시 키는 자유 입력이 아니라 고정 `key`** 다 (`src/lib/suggestions.ts`).
  입력 텍스트를 해시하면 "경력이 어떻게 되나요"와 "경력 어떻게 되세요"가 다른
  키가 되어 거의 안 맞고, 방문자가 조금씩 다른 문장으로 캐시를 채울 수 있다
- **JSON Resume 테마는 쓰지 않는다.** 그 스키마에는 회고·기여도·4단 본문이
  들어갈 자리가 없다. 테마 교체의 이점보다 데이터 손실이 크다
- **admin 은 `readers.role`** 로 한다(사용자 결정). 4자리 비밀번호가 admin
  권한까지 여는 셈이므로 `/admin` 진입 시 **비밀번호 재확인** + admin 세션
  **TTL 2시간**(reader 는 30일) + 로그인 시도 **IP당 15분 5회 잠금**
- **reader 는 무제한이지만 `dailyCap` 은 남긴다.** 계정이 뚫려도 무한 과금은 막는다
- **익명 제한은 완벽할 수 없다**(사용자도 지적). 시크릿 창이면 `clientId` 가
  초기화된다. **전역 일일 캡과 OpenAI 프로젝트 하드 리밋이 실제 방어선**이다

## 함정

### ⚠️ 컬렉션 이름은 모델 정의에서 명시한다

mongoose 는 모델 이름을 복수로 바꿔 컬렉션 이름을 만든다. 이 앱은 이름이
섞여 있어 자동 규칙으로 다 맞출 수 없다.

| 모델 | 자동 | 실제로 쓸 이름 |
|---|---|---|
| `Portfolio` | `portfolios` | **`portfolio`** |
| `Reader` | `readers` | `readers` ✅ 우연히 맞는다 |
| `ReaderHistory` | `readerhistories` | **`readers_history`** |

어긋나면 **조용히 새 컬렉션이 생기고** 조회만 0건이 된다.
`src/lib/db.ts` 의 `defineModel(name, schema, collection)` 로 전부 명시한다.

### ⚠️ Notion 이미지 URL 은 1시간 뒤 깨진다

`X-Amz-Expires=3600` 이다. 실측 — 발급 78분 뒤 **403 Forbidden**.
그래서 `migrate-images.mjs` 는 페이지마다 **Notion 에서 새 URL 을 받아 곧바로
내려받아 올린다.** 스냅샷에 담아 두고 나중에 올리는 구조로는 동작하지 않는다.
→ [[Cloudflare R2]]

### ⚠️ Notion 마크다운의 함정 4개

`/pages/{id}/markdown` 응답을 다루면서 겪은 것. 전부 오류를 내지 않았다.

1. **다단(`<columns>`) 안의 내용이 탭으로 들여써서 온다.** 태그만 지우면
   `\t\t### 회사명` 이 남아 `^###` 에 안 걸린다 — 루트 페이지의 회사 경력이
   전부 다단 안에 있어 **경력 파싱이 0건**이 됐다. 헤딩만 들여쓰기를 없앤다
   (불릿은 중첩 목록의 의미가 있어 건드리지 않는다)
2. **강조가 슬래시를 감싸 닫힌다** — `### **트랙스로지스 테크팀  /** 2025.09`
   라서 `\/\s*[0-9]` 로는 하나도 안 걸렸다
3. **이미지 alt 안에 링크가 중첩된다** —
   `![[*qoo10.com*](https://qoo10.com)](https://prod-files…)`.
   alt 를 `[^\]]*` 로 두면 첫 `]` 에서 끊겨 파편이 남고, 그 파편이 링크로
   잡혀 프로필 링크에 `[*qoo10.com*` 이 들어왔다
4. **파일 없는 이미지 블록을 `![]()` 로 준다.** 3번을 고치며 URL 을
   `([^)]*)` 로 0개 이상 허용했더니 이것까지 잡혀 R2 이관에서
   "Failed to parse URL from" 으로 3장이 실패했다

**앞의 수정이 뒤를 깨뜨린 경우도 있었다** — 1번을 고친 뒤 루트 페이지가
`## 반갑습니다!` 로 시작하게 되어 `split(/^##\s/m)[0]` 이 빈 문자열을 주고
`/resume` 의 "01 소개" 가 통째로 비었다.

### ⚠️ 가시성을 애니메이션에 의존시키면 안 된다

햄버거 메뉴를 `AnimatePresence` + `motion.nav` 로 만들었더니 **마운트는 되는데
화면에 안 보였다** — `initial`(opacity 0)에 멈추고 `animate` 가 끝까지 가지
않았다. `tw-animate-css` 의 `animate-in fade-in` 으로 바꿔도 같았다(그쪽도
시작이 opacity 0 이다).

**접근성 트리에는 링크가 다 보이고 `aria-expanded` 도 true 라서 "정상 동작"
으로 오판했다.** `read_page` 나 `find` 만으로는 못 잡는다 —
**computed `opacity`·`visibility`·`getBoundingClientRect` 를 함께 재야 한다.**

대응 — 등장 애니메이션을 없애 기본 상태를 보이는 상태로 뒀다.
이 저장소는 `framer-motion` 과 `motion` 두 패키지를 함께 들고 있다.

### `_` 로 시작하는 폴더는 라우팅에서 제외된다

`src/app/api/_diag/route.ts` 를 만들었더니 404 였다. Next App Router 의
private folder 규칙이다.

### pnpm 12 는 빌드 스크립트를 차단한다

`ERR_PNPM_IGNORED_BUILDS` 로 **설치가 실패한다**(`sharp` ·
`@tsparticles/engine`). 이 설정의 집이 `package.json` 의 `"pnpm"` 필드에서
`pnpm-workspace.yaml` 로 옮겨졌는데, 옮겨도 막혀서
`pnpm approve-builds --all --yes` 로 통과시켰다. 다른 앱은 npm 이라 처음 만난다.

### 원본에 남아 있던 것들

- ⚠️ **원작자의 서드파티 애널리틱스**(`datafa.st`, `data-domain="toukoum.fr"`)가
  살아 있어 **배포된 우리 사이트의 방문자 트래픽이 원작자 계정으로** 가고 있었다
- `.gitignore` 에 `.env` 만 있고 **`.env.local` 이 없었다**
- `next dev` 가 포트 3000 기본값이라 **myjane 포털과 충돌**한다
- `metadata.icons` 가 저장소에 없는 `/apple-touch-icon.svg` 를 가리켜 iOS 가 404
- `errorHandler` 가 내부 에러를 그대로 반환했다 — 배포된 사이트에서
  `t.unshift is not a function` 을 실제로 받았다
- `messages` 배열을 클라이언트가 통째로 보내서, `role:"system"` 을 끼워 넣으면
  **범용 GPT 프록시로 쓸 수 있었다.** 500자 제한으로는 안 막힌다
- `GITHUB_TOKEN` 은 `toukoum/portfolio` 의 star 수를 읽는 용도였고 **호출하는
  곳이 이미 없었다** — 그래서 필요 없다

### `strongmin.notion.site` 는 공개돼 있다 — WebFetch 로는 안 보인다

각 페이지에 `public_url` 이 살아 있어 링크 공유가 된다. 그런데 notion.site 는
JS SPA 라 **WebFetch 로 받으면 껍데기만 온다.** "비공개"로 오판하기 쉽다.

### Notion MCP 의 데이터소스 조회가 깨져 있다

`query-data-source` · `retrieve-a-data-source` 가 `invalid_request_url` 로
실패한다(2026-09-08). **공개 REST API 는 문제없다** —
`POST /v1/databases/{id}/query` 와 `GET /v1/pages/{id}/markdown` 둘 다 정상.
스크립트는 REST 를 쓴다(`NOTION_TOKEN`).

## 진행 상황

- [x] Phase 0 — 수집 파이프라인, `portfolio` 101건, 컬렉션 8개, 시드
- [x] Phase 1 — 포크·저장소·Vercel·도메인. `vercel.json` 에 `icn1`+`nextjs`
- [x] Phase 2 — 원작자 흔적 제거 (public 57개 · 컴포넌트 6개 · 애널리틱스)
- [x] Phase 3 — tool 6개를 DB 조회로, 페르소나 재작성
- [x] Phase 4 — `/resume` `/projects` `/projects/[slug]` `/faq`
- [x] 이미지 77장 R2 이관, 아바타·파비콘·OG, 햄버거 메뉴, 한글화
- [x] FitLog 손등록 + 실환경 캡쳐 5장 (myjane 통합 로그인은 부가 작업으로)
- [x] Phase 5 — 답변 캐시 · 방어 계층 · reader 로그인 · admin (로컬 검증 통과)
- [ ] **Vercel 환경 변수 등록 + `CHAT_ENABLED=true` → 운영 검증**
      → [[D jangmini 구축]]

## 관련

[[D jangmini 구축]] · [[MongoDB Atlas]] · [[Cloudflare R2]] ·
[[Vercel 배포 패턴]] · [[AI 채팅 패턴]] · [[도메인과 DNS]]
