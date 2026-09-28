---
title: Projects MOC
type: moc
tags: [moc]
updated: 2026-09-18
---

# Projects MOC

## myjane.co.kr 서비스

회원(`user` DB)과 세션 쿠키(`.myjane.co.kr`)만 공유한다. 데이터와 화면은 서로 별개다
→ [[인증과 세션 공유]] · [[서비스 카테고리와 카피 원칙]]

- [[MyJane]] — 포털 · 통합 로그인 (admin 예정)

**공부 기록**

- [[SnapWord]] — 단어장 · `vocab`
- [[SnapNote]] — 오답노트 · `math`

**건강 기록**

- [[FitLog]] — 인바디 기록 · `fit` · **개발 중**

**습관 기록**

- [[2hbk]] — 목표·스티커 · `hamhibokka` · **이관 완료, 배포 대기**
  네 앱 중 유일하게 **이메일 + 비밀번호** 로그인이다

**성향 기록**

- [[TypeLog]] — 성향 놀이·회차별 변화 · `type` · **계획**
  질문지는 [[설문지 JSON 작성 지침]] 규격으로 등록한다

**마음 기록**

- [[CalmTouch]] — 만지면 움직이고 가만히 두면 잔잔해지는 화면 24장면 · DB 없음 · **배포됨 (calmtouch.myjane.co.kr · 2026-09-10)**
  회원 기능이 없어 로그인 없이 쓴다. 포털 랜딩·/link·다섯 앱 스위처에 들어갔다

**프롬프트 기록**

- [[AIKit]] — 첫 메뉴 **Prompt Log** · `aikit` · **계획**
  프롬프트 + 입력·결과 이미지를 한 묶음으로 보관한다. 이미지 생성은 하지 않고,
  다른 AI 에서 만든 결과를 되가져와 모은다 → [[H AIKit 구축]]
  이미지는 기본 비공개(본인만)이고, 골라서 공유 링크로 열 수 있다

## 별개 서비스

- [[jangmini]] — AI 챗봇형 개인 포트폴리오 · `jangmini.myjane.co.kr` · `jangmini` · **계획**
  **myjane 서비스가 아니다.** 도메인만 하위에 두고 회원·세션·디자인·admin 을 공유하지
  않는다. 왜 다른지는 [[jangmini]] 첫 표에 있다 → [[D jangmini 구축]]
- [[Ignite]] — 건축사무소 · Vercel 이관 대기

## 참고 사이트

- [[결쩜사]] — 페이지 패턴 분석 대상
