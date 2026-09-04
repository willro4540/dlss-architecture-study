# 검증 로그 (verification_log.md)

이 저장소의 적대적 재검증(adversarial re-verification) 기록입니다. 방법론은
`research-verification` 스킬을 따릅니다 — "이미 근거 등급이 매겨져 있으니 믿을만하다"가
아니라 "이 문서의 모든 것이 틀렸을 수 있다고 가정하고 하나씩 재확인한다"는 마음가짐으로
원 작성자와 별도 세션에서 수행했습니다.

## 2026-09-04 — DLSS5_Report.pdf 페이지별 재검증

**대상**: `_scratch/DLSS5_Report.pdf` (NVIDIA "DLSS 5: Real-Time Generative Rendering"
기술 리포트, 발행일 2026-09-01, LaTeX 조판, 24페이지, 293MB — 이전 세션에서는 10MB 제한
때문에 자동 도구로 읽지 못하고 리서치 웹페이지 요약에만 의존했었음).

**검증 방법(재현 가능한 절차)**:
```bash
# 1. PDF 페이지 수/메타데이터 확인
python3 -c "
from pypdf import PdfReader
r = PdfReader('_scratch/DLSS5_Report.pdf')
print('pages:', len(r.pages))
print('metadata:', r.metadata)
"
# 결과: pages: 24, Creator: LaTeX with hyperref, CreationDate: 2026-09-01

# 2. 24페이지 전체 텍스트 추출 (Read 도구는 100MB 초과 PDF를 못 열어서 우회)
PYTHONIOENCODING=utf-8 python3 -c "
from pypdf import PdfReader
r = PdfReader('_scratch/DLSS5_Report.pdf')
for i, p in enumerate(r.pages):
    print('=====PAGE', i+1, '=====')
    print(p.extract_text() or '')
" > dlss5_pages_all.txt
# 3. 추출된 텍스트를 페이지 순서대로 처음부터 끝까지 직접 읽고 기존 docs/01, docs/03,
#    README.md의 모든 주장과 한 줄씩 대조
```
추가로 `research.nvidia.com/labs/adlr/DLSS5/` 웹페이지를 WebFetch로 재조회해, PDF와
웹페이지 두 1차 출처가 서로 일치하는지도 교차 확인했습니다.

| 문서/섹션 | 검증 대상 주장 | 분류 | 근거 등급 | 검증 방법 | 검증 결과 | 비고 |
|---|---|---|---|---|---|---|
| docs/01 §1 | "the first DLSS technology that generates final display appearance, rather than reconstructing an expensive reference result of a legacy renderer" — 직접 인용 표기 | 로직 | (a)→정정 | PDF p.2 원문 대조 + `research.nvidia.com/labs/adlr/DLSS5/` WebFetch 재조회 | **오류(경미)**: 실제 원문은 "DLSS 5 is the first DLSS technology to generate the final displayed appearance rather than reconstruct a higher-cost reference output from the conventional renderer." 의미는 동일하나 단어 선택이 달라 verbatim 인용이 아니었음. 두 1차 출처(PDF·웹페이지)는 서로 완전히 일치 확인 | docs/01 정정 완료 |
| docs/03 §1 | 모델을 "single-stage pixel-space diffusion"으로 서술 | 아키텍처/로직 | (a)→정정 | PDF p.1,3,6 Sec 1.3/2.3/2.4 대조 | **표현 정정**: 원문은 일관되게 "one-step"(1스텝) 사용. Diffusion step 수가 정확히 1이라는 뜻이며, 이전 "미확인" 항목이 이번에 해소됨 | docs/01·03 정정 완료 |
| docs/03 §2 | G-buffer(albedo/normal/depth/roughness/metalness)가 조건화 **입력**으로 쓰인다 | 로직/아키텍처 | (a)→**CONFIRMED 오류** | PDF p.2 Abstract, p.4~5 Sec 2.1 원문 정독 + WebFetch로 웹페이지도 동일 여부 재확인 | **실질 오류**: 배포된 모델의 실제 추론 입력은 RGB 프레임 + 모션벡터 + 시간적 상태 + 아트디렉션 값뿐. Albedo/normal/lighting은 **학습 시에만** 쓰는 supervision 신호이고 "not inputs to the shipped model"이라고 원문이 명시. depth/roughness/metalness는 24페이지 전체·웹페이지 어디에도 등장하지 않음(평가 지표로서 albedo/normal/depth가 등장하나 이는 사후 구조 정합성 측정용) | docs/03 §2, README 다이어그램 정정 완료 |
| docs/03 §5 | "3D-Guided"라는 이름을 NVIDIA가 "원문 명시적으로 부정/경고"한다 | 로직 | (a)→**CONFIRMED 오류** | PDF 24페이지 전체 재독 + WebFetch로 웹페이지 재조회, "misconception"/"does not mean" 류 문구 검색 | **실질 오류**: 그런 명시적 경고 문구는 PDF에도 웹페이지에도 없음. 원문은 "3D-guided neural rendering"을 "a renderer-grounded generative approach"로 정의할 뿐, 오해 소지를 해명하는 문장은 없음 | docs/03 §5 등급 (a)→(d)로 하향, 저장소 자체 해석으로 재작성 완료 |
| docs/01 §4, docs/03 §7 | 파라미터 수/VRAM/레이턴시가 "NVIDIA 미공개"(back2gaming 인용, 등급 b) | 아키텍처 | (b)→**갱신 필요** | PDF p.7 Sec 2.6, p.17 Sec 4.3.3 원문 확인 | **최신 정보 아님**: 2026-09-01 발행 리포트에 154M 파라미터, RTX 5090 기준 731MB VRAM, 약 8ms/프레임(4K)이 실제로 공개돼 있음. 이전 "미공개" 보도는 이 리포트 발행 이전 시점이었을 가능성 | docs/01, docs/03 §7~8 갱신 완료 |
| docs/03 §4 | RTX50 전용, "5세대 Tensor Core" 타겟 | 아키텍처 | (a)→(c)로 명확화 | PDF 24페이지 전체에서 "Tensor Core" 관련 언급 검색 | RTX50 전용/온디바이스 실행은 (a) 확인됨(p.2, p.7). 다만 "5세대 Tensor Core"라는 표현 자체는 원문에 없음(원문은 "Blackwell GPUs with TMA instructions"만 언급) — 미디어 인용 등급(c)을 원본 리포트가 강화해주지 않음을 명시 | docs/03 §4 정정 완료 |
| docs/03 §3 | 학습 방법 = "renderer-derived scene property consistency supervision", loss function/데이터셋 미확인 | 로직 | (a) 유지 / (d) 유지 | PDF p.4~5 Sec 2.1, p.4 Eq.1~2 원문 확인 | **이상 없음(부분 확인)**: supervision 설명은 원문("attribute-specific consistency supervision")과 취지 일치, 정확한 재인용으로 갱신. 정확한 loss 수식과 학습 데이터셋 구성은 24페이지 전체에서도 끝내 공개되지 않음을 재확인(Eq.1~2는 "informally"라고 원문이 명시한 개념적 서술일 뿐) | docs/03 §3 재확인 완료, 오류 없음 |
| docs/01 §3 | RTX40 이하 미지원 사유 미공개 | 하드웨어 | (c) 유지 | PDF 24페이지 전체에서 RTX40/이전 세대 미지원 사유 검색 | **이상 없음**: 원문에도 40시리즈 배제의 구체적 아키텍처 근거는 없음(구현을 Blackwell+TMA에 최적화했다는 서술뿐) — 기존 등급(c) 그대로 정확 | 변경 없음 |
| docs/03 (신규) | — | 아키텍처/평가 | (a) 신규 | PDF p.7~8 Sec 2.6, p.11~17 Sec 4 전체 정독 | **신규 확인 사실**: 154M 파라미터 트랜스포머(DLSS4 attention 재사용), FP8 위주 + 일부 FP16, TMA 최적화, 109개 장면 평가셋, GPT Image 2/Gemini 3 Pro와 비교, 유저스터디 794건 | docs/03 §8로 신규 추가 |

**이번 재검증 범위**: `DLSS5_Report.pdf` 24페이지 전체 + 이 PDF를 인용하는 `docs/01`,
`docs/03`, `README.md`의 관련 서술만 대상으로 했습니다. `docs/02`, `docs/04~06`
(SDK 실측, 교차검증, 인젝션 생태계)은 이 PDF와 무관한 별도 근거(로컬 SDK 헤더 grep,
서드파티 도구 README)에 기반하므로 이번 패스에서는 재검증하지 않았습니다.

**총평**: 문서 6곳 중 2곳(§2 입력 컨디셔닝, §5 "3D-Guided" 오해 해명)에서 **등급 (a)로
표기됐음에도 원문에 없는 내용**을 확정 사실처럼 서술한 실질 오류를 발견해 정정했습니다.
나머지는 표현 정밀화(단어 선택 불일치, "single-stage"→"one-step") 또는 최신 정보 반영
(파라미터/VRAM 공개 갱신)이었고, 1곳(§3, 학습 방법)은 이상 없음을 확인했습니다.
