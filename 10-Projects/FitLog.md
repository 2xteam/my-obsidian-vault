---
title: FitLog
type: project
status: 개발중
domain: fitlog.myjane.co.kr
repo: https://github.com/2xteam/fitlog
local: C:\Dev\fitlog
branch: main
db: fit
tags: [project, myjane-family]
updated: 2026-09-02
---

# FitLog

인바디 결과지를 사진으로 찍어 기록하고 체성분 추이를 관리하는 앱.
나중에 달리기·활동·운동 기록까지 확장할 계획이라 이름을 `Snap*` 계열에서 벗어나게 지었다.

## 스택

Next.js 15 · MongoDB(`fit` DB) · OpenAI Vision · Cloudflare R2(`fitlog` 버킷) · Vercel `icn1`
로컬 개발 포트 **3003** (SnapWord 3000 · SnapNote 3001 · myjane 3002).

## 디자인

**결쩜사 패턴을 따른다** — SnapWord·SnapNote의 검정+민트 계열이 아니다.

- 라이트 기본 `#fdfbff` / `#7c3aed`, 다크 `#190527` / `#a78bfa`
- Pretendard, 중립색에 보라를 섞어 순수 검정·회색을 쓰지 않음
- 화면은 `.sheet`(plain / tint / dark / gold) 단위로 쌓고, 시트마다 eyebrow → headline → 콘텐츠
- `components/Sheet.tsx`가 시트와 실(結) 장식을 담당
→ [[결쩜사 페이지 패턴]] · [[레퍼런스 사이트 분석 방법]]

## 데이터 모델

| 컬렉션 | 내용 |
|---|---|
| `measurements` | 인바디 측정 1건 — 아래 참조 |
| `weights` | 체중 기록 (체중계는 매일, 인바디는 몇 달에 한 번이라 분리) |
| `activities` | 운동·활동 기록 (예정) |
| `chatthreads` | AI Fit 상담사 |
| `user` DB `users` | 세 앱 공용 + `heightCm` `gender` `birthYear` (FitLog 전용 필드) |

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
- [ ] 체중 기록 · 추이 그래프 · BMI 계산기
- [ ] AI Fit 상담사 (최근 측정값 컨텍스트 주입)
- [ ] 활동·운동 기록 (2차)

## 테스트 자산

실제 인바디 결과지 7장(270S/270×2/970/720/구형×2, 2014~2026)이 있다.
기종이 모두 달라 추출 검증 데이터로 적합하다.
