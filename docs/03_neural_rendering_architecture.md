# 03. 3D-Guided Neural Rendering 아키텍처 (NVIDIA 리서치 발표 기반)

> 소스 코드가 없으므로(`docs/02`) 이 문서는 전적으로 NVIDIA의 공개 리서치 발표(웹페이지 +
> 기술 리포트 PDF)에 의존합니다. **2026-09-04 재검증**: 원본 기술 리포트 PDF
> (`_scratch/DLSS5_Report.pdf`, 293MB, 24페이지, LaTeX 조판, 발행일 2026-09-01)를
> `pypdf`로 페이지별 텍스트를 직접 추출해 전체를 재검증했고, 리서치 웹페이지
> (`research.nvidia.com/labs/adlr/DLSS5/`)도 WebFetch로 재대조했습니다. 그 결과 아래
> 두 곳에서 실질적 오류를 발견해 수정했습니다(§2, §5) — 이전 버전은 등급을 매겼음에도
> 원문과 다른 내용을 "등급 (a)"로 표기하고 있었습니다.

## 1. 모델 구조

**등급 (a) — PDF `_scratch/DLSS5_Report.pdf` p.1 Abstract, p.3 Sec 1.2, p.6 Sec 2.3~2.4
원문 직접 확인**: 정확한 표현은 **"one-step pixel-space diffusion model"**(1스텝 픽셀
공간 디퓨전 모델)입니다. 렌더링 파이프라인 끝단에 붙는 생성형 렌더링 스테이지로, 게임
엔진이 이미 그린 완성된 컬러 프레임을 입력받아 "그 프레임의 재질과 조명이 보이는 방식을
바꾼다(changes how materials and illumination look)"(p.3 Sec 1.2).
> 재검증 메모: 이전 버전은 이를 "single-stage"로 표기했으나, 원문 전체(24페이지)를
> 재확인한 결과 리포트는 일관되게 "one-step"이라는 표현을 씁니다 — diffusion step 수가
> 정확히 **1**이라는 뜻(§3의 "미확인" 항목이었던 diffusion step 수가 이번에 해소됨).

## 2. 입력 데이터 (컨디셔닝)

**등급 (a) — PDF p.4~5 Sec 2.1 원문 직접 확인, 실질 오류 정정**: 이전 버전은
"G-buffer(albedo/normal/depth/roughness/metalness)가 조건화 입력으로 쓰인다"고
적었으나, 원본 리포트를 재확인한 결과 **틀린 내용**이었습니다. 정확한 구조는 다음과
같이 학습 시점과 추론(배포) 시점이 분리됩니다:

- **추론(배포된 모델의 실제 입력, p.2 Abstract "At inference, the model is conditioned
  on the current rendered frame, engine motion vectors, carried temporal state, and
  artistic-direction values")**: 렌더링된 RGB 프레임 자체(dense, pixel-aligned
  observation) + 엔진 모션 벡터 + 캐리된 시간적 상태(temporal state) + 아트 디렉션 값
  (마스크, structure/tone 강도) — **이것이 전부**입니다. albedo·normal·depth 같은
  별도 G-buffer 채널은 배포된 모델에 입력으로 들어가지 않습니다.
- **학습(오직 학습 때만 쓰는 감독 신호, p.5 Sec 2.1 "These signals are used only during
  training and are not inputs to the shipped model")**: 렌더러가 만든 **albedo, surface
  normal, lighting(조명) 유래 정보**만 "attribute-specific consistency supervision"으로
  사용됩니다. **depth, roughness, metalness는 24페이지 전체에서 조건화·학습 신호
  어디에도 언급되지 않습니다** — 이전 버전의 "G-buffer 전체를 조건화에 쓴다"는 서술은
  일반적인 G-buffer 개념을 DLSS5에 그대로 투영한 추정이었을 뿐, 원문 근거가 없었습니다.
  (참고: albedo/normal/depth는 §4 평가 지표(structure alignment 측정)에서도 등장하지만,
  이는 생성 결과의 구조 정합성을 사후에 비교하는 평가용 버퍼이지 모델 입력이 아닙니다.)

개발자는 마스크와 구조/톤 강도(structure/tone intensity, Sec 3)를 조절해 DLSS5가 장면의
어느 부분을 얼마나 바꿀지 제어할 수 있습니다 — 이 부분은 원래 서술대로 정확합니다.

## 3. 학습 방법

**등급 (a) — supervision 신호와 diffusion step 수는 확인됨, loss function·데이터셋은
여전히 미확인**: 학습 시 렌더러가 만든 albedo, surface normal, lighting 신호로
"attribute-specific consistency supervision"을 수행해(p.5 Sec 2.1), 생성 결과가
"저작된 장면(authored scene)"에서 벗어나지 않도록 그라운딩됩니다. Diffusion step 수는
**1(one-step)**로 확정 확인(등급 a, 위 §1 참고). 다만 **정확한 손실 함수 수식**과
**학습 데이터셋의 구체적 구성(출처, 규모, 라이선스)**은 24페이지 전체를 재확인해도
리포트 어디에도 공개돼 있지 않습니다(등급 d, 원본 리포트가 유일한 1차 문서이므로 추가
공개 문서 없이는 해소 불가) — Eq. 1~2(p.4)는 "학습이 지향하는 목표"를 개념적으로
서술한 것이지("informally"라고 원문이 명시) 실제 학습 절차의 수식이 아닙니다.

## 4. 추론 방식

**등급 (a) — PDF p.2 Abstract/Sec 1.2, p.7 Sec 2.6 확인**: RTX 50 시리즈 GPU에서
**온디바이스** 실행, "기존 게임 파이프라인 내 렌더링 단계로 로컬 실행"(클라우드 추론
아님). 원문은 "Blackwell GPUs with TMA instructions"에 최적화했다고만 명시할 뿐,
**"5세대 Tensor Core"라는 표현 자체는 24페이지 어디에도 나오지 않습니다** — 이 표현은
미디어 인용(등급 b~c)에서 온 것이며, 원본 리포트가 이를 독립적으로 강화해주지는
않습니다(등급 c로 유지, 아래 §7 참고).

## 5. "3D-Guided"라는 이름이 오해하기 쉬운 지점

**등급 (d)로 하향 — 재검증 결과 원문 근거 없음**: 이전 버전은 "NVIDIA 리서치 페이지가
'3D-Guided가 3D 월드 전체를 재계산한다는 뜻이 아니다'라고 직접 경고한다"고 등급 (a)로
적었으나, PDF 24페이지 전체와 리서치 웹페이지를 2026-09-04에 WebFetch로 재대조한 결과
**그런 명시적 부정·경고 문구는 어디에도 없습니다**. 원문은 "3D-guided neural
rendering"을 "a renderer-grounded generative approach"(렌더러에 근거를 둔 생성
방식)라고 정의할 뿐(p.3 Sec 1.2), "오해하기 쉽다"는 언급이나 이를 해명하는 문장은
없습니다.

아래 서술은 원문의 "renderer-grounded" 정의와 §2 재검증 결과(추론 시 RGB 프레임 +
모션벡터 + 시간상태 + 아트디렉션만 입력)를 근거로 한 **이 저장소의 해석적 추론**이지,
NVIDIA가 직접 한 말이 아닙니다: DLSS5는 지오메트리·카메라·애니메이션·기본 장면 구조는
여전히 게임 엔진이 렌더링하고, DLSS5는 그 위에 조명/재질 외형만 생성형으로
재해석한다고 볼 수 있습니다(등급 d, 추가 1차 출처 없이는 확정 불가).

## 6. 이전 DLSS(4/4.5)와의 파이프라인 위치 차이

| | 기존 DLSS 1~4.5 | DLSS5 |
|---|---|---|
| 문제 유형 | 재구성(부족한 픽셀/프레임 추정) | 생성(완성된 프레임의 외형 재해석) |
| 파이프라인 위치 | 렌더링 중간(업스케일링) 또는 프레임 사이(프레임생성) | 렌더링 파이프라인 **끝단** |
| 입력 | 저해상도 프레임, 모션 벡터 | 완성된 컬러 프레임 + 모션벡터 + 시간적 상태 + 아트디렉션 값 (§2 재검증, G-buffer 채널 아님) |
| 목표 | 레퍼런스(고비용 렌더링 결과)에 근접 | 새로운 최종 외형 자체를 생성 |

## 7. 공개 안 된 것 (등급 명시로 과신 방지)

**2026-09-04 재검증으로 갱신**:
- ~~파라미터 규모, VRAM 사용량, 프레임타임 오버헤드: NVIDIA 미공개~~ → **이제 공개됨**
  (등급 a, PDF p.7 Sec 2.6 / p.17 Sec 4.3.3): 약 154M 파라미터 트랜스포머, RTX 5090
  기준 731MB VRAM, 4K 기준 약 8ms/프레임. 상세는 아래 §8. back2gaming의 "unpublished"
  확인은 이 기술 리포트(2026-09-01 발행) 이전 시점 보도였을 가능성이 높습니다.
- RTX 40 시리즈가 안 되는 구체적 기술 사유(단순 "Blackwell GPU + TMA 명령어 최적화"라는
  구현 선택 외에, 40시리즈가 원천적으로 불가능한 아키텍처적 근거): 여전히 미확인(등급 c)
  — 리포트는 "we currently optimize the implementation for Blackwell GPUs"라고만
  말할 뿐, 40시리즈 배제 이유를 설명하지 않습니다.
- 정확한 loss function 수식, 학습 데이터셋 구성(출처/규모/라이선스): 여전히
  미확인(등급 d) — §3 참고. Diffusion step 수는 이번 재검증으로 **1(one-step)**로 확정
  확인되어 이 목록에서 제외.

## 8. 원본 리포트에서 새로 확인된 아키텍처·평가 사실 (등급 a, 2026-09-04 추가)

**아키텍처 (p.7~8 Sec 2.6)**: 트랜스포머 백본, 약 1억 5,400만(154M) 파라미터. Attention
레이어는 DLSS4(`NVIDIA Corporation, 2025b`)에서 쓴 트랜스포머 구조를 재사용·확장.
대부분의 GEMM·attention 연산은 FP8, 수치에 민감한 일부만 FP16 유지 — FP16 대비 화질
저하는 작고 추론 속도는 약 2배 향상. Blackwell GPU의 TMA(Tensor Memory Accelerator)
명령어를 활용해 최적화.

**평가 방법 (p.11~17 Sec 4)**: 109개 정합 장면(path-traced 20개 + non-path-traced 89개)
데이터셋으로 GPT Image 2, Gemini 3 Pro(Nano Banana Pro) 두 오프라인 파운데이션 모델과
비교. DINOv2 patch distance·LPIPS(콘텐츠/정체성 정합), albedo SSIM/PSNR·normal 각도
오차·depth Spearman 상관(구조 정합, 이 세 버퍼는 **평가 지표로만** 등장 — §2의 정정
참고), 블라인드 페어와이즈 유저 스터디(794건, HUD 편향 21건 제외)로 체감 사실성 측정.
DLSS5는 구조 정합 전 지표에서 1위, 유저 스터디에서는 GPT Image 2와 대등(50.63% vs
49.37%), Gemini 3 Pro보다 우세(62.00% vs 38.00%), 특히 캐릭터 중심 콘텐츠에서 격차가
더 큼.
