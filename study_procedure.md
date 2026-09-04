# 세션 기록 (session log)

| 날짜 | 무엇을 했는가 | 방법 | 다음 할 일 |
|---|---|---|---|
| 2026-09-04 | 저장소 신설. DLSS5가 실존 기술인지부터 검증(2026-03-16 발표, 2026-09-03 정식 출시, RTX 50 전용 확인). 공식 `github.com/NVIDIA/DLSS`를 `_source/`에 클론(커밋 `a291cc7`, 1.5GB라 `.gitignore` 처리)해 SDK가 헤더+사전컴파일 DLL뿐임을 실측, `nvsdk_ngx_defs.h`의 `NVSDK_NGX_Feature` enum(0~18)에 DLSS5용 항목이 없음을 grep으로 직접 확인(등급 a). NVIDIA 리서치 페이지(`research.nvidia.com/labs/adlr/DLSS5/`)로 3D-Guided Neural Rendering 아키텍처(single-stage pixel-space diffusion) 확인. 사용자가 세션 중간에 던진 "강제 인식" 질문을 계기로 NVIDIA App 공식 오버라이드(`ApplicationStorage.json`+`fingerprint.db`)와 서드파티 DLL 인젝션(1-Click-DLSS5, OptiScaler, ReShade addon) 두 레이어를 분리 분석. `streamline-architecture-study`/`nvidia-ace-architecture-study`와 교차검증해 "핵심 IP는 닫고 배관만 연다" 패턴의 극단형(배관조차 미공개)임을 확인. docs/01~06, README, study_procedure.md 작성. | 공식 SDK 헤더 직접 grep 실측 + NVIDIA 리서치 페이지/뉴스 인용 + 서드파티 도구 README 자기소개 인용(등급 c~d로 명시 하향, 참고 원칙: [[feedback_reference_not_reuse]]) | `DLSS5_Report.pdf`(10MB 초과로 자동 도구로 미독) 로컬 다운로드 후 페이지 단위 직접 재검증(diffusion step 수, loss function, 학습 데이터셋 확인), `NVIDIA/DLSS` 리포 최신 커밋 재확인해서 DLSS5 API 반영 여부 갱신, `ApplicationStorage.json` 실제 파일 로컬 확인 |

## 용어 사전 (새 용어 나올 때마다 추가)

| 용어 | 발음 | 한글 번역/뜻 |
|---|---|---|
| DLSS5 (3D-Guided Neural Rendering) | 디엘에스에스 파이브 | 렌더링 파이프라인 끝단에서 조명/재질 외형을 생성형 AI로 재해석하는 기술. 기존 DLSS(재구성)와 카테고리가 다름(생성) |
| NGX | 엔지엑스 | NVIDIA의 신경망 추론 런타임/SDK 계층. DLSS/DLSS-G/DLSS-RR이 전부 이 위에서 동작 |
| Pixel-space diffusion | 픽셀 스페이스 디퓨전 | 잠재공간이 아니라 픽셀 자체에서 직접 노이즈 제거 과정을 거치는 확산 모델 방식 |
| G-buffer | 지버퍼 | 렌더링 중간에 생성되는 알베도/노멀/뎁스/러프니스 등 장면 속성 데이터 묶음 |
| Fingerprint DB | 핑거프린트 디비 | 게임 실행파일을 식별하기 위한 NVIDIA의 지문 데이터베이스(App이 게임 자동 인식에 사용) |
| DLL 프록시 주입 | 디엘엘 프록시 인젝션 | 원본 DLL 자리에 가짜 DLL을 놓아 호출을 가로채는 표준 로컬 후킹 기법(ReShade가 대표적) |
| PE Import Table | 피이 임포트 테이블 | 실행파일이 어떤 외부 DLL/함수를 쓰는지 정적으로 기록된 헤더 영역 |

## 오늘 세션 중단 지점 (2026-09-04)

- **지금까지 확인된 것**: README.md, docs/01~06, study_procedure.md, .gitignore 작성 완료.
  `_source/`는 로컬에만 존재(1.5GB, 미커밋). git init만 완료, 첫 커밋은 아직 안 함.
- **아직 미완성인 것**: git add/commit — 이 시리즈는 관례상 사용자가 직접 커밋
  ([[feedback_user_does_own_commits]] 원칙은 v2 프로젝트 한정이지만, 이 스터디도
  Claude가 대신 commit/push하지 않고 명령만 제공하는 편이 일관적).
- **다음에 이어서 할 구체적 단계**: (1) `DLSS5_Report.pdf` 로컬 다운로드 후 직접 재검증,
  (2) `NVIDIA/DLSS` 리포 재클론으로 DLSS5 헤더 반영 여부 갱신, (3) NVIDIA App을 실제
  설치해 `ApplicationStorage.json` 실물 구조 확인(로컬에 RTX GPU 없으면 스킵 가능,
  [[reference_user_machines]] 참고 — 사용자 두 컴퓨터 다 내장그래픽이라 이 항목은
  우선순위 낮음).
