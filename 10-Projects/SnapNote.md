---
title: SnapNote
type: project
status: 운영중
domain: snapnote.myjane.co.kr
repo: https://github.com/2xteam/snapnote
local: C:\Dev\SnapNote
branch: master
db: math
tags: [project, myjane]
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

## 2026-09-02 변경

- 앱 아이콘을 **게이지 호 스타일**로 교체 (`public/app-icon.svg`가 원본)
  → [[앱 공통 UI와 아이콘]]
- 푸터의 `MyJane` 평문 링크를 워드마크 `my`+`jane`으로 교체.
  `jane`은 브랜드 보라 고정 — 이 앱의 액센트(민트)를 쓰면 초록색이 된다
- 상단 네비를 주 기능(`Home` `Folders` `Print`)만 남기고
  부가 기능(`My` `Notice` `Q&A` `Logout`)은 `More ▾`로 접었다.
  모바일 햄버거 메뉴에서는 구분선 아래로 내려간다
