---
title: MyJane
type: project
status: 운영중
domain: www.myjane.co.kr
repo: https://github.com/2xteam/myjane
local: C:\Dev\myjane
branch: main
tags: [project, myjane-family]
updated: 2026-09-02
---

# MyJane

앱 패밀리 포털. SnapWord · SnapNote · FitLog로 이동하는 링크 사이트.

## 성격

**의존성은 Next/React뿐이고 전부 정적 페이지로 빌드된다.** 서버 로직·환경변수가 없어
Vercel에서 함수 호출이 발생하지 않는다.

## 브랜딩

MJ 아이콘(네이비 `#0c2343` + 앰버 `#f9c22a`)에서 색을 가져왔다.
`public/myjane-icon.png`(512px)가 원본이고 나머지 크기는 여기서 파생한다.

| | 배경 | 강조 |
|---|---|---|
| 다크(기본) | `#0c2343` | `#f9c22a` |
| 라이트 | `#f4f6fc` | `#f2951a` (텍스트용 `#a2620a`) |

앰버는 흰 배경에서 대비가 부족해 텍스트 강조에는 어둡게 조정한 `--accent-ink`를 쓴다.

## 앱 카드 추가하기

`app/page.tsx`의 `APPS` 배열만 수정하면 된다.

## 예정 작업

- [ ] **통합 admin** — 앱별 탭으로 회원·토큰·문의·공지·통계 관리
- [ ] **통합 로그인/회원가입** — 각 앱이 아니라 여기서 처리하고, 파라미터로 출처 앱을 받아
      FitLog에서 온 경우에만 신체 프로필(키·성별·생년)을 추가로 받는 분기
- [ ] FitLog 카드 추가
