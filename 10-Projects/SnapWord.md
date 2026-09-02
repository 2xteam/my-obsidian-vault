---
title: SnapWord
type: project
status: 운영중
domain: snapword.myjane.co.kr
repo: https://github.com/2xteam/snapword
local: C:\Dev\SnapWord
branch: main
db: vocab
tags: [project, myjane-family]
updated: 2026-09-02
---

# SnapWord

교재·화면을 찍으면 OpenAI Vision이 단어를 뽑아 단어장을 만들어 주는 학습 앱.

## 스택

Next.js 15 (App Router) · React 19 · TypeScript · MongoDB Atlas · OpenAI · Vercel(`icn1`)
서체·테마는 CSS 변수 기반 다크/라이트/네온핑크/커스텀 4종 ([[MyJane 앱 패밀리 디자인]]).

## 데이터

| 위치 | 내용 |
|---|---|
| `vocab` DB | `vocabularies` `words` `folders` `study_records` `test_sessions` `test_results` `chat_threads` `notices` `events` `inquiries` `ai_cache` `openai_request_logs` |
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
