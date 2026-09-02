---
title: SnapNote
type: project
status: 운영중
domain: snapnote.myjane.co.kr
repo: https://github.com/2xteam/snapnote
local: C:\Dev\SnapNote
branch: master
db: math
tags: [project, myjane-family]
updated: 2026-09-02
---

# SnapNote

틀린 문제를 찍어 모노톤으로 정리해 나만의 오답노트를 만드는 앱.

> ⚠️ 기본 브랜치가 **`master`** 다 (`main` 아님). Vercel Production Branch도 master.

## 스택

SnapWord와 동일 (Next.js 15 · MongoDB · OpenAI · Vercel `icn1`).
이미지 저장은 Cloudflare R2 (`snapnote-uploads` 버킷).

## 데이터

| 위치 | 내용 |
|---|---|
| `math` DB | `wrong_notes` `wrong_items` `folders` `chat_threads` `notices` `events` `inquiries` `openai_request_logs` |
| `user` DB | `users` — 세 앱 공용 |

## 이미지 처리 흐름

촬영 → `ImageCropper`(크롭) → `MonoAdjust`(임계값 이진화) → R2 업로드.

이진화 PNG는 잡음이 많을수록 커진다. 긴 변 2400px로 제한하고, 3.5MB를 넘으면
1800 → 1400px로 자동 재인코딩한다. 형식은 PNG 유지 — 이진 이미지는 JPEG보다 PNG가 작다.
→ [[이미지 업로드 패턴]]

## 현재 상태

- [x] Vercel 이관 · 도메인 · DB · 메일(PIN 찾기) 정상
- [x] 하단 공통 푸터
- [ ] `NEXT_PUBLIC_COOKIE_DOMAIN` Config 재등록
