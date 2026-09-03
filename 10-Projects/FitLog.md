---
title: FitLog
type: project
status: 개발중
domain: fitlog.myjane.co.kr
repo: https://github.com/2xteam/fitlog
local: C:\Dev\fitlog
branch: main
db: fit
tags: [project, myjane]
updated: 2026-09-02
---

# FitLog

인바디 결과지를 사진으로 찍어 기록하고 체성분 추이를 관리하는 앱.
나중에 달리기·활동·운동 기록까지 확장할 계획이라 이름을 `Snap*` 계열에서 벗어나게 지었다.

## 스택

Next.js 15 · MongoDB(`fit` DB) · OpenAI Vision · Cloudflare R2(`fitlog` 버킷) · Vercel `icn1`
로컬 개발 포트 **3003** (SnapWord 3000 · SnapNote 3001 · myjane 3002).
`npm run dev` / HTTPS가 필요하면 `npm run dev:https` → [[인증과 세션 공유]]

DB 이름은 URI 경로가 아니라 **코드에서 `dbName: "fit"`으로 못 박는다.**
운영 URI가 `math`를 가리켜 측정 기록이 SnapNote DB에 쌓인 사고가 있었다
→ [[MongoDB Atlas]]

## 디자인

**결쩜사 패턴을 따른다** — SnapWord·SnapNote의 검정+민트 계열이 아니다.

- 라이트 기본 `#fdfbff` / `#7c3aed`, 다크 `#190527` / `#a78bfa`
- Pretendard, 중립색에 보라를 섞어 순수 검정·회색을 쓰지 않음
- 화면은 `.sheet`(plain / tint / dark / gold) 단위로 쌓고, 시트마다 eyebrow → headline → 콘텐츠
- `components/Sheet.tsx`가 시트와 실(結) 장식을 담당
- 상단 메뉴 표기는 영어 — `Home` `Inbody` `History`. 부가 기능(My · Notice · Q&A ·
  Logout)은 `More ▾`로 접는다 → [[앱 공통 UI와 아이콘]]
→ [[결쩜사 페이지 패턴]] · [[레퍼런스 사이트 분석 방법]]

## 데이터 모델

| 컬렉션 | 내용 |
|---|---|
| `measurements` | 인바디 측정 1건 — 아래 참조. **체중만 넣은 기록도 여기 들어간다** |
| `activities` | 운동·활동 기록 (예정) |
| `chatthreads` | AI Fit 상담사 |
| `user` DB `users` | 앱 공용 + `heightCm` `gender` `birthYear` (FitLog 전용 필드) |

> `weights` 컬렉션과 `/weight` 화면은 **2026-09-02에 없앴다.** 체중을 따로 관리하니
> 같은 지표가 두 곳에 생겨 추이가 갈렸다. 지금은 결과지 없이 체중만 입력해도
> `source: "manual"` 인바디 기록 1건으로 저장된다. 상단 탭에서도 빼고
> 등록 화면(`/measurements/new`) 안의 "체중만 기록" 시트로 옮겼다.

### measurements 설계 원칙

기종(270 · 270S · 720 · 970 · 구형)마다 인쇄 항목이 달라 **대부분 선택값**이다.

- 수치는 `{ value, min, max }` — 결과지에 인쇄된 **표준범위까지** 저장
- 스키마에 없는 항목은 `etc: [{label, value, unit}]` 에 보관하고 화면에 그대로 나열
  같은 label이 반복해 쌓이면 정식 필드로 승격 검토
- `userId + measuredDate` 유니크 — **날짜당 1건**
  하루에 여러 번 재도 체성분이 변한 게 아니라 식사·수분·운동에 따른 측정 조건 차이다.
  단 `measuredAt`에는 시각까지 남겨 조건 비교에 쓴다

## 추출 정확도 장치

인바디 결과지는 내부 계산이 맞아떨어진다. 이 성질로 Vision의 숫자 오인식을 잡는다
(`lib/inbody.ts` → `validateMeasurement`).

| 검사 | 허용 오차 |
|---|---|
| 체수분 + 단백질 + 무기질 + 체지방 = 체중 | ±0.6kg |
| BMI = 체중 ÷ (키m)² | ±0.4 |
| 체지방률 = 체지방량 ÷ 체중 × 100 | ±0.8%p |
| 제지방량 = 체중 − 체지방량 | ±0.6kg |
| 골격근량 ≤ 제지방량 | — |

**추출 결과는 저장 전 반드시 검토 화면을 거친다.** 자동 저장하지 않는다.
→ [[OpenAI Vision 추출 패턴]]

## 프로필 게이트

키·성별·출생연도가 없으면 `/api/measurements/extract`가 HTTP 428로 거부한다.
인바디 표준범위와 기초대사량이 성별·연령 기준이라 없으면 해석이 불가능하다.
**몸무게는 게이트에 넣지 않는다** — 결과지에서 추출되므로 미리 받으면 값이 충돌한다.

## 진행 상황

- [x] 골격 · 인증(회원 공유) · 테마 · 네비 · 푸터
- [x] 데이터 모델 · 필드 카탈로그 · 정합성 검증
- [x] Vision 추출 파이프라인 · `/api/measurements` `/api/measurements/extract`
- [x] 도메인 연결 (`fitlog.myjane.co.kr`)
- [x] 결쩜사 팔레트·서체·시트 구조 적용
- [x] 프로필 화면(`/my`) + 게이트 (프로필 없으면 추출이 428로 거부)
- [x] 업로드 → 검토·수정 → 저장 (`/measurements/new`)
- [x] 측정 목록(직전 대비 증감) · 상세(표준범위 막대 · etc · 원본)
- [x] 통합 로그인 연결 (운영 도메인에서 포털로 리디렉트)
- [x] 결과지 **여러 장 동시 업로드** (순차 추출 → 장마다 검토 → 저장)
- [x] 체중만 기록 (별도 컬렉션 없이 인바디 기록으로)
- [x] History — 항목별 접기/펼치기 + 좌우 스크롤 추이 그래프
- [x] **Inbody + History 통합** (2026-09-02) — 아래 참조
- [x] 핵심 3종 **삼각 레이더** (적정 범위 음영 + 내 위치) → [[SVG 차트 패턴]]
- [ ] BMI 계산기
- [ ] AI Fit 상담사 (최근 측정값 컨텍스트 주입)
- [ ] 운영 환경 변수 보강 — `R2_*` 5개 · `OPENAI_VISION_MODEL` · `SMTP_*`
- [ ] 실제 결과지 7장으로 추출 정확도 검증 · 프롬프트 튜닝
- [ ] 활동·운동 기록 (2차)

## 화면 구성 — Inbody 한 곳에서 본다

목록(Inbody)과 추이(History)가 나뉘어 있어 같은 기록을 보려고 오가야 했다.
2026-09-02에 `/measurements` 하나로 합쳤다. **해석이 먼저, 원본이 나중**이다.

| 순서 | 시트 | 내용 |
|---|---|---|
| 1 | INBODY (dark) | 헤드라인 + **결과지 등록 버튼** |
| 2 | MAIN | 삼각 레이더(최근 1건) + 핵심 3종 추이 접기/펼치기 |
| 3 | BY FIELD | 값이 있는 전 항목 접기/펼치기 |
| 4 | RECORDS | **등록된 결과지 — 접어둠**. 펼침 버튼 옆에 등록 버튼을 한 번 더 |

항목이 36개라 한 줄로 늘어놓으면 무엇을 보는지 알기 어렵다. 상세와 항목별 보기는
**결과지의 분석 구획**으로 묶는다 — 경로 첫 마디가 곧 구획이라 표시를 따로 달지
않는다(`groupFields()` · `GROUP_LABELS`).

- 등록 버튼은 맨 위와 결과지 목록 옆, **두 곳**에 둔다
- `/charts`는 경로만 남기고 `/measurements`로 넘긴다 (예전 링크·북마크)
- 상단 메뉴는 `Home` · `Inbody` (History 제거)
- 추이 그래프 조각은 `components/TrendChart.tsx`에 모아 두 자리에서 함께 쓴다
  → [[SVG 차트 패턴]]

## 기록 수정

`/measurements/[id]/edit` — 검사일시 · 수치 · `etc`를 고치고, 원본 사진을 나중에
붙인다. 표준범위(min·max)는 손대지 않고 값만 바꾼다.

**저장(POST)과 수정(PATCH)은 다른 일이다.** POST는 `(userId, measuredDate)` 기준
upsert라 날짜를 바꾸면 다른 날 기록을 덮어쓴다. 그래서 이 문서만 고치는 PATCH를
따로 뒀고, 날짜를 옮길 때 같은 날 기록이 있으면 409로 막는다.

> ⚠️ **PATCH는 보낸 잎(leaf)만 dot-path로 `$set` 한다.**
> `{composition: {totalBodyWater: …}}` 처럼 구획을 통째로 `$set` 하면
> payload에 없는 형제 필드(체중·단백질 …)가 **조용히 사라진다.**
> 2026-09-02 실제로 한 건을 날렸다가 화면에 남아 있던 값으로 복구했다.
> 복구가 맞았는지는 인바디 합계 규칙(체수분+단백질+무기질+체지방=체중)이
> 다시 통과하는 것으로 확인했다.

## 원본 결과지

`imageUrl`에 R2 주소를 담는다. 추출 때 R2 설정이 없거나 업로드가 실패하면
**값은 저장되고 원본만 빈다.** 예전에는 조용히 넘어갔는데, 지금은

- 추출 응답이 `imageError`를 함께 돌려주고 검토 화면에 표시한다
- 상세·목록에서 원본 링크를 보여주고, 없으면 첨부 경로를 안내한다
- `POST /api/measurements/[id]/image`로 나중에 사진만 붙일 수 있다

## AI Fit 상담사

버튼(FAB)은 **근육을 불끈 쥐는 팔**이다. 회전하지 않고, 힘을 줬다 폈다 하며
그 순간 링과 스파크가 한 번 퍼진다(`fab-arm` · `fab-bicep` · `fab-burst` · `fab-spark`).
대화가 없을 때는 상담사 소개와 **질문 예시 칩**을 대화처럼 먼저 보여준다.

> ⚠️ **아직 백엔드가 없다.** `/api/chat/threads` 등이 FitLog에 없어 질문을 보내면
> "지금은 상담사와 연결할 수 없어요"가 뜬다. 화면만 준비된 상태다.

## 테스트 자산

실제 인바디 결과지 7장(270S/270×2/970/720/구형×2, 2014~2026)이 있다.
기종이 모두 달라 추출 검증 데이터로 적합하다.
