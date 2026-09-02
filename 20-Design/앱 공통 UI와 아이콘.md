---
title: 앱 공통 UI와 아이콘
type: design-system
tags: [design, myjane]
updated: 2026-09-02
---

# 앱 공통 UI와 아이콘

SnapWord · SnapNote · FitLog · 포털이 **기술적으로** 공유하는 시각 체계.
서비스를 하나의 제품처럼 설명하는 것과는 다른 이야기다 → [[서비스 카테고리와 카피 원칙]]

## 테마 4종

`lib/theme.ts`에 정의하고 `data-theme` 속성으로 전환한다. 사용자가 고를 수 있다.

| 테마 | SnapWord · SnapNote | FitLog |
|---|---|---|
| **퍼플** | **기본** `#190527` / `#a78bfa` | 기본(`dark`) `#190527` / `#a78bfa` |
| 다크 | `#000000` / `#2ee8ae` | — |
| 화이트 | `#f2f2f7` / `#1ab485` | `#fdfbff` / `#7c3aed` |
| 네온핑크 | 폐지 (2026-09-02) | `#050008` / `#ff4ecd` |
| 커스텀 | 사용자 지정 | 사용자 지정 |

2026-09-02에 두 Snap 앱의 네온핑크를 **FitLog와 같은 보라 팔레트로 교체하고
기본 테마로 삼았다.** `data-theme` 값은 `violet`을 그대로 재사용해서
기존 선택이 살아남는다. 검정+민트 다크 테마는 선택지로 남겼다.

기본 테마를 바꿀 때 함께 고쳐야 하는 곳 —

- `app/globals.css`의 `:root` (첫 페인트가 기본 테마와 같아야 한다)
- `lib/theme.ts`의 `DEFAULT_THEME`
- `components/ThemeProvider.tsx`의 초기 상태·컨텍스트 기본값
- `app/manifest.ts`의 `background_color` · `theme_color`

**포털(www.myjane.co.kr)은 라이트 전용**이다. 테마 전환이 없고 결쩜사 팔레트
(`#fdfbff` / `#7c3aed` / 골드 `#c9a84c`)를 쓴다. 참고 사이트가 라이트 전용이라
다크를 얹는 순간 대조가 어긋났다 → [[MyJane]]

## CSS 변수

```css
--bg-primary --bg-secondary --bg-card --bg-elevated
--border --border-subtle
--text-primary --text-secondary --text-muted
--accent --accent-hover --accent-subtle
--point --point-subtle
--danger --danger-subtle --success --success-subtle --warning
--input-bg
```

색을 직접 쓰지 않고 **항상 변수로 참조**한다. 그래야 테마 전환이 깨지지 않는다.
예외는 아래 워드마크의 브랜드 보라 하나뿐이다.

## 공통 요소

- **TopNav** — 로고 + 앱 스위처(다른 앱 아이콘) + 메뉴 + 테마 선택
  주기능이 아닌 항목(My · Notice · Q&A)은 PC에서 `More ▾` 드롭다운,
  모바일에서는 구분선 아래로 내린다. 상단 메뉴 표기는 영어(`Home` `Inbody` `History`)
- **FloatingChat** — 우하단 AI 상담 버튼
- **SiteFooter** — 서비스 한 줄 설명 + myjane 워드마크 + `@2026 MyJane All rights reserved`
  (**좌측 정렬**)
- **Toast** — 상단 알림

접힌(모바일) 메뉴의 `Logout`은 위 링크들과 **같은 줄 높이·정렬**로 맞춘다.
`.topnav-logout`(작은 알약 버튼)을 그대로 쓰면 글씨가 작고 안쪽으로 들여써져 튄다.
`.topnav-menu-link .topnav-menu-logout` 조합으로 두고 밑줄만 없앤다.

## myjane 워드마크

푸터에서 포털로 나가는 링크는 `MyJane` 평문이 아니라 워드마크로 쓴다.

```html
<a class="myjane-mark" href="https://www.myjane.co.kr">my<span>jane</span></a>
```

| 부분 | 색 | 이유 |
|---|---|---|
| `my` | `var(--text-primary)` | 배경을 따라간다. 흰 배경에서 검정, 어두운 배경에서 흰색 |
| `jane` | 다크 `#a78bfa` · 라이트 `#7c3aed` · custom `#8b5cf6` | **브랜드 색이라 앱 테마와 무관하게 보라 고정** |

`jane`에 `var(--accent)`를 쓰면 안 된다. SnapWord·SnapNote의 액센트는 민트여서
초록색 `jane`이 나온다. custom 테마는 배경을 미리 알 수 없어 양쪽에서 읽히는
중간 톤(`#8b5cf6`)을 쓴다.

포털 자신의 푸터는 어두운 배경이므로 `my`가 흰색, `jane`이 `#a78bfa`다
(`.site-footer-brand .brand-word > span`).

`my`와 `jane`은 **한 요소 안의 텍스트 노드 + span**이다. flex + `gap`을 주면
둘 사이가 벌어져 `my jane`으로 보인다. `.brand-word`로 감싸 `display:inline-block`으로 둔다.

## 앱 아이콘 — 게이지 호

2026-09-02에 흰 타일 + 빨간 선화에서 **짙은 보라 타일 + 게이지 호**로 전면 교체했다.
세 앱이 같은 기하학을 쓰고 안쪽 심볼만 다르다.

```
타일   192×192, rx 44, linear-gradient(150deg, #3a1266, #1c0733)
호     <circle cx="96" cy="106.46" r="60"> 2개를 rotate(158.96 96 106.46) 안에 둠
       stroke-width 11, round cap
       라일락 트랙  stroke-dasharray="232.6 144.4"  #6b4a9e
       골드 진행    stroke-dasharray="128 249"      #e8d18a
심볼   흰색 선, stroke-width 11, round cap
       SnapWord  가로선 3개 (y=82 / 102 / 122)
       SnapNote  체크  M68 102 L88 122 L128 74
       FitLog    꺾은선 M64 114 L86 90 L106 100 L128 74
```

**두 호는 같은 `<circle>`을 `stroke-dasharray`만 바꿔 그린다.** 처음에는 골드 호를
`<path>`로 따로 그렸는데 끝점이 라일락 원에서 13.8px(SnapWord) / 6.2px(FitLog)
벗어나 "금색이 붕 떠 보인다"는 지적을 받았다. 같은 원을 쓰면 어긋날 수가 없다.

SVG 원본은 각 저장소 `public/app-icon.svg`에 둔다. PNG는 sharp로 파생한다.

| 파일 | 크기 | 쓰임 |
|---|---|---|
| `icon.png` | 192 | PWA · 일반 |
| `favicon.png` | 32 | 탭 |
| `site-title-icon.png` | 64 | TopNav 로고 |
| `*-link-icon.png` | 56 | 다른 앱 스위처 |
| `<app>-icon-512.png` | 512 | 원본 보관 |

포털 아이콘(MJ)은 **사용자가 준 원본 이미지**를 쓴다. 직접 그려서 비슷하게 만들면
"조금 다르다"는 지적이 나온다. 원본은 `myjane/public/app-icon-source.jpg`에 보관하고
필요한 크기는 여기서 파생한다.
