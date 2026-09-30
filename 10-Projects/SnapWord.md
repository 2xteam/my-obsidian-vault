---
title: SnapWord
type: project
status: 운영중
domain: snapword.myjane.co.kr
repo: https://github.com/2xteam/snapword
local: C:\Dev\SnapWord
branch: main
db: vocab
tags: [project, myjane]
updated: 2026-09-30
---

# SnapWord

교재·화면을 찍으면 OpenAI Vision이 단어를 뽑아 단어장을 만들어 주는 학습 앱.

## 스택

Next.js 15 (App Router) · React 19 · TypeScript · MongoDB Atlas · OpenAI · Vercel(`icn1`)
서체·테마는 CSS 변수 기반 다크/라이트/네온핑크/커스텀 4종 ([[앱 공통 UI와 아이콘]]).

## 데이터

| 위치 | 내용 |
|---|---|
| `vocab` DB | `vocabularies`(`language`: `en`·`hanja`) `words` `folders` `study_records` `test_sessions` `test_results` `chat_threads` `notices` `events` `inquiries` `ai_cache` `openai_request_logs` |
| `user` DB | `users` — **세 앱 공용** ([[인증과 세션 공유]]) |

## 주요 기능

- 사진 → 단어 추출 (`/api/openai-vision`), 텍스트 붙여넣기 분석 (`/api/analyze-text`)
- 단어장·폴더·학습·테스트·오답 관리·인쇄
- AI 상담 채팅 (`FloatingChat` + `/api/chat/threads`)
- 토큰 차감 (Vision 1회 10토큰, `lib/useToken.ts`)
- 관리자 화면 `/admin` — 회원·통계·문의·공지

## 이관 이력

2026-09-01 Cloudways → Vercel. 자세한 내용은 저장소 `docs/VERCEL_MIGRATION.md`.
핵심은 [[Vercel 배포 패턴]]과 [[이미지 업로드 패턴]]에 정리했다.

## 현재 상태

- [x] Vercel 이관 · 도메인 연결 · DB·메일 정상
- [x] 하단 공통 푸터 (MyJane 링크)
- [ ] `NEXT_PUBLIC_COOKIE_DOMAIN`을 **Config 타입**으로 재등록 → 세션 공유 활성화

## AI 채팅

스트리밍·되묻기·이어질 질문·스켈레톤 — 세 앱이 같은 구조를 쓴다
→ [[AI 채팅 패턴]]

## 2026-09-02 변경

- 앱 아이콘을 **게이지 호 스타일**로 교체 (`public/app-icon.svg`가 원본)
  → [[앱 공통 UI와 아이콘]]
- 푸터의 `MyJane` 평문 링크를 워드마크 `my`+`jane`으로 교체.
  `jane`은 브랜드 보라 고정 — 이 앱의 액센트(민트)를 쓰면 초록색이 된다
- 네온핑크 테마를 FitLog와 같은 **보라 팔레트로 교체하고 기본 테마로** 삼음
  (`data-theme="violet"` 재사용, 검정+민트 다크는 선택지로 유지)
- `dbName`을 코드에서 명시 — URI 경로에 의존하지 않는다 → [[MongoDB Atlas]]
- 상단 네비를 주 기능(`Home` `Folders` `Print`)만 남기고
  부가 기능(`My` `Notice` `Q&A` `Logout`)은 `More ▾`로 접었다.
  모바일 햄버거 메뉴에서는 구분선 아래로 내려간다

## 색

여섯 앱이 **먹청 팔레트**를 공유한다. 원본은 `myjane/design/palette.json` 하나다.
색을 바꿀 때는 거기만 고치고 myjane 에서 `npm run palette -- --write` 를 돌린다.
`app/palette.css` 는 생성 파일이라 직접 고치지 않는다 → [[먹청 톤 팔레트]]

## 학습 언어 — 영어 · 한자 (2026-09-30)

기본은 영어. 화면 문구는 계속 한국어이고, 바뀌는 건 **공부하는 대상**뿐이라 i18n 은 쓰지 않는다.

| 자리 | 어떻게 |
|---|---|
| 단어장 | `VocabularyDeck.language` = `en` \| `hanja`. **필드가 없는 옛 단어장은 영어** — 조회는 `{ language: { $ne: "hanja" } }` |
| 홈 | 상단 `English` / `漢字` 탭. 고른 값은 **브라우저 localStorage** 에만 둔다 (공용 `users` 컬렉션은 여섯 앱이 쓰므로 건드리지 않는다) |
| 한자일 때 홈 | 오늘의 Word(Merriam-Webster) · English Grammar 를 **요청조차 하지 않는다**. 자리에 네이버 한자사전 카드 |
| 추출 | 화면은 `vocabId` 만 보내고, 서버가 **내 단어장**에서 언어를 읽어 한자 지침을 붙인다 (`lib/deckLanguage.ts`) |
| 한자 출력 | 키는 영어와 같다. 훈·음은 칸을 따로 두지 않고 `meaning` 맨 앞 — `"배울 학 — 배우다"` |
| 사전 | 단어에 한자(`\p{Script=Han}`)가 섞이면 `hanja.dict.naver.com/#/search?query=…&range=all`. 단어 글자로 고르므로 단어장이 섞이는 오답 화면에서도 맞다 |
| 로그인 | 앱 로컬 로그인은 없앴다. 로컬은 localhost:3000 포털 → [[인증과 세션 공유]] |
| 가이드 | `GuideStep.languages` — 영어 RSS 단계는 영어일 때만 |
| AI 채팅 | "영어·한자 학습 전용". 한자 질문은 훈·음·부수·한자어로 답한다 |

원본 코드: `lib/studyLanguage.ts`(순수 함수 — 서버도 쓴다) · `lib/useStudyLanguage.ts`(클라이언트 훅).
`"use client"` 파일을 라우트가 import 하지 않도록 둘로 나눴다.

### 2026-09-30 추가 분리

| 자리 | 어떻게 |
|---|---|
| 채팅 | 영어·한자 **지시문을 따로**(`lib/chatOpenAi.ts`). RAG 청크마다 `language`. `ChatThread.language` 로 대화방이 갈리고 목록·칩·첫 안내·제목 짓기가 언어별. 카드의 "AI에게 질문" 은 그 단어장 언어로 |
| AI 캐시 | 한자는 키 앞에 `hanja:` (`aiCacheKeyFor`). 영어 키는 그대로라 옛 캐시가 맞는다 |
| 사진 추출 | 한자는 `gpt-4o` · 인쇄된 훈음 기준 · `cellCount` 로 이어 받기 · 한자만 남기기 → [[OpenAI Vision 추출 패턴]] |
| 칸 이름 | `WORD_FIELD_LABELS` — 한자·훈음·뜻·한자어·예문·유의자·반의자. 저장 키는 같고 보이는 이름만 |
| 글자 크기 | 한자가 섞인 단어는 **2배**(`wordFontSize`) — 학습·오답·시험 보기·점수·인쇄·입력 |
| 폴더 | `Folder.language`(기본값 없음). 옛 폴더는 **안의 단어장으로 짐작**, 둘 다 있으면 양쪽에 보인다(`lib/folderLanguage.ts`). 폴더 목록에도 탭 |
| 빈 단어장 | 단어가 없으면 Study · Test · Score 를 감춘다 |
| 사전 | 한자사전 카드는 검색어 없이 첫 화면. 사전 링크는 `window.open` 으로 **언제나 새 창** |

여러 단어장이 섞이는 화면(오답 복습·인쇄)은 단어 글자(`\p{Script=Han}`)로 언어를 고른다.

남은 것:
- [ ] 한자 전용 시험 유형 (훈음 → 글자, 글자 → 음) — `TEST_RESULT_TYPES` 확장
- [ ] 예문 문제에서 정답 글자 가리기 (한자에서 특히 드러난다)
- [ ] 오늘의 한자 (교육용 기초한자 목록을 앱에 두는 안)
- [ ] 폴더 옮기기 — 영어·한자가 섞인 옛 루트 폴더("My")를 가를 수단
- [ ] 이체자(亻·忄·扌)를 인쇄된 대로 둘지
