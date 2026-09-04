---
title: 설문지 JSON 작성 지침
type: pattern
tags: [pattern, typelog, quiz, ai]
updated: 2026-09-04
---

# 설문지 JSON 작성 지침

[[TypeLog]]에 질문지를 등록할 때 쓰는 규격이다.
**AI가 이 노트를 읽고 JSON을 만들고, 관리자는 그것을 붙여넣기만 한다.**
관리자는 문항을 하나하나 확인하지 않는다 — 세부는 사람이 아니라 **검증기**가 본다.

데이터 모델의 배경(설문지 생애주기, POMP가 왜 필요한지)은 [[TypeLog]]에 있다.

## 언제 쓰나

모드는 **둘뿐**이다. 부분 수정을 하지 않고 언제나 **전체를 다시 만들어 한 번에
붙여넣는다.**

| `mode` | 무엇을 보내나 | 언제 |
|---|---|---|
| `upsert` | 다섯 덩어리 전부 | 만들 때 · `draft`에서 고칠 때 |
| `content` | `quizSlug` + `resultTypes`만 | **공개 후** 결과 문장을 다듬을 때 |

`upsert`가 설문지 상태에 따라 하는 일 —

| 설문지 상태 | `upsert` 를 보내면 |
|---|---|
| 없음 | 새로 만든다 (`draft`로 생성) |
| `draft` (off) | **문항·outcomes·resolver·resultTypes를 통째로 교체**한다. 옛 내용은 남지 않는다 |
| `published` (on) | **거부한다.** 잠긴 설문지다 |

문항을 고치고 싶으면 `draft`로 되돌린 뒤(응답 0건일 때만 가능) 전체를 다시 보낸다.
응답이 있으면 되돌릴 수 없다 — 새 `slug`로 만든다.

`content`는 공개 후에도 쓸 수 있다. **결과 문장은 응답 계산에 영향이 없기 때문**이다.
오타 하나 고치려고 새 `slug`를 만들 수는 없다. 단 `code`·`vector`는 바꿀 수 없다 —
그건 채점 결과와 이어져 있다.

```json
{ "$schema": "typelog/quiz/v1", "mode": "content",
  "quizSlug": "animal-friend",
  "resultTypes": [ { "code": "dolphin", "name": "다정한 돌고래", "emoji": "🐬",
                     "content": { "summary": "...", "strengths": ["..."] } } ] }
```

## 등록 흐름

```
1. AI가 이 노트의 규격대로 JSON을 만든다 (이미지·이모지도 조사해서 채운다)
2. 관리자가 /admin/quizzes 의 입력창에 붙여넣고 [검사]
3. 검증 리포트를 본다 — 오류 0개면 [등록]  (여기까지는 계속 draft다)
4. 미리보기로 한 바퀴 돌려 본 뒤 [공개(publish)]
5. 공개하면 문항은 잠긴다
```

---

# 최상위 구조

```json
{
  "$schema": "typelog/quiz/v1",
  "mode": "upsert",
  "quiz": { ... },
  "outcomes": [ ... ],
  "items": [ ... ],
  "resolver": { ... },
  "resultTypes": [ ... ]
}
```

`mode: "upsert"`면 다섯 덩어리가 **항상 함께** 온다. 일부만 보내는 것은
`mode: "content"` 하나뿐이다.

## `quiz` — 설문지 메타

| 필드 | 필수 | 설명 |
|---|---|---|
| `slug` | ✅ | URL·식별자. 소문자 영문 + 하이픈. 예 `animal-friend` `kids-16type` |
| `title` | ✅ | 화면 제목. 아이가 읽는 문장. 20자 이내 |
| `tagline` | ✅ | 한 줄 소개. 40자 이내 |
| `category` | ✅ | `personality` · `animal` · `name` · `character` · `etc` |
| `ageRange` | ✅ | `{ "min": 6, "max": 12 }` |
| `cover` | ✅ | `{ "emoji": "🦊", "imageUrl": null }` |
| `schedule` |  | 공개 예약 — 아래 참조 |
| `estimatedMinutes` |  | 비우면 문항 수로 자동 계산 (문항당 8초) |
| `shuffle` |  | 문항을 섞을지. 기본 `false` |
| `order` |  | 목록 정렬. 작은 값이 앞 |

> `status`는 JSON에 넣지 않는다. 등록은 언제나 `draft`이고, 공개는 admin 화면의
> 버튼으로 한다. JSON으로 공개까지 하면 검증만 통과한 질문지가 바로 노출된다.

### `schedule` — 공개 예약과 오프닝 문구

```json
"schedule": {
  "startAt": "2026-09-10T09:00:00+09:00",
  "endAt": null,
  "openingNoticeText": "9월 10일에 만나요. 어떤 동물이 나올지 기대해 주세요 🦊"
}
```

| 필드 | 동작 |
|---|---|
| `startAt` | 이 시각 전에 **풀려고 접근하면 막는다.** 없으면 공개 즉시 풀 수 있다 |
| `endAt` | 지나면 새 응시를 막는다. 지난 기록은 계속 볼 수 있다. 없으면 무기한 |
| `openingNoticeText` | `startAt` 전에 접근한 사람에게 띄우는 문구. **없으면 기본 문구** |

`schedule`은 **"지금 이 사람이 이 문제를 풀 수 있나"만 판정한다.** 목록을 어떻게 그릴지는
별개다.

기본 문구 — `곧 만나요. 조금만 기다려 주세요 ✨`

> ⚠️ **`startAt`·`endAt`은 오프셋을 반드시 붙인다.** `"2026-09-10T09:00"`처럼 오프셋
> 없이 쓰면 서버(UTC)와 KST가 9시간 어긋나 "공개했는데 안 열린다"가 된다.
> 검증기가 오프셋 없는 문자열을 **오류로 막는다.** 한국 시간은 `+09:00`.

`openingNoticeText`는 40자 이내로, **기대하게 만드는 문장**으로 쓴다.
"준비 중입니다" 같은 사무적인 문장을 쓰지 않는다.

## `outcomes` — 결과 변수 선언

**점수가 모이는 통**이다. 문항의 가중치는 여기 선언한 `id`만 가리킬 수 있다.

```json
[
  { "id": "EI", "kind": "axis", "label": "에너지",
    "left":  { "code": "E", "label": "바깥으로" },
    "right": { "code": "I", "label": "안으로" } },

  { "id": "lion", "kind": "tally", "label": "사자" },
  { "id": "total", "kind": "sum",  "label": "총점" },
  { "id": "warmth", "kind": "dim", "label": "따뜻함" }
]
```

| `kind` | 언제 | 규칙 |
|---|---|---|
| `axis` | 양극 축 (A냐 B냐) | `left`·`right` 필수. **음수는 left, 양수는 right** |
| `tally` | 여러 후보 중 최다 득표 | 가중치는 **0 이상**만 |
| `sum` | 총점 하나 | 한 설문지에 하나만 |
| `dim` | 벡터 차원 (최근접 매칭) | `resultTypes[].vector`와 짝 |

한 설문지에 **`kind`를 섞어도 된다.** 축 4개로 코드를 만들면서 `sum`으로 "활발함
점수"를 함께 보여줄 수 있다.

## `items` — 문항

```json
{
  "linkId": "animal_playground",
  "type": "choice",
  "text": "친구들이 많은 놀이터에 갔을 때 나는?",
  "emoji": "🛝",
  "imageUrl": null,
  "required": true,
  "repeats": false,
  "options": [
    { "oid": "a", "text": "먼저 가서 같이 놀자고 해요", "emoji": "🙋",
      "weights": { "lion": 2, "dolphin": 1 } },
    { "oid": "b", "text": "조금 지켜보다가 다가가요", "emoji": "👀",
      "weights": { "rabbit": 2 } }
  ]
}
```

### `linkId`

응답이 이 값으로 저장된다. **공개(publish) 시점부터 영구 식별자**이고, 그 뒤로는
바꿀 수 없다. `draft` 동안에는 얼마든지 바꿔도 된다 — 응답이 없기 때문이다.

- 한 JSON 안에서 유일해야 한다
- 규칙: `<slug의 첫 단어>_<내용 요약>`, 소문자 + 밑줄. 예 `animal_playground` `type16_party`
- 순번(`q1` `q2`)을 쓰지 않는다. 문항 순서를 바꾸면 번호의 의미가 깨진다

### 문항 타입

| `type` | 답 | 점수 기여 |
|---|---|---|
| `choice` | 선택지 중 하나 (`repeats: true`면 여러 개) | 고른 선택지의 `weights` |
| `boolean` | 예 / 아니오 | `weightsTrue` · `weightsFalse` |
| `integer` | 척도·슬라이더 | `scale` + `coefficient` |
| `ranking` | 순서 정하기 | `rankCoefficients` |
| `string` | 자유 입력 | **점수 0.** 결과 문장 치환용 |
| `date` | 날짜 | 점수 0 |
| `display` | 답이 없는 안내 화면 | 없음 |
| `group` | 문항 묶음 (`items` 중첩) | 없음 |

`integer` 예 —

```json
{
  "linkId": "type16_party_energy",
  "type": "integer",
  "text": "사람이 많은 곳에 있으면 힘이 나요",
  "scale": { "min": 1, "max": 5,
             "labels": ["아니에요", "조금 아니에요", "반반이에요", "조금 그래요", "정말 그래요"] },
  "coefficient": { "EI": -1 },
  "reverse": false,
  "required": true
}
```

기여 = (`value` − 중앙값) × `coefficient`. 위 예에서 5를 고르면 `EI`에 −2 (= left `E`).

### 선택 필드

| 필드 | 뜻 |
|---|---|
| `required` | 기본 `true`. `false`면 건너뛸 수 있고 **min/max 계산에서도 빠진다** |
| `reverse` | 역채점. `true`면 척도를 뒤집어 계산 |
| `enableWhen` | 조건부 노출. `{ "linkId": "animal_pet", "eq": "yes" }` |
| `note` | 관리자용 메모. 화면에 안 나온다 |

## `resolver` — 결과를 정하는 방법

```json
{ "strategy": "argmax", "tiebreak": ["dolphin", "lion", "rabbit", "owl"] }
```

| `strategy` | 추가 필드 | 설명 |
|---|---|---|
| `argmax` | `tiebreak` ✅ | 가장 높은 `tally`의 `id`가 결과 코드 |
| `letters` | `order` ✅ · `threshold` | 축마다 `pomp`가 임계값(기본 50) 미만이면 `left.code`, 이상이면 `right.code`. `order` 순서로 이어 붙인 문자열이 코드. **`tiebreak` 는 쓰지 않는다** — 축마다 독립으로 갈라 결과가 하나로 정해진다 |
| `nearest` | `metric` (`cosine`·`euclidean`) | 응답 벡터와 `resultTypes[].vector`의 최근접 |
| `band` | `on` ✅ · `bands` ✅ | `{ "on": "total", "bands": [ {"lt":34,"code":"low"}, {"lt":67,"code":"mid"}, {"code":"high"} ] }` |
| `matrix` | `rows` ✅ · `cols` ✅ · `cells` ✅ | 두 축을 구간으로 나눈 조합표 |
| `rules` | `rules` ✅ | 조건 목록. 위에서부터 처음 맞는 코드 |

`rules`는 **수식 문자열이 아니라 구조**로 쓴다.

```json
{ "strategy": "rules", "rules": [
  { "when": { "all": [ { "outcome": "lion", "gte": 70 },
                       { "answer": "animal_pet", "eq": "yes" } ] },
    "code": "lion_keeper" },
  { "when": { "any": [ { "outcome": "rabbit", "gte": 80 } ] }, "code": "rabbit" },
  { "when": true, "code": "dolphin" }
]}
```

쓸 수 있는 것만 쓴다 — `all` · `any` · `not` / `outcome` · `answer` / `eq` · `ne` ·
`gte` · `lte` · `in`. **마지막 규칙은 반드시 `"when": true`** 로 기본값을 둔다.

`outcome` 값은 언제나 **POMP 0~100**으로 비교한다. 원점수가 아니다.

> **`letters` 에서는 축마다 문항을 홀수 개로 둔다.** 짝수면 원점수가 정확히 0 이 되는
> 응답이 생기고, 그때 POMP 가 딱 50 이라 임계값 쪽(오른쪽 letter)으로 기계적으로
> 떨어진다. 홀수면 그 경우가 아예 없다 — 예: 축마다 3문항에 ±2 를 주면 원점수가
> −6 · −2 · +2 · +6 만 나온다.

## `resultTypes` — 준비된 답안

```json
{
  "code": "dolphin",
  "name": "다정한 돌고래",
  "subtitle": "친구 마음을 먼저 알아채는 아이",
  "emoji": "🐬",
  "imageUrl": "https://upload.wikimedia.org/wikipedia/commons/.../dolphin.jpg",
  "imageCredit": "Wikimedia Commons · CC BY-SA 4.0 · 촬영자명",
  "color": "#7c3aed",
  "vector": { "warmth": 3, "energy": 1 },
  "rarity": "common",
  "content": {
    "summary": "친구가 무슨 생각을 하는지 먼저 알아채는 편이에요. ...",
    "strengths": ["친구를 잘 도와줘요", "함께 하는 놀이를 좋아해요"],
    "cautions": ["혼자 있는 시간도 필요해요"],
    "tips": ["오늘 친구에게 고마웠던 일을 하나 말해 볼까요?"],
    "goodWith": ["lion", "owl"],
    "funFact": "돌고래는 서로를 이름으로 불러요!"
  }
}
```

| 필드 | 필수 | 규칙 |
|---|---|---|
| `code` | ✅ | `resolver`가 만드는 코드와 **정확히** 같아야 한다 |
| `name` | ✅ | 15자 이내. 아이가 좋아할 별명 |
| `emoji` | ✅ | 이미지가 없어도 화면이 완성되어야 한다 |
| `content.summary` | ✅ | 2~3문장, 120자 이내. **해요체** |
| `content.strengths` | ✅ | 2~4개 |
| `content.cautions` |  | 0~2개. **단점이 아니라 "이럴 땐 조금 힘들어요"** |
| `vector` | `nearest`일 때 ✅ | `dim` outcome과 같은 키 |

### 문장 규칙 — **해요체**, 결쩜사 톤

아이 대상이라고 반말로 쓰지 않는다. [[결쩜사 카피 톤앤매너]]를 그대로 따른다.

- **전부 해요체.** `~합니다`(격식)도 `~한다`(단정)도, 반말도 쓰지 않는다
- 짧게 끊고 줄바꿈으로 호흡을 만든다. 한 문장에 다 넣지 않는다
- **숫자로 증명하지 않는다.** 점수·확률·등급·"상위 몇 %"를 쓰지 않는다
- 단정하지 않는다. "당신은 ~입니다"가 아니라 **"~한 편이에요"**
- **틀린 타입은 없다.** 어떤 결과도 좋게 읽혀야 한다
- "검사·진단·성격유형" 대신 "놀이·성향"
- `cautions`는 단점이 아니다. "이럴 땐 조금 힘들 수 있어요" 쪽으로
- 답을 문장에 넣을 수 있다 — `{{answer.<linkId>}}`.
  예: `"{{answer.animal_name}}님과 가장 가까운 건 돌고래예요."`

```
좋아하는 건 이런 순간이에요.
사람이 많은 자리보다,
마음 맞는 한 사람과 보내는 시간요.
```

---

# 이모지와 이미지 — 반드시 조사해서 채운다

관리자가 나중에 채우지 않는다. **JSON을 만드는 시점에 실제로 쓸 수 있는 값을 넣는다.**

## 우선순위

1. **이모지를 먼저 쓴다.** 네트워크·라이선스·깨짐이 없다. 대부분 이모지로 충분하다
2. 이모지로 표현이 안 될 때만 `imageUrl`
3. 둘 다 어려우면 `imageUrl: null` + 이모지만. **이미지 없이도 화면이 완성되어야 한다**

## 이모지 규칙

- 유니코드 문자를 **그대로** 넣는다. `:dolphin:` 이나 `U+1F42C` 같은 표기를 쓰지 않는다
- 문항·선택지·결과 타입마다 **서로 다른** 이모지를 쓴다. 같은 이모지가 반복되면 구분이 안 된다
- 국기·종교·정치·신체 특징 이모지는 쓰지 않는다
- 피부색·성별 변형(ZWJ 조합)은 피하고 중립 이모지를 쓴다
- 이모지 2개를 붙이지 않는다 (`🦊✨` ✕) — 크기가 흐트러진다

## 이미지 URL 규칙

**실제로 존재하는 URL을 검색해서 확인하고 넣는다.** 만들어 낸 주소를 쓰지 않는다.

| 조건 | 왜 |
|---|---|
| `https://` 로 시작 | 혼합 콘텐츠 차단 |
| **직접 이미지 파일** (`Content-Type: image/*`) | 페이지 주소를 넣으면 안 나온다 |
| 핫링크를 허용하는 출처 | 막아 두면 조용히 빈칸이 된다 |
| 라이선스가 명확 | `imageCredit`에 **출처 · 라이선스 · 저작자**를 적는다 |
| 가로 800px 이상, 5MB 이하 | 카드에 쓰기 적당한 크기 |

권장 출처 —

| 출처 | 형태 | 라이선스 |
|---|---|---|
| **OpenMoji** | `https://openmoji.org/data/color/svg/1F42C.svg` | CC BY-SA 4.0 |
| **Twemoji** (jsDelivr) | `https://cdn.jsdelivr.net/gh/jdecked/twemoji@latest/assets/svg/1f42c.svg` | CC BY 4.0 |
| **Wikimedia Commons** | `https://upload.wikimedia.org/wikipedia/commons/...` | 파일마다 다름 — 반드시 확인 |
| **Unsplash** | `https://images.unsplash.com/photo-...` | Unsplash License |

> 그림 스타일을 한 설문지 안에서 섞지 않는다. 사진과 이모지 일러스트를 번갈아 쓰면
> 결과 카드가 조각조각으로 보인다. **한 설문지는 한 스타일**로 간다.

---

# 검증기가 무엇을 보나

관리자가 세부를 확인하지 않으므로 **기계가 대신 본다.** AI는 아래를 통과하도록 만든다.
FitLog가 인바디 합계 규칙으로 Vision의 오인식을 잡는 것과 같은 장치다.

## 오류 (막는다)

1. JSON 파싱 실패 · 필수 필드 누락 · 타입 불일치
2. `mode: "upsert"`인데 이미 **`published`인 `slug`** 다 (`content`는 허용)
   — `content`에서 `code`나 `vector`를 바꾸려 한 것도 오류다
3. `linkId`가 이 JSON 안에서 중복이다
4. `weights`·`coefficient`의 키가 `outcomes`에 없다
5. `tally` outcome에 **음수** 가중치가 있다
6. `axis` outcome에 `left`·`right`가 없다
7. `letters` — `order`의 축 개수와 `resultTypes[].code` 길이가 다르다,
   또는 **가능한 조합 중 없는 코드가 있다** (축 4개 = 16개 모두 필요)
8. `argmax` — `tally` outcome인데 대응하는 `resultTypes.code`가 없다
9. `nearest` — `vector`가 없는 결과 타입이 있거나 차원이 안 맞는다
10. `band` — 구간에 **빈틈이나 겹침**이 있다
11. `tiebreak`에 빠진 코드가 있다
12. `rules`의 마지막이 `"when": true`가 아니다
13. `enableWhen`이 **자기 뒤에 오는 문항**을 가리킨다 (순환·전방 참조)
14. `schedule.startAt`·`endAt`에 **시간대 오프셋이 없다**, 또는 `endAt`이 `startAt`보다 이르다
15. `imageUrl`이 200을 주지 않거나 `Content-Type`이 이미지가 아니다
16. `content.summary` · `strengths` · `emoji` 누락
17. `quiz.ageRange` 의 `min`·`max` 누락, `min > max`, 3~19세 밖
18. `quiz.disclaimer` 가 빈 문자열이다 (없으면 아예 넣지 않는다)

## 경고 (등록은 되지만 리포트에 뜬다)

- **도달 불가능한 결과 타입** — 어떤 답 조합으로도 나올 수 없다
- **한쪽으로만 가는 축** — 모든 문항이 한 방향 점수만 준다. 결과가 사실상 고정된다
- **결과 분포 치우침** — 아래 몬테카를로 참조
- 문항 수 6개 미만 (결과가 우연에 좌우된다) 또는 40개 초과 (아이가 못 끝낸다)
- 문항 텍스트 60자 초과 · `summary` 120자 초과 · `openingNoticeText` 40자 초과
- 같은 이모지 중복 사용
- `imageUrl`은 있는데 `imageCredit`이 없다
- `string`·`date` 문항이 결과 문장에서 한 번도 쓰이지 않는다
- `startAt`이 **과거**다 (예약하려던 것 아닌가?)
- `quiz.ageRange` 범위가 10살을 넘는다 — `6~19세` 는 아무 말도 하지 않는 것과 같다
- 화면에 보이는 문장에 **마크다운으로 보이는 기호**가 있다 (`**굵게**` · `` `코드` `` ·
  `[링크](...)`). 화면은 평문으로 그리므로 별표가 그대로 보인다. JSON 을 쓰는 쪽은
  마크다운 습관이 있어서 자꾸 들어가는데, 오류가 아니라 그냥 이상한 글자로 남아
  아무도 알려주지 않는다 (2026-09-04 `superpower` 는 공개된 뒤에야 발견됐다)

## 몬테카를로 분포 점검

**관리자가 문항을 안 읽어도 이상을 잡는 장치다.**
무작위 응답 2,000회를 돌려 결과 코드의 분포를 히스토그램으로 보여준다.

| 신호 | 뜻 |
|---|---|
| 한 코드가 40% 넘게 나온다 | 가중치가 그쪽으로 치우쳤다 |
| 2% 미만인 코드가 있다 | 사실상 안 나오는 타입이다 |
| 한 번도 안 나온 코드 | 위 "도달 불가능" 오류와 같다 |
| 1등·2등 격차 평균이 아주 작다 | 동점이 잦다. `tiebreak`가 결과를 지배한다 |
| **고르면 나올 값의 2.5배가 넘는다** | 타입 개수에 견준 치우침 |
| **고르면 나올 값의 0.4배 미만이다** | 같음 |
| **동점률(`tieRate`)이 5% 넘는다** | 배열 순서가 결과를 정한다 |

무작위 응답은 균등 분포라 실제 사람의 분포와 다르다. **절대적인 기준이 아니라
"극단적으로 치우쳤는지"만 본다.**

> ⚠️ **절대값(40% / 2%)만 보면 못 잡는다.** 8타입 질문지에서 22.6% 는 절대값 게이트를
> 조용히 통과하지만, 고른 분포라면 12.5% 여야 하니 1.8배다. 타입이 16개면 같은
> 22.6% 가 3.6배로 훨씬 심각하다. 그래서 **`100 / 타입수` 에 견주어** 본다.
> (2026-09-04 `english-name` 이 `resultTypes` 배열 순서대로 결정되고 있던 것을
>  이 게이트가 없어서 놓쳤다)

> ⚠️ **`meanConfidence` 로는 동점을 못 잡는다.** `nearest` 의 점수는 거리라서 눈금이
> 크고, 축 하나가 정확히 중간값이어서 판별력이 0인데도 평균 격차는 커 보인다.
> 그래서 **1·2등이 정확히 동점이던 비율**(`tieRate`)을 따로 센다.

**공개 전 마지막 관문이다.** 문항이 잠기고 나면 고칠 수 없으므로, 이 히스토그램이
이상하면 `draft` 상태에서 JSON을 다시 만들어 붙여넣는다.

---

# 추천 연령과 안내 문구

`quiz.ageRange` 는 **필수다.** 목록이 카드마다 `6~12세 추천` 을 보여주고, 나이대
묶음으로 거르는 데도 쓴다. 없으면 화면에 아무것도 안 뜬다.

```json
"ageRange": { "min": 6, "max": 12 }
```

- 3세부터 19세 사이로 적는다
- 범위는 **10살 이내**로 좁힌다. `6~19세` 는 알려주는 값이 없다
- 실제로 읽을 수 있고 답할 수 있는 나이로 잡는다 — 문항에 "모둠 활동" 이 나오면
  학교에 다니는 나이여야 한다

실제로 있는 **브랜드·나라 이름**을 결과로 쓰면 `quiz.disclaimer` 를 붙인다.
목록 카드와 결과 화면 아래에 작게 나온다.

```json
"disclaimer": "실제로 있는 브랜드 이름을 재미로 빌려 왔어요. 각 브랜드와는 아무 관련이 없고, 사라고 권하는 것도 아니에요."
```

90자 이내로 쓴다. 없으면 필드를 아예 넣지 않는다(빈 문자열은 오류다).
지켜야 할 나머지 규칙은 [[TypeLog]] 의 "실제 브랜드 이름을 결과로 쓸 때" 참고.

---

# `nearest` 로 만들 때

결과 타입마다 벡터를 두고 응답 벡터와 가장 가까운 것을 고른다. 축을 ±offset 부호
조합으로 만드는 것이 가장 쉬운데, **거기에 함정이 두 개 있다.**

### 1. 부호 조합을 일부만 고르면 안 된다

축이 4개면 조합은 16가지다. 그중 10개만 골랐더니 분포가 leo 21.1% · mia 2.0% 로
벌어졌다. **부호 하나만 다른 이웃이 빠진 타입이 그 공간까지 가져가** 셀이 커지기
때문이다.

→ **짝수 패리티 8개만 쓴다** (음수 부호가 0·2·4개인 조합). 서로 1-플립 이웃이
하나도 없어서 셀 크기가 같다. 4차원 정육면체의 절반(demicube)이다.

### 2. 네 축의 offset 을 같은 값으로 두면 안 된다

부호가 축 `i`·`j` 만 다른 두 타입의 거리 차이는 `4·o·(POMP_i − POMP_j)` 다.
offset 이 같으면 **두 축의 POMP 가 같기만 해도 정확히 동점**이고, POMP 가 가질 수
있는 값은 열 가지뿐이라 이 일이 아주 잦다 — 무작위 응답의 28.9% 가 동점이었고
결과는 `resultTypes` 배열 순서대로 줄을 섰다.

→ **offset 을 서로 다르게 둔다.** 25 · 23 · 21 · 19 처럼 잡으면
`o_i·q_i = o_j·q_j` 를 만족하는 정수해가 없어 상쇄가 일어나지 않는다.

### 3. POMP 가 정확히 50 에 닿지 않게 한다

offset 이 ±o 면 `|50−(50−o)| = |50−(50+o)|` 이므로 POMP 가 50인 축은 판별력이 0이다.
축마다 3문항 · 4지선다에 가중치 `3·2·1·0` 을 주면 raw 는 0..9 라서 POMP 가 `100/9`
의 배수뿐이고 50 이 없다. (3지선다 `2·1·0` 은 raw 3 = POMP 50 을 26% 확률로 만든다)

이 셋을 지키면 8타입 분포가 11.5~14.5%(이상치 12.5%), 동점률 0% 가 된다.

---

# 전체 예시 — `argmax` (동물, 축약)

```json
{
  "$schema": "typelog/quiz/v1",
  "mode": "upsert",
  "quiz": {
    "slug": "animal-friend",
    "title": "나와 가까운 동물은?",
    "tagline": "여덟 가지 질문으로 찾는 나의 동물 친구",
    "category": "animal",
    "ageRange": { "min": 6, "max": 12 },
    "cover": { "emoji": "🦊", "imageUrl": null },
    "schedule": {
      "startAt": "2026-09-10T09:00:00+09:00",
      "endAt": null,
      "openingNoticeText": "9월 10일에 만나요. 어떤 동물이 나올지 기대해 주세요 🦊"
    },
    "order": 20
  },
  "outcomes": [
    { "id": "lion",    "kind": "tally", "label": "사자" },
    { "id": "rabbit",  "kind": "tally", "label": "토끼" },
    { "id": "dolphin", "kind": "tally", "label": "돌고래" },
    { "id": "owl",     "kind": "tally", "label": "부엉이" }
  ],
  "items": [
    { "linkId": "animal_playground", "type": "choice", "emoji": "🛝", "required": true,
      "text": "친구들이 많은 놀이터에 갔을 때 나는?",
      "options": [
        { "oid": "a", "text": "먼저 가서 같이 놀자고 해요", "emoji": "🙋",
          "weights": { "lion": 2, "dolphin": 1 } },
        { "oid": "b", "text": "조금 지켜보다가 다가가요", "emoji": "👀",
          "weights": { "rabbit": 2 } },
        { "oid": "c", "text": "무슨 놀이인지 먼저 살펴봐요", "emoji": "🔍",
          "weights": { "owl": 2 } }
      ] }
  ],
  "resolver": { "strategy": "argmax", "tiebreak": ["dolphin", "owl", "rabbit", "lion"] },
  "resultTypes": [
    { "code": "dolphin", "name": "다정한 돌고래", "emoji": "🐬", "color": "#7c3aed",
      "content": {
        "summary": "친구 마음을 먼저 알아채는 편이에요. 같이 웃는 시간을 가장 좋아해요.",
        "strengths": ["친구를 잘 도와줘요", "함께 하는 놀이를 좋아해요"],
        "cautions": ["혼자 쉬는 시간도 필요해요"],
        "tips": ["오늘 고마웠던 일을 하나 말해 볼까요?"],
        "goodWith": ["lion", "owl"],
        "funFact": "돌고래는 서로를 이름으로 불러요!"
      } }
  ]
}
```

---

# AI에게 줄 프롬프트

이 노트를 함께 주고 아래를 덧붙인다.

```
[[설문지 JSON 작성 지침]]의 규격대로 질문지 JSON을 만들어 주세요.

주제:        나와 가까운 동물은?
결과 타입:   6종
문항 수:     10개
대상:        초등 저학년
스코어링:    argmax
공개 예정:   2026-09-10 오전 9시 (KST)

지킬 것
- 다섯 덩어리(quiz · outcomes · items · resolver · resultTypes)를 전부 담는다
- linkId는 규칙대로, 순번을 쓰지 않는다
- 결과 타입마다 도달 가능한 답 조합이 실제로 있어야 한다
  (한 결과에만 점수를 몰아주는 문항을 만들지 않는다)
- 결과 6종의 출현 확률이 비슷해야 한다. 가중치를 배분해 확인해 볼 것
- tiebreak에 6종 전부 넣는다
- schedule.startAt 은 +09:00 오프셋을 반드시 붙인다
- openingNoticeText 는 기대하게 만드는 문장으로 40자 이내
- 이모지는 전부 서로 다르게, 유니코드 문자로
- 이미지가 필요하면 실제로 접근 가능한 URL을 검색해 확인하고
  imageCredit에 출처·라이선스·저작자를 적는다. 확실하지 않으면 null로 두고 이모지만
- 문장은 전부 해요체. 반말·격식체를 쓰지 않는다 (결쩜사 톤)
- 점수·확률·등급 같은 계량 표현을 쓰지 않고, 단정하지 않는다 ("~한 편이에요")
- 어떤 결과도 좋게 읽히도록
- JSON만 출력한다 (설명·주석 없이)
```

---

# 자주 나는 실패

| 증상 | 원인 | 해결 |
|---|---|---|
| 등록이 "이미 공개된 질문지" 로 막힌다 | `published`에 `upsert`를 보냈다 | 문장만 고치는 것이면 `mode: "content"`. 문항을 고치는 것이면 응답 0건일 때 `unpublish` 후 다시, 응답이 있으면 **새 `slug`** |
| 공개했는데 안 열린다 | `startAt`에 오프셋이 없어 9시간 밀렸다 | `+09:00`을 붙인다 |
| 오픈 예정 카드에 엉뚱한 문구가 뜬다 | `openingNoticeText`를 안 넣었다 | 넣거나, 기본 문구로 둔다 |
| "없는 코드" 오류 (`letters`) | 축 4개면 16종이 다 필요한데 몇 개만 만들었다 | 빠진 조합의 `resultTypes`를 채운다 |
| 결과가 항상 같게 나온다 | 모든 문항이 한 outcome에만 점수를 준다 | 문항마다 여러 후보에 배분한다 |
| 분포 점검에서 한 타입이 60% | 그 타입에 붙은 가중치가 크다 | 가중치를 낮추거나 다른 타입에 얻을 기회를 만든다 |
| 이미지가 빈칸으로 나온다 | 페이지 URL을 넣었거나 핫링크가 막혔다 | 직접 이미지 URL로 바꾼다. 안 되면 `null` + 이모지 |
| `rules`가 의도대로 안 걸린다 | 임계값을 원점수 감각으로 썼다 | `outcome` 비교는 **0~100(POMP)** 이다. 문항이 8개여도 60은 60이다 |
| 결과 문장이 앱과 톤이 다르다 | 반말·격식체를 썼다 | 전부 해요체 → [[결쩜사 카피 톤앤매너]] |

## 관련

- [[TypeLog]] — 데이터 모델과 결정 사항
- [[OpenAI Vision 추출 패턴]] — 사람이 검토하지 않는 입력을 기계가 검사하는 같은 발상
- [[결쩜사 카피 톤앤매너]] — 결과 문장의 해요체와 문장 리듬
