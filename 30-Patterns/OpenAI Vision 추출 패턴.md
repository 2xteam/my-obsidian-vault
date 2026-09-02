---
title: OpenAI Vision 추출 패턴
type: pattern
tags: [pattern, openai, vision]
updated: 2026-09-02
---

# OpenAI Vision 추출 패턴

사진에서 구조화된 데이터를 뽑는 작업의 공통 골격.
[[SnapWord]](교재 → 단어), [[SnapNote]](오답 이미지), [[FitLog]](인바디 결과지 → 수치)에 쓴다.

## 호출

```ts
const res = await client.chat.completions.create({
  model: process.env.OPENAI_VISION_MODEL ?? "gpt-4o",
  response_format: { type: "json_object" },   // JSON 강제
  temperature: 0,                              // 수치 추출은 창의성이 해롭다
  messages: [
    { role: "system", content: SYSTEM_PROMPT },
    { role: "user", content: [
      { type: "text", text: `${지시}\n\n${JSON_SHAPE}` },
      { type: "image_url", image_url: { url: dataUrl, detail: "high" } },
    ]},
  ],
});
```

- 채팅용 모델(`OPENAI_MODEL`)과 Vision 모델(`OPENAI_VISION_MODEL`)을 **환경 변수로 분리**한다.
  채팅은 저렴한 모델, 추출은 정확한 모델
- 이미지는 업로드 전에 축소한다 → [[이미지 업로드 패턴]]

## 프롬프트에 반드시 넣을 것

1. **인쇄된 값만 옮기고 계산·추정하지 말 것**
2. **없는 값은 null** — 억지로 채우지 말 것
3. **흐릿하면 추측하지 말고 null**
4. 단위를 제거한 숫자만
5. **스키마에 자리가 없는 항목은 버리지 말고 `etc` 배열에 담을 것**
6. 출력 JSON 형태를 예시로 통째로 보여줄 것

5번이 중요하다. 스키마를 처음부터 완벽하게 만들 수 없으므로,
버려지는 데이터를 `etc`로 받아두고 **반복해서 쌓이는 항목을 정식 필드로 승격**한다.

## 검증 — 도메인 규칙으로 오인식을 잡는다

Vision은 `8↔3`, `0.90↔0.98` 같은 오인식을 반드시 낸다.
**문서 자체의 내부 정합성**을 검사에 쓰면 상당수를 잡을 수 있다.

FitLog(인바디)의 경우:

```
체수분 + 단백질 + 무기질 + 체지방 = 체중   (±0.6kg)
BMI = 체중 ÷ (키m)²                        (±0.4)
체지방률 = 체지방량 ÷ 체중 × 100            (±0.8%p)
제지방량 = 체중 − 체지방량                  (±0.6kg)
골격근량 ≤ 제지방량
```

여기에 상식 범위(체지방률 1~60%, 체중 20~250kg)를 더한다.

## 검토 화면을 반드시 거친다

추출 즉시 저장하지 않는다.

```
사진 → 추출 → [검토 화면: 원본 이미지 + 추출값 폼 + 경고 표시] → 저장
```

- 검증에 걸린 필드는 빨갛게 표시하고 이유를 문장으로 보여준다
- 사용자가 고칠 수 있어야 한다
- 수정 여부를 `extraction.editedByUser`로 남겨두면 나중에 모델 품질을 평가할 수 있다

## 비용 관리

- 토큰 차감(`lib/useToken.ts`) — Vision 1회 10토큰
- 요청 로그를 `openai_request_logs`에 남긴다
- 같은 이미지 재분석을 막는 캐시(`ai_cache`)
