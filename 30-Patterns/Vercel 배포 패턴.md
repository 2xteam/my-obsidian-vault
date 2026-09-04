---
title: Vercel 배포 패턴
type: pattern
tags: [pattern, vercel, infra]
updated: 2026-09-04
---

# Vercel 배포 패턴

SnapWord · SnapNote · myjane · FitLog · 2hbk를 옮기며 실제로 겪은 것만 적는다.
새 프로젝트를 올릴 때 이 문서를 먼저 본다.

## 프로젝트 생성

1. Add New → Project → 저장소 Import
2. **Node.js Version 22.x** — 20 미만에서는 `File` 글로벌이 없어 이미지 업로드가 502
3. **Production Branch 확인** — SnapNote는 `master`다. 기본값(main)을 그대로 두면 배포가 안 된다
4. **Framework Preset이 `Next.js`인지 확인** — 아래 함정 참고
5. `vercel.json`에 리전과 프레임워크를 고정

```json
{ "regions": ["icn1"], "framework": "nextjs" }
```

사용자·Atlas가 모두 한국이라 서울 리전이 체감 성능을 좌우한다. Hobby 플랜에서도 적용된다.

### ⚠️ Framework Preset이 `Other`로 잡히면 조용히 404

2026-09-03 2hbk, **2026-09-04 TypeLog에서 또 겪었다.**
**빌드는 성공(`● Ready`)하는데 `/`가 404**다.
커스텀 도메인은 물론 `<프로젝트>.vercel.app`까지 404라 도메인 문제로 착각한다.

TypeLog는 **저장소가 비어 있을 때 프로젝트를 만들어서** 감지할 것이 없었다.
코드를 올리기 전에 Vercel을 연결하면 반드시 이렇게 된다.

> ⚠️ **다른 앱의 `vercel.json`을 복사하면 이 대응이 사라진다.** FitLog의 파일에는
> `framework`가 없다 — 프리셋이 자동 감지돼서 필요가 없었기 때문이다.
> 그걸 새 프로젝트에 복사하면 이 함정을 그대로 밟는다. TypeLog가 그랬다.

원인은 Output Directory다. `Other` 프리셋은

```
Build Command      npm run build          ← 잘 돈다
Output Directory   `public` if it exists  ← 정적 사이트로 취급한다
```

`public/`에 `index.html`이 없으니 내보낼 게 없고, **Next.js 서버·함수가 아예
연결되지 않는다.** 에러도 경고도 없다.

확인 — `npx vercel project inspect <프로젝트>`의 Framework Settings를 본다.

**한 줄로 가려내는 법**: `public/`에 있는 파일을 직접 불러 본다.

```bash
curl -s -o /dev/null -w "%{http_code}\n" https://<도메인>/            # 404
curl -s -o /dev/null -w "%{http_code}\n" https://<도메인>/icon.png    # 200 ← 이 조합이면 확정
```

**루트는 404인데 `public/` 파일은 200**이면 `public/`이 출력 디렉터리로 서빙되고
있다는 뜻이다. 도메인·DNS 문제라면 둘 다 404다. TypeLog에서 DNS·도메인 검증·
alias·빌드 로그를 전부 확인하고 나서야 여기에 도달했는데, 이 두 줄이면 처음부터
좁힐 수 있었다.

대응 — **`vercel.json`에 `"framework": "nextjs"`를 적는다.** 이 값이 대시보드
설정을 덮으므로 저장소에 두면 다시 어긋날 일이 없다. 대시보드에서만 고치면
프로젝트를 다시 만들 때 똑같이 밟는다.

## 환경 변수 — 함정 3개

### ① 배포 시점에 고정된다

환경 변수는 **그 배포를 만들 때 스냅샷**된다. 나중에 추가해도 기존 배포는 모른다.
→ **Deploy 누르기 전에 전부 입력**한다. 이미 배포했다면 Redeploy.

### ② 프로젝트 설정 vs 계정 공용

사이드바의 "Environment Variables"는 **계정(팀) 공용** 페이지다.
여기 등록한 변수는 프로젝트에 연결하지 않으면 함수에 주입되지 않는다.
반드시 `프로젝트 → Settings → Environment Variables` 에 넣는다.
URL에 `/~/`가 있으면 공용 페이지다.

### ③ `NEXT_PUBLIC_*`은 Secret이 아니라 **Config**

`NEXT_PUBLIC_*`은 브라우저 번들에 문자열로 박혀야 동작한다.
**Secret 타입은 브라우저에 노출되지 않아 값이 주입되지 않는다.**
게다가 저장된 Secret은 write-only라 Config로 바꿀 수 없다 → **삭제 후 재등록**해야 한다.

증상: 기능이 조용히 동작하지 않는다(예: 쿠키 도메인 미적용).
확인법:

```bash
# 배포된 번들에 값이 박혀 있는지 직접 확인
curl -sS https://<도메인>/ | grep -o '/_next/static/chunks/[^"]*\.js' | sort -u \
  | while read p; do curl -sS "https://<도메인>$p" | grep -l "찾는값" && echo "$p"; done
```

빌드 캐시를 재사용하면 예전 번들이 남을 수 있으므로 **"Use existing Build Cache" 해제** 후 재배포.

## 플랫폼 제약

| 항목 | 값 | 대응 |
|---|---|---|
| 요청 본문 | **4.5MB** | 함수에 닿기 전에 `FUNCTION_PAYLOAD_TOO_LARGE`로 잘린다 → [[이미지 업로드 패턴]] |
| 함수 실행시간 | Hobby 최대 60초 | `maxDuration`이 60을 넘으면 **배포가 거부된다** |
| 고정 IP | 없음 | Atlas Network Access에 `0.0.0.0/0` 필요 |

`experimental.serverActions.bodySizeLimit`을 아무리 올려도 소용없다. 플랫폼 한도가 우선한다.

## 도메인 연결 (가비아 기준)

1. Vercel → Settings → Domains에 도메인 추가 → 표시되는 값 복사
2. My가비아 → 서비스관리 → 도메인 → DNS 정보 → **DNS 관리**
3. 서브도메인은 **CNAME**, apex(`@`)는 **A 레코드** (가비아는 apex에 CNAME 불가)
4. 같은 호스트에 A와 CNAME은 공존 불가 → **기존 A를 먼저 삭제**
5. 호스트 칸에는 서브도메인만 (`snapword`, `www`). 전체 도메인을 넣으면 중복된다
6. 가비아는 CNAME 값 끝에 마침표를 요구할 수 있다 (`xxx.vercel-dns-017.com.`)
7. 저장 후 Vercel에서 **Refresh** — 검증·인증서 발급이 빨라진다

TTL을 600으로 낮춰두면 롤백이 10분 내에 된다.

## 배포 확인

```bash
curl.exe -sSI https://<도메인>/
```

`Server: Vercel` + `x-vercel-id: icn1::...` 이면 전환 완료.
`Server: nginx`가 보이면 아직 예전 서버이거나 로컬 DNS 캐시(`ipconfig /flushdns`).

## 요금

Hobby는 약관상 비상업적 용도다. 실사용 서비스를 운영 도메인에 붙이려면 Pro 전환을 검토한다.
