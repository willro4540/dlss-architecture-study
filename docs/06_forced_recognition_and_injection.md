# 06. 화이트리스트 인식 메커니즘 + 서드파티 강제 주입 생태계

> "강제 인식이라면 어떻게 인식하고 기술을 사용하는지" 질문에서 출발한 문서. **레이어가
> 완전히 다른 두 메커니즘**이 섞여서 논의되는 경우가 많아 분리해서 정리합니다.

## 1. 레이어 1 — NVIDIA 자체 공식 오버라이드 (해킹 아님)

**등급 (a) — NVIDIA 공식 뉴스 포스트로 확인**: NVIDIA App에 "Global DLSS Overrides"라는
공식 베타 기능이 이미 존재(RTX 40 Smooth Motion과 같이 발표됨).

**동작 방식(등급 b, 커뮤니티 도구 README+포럼 교차 확인)**:
1. 게임을 처음 실행하면 NVIDIA App이 `ApplicationOntology fingerprint.db`(NVIDIA가
   배포하는 게임 실행파일 지문 DB)로 게임을 식별.
2. 식별 결과에 따라 `%LocalAppData%\NVIDIA Corporation\NVIDIA app\NvBackend\ApplicationStorage.json`에
   게임별 프로필이 생성되고, `Disable_FG_Override`/`Disable_RR_Override`/
   `Disable_SR_Override`/`Disable_RR_Model_Override`/`Disable_SR_Model_Override` 같은
   키가 기본값(대개 `true` = 오버라이드 비활성)으로 채워짐.
3. 이 값을 `true→false`로 바꾸면 드라이버가 원래 화이트리스트 대상이 아닌 게임에도
   최신 DLSS 프리셋/모델을 강제 적용.

**커뮤니티 도구의 역할**: `kaanaldemir/DLSS-Override-For-All-Games`는 이 JSON 편집을
GUI로 자동화한 것뿐 — NGX 내부 인식 로직을 속이는 게 아니라 **NVIDIA가 이미 만들어둔
공식 스위치를 대신 눌러주는 것**입니다. 인식 자체(지문 매칭)는 그대로 NVIDIA 통제
하에 있고, 바뀌는 건 "인식된 게임에 뭘 켜줄지"뿐입니다.

## 2. 레이어 2 — 서드파티 인젝션 체인 (실제 가짜 인식)

**등급 (c~d) — 해당 도구 자신의 README 자기소개 인용, 독립 검증 없음**: `1-Click-DLSS5`는
NGX 화이트리스트 판정 자체를 건드리지 않고, **그래픽 API 호출을 직접 가로채서 딴 걸
먹입니다**.

메커니즘(전부 해당 도구 README 자기소개 인용, 등급 c):
1. **PE Import Table 스캐닝** — 게임 exe를 정적으로 열어 D3D11/D3D12/Vulkan/OpenGL 중
   무엇을 쓰는지 확인.
2. **DLL 프록시 주입** — 게임이 로드할 `dxgi.dll`/`d3d12.dll`/`version.dll` 자리에 프록시
   DLL을 게임 폴더에 배치, 원본 호출은 통과시키며 중간에서 가로챔. **이건 ReShade가
   10년 넘게 써온 표준적인 로컬 렌더링 후킹 방식이며, DRM 우회나 네트워크 공격이
   아닙니다** — 등급 b(ReShade 방식 자체는 업계에 널리 알려진 공지 사실).
3. **경로별 리다이렉션**(등급 c, 자기소개 인용):
   - 이미 DLSS 붙은 게임 → Streamline interposer(`streamline-architecture-study/docs/03`에서
     이미 등급 a로 실측 확인한 그 후킹 계층)로 NGX 호출을 DLSS5 파이프라인으로 전환.
   - FSR2/XeSS만 있는 게임 → `OptiScaler`(DLSS5 전용이 아닌 기존 유명 커뮤니티 프로젝트)가
     FSR/XeSS 호출을 DLSS-NR 호출로 리다이렉트.
   - 업스케일러가 아예 없는 게임 → ReShade 애드온 체인 + "Lumenite Kernel" 광학흐름으로
     모션벡터를 **자체 추정**해서 DLSS5 입력을 합성.

## 3. 자기비판 — 등급 필터링을 반드시 유지할 것

- 레이어 2의 "성공적으로 동작한다"는 주장은 전부 **개발자 자기 서술**이며, DLSS5 공식
  출시(2026-09-03) 직후 만들어진 신생 개인 프로젝트라 코드 품질·실제 화질·안정성이
  독립적으로 검증되지 않았습니다. 스터디 문서에는 "이런 메커니즘을 자칭한다"까지만
  등급을 매겨 남기고, 실제 성능/화질은 별도 실측 전까지 미확정으로 취급합니다.
- [[feedback_cite_credible_external_analysis]] 원칙에 따라, 활동성(스타 수, 커밋 빈도,
  이슈 대응) 정량 확인 없이 이 저장소를 "검증된 기술"로 인용하면 안 됩니다.

## 4. 실용적 리스크 (회피 기법 아님, 부작용 기록)

게임 폴더에 DLL을 심는 방식이므로, **온라인 대전 게임에서 안티치트(EAC/BattlEye/VAC)에
치트 프로그램으로 오탐될 수 있음**. 이건 우회 방법 안내가 아니라 이 인젝션 방식 자체의
잘 알려진 부작용이며, 싱글플레이/오프라인 게임에서의 그래픽 모딩(ReShade와 동일 범주)
용도로만 안전합니다.

## 5. 열린 질문

- `ApplicationStorage.json`의 실제 전체 스키마(어떤 키가 몇 개나 있는지)는 미확인 —
  실제 파일을 로컬에서 열어봐야 확정 가능(NVIDIA App 설치 후 재검증 필요).
- `fingerprint.db`가 정확히 무엇으로 게임을 식별하는지(exe 해시? 파일명? 서명?)는
  공개 자료에서 확인 안 됨(등급 d, 추정 배제).
