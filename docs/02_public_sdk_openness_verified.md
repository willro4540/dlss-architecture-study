# 02. 공식 GitHub SDK 실측 — 뭐가 공개돼 있고 뭐가 없는지

> 이 문서가 원래 질문("깃허브에 존재하면 뜯어보자")에 대한 직접 답변입니다.

## 1. 저장소 구조 (등급 a, 실측)

`github.com/NVIDIA/DLSS` 저장소를 커밋 `a291cc7`(2026-06-23)에서 직접 클론해 확인:

```
_source/
├── LICENSE.txt                                          # NVIDIA RTX SDKs LICENSE (독점, 8절)
├── DLSS Donut and Sample Source_Pre-Release License.pdf  # 샘플앱 전용 별도 라이선스
├── README.md
├── include/       # C++ 헤더 14개 (API 선언만, 구현 없음)
├── lib/
│   ├── Windows_x86_64/{dev,khr,rel,uwp,vs2010...}/
│   │   └── rel/nvngx_dlss.dll   (58.9MB, 사전컴파일)
│   │   └── rel/nvngx_dlssd.dll  (40.9MB, 사전컴파일)
│   │   └── rel/nvngx_dlssg.dll  (7.5MB, 사전컴파일)
│   └── Linux_x86_64/
├── doc/           # 프로그래밍 가이드 PDF (사용법 문서, 구현 설명 아님)
├── NVIDIAImageScaling/  # 서브모듈, 별도 오픈소스(비-NGX 업스케일러)
└── utils/
```

**핵심**: `include/`는 함수 시그니처·구조체·enum 선언뿐이고 `.cpp` 구현 파일이 단 하나도
없습니다. `lib/`의 3개 DLL은 전부 사전컴파일된 바이너리입니다. 즉 이 저장소로 할 수 있는
것은 "이 API를 내 렌더러에서 어떻게 호출하는가"까지이고, "신경망이 내부에서 어떻게
동작하는가"는 이 저장소만으로는 알 수 없습니다. Streamline(`streamline-architecture-study`)의
`sl.dlss`/`sl.reflex`처럼 `.cpp` 소스가 실제로 있는 경우와 명확히 다릅니다.

## 2. Feature enum 실측 — DLSS5 API가 아직 없다는 직접 증거

**등급 (a), 실측**: `include/nvsdk_ngx_defs.h` 188~236행의 `NVSDK_NGX_Feature` enum
전체를 직접 확인:

```c
NVSDK_NGX_Feature_Reserved0             = 0,
NVSDK_NGX_Feature_SuperSampling         = 1,   // DLSS 업스케일링
NVSDK_NGX_Feature_InPainting             = 2,
NVSDK_NGX_Feature_ImageSuperResolution  = 3,
NVSDK_NGX_Feature_SlowMotion            = 4,
NVSDK_NGX_Feature_VideoSuperResolution  = 5,
NVSDK_NGX_Feature_Reserved1             = 6,
NVSDK_NGX_Feature_Reserved2             = 7,
NVSDK_NGX_Feature_Reserved3             = 8,
NVSDK_NGX_Feature_ImageSignalProcessing = 9,
NVSDK_NGX_Feature_DeepResolve           = 10,
NVSDK_NGX_Feature_FrameGeneration       = 11,  // DLSS-G
NVSDK_NGX_Feature_DeepDVC               = 12,
NVSDK_NGX_Feature_RayReconstruction     = 13,  // DLSS-RR (DLSS4 계열)
NVSDK_NGX_Feature_Reserved14            = 14,
NVSDK_NGX_Feature_Reserved15            = 15,
NVSDK_NGX_Feature_Reserved16            = 16,
NVSDK_NGX_Feature_Reserved17            = 17,
NVSDK_NGX_Feature_Reserved18            = 18,
NVSDK_NGX_Feature_Count,
```

이 enum 어디에도 "NeuralRendering", "3DGuided" 류의 명명된 항목이 없습니다.
`grep -i "DLSS5\|NeuralRender\|3DGuided"`를 헤더 전체에 돌려도 **매치 0건**(등급 a, 실측).
`Reserved0/1/2/3/14~18` 9개 슬롯이 비어있어, DLSS5가 나중에 이 중 하나를 쓰도록 예약된
것일 수는 있지만 이 커밋 시점(2026-06-23, DLSS5 출시 약 2개월 전)에는 **공개 SDK에
DLSS5 전용 API 표면이 전혀 없습니다**.

## 3. 해석 — 결론이 아니라 가설 (등급 c로 명시)

가능한 설명 세 가지를 등급 구분해서 남깁니다(어느 것도 직접 확인은 안 됨):

1. **SDK 리포 갱신 지연** — DLSS5는 9/3 출시됐지만 공개 GitHub 저장소는 별도 배포
   주기라 아직 안 올라왔을 가능성(등급 c, 추정).
2. **초기엔 비공개 얼리액세스 채널로만 배포** — NBA 2K27처럼 소수 확정 타이틀에만
   NVIDIA가 직접 통합 지원하고, 범용 SDK 공개는 나중으로 미뤘을 가능성(등급 c, 추정).
3. **기존 enum 슬롯 재사용** — 예를 들어 `SuperSampling`(1)의 파라미터 확장으로
   구현했을 가능성(등급 d, 근거 없음, 배제 못 함).

**다음 세션 TODO**: `github.com/NVIDIA/DLSS`의 최신 커밋을 재확인해서 DLSS5 헤더가
반영됐는지 갱신 확인 필요(`study_procedure.md` 열린 질문 참고).

## 4. 라이선스 (등급 a, 실측)

`LICENSE.txt`는 "NVIDIA RTX SDKs LICENSE" — MIT가 아닌 **NVIDIA 독점 라이선스**입니다.
Streamline과 달리:
- 수정/파생물 작성은 허용되지만 "material additional functionality"가 있어야 배포 가능
  (2조 a항).
- 소스 재사용이 자유로운 오픈소스가 아니라, "이 SDK를 써서 앱을 만들 권리"를 주는
  라이선스입니다. [[feedback_reference_not_reuse]] 원칙이 여기선 선택이 아니라
  **라이선스 자체의 강제 조건**입니다 — 구조 참고를 넘어서면 계약 위반입니다.
