# FreeCAD + Opencode MCP: 예제 프로젝트 (Bracket Plate)

> 이 예제는 [FreeCAD-Opencode-MCP-GUIDE.md](./FreeCAD-Opencode-MCP-GUIDE.md)의 워크플로우를 실제로 따라해보는 실습 프로젝트입니다. 간단한 **브래킷 플레이트(Bracket Plate)**를 파라메트릭하게 모델링하고, 검증 후 `.FCStd` + `.STEP`으로 산출하는 과정을 다룹니다.

## 1. 목표

Opencode + FreeCAD MCP를 활용해 다음 요구사항을 만족하는 3D 파트를 설계합니다.

| 항목 | 요구사항 | 비고 |
|---|---|---|
| **부품명** | `Bracket_Plate` | 명확한 식별자 사용 |
| **재질 개념** | 알루미늄/강판 등 일반적인 브래킷 (단순 모델) | 치수 위주 실습 |
| **단위** | `mm` | FreeCAD 기본 단위 기준 |
| **방법론** | **Sketcher + PartDesign** (파라메트릭 히스토리 유지) | Boolean 남용 지양 |
| **스케치** | **Fully Constrained** | 구속 기반 설계 |
| **산출물** | `.FCStd`, `.STEP` | 수정성 + 호환성 동시 확보 |
| **검증** | 주요 치수/홀 직경/간격 측정값 기반 검증 | "예쁘다"가 아닌 수치 검증 |

## 2. 설계 스펙

아래 스펙을 기준으로 모델링합니다. Opencode에 그대로 복사해서 입력해도 무방합니다.

| 파라미터 | 값 (mm) | 설명 |
|---|---|---|
| `PLATE_LENGTH` | `60` | 플레이트 전체 길이 |
| `PLATE_WIDTH` | `40` | 플레이트 전체 폭 |
| `PLATE_THICKNESS` | `5` | 플레이트 두께 (Pad 두께) |
| `RADIUS_CORNER` | `4` | 모서리 라운드(필요 시 Chamfer/Fillet 적용) |
| `HOLE_DIAMETER` | `8` | 고정용 원형 홀 지름 |
| `HOLE_X_OFFSET` | `12` | 첫 번째 홀 중심 X 오프셋 (좌측 기준) |
| `HOLE_Y_CENTER` | `20` | 홀 중심 Y (PLATE_WIDTH/2) |
| `HOLE_SPACING` | `24` | 두 홀 사이 간격 |
| `MOUNT_TAB_LENGTH` | `16` | 측면 탭 길이 (선택적 디테일) |

**형상 개요**

- 직사각형 베이스 플레이트 (둥근 모서리 고려)
- 양쪽 또는 한쪽에 장착용 탭이 있는 간단한 L형/플레이트형 브래킷
- 2개의 고정 홀 (일직선 배치)

> 실습의 목적상 **최소 구현(직사각형 + 2홀)** 부터 시작해, 점진적으로 탭/라운드 등을 추가하는 게 가장 안전합니다.

## 3. 준비 상태 확인

실습 전 아래 항목을 반드시 확인하세요.

- [ ] **FreeCAD 실행 중** (GUI 모드)
- [ ] **Opencode 실행 중** (워크스페이스 루트)
- [ ] `.opencode/mcp.json`에 `freecad` MCP 서버가 등록되어 있음
- [ ] FreeCAD Workbench에 `MCP` 항목 노출 확인
- [ ] `uvx --version` 정상 동작
- [ ] 산출물 저장을 위한 **작업 폴더** 생성 예정 (`C:\Users\<YourUser>\Desktop\FreeCAD-Opencode-MCP\examples\bracket_plate\` 권장)

## 4. Opencode에 입력할 프롬프트 (권장)

아래 프롬프트를 Opencode에 그대로 복사해서 실행하면, 가이드의 TodoWrite/검증 원칙을 자동으로 따르게 됩니다.

```text
# 작업 목표
FreeCAD + Opencode MCP를 사용해 "Bracket_Plate"를 파라메트릭(PartDesign, Sketcher 기반)으로 모델링해줘.

# 기본 스펙 (mm)
- PLATE_LENGTH = 60
- PLATE_WIDTH  = 40
- PLATE_THICKNESS = 5
- HOLE_DIAMETER = 8
- HOLE_X_OFFSET = 12
- HOLE_Y_CENTER = 20
- HOLE_SPACING  = 24

# 설계 원칙 (반드시 준수)
1. PartDesign + Sketcher 우선. Boolean 남용 금지
2. 스케치는 Fully Constrained 상태로 완성
3. 파라메트릭 감각 유지 (가능하면 값을 변수화해 추후 수정 용이하게)
4. 단위는 mm
5. 히스토리(Feature Tree) 유지
6. 생성 후 반드시 수치 검증(치수/홀 간격/홀 지름 등)을 실제 측정값으로 확인

# TodoWrite 사용
작업을 아래 순서로 세분화하여 TodoWrite로 관리하고, 한 번에 하나(in_progress)만 작업해줘.
- [1] 요구사항 정리/확인
- [2] FreeCAD 문서 생성 (문서명: Bracket_Plate)
- [3] Body 생성 및 구조 설정
- [4] 베이스 플레이트 Sketch 작성 (60x40) + 구속 적용
- [5] Pad로 두께 5 적용
- [6] 상단면 기준으로 2개 홀 Sketch 작성 (원형, 중심거리 24, Y=20)
- [7] Pocket으로 홀 제거 (Through All 또는 적정 깊이)
- [8] 형상 검증 (뷰포트 확인 + 주요 치수 측정)
- [9] 수치 검증 (PLATE_LENGTH/PLATE_WIDTH, 두 홀 중심간 거리, HOLE_DIAMETER)
- [10] Export: Bracket_Plate.FCStd 및 Bracket_Plate.step → Desktop/FreeCAD-Opencode-MCP/examples/bracket_plate/
- [11] Export 파일 존재/크기 확인(Bash) 후 최종 요약

# 검증 요구사항
- PLATE_LENGTH == 60 mm, PLATE_WIDTH == 40 mm
- 두 홀 중심 간격 == 24 mm (정확도 허용 ±0.01mm)
- HOLE_DIAMETER == 8 mm
- 두께 == 5 mm
- 문서에 에러/경고 없는 상태

# 추가 지시
- FreeCAD GUI가 실행 중임을 전제로 작업
- 에러 발생 시 전체 재생성 말고, 에러 메시지를 근거로 구체적으로 수정
- 불필요한 설명은 최소화하고, 실행 가능한 액션 위주로 진행
```

## 5. 예상 동작 흐름 (Opencode 관점)

Opencode가 위 프롬프트를 수신하면 대략 아래 순서로 MCP 툴을 호출하게 됩니다.

| 순서 | 예상 MCP/도구 호출 | 설명 |
|---|---|---|
| **1** | `TodoWrite` | 위 11개 항목을 `pending`으로 생성 |
| **2** | FreeCAD MCP 툴 | `new_document("Bracket_Plate")` 또는 유사 툴 호출 |
| **3** | FreeCAD MCP 툴 | `add_body()` → `add_sketch()` (XY 평면 등) |
| **4** | FreeCAD MCP 툴 | `draw_rectangle()` + `add_constraint()` (길이/폭/대칭 등)로 Fully Constrained 유도 |
| **5** | FreeCAD MCP 툴 | `pad(sketch, thickness=5)` |
| **6** | FreeCAD MCP 툴 | 두 홀용 Sketch 생성 → 원 2개 + 구속(위치/간격) |
| **7** | FreeCAD MCP 툴 | `pocket(sketch_holes, through_all=True)` |
| **8** | FreeCAD MCP 툴 | `measure_distance()`, `measure_diameter()` 등 측정 툴 또는 Python `App.ActiveDocument...` 실행 |
| **9** | FreeCAD MCP 툴 | `export_fcstd()`, `export_step()` (대상 경로 지정) |
| **10** | `Bash` | `Test-Path`, `Get-Item` 등으로 파일 존재/크기 검증 (Windows PowerShell 기준) |
| **11** | `TodoWrite` | 각 항목을 `in_progress`→`completed`로 실시간 업데이트 |

> 실제 툴명은 FreeCAD MCP 구현체([neeka-nat/freecad-mcp](https://github.com/neeka-nat/freecad-mcp))의 스키마에 따라 약간 다를 수 있으나, **의도(생성→피처→측정→내보내기)는 동일**합니다.

## 6. 검증 체크리스트 (수동 확인)

Export 완료 후, FreeCAD에서 직접 열어 아래 항목을 확인해도 좋습니다.

| 체크 항목 | 기준 | 확인 방법 |
|---|---|---|
| **Feature Tree** | `Body → Sketch → Pad → Sketch001 → Pocket` 형태 (또는 유사) | FreeCAD Tree View 확인 |
| **Fully Constrained** | Sketch가 "Fully constrained" 상태 | Sketcher 작업대 또는 상태 표시 확인 |
| **전체 길이** | `60.00 mm` | Measure Tool / Dimension |
| **전체 폭** | `40.00 mm` | Measure Tool / Dimension |
| **두께** | `5.00 mm` | Pad 프로퍼티/측정 |
| **홀 지름** | `8.00 mm` | Sketch 원 지름 또는 Hole 측정 |
| **홀 중심 간격** | `24.00 mm` | 두 원 중심 간 거리 측정 |
| **대칭성** | Y=20 기준 좌우 대칭 양호 | 시각+측정 |
| **모서리** | (적용 시) 일관성 확인 | 시각 확인 |
| **파일 존재** | `Bracket_Plate.FCStd`, `Bracket_Plate.step` 모두 존재 | 파일 탐색기 또는 Bash 결과 확인 |
| **STEP 호환성** | 다른 CAD에서 열 수 있는지(간이) | 필요 시 FreeCAD/다른 뷰어로 열어 구조 확인 |

## 7. 확장 과제 (실무형 연습)

기본 예제를 완료한 뒤, 아래 순서로 난이도를 올려보면 실무 감각을 기를 수 있습니다.

| 순번 | 과제 | 학습 포인트 |
|---|---|---|
| **1** | **모서리 라운드(Fillet)** 적용 | `RADIUS_CORNER=4`로 상단/하단 모서리 Fillet 추가 |
| **2** | **스프레드시트 연동** | FreeCAD `Spreadsheet`에 파라미터 정의 → Sketch/Pad에서 참조 (진짜 파라메트릭) |
| **3** | **장착 탭 추가** | `MOUNT_TAB_LENGTH=16` 탭을 한쪽에 추가 (Sketch→Pad) |
| **4** | **체결 고려** | 홀에 간단한 Countersink/Counterbore 개념 적용 (현실적 모델링) |
| **5** | **TechDraw 연습** | 도면 생성(뷰/치수/제목란) 후 PDF/SVG Export (선택) |
| **6** | **Git 버전관리** | `examples/bracket_plate/`를 Git 초기화 → `.FCStd`는 대용량이지만, 변경 시점/의도를 커밋으로 기록 |
| **7** | **리팩터링** | "길이 70, 홀 간격 30"으로 수정해달라고 자연어만 바꿔 재실행 (파라메트릭 수정성 검증) |

## 8. 예상 산출물 구조

실습 완료 시 바탕화면 기준 아래 구조가 형성됩니다.

```text
C:\Users\Administrator\Desktop\FreeCAD-Opencode-MCP\
├── FreeCAD-Opencode-MCP-GUIDE.md
└── examples/
    └── bracket_plate/
        ├── Bracket_Plate.FCStd
        └── Bracket_Plate.step
```

## 9. 자주 묻는 질문 (FAQ)

| 질문 | 답변 |
|---|---|
| **Q. GUI 없이 Headless로도 가능한가?** | A. 기술적으로는 가능하지만, 본 예제의 **시각 피드백 루프 + 검증** 의도상 **GUI 모드가 가장 직관적이고 안정적**입니다. |
| **Q. Sketch가 자꾸 미완전 구속이 될 때는?** | A. "모든 선/점에 구속을 추가해 **Fully Constrained**로 만들어줘" 라고 명시적으로 지시하세요. 치수/수평/수직/대칭/등거리 등을 우선 고려합니다. |
| **Q. Pocket이 Through All이 안 먹힐 때는?** | A. "Type: Through All", "Direction: Normal" 등 명확히 지시하거나, 에러 로그를 보여달라고 요청해 수정하는 게 가장 빠릅니다. |
| **Q. Opencode가 측정값을 잘 못 읽어올 때는?** | A. FreeCAD 측정 툴의 실제 값을 직접 캡쳐/보고하게 하거나, Python으로 `Part.makeCompound` 등 문서 객체를 순회해 길이/거리값을 계산해달라고 요청하세요. |
| **Q. STEP Export 경로가 이상해질 때는?** | A. **절대경로**(예: `C:/Users/Administrator/Desktop/.../Bracket_Plate.step`)를 명확히 지정하도록 지시하는 게 안전합니다. |

## 10. 마무리

이 예제의 핵심은 **"자연어 → 파라메트릭 설계 → 수치 검증 → 표준 포맷(.FCStd+.STEP) 내보내기"**라는 엔지니어링형 루프를 몸소 체험하는 것입니다.

영상에서 보여준 **ChatGPT Desktop + FreeCAD MCP**의 아이디어를 **Opencode의 계획/검증/버전관리력과 결합**하면, 단순 프로토타이핑을 넘어 **재사용 가능한 파라메트릭 부품 라이브러리**까지 확장해갈 수 있습니다.

**추천 첫 실습:** 위 프롬프트를 복사해서 바로 실행해보는 것부터 시작하세요. 완성 후에는 꼭 **Feature Tree + 측정값 체크리스트**를 직접 눈으로 확인해보는 게 가장 큰 학습이 됩니다.
