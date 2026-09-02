---
title: SVG 차트 패턴
type: pattern
tags: [pattern, chart, frontend]
updated: 2026-09-02
---

# SVG 차트 패턴

차트 라이브러리를 쓰지 않고 SVG를 직접 그린다. 번들이 커지지 않고
CSS 변수(테마)를 그대로 쓸 수 있다. [[FitLog]] `app/(app)/charts/page.tsx`,
`components/BodyRadar.tsx`가 실제 구현.

## 라인 차트 — x축은 시간 비례가 아니라 순서

인바디는 몇 달~몇 년 간격이 뒤섞인다. x를 **시간에 비례**시키면 가까운 시기의
기록이 한 곳에 뭉쳐 날짜 라벨이 겹쳐 읽을 수 없다.

```ts
const STEP = 116;                       // 점 하나당 최소 간격
const W = pad.l + pad.r + Math.max(1, n - 1) * STEP;
const px = (i: number) => pad.l + i * STEP;   // 순서(ordinal) 간격
```

간격을 라벨이 겹치지 않을 만큼 확보하면 폭이 컨테이너를 넘어간다. 넘어간 만큼은
**좌우 스크롤**로 본다. 이 조합이라야 "날짜가 다 보인다" + "기간이 겹치지 않는다"를
동시에 만족한다.

### 함정 — grid/flex 자식은 스크롤되지 않는다

`overflow-x: auto`를 걸어도 스크롤이 안 되면 대개 부모가 grid나 flex다.
그 자식의 `min-width` 기본값이 `auto`여서 **내용(넓은 SVG)만큼 늘어나** 버리고,
스크롤 컨테이너가 생기지 않는다.

```css
/* 스크롤 컨테이너 자신 */
overflow-x: auto; overscroll-behavior-x: contain;
/* 그 조상 중 grid/flex 자식인 요소 */
min-width: 0;
```
grid 쪽은 `grid-template-columns: minmax(0, 1fr)`로 둔다. `1fr`은 `minmax(auto, 1fr)`이라
같은 문제를 일으킨다.

스크롤 가능 여부는 `ResizeObserver`로 `scrollWidth > clientWidth`를 확인해
"좌우로 넘겨보세요 →" 안내를 조건부로 띄운다.

## 레이더(삼각) 차트 — 단위가 다른 축을 한 그림에

체중(kg) · 골격근량(kg) · 체지방률(%)은 스케일이 달라 값을 반지름에 바로 쓸 수 없다.

**각 축의 적정 범위를 같은 반지름 구간에 고정 매핑**한다.

```
적정 하한 → R_IN  (0.46 R)
적정 상한 → R_OUT (0.78 R)
r(value) = R_IN + (value - min) / (max - min) * (R_OUT - R_IN)
```

그러면 적정 범위가 **두 삼각형 사이의 고른 띠**가 되고, 내 점이

- 띠 안 → 적정
- 띠 안쪽 → 부족
- 띠 밖 → 초과

로 계산 없이 눈으로 읽힌다. 띠는 도넛처럼 그린다.

```jsx
<path d={`${outerTriangle} ${innerTriangle}`} fillRule="evenodd" fill="var(--success-subtle)" />
```

값이 범위를 크게 벗어나면 점이 축 라벨을 덮으므로 반지름을 `0.93 R`에서 잘라둔다.

축 라벨은 `textAnchor="middle"`로 통일하는 편이 안전하다. 좌우 축에 `start`/`end`를
쓰면 긴 한글 라벨(체지방률)이 viewBox를 몇 px 넘긴다.

## 적정 범위는 어디서 오나

결과지에 인쇄된 표준범위가 있으면 그것을 쓰고(`{value, min, max}`),
없으면 계산해 채운 뒤 **계산했다는 사실을 화면에 밝힌다**.

| 항목 | 없을 때 |
|---|---|
| 체중 | BMI 18.5~23 × (키m)² |
| 골격근량 | 제지방량 표준범위 × (골격근 ÷ 제지방) 비율 |
| 체지방률 | 남 10~20% / 여 18~28% |

→ `lib/inbody.ts`의 `printedRange()` · `buildRadarAxes()`

## 접기/펼치기와 함께 쓰기

항목이 30개를 넘으면 전부 그리면 화면이 길어진다. 한 줄 아코디언으로 두고
**접힌 상태에도 최신값을 보여준다.** 펼치지 않고 훑을 수 있어야 목록이 쓸모 있다.
같은 행 컴포넌트를 요약 섹션과 전체 목록에서 함께 쓴다(`FieldRow`).
