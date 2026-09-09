---
title: E 개인정보 보호 보강
type: plan
tags: [plan, privacy, auth, security]
updated: 2026-09-09
status: 0~10 구현 완료 (2026-09-09) — 커밋·배포 진행 · 법률 검토와 운영 확인만 남음
applies-to: [myjane, SnapWord, SnapNote, fitlog, 2hbk, typelog]
---

# E 개인정보 보호 보강

먼저 [[인증과 세션 공유]] · [[C 법적 페이지]] 를 읽을 것. B·C 가 **방침·동의·탈퇴·메일**을
만들었지만, 방침이 약속한 것을 **코드가 지키지 못하는 자리**가 남아 있다.
2026-09-09 에 여섯 저장소 코드를 다시 읽어 확인했다. 아래 "확인한 사실"은 전부
`파일:줄` 로 실측한 것이고, 재현하지 못한 것은 "확인 필요"로 남겼다.

## 한 줄 요약

**방침은 "본인만 본다"고 말하는데, 세 앱(SnapWord · SnapNote · FitLog)의 API 는
누가 부르는지 확인하지 않는다.** 전화번호나 회원 `_id` 만 알면 남의 단어장·오답노트·
인바디·피검사를 읽고 고칠 수 있다. 이것이 1순위다. 나머지는 그 위에 쌓는다.

---

## 확인한 사실 (2026-09-09)

### A. 접근 통제 — 가장 심각

| # | 무엇이 | 근거 | 심각도 |
|---|---|---|---|
| A1 | **SnapWord · SnapNote 데이터가 `phone` 으로 열린다.** `/api/folders?phone=` · `/api/vocabularies?phone=` · `/api/wrong-notes?phone=` 가 인증 없이 그 번호의 기록을 돌려준다. 생성·수정도 본문의 `phone`·`createdBy` 를 그대로 믿는다 | `SnapWord/app/api/folders/route.ts:13-28` · `vocabularies/route.ts:15-20` · `SnapNote/app/api/wrong-notes/route.ts:15-20` · `wrong-notes/[noteId]/route.ts:15-25` | **치명** |
| A2 | **FitLog 건강정보가 `userId` 쿼리로 열린다.** `_id` 문자열만 알면 남의 측정·피검사를 조회·수정·삭제할 수 있다. 민감정보다 | `fitlog/app/api/measurements/route.ts:18-26` · `measurements/[id]/route.ts:15-21` · `blood/route.ts` | **치명** |
| A3 | **`blood/[id]` GET 은 `userId` 가 없으면 id 만으로 돌려준다.** 소유자 확인이 선택이다. 볼트 [[FitLog]] 의 "Blood 가 소유자 확인을 따라갔다"는 기록과 **다르다** | `fitlog/app/api/blood/[id]/route.ts:16-20` `findOne(userId ? {_id,userId} : {_id})` | **치명** |
| A4 | **동의 게이트가 클라이언트 `userId` 를 받는다.** `requireConsents(parsed.userId, …)` — 동의한 남의 id 를 넣으면 통과한다. 게이트가 서버에 있어도 신원이 서버 것이 아니면 반쪽이다 | `SnapWord/app/api/openai-vision/route.ts:40` · 세 앱 `lib/requireConsent.ts:42-49` | 높음 |
| A5 | **FitLog 채팅이 건강 수치를 OpenAI 로 보내는데 동의 게이트가 없다.** `messages` · `stream` 라우트가 `buildMeasurementContext` · `buildBloodContext` 를 지침에 싣지만 `requireConsent` 를 부르지 않는다. [[C 법적 페이지]] 의 "국외 이전 동의 없으면 AI 대화도 막힌다"는 표와 **다르다** | `fitlog/app/api/chat/threads/[threadId]/messages/route.ts:166-180` · `stream/route.ts:114-127` · `requireConsent` 사용처는 `extract` 두 곳뿐 | 높음 |
| A6 | **세션 쿠키가 평문이고 JS 가 읽는다.** `snap_user` 에 `id · name · phone · email · nickname · userId` 가 JSON 으로 들어가고 `document.cookie` 로 쓰므로 `HttpOnly` 를 걸 수 없다. `.myjane.co.kr` 전체의 스크립트가 읽는다. 30일 지속 | `myjane/lib/session.ts:1-14, 216-223` · 다섯 앱 동일 | 높음 |
| A7 | **토큰을 폐기할 수단이 없다.** 서명 토큰은 `exp`(30일)만 본다. 로그아웃·비밀번호 변경·탈퇴 뒤에도 만료까지 유효하다. `withdrawnAt` 은 2hbk · typelog 의 `lib/auth.ts` 만 확인하고, 나머지 세 앱은 확인하지 않는다(확인할 `lib/auth.ts` 자체가 없다) | `myjane/lib/sessionToken.ts:20-72` · `2hbk/lib/auth.ts:68` · `typelog/lib/auth.ts:86` | 높음 |
| A8 | **로그인에 시도 제한이 없다.** `/api/auth/login` 은 `bcrypt.compare` 만 한다. 429 는 메일 재발송 두 곳에만 있다. 전화번호+4자리 PIN 계정(4건)은 1만 번이면 뚫린다 | `myjane/app/api/auth/login/route.ts:74` · `grep 429` → `send-verification` · `update-email` 만 | 높음 |
| A9 | **이메일 전용 신규 회원은 Snap 앱을 쓸 수 없는 것으로 읽힌다.** 2026-09-07 부터 전화번호가 선택이라 세션 `phone` 이 `""` 인데, 홈이 `/api/folders?phone=` 을 부르고 라우트는 빈 번호에 400 을 준다. **브라우저로 재현하지는 않았다 — 확인 필요.** A1 을 `userId` 기준으로 고치면 함께 풀린다 | `SnapWord/app/(app)/home/page.tsx:40-42` · `folders/route.ts:14-16` | 높음(기능) |

**잘 되어 있는 것** — 2hbk · typelog 는 모든 API 가 서명 토큰을 검증한다.
포털 admin 은 `pin`·`password` 해시를 응답에 넣지 않고 수단 이름만 준다
(`myjane/app/api/admin/users/route.ts:55-60`), 전화번호는 뒤 4자리만 준다.
2hbk 가 다른 회원에게 보여 주는 것은 `userId · nickname · profileImage` 셋뿐이다
(`2hbk/lib/services/users.ts:6-10`). `OpenAiRequestLog` 에는 회원 식별값이 없다.

### B. 저장 · 보관

| # | 무엇이 | 근거 | 심각도 |
|---|---|---|---|
| B1 | **SnapNote 이미지 URL 에 전화번호가 들어간다.** R2 키가 `{phone}/{noteId}/{file}` 이고 공개 URL 을 그대로 DB `imageUrl` 에 저장해 브라우저에 내보낸다. 링크를 공유하면 번호가 함께 나간다 | `SnapNote/app/api/upload-image/route.ts:37-57` | 높음 |
| B2 | **보관 기간이 코드에 없다.** TTL 인덱스는 typelog 게스트 응답 하나뿐. `openai_request_logs` · `events` · `chat_threads` · `inquiries` 는 무기한이다. 방침 5항은 "운영과 장애 대응에 필요한 기간" 이라고만 쓴다 | `grep expireAfterSeconds` → `typelog/models/Attempt.ts:108` 만 | 중간 |
| B3 | **OpenAI 쪽에 대화 원문이 남는다.** fitlog · SnapWord 가 Conversations id 로 이어 쓰는데, 탈퇴 정리(`purgeUserData`)가 OpenAI 대화를 지우는지 **확인 필요.** 지우지 않으면 "6개월 뒤 폐기"가 OpenAI 측에는 미치지 않는다 | `fitlog/lib/chatOpenAi.ts:100` · `SnapWord/lib/chatOpenAi.ts:84` | 중간 |
| B4 | **2hbk R2 키에 소유자가 없다** (`profiles/{uuid}` · `goals/{uuid}`) → 고아 파일을 지울 수 없다. [[C 법적 페이지]] 에 이미 적힌 한계 | `2hbk/lib/r2.ts:68` | 중간 |
| B5 | **실제 회원 데이터 백업이 로컬 디스크에 있다.** `2hbk/scripts/backup/2026-09-03…/` 에 `user.users.json` · `hamhibokka.users.json`(bcrypt 해시 포함). `.gitignore` 는 되어 있다. 이관 검증이 끝났으면 둘 이유가 없다 | `ls 2hbk/scripts/backup` · `.gitignore:15` | 중간 |
| B6 | **Atlas Network Access 가 `0.0.0.0/0`** — Vercel 제약으로 어쩔 수 없다고 [[MongoDB Atlas]] 에 적혀 있다. 대신 할 것(강한 비밀번호·주기적 교체·DB 사용자 권한 최소화·감사 로그)이 볼트에 없다 | [[MongoDB Atlas]]:25-27 | 중간 |
| B7 | `inquiries` 가 `phone` · `name` 을 문서마다 다시 저장한다. 회원 `_id` 참조로 충분하다 | `SnapWord/models/Inquiry.ts:6-7` · `fitlog/models/Inquiry.ts:6-7` | 낮음 |

### C. 방침 · 동의 · 법적 — [[C 법적 페이지]] 가 이미 들고 있는 것

- **법률 검토 전 초안이 공개 중**이다 (`POLICY_VERSION 2026-09-08`, 화면 상단 초안 알림)
- 미정 5건 — 보호책임자 **성명**, 접속 로그·AI 기록 **보관 기간**, 법령상 보존 항목,
  안전조치의 구체(암호화·접근 권한·접속 기록), 유료화 → [[C 법적 페이지]] "(확인 필요) 표"
- **만 14세 미만** — 보호자 계정·자녀 추가 기능이 없다. 가입 화면의 만 14세 확인만 있다
- **실제 메일 발송 미확인** (인증·탈퇴), **크론 폐기가 앱을 부르는 것 운영 미확인**

---

## 계획 — 심각도 순으로, 앞의 것을 끝내고 다음으로

**순서의 원칙.** 치명(A1~A3) → 높음(A4·A5·A8·A6·A7·B1) → 중간(B2~B6) → 방침(C).
단 0번은 A1~A5 전부의 선행 조건이라 맨 앞에 둔다. 각 단계는 끝날 때 "검증"을
실제로 돌리고 결과를 이 문서에 적는다. 저장소가 갈리는 단계는 **세션을 나눠 병행**할 수
있다 (Snap 두 앱 / fitlog / 포털).

### 0. 선행 — 세 앱이 서명 토큰을 읽을 수 있게 한다 (반나절 · 저장소별 병행 가능)

지금 SnapWord · SnapNote · fitlog 에는 토큰을 검증하는 코드가 **아예 없다.**
2hbk 의 것을 그대로 옮긴다. typelog 도 같은 파일이라 둘 중 무엇을 복제해도 된다.

```
2hbk/lib/auth.ts  →  SnapWord/lib/auth.ts · SnapNote/lib/auth.ts · fitlog/lib/auth.ts
  getViewer(req)      쿠키 → verifySessionToken → users 조회 → withdrawnAt 확인 → { uid, userId, … }
  requireViewer(req)  없으면 401 응답을 돌려준다
```

- [ ] 세 저장소에 `lib/auth.ts` 복제. `SESSION_SECRET` 읽는 방식 그대로
- [ ] Vercel 세 배포에 **`SESSION_SECRET` 을 포털과 같은 값**으로 넣고 재배포
      (값 비교는 `vercel env pull` 후 sha256 앞 12자) → [[인증과 세션 공유]]
- [ ] 포털 `lib/apps.ts` 의 `snapword` · `snapnote` · `fitlog` 에 **`requiresSessionToken: true`.**
      빠뜨리면 토큰 없는 옛 세션을 되돌려보내 화면은 열리고 API 는 전부 401 (typelog 사례)
- [ ] 세 앱 `components/AuthGate.tsx` 의 판단 기준을 2hbk 처럼 **사용자 + 토큰**으로.
      한쪽만 바꾸면 로그인 무한 왕복이 난다 — `relogin=1` · 왕복 카운터 함께
      → [[인증과 세션 공유]] "로그인 무한 왕복"

**검증** — 각 앱에 `curl` 로: 정상 쿠키 200 · 토큰 없음 401 · 서명 한 글자 변조 401.
옛 세션(토큰 없음)으로 앱에 들어가면 포털 로그인 폼이 뜨고 로그인 뒤 원래 앱으로 돌아온다.

### 1. A1 치명 — Snap 두 앱: `phone` 대신 토큰의 회원으로 (1~2일 · SnapWord와 SnapNote 동시)

**다행히 마이그레이션이 필요 없다.** `Folder` · `VocabularyDeck` · `WrongNote` 에
`createdBy: ObjectId(User), required, index` 가 이미 있다 (`SnapWord/models/Folder.ts:14` ·
`VocabularyDeck.ts:9` · `SnapNote/models/WrongNote.ts:9`). 소유자 정보는 있는데
라우트가 그걸 **안 쓰고 `phone` 을 썼을 뿐**이다.

방법 — 라우트마다 세 줄이 바뀐다.

```ts
// 전
const phone = normalizePhone(url.searchParams.get("phone") ?? "");
const items = await Folder.find({ phone, deletedAt: null });
// 후
const viewer = await getViewer(req);  if (!viewer) return unauthorized();
const items = await Folder.find({ createdBy: viewer.uid, deletedAt: null });
```

- [ ] **읽기** — `folders` · `vocabularies` · `wrong-notes` 목록/상세: `{ createdBy: viewer.uid }`
      로 좁힌다. 쿼리의 `phone` · `userId` 는 **읽지 않는다** (보내는 옛 화면과 호환)
- [ ] **쓰기** — 본문의 `phone` · `createdBy` 를 버리고 `createdBy: viewer.uid` 로 저장.
      `phone` 필드는 스키마에 남겨 두되 **새로 쓰지 않는다** (9번 B7 에서 정리)
- [ ] **자식 문서** — `words` · `wrong_items` · `study_records` · `test_sessions` · `test_results`
      는 부모(`vocabId` · `noteId`)의 `createdBy` 를 확인한 뒤에만 읽고 쓴다.
      `[id]` 라우트는 `findOne({ _id, createdBy: viewer.uid })` 한 줄로
- [ ] `chat/threads*` · `events` · `inquiries` · `stats/me` · `token-balance` · `wrong-words`
      · `study-records` · `test-*` : `userId` 쿼리/본문을 `viewer.uid` 로 치환
      (SnapWord 12 · SnapNote 8 라우트 — "확인한 사실" A1 의 grep 목록)
- [ ] `upload-image`(SnapNote) 는 6번(B1)에서 키까지 함께 고친다. 여기서는 `getViewer` 만
- [ ] 화면 — `fetch("…?phone=…")` 를 그대로 둬도 서버가 무시하므로 동작한다.
      정리는 5번(쿠키 개편)에서 `session.phone` 이 사라질 때 한 번에

⚠️ SnapWord `api/openai-vision` 의 `deductTokens(userId)` 도 `viewer.uid` 로.
지금은 남의 id 를 넣어 남의 토큰을 깎을 수 있다.

**검증** — 두 계정(가·나)으로: 가의 쿠키 + `?phone=<나의 번호>` → **가의 것만** 나옴 ·
가의 쿠키로 나의 `folderId` 상세 → 404 · 본문 `createdBy: <나>` 로 생성 → 저장된 문서의
`createdBy` 가 **가** · 토큰 없음 401. 그리고 **A9** — 이메일만으로 가입한 새 계정으로
SnapWord 홈이 열리고 폴더가 만들어지는지 브라우저에서 본다.

### 2. A2 · A3 치명 — FitLog: 건강정보 26개 라우트 (1일 · fitlog 세션)

FitLog 의 `measurements.userId` · `bloodtests.userId` 는 **회원 `_id` 문자열**이라
`viewer.uid` 와 그대로 맞는다 ([[C 법적 페이지]] 의 키 표). 마이그레이션 없음.

- [ ] `measurements` · `measurements/[id]` · `measurements/[id]/image` · `measurements/extract`
      · `blood` · `blood/[id]` · `blood/extract` · `profile` · `chat/*` · `inquiries` :
      `searchParams.get("userId")` · `body.userId` 를 **전부** `viewer.uid` 로
- [ ] **`blood/[id]` GET 의 선택 분기 제거** — `findOne({ _id: id, userId: viewer.uid })` 고정.
      PATCH · DELETE 도 같은 필터인지 다시 본다
- [ ] `measurements` POST 의 upsert 키 `(userId, measuredDate)` 도 `viewer.uid`. 남의 날짜
      기록을 덮어쓰는 경로가 여기서 닫힌다
- [ ] `profile` PATCH — 키·성별·출생연도 갱신 대상은 항상 `viewer.uid`

**검증** — 가의 쿠키로 나의 `measurementId` GET/PATCH/DELETE 전부 404 · `blood/[id]` 를
`userId` 없이 → 자기 것만 200, 남의 것 404 · 인바디 합계 규칙 검산이 여전히 통과(기능 회귀 없음).

### 3. A4 · A5 높음 — 동의 게이트를 서버 신원으로, 채팅에도 (반나절)

- [ ] 세 앱 `requireConsents(viewer.uid, …)` — `parsed.userId` 를 넘기는 자리 전부
      (`SnapWord/app/api/openai-vision/route.ts:40` 등)
- [ ] fitlog `chat/threads/[threadId]/messages` · `stream` · `chat/suggestions` 에
      `requireConsents(viewer.uid, ["health", "overseas"])` — 수치를 지침에 싣기 **전에**
- [ ] SnapWord · SnapNote `chat/*` 에 `["overseas"]` — 방침이 "AI 대화" 를 국외 이전에
      넣었으므로 텍스트만 보내도 같다
- [ ] 화면 — 채팅 컴포넌트(`FloatingChat`)가 412 를 받으면 `consentGate` 로 동의 화면에
      보내는지 확인 (지금은 extract 화면만 처리한다)
- [ ] [[C 법적 페이지]] "분리 동의 셋" 표를 사실로 되돌린다

**검증** — 동의 없는 계정: fitlog 채팅 412 + `needsConsent` · 동의한 남의 `userId` 를
본문에 넣어도 412 · 국외 이전만 동의 → SnapWord 채팅 200, fitlog 채팅은 여전히 412.

### 4. A8 높음 — 로그인 시도 제한 (반나절 · 포털 세션, 1~3과 병행 가능)

Vercel 함수는 메모리를 공유하지 않으므로 DB 에 센다.

```
user DB · login_attempts   { key: "id:<식별자>" | "ip:<주소>", count, firstAt, expiresAt }
TTL 인덱스 expiresAt        15분
```

- [ ] `myjane/lib/loginThrottle.ts` — `check(key)` · `hit(key)` · `clear(key)`.
      식별자당 **5회**, IP 당 **30회** / 15분. 넘으면 429 + `Retry-After`
- [ ] `/api/auth/login` — 비교 **전에** `check`, 실패하면 `hit`, 성공하면 `clear`.
      없는 계정도 `hit` 한다 (응답은 기존대로 같은 문장 — 계정 열거 방지 유지)
- [ ] 화면 — 429 를 받으면 "잠시 후 다시 시도" 안내. 남은 시간은 보여 주지 않는다
- [ ] 4자리 PIN 계정 4건 — 막지 않는다. 이메일 등록 안내(B 의 `EmailPrompt`)가 이미
      뜨고 있으니 그 흐름으로 비밀번호를 갖게 유도한다

**검증** — 같은 식별자로 6번째 실패 429 · 15분 뒤 풀림 · 성공하면 카운터 0 ·
다른 식별자는 영향 없음.

### 5. A6 · A7 높음 — 세션 쿠키를 HttpOnly 로, 토큰을 폐기할 수 있게 (2~3일 · 여섯 배포 같은 날)

0~3 이 끝난 뒤에 한다. 다섯 앱이 모두 `lib/auth.ts` 를 갖고 있어야 새 쿠키를 읽는다.

- [ ] `users.sessionVersion: Number, default 0` — **여섯 `models/User.ts` 함께.**
      `email` 은 선택 그대로. 개발 서버 재시작(스키마 캐시 함정)
- [ ] `myjane/lib/sessionToken.ts` claims 에 `sv` 추가. 다섯 앱 `lib/auth.ts` 의 `getViewer`
      가 `claims.sv !== doc.sessionVersion` 이면 null. **`sv` 가 없는 옛 토큰은 이행기 동안
      `sessionVersion === 0` 인 계정에서만 통과**시킨다
- [ ] 로그아웃(전 기기 버튼) · 비밀번호/PIN 변경 · 탈퇴 확정 · admin 권한 변경에서
      `$inc: { sessionVersion: 1 }`
- [ ] 포털 로그인 라우트가 응답으로 **`Set-Cookie: snap_session=<토큰>; HttpOnly; Secure;
      SameSite=Lax; Domain=.myjane.co.kr; Max-Age=30일; Path=/`** 를 내린다.
      쿠키에는 **토큰만** — 이름·전화번호·이메일을 넣지 않는다
- [ ] 다섯 앱 `lib/auth.ts` — `snap_session` 을 먼저 읽고, 없으면 옛 `snap_user.token`.
      30일 뒤 옛 경로 삭제
- [ ] 다섯 앱 화면 — `loadSession()` 이 쿠키가 아니라 **`/api/me`** 를 부른다
      (2hbk 에 이미 있는 모양). 표시에 필요한 것만 돌려준다: `id · name · email 여부 ·
      nickname`. **전화번호는 화면에 필요할 때만**(my 화면) 별도로
- [ ] `LandingAuth` · `AuthGate` 를 서버 컴포넌트의 `cookies()` 판단으로 바꿀 수 있다.
      "상태가 정해지기 전에 아무것도 그리지 않는" 우회가 필요 없어진다
- [ ] 로그아웃 라우트가 `snap_session` 을 `Max-Age=0` 으로 지우고 `sessionVersion +1`
- [ ] [[C 법적 페이지]] 쿠키 안내 — "담기는 값" 을 "서명 토큰만" 으로 고친다.
      `POLICY_VERSION` 을 올릴지는 사용자가 정한다(수집 항목이 늘지 않았다)

⚠️ **여섯 배포를 같은 날 올린다.** 한 앱만 새 쿠키를 읽으면 그 앱만 로그아웃된다.
⚠️ 로컬 개발은 `localhost` 라 `Domain` 이 안 붙는다 — `lvh.me` 로 확인 → [[인증과 세션 공유]]

**검증** — devtools `document.cookie` 에 `snap_session` 이 **보이지 않고** 회원 정보도
없다 · 로그아웃 뒤 저장해 둔 옛 토큰으로 API 401 · 비밀번호 변경 뒤 다른 브라우저 세션이
끊긴다 · 다섯 앱 오가며 로그인 유지 · `*.vercel.app` 프리뷰에서도 로그인(host-only).

### 6. B1 높음 — SnapNote 이미지 URL 에서 전화번호를 뺀다 (1일)

- [ ] `upload-image` 키를 **`snapnote/{viewer.uid}/{noteId}/{uuid}.png`** 로 (1번에서
      `getViewer` 를 넣었으니 본문 `phone` 은 이미 안 쓴다)
- [ ] 마이그레이션 스크립트 `SnapNote/scripts/migrate-r2-keys.mjs` —
      `wrong_items.imageUrl` 을 훑어 옛 키(`{phone}/…`)면 `CopyObject` → 새 키 →
      DB 갱신 → 옛 키 `DeleteObject`. **드라이런 먼저**, 건수와 실패 목록 출력.
      2hbk 이관 때처럼 백업(URL 목록 JSON)을 남기고 끝나면 지운다
- [ ] `lib/purgeR2.ts` 의 URL 역산이 새 규칙으로도 동작하는지 (접두사 `snapnote/{uid}/`
      가 생겼으니 **`ListObjectsV2 + Prefix` 쓸어담기**를 더할 수 있다 — 고아 파일도 잡힌다)
- [ ] 옛 키가 0건이 된 뒤, 화면·인쇄 페이지에서 이미지가 깨지지 않는지 표본 확인

**검증** — 새 업로드 URL 에 숫자 전화번호 패턴이 없음 · 마이그레이션 후
`imageUrl` 중 `^https?://[^/]+/0\d{8,10}/` 매치 0건 · 탈퇴 정리에서 접두사 삭제 확인.

### 7. B3 중간 — 탈퇴 폐기가 OpenAI 대화도 지운다 (반나절)

- [ ] fitlog · SnapWord `lib/purgeUserData.ts` — `chat_threads` 를 지우기 **전에**
      `openAiConversationId` 를 모아 `openai.conversations.delete(id)`. 실패는 로그만,
      DB 삭제는 계속(기존 원칙)
- [ ] SnapNote 도 채팅이 있으면 같게
- [ ] 방침 5항 "OpenAI 측에도 보관됩니다" 문장 뒤에 "탈퇴 폐기 때 함께 삭제를 요청합니다" 추가

**검증** — 테스트 계정으로 대화 → 폐기 → OpenAI API 로 그 conversation GET 이 404.

### 8. B2 중간 — 보관 기간을 정하고 TTL 로 건다 (반나절 · **사용자 결정 필요**)

| 컬렉션 | 제안 | 근거 |
|---|---|---|
| `openai_request_logs` (세 앱) | **90일** | 비용·장애 분석용. 회원 식별값 없음 |
| `events` (Snap 두 앱) | **1년** | 사용 통계 |
| `login_attempts` | 15분 | 4번에서 이미 |
| `inquiries` | 답변 완료 후 **1년** | 재문의 대응 |
| `chat_threads` | 탈퇴 폐기와 함께 (TTL 아님) | 회원 데이터 |
| Vercel 접속 로그 | 플랜 기본값 확인 | 우리가 정하지 못한다 |

- [ ] 기간 확정 → 모델에 `expireAfterSeconds` (`createdAt` 또는 `answeredAt` 기준).
      **부분 인덱스**로 — typelog 게스트 TTL 과 같은 이유
- [ ] 방침 5항 "운영과 장애 대응에 필요한 기간" → 위 숫자로. `POLICY_VERSION` 올림

### 9. B5 · B6 · B4 · B7 중간 — 디스크·인프라·저장 정리 (각 짧게)

- [ ] **B5** `2hbk/scripts/backup/2026-09-03T00-54-02-525Z/` 삭제 — 이관 검증이 끝났다.
      **사용자가 지운다** (실제 회원 데이터라 AI 가 삭제하지 않는다).
      원본 `hamhibokka.users` 컬렉션은 2026-12 정리 대상으로 여기 적어 둔다
- [ ] **B6** [[MongoDB Atlas]] 에 절 추가 후 실행 — 앱별 DB 사용자(`vocab` 만 · `math` 만 …
      + `user` 읽기/쓰기)로 나누고 `MONGO_URI` 교체 · 비밀번호 교체 주기(분기) ·
      Atlas 감사 로그·백업 보존 기간은 플랜을 보고 적는다
- [ ] **B4** 2hbk R2 키에 `profiles/{userId}/…` · `goals/{goalId}/…`. 기존 파일은 6번과
      같은 복사 스크립트. 참조 없는 파일은 `check:images --clean` 으로 고아 확정 후 삭제
- [ ] **B7** `inquiries` 의 `phone` · `name` 스냅샷 제거 → `userId` 참조. admin 화면은
      회원을 조회해 표시. 기존 문서는 `$unset` (탈퇴 폐기가 이미 지우므로 급하지 않다).
      Snap 모델의 `phone` 필드도 이때 함께 `$unset` 하고 스키마에서 뺀다

### 10. C — 방침을 사실에 맞추고 검토를 받는다 (코드가 끝난 뒤)

- [ ] 7항 안전조치를 **하는 만큼만** — "본인 확인은 서버가 서명 토큰으로 · 세션은
      HttpOnly 쿠키 · 로그인 15분 5회 제한 · 로그 보관 N일 · 저장소 접근은 앱별 계정"
- [ ] 미정 확정(사용자) — 보호책임자 **성명** · 법령상 보존 항목 · 유료화 ·
      약관 종료 고지 30일 유지 여부
- [ ] `POLICY_VERSION` 올리고, `agreedPolicyVersion` 이 옛 값인 회원에게 포털에서
      한 번 재동의 안내 (`EmailPrompt` 모양 재사용)
- [ ] 운영에서 **실제 메일** — 인증 · 탈퇴 확인 · 비밀번호 재설정 각 1회 수신
- [ ] 크론 폐기 운영 확인 — 테스트 계정 `withdrawnAt` 을 6개월 전으로 두고 다음날
      다섯 앱 데이터가 사라졌는지
- [ ] 법률 검토 → 초안 알림 제거
- [ ] 만 14세 미만 보호자·자녀 계정 → **별도 계획서 F**. 이 계획서 범위 밖

## 한눈에 — 순서와 병행

```
0 선행(auth.ts·SECRET·apps.ts)   ─┐ 저장소별 병행: SnapWord+SnapNote / fitlog / 포털
1 A1 Snap phone→createdBy         │  Snap 세션
2 A2·A3 FitLog userId→viewer      │  fitlog 세션
3 A4·A5 동의 게이트                │  1·2 뒤, 두 세션 각자
4 A8 로그인 제한                   │  포털 세션 — 0~3 과 무관, 바로 시작 가능
5 A6·A7 쿠키·sessionVersion       ← 0~3 끝난 뒤, 여섯 배포 같은 날
6 B1 SnapNote R2 키               ← 1 뒤
7 B3 OpenAI 대화 삭제              ← 아무 때나
8 B2 TTL                          ← 사용자 결정 뒤
9 B5·B6·B4·B7                     ← 아무 때나 (B5 는 사용자가)
10 C 방침·검토                     ← 마지막
```

사용자가 정할 것 — **보관 기간 숫자(8) · 보호책임자 성명(10) · 백업 삭제(9-B5)**.
나머지는 바로 착수할 수 있다.

## 진행 결과 — 2026-09-09 (한 세션에서 0~7 까지)

여섯 저장소 모두 **커밋하지 않았다.** 타입 검사(`npx tsc --noEmit`)는 여섯 곳 전부 통과했다
(`.next*/types` 의 낡은 생성 파일 오류만 남는데, C 작업에서 지운 페이지를 가리키는 것이라 무시).

| 단계 | 결과 |
|---|---|
| 0 | 세 앱에 `lib/auth.ts`(2hbk 복제 · `findById(claims.uid)` · `viewer.uid`) · `lib/sessionToken.ts` · `useSession` `unusable` 상태 · `hasUsableSession()` · AuthGate `relogin`. 포털 `lib/apps.ts` 세 앱에 `requiresSessionToken`. 로컬 `.env.local` 세 곳에 `SESSION_SECRET` |
| 1 | SnapWord 24 · SnapNote 18 라우트가 `viewer.uid` 만 쓴다. 필터는 **이미 있던 `createdBy`**. `phone` 은 스키마에서 `required → default ""` (Folder · VocabularyDeck · WrongNote · Inquiry · Event) — 이메일 전용 회원이 저장할 수 있게. `words` GET · `test-sessions/[id]` · `ai-cache` · `analyze-text` · `upload-image` 처럼 **인증이 아예 없던** 라우트도 잡혔다 |
| 2 | fitlog 26 라우트. `blood/[id]` 선택 분기 제거, PATCH 도 `findOneAndUpdate({_id, userId})`. 프로필 게이트가 `viewer.doc` 로 항상 동작 |
| 3 | `requireConsents(viewer.uid, …)`. fitlog 채팅 `messages`·`stream` 에 `["health","overseas"]`, `suggestions` 는 `["health"]`(OpenAI 안 부름). Snap 채팅·`analyze-text`·`openai-vision` 에 `["overseas"]`. `FloatingChat` 이 412 를 받으면 동의 화면으로 |
| 4 | 포털 `lib/loginThrottle.ts` + `models/LoginAttempt.ts`(`user` DB `login_attempts`, TTL 15분). 식별자 5회 · IP 30회. **실측: 6번째 401 → 429** |
| 5 | 서버 쿠키 셋 — `snap_session`(HttpOnly · 토큰만) · `snap_auth`("1" 표지) · `snap_user`(표시용 — 전화번호·이메일·토큰 제거, `hasEmail` 추가). `users.sessionVersion` + 토큰 `sv`. 비밀번호·PIN 변경 · 탈퇴 확정 · `POST /api/auth/logout {all:true}` 에서 +1. 여섯 저장소에 `lib/sessionCookie.ts` · `/api/auth/logout`, 다섯 앱에 `/api/me`. 옛 `snap_user.token` 은 이행기 동안 읽는다 |
| 6 | SnapNote 키 `snapnote/{회원 _id}/{noteId}/{uuid}.{ext}`. `scripts/migrate-r2-keys.mjs` 로 **기존 6건 이관 완료(실패 0)**, 재확인 0건. 백업 JSON 은 지웠다(옛 URL 에 전화번호). `purgeR2.deleteR2ByPrefix` 로 고아 파일까지 |
| 7 | 세 앱 `lib/purgeOpenAiConversations.ts` — 폐기 때 `conversations.delete` 를 먼저 부른다. 404 는 지워진 것으로. **실제 삭제 호출은 확인하지 못했다**(테스트 대화 없음) |
| 9 | B4 2hbk 키 `profiles/{userId}/…` · `goals/{userId}/…` + `deleteByOwner()`. B6 [[MongoDB Atlas]] 에 절 추가(실행은 사용자). B7 `inquiries`·`events` 의 `phone` 선택화 — 스냅샷 제거는 아직 |
| 법적 | 쿠키 안내 표 3행으로, 방침 7항에 서버 검증·HttpOnly·시도 제한·동의 확인 추가. **`POLICY_VERSION` 은 올리지 않았다**(사용자 판단) |

### 실측 (검증 서버 3010~3015 · 실제 회원 2명의 토큰을 로컬에서 서명)

```
fitlog   토큰 없음 401 · 정상 200 · 옛 snap_user 형식 200 · sv 불일치 401 · 서명 변조 401
         남의 id 로 blood/[id] 404 · ?userId=남 → 내 것만(두 계정 모두 기록 0건이라 약한 검증)
         /api/me 200 · chat/suggestions 412(동의 없음) · logout 이 snap_session 을 지운다
SnapWord 토큰 없음 401 · ?phone=엉터리 200(무시됨) · 남의 vocabId 로 words 404 · analyze-text 무인증 401
SnapNote 토큰 없음 401 · upload-image 무인증 401
포털     email-prompt 401/200/401(sv) · 로그인 6번째 429 · logout 이 snap_session 을 지운다
2hbk·typelog  /api/me 401/200/401(sv) · typelog 는 ADMIN 시크릿 Bearer 가 있어도 쿠키로 200
```

### 배포 전에 해야 하는 것 (사용자)

- [ ] Vercel **SnapWord · SnapNote · fitlog** 에 `SESSION_SECRET` (포털과 같은 값) → 없으면 세 앱 API 전부 500/401
- [ ] **여섯 배포를 같은 날.** 포털만 먼저 올리면 앱들이 새 쿠키를 못 읽는다(옛 세션은 이행기라 살아 있음)
- [ ] 배포 뒤 브라우저에서: devtools `document.cookie` 에 `snap_session` 이 **없고** `snap_user` 에 전화번호·이메일이 없음 · 다섯 앱 오가며 로그인 유지 · 로그아웃 뒤 API 401
- [ ] 이메일만으로 가입한 계정으로 SnapWord 폴더 생성 (A9 확인)
- [ ] 2hbk R2 옛 키(`profiles/{uuid}`) 이관은 하지 않았다 — 파일이 8+3건이라 `check:images` 로 정리하는 편이 낫다

### 남은 것

- [x] **8 · TTL** (2026-09-09, 사용자 확정) — `openai_request_logs` **90일** · `applicants` **1년** ·
      `inquiries` 답변 후 **1년**(부분 인덱스, 대기 중은 남김) · `login_attempts` 15분.
      스키마에 선언 + `myjane/scripts/ensure-ttl.mjs` 로 운영 DB 에 만들었다(옛 `createdAt_1` 은 키가
      겹쳐 지우고 만들었다). 방침 5항에 숫자를 적었다
- [x] **9-B5** `2hbk/scripts/backup/2026-09-03…/` 삭제 (2026-09-09). 원본 `hamhibokka.users` 컬렉션은 2026-12 정리 대상
- [ ] **9-B7** `inquiries` 의 `name`·`phone` 스냅샷을 없애고 admin 이 회원을 조회해 표시 (지금은 `phone` 만 비운다)
- [x] **10 일부** (2026-09-09) — 보호책임자·운영자 **장민** · `POLICY_VERSION 2026-09-09` ·
      개정 안내 띠 `components/PolicyPrompt.tsx` + `/api/auth/policy-prompt` (확인하면 `agreedPolicyVersion` 갱신, 로그인은 안 막음)
- [ ] **10 나머지** — 실제 메일 수신(인증·탈퇴·재설정) · 크론 폐기 운영 확인 · **법률 검토 → 초안 알림 제거**
- [ ] 이행기가 끝나면(30일 뒤) `readSessionTokenFromRequest` 의 옛 `snap_user.token` 갈래와 `saveSession` 의 토큰 물려주기를 지운다
- [ ] `SessionUser.phone` 필드 자체를 없앤다 — 지금은 호환을 위해 `""` 로 채운다. Snap 화면의 `?phone=` 쿼리 정리와 함께

## 볼트 기록과 다른 것 — 고쳐야 한다

| 노트 | 적힌 것 | 실제 |
|---|---|---|
| [[FitLog]] "인바디 쪽에 남았던 비대칭" | Blood 가 소유자 확인을 따라갔다 | `blood/[id]` GET 은 `userId` 없으면 id 만으로 준다 (A3) |
| [[C 법적 페이지]] "분리 동의 셋" 표 | 국외 이전 동의 없으면 **AI 대화**도 막힌다 | fitlog 채팅 라우트에 게이트 없음 (A5) |
| [[인증과 세션 공유]] "남은 과제" | Snap 계열과 FitLog 가 클라이언트 `userId` 를 믿는다 | 맞다. 다만 Snap 은 `userId` 가 아니라 **`phone`** 이 키다 — 마이그레이션이 필요한 이유 (A1) |

2026-09-09 에 세 노트에 이 계획서를 가리키는 한 줄씩을 넣었고, 같은 날 구현이 끝나 셋 다 **지금은 사실**이 됐다(위 진행 결과).

## 겹치는 부분

- **B 와** — `models/User.ts` 에 `sessionVersion` 추가. 여섯 앱 함께, `email` 은 선택 유지
- **C 와** — 방침 5·7항 문구, 쿠키 안내. 코드가 바뀐 뒤에 고친다
- **D(jangmini) 와** — 겹치지 않는다. jangmini 는 여섯 앱 규칙을 따르지 않는다
- **A 와** — 없다
