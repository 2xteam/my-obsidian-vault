---
title: A 디자인 요소 추가
type: plan
tags: [plan, design]
updated: 2026-09-07
status: 여섯 앱 구현 완료 — 배포 대기
applies-to: [myjane]
---

# A 디자인 요소 추가

결쩜사에서 가져올 만한 요소 네 가지. **myjane 에서 먼저 만들어 확인하고**,
좋으면 다섯 앱으로 옮긴다 → [[여섯 앱 디자인 시스템]]

전부 CSS 와 작은 컴포넌트라 서버·DB 를 건드리지 않는다.

## 1. 스크롤 프로그레스 바

헤더 바로 아래에 얇은 띠가 차오르며 페이지가 얼마나 남았는지 보여준다.

- 굵기 2~3px, 헤더의 `borderBottom` 자리를 대신하거나 그 아래에 겹친다
- 색은 `--accent`. 배경은 투명(트랙을 그리지 않는다)
- `position: sticky` 헤더 안에 두면 스크롤을 따라온다

**구현 주의**

- `scroll` 이벤트를 그대로 쓰면 프레임마다 계산한다. `requestAnimationFrame` 으로 묶는다
- 진행률 = `scrollTop / (scrollHeight - clientHeight)`. 분모가 0 이면(스크롤이 없는
  짧은 페이지) **0 으로 나눈다** — typelog 처럼 짧은 페이지가 실제로 있다
- `prefers-reduced-motion` 에서는 transition 을 끈다 (globals.css 에 이미 규칙 있음)
- 클라이언트 컴포넌트여야 한다(`"use client"`)

## 2. 밑줄 포인트 요소

헤드라인의 한 부분에 형광펜 같은 밑줄을 깔아 강조한다.
결쩜사가 "자기이해 리포트" 에 노란 밑줄을 쓴 방식이다.

- 글자 **뒤에** 깔리는 띠다. `text-decoration` 이 아니라 배경으로 그린다
- `background: linear-gradient(transparent 60%, var(--gold-soft) 60%)` 형태가
  가장 단순하다. 또는 `box-shadow: inset 0 -0.35em 0 var(--gold-soft)`
- 금색을 쓴다 — **면적이라 `--gold` 계열이 맞고 글자색이 아니다**
  (`--point-ink` 는 글자용) → [[먹청 톤 팔레트]]
- 한 헤드라인에 **한 군데만**. 두 곳이면 강조가 사라진다

⚠️ `.headline span` 이 이미 그라디언트 글자색(`background-clip: text`)에 쓰이고 있다.
밑줄용으로 같은 선택자를 쓰면 충돌한다 — 새 클래스를 만든다.

## 3. 프로세스 소개 방식

결쩜사의 `01 ~ 05` 세로 타임라인. 왼쪽에 번호 원과 연결선, 오른쪽에
배지 + 제목 + 설명, 필요하면 펼쳐보기.

우리 앱에는 이미 단순한 3단계 목록(`stepsStyle`)이 있다. 그것을 이 형태로 올린다.

- 번호 원과 원을 잇는 세로선. 마지막 항목은 선을 그리지 않는다
- 상태 배지(`지금 바로` · `무료` · `프로필 확인 후`) — 조건을 한눈에 보여주는 장치다.
  글자용 색을 쓴다(`--accent-ink` · `--point-ink`)
- 연결선은 `--border`, 번호 원은 `--accent-subtle` 배경 + `--accent-ink` 글자

**어디에 쓰나** — fitlog `HOW IT WORKS`(3단계), 2hbk 목표 만들기 안내,
myjane 소개의 `ABOUT MYJANE`. 단계가 **3개 이상이고 순서가 중요할 때**만 쓴다.

## 4. 카드 라운딩 포인트

왼쪽위·오른쪽하단의 라운딩만 절반으로 줄여 대각선 방향성을 만든다.
시선이 좌상 → 우하로 흐르는 방향과 같아서 자연스럽다.

```css
/* .sheet 는 border-radius 24px */
.sheet--point {
  border-radius: 12px 24px 12px 24px;
}
```

### ⚠️ 어디에 넣을지가 이 요소의 전부다

아무 카드에나 쓰면 방향성이 서로 상쇄돼 **의미가 없어진다.**
2026-09-07 에 사용자와 합의한 기준은 "지금 이걸 보세요"인 카드다.

| 넣는다 | 왜 |
|---|---|
| 히어로 시트 (`sheet--dark`) | 페이지의 시작. 딱 하나뿐이라 남용될 수 없다 |
| 마무리 CTA 시트 | 페이지의 끝. 히어로와 짝을 이뤄 문서를 여닫는다 |
| fitlog 홈 `LATEST · BLOOD` — **벗어난 항목이 있을 때만** | "봐야 할 게 있다"는 상태 신호. 조건부라서 눈에 띈다 |
| 2hbk 목표 달성 카드 | 금색 판으로 바뀌는 그 순간의 강조 |

| 넣지 않는다 | 왜 |
|---|---|
| 서비스 카드 5개 | 나란히 놓여 방향성이 상쇄된다 |
| 목록의 반복 항목 | 포인트가 아니라 배경이 된다 |
| 입력 폼 카드 | 읽고 채우는 곳이라 시선을 끌 이유가 없다 |

**한 화면에 최대 1~2개.** 그 이상이면 포인트가 아니라 스타일이다.

## 진행 순서

1. [x] myjane 에 네 요소 구현 · 대비 검사 — `fcd7097`
2. [x] 화면으로 확인받기 — 라운딩을 12px → 6px 로 한 번 더 줄였다
3. [x] [[여섯 앱 디자인 시스템]] 에 규칙 적기
4. [x] `design:check` 규칙 G — 라운딩 2개 · 밑줄 1개를 넘으면 막는다
5. [x] 다섯 앱으로 옮기기 — `SnapWord 10d9f43` `SnapNote 66a7ff3` `fitlog 58373b5` `2hbk 24effc0` `typelog 523729f`
6. [ ] 배포  ← **여기**

## myjane 구현 결과 (2026-09-07 · `fcd7097`)

| 요소 | 어디에 | 파일 |
|---|---|---|
| 진행 띠 | `.site-top` 안 (sticky 로 바꿨다) | `components/ScrollProgress.tsx` |
| 형광 밑줄 | ABOUT MYJANE 헤드라인 "계정뿐이에요" | `.mark` |
| 타임라인 | 새 시트 "시작하는 순서" 3단계 | `FLOW` in `app/page.tsx` |
| 라운딩 포인트 | 히어로 · 마무리 CTA | `.sheet--point` |

CSS 는 `app/globals.css` 가 아니라 **`app/elements.css`** 에 있다. B·C 세션이
같은 트리에서 globals.css 를 고치고 있어서 충돌을 피했다.

### ⚠️ 밟은 함정 둘 — 다섯 앱에 옮길 때 그대로 만난다

**① `@import` 로는 순서를 못 만든다.**
`globals.css` 안에 `@import "./elements.css";` 를 두면 안 된다. `@import` 는 파일
맨 앞이라야 하니 규칙이 **늘 globals.css 보다 먼저** 들어가고, 특이도가 같은
`.sheet--point` 가 `.sheet { border-radius: 24px }` 에 진다. 조용히 진다 —
에러가 없고 라운딩만 안 먹는다. `layout.tsx` 에서 globals.css **다음 줄**에
불러야 한다.

**② `.headline span` 이 `.mark` 를 이긴다.**
`.headline span` 은 특이도 (0,1,1), `.mark` 는 (0,1,0). 그 규칙이
`background-clip: text` + `color: transparent` 라서 금색 띠가 사라지고 글자가
petrol 그라디언트로 칠해진다. **대비 1.00 으로 측정됐다** — 눈으로는 "그냥 강조
글자" 로 보여서 놓치기 쉽다. `.headline .mark` 로 특이도를 맞추고
`-webkit-text-fill-color: currentColor` 로 되돌린다.

### 팔레트에 없던 토큰

`--point-ink` 가 myjane 어휘에 빠져 있었다 (`scripts/sync-palette.mjs` 의 `NAMES.myjane`).
`design:check` 규칙 B 가 잡았다. 다섯 앱 어휘에는 원래 있었다.

### 측정 (1280×900 · `dev:verify` 3010)

| 자리 | 대비 | 기준 |
|---|---|---|
| 밑줄 띠 위 글자 (24px/700) | 12.48 | 3.0 |
| 타임라인 제목 (15.2px/700) | 14.67 | 4.5 |
| 타임라인 본문 (13.6px) | 4.94 | 4.5 |
| 배지 `--accent-ink` (11px/700) | 5.46 | 4.5 |
| 배지 `--point-ink` (11px/700) | 5.23 | 4.5 |
| 진행 띠 vs 헤더 배경 | 6.98 | 3.0 (비텍스트) |

### ⚠️ 이 화면을 CDP 로 잴 때

브라우저 창이 다른 창에 가려 있으면 —

- `innerWidth` 가 **0** 이 된다. 그 상태의 폭·높이 측정값은 전부 쓰레기다.
  색(computed style)은 멀쩡하다. 재기 전에 `innerWidth` 를 먼저 확인한다
- `transition` 이 진행 중인 값에서 멈춘다. `.scroll-progress::after` 의
  `transform` 이 `scaleX(0)` 으로 읽혀 "안 움직인다" 고 오진했다.
  `transition: none !important` 를 잠깐 얹고 다시 읽으면 실제 값이 나온다
- 스크린샷이 5초 타임아웃으로 실패하거나 빈 화면이 나온다

→ [[개발 서버와 검증 환경]]

## 다섯 앱으로 옮긴 결과 (2026-09-07)

`myjane c7e52e1` 에서 CSS 원본을 **`design/elements.css`** 로 옮겼다. 팔레트와
같은 구조다 — `npm run elements -- --write` 가 여섯 앱의 `app/elements.css` 를
쓴다. 그 스크립트는 CSS 만 옮기고 나머지는 **검사만** 한다:
`layout.tsx` 의 import 순서 · `ScrollProgress.tsx` 존재 · 다섯 앱 `Sheet.tsx` 동일성.

| 앱 | 밑줄 자리 | 타임라인 |
|---|---|---|
| SnapWord | WHAT YOU GET "옮겨 적지 않아도" | `STEPS` → `.flow` |
| SnapNote | WHAT YOU GET "다시 풀 것만" | `STEPS` → `.flow` |
| fitlog | WHAT YOU GET "흐름을" | `STEPS` → `.flow` |
| 2hbk | THREE MODES "같이 해도" | `STEPS` → `.flow` |
| typelog | HOW IT WORKS "세 걸음" | 인라인 배열 → `STEPS` + `.flow` |

typelog 는 밝은 plain 시트가 마무리 CTA 뿐이었다. 거기엔 라운딩 포인트가
가므로 밑줄은 tint 시트(HOW IT WORKS)에 넣었다. 타임라인 번호도 다섯 앱 중
혼자 `pill--gold` 였는데 `.flow-num` 으로 통일했다.

### 옮기면서 새로 알게 된 것 셋

**① 다크 테마에서 형광펜은 성립하지 않는다.**
형광펜은 "밝은 띠 + 어두운 글자" 다. 다크에서는 글자가 밝아야 카드 위에서
읽히니 띠도 어두워야 하고, 그러면 금색이 탁한 올리브(rgb 96 108 87)로 보인다 —
실제로 얼룩처럼 렌더됐다. 띠를 밝히면 반대로 글자가 3.4:1 로 떨어진다.
그래서 `[data-theme="dark"]` 에서는 **금색 밑줄 3px** 로 바꿨다. 글자 위를
덮지 않으니 글자 13.66:1, 금색 선 9.7:1 로 둘 다 지킨다. myjane 은 다크가
없어서 이 블록이 걸리지 않는다.

**② 타임라인 가운데 정렬은 기본값이 될 수 없다.**
myjane 의 시트는 헤드라인이 가운데라 `margin-inline: auto` 가 맞았지만, 다섯
앱은 왼쪽 정렬이라 타임라인이 제 헤드라인보다 안쪽으로 들어가 어긋났다.
`.flow--center` 로 뺐다 — 가운데 정렬 시트에서만 붙인다.

**③ 어휘가 둘이라 CSS 한 파일이 그냥은 안 돌았다.**
myjane 은 `--text`/`--accent-soft`/`--text-dim`, 다섯 앱은
`--text-primary`/`--accent-subtle`/`--text-secondary` 다. `elements.css` 안에서
`--el-*` 로 묶어 다리를 놓았다. 그리고 다섯 앱에 **`--accent-ink` 와
`--gold-soft` 가 아예 없었다** — 그래서 타임라인 번호가 면적용 `var(--accent)`
를 글자색으로 쓰고 있었다. 팔레트 어휘에 추가했다.

⚠️ 다리(`var(--x, 대체값)`)는 **검사 규칙 B 가 못 본다.** 규칙 B 의 정규식은
대체값 없는 `var(--x)` 만 잡는다. 여기서는 오타를 잡아주지 못한다.

### 검사기가 주석을 읽던 문제

`elements.css` 주석에 설명으로 쓴 `var(--radius-lg)` 와 `var(--x)` 를 실제
사용으로 읽어 "미정의 변수 7건" 을 냈다. 규칙 B~C 가 주석을 지운 사본을 보게
고쳤다. C 기준선이 46 → **45** 로 줄었다.

> 설명을 쓰면 화내는 도구는 설명을 안 쓰게 만든다.

## 옮길 때

`components/Sheet.tsx` 는 다섯 앱에 **복사본**이다. `sheet--point` 같은 변형을
prop 으로 받게 하려면 다섯 개를 함께 고쳐야 한다 →
[[여섯 앱 디자인 시스템]] "다음에 만들 것"
