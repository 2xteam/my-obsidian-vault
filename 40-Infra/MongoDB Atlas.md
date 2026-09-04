---
title: MongoDB Atlas
type: infra
tags: [infra, mongodb]
updated: 2026-09-04
---

# MongoDB Atlas

Cluster0 하나에 서비스별 DB를 나눠 쓴다 → [[인증과 세션 공유]]

| DB | 쓰는 곳 |
|---|---|
| `user` | 회원 — 다섯 앱 공용 |
| `vocab` | SnapWord |
| `math` | SnapNote |
| `fit` | FitLog |
| `hamhibokka` | 2hbk |
| `type` | TypeLog |

`admin` · `local` 은 MongoDB 시스템 DB다. 우리가 만든 것이 아니다.

## Vercel에서 접속하려면

Network Access에 **`0.0.0.0/0`** 이 있어야 한다. Vercel Functions는 고정 IP가 없다.
고정 IP나 Private Endpoint는 상위 유료 플랜이 필요하므로, 현실적인 대응은
`0.0.0.0/0` + 충분히 강한 DB 비밀번호다.

## 함정

### 무료(M0) 클러스터 자동 일시정지

일정 기간 미사용 시 **Paused** 상태가 된다. 이 상태에서는 연결이
`serverSelectionTimeoutMS`만큼 기다리다 타임아웃된다.
증상이 "환경 변수 문제"처럼 보이지만 Atlas 대시보드에서 **Resume**하면 끝이다.
(2026-09-01 SnapWord 이관 중 실제로 겪음)

### URI 경로에 DB 이름을 의존하지 말 것

**2026-09-02 사고** — FitLog 운영 환경의 `MONGO_URI`가 `.../math`(SnapNote의 DB)를
가리키고 있었다. 다른 앱의 연결 문자열을 복사해 넣은 것이다.

- 로그인은 정상이었다. `users`는 `MONGO_USER_DB`(기본 `user`)로 따로 골라 쓰기 때문
- 측정 기록만 조용히 `math` DB에 쌓였다. **에러도 경고도 없었다**
- 증상: 로컬에서는 8건이 보이는데 운영에서는 방금 넣은 1건만 보였다

대응 — 앱마다 **`dbName`을 코드에서 명시**한다. URI 경로가 무엇이든 이 값이 이긴다.

```ts
const MONGODB_DB = "fit"; // 상수. 환경 변수로 바꿀 수 없다
mongoose.connect(MONGODB_URI, { dbName: MONGODB_DB, ... });
```

> ⚠️ **2026-09-04 같은 사고가 다시 났다.** TypeLog 의 `quizzes`·`resulttypes`·
> `attempts` 33건이 2hbk 의 `hamhibokka` DB 안에 들어가 있었다. `lib/db.ts` 는
> "URI 경로를 믿지 않는다" 며 `dbName` 을 명시하고 있었는데도다 — 그 값이
> `process.env.MONGO_DB ?? "type"` 이었고, 2hbk 의 `.env.local` 을 복사할 때
> **`MONGO_DB=hamhibokka` 도 같이 따라왔다.** URI 경로를 안 믿는 대신 환경 변수를
> 믿은 셈이다.
>
> **DB 이름을 환경 변수로 받지 않는다.** 그 앱의 고정된 사실이므로 코드에만 둔다.
> `MONGO_USER_DB` 도 같은 이유로 없앴다(공유 DB `user` 는 서비스 전체의 상수다).
> 증상은 여전히 조용하다 — 오류도 경고도 없고, `type` DB 는 컬렉션 3개가 전부
> 0건인 채로 있었다.

| 앱 | dbName |
|---|---|
| SnapWord | `vocab` |
| SnapNote | `math` |
| FitLog | `fit` |
| 2hbk | `hamhibokka` |
| TypeLog | `type` |

통합 admin 도 이 클러스터를 쓰지만 **앱 DB 를 직접 읽지는 않는다.**
회원(`user`)만 직접 읽고 나머지는 각 앱의 API 를 거친다 → [[통합 admin]]

**`MONGO_DB` 환경 변수는 두지 않는다.** 있으면 언젠가 다른 앱 값이 복사돼 들어온다.
`.env.example` 에도 적지 않는다 — 적어두면 채우게 된다.

### 스키마에 필드를 더했는데 저장되지 않으면 — 모델 캐시

> ⚠️ **2026-09-04.** `quizzes` 스키마에 `disclaimer` 를 더하고 등록했는데 필드가
> 아예 생기지 않았다. 오류도 경고도 없었고 화면에만 아무것도 안 떴다.
>
> mongoose 는 컴파일한 모델을 `mongoose.models` 에 담아 두는데 이 객체는
> `globalThis` 에 있어서 **Next 의 HMR 을 넘어 살아남는다.** 그래서 스키마를
> 고쳐도 옛 모델이 계속 쓰이고, `strict` 가 모르는 경로를 `$set` 에서
> **조용히 버린다.** 개발 서버를 재시작하기 전까지 계속 그렇다.
>
> 대응 — 모델을 가져오는 자리를 한 곳으로 모으고, **개발 중 스키마 객체가
> 바뀌었을 때만** 다시 컴파일한다. HMR 로 모듈이 다시 실행되면 스키마는 새
> 객체가 되므로 동일성 비교로 알 수 있다. 매 요청마다 지우고 만들지 않는다.
>
> ```ts
> // lib/model.ts
> const existing = mongoose.models[name];
> if (existing && process.env.NODE_ENV !== "production" && existing.schema !== schema) {
>   mongoose.deleteModel(name);
> }
> return mongoose.models[name] ?? mongoose.model(name, schema, collection);
> ```
>
> `useDb()` 로 얻은 다른 연결의 모델은 `mongoose.models` 가 아니라 그 연결에
> 달려 있어 따로 처리한다(`defineModelOn`).
>
> **증상을 기억할 것: 새 필드가 없는데 아무도 알려주지 않는다.** 스키마를 고친
> 뒤에는 등록해 보고 **DB 를 직접 확인**한다.

## 새 앱의 DB 를 만들 때

**앱별 DB 는 Atlas 에서 미리 만든다.** 기술적으로는 mongoose 가 첫 쓰기에 자동으로
만들지만, 그렇게 두면 배포하고 한참 뒤에야 DB 목록에 나타나 **어느 앱 것이 있고
없는지 대시보드만 봐서는 알 수 없다.** 미리 만들어 두면 목록이 곧 앱 목록이 된다.
(2026-09-04 사용자 확인 — 지금까지 그렇게 해 왔다)

> ⚠️ **Atlas 는 빈 DB 를 만들 수 없다.** DB 를 만들 때 컬렉션 하나를 함께 만들어야
> 하고, 그 컬렉션을 지우면 DB 도 사라진다. 그래서 첫 컬렉션 하나는 손으로 만든다.

1. Atlas → Browse Collections → **Create Database**
   - Database name: 그 앱의 dbName (아래 표)
   - Collection name: 그 앱의 첫 컬렉션 하나 (나머지는 mongoose 가 만든다)
2. **DB 사용자에게 그 DB 의 readWrite** — 기존 사용자가 `readWriteAnyDatabase` 면 그대로 된다
3. **Network Access `0.0.0.0/0`** — 이미 있다
4. **`MONGO_URI`** 를 `.env.local` 과 Vercel 프로젝트에 넣고 **재배포**
   (환경 변수는 배포 시점에 스냅샷된다 → [[Vercel 배포 패턴]])

인덱스는 모델을 처음 쓸 때 mongoose 가 만든다(`autoIndex` 기본값이 켜져 있다).
컬렉션도 첫 쓰기에 생기므로 1번에서 전부 만들 필요는 없다.

TypeLog 에 점검 스크립트를 뒀다(`scripts/check-db.mjs`, `npm run db:check`).
연결·**DB 이름 대조**(URI 경로 vs 코드가 못 박은 값)·컬렉션·인덱스·환경 변수를
한 번에 보여준다. `-- --ensure` 를 붙이면 빠진 인덱스를 만든다.
다른 앱에도 그대로 복사해 쓸 수 있다.

### 연결 문자열 형식

Windows 로컬에서 SRV 조회가 실패하면 표준 URI(`mongodb://` + 샤드 3개)를 쓰게 되는데,
**Atlas가 클러스터를 이전하면 샤드 호스트명이 바뀌어 조용히 끊긴다.**
서버는 SRV 조회에 문제가 없으므로 **Vercel 환경 변수에는 `mongodb+srv://`** 를 쓴다.

## 서버리스 연결 설정

```ts
mongoose.connect(MONGODB_URI, {
  maxPoolSize: 5,          // 인스턴스가 여러 개 뜨므로 인스턴스당 풀은 작게
  bufferCommands: false,
  serverSelectionTimeoutMS: 20000,
});
```

커넥션 캐시는 **프로덕션에서도 `globalThis`에** 둔다. 인스턴스 재사용 시 물려받는다.
