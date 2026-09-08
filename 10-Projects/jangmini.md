---
title: jangmini
type: project
status: 계획
domain: jangmini.myjane.co.kr
repo: https://github.com/2xteam/jangmini
local: C:\Dev\jangmini
branch: main
db: jangmini
tags: [project, portfolio]
updated: 2026-09-08
---

# jangmini

방문자가 물으면 **AI가 내 이력서를 근거로 답하는 포트폴리오.** 이름은 "장민"에
제미나이를 얹은 것이다.

> **2026-09-08 현재 계획 단계다.** 저장소는 비어 있고 코드가 없다.
> 확정된 것만 "결정 사항"에 적고 나머지는 체크박스로 둔다.
> 단계별 진행 계획은 [[D jangmini 구축]]에 있다.

## 여섯 앱과 다른 점 — 먼저 읽을 것

**myjane 여섯 앱의 규칙을 따르지 않는다.** 성격이 다른 앱이다.

| 항목 | 여섯 앱 | jangmini |
|---|---|---|
| 회원 | `user` DB 공유 | **공유하지 않는다** — `jangmini` DB 의 `reader` |
| 세션 쿠키 | `snap_user` · `.myjane.co.kr` | `jangmini_reader` · **호스트 전용** |
| 로그인의 뜻 | 서비스를 쓰려면 필요 | **제한 해제일 뿐** — 익명도 전부 볼 수 있다 |
| admin | 포털 통합 admin | **이 앱 자체 라우트** `/admin` |
| 디자인 | 여섯 앱 디자인 시스템 · 결쩜사 패턴 | 포크 기반. **따르지 않는다** |
| 카피 원칙 | 공부·건강·습관·성향 기록 | 해당 없음 — 개인 포트폴리오다 |

[[서비스 카테고리와 카피 원칙]]의 "쓰지 않는 표현"도 적용 대상이 아니다.
포털 랜딩과 앱 스위처에 jangmini 를 **넣지 않는다.**

> 이 표는 다음 세션이 jangmini 를 "일곱 번째 기록 앱"으로 보고 여섯 앱 규칙에
> 맞춰 "고치는" 것을 막기 위해 있다. 다르게 만든 것이 맞다.

## ⚠️ myjane 쿠키가 여기까지 온다

myjane 세션 쿠키 `snap_user` 는 `.myjane.co.kr` 에 붙는다. jangmini 는 그 하위
도메인이라 **이 쿠키가 요청마다 자동으로 도착한다.**

**완전히 무시한다.** 이걸 보고 로그인 상태를 판단하는 코드를 쓰지 않는다.
jangmini 의 로그인 여부는 `jangmini_reader` 쿠키만으로 판단한다.

같은 이유로 jangmini 에는 **`NEXT_PUBLIC_COOKIE_DOMAIN` 을 두지 않는다.**
두면 reader 쿠키가 `.myjane.co.kr` 로 퍼져 여섯 앱에 전송된다.

## 스택

Next.js 15 App Router · React 19 · Vercel AI SDK · OpenAI · MongoDB(`jangmini`) · Vercel `icn1`
로컬 개발 포트 **3006**, 검증 서버 **3016** (전체 표는 [[Home]]).

원본은 [toukoum/portfolio](https://github.com/toukoum/portfolio) 포크다.
**원본과 갈라지는 지점** — 콘텐츠가 하드코딩(`src/components/projects/Data.tsx`,
각 tool 의 반환 문자열)이던 것을 `jangmini` DB 조회로 바꾼다. 원본에 없는 것:
reader 로그인 · admin · 브라우징 페이지 · 답변 캐시 · rate limit · 예산 캡.

DB 이름은 URI 경로가 아니라 코드에서 **`jangmini` 로 못 박는다** → [[MongoDB Atlas]]
**`MONGO_DB` 환경변수를 만들지 않는다.** 그 앱의 고정된 사실이다.

## 데이터

`jangmini` DB, 컬렉션 7개. 이름은 **단수로 쓴다**(아래 함정 참고).

| 컬렉션 | 무엇이 있나 |
|---|---|
| `portfolio` | Notion·이력서에서 수집한 콘텐츠. `kind` 로 구분 |
| `settings` | key-value 설정. **재배포 없이** 한도·모델·킬스위치를 바꾼다 |
| `reader` | 관리자가 수동 발급하는 계정. 만료일 있음. `role` 로 admin 겸용 |
| `reader_history` | 질문 이력 **겸 카운터** |
| `suggestions` | 추천 질문 (고정 `key`) |
| `answers` | 사전 생성 답변 캐시 |
| `usage` | 일별 집계 — 전역 예산 캡 판정용 |

`portfolio.kind` — `profile` `experience` `project` `skill` `education`
`certificate` `activity` `essay`

`portfolio.visibility` 가 안전장치다. 연락처·급여·재직 중 내부 정보는 `private` 로
넣고 **공개 조회 쿼리가 집지 않게** 한다. 프롬프트로만 막으면 언젠가 새 나간다.

카운터를 `reader_history` 로 겸하는 이유 — 별도 카운터 컬렉션을 두면 "제한에
걸렸는데 무슨 질문이었는지 모른다"가 된다. 대신 인덱스가 필수다:
`{clientId,createdAt}` · `{ipHash,createdAt}` · `{readerId,createdAt}`.

IP 는 **원문을 저장하지 않고 `sha256(ip+salt)` 만** 둔다.

### 수집 원본

| 원본 | 무엇을 가져오나 |
|---|---|
| Notion DB `포트폴리오 이력` (48행) | 프로젝트. `이름`·`기간`·`기술스택`·`기여도` + 본문 4단(개요/역할/성과/회고) |
| Notion DB `포트폴리오 리소스` (19행) | 스킬. `분류`(소프트웨어·전문적 지식·언어)·숙련도 |
| Notion 루트 페이지 | 프로필·회사 4곳 경력·Side Project 3건 |
| 이력서 docx | Notion 에 **없는 것** — 학력 상세·자격증 취득일·대외활동 9건·병역·자기소개 |

**충돌 규칙** — 프로젝트 본문은 **Notion 이 원본**(4단 구조가 일정하고 최신),
이력서 부가정보는 **docx 가 원본**. `source.type` 으로 구분한다.

## 결정 사항

- **AI 채팅은 부가 기능이고 DB 브라우징이 본체다.** 원본은 채팅만 있어서 채팅이
  막히면 사이트가 죽는다. `/projects` `/resume` `/skills` `/faq` 를 따로 둬서,
  예산 캡에 걸렸을 때 보낼 곳이 있게 한다. 검색 색인과 지원서 링크도 여기서 나온다
- **추천 질문의 답변은 미리 구워 DB에 넣는다.** 첫 방문자에게 캐시 미스를 떠넘기지
  않고, 채용 담당자가 볼 확률이 가장 높은 답변을 내가 검수한 문장으로 고정한다
- **캐시 키는 자유 입력이 아니라 고정 `key`(slug)** 다. 입력 텍스트를 해시하면
  "경력이 어떻게 되나요"와 "경력 어떻게 되세요"가 다른 키가 되어 거의 안 맞고,
  악의적 방문자가 조금씩 다른 문장으로 **내 DB를 채울** 수 있다
- **admin 은 `reader.role`** 로 한다(2026-09-08 사용자 결정). 로그인 폼과 컬렉션이
  하나라 단순하다. 대신 4자리 비밀번호가 admin 권한까지 여는 셈이므로
  `/admin` 진입 시 **비밀번호 재확인**과 **admin 세션 TTL 2시간**(reader 는 30일)을 둔다
- **reader 는 무제한이지만 `dailyCap` 은 남긴다**(2026-09-08 사용자 결정).
  정상 사용에는 걸리지 않는 값이고, 계정이 뚫려도 무한 과금은 막는다
- **익명 제한은 완벽할 수 없다**(사용자도 지적). 시크릿 창이면 `clientId` 가
  초기화된다. 그래서 **전역 일일 캡과 OpenAI 프로젝트 하드 리밋이 실제 방어선**이고,
  익명 제한은 "실수로 많이 쓰는 것"을 막는 용도다

## 함정

### ⚠️ mongoose 가 컬렉션 이름을 복수로 바꾼다

`mongoose.model("Reader", schema)` 는 컬렉션을 **`readers`** 로 만든다.
Atlas 에 손으로 만든 `reader` 와 어긋난 채 **조용히 새 컬렉션이 생긴다.**
오류도 경고도 없고 조회만 0건이 된다.

**세 번째 인자로 이름을 명시한다.**

```ts
mongoose.model("Reader", schema, "reader");
```

`portfolio` · `settings` · `reader` · `reader_history` · `suggestions` ·
`answers` · `usage` 전부 해당한다.

### ⚠️ Notion 이미지 URL 은 1시간 뒤 깨진다

Notion API 가 주는 이미지 URL 은 S3 presigned 이고 **`X-Amz-Expires=3600`** 이다.
그대로 DB 에 저장하면 **한 시간 뒤 전부 깨진다.**

수집 시점에 **즉시 내려받아 R2 로 재호스팅**하고 영구 URL 로 치환한다
→ [[이미지 업로드 패턴]] · [[Cloudflare R2]]

### Notion MCP 의 데이터소스 조회가 깨져 있다

`query-data-source` · `retrieve-a-data-source` 가 `invalid_request_url` 로 실패한다
(2026-09-08 확인). DB 행을 `get-block-children` 으로도 못 가져온다.

**우회 경로** — `post-search`(object=page)로 페이지 목록을 받아 `parent.database_id`
로 걸러내고, 각 행은 `retrieve-page-markdown` 으로 본문을 읽는다. 69개 전부 읽힌다.

### `strongmin.notion.site` 는 공개돼 있다 — WebFetch 로는 안 보인다

각 페이지에 `public_url` 이 살아 있어 링크 공유가 된다. 그런데 notion.site 는
JS SPA 라 **WebFetch 로 받으면 껍데기만 온다.** "비공개"로 오판하기 쉽다.
내용을 읽을 때는 MCP 를 쓰고, 공유할 때만 `public_url` 을 쓴다.

### 빈 저장소에 Vercel 프로젝트를 먼저 만들지 않는다

`2xteam/jangmini` 는 지금 ref 가 0개다. 이 상태로 Vercel 프로젝트를 만들면
프리셋이 `Other` 로 잡혀 **빌드는 성공하는데 `/` 가 404** 다.
코드를 push 한 뒤에 만들고, `vercel.json` 에 `"framework": "nextjs"` 를 적는다
→ [[Vercel 배포 패턴]] (TypeLog 에서 겪은 것과 같다)

## 진행 상황

- [ ] Phase 0 — Notion·docx 수집 스크립트, 스키마, `portfolio` 적재
- [ ] Phase 1 — 포크·저장소 push·Vercel 프로젝트·OpenAI 전용 키와 하드 리밋
- [ ] Phase 2 — 원작자 개인정보·에셋 제거
- [ ] Phase 3 — tool 을 DB 조회로 교체
- [ ] Phase 4 — 브라우징 페이지 · FAQ · reader 로그인 · admin
- [ ] Phase 5 — 페르소나 · 7층 방어 · 메타데이터 · 배포 · 검증
- [ ] 가비아 `jangmini` CNAME 연결 → [[도메인과 DNS]]

## 관련

[[D jangmini 구축]] · [[MongoDB Atlas]] · [[Vercel 배포 패턴]] · [[AI 채팅 패턴]] · [[Cloudflare R2]]
