---
title: D jangmini 구축
type: plan
tags: [plan, jangmini, portfolio, openai]
updated: 2026-09-09
status: Phase 0~5 구현·로컬 검증 완료 · 운영 개방 대기
applies-to: [jangmini]
---

# D jangmini 구축

AI 챗봇형 포트폴리오 [[jangmini]] 를 0부터 배포까지. A·B·C 와 무관한 별개
트랙이고 여섯 앱 파일을 건드리지 않는다.

앱 자체의 사실(스택·데이터·결정·함정)은 [[jangmini]] 에 있다.
**이 문서는 순서와 남은 일만** 적는다.

## 지금 상태 — 2026-09-09

**Phase 0~5 구현과 로컬 검증이 끝났다.** 남은 것은 운영 개방뿐이다.

- 배포는 살아 있고 문서 페이지 넷이 동작한다
- **운영 채팅은 503** — Vercel 에 `CHAT_ENABLED` 가 없다 (의도한 상태)
- 로컬에서는 채팅이 열려 있고 검증을 전부 통과했다

```bash
cd C:/Dev/jangmini
git log --oneline -1     # ee6f72c 이어야 한다
pnpm db:check            # portfolio 101 · settings 12 · readers 1 · suggestions 16 · answers 16
pnpm r2:check            # 권한 3단 + 공개 URL 대조
pnpm dev                 # 3006
```

⚠️ **개발 서버가 떠 있으면 `pnpm build` 를 돌리지 않는다.** 같은 `.next/` 를
써서 개발 서버가 먹통이 된다. 타입은 `npx tsc --noEmit`.
→ [[개발 서버와 검증 환경]]

---

## 남은 일 — 운영 개방

### ① Vercel 환경 변수

프로젝트 → Settings → Environment Variables. **배포 전에 전부 넣는다** —
환경 변수는 그 배포를 만들 때 스냅샷된다 → [[Vercel 배포 패턴]]

| 변수 | 값 |
|---|---|
| `MONGO_URI` | **`mongodb+srv://`** — 로컬의 표준 URI 를 복사하면 안 된다 |
| `SESSION_SECRET` | `.env.local` 값. myjane 것과 달라야 한다 |
| `IP_HASH_SALT` | `.env.local` 값 |
| `OPENAI_API_KEY` | 아래 ③ 참고 |
| `NOTION_TOKEN` | 수집 스크립트만 쓴다. 런타임에 필요 없다 |
| `R2_*` 5개 | 이미지는 이미 R2 에 있고 URL 이 DB 에 박혀 있어, 런타임에 필요 없다 |
| **`CHAT_ENABLED`** | **`true` — 맨 마지막에 넣는다** |

**넣지 않는 것** — `MONGO_DB` · `NEXT_PUBLIC_COOKIE_DOMAIN` ·
`SEED_READER_PASSWORD` · `GITHUB_TOKEN` · `OPENAI_VISION_MODEL`

`NEXT_PUBLIC_*` 은 Secret 이 아니라 **Config** 타입이어야 한다(지금은 쓰는
값이 없다) → [[Vercel 배포 패턴]]

### ② 운영 검증 (로컬에서 통과한 것을 다시)

```bash
# 캐시 히트 — source=cache 여야 한다
curl -sD- -o/dev/null -X POST https://jangmini.myjane.co.kr/api/chat \
  -H 'Content-Type: application/json' -H 'Origin: https://jangmini.myjane.co.kr' \
  --data-binary @payload.json | grep -i x-jangmini-source
```

⚠️ **한글 페이로드를 셸에 인라인으로 넣지 않는다.** Git Bash 를 통과하며
깨져서 `detectLang` 이 `en` 을 주고, 캐시가 안 맞는다. 이걸로 "캐시가 고장났다"
고 오판했다 — **UTF-8 파일로 만들어 `--data-binary @파일`** 로 보낸다.

- [ ] 추천 질문 5개 → `source=cache`
- [ ] 자유 질문 → `source=openai`
- [ ] 페르소나 거절 3종 (연봉 / 파이썬 코드 / 인젝션)
- [ ] 익명 제한 → 429 + `browse=true`
- [ ] reader 로그인 → 제한 해제. 쿠키에 `domain=` 이 **없어야** 한다
- [ ] `/admin` → 비로그인 차단 · step-up 요구 · 4탭 동작
- [ ] myjane 여섯 앱이 멀쩡한지 (`www` `snapword` `fitlog` …)

### ③ OpenAI 전용 프로젝트 키 — **아직 미결**

여러 번 알렸고 아직 그대로다. **방어 7층 중 L4 가 비어 있다.**

지금 키는 SnapWord·SnapNote·FitLog 와 공유한다. jangmini 가 어뷰징당하면
그 앱들의 예산까지 태우고, 키를 회수하면 그 앱들이 함께 죽는다.

L2(익명 제한)는 시크릿 창이면 초기화되므로 완벽할 수 없다. 그래서 실제
방어선은 L3(전역 일일 캡)과 **L4(OpenAI 프로젝트 monthly hard limit)** 다.
L4 는 우리 코드에 버그가 있어도 작동하는 유일한 층이다.

바꾸는 것은 `.env.local` 과 Vercel 의 한 줄이다.

---

## 그다음 (급하지 않음)

- [ ] 영문 답변 캐시 — 지금은 `ko` 16건만 있다. 영문 질문 칩이 생길 때
      `pnpm warm -- --write --lang=en` 을 돌린다
- [ ] `/stack` 기술 스택 연대기 전용 페이지 (지금은 `/resume` 04 절에 표로 있다)
- [ ] `public/resume.pdf` 다운로드 버튼
- [ ] 슬러그 손질 — 대표 7건은 다듬었고 나머지 38건은 로마자 자동 생성이다
      (`pnpm ingest -- --write --reslug`)
- [ ] `contribution` 이 빈 11건 (2018~2022 큐텐). 노션에서 채우면 반영된다
- [ ] 노션 태그 `cloudflare` 가 한 번도 안 붙었다. Ignite 에 붙일 만하다
- [ ] `profile/profile/1-*.png` (2.2MB)가 본인 사진인지 확인 — 맞으면 히어로에
      쓸 수 있다. 지금은 사용자가 준 메모지를 쓴다

## 검증에서 배운 것

- **한글을 셸 인라인으로 보내면 깨진다.** 파일로 만들어 `--data-binary`.
  이걸로 캐시가 고장난 줄 알았다
- **캐시 히트는 전역 캡을 우회한다.** 의도한 설계지만, 캡을 검증할 때는
  **자유 질문**으로 해야 한다. 추천 질문으로 테스트하면 캡이 안 걸려서
  "캡이 동작하지 않는다"고 오판한다
- **DOM 존재만 보고 판단하지 않는다** → [[화면 확인의 함정]]
- 챗 화면이 지운 메모지 영상 3개를 참조해 404 를 네 번 받고 있었다.
  화면은 정상으로 보였다 — **네트워크와 콘솔을 함께 봐야** 잡힌다

## 하지 않기로 한 것

- **JSON Resume 테마** — 그 스키마에 회고·기여도·4단 본문이 들어갈 자리가 없다
- **myjane 디자인 시스템 적용** — 사용자가 명시적으로 하지 말라고 했다
- **자유 입력 캐시** — 텍스트를 해시하면 거의 안 맞고, 방문자가 조금씩 다른
  문장으로 캐시 컬렉션을 채울 수 있다
- **`CHAT_ENABLED` 를 DB 설정으로** — DB 를 못 읽을 때도, DB 가 뚫렸을 때도
  과금 경로가 열려선 안 된다. 환경 변수로 남긴다
- **등장 애니메이션에 가시성을 의존** → [[화면 확인의 함정]]

## 관련

[[jangmini]] · [[Plans MOC]] · [[MongoDB Atlas]] · [[Cloudflare R2]] ·
[[Vercel 배포 패턴]] · [[AI 채팅 패턴]] · [[화면 확인의 함정]]
