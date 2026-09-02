# my-obsidian-vault

2xteam 서비스들의 지식 저장소. [Obsidian](https://obsidian.md)으로 열면 링크·그래프가 동작하고,
GitHub에서는 그냥 마크다운으로 읽힌다.

## 여는 법

Obsidian → `Open folder as vault` → 이 폴더 선택. [[Home]]에서 시작한다.

## 구조

```
Home.md          최상위 목차 (MOC)
00-Meta/         볼트 사용 규칙, AI 협업 규칙
10-Projects/     앱·사이트별 정보 (SnapWord · SnapNote · FitLog · Ignite · MyJane · 결쩜사)
20-Design/       디자인 시스템, 페이지 패턴, 카피 가이드
30-Patterns/     반복해서 쓰는 개발 패턴과 함정
40-Infra/        도메인·DB·스토리지·배포
Templates/       새 노트 틀
```

## AI 작업 시

각 프로젝트 저장소의 `CLAUDE.md`가 이 볼트를 가리킨다.
작업 전에 관련 노트를 읽고, 작업 후 바뀐 사실을 반영한다. → `00-Meta/AI 협업 규칙.md`

## 주의

**비밀값(API 키·비밀번호·연결 문자열)을 넣지 않는다.** 공개 저장소다. 변수 이름까지만 적는다.
