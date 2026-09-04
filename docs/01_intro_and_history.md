# 01. DLSS5가 뭔지, 기존 DLSS와의 성격 차이, 타임라인

## 1. DLSS 시리즈 안에서 DLSS5의 위치

**등급 (b) — 복수의 독립 매체 보도로 확인, NVIDIA 원문 문구는 리서치 페이지 문구 인용**:
DLSS 1~4는 전부 "정보가 부족한 상태(저해상도 프레임, 또는 프레임 사이 간격)를 AI로
**재구성(reconstruct)**"하는 기술이었습니다 — 업스케일링(해상도 복원), 프레임 생성(중간
프레임 보간), 레이 리컨스트럭션(디노이징)까지 전부 "이미 있어야 할 정답에 가까운 값을
추정"하는 문제였습니다.

DLSS5는 다릅니다. NVIDIA 리서치 페이지(`research.nvidia.com/labs/adlr/DLSS5/`) 인용:
"the first DLSS technology that generates final display appearance, rather than
reconstructing an expensive reference result of a legacy renderer"(등급 a, 직접 인용).
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

- DLSS5의 실제 파라미터 수, VRAM 사용량, 프레임당 레이턴시 오버헤드는 **NVIDIA가 공개
  안 함**(back2gaming 리뷰에서 명시적으로 "unpublished" 확인, 등급 b). 벤치마크 매체의
  자체 실측이 나오면 갱신 필요.
- `research.nvidia.com/labs/adlr/files/DLSS5_Report.pdf`(기술 리포트)는 파일 크기가
  10MB를 넘어 이번 세션에서 자동 도구로 전체를 읽지 못했습니다 — 로컬 다운로드 후
  페이지 단위로 직접 읽는 재검증이 필요합니다(다음 세션 TODO, `study_procedure.md` 참고).
