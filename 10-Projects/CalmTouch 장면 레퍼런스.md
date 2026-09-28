---
title: CalmTouch 장면 레퍼런스
type: reference
tags: [calmtouch, reference, webgl, opensource]
updated: 2026-09-10
---

# CalmTouch 장면 레퍼런스

[[CalmTouch]] 의 다음 장면들을 **물감 번짐과 같은 품질**로 만들 수 있는지, 쓸 만한 오픈소스가
있는지 2026-09-10 에 조사한 결과. 라이선스 · 스타 · 마지막 push 는 GitHub API 로 그날 읽었다.

## 결론

**전부 가능하다.** 세 갈래로 나뉜다.

| 갈래 | 장면 | 왜 |
|---|---|---|
| **이미 있는 유체 엔진 재사용** | 구름 · 먹 마블링 · 물결 | 셰이더 한두 개(부력 · 높이장)만 더하면 된다 |
| **검증된 MIT 라이브러리 위에** | 공(matter.js) · 반짝이(tsParticles 또는 직접) · 슬라임(Verly.js + 메타볼 셰이더) · 모래(gpu-io 셀룰러 오토마타) | 물리 · 파티클은 라이브러리가 낫다. 렌더만 우리가 맡는다 |
| **참고만 하고 직접 쓴다** | 은하(GPGPU 파티클) · 반응확산 · 점균 | 코드가 작고(셰이더 100~200줄) 우리 엔진 구조에 바로 들어간다 |

품질은 라이브러리가 아니라 **네 가지 습관**에서 나온다 — 아래 "고퀄리티 체크리스트".

## 기술 바탕 (엔진 층)

| 라이브러리 | 역할 | 라이선스 | ★ | 최근 push | 메모 |
|---|---|---|---|---|---|
| [gpu-io](https://github.com/amandaghassaei/gpu-io) | WebGL GPGPU 프레임워크 — 유체 · CA · 파티클 | MIT | 1.5k | 2024-01 | WebGL1 폴백. 우리 FluidSim 을 이걸로 갈아탈 필요는 없지만 **모래 · 반응확산 · 점균**에 쓰기 좋다 |
| [three.js](https://github.com/mrdoob/three.js) | 3D · GPGPU(GPUComputationRenderer) | MIT | — | 활발 | 은하 · 새떼(boids) 예제가 공식에 있다 |
| [matter.js](https://github.com/liabru/matter-js) | 2D 강체 물리 | MIT | 18k | 2024-08 | 공 장면. 터치 · 모바일 지원이 문서화돼 있다 |
| [rapier.js](https://github.com/dimforge/rapier.js) | 2D/3D 물리 (Rust→WASM) | Apache-2.0 | 700 | 2026-07 | 공이 수백 개면 이쪽. WASM 로딩 비용 있음 |
| [Verly.js](https://github.com/anuraghazra/Verly.js) | Verlet 물리 — 천 · 줄 · 소프트바디 | MIT | 700 | 2025-02 | 슬라임 · 천 커튼 |
| [tsParticles](https://github.com/tsparticles/tsparticles) | 파티클 · `repulse` 인터랙션 · firefly 프리셋 | MIT | 9k | 2026-09 | 반짝이를 가장 빨리 만든다. 단 "예쁨"은 직접 그려야 나온다 |
| [@box2d/particles](https://www.npmjs.com/package/@box2d/particles) | LiquidFun(Box2D 입자 유체 · 탄성체) TS 포트 | (확인 필요) | 66 | 2025-02 | 슬라임의 정석적 해법. 라이선스 파일이 API 에 안 잡혔다 — 쓰기 전에 확인 |
| [Tone.js](https://tonejs.github.io/) | Web Audio | MIT | — | 활발 | 공 튕김 · 윈드차임 소리. **사용자 제스처 뒤에만 시작** |

## 예정 장면 여섯 개

### 1. 은하 — 돌고 있는 별을 흩뜨리면 제자리로

| | |
|---|---|
| 접근 | GPGPU 파티클. 위치 텍스처에 **기준 궤도 위치**를 저장하고, 포인터 반경 안은 밀어내고, 스프링(감쇠)으로 복귀. 1만~10만 점 |
| 레퍼런스 | [brunoimbrizi/interactive-particles](https://github.com/brunoimbrizi/interactive-particles) ★1.2k (라이선스 없음 — 기법만) · [Codrops GPGPU 파티클 튜토리얼 2024](https://tympanus.net/codrops/2024/12/19/crafting-a-dreamy-particle-effect-with-three-js-and-gpgpu/) · [particle-galaxy-3js](https://github.com/anushkachauhxn/particle-galaxy-3js) · Bruno Simon 의 Three.js Journey 은하 생성기 패턴 |
| 품질 포인트 | 가산 블렌딩 + 부드러운 스프라이트, 중심은 밝고 바깥은 차게(두 색 그라디언트), 나선 팔마다 무작위 흩뿌림, 회전 속도는 반경에 반비례(케플러 느낌). 복귀는 **임계 감쇠 스프링**이라야 출렁이지 않는다 |
| 입력 | 포인터 = 밀어내기 · 스크롤 = 줌/회전 속도 · 기울기 = 카메라 살짝 기울임 |
| 난이도 | 중간 (셰이더 2개 · 우리 FluidSim 과 같은 핑퐁 FBO 구조) |
| 폴백 | Canvas 2D 3~5천 점. 저사양 폰에서도 60fps |

### 2. 공 — 튕기고 기울이면 굴러간다

| | |
|---|---|
| 접근 | **matter.js** (MIT). 원 몇 개 + 벽 4장 + 마찰 · 반발. `deviceorientation` → `engine.gravity` |
| 레퍼런스 | [matter.js 데모](https://brm.io/matter-js/demo/) · 벤치마크 [napejs.org/benchmark](https://napejs.org/benchmark.html) (matter vs planck vs rapier) |
| 품질 포인트 | 물리는 라이브러리, **렌더는 직접** — 바닥 그림자(공과 거리에 따라 흐려짐), 충돌 순간 살짝 눌리는 스쿼시, 벽에 닿을 때 짧은 소리(Tone.js 멤브레인 신스). 공 수는 3~12개, 질량 다르게 |
| 입력 | 탭 = 그 자리 공을 튕김 · 드래그 = 던지기 · 기울기 = 중력 · 흔들기(`devicemotion`) = 전부 튀어오름 |
| 난이도 | 낮음 |
| 주의 | iOS 기울기 허용은 이미 PlayScreen 에 있는 `requestPermission` 흐름을 그대로 쓴다 |

### 3. 모래 — 문질러 모으고 흩기

두 가지 다른 것이 있다. **마음 진정에는 (b) 가 맞다.**

| | (a) 떨어지는 모래(셀룰러 오토마타) | (b) 젠 가든 — 모래 표면을 긁는다 |
|---|---|---|
| 접근 | GPU CA. 셀마다 모래 양, 아래 · 대각으로 흐른다 | 높이장(height field). 포인터가 지나간 자리를 파고 양옆에 둔덕. 높이장의 기울기로 음영 |
| 레퍼런스 | [sandspiel](https://github.com/MaxBittker/sandspiel) MIT ★3.2k (Rust+WASM, 통합은 무겁다) · [falling-sand-shader](https://github.com/m4ym4y/falling-sand-shader) (GPU 프래그먼트 셰이더 방식 · 라이선스 없음) · gpu-io 의 CA 예제 | [geoffreylitt/zen-garden](https://github.com/geoffreylitt/zen-garden) (Phaser · 라이선스 없음) · [Silent Sand](https://silentsand.me/) (참고용 UX) · [Cory Trimm 디지털 젠 가든](https://corytrimm.com/posts/making-the-web-fun-again/) |
| 품질 포인트 | 입자 색 3~4단계로 흩뿌려야 "모래"로 보인다 | **갈퀴 자국이 여러 줄**(포인터 폭 안에 3~5개 골), 높이장 노멀로 빛 방향 음영, 손을 떼면 아주 천천히 평평해지는 옵션 |
| 난이도 | 중간~높음 | 낮음~중간 (우리 유체 엔진의 FBO 핑퐁 그대로, 셰이더 2개) |

> ⚠️ **Sand Game JS 는 쓰지 않는다.** 코드는 공개돼 있지만 라이선스가 "배포 · 파생물 금지, 허가 필요"다
> ([LICENSE](https://github.com/Hartrik/sand-game-js/blob/master/LICENSE.md)). 참고도 하지 않는 편이 깔끔하다.

### 4. 슬라임 — 누르면 눌리고 당기면 늘어난다

| | |
|---|---|
| 접근 A (권장) | **스프링-질점 메시** (Verly.js MIT 또는 직접 Verlet 60줄) + **메타볼/SDF 셰이더**로 겉면 렌더. 질점을 그대로 그리면 그물처럼 보인다 — 겉면을 부드럽게 그려야 슬라임 |
| 접근 B | LiquidFun 탄성 입자 그룹 (`@box2d/particles`). 가장 "진짜" 같지만 의존성이 크다 |
| 레퍼런스 | [Verly.js](https://github.com/anuraghazra/Verly.js) MIT · [bloob](https://github.com/onsetsu/bloob) MIT (JS 소프트바디 엔진) · [Jelly 3D 데모](https://45deg.github.io/jelly/) · [Gooey metaballs 셰이더 글](https://vishald.com/blog/gooey-webgl/) · [sjpt/metaballsWebgl](https://github.com/sjpt/metaballsWebgl) · Coding Train #177 Soft Body |
| 품질 포인트 | 반투명 + 안쪽 하이라이트(스페큘러) + 가장자리 두께감, 손 뗄 때 **과감쇠**로 두 번쯤 출렁이고 멈춤, 누르는 동안 살짝 색이 진해짐. 소리는 없는 게 낫다 |
| 입력 | 누르기 = 눌림 · 드래그 = 늘림 · 두 손가락 = 양쪽 당기기 · 기울기 = 슬라임이 그쪽으로 늘어짐 |
| 난이도 | 중간 |

### 5. 구름 — 만지면 흩어지고 다시 뭉친다

| | |
|---|---|
| 접근 | **우리 유체 엔진 재사용.** 염료를 흰 안개로, `velocityDissipation` 낮게, **부력 항**(온도/밀도에 비례해 위로) 하나 추가. 포인터는 힘만 주고 염료는 거의 넣지 않는다. "다시 뭉침"은 낮은 확산 + 화면 밖으로 새는 만큼 가운데서 조금씩 보충 |
| 레퍼런스 | [Rachel Bhadra Smoke Simulation](https://rachelbhadra.github.io/smoke_simulator/index.html) (Navier-Stokes + 부력) · [githole/webglSmoke](https://github.com/githole/webglSmoke) · [nociza/SmokeSculpter](https://github.com/nociza/SmokeSculpter) (3D) · 대안: fbm 노이즈 구름을 포인터로 왜곡하는 프래그먼트 셰이더 하나짜리 |
| 품질 포인트 | 배경을 하늘색 대신 **짙은 남색~새벽색 그라디언트**로, 안개는 밀도에 따라 청백→회청, 얕은 음영(이미 DISPLAY_FRAG 에 있음) |
| 난이도 | **낮음** — 가장 먼저 만들 것 |

### 6. 반짝이 — 손끝을 피해 도망 다니는 빛

| | |
|---|---|
| 접근 | Canvas 2D 파티클 200~600개. 배회(Perlin/curl 노이즈) + 포인터 반발력 + 가까이 오면 밝아짐 + 깜빡임 주기. tsParticles 의 `repulse` + firefly 프리셋으로 30분 안에 시작할 수 있지만, **결과물의 품질은 직접 그린 쪽이 낫다** |
| 레퍼런스 | [tsParticles firefly 프리셋](https://particles.js.org/samples/presets/firefly.html) MIT · three.js 공식 [webgl_gpgpu_birds](https://threejs.org/examples/webgl_gpgpu_birds.html) (마우스가 포식자 — 새떼가 피한다) · [simianarmy/threejs-boids](https://github.com/simianarmy/threejs-boids) |
| 품질 포인트 | 글로우는 `shadowBlur` 대신 **가산 블렌딩 스프라이트**(성능), 색은 노랑-초록 한 계열, 도망칠 때 방향이 아니라 **속도만** 바뀌게(허둥대면 불안하다), 몇 마리는 안 피하고 손끝에 앉는다 |
| 난이도 | 낮음 |

## 새 아이디어 — 마음 진정에 맞는 것

적합도 ●●● 높음 · 난이도는 우리 코드베이스 기준.

| 장면 | 한 줄 | 적합도 | 난이도 | 오픈소스 |
|---|---|---|---|---|
| **물결** | 잔잔한 수면. 탭하면 동심원, 손을 끌면 잔물결. 기울이면 물이 그쪽으로 | ●●● | 낮음 | [sirxemic/jquery.ripples](https://github.com/sirxemic/jquery.ripples) MIT ★1.1k (핑퐁 높이장 + 굴절) · [gorodroz/simple-water-waves-shader](https://github.com/gorodroz/simple-water-waves-shader) · [dghez/mouse-effects-webgl-water](https://github.com/dghez/mouse-effects-webgl-water) |
| **먹 마블링(스미나가시)** | 물 위 먹 한 방울, 바람 불듯 저으면 무늬 | ●●● | 낮음 | 우리 유체 엔진 + 확산 0 · 점성 높음. 참고 [Amanda Ghassaei Digital Marbling](https://blog.amandaghassaei.com/2022/10/25/digital-marbling/) · [marblizer](https://github.com/nickswalker/marblizer) MIT · Ink Garden |
| **비 오는 창** | 유리에 맺힌 빗방울, 손가락으로 닦으면 김이 걷힘 | ●●● | 중간 | [codrops/RainEffect](https://github.com/codrops/RainEffect) ★1.8k (Lucas Bebber · **라이선스 없음 → 기법만**) |
| **반응확산** | 얼룩무늬가 천천히 자란다. 만지면 그 자리에서 새 무늬 | ●●○ | 낮음 | [piellardj/reaction-diffusion-webgl](https://github.com/piellardj/reaction-diffusion-webgl) MIT · [amandaghassaei/ReactionDiffusionShader](https://github.com/amandaghassaei/ReactionDiffusionShader) · [Jason Webb Playground](https://jasonwebb.github.io/reaction-diffusion-playground/) |
| **점균(physarum)** | 수십만 입자가 실 같은 그물을 짠다. 손끝이 먹이 | ●●○ | 중간 | [nicoptere/physarum](https://github.com/nicoptere/physarum) Unlicense ★300 · [fogleman/physarum](https://github.com/fogleman/physarum) MIT (Go · 참고) · Sage Jenson "36 Points" (기법의 원전) |
| **새떼 · 물고기떼** | 손을 피하면서도 무리를 지킨다 | ●●○ | 중간 | three.js `webgl_gpgpu_birds` MIT · [trykhov/FishPond](https://github.com/trykhov/FishPond) · [li-cai/koipond](https://github.com/li-cai/koipond) |
| **천 커튼** | 바람에 흔들리는 얇은 천. 손으로 젖히면 되돌아옴 | ●●● | 낮음 | Verly.js MIT (cloth 15줄) · [araghava/cloth-sim](https://github.com/araghava/cloth-sim) (Canvas · 라이선스 없음) |
| **라바 램프** | 방울이 천천히 오르고 합쳐진다. 만지면 갈라짐 | ●●● | 낮음 | 메타볼 프래그먼트 셰이더 하나. [Painting with Math 글](https://damianvandermerwe.com/blog/painting-with-math-lava-lamp-shader) · [ELevin125/metaballs-lava-lamp](https://github.com/ELevin125/metaballs-lava-lamp) (p5) |
| **김 서린 유리** | 흐린 유리를 손가락으로 닦으면 뒤 풍경(그라디언트)이 보이고 다시 서림 | ●●● | **아주 낮음** | 라이브러리 불필요. 마스크 텍스처 한 장 |
| **윈드차임** | 기울이면 바람, 관이 부딪혀 소리 | ●●○ | 중간 | matter.js 진자 + Tone.js. 참고 [Vibe Chimes](https://vibechimes.com/) (닫힌 소스) |
| **별자리 잇기** | 밤하늘 점을 손끝으로 이으면 선이 남고 서서히 사라진다 | ●●○ | 낮음 | 직접. 별 배경은 [warpspeed](https://github.com/adolfintel/warpspeed) 참고 |
| **호흡 원** | 커지고 작아지는 원에 숨을 맞춘다. 손을 대면 속도가 따라온다 | ●●● | 아주 낮음 | 직접 (CSS/Canvas). 카테고리 "마음 기록"에 가장 가깝다 — 횟수를 기록할 수 있다 |
| **Fluid Paint 계열** | 붓으로 그리는 유화 물감 | ●○○ | 높음 | [dli/paint](https://github.com/dli/paint) MIT ★3k · [dli/fluid](https://github.com/dli/fluid) MIT (3D 입자 유체). 진정보다 "창작" 쪽 |

**추천 순서** — 김 서린 유리 → 구름 → 물결 → 먹 마블링 → 반짝이 → 라바 램프 → 공 → 천 커튼 → 은하 → 슬라임 → 젠 가든 모래 → 반응확산 · 점균 · 새떼.
앞의 넷은 이미 있는 FBO 핑퐁 구조를 그대로 쓴다.

## 고퀄리티 체크리스트 — toukoum 느낌은 여기서 온다

1. **GPU 에서 부드럽게** — 블러 · 글로우 · 음영은 셰이더로. Canvas 2D `shadowBlur` 는 폰에서 프레임을 떨어뜨린다
2. **감쇠와 이징** — 모든 움직임은 `1 + k·dt` 감쇠 또는 임계 감쇠 스프링. 즉시 멈추거나 즉시 돌아오면 장난감처럼 보인다
3. **가만히 둘 때도 살아 있다** — 흐름(드리프트) 같은 저강도 자율 움직임. 이미 PlayScreen 에 있다
4. **색 규율** — 한 장면에 한 계열(2~4색), 배경은 항상 짙은 먹청/남색, 흰색은 절대 100% 로 가지 않게(`dyeClamp` 와 같은 상한)
5. **입력은 힘, 염료는 조금** — 포인터가 색을 "그리는" 게 아니라 흐름을 "건드리는" 느낌이 되도록 힘 대 염료 비율을 낮게
6. **첫 프레임이 비어 있지 않게** — 진입 시 burst 몇 방울 (이미 있음)

## 공통 기술 메모

- **입력 추상화는 이미 있다.** `PlayScreen` 의 포인터 · 휠 · 기울기 · 흐름을 엔진 인터페이스(`splat(x,y,dx,dy)` 류)로 넘긴다. 새 엔진은 같은 메서드 이름을 갖게 하면 UI 를 다시 만들지 않는다
- **햅틱은 기대하지 말 것.** `navigator.vibrate` 는 iOS Safari 가 지원하지 않는다. 체크박스 트릭이 iOS 17.4~26.4 에서만 통했고 26.5 에서 막혔다. 촉감 대신 **소리**(옵션 · 기본 끔)로
- **소리는 Tone.js** — 반드시 사용자 제스처(탭) 뒤에 `Tone.start()`. 기본은 무음, 도구 바에 스피커 토글
- **성능 상한** — `devicePixelRatio ≤ 2`, 시뮬레이션 해상도는 128~256, 저사양은 `supportLinearFiltering` 로 이미 낮춘다
- **WASM 라이브러리(rapier · sandspiel · LiquidFun)** 는 첫 로딩 1~3MB. 장면 진입 때 지연 로드하고 로딩 표시를 둔다

## 라이선스 주의

| 상태 | 대상 | 어떻게 |
|---|---|---|
| MIT · Apache · Unlicense | 위 표에 표시된 것들 | 코드 사용 가능. 저작권 고지는 `docs/THIRD_PARTY.md` 에 모아 둔다 |
| **라이선스 없음** | interactive-particles · RainEffect · falling-sand-shader · zen-garden · cloth-sim · mouse-effects-webgl-water | **기법만 참고**하고 코드는 직접 쓴다. 저장소에 LICENSE 가 없으면 법적으로 all rights reserved 다 |
| **비오픈소스** | Sand Game JS · Weave Silk · Vibe Chimes | 참고 · 벤치마크만 |

## 관련

[[CalmTouch]] · [[여섯 앱 디자인 시스템]] · [[레퍼런스 사이트 분석 방법]]
