# FreeCAD + Opencode MCP 연동 가이드

> 이 문서는 유튜브 영상 *"I Gave GPT-6 Astra Control of FreeCAD (It Designs Like an Engineer)"* ([M5KOgtk9VfI](https://youtu.be/M5KOgtk9VfI)) 의 접근 방식을 기반으로, **ChatGPT Desktop MCP 연동 방식을 Opencode(CLI 에이전트)에 그대로 이식**하여 실무형 CAD 자동화 워크플로우로 정리한 가이드입니다.

## 1. 개요

영상의 핵심은 **LLM이 MCP(Model Context Protocol)를 통해 FreeCAD를 직접 제어**하는 방식입니다. LLM이 FreeCAD Python 코드를 생성하고, MCP 브릿지를 통해 FreeCAD에 전송하면 FreeCAD가 즉시 실행하며, 뷰포트 스냅샷을 피드백으로 받아 반복적으로 설계를 개선해 나가는 **시각적 피드백 루프(Observe → Generate → Execute → Verify)**를 구현합니다.

이 가이드는 동일한 FreeCAD MCP 서버([`neeka-nat/freecad-mcp`](https://github.com/neeka-nat/freecad-mcp))를 Opencode에 연결하여, CLI 환경에서도 동일한 루프를 활용하되 **작업 구조화, 파일/버전 관리, 검증 자동화** 측면에서 실무에 더 적합하도록 최적화합니다.

## 2. 핵심 원리 (영상 요약)

| 단계 | 동작 | 설명 |
|---|---|---|
| 1 | **질의 입력** | 사용자가 자연어로 모델링 요구사항을 입력합니다. |
| 2 | **계획/도구 호출** | LLM이 수행할 작업을 판단하고 MCP 툴 호출을 결정합니다. |
| 3 | **Python 코드 생성** | MCP를 통해 FreeCAD에서 실행할 Python 코드를 생성합니다. |
| 4 | **실행** | FreeCAD MCP 서버가 해당 코드를 FreeCAD Python 환경에서 실행합니다. |
| 5 | **피드백 수집** | FreeCAD가 현재 문서/뷰포트 상태(및 스냅샷)를 MCP를 통해 반환합니다. |
| 6 | **검증/반복** | LLM이 결과를 시각/수치적으로 확인한 뒤, 수정 명령을 다시 전송하여 반복적으로 개선합니다. |

핵심은 **자연어 → FreeCAD Python API 실행 → 시각/상태 피드백 루프**를 MCP로 표준화한 점입니다.

## 3. 아키텍처

Opencode는 MCP 클라이언트이므로, 영상에서 사용한 FreeCAD MCP 서버를 그대로 연결할 수 있습니다.

```text
Opencode (CLI/TUI Agent)
  │
  ├─ MCP Client (stdio transport)
  │   └─ uvx freecad-mcp  (FreeCAD MCP Server, Python)
  │       └─ FreeCAD (GUI 권장 / Headless 선택)
  │            └─ FreeCAD Python API
  │                 ├─ Sketcher
  │                 ├─ PartDesign
  │                 ├─ Part / Body
  │                 └─ TechDraw (선택)
  │
  └─ 로컬 도구 병행
      ├─ Read/Edit/Glob/Grep  (코드/파일 검증)
      ├─ Bash                 (Export, 파일 확인, Git)
      └─ TodoWrite            (작업 구조화/진행 관리)
```

- **전송 방식**: `stdio` (로컬 전용, 가장 단순하고 안정적)
- **실행 원리**: FreeCAD가 **사전에 실행 중**이어야 MCP 서버가 FreeCAD와 연결/제어할 수 있습니다.
- **피드백 루프**: MCP 툴 응답(문서 상태/스냅샷 등)을 기반으로 Opencode가 다음 액션을 스스로 결정합니다.

## 4. Opencode vs ChatGPT Desktop 비교

| 항목 | ChatGPT Desktop (영상) | Opencode (본 가이드) | 비고 |
|---|---|---|---|
| **환경** | 데스크톱 앱 (GUI 중심) | CLI/TUI 에이전트 | 터미널 중심, 자동화/스크립트에 유리 |
| **작업 관리** | 단일 대화 위주 | **[TodoWrite](https://docs.opencode.ai/)**로 다단계 작업 구조화 | 복합 태스크(모델링+도면+내보내기)에 강함 |
| **파일/버전 관리** | 제한적 | `Bash` + `Git` 직접 활용 | `.FCStd`, `.STEP`, `.STL` 버전 추적/리뷰 가능 |
| **도구 조합** | MCP 위주 | **MCP + 로컬 도구** (Read/Edit/Glob/Grep/Bash) 병행 | 검증/후처리/체크를 동일한 세션에서 처리 |
| **탐색/분석** | 제한적 | `Glob/Grep/Task` 병렬 활용 | 레퍼런스 부품/기존 설계 탐색에 유리 |
| **설정/커스텀** | 폐쇄형 | 로컬 설정/에이전트 커스텀 자유도 높음 | 프로젝트별 컨벤션/룰 고정에 최적 |
| **실무 자동화** | 실험/보조 위주 | 설계→검증→산출물 관리까지 **워크플로우 클로징** 가능 | 엔지니어링 워크플로우에 적합 |

## 5. 사전 요구사항

| 소프트웨어 | 권장 버전 | 설치 경로/링크 |
|---|---|---|
| **[FreeCAD](https://www.freecad.org/)** | `1.1.x` (권장) | [https://www.freecad.org/downloads/](https://www.freecad.org/downloads/) |
| **[Opencode](https://opencode.ai/)** | 최신 버전 | [https://opencode.ai/docs/installation](https://opencode.ai/docs/installation) |
| **[uv / uvx](https://docs.astral.sh/uv/)** | 최신 | [https://docs.astral.sh/uv/getting-started/installation/](https://docs.astral.sh/uv/getting-started/installation/) |
| **FreeCAD MCP** | 최신 | [https://github.com/neeka-nat/freecad-mcp](https://github.com/neeka-nat/freecad-mcp) (ZIP 다운로드) |
| **Python** | 시스템 내장/uv가 관리 | `uvx`가 자동으로 처리하므로 별도 관리 불필요 |

> **중요 (Windows)**: `uvx` 설치 후 **반드시 PowerShell을 완전 종료 후 재실행**해야 PATH가 갱신됩니다. (영상 Step 3 참고)

## 6. 설치 절차

### Step 1. ChatGPT Desktop 불필요 (Opencode 사용)
영상은 ChatGPT Desktop 전용이지만, 본 가이드는 **Opencode**를 사용하므로 Step 1의 ChatGPT Desktop 설치는 생략해도 됩니다.

### Step 2. FreeCAD 설치
1. [freecad.org](https://www.freecad.org/)에서 운영체제에 맞는 설치파일 다운로드
2. 설치 진행 (기본 설정 권장)
3. 설치 완료 후 FreeCAD 실행 가능 여부 확인

### Step 3. uvx 설치 (Windows PowerShell, 관리자 권한)

1. `Windows 시작` → `PowerShell` 검색 → `우클릭` → `관리자 권한으로 실행`
2. 공식 설치 명령어 실행 ([uv 설치 가이드](https://docs.astral.sh/uv/getting-started/installation/) 참조)

```powershell
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

3. **PowerShell을 완전 종료**한 뒤, **다시 관리자 권한으로 실행**
4. 설치 확인

```powershell
uvx --version
```

버전이 정상적으로 출력되면 설치 성공입니다.

### Step 4. FreeCAD MCP 애드온 설치

1. [freecad-mcp GitHub](https://github.com/neeka-nat/freecad-mcp) → `Code` → `Download ZIP` 다운로드
2. ZIP 압축 해제
3. 압축 해제 폴더 내부의 `FreeCAD-MCP/` (또는 저장소 루트의 MCP 관련 애드온 폴더)를 FreeCAD Mod 경로에 복사

**FreeCAD Mod 경로 (Windows)**

```text
%APPDATA%\FreeCAD\1.1\Mod\
```

실행 방법: `Win + R` → `%APPDATA%\FreeCAD\1.1\Mod\` 입력 → `Enter`

- 해당 폴더에 `FreeCAD-MCP/` 폴더를 통째로 붙여넣기
- `Mod/` 폴더에 `mod`가 없으면 새로 생성
- **FreeCAD 버전 < 1.1**인 경우: 버전 하위폴더(`1.1/`)가 아닌 **FreeCAD 루트의 `Mod/`**에 복사 (영상 설명 준수)

4. FreeCAD를 **완전 재시작**
5. FreeCAD 상단 Workbench 드롭다운에서 `MCP` 항목이 나타나는지 확인 → 설치 완료

### Step 5. Opencode MCP 서버 설정

Opencode는 `.opencode/mcp.json` 파일을 통해 MCP 서버를 등록합니다. 프로젝트 루트 또는 전역(`~/.config/opencode/` 또는 Windows 환경에 맞게) 설정 가능합니다. 프로젝트별로 고정하는 걸 권장합니다.

**워크스페이스 루트(또는 원하는 위치)에 `.opencode/mcp.json` 생성**

```json
{
  "mcpServers": {
    "freecad": {
      "type": "stdio",
      "command": "uvx",
      "args": ["freecad-mcp"],
      "timeout": 120000
    }
  }
}
```

**설정 옵션 설명**

| 필드 | 값 | 설명 |
|---|---|---|
| `type` | `stdio` | 로컬 프로세스 통신. FreeCAD 제어에 가장 안정적 |
| `command` | `uvx` | `uvx`로 Python MCP 서버를 즉시 실행 |
| `args` | `["freecad-mcp"]` | PyPI/레지스트리의 최신 freecad-mcp 실행 |
| `timeout` | `120000` (ms) | 복잡한 대형 모델링 시 타임아웃 여유 확보 (권장) |

**로컬 소스 수정본 사용 시 (선택)**

FreeCAD MCP를 커스텀/수정해서 사용하고 싶다면 아래와 같이 경로를 지정할 수 있습니다.

```json
{
  "mcpServers": {
    "freecad": {
      "type": "stdio",
      "command": "uvx",
      "args": ["--from", "C:/path/to/freecad-mcp", "freecad-mcp"],
      "timeout": 120000
    }
  }
}
```

### Step 6. 연결 확인

1. **FreeCAD를 먼저 실행** (GUI 모드 권장)
2. Opencode를 실행 (`opencode` CLI)
3. 대화에서 간단한 테스트 프롬프트를 입력

```text
FreeCAD에 빈 문서를 하나 만들고, Body를 생성한 뒤 "MCP 연결 테스트"라는 이름의 Sketch를 추가해줘.
```

4. Opencode가 `freecad` MCP 툴을 호출하고, FreeCAD에서 문서/오브젝트가 생성되면 연결이 정상입니다.

## 7. 권장 워크플로우 (엔지니어링 관점)

영상의 "Designs Like an Engineer(엔지니어처럼 설계)"를 실현하려면 **파라메트릭(Parametric) + 히스토리 기반(Feature History)** 설계를 고정하는 게 가장 중요합니다.

### 7.1 설계 원칙 (Best Practices)

| 원칙 | 권장 방식 | 이유 |
|---|---|---|
| **1. PartDesign 우선** | `Sketch → Pad/Pocket → Chamfer/Fillet` 순서 고정 | Feature 히스토리를 보존해 나중에 수정이 용이 (파라메트릭 설계의 핵심) |
| **2. Sketch는 Fully Constrained** | 모든 스케치는 **완전 구속(Fully Constrained)** 상태로 마무리 | 치수 변화 시 예기치 않은 변형 방지, 예측 가능한 설계 |
| **3. Boolean 남용 지양** | 직접 솔리드(Boolean Cut/Union/Fuse) 남발 대신 피처 기반 우선 | 히스토리 추적성/수정성 저하 방지, 안정성 향상 |
| **4. 파라메트릭 변수 활용** | FreeCAD `Spreadsheet` 또는 변수로 주요 치수 정의 | "재설계(What-if)"가 쉬워지고 일관성 유지 |
| **5. 단위/공차 명시** | `mm`, 중요 치수, 공차/반지름 허용치 명시 | 실무(제작/가공) 기준과의 정합성 확보 |
| **6. 측정 기반 검증** | 생성 후 치수/거리/면적/볼륨을 **실제로 측정**해 검증 요청 | LLM의 추론 오류를 줄이고 신뢰도 향상 |
| **7. 산출물 이원화** | `.FCStd`(히스토리 보존) + `.STEP`(호환성) 동시 Export | 설계 수정성(FCStd)과 CAD 호환성(STEP) 모두 확보 |

### 7.2 Opencode 작업 구조화 (TodoWrite)

Opencode의 가장 큰 강점은 **[TodoWrite](https://docs.opencode.ai/features/todo/)**를 통한 다단계 작업 관리입니다. 복잡한 3D 모델링은 반드시 작업을 세분화해 `in_progress`를 1개로 유지하는 걸 권장합니다.

**권장 Todo 템플릿 (일반적인 파트 모델링)**

```text
1. [pending] 요구사항 정리 및 설계 방향 수립 (치수/목적/제약 조건)
2. [pending] FreeCAD 문서 생성 및 Body/Part 구조 설정
3. [pending] 스케치(Sketch) 작성 및 구속(Constraints) 적용 (Fully Constrained)
4. [pending] 1차 피처 생성 (Pad/Extrude 등)
5. [pending] 추가 피처 (Pocket/Hole/Chamfer/Fillet/Round)
6. [pending] 형상 검증 (치수/대칭/간섭 체크, 뷰포트 확인)
7. [pending] 치수/볼륨 측정 기반 수치 검증
8. [pending] Export (.FCStd + .STEP) + 파일 존재/유효성 확인
9. [pending] 최종 검토 및 정리
```

Opencode는 작업 진행 중 이 Todo 리스트를 실시간으로 업데이트하므로, 대화가 길어져도 컨텍스트/방향성을 잃지 않고 일관된 흐름으로 작업을 완료할 수 있습니다.

### 7.3 GUI vs Headless 선택

| 모드 | 장점 | 단점 | 권장 용도 |
|---|---|---|---|
| **GUI (FreeCAD 실행)** | 뷰포트 스냅샷/시각 피드백 루프 최대 활용, 실시간 확인 가능 | GUI 리소스 소모 | **권장 (영상과 동일한 루프, 검증 품질 최고)** |
| **Headless (`--headless`)** | 백그라운드 자동화, CI/CD 가능 | 시각적 피드백/스냅샷 활용 제한, 형상 미관 검증 어려움 | 순수 치수/파라메트릭/볼륨 기반 자동화, 배치 처리 |

**권장**: 설계 품질/정확도 검증을 중시한다면 **항상 FreeCAD GUI를 먼저 실행한 상태**에서 Opencode를 사용하는 구성을 권장합니다.

### 7.4 안정성/신뢰성 팁

| 팁 | 적용 방법 |
|---|---|
| **FreeCAD 선실행** | Opencode 실행 **전** FreeCAD를 먼저 실행해 두세요. MCP 연결 타이밍이 가장 안정적입니다. |
| **단일 문서 원칙** | 한 번에 **하나의 FreeCAD 문서(Document)**만 작업하도록 지시하세요. 다중 문서 동시 제어는 상태 충돌 위험이 있습니다. |
| **에러 기반 수정 루프** | FreeCAD Python 실행 예외가 발생하면, **전체 재생성 대신 에러 메시지를 기반으로 수정**하도록 프롬프트에 명시하세요. |
| **측정값 우선 검증** | "뷰포트가 좋아 보인다"고 말하지 말고, **치수/거리/볼륨/바운딩박스 등을 실제로 측정해서 보고**하도록 요청하세요. |
| **점진적 증분** | 한 번에 너무 복잡한 형상보단 **작은 스텝으로 증분**하며 매 스텝마다 검증 후 다음 단계로 진행하세요. |
| **Export 검증** | `.FCStd/.STEP` Export 후 `Bash`로 **파일 존재 여부 + 파일 크기**까지 확인해 검증 루프를 닫으세요. |

## 8. 트러블슈팅

| 증상 | 원인 | 해결 방법 |
|---|---|---|
| **MCP 툴이 안 보임 / 호출 안 됨** | Opencode가 `mcp.json`을 로드하지 못함 | `.opencode/mcp.json` 경로/JSON 문법 확인, Opencode 재시작 |
| `uvx: command not found` | PATH 미갱신 | `uvx` 설치 후 **PowerShell 완전 종료 후 재실행** (Step 3 필수) |
| **FreeCAD에 MCP Workbench가 안 나타남** | Mod 폴더 복사 경로/버전 불일치 | `%APPDATA%\FreeCAD\1.1\Mod\FreeCAD-MCP/` 경로 확인, FreeCAD **완전 재시작** |
| **MCP 연결 실패 (연결 안 됨)** | FreeCAD가 실행되지 않은 상태에서 Opencode 시작 | **반드시 FreeCAD를 먼저 실행**한 뒤 Opencode 실행 |
| **명령 타임아웃** | 복잡한 Boolean/큰 피처 처리 | `mcp.json` `timeout`을 `180000`(3분) 이상으로 늘리기 |
| **Python 예외 반복** | Sketch 구속 미완성/잘못된 피처 순서 | **Fully Constrained Sketch** 유도, PartDesign 워크플로우(Sketch→Pad/Pocket) 고정 지시 |

## 9. 결론

**권장 구성:** `Opencode + uvx freecad-mcp (stdio) + FreeCAD GUI (사전 실행)`

- 영상에서 제시한 MCP 패턴은 **그대로 유지**하면서,
- Opencode 고유의 **TodoWrite(계획/진행 관리) + 로컬 도구(Bash/Grep/Read/Edit) + Git 연계(버전관리)**를 결합하면 단순 생성형 CAD를 넘어 **설계 → 검증 → 산출물 관리가 닫힌 엔지니어링 워크플로우**로 확장할 수 있습니다.
- 실무 적용 시 핵심 원칙은 **Sketcher+PartDesign 우선 (파라메트릭 히스토리 유지), Fully Constrained, 측정값 기반 검증, `.FCStd + .STEP` 동시 Export**입니다.

간단히 요약하면, **ChatGPT Desktop 대비 Opencode 조합이 실무형 CAD 자동화/버전관리 측면에서 훨씬 더 적합한 구성**입니다.
