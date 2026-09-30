---
title: 통합 admin
type: pattern
tags: [pattern, admin, myjane]
updated: 2026-10-01
---

# 통합 admin

`www.myjane.co.kr/admin` 에서 네 앱을 **탭으로 오가며** 관리한다.
2026-09-04에 만들었다. 그전에는 SnapWord 에만 자체 admin 이 있었다.

## 핵심 — 포털은 남의 DB 를 직접 읽지 않는다

읽기도 쓰기도 **각 앱의 `/api/admin/*`** 을 거친다.

```
포털 /admin  ──▶  포털 /api/admin/apps/<앱>/<항목>  ──▶  <앱>/api/admin/<항목>
   (세션 서명 토큰으로 관리자 확인)      (공유 비밀 ADMIN_API_SECRET)
```

같은 클러스터라 `useDb()` 로 바로 읽을 수도 있었다. 그렇게 하지 않은 이유 —

- 스키마와 검증이 **그 앱에** 남는다. 앱이 필드를 바꿔도 포털을 고칠 일이 없다
- 쓰기(공지 발행·문의 답변)가 원래 그 데이터를 아는 쪽에서 일어난다

예외는 **회원**뿐이다. `user` DB 는 포털 자신의 데이터라 직접 읽는다.

## 응답 모양을 앱마다 같게

앱이 무엇을 세는지 포털은 모른다. 모양만 안다.

```ts
stats  : { label: string; value: number }[]                       // 타일
tables?: { title: string; columns: string[]; rows: (string|number)[][] }[]  // 순위·추이
```

덕분에 **앱에서 항목을 늘리면 포털을 고치지 않아도** 화면에 나온다.

## 기능이 앱마다 다르다

| 앱 | 요약 | 공지 | 문의 |
|---|---|---|---|
| SnapWord | ✓ (표 포함) | ✓ | ✓ |
| SnapNote | ✓ | ✓ | ✓ |
| FitLog | ✓ | ✓ | ✓ |
| 2hbk | ✓ | — | — |

`lib/adminApps.ts` 의 `features` 로 선언한다. 없는 탭을 그리면 404 를 부르게 되는데,
**앱이 죽은 것과 구분이 안 된다.**

## 포털 자신의 데이터를 다루는 화면

회원 관리(`/admin/members`) 말고 하나 더 — **공유 링크(`/admin/share`, 2026-09-10)**. `/share` 공개 화면의 목록을
등록·수정·정렬·숨김한다. 둘 다 `user` DB 를 직접 읽는 포털의 것이라 앱 API 를 거치지 않는다 → [[MyJane]] "/share"

## 권한

`users.adminRole` — `master` · `operator` · `null`

- **마스터**는 `scripts/seed-admin.mjs` 로만 만든다. admin API 로 마스터를 세울 수
  있게 두면 운영자 하나만 뚫려도 마스터가 늘어난다
- 마스터가 운영자를 세우고 내린다. 운영자는 권한 관리를 못 한다
- 마스터 계정은 API 로 건드릴 수 없다(자기 발등을 찍는 것도 막는다)

## 대리 로그인 (2026-10-01)

`/admin/members` 의 **"이 계정으로 로그인"** — 관리자가 고객 계정으로 들어가 고객이 보는 화면을 그대로 본다.
원본: `myjane/lib/impersonation.ts`

| 단계 | 어떻게 |
|---|---|
| 시작 | `POST /api/admin/users/:id/impersonate` — 마스터·운영자. **관리자·탈퇴·자기 계정은 막는다**. 고객 토큰(`exp` 1시간, `imp` = 관리자 `_id`)을 `snap_session` 에, 관리자 본인 토큰은 `snap_admin_restore`(HttpOnly)에 맡긴다 |
| 만료 | 토큰 `exp` 가 정한다. **모든 앱이 exp 를 검사하므로 앱 코드를 고치지 않아도 함께 끊긴다** |
| 연장 | `POST /api/impersonation/extend` — 남은 **5분 이내에만** 1시간씩. 매번 관리자 권한을 다시 본다 |
| 종료 | `POST /api/impersonation/end` — 맡긴 토큰으로 관리자 세션을 되돌리고 `/admin/members` 로. 만료 뒤에도 된다 |
| 경고 바 | `components/ImpersonationBar.tsx` — **포털과 여섯 앱에 복사본**. 읽을 수 있는 표지 `snap_imp`({exp, name, by})를 보고 맨 위에 빨간 바. 남은 시간 초 단위, 0 이면 자동 종료. 연장·종료는 포털로 폼 POST |

지키는 것

- 대리 세션에서는 `requireSessionUser` 가 **언제나 403** — 비밀번호·탈퇴·이메일·동의·자녀 관리
- **프로필 전환(`/api/auth/switch`)·소셜 연결도 막는다.** 둘 다 30일짜리 새 토큰을 내줘서 1시간 제한을 벗어나는 통로였다
- 시작·연장·종료·만료를 `user` DB `admin_audit_logs` 에 남긴다(누가·누구로·언제·IP·UA). 지우지 않는다
- 연장·종료는 우리 도메인 Origin 만, 돌아갈 `next` 도 우리 도메인만(오픈 리다이렉트 방지)
- ⚠️ 표지 `snap_imp` 는 만료 뒤 **10분 더** 남긴다. 세션과 같이 사라지면 바가 0초를 못 봐서 자동 종료를 못 보낸다
- 로그아웃하면 `snap_imp` · `snap_admin_restore` 도 지운다(포털·여섯 앱 `clearSessionCookieHeaders`)
- 대리 중 기록을 바꾸면 **실제 고객 기록이 바뀐다.** 읽기 전용 모드는 없다

## 확인 창은 화면 안에 (2026-10-01)

관리 화면의 확인은 브라우저 `confirm()` 을 쓰지 않는다 → `app/admin/ConfirmDialog.tsx` (`useConfirm`).
기본 대화상자를 막는 브라우저(인앱·자동화 창)에서는 `confirm()` 이 곧바로 false 를 돌려줘
**버튼을 눌러도 아무 일도 없었다** — 대리 로그인·운영자 세우기·공지 내리기·링크 지우기 네 곳 모두.

## 목록 정렬·페이징 (2026-10-01)

공용 부품 `app/admin/Pagination.tsx` — `Pagination`(총 건수·쪽 번호·쪽당 10/20/50/100/직접 입력, 상한 500) ·
`SortableTh`(머리칸을 눌러 정렬) · `listQueryDefaults` · `listQueryParams`.

- **서버에서** 정렬·자른다. API 는 `page`(1부터) · `pageSize` · `sort` · `dir` 를 받고 `total` 을 돌려준다
- 같은 값은 `_id` 로 순서를 고정한다 — 쪽을 넘길 때 같은 행이 두 번 보이거나 빠지지 않게
- 계산이 필요한 정렬(보이는 이름·수단 수·권한 순위)은 `aggregate` + `$facet`(행과 총 건수를 한 번에)
- 첫 사용처: 회원 목록(기본 가입일 최근 순 20건). 옛 목록은 100건에서 잘려 대시보드 회원 수도 100 에 묶여 있었다

## ⚠️ PIN 을 쿼리스트링으로 검사하고 있었다

옮기기 전 SnapWord 의 admin API 는 이랬다.

```ts
const ADMIN_PIN = "1956";                       // 공개 저장소에 그대로
if (searchParams.get("pin") !== ADMIN_PIN) ...  // URL 이라 접근 로그에 남는다
```

지금은 **Authorization 헤더 + 공유 비밀**이다(`lib/adminApi.ts`).

- 다섯 배포가 `ADMIN_API_SECRET` 을 같은 값으로 갖는다
- `timingSafeEqual` 로 비교하되 길이를 먼저 본다(길이가 다르면 예외를 던진다)
- **비밀이 설정되지 않았으면 막는다(503).** 열어 두면 아무나 관리 API 를 부른다
- 값이 같은지는 대시보드에서 못 읽는다. `vercel env pull` 로 받아 sha256 앞자리만 비교한다

## 함정

### 집계는 실제 스키마를 보고 짤 것

옮기면서 옛 코드의 집계 두 개가 틀린 것을 발견했다.

- 단어 수를 `$size: "$words"` 로 셌는데 **단어는 단어장 문서에 박혀 있지 않다.**
  `words` 컬렉션에 `vocabId` 로 달려 있어 늘 0 이었다. `$lookup` 으로 고쳤다
- 시험 표가 `totalCount`·`items` 를 읽었는데 실제 필드는 `total`·`correct`·`score` 다

화면에 0 이 찍혀도 아무도 오류라고 알려주지 않는다. **표를 만들면 값이 그럴듯한지
실제 DB 와 대조한다.**

### ⚠️ 로컬에서 앱을 `dev:https` 로 띄우면 포털이 앱을 못 부른다

**2026-09-04에 겪었다.** 관리 화면에서 **앱 조회가 전부 502** 였다.
포털 자기 데이터(회원·권한)는 200 이라 "권한 문제"처럼 보이지 않고
"앱이 다 죽었나" 처럼 보인다.

```
/api/admin/me                    200  ✓
/api/admin/users                 200  ✓
/api/admin/apps/fitlog/stats     502  ✗ "FitLog에 연결할 수 없습니다 (fetch failed)"
```

원인은 스킴이다. `resolveOrigin()` 이 로컬에서 `http://localhost:<포트>` 를 부르는데
`next dev --experimental-https` 로 띄운 앱은 **HTTPS 만 받는다.** 평문 요청에는
연결을 그냥 닫아서 `fetch failed` 만 남는다.

대응 — `.env.local` 에 **`ADMIN_LOCAL_HTTPS=1`**.

- `resolveOrigin()` 이 `https://localhost:<포트>` 를 부른다
- 앱들이 **mkcert 자기서명 인증서**를 쓰므로 Node 가 인증서를 못 믿는다.
  이 플래그가 켜져 있고 개발 모드일 때만 `NODE_TLS_REJECT_UNAUTHORIZED=0` 을 둔다
  (`lib/appAdminApi.ts`). 더 엄격하게 하려면 셸에서 `NODE_EXTRA_CA_CERTS` 를
  mkcert 루트 CA(`%LOCALAPPDATA%\mkcert
ootCA.pem`)로 지정한다 —
  Node 가 **시작할 때** 읽으므로 `.env.local` 에 넣어도 안 먹는다
- 운영에서는 이 완화가 절대 켜지지 않는다

### ⚠️ 선언한 기능을 앱이 갖고 있지 않으면 "불러오는 중…" 에서 멈춘다

`lib/adminApps.ts` 에 `features: ["stats", ...]` 를 적었는데 그 앱에
`/api/admin/stats` 가 없으면, 화면이 영원히 로딩으로 남는다. 같은 날 TypeLog 에서
겪었다 — `stats` 를 선언하고 엔드포인트를 나중에 만들었다.

**기능을 선언하기 전에 엔드포인트를 먼저 만든다.** 순서를 바꾸면 화면이
거짓말을 한다.

### 앱이 안 떠 있을 때를 구분해서 알린다

재배포 중이거나 내려가 있으면 `fetch failed` 가 난다. 그대로 두면 "데이터 없음"과
구분이 안 되므로 **어느 앱인지 이름을 넣어** 알린다
(`SnapNote에 연결할 수 없습니다`).

### 외부 API 를 관리 화면에 매달지 말 것

옛 admin 은 환율을 열 때마다 외부 API 로 가져왔다. 그쪽이 느리면 관리 화면도
같이 느려진다. 뺐다.

## 관련

[[MyJane]] · [[인증과 세션 공유]] · [[MongoDB Atlas]]
