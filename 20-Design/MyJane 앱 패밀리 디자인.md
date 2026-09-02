---
title: MyJane 앱 패밀리 디자인
type: design-system
tags: [design, myjane-family]
updated: 2026-09-02
---

# MyJane 앱 패밀리 디자인

SnapWord · SnapNote · FitLog가 공유하는 시각 체계.

## 테마 4종

`lib/theme.ts`에 정의하고 `data-theme` 속성으로 전환한다. 사용자가 고를 수 있다.

| 테마 | 배경 | 강조 |
|---|---|---|
| 다크(기본) | `#000000` | `#2ee8ae` |
| 화이트 | `#f2f2f7` | `#1ab485` |
| 네온핑크 | `#050008` | `#ff4ecd` |
| 커스텀 | 사용자 지정 | 사용자 지정 |

## CSS 변수

```css
--bg-primary --bg-secondary --bg-card --bg-elevated
--border --border-subtle
--text-primary --text-secondary --text-muted
--accent --accent-hover --accent-subtle
--point --point-subtle
--danger --success --warning
--input-bg
```

색을 직접 쓰지 않고 **항상 변수로 참조**한다. 그래야 테마 전환이 깨지지 않는다.

## 포털(myjane.co.kr)은 다른 팔레트

포털만 MJ 아이콘에서 가져온 네이비 `#0c2343` + 앰버 `#f9c22a`를 쓴다.
세 앱(검정+민트)과 구분되어 "묶는 자리"라는 성격이 드러난다. → [[MyJane]]

## 공통 요소

- **TopNav** — 로고 + 앱 스위처(다른 앱 아이콘) + 메뉴 + 테마 선택
- **FloatingChat** — 우하단 AI 상담 버튼
- **SiteFooter** — 서비스 한 줄 설명 + `MyJane` 링크 + `@2026 MyJane All rights reserved` (좌측 정렬)
- **Toast** — 상단 알림

## 앱 아이콘

흰 타일 + 빨간 선화(`#e32833`) 스타일. 각 앱이 서로의 아이콘을 앱 스위처에 갖고 있다.

- `snapword-link-icon.png` — 책 + 물음표 + 카메라
- `snapnote-link-icon.png` — 책 + 시그마 + 컴퍼스
