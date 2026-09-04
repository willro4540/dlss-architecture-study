# 01. DLSS5가 뭔지, 기존 DLSS와의 성격 차이, 타임라인

## 1. DLSS 시리즈 안에서 DLSS5의 위치

**등급 (b) — 복수의 독립 매체 보도로 확인, NVIDIA 원문 문구는 리서치 페이지 문구 인용**:
DLSS 1~4는 전부 "정보가 부족한 상태(저해상도 프레임, 또는 프레임 사이 간격)를 AI로
**재구성(reconstruct)**"하는 기술이었습니다 — 업스케일링(해상도 복원), 프레임 생성(중간
프레임 보간), 레이 리컨스트럭션(디노이징)까지 전부 "이미 있어야 할 정답에 가까운 값을
추정"하는 문제였습니다.

DLSS5는 다릅니다. NVIDIA 원문 정확한 인용(등급 a — 원본 PDF `DLSS5_Report.pdf` p.2 및
`research.nvidia.com/labs/adlr/DLSS5/` 웹페이지 양쪽에서 동일 문구로 2026-09-04 직접
재대조 완료): **"DLSS 5 is the first DLSS technology to generate the final displayed
appearance rather than reconstruct a higher-cost reference output from the conventional
renderer."**
> 재검증 메모: 이전 버전 문서는 이 문장을 "the first DLSS technology that generates
> final display appearance, rather than reconstructing an expensive reference result of
> a legacy renderer"로 인용하며 "직접 인용"이라 표기했으나, 실제 원문과 단어 선택이
> 달라 verbatim 인용이 아니었습니다(의미는 동일). 이번 재검증에서 정확한 원문으로
> 교체했습니다.

즉 "정답에 가까운 값 추정"이 아니라 **렌더러가 이미 그린 프레임의 조명·재질 외형을
생성형 AI가 다시 그림** — 카테고리 자체가 재구성(discriminative/regression)에서
생성(generative)으로 바뀐 것입니다.

## 2. 타임라인

**등급 (b) — 복수 매체 교차 확인**:
| 시점 | 사건 |
|---|---|
| 2026-03-16 | 최초 발표. NVIDIA가 "2018년 실시간 레이트레이싱 이후 가장 큰 그래픽스 돌파구"로 소개 |
| 2026-04-21 | 공개 SDK `v310.6.0` — DLSS Frame Generation 5x/6x 모드 추가 (기존 DLSS4 계열 기능, DLSS5 아님) |
| 2026-06-23 | 공개 SDK `v310.7.0`(이 스터디의 `_source/` 기준 커밋) — Nsight 트래킹 개선, 버그 수정만. **DLSS5 API는 이 시점까지 미반영**(등급 a, 실측 — `docs/02`) |
| 2026-09-03 | DLSS5 정식 출시. `NBA 2K27` 첫 지원 타이틀. 이후 15개 타이틀 확정 발표(Starfield, Assassin's Creed Shadows, Hogwarts Legacy, Oblivion Remastered 등) |

## 3. 하드웨어 요구사항

**등급 (b)**: GeForce RTX 50 시리즈 전용, RTX 40 이하 미지원. 근거 매체는 "5세대 Tensor
Core를 타겟"이라고만 설명할 뿐 왜 40시리즈가 안 되는지 구체적 기술 사유(예: 특정 명령어
세트, 메모리 대역폭 요구치)는 공개 자료에서 확인되지 않았습니다(등급 c — 미확인 항목으로
남김, 추후 NVIDIA가 공식 기술 사유를 밝히면 갱신).

## 4. 열린 질문

**2026-09-04 재검증으로 해소된 항목** (등급 (a) — 원본 PDF `DLSS5_Report.pdf` 24페이지
전체 텍스트 직접 재검증, `_scratch/DLSS5_Report.pdf` p.7 Sec 2.6 / p.17 Sec 4.3.3):
- 파라미터 수: 약 1억 5,400만(154M) 파라미터 트랜스포머 백본 (p.7).
- VRAM 사용량: 731MB 피크 GPU 메모리 (RTX 5090, 3840×2160 기준, p.17).
- 프레임당 레이턴시: 약 8ms/프레임 (RTX 5090, 4K 기준, p.17).
- 이전 문서는 이 세 항목을 "NVIDIA 미공개"(back2gaming 인용, 등급 b)로 남겨뒀으나, 이
  기술 리포트(2026-09-01자)가 정식 출시(2026-09-03) 직전에 공개한 것으로 확인되어
  더 이상 미공개가 아닙니다. 상세는 `docs/03` 참고.

**여전히 미확인인 항목**:
- RTX 40 이하가 안 되는 구체적 아키텍처 사유(단순 "Blackwell GPU + TMA 명령어 최적화"
  라는 구현 선택 외에, 40시리즈가 원천적으로 불가능한 이유)는 리포트에도 명시 안 됨(등급 c).
- 정확한 손실 함수(loss function) 수식과 학습 데이터셋 구성(출처, 규모, 라이선스)은
  24페이지 전체를 재확인해도 리포트에 공개되지 않음(등급 d, `docs/03` §3 참고) — 이
  리포트가 유일한 1차 기술 문서이므로, 추가 공개 문서가 나오지 않는 한 이 항목은
  열린 질문으로 남습니다.
