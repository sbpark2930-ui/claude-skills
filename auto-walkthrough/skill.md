---
name: auto-walkthrough
description: 코드베이스 콜 그래프를 Execution Tracer·Failure Hunter·Proposal Writer·Interface Auditor 4에이전트가 자동으로 리뷰·토론하고, 합의된 수정안을 문서로 제시한 뒤 사용자 확인 후 코드를 실제로 수정하는 자동 리뷰-협의-수정 스킬
argument-hint: "[<파일경로>:<함수명>] | [<파일경로>] [--boundary <경계경로>] [--rounds N] [--sort severity]"
---

<Purpose>
Auto Walkthrough는 code-walkthrough와 동일한 콜 그래프 추적 방식을 사용하되, 사용자가 전혀 개입하지 않아도 된다. 네 가지 전문 에이전트가 각 함수를 독립적으로 리뷰한 후 토론을 통해 합의에 이른다. 모든 함수 토론이 완료되면 수정 합의 목록을 담은 문서를 사용자에게 제시하고, 사용자가 확인 후 "진행"을 승인하면 코드를 실제로 수정한다.

**4에이전트 역할 (출력 형식이 역할 경계를 강제한다):**
- **Execution Tracer (실행 추적):** 코드가 실행되는 순서를 ENTRY/STEPS/EXIT/SIDE_EFFECTS 형식으로 서술. 판단 없음, 실패 시나리오 없음, 대안 없음.
- **Failure Hunter (실패 탐색):** FAILURE_N {trigger/path/outcome/evidence/severity} 형식으로 실패 경로만 서술. 정상 동작 설명 없음, 수정 제안 없음.
- **Proposal Writer (제안 작성):** PROPOSAL_N {current/problem/replace/benefit} 형식 — 기존 코드 인용 필수. 실패 시나리오 없음, 동작 서술 없음.
- **Interface Auditor (인터페이스 감사):** CALL_SITE {caller/arg/expected/match} 형식으로 호출자-피호출자 계약만 검증. 토론 불참.

핵심 흐름:
1. 진입 함수(또는 파일)를 파악하고 Glob/Grep/Read로 콜 그래프를 추적한다. git blame/log로 각 함수의 변경 이력도 수집한다.
2. Figma MCP → Mermaid 코드블록 순으로 다이어그램을 생성한다.
3. 각 함수마다 4에이전트가 병렬 독립 리뷰 → 역할 경계 위반 감지 → 토론 → 합의를 진행한다 (코드 수정 없음). 단, Interface Auditor는 독립 리뷰에만 참여하고 토론에는 불참한다.
4. 전체 토론 완료 후 수정 합의 목록을 담은 문서를 사용자에게 제시한다.
5. 사용자가 확인하고 "진행"을 승인하면 코드를 실제로 수정하고 테스트를 실행한다.
</Purpose>

<Use_When>
- "auto walkthrough", "자동 워크스루", "자동으로 코드 분석해줘" 같은 요청이 들어올 때
- 여러 관점에서 코드를 자동 리뷰하고, 문제점과 대안을 문서화하고 싶을 때
- 수정 전 전체 토론 결과를 먼저 보고 내가 결정하고 싶을 때
- 오케스트레이션 프롬프트 같은 로직이 실제로 의도대로 동작하는지 검증하고 싶을 때
- 사용자가 자리를 비우는 동안 백그라운드에서 코드 분석을 돌리고, 돌아와서 결과를 검토하고 싶을 때

**기존 스킬들과의 차별점:**
- **code-walkthrough**: 사용자가 "다음"을 선택해야만 진행 (이해 전용, 수정 없음)
- **auto-walkthrough**: 4에이전트가 자동 토론 → 수정안 문서 제시 → 사용자 승인 후 수정 + 테스트 검증
- **code-audit-interview**: 이슈를 찾고 사용자가 항목별로 승인하며 수정 (하나씩 인터랙티브)
</Use_When>

<Do_Not_Use_When>
- 특정 함수에 대해 직접 질문하고 대화를 주고받고 싶을 때 — code-walkthrough를 사용하세요
- 항목별로 하나씩 검토하고 선택적으로 수정하고 싶을 때 — code-audit-interview를 사용하세요
- 특정 리팩토링 계획을 실행하고 싶을 때 — refactor-interview를 사용하세요
</Do_Not_Use_When>

<Execution_Policy>
- 콜 그래프는 반드시 Glob/Grep/Read로 실제 코드를 스캔하여 추적한다 (추측 금지)
- 외부 라이브러리(SQLAlchemy, httpx, redis 등) 함수 호출은 탐색 중단 + 다이어그램 점선 노드
- 순환 참조(A→B→A) 감지 시 경고 표시 후 더 이상 재귀하지 않는다
- **Phase 3 (토론):** Edit 도구 절대 사용 금지 — 합의해도 코드를 즉시 수정하지 않는다
- **Phase 5 (실행):** 사용자 승인 후에만 Edit 도구로 코드를 수정하고, 각 수정 직후 테스트를 실행한다
- 4에이전트(ET/FH/PW/IA)는 독립 리뷰 시 Agent 도구로 병렬 호출한다
- Interface Auditor는 독립 리뷰에만 참여하고, 토론 라운드에는 불참한다 (findings를 다른 에이전트 프롬프트에 포함)
- 토론 라운드는 ET/FH/PW 3에이전트를 순차 호출한다 (이전 에이전트 의견을 반영해야 하므로)
- 각 에이전트 응답 후 역할 경계 위반(금지 패턴)을 감지하여 위반 시 1회 재시도한다
- 합의 기준: 3/3 만장일치 — 1명이라도 미납득이면 N라운드까지 토론 계속
- 토론 중 입장 변경 시 이전 라운드에서 새로 제시된 근거를 코드 줄번호 또는 인용으로 명시해야 인정한다
- N라운드 후 만장일치 미달이면 수정 보류 + 문서에 대립 내용 기록
- 사용자 개입 없이 Phase 3까지 자동 완료 (AskUserQuestion — 진입점 파악 때만 허용)
- Phase 4에서 문서 제시 + 사용자 확인은 AskUserQuestion 사용 (하드 블락)
</Execution_Policy>

<Steps>

## Phase 1: 진입점 파악

**목적:** 워크스루 시작점이 되는 파일/함수를 결정하고 콜 그래프 초기 추적을 시작한다.

### Step 1-1: 입력 파싱

`{{ARGUMENTS}}`를 파싱한다:
- `<파일경로>:<함수명>` → 지정된 함수에서 시작
- `<파일경로>` → 파일의 주요 진입 함수를 AI가 탐지
- `--boundary <경계경로>` → 탐색 경계 지정 (기본: 프로젝트 전체 내부)
- `--rounds N` → 토론 최대 라운드 수 (기본: 3)
- `--sort severity` → Phase 4 수정 예약 목록을 severity 순으로 정렬 (기본: 콜 그래프 순서)
  - 정렬 우선순위: severe failures > proposals with high benefit > minor improvements
- 인수 없음 → Step 1-2로 이동

### Step 1-2: 진입 함수 탐지 (인수 없을 때)

Glob/Grep으로 코드베이스를 스캔하여 주요 진입 함수 후보를 제안한다.

인수가 없을 때에만 `AskUserQuestion`으로 진입 함수를 선택받는다.

### Step 1-3: 콜 그래프 추적 + 변경 이력 수집

Glob/Grep/Read로 진입 함수에서 시작하는 콜 그래프를 추적한다:

**탐색 범위:**
- 프로젝트 내부 함수: 탐색 계속
- 외부 라이브러리(SQLAlchemy, httpx, redis, fastapi, openai 등): 탐색 중단
- `--boundary` 지정 시: 해당 경로 내 함수까지만 탐색
- 기본 탐색 깊이: 5레벨, 최대 함수 수: 50개

**순환 참조 감지:**
- `visited_nodes` 집합으로 방문 함수를 추적
- 이미 방문한 함수 재방문 시 순환 경고 노드로 처리

**복잡도 측정:**
각 함수의 줄 수를 기록한다. 이후 Phase 3에서 라운드 수 조정과 미검증 태그에 사용된다.
```
node_complexity = {
  "lines": {함수 줄 수},
  "tier": "small(<50줄)|medium(50-150줄)|large(>150줄)"
}
```

**변경 이력 수집 (Historian 통합):**
각 함수에 대해 git blame/log를 실행하여 다음 메타데이터를 수집한다:
```
# Step 1: 최근 커밋 메시지 수집 (항상 실행)
git log --oneline -5 -- {file_path}
git blame -L {line_start},{line_end} {file_path}

# Step 2: 조건부 diff — 주의가 필요한 커밋만 코드 변경 내역 확인
TRIGGER_KEYWORDS = ["revert", "fix", "hotfix", "HACK", "workaround", "TODO", "FIXME", "rollback", "temporary", "broken"]

for commit in recent_commits:
  if any(keyword in commit.message.lower() for keyword in TRIGGER_KEYWORDS):
    git show {commit_hash} -- {file_path}   # 해당 커밋의 diff만 확인
```

**조건부 diff 근거:** revert/fix/hotfix/HACK 등 키워드가 포함된 커밋은 "이전에 시도했다가 문제가 생겨서 현재 방식으로 된 것"일 가능성이 높다. 실제 코드 변경 내역을 확인해야 에이전트가 동일한 실수를 반복 제안하지 않는다.

수집 결과를 노드 메타데이터에 포함:
```
node_metadata = {
  "function": "{함수명}",
  "file": "{file_path}:{line_start}-{line_end}",
  "lines": {줄 수},
  "complexity_tier": "small|medium|large",
  "recent_commits": ["커밋1 요약", "커밋2 요약", ...],
  "last_modified": "{날짜}",
  "author": "{마지막 수정자}",
  "has_todo": true|false,
  "related_issues": ["#123", ...],
  "caution_diffs": [
    {
      "commit": "{hash} {message}",
      "diff_summary": "{변경 요약}",
      "trigger_keyword": "revert|fix|..."
    }
  ]
}
```

**git 사용 불가 시:** .git 디렉토리가 없거나 git 명령 실패 시 변경 이력 필드를 null로 설정하고 진행한다.

추적 완료 후 노드 목록을 방문 순서(DFS)로 정렬한다.

---

## Phase 2: 다이어그램 생성

**목적:** Phase 1에서 추적한 콜 그래프를 시각화한다. Figma MCP → Mermaid 코드블록 순으로 폴백한다.

### Step 2-1: 노드 라벨 형식 (3줄 고정)

```
funcName()
file.py:L{시작줄번호}
한줄 역할 요약 (최대 20자)
```

| 노드 유형 | Mermaid 스타일 |
|---|---|
| 내부 함수 | `["funcName()\nfile.py:L10\n한줄 역할 요약"]` |
| 외부 라이브러리 | `["external_call()"]:::external` |
| 순환 참조 | `["funcName() ⚠️ 순환"]:::cycle` |

### Step 2-2: Figma MCP 우선 시도

**단계 1 — 가용 여부 확인:** `<available-deferred-tools>` 목록에서 `mcp__claude_ai_Figma__generate_diagram` 이름이 보이는지 확인한다.
- **보이면 → 단계 2로 진행 (deferred 상태여도 무조건 시도)**
- **안 보이면 → Mermaid 폴백**

**단계 2 — 로드 & 호출:** ToolSearch로 스키마를 로드한 뒤 즉시 호출한다.
```
ToolSearch("select:mcp__claude_ai_Figma__generate_diagram")
→ 성공: mcp__claude_ai_Figma__generate_diagram(name=..., mermaidSyntax=...) 호출
→ 실패: Mermaid 폴백
```

### Step 2-3: Mermaid 폴백

```
다이어그램 생성 완료 (Mermaid)

%%{init: {'theme': 'default'}}%%
flowchart LR
    classDef external stroke-dasharray: 5 5, fill:#f5f5f5, color:#888
    classDef cycle fill:#fff3cd, stroke:#ffc107, color:#856404
    ...

총 탐색 함수: {N}개 | 외부 스탑: {M}개 | 순환 참조: {K}개
```

---

## Phase 3: 4에이전트 리뷰-토론 루프 (수정 없음)

**목적:** 콜 그래프의 각 함수를 방문 순서대로 자동으로 처리하며, 4에이전트가 독립 리뷰 → 역할 경계 위반 감지 → 토론을 통해 "수정 필요/불필요"를 결정한다. **이 단계에서는 코드를 수정하지 않는다.**

### Step 3-1: 에이전트 정의

| 에이전트 | 관점 | 출력 형식 | 토론 참여 |
|---|---|---|---|
| **Execution Tracer (ET)** | 실행 추적 | ENTRY/STEPS/EXIT/SIDE_EFFECTS | O |
| **Failure Hunter (FH)** | 실패 탐색 | FAILURE_N {trigger/path/outcome/evidence/severity} | O |
| **Proposal Writer (PW)** | 제안 작성 | PROPOSAL_N {current/problem/replace/benefit} | O |
| **Interface Auditor (IA)** | 인터페이스 감사 | CALL_SITE {caller/arg/expected/match} | X (독립 리뷰만) |

**역할 경계 원칙:**
각 에이전트의 출력 형식 자체가 다른 역할의 내용이 들어올 공간을 구조적으로 차단한다. 언어 지시가 아닌 형식 강제로 역할을 분리한다.

- ET는 STEPS 필드에 "실행 순서"만 채운다. "실패 가능성"이나 "더 나은 방법"을 쓸 필드가 없다.
- FH는 FAILURE_N 블록에 "트리거-경로-결과-증거"만 채운다. 정상 동작 설명이나 수정 코드를 쓸 필드가 없다.
- PW는 PROPOSAL_N 블록에 "현재 코드 인용-문제-대체-효과"만 채운다. 실패 시나리오나 동작 서술을 쓸 필드가 없다.
- IA는 CALL_SITE 블록에 "호출자-인수-기대타입-일치여부"만 채운다. 내부 동작에 개입하지 않는다.

**Interface Auditor 역할 상세:**
Interface Auditor는 현재 함수를 단독으로 보지 않고, 콜 그래프에서 이 함수를 호출하는 함수(callers)와 이 함수가 호출하는 함수(callees)의 시그니처/반환값/타입을 함께 검토한다. 토론에는 참여하지 않으며, findings가 다른 3에이전트의 토론 프롬프트에 컨텍스트로 포함된다.

### Step 3-1b: 테스트 커버리지 사전 확인

각 함수에 대해 테스트 커버리지를 확인한다:

```
# 테스트 파일에서 함수명/클래스명 참조 확인
Grep("{function_name}", glob="**/test_*|**/*_test*|**/*.test.*|**/*.spec.*")
```

- 참조하는 테스트 파일이 발견되면: `has_tests: true`
- 발견되지 않으면: `has_tests: false` → 노드에 `[⚠️ 테스트 없음]` 태그 추가

진행 표시에 테스트 커버리지 표시:
```
[{현재}/{전체}] {함수명}() 리뷰 중... ({complexity_tier}, {lines}줄) [⚠️ 테스트 없음]
```

### Step 3-2: 병렬 독립 리뷰

함수마다 4에이전트를 **병렬로** Agent 도구로 호출한다.

**복잡도 기반 라운드 수 조정:**
```
node_complexity.tier == "small"  → MAX_ROUNDS = min(N, 2)  # 소형 함수: 토론 단축
node_complexity.tier == "medium" → MAX_ROUNDS = N           # 중형 함수: 기본값
node_complexity.tier == "large"  → MAX_ROUNDS = N + 1       # 대형 함수: 토론 연장
```

**누적 발견사항 전달:**
```
accumulated_findings = []  # 전체 리뷰 루프에 걸쳐 유지

# 각 함수 리뷰 완료 시 업데이트:
accumulated_findings.append({
  "function": "{함수명}",
  "key_findings": ["핵심 발견1", "핵심 발견2"],
  "verdict": "수정 예약|보류|불필요",
  "patterns": ["발견된 반복 패턴"],
  "interface_audit_findings": ["인터페이스 계약 이슈 (있을 경우)"]
})
```

진행 표시 출력:
```
[{현재}/{전체}] {함수명}() 리뷰 중... ({complexity_tier}, {lines}줄)
  파일: {file_path}:{line_start}
  변경 이력: {last_modified} by {author} | {recent_commits[0]}
  누적 발견 패턴: {accumulated_findings에서 추출한 반복 패턴 목록}
  → ET·FH·PW·IA 병렬 리뷰 시작
```

---

**[Execution Tracer 전용 프롬프트]**

```
당신은 Execution Tracer입니다. 이 함수가 실행될 때 일어나는 일을 순서대로 추적합니다.

⚠️ 출력 형식이 역할 경계를 강제합니다. 아래 JSON 형식 외의 내용은 쓰지 마세요.
- "실패", "오류", "fail", "break"가 포함된 문장 금지 (→ Failure Hunter 영역)
- "더 나은", "대신", "replace", "개선", "should" 가 포함된 문장 금지 (→ Proposal Writer 영역)
- 판단이나 평가 금지. 오직 "무슨 일이 일어나는가"만 서술하세요.

반환 JSON:
{
  "perspective": "ExecutionTracer",
  "verdict": "납득|미납득",
  "score": 0.0~1.0,
  "execution_trace": {
    "entry": "제어가 이 함수에 진입하는 조건/호출 컨텍스트",
    "steps": [
      "1. [주어] [동사] [목적어] — L{줄번호}",
      "2. ...",
      ...
    ],
    "exit": "반환값 또는 예외 발생 경로 (분기 있으면 모두 나열)",
    "side_effects": ["외부 상태 변경 목록 (없으면 빈 배열)"]
  },
  "intent_gap": "의도와 실제 동작 사이의 괴리 (없으면 null) — 줄 번호 포함",
  "modification_needed": true|false,
  "modification_proposal": "수정 방향 한 줄 (없으면 null)",
  "reasoning": "납득/미납득 판단 근거"
}

score: 0.0(의도와 동작이 크게 다름)~1.0(의도와 동작이 완전히 일치)
verdict '납득': 코드가 의도한 동작을 정확히 수행하고 있음
verdict '미납득': 의도와 실제 동작 사이에 명확한 괴리가 있음

---
함수: {function_name}
파일: {file_path}:{line_start}-{line_end}
코드: {실제 코드 전체}

[변경 이력]
{node_metadata}

[누적 발견사항]
{accumulated_findings}
```

---

**[Failure Hunter 전용 프롬프트]**

```
당신은 Failure Hunter입니다. 이 함수가 실패하거나 예상과 다르게 동작하는 경로를 탐색합니다.

⚠️ 출력 형식이 역할 경계를 강제합니다. 아래 JSON 형식 외의 내용은 쓰지 마세요.
- "정상 동작", "반환합니다", "호출합니다" 등 동작 서술 금지 (→ Execution Tracer 영역)
- "대신", "replace", "개선", "더 나은" 이 포함된 수정 제안 금지 (→ Proposal Writer 영역)
- FAILURE 블록에는 trigger → path → outcome → evidence 4요소가 반드시 있어야 합니다.
  trigger 없이 "~할 수 있다" 식의 막연한 추측은 failures 배열에 넣지 마세요.

반환 JSON:
{
  "perspective": "FailureHunter",
  "verdict": "납득|미납득",
  "score": 0.0~1.0,
  "failures": [
    {
      "id": "F1",
      "trigger": "이 구체적 조건/입력값이 충족될 때",
      "path": "L{줄}: 어떤 코드 경로를 따라",
      "outcome": "무엇이 어떻게 깨지는가 (예외 타입, 잘못된 상태 등)",
      "evidence": "\"코드 직접 인용\" — 줄 번호 포함",
      "severity": "minor|severe"
    }
  ],
  "security_checklist": {
    "input_validation": "OK|이슈 — {상세}|해당없음",
    "auth": "OK|이슈 — {상세}|해당없음",
    "secrets": "OK|이슈 — {상세}|해당없음",
    "race_condition": "OK|이슈 — {상세}|해당없음",
    "resource_leak": "OK|이슈 — {상세}|해당없음",
    "infinite_loop": "OK|이슈 — {상세}|해당없음",
    "memory": "OK|이슈 — {상세}|해당없음",
    "timeout": "OK|이슈 — {상세}|해당없음"
  },
  "modification_needed": true|false,
  "modification_proposal": "수정 방향 한 줄 (없으면 null)",
  "reasoning": "납득/미납득 판단 근거"
}

score: 0.0(심각한 실패 경로 다수)~1.0(실패 경로 없음)
verdict '납득': 심각한 실패 시나리오 없음 (minor 이슈는 납득 가능)
verdict '미납득': severe 실패 시나리오 1개 이상 또는 보안 체크리스트 이슈 존재

---
함수: {function_name}
파일: {file_path}:{line_start}-{line_end}
코드: {실제 코드 전체}

[변경 이력]
{node_metadata}

[누적 발견사항]
{accumulated_findings}
```

---

**[Proposal Writer 전용 프롬프트]**

```
당신은 Proposal Writer입니다. 현재 코드를 더 명확하고 안전하고 간결하게 달성하는 대안을 제안합니다.

⚠️ 출력 형식이 역할 경계를 강제합니다. 아래 JSON 형식 외의 내용은 쓰지 마세요.
- "실패 시나리오", "trigger:", "outcome:", "경쟁 조건", "타임아웃 시" 등 실패 경로 서술 금지 (→ Failure Hunter 영역)
- "반환합니다", "호출합니다" 등 동작 설명 금지 (→ Execution Tracer 영역)
- PROPOSAL 블록의 current 필드에는 반드시 실제 코드 스니펫을 직접 인용해야 합니다.
  기존 코드를 인용하지 않은 제안은 proposals 배열에 넣지 마세요.

반환 JSON:
{
  "perspective": "ProposalWriter",
  "verdict": "납득|미납득",
  "score": 0.0~1.0,
  "proposals": [
    {
      "id": "P1",
      "current": "\"현재 코드 스니펫 직접 인용\" — L{줄번호}",
      "problem": "측정 가능한 문제 한 줄 (가독성 저하 / 버그 위험 / 불필요한 복잡도 등)",
      "replace": "대체 코드 (실행 가능한 수준)",
      "benefit": "개선 효과 — 정량적으로 (예: 중복 제거 3줄 / 타임아웃 방어 추가 / 가독성 향상)"
    }
  ],
  "modification_needed": true|false,
  "reasoning": "납득/미납득 판단 근거 — 현재 구현이 합리적이면 납득"
}

score: 0.0(개선 여지 매우 큼)~1.0(현재 구현이 최선)
verdict '납득': 현재 구현이 합리적 수준 (완벽하지 않아도 됨)
verdict '미납득': 더 명확하고 안전한 대안이 있음

---
함수: {function_name}
파일: {file_path}:{line_start}-{line_end}
코드: {실제 코드 전체}

[변경 이력]
{node_metadata}

[누적 발견사항]
{accumulated_findings}
```

---

**[Interface Auditor 전용 프롬프트]**

```
당신은 Interface Auditor입니다. 이 함수의 호출자(callers)와 피호출자(callees)의 계약을 검증합니다.

⚠️ 출력 형식이 역할 경계를 강제합니다. 아래 JSON 형식 외의 내용은 쓰지 마세요.
- 내부 동작 서술, 실패 시나리오, 수정 제안 금지.
- 오직 "호출자가 전달하는 것 vs 이 함수가 기대하는 것" 불일치만 보고하세요.
- Interface Auditor는 토론에 참여하지 않습니다. findings만 반환하세요.

반환 JSON:
{
  "perspective": "InterfaceAuditor",
  "verdict": "납득|미납득",
  "score": 0.0~1.0,
  "call_sites": [
    {
      "caller": "호출_함수명:L{줄번호}",
      "arg": "전달하는 값/타입",
      "expected": "이 함수가 기대하는 타입/구조",
      "match": "OK | MISMATCH — {불일치 상세}"
    }
  ],
  "interface_checks": {
    "param_type_match": "OK|불일치 — {상세}",
    "return_type_match": "OK|불일치 — {상세}",
    "null_propagation": "OK|위험 — {상세}",
    "exception_handling": "OK|미처리 — {상세}",
    "data_structure_match": "OK|불일치 — {상세}",
    "side_effects": "OK|주의 — {상세}"
  },
  "modification_needed": true|false,
  "reasoning": "납득/미납득 판단 근거"
}

score: 0.0(인터페이스 계약 다수 불일치)~1.0(모든 계약 일치)

---
함수: {function_name}
파일: {file_path}:{line_start}-{line_end}
코드: {실제 코드 전체}

[호출자(callers) — 시그니처 + 호출 코드 스니펫]
{callers 정보}

[피호출자(callees) — 시그니처 + 반환 타입]
{callees 정보}

[변경 이력]
{node_metadata}
```

---

### Step 3-2b: 역할 경계 위반 감지 (post-processing)

각 에이전트 응답을 받은 즉시 금지 패턴을 체크한다. 별도 LLM 호출 없이 패턴 매칭으로 수행한다.

```
FORBIDDEN_PATTERNS = {
  "ExecutionTracer": [
    r"실패할\s*수", r"오류가\s*발생", r"fail", r"break", r"더\s*나은", r"대신", r"should", r"개선"
  ],
  "FailureHunter": [
    r"정상\s*동작", r"반환합니다", r"호출합니다", r"대신\s+", r"replace", r"개선", r"더\s*나은"
  ],
  "ProposalWriter": [
    r"실패\s*시나리오", r"trigger\s*:", r"outcome\s*:", r"경쟁\s*조건", r"타임아웃\s*시"
  ]
}

for agent, response in agent_responses.items():
  violations = [p for p in FORBIDDEN_PATTERNS[agent] if re.search(p, str(response))]
  if violations:
    # 1회 재시도: 위반 패턴을 명시하여 해당 내용 제거 요청
    retry_response = call_agent(agent, prompt + f"\n\n이전 응답에서 [{', '.join(violations)}] 패턴이 감지됐습니다. 해당 내용을 제거하고 출력 형식만 채워주세요.")
    if retry_response has no violations:
      use retry_response
    else:
      # 재시도도 위반: 위반 마킹 후 진행 (차단하지 않음)
      mark response with "[⚠️ 역할 경계 위반 감지 — 재시도 후에도 잔존]"
```

### Step 3-3: 리뷰 결과 집계

```
리뷰 결과: {함수명}() ({complexity_tier}, {lines}줄)
  ET (Execution Tracer): {납득|미납득} ({score:.2f}) — intent_gap: {요약 or "없음"}
  FH (Failure Hunter):   {납득|미납득} ({score:.2f}) — failures: {F개} (severe: {S개})
                          보안 체크: {이슈 항목만 표시}
  PW (Proposal Writer):  {납득|미납득} ({score:.2f}) — proposals: {P개}
  IA (Interface Auditor):{납득|미납득} ({score:.2f}) — mismatches: {M개}
  평균 점수: {avg:.2f} (ET/FH/PW 3에이전트 기준)
  경계 위반: {위반 감지된 에이전트 목록 or "없음"}
```

**Interface Auditor findings 전달:** IA의 verdict가 "미납득"이면 interface_checks 결과를 토론 라운드의 모든 에이전트 프롬프트에 포함한다. "납득"이면 요약만 포함.

**합의 판정 기준:** IA는 토론에 불참하므로 합의는 ET/FH/PW 3에이전트 기준으로 판정한다. 단, IA가 "미납득"이면 해당 함수는 합의와 무관하게 수정 예약에 IA findings를 필수 포함한다.

**분기:**
- **1명 이상 미납득 → Step 3-4** (토론 시작)
- **3/3 모두 납득 + 평균 점수 ≥ 0.75 → Step 3-7** (수정 불필요 판정)
- **3/3 모두 납득 + 평균 점수 < 0.75 → Step 3-4** (점수 기반 토론 진입)
  - 토론 프롬프트에 추가: "전원 납득이지만 평균 점수가 {avg:.2f}로 낮습니다. 개선 여지가 있는지 재검토하세요."

### Step 3-4: 토론 라운드

**토론 순서 로테이션:** 매 라운드마다 에이전트 발언 순서를 로테이션하여 특정 에이전트가 항상 마지막(유리한 위치)에서 발언하는 편향을 방지한다.

```
ROTATION_ORDERS = [
  ["ExecutionTracer", "FailureHunter", "ProposalWriter"],    # 라운드 1
  ["FailureHunter", "ProposalWriter", "ExecutionTracer"],    # 라운드 2
  ["ProposalWriter", "ExecutionTracer", "FailureHunter"],    # 라운드 3
]
```

**토론 라운드 에이전트 호출 시 추가 컨텍스트:**
```
이것은 토론 라운드 {round}/{MAX_ROUNDS}입니다.

이전 라운드 의견:
  ET: {verdict} ({score:.2f}) — {reasoning 요약}
  FH: {verdict} ({score:.2f}) — {failures 요약}
  PW: {verdict} ({score:.2f}) — {proposals 요약}

[Interface Auditor 독립 리뷰 결과 — 토론 불참, findings만 참고]
  IA: {verdict} ({score:.2f}) — {interface_checks 요약}

다른 에이전트들의 의견을 검토한 뒤 입장을 업데이트하세요.
Interface Auditor가 발견한 인터페이스 이슈도 고려하세요.

⚠️ 입장을 바꾸는 경우 반드시 이전 라운드에서 새로 제시된 근거를 명시해야 합니다.
   "다른 에이전트 의견에 동의한다"는 이유만으로는 입장 변경이 인정되지 않습니다.
   변경 근거에 코드 줄 번호 또는 직접 인용이 포함되어야 합니다.

반환 JSON에 다음 필드를 추가하세요:
  "position_changed": true|false,
  "changed_due_to": {              // position_changed=true일 때만
    "agent": "어떤 에이전트의 의견으로",
    "evidence": "구체적 근거 — 코드 줄번호 또는 인용 포함 (막연한 동의 불인정)"
  }
```

**sycophancy 감지:**
토론 라운드 집계 시, `position_changed: true`인데 `changed_due_to.evidence`가 줄 번호나 코드 인용 없이 막연하면:
```
→ 해당 입장 변경을 [근거 불충분] 으로 마킹
→ 이전 라운드의 verdict를 그대로 유지
→ 합의 계산에 이전 verdict 사용
```

**토론 진행 표시:**
```
  [토론 라운드 1/{MAX_ROUNDS}] (순서: ET → FH → PW)
    ET: 납득 (입장 유지)
    FH: 납득 → 미납득 변경 (근거: L103 asyncio.gather에서 return_exceptions=False — 예외 전파 확인)
    PW: 미납득 (입장 유지)
    현재 합의: 1/3 | 평균 점수: {avg:.2f}
```

### Step 3-5: 합의 달성 → 수정 예약 (코드 미수정)

3/3 만장일치 달성 시 수정을 "예약"한다. **Edit 도구를 사용하지 않는다.**

```
합의 달성 (3/3 만장일치) — 수정 예약 (Phase 5에서 실행)
  ET 발견: {intent_gap 요약}
  FH 발견: {failures 요약}
  PW 제안: {proposals 요약}
  → 통합 수정안을 문서에 기록 (실제 수정은 사용자 승인 후)
```

**진행 표시:**
```
[{번호}/{전체}] {함수명}() ━━━━━━━━━━━━━━━━━━━━ 완료 (수정 예약)
  합의: {토론 {N}라운드 만장일치 | 초기 리뷰 만장일치}
  평균 점수: {avg:.2f}
  수정 요약: {한 줄 수정 내용}
```

### Step 3-6: 합의 실패 → 수정 보류

N라운드 후 만장일치 미달:

```
[{번호}/{전체}] {함수명}() ━━━━━━━━━━━━━━━━━━━━ 완료 (수정 보류)
  {N}라운드 토론 후 {K}/3 합의 — 만장일치 미달
  미납득: {에이전트명} ({근거 불충분 마킹 여부})
  → 문서에 대립 내용 기록
```

### Step 3-7: 수정 불필요

3/3 모두 초기 납득 + 평균 점수 ≥ 0.75:

```
[{번호}/{전체}] {함수명}() ━━━━━━━━━━━━━━━━━━━━ 완료 (수정 불필요)
  ET: 납득 ({score:.2f})
  FH: 납득 ({score:.2f})
  PW: 납득 ({score:.2f})
  {complexity_tier == "large" ?
    "[⚠️ 미검증 가능성 — 복잡 함수({lines}줄): 에이전트가 모든 실행 경로를 탐색하지 못했을 수 있음]"
  }
```

**미검증 태그 기준:** 함수 줄 수 > 150이면 수정 불필요 판정이더라도 미검증 태그를 표시한다. 높은 납득 점수가 완벽한 코드를 의미하지 않을 수 있음을 투명하게 표시한다.

### Step 3-8: 전체 진행률

각 함수 완료 후 진행률을 출력한다:
```
전체 진행: ████████░░ {완료}/{전체} ({%}%) | 수정 예약: {예약_수}개 | 보류: {보류_수}개
```

---

## Phase 4: 문서 제시 및 사용자 확인 (하드 블락)

**목적:** 모든 함수 토론이 완료되면 수정 합의 목록을 담은 문서를 저장하고, 사용자에게 제시한 뒤 승인을 기다린다.

### Step 4-1: 문서 저장

두 파일을 Write 도구로 저장한다.

**`.omc/walkthroughs/auto-{slug}-summary.md` (사용자 요약 보고서):**

```markdown
# auto-walkthrough 요약 보고서: {slug}

## 메타데이터
- 생성 일시: {date}
- 진입점: {file}:{function}
- 방문 함수: {total}개 | 수정 예약: {예약}개 | 수정 보류: {보류}개 | 수정 불필요: {불필요}개
- 미검증 태그: {미검증_수}개 (복잡 함수 수정 불필요 판정)

## 콜 그래프 다이어그램

[Mermaid 코드블록]

## 수정 예약 목록 (사용자 승인 대기)

### 1. {function_name}() — {file}:{line_range} ({lines}줄)

**합의 경위:** {초기 납득 | 토론 N라운드 만장일치}

| 에이전트 | 발견사항 | 수정 방향 |
|---|---|---|
| ET | intent_gap: {요약} | {proposal} |
| FH | failures: {F개} / severe: {S개} | {proposal} |
| PW | proposals: {P개} | {replace 요약} |
| IA | mismatches: {M개} | {proposal} |

**통합 수정안:**
{통합된 수정 내용 — 실제 코드 형태 우선, 의사 코드 폴백}

---

## 수정 보류 목록 (만장일치 미달)

### 1. {function_name}() — {file}:{line_range}

**합의 현황:** {K}/3 — {미납득 에이전트}가 납득 못 함

**대립 내용:**
- ET: {verdict} — {reasoning}
- FH: {verdict} — {reasoning}
- PW: {verdict} — {reasoning}
{근거 불충분 마킹이 있을 경우: "⚠️ 근거 불충분 마킹: {에이전트} 입장 변경이 인정되지 않음"}

---

## 토론 압축 뷰

각 함수의 토론 결과를 "합의 포인트"와 "대립 포인트"로 압축한다.

| 함수 | 합의 포인트 | 대립 포인트 |
|------|-------------|-------------|
| {func()} | {전원 동의한 핵심 발견 요약} | {의견이 갈린 쟁점 요약 or "없음"} |

---

## 수정 불필요 목록

| 함수 | 파일 | 평균 점수 | 비고 |
|---|---|---|---|
| {func()} | {file:L#} | {avg:.2f} | {미검증 태그 or "-"} |

---

## 함수별 전체 결론

| 함수 | 파일 | 결론 | 평균 점수 | 복잡도 |
|---|---|---|---|---|
| {func()} | {file:L#} | 수정 예약 / 보류 / 불필요 | {score:.2f} | {lines}줄 |
```

**`.omc/walkthroughs/auto-{slug}-review.md` (리뷰 로그 — 전체 토론 기록):**

```markdown
# auto-walkthrough 리뷰 로그: {slug}

---

## 1. {function_name}() — {file}:{line_range}

### 독립 리뷰

**ET (Execution Tracer):** {verdict} ({score:.2f})
  entry: {요약} | steps: {N}개 | intent_gap: {요약 or "없음"}

**FH (Failure Hunter):** {verdict} ({score:.2f})
  failures: {N}개 | severe: {S}개 | security_issues: {이슈 항목}

**PW (Proposal Writer):** {verdict} ({score:.2f})
  proposals: {N}개

**IA (Interface Auditor):** {verdict} ({score:.2f})
  mismatches: {N}개 | {interface_checks 불일치 항목}

역할 경계 위반: {위반 감지 결과}

### 토론 라운드 1 (있을 경우)

  순서: {order}
  ET: {verdict} ({입장 변경 여부} | {changed_due_to 요약 or "입장 유지"})
  FH: {verdict} ({입장 변경 여부} | ...)
  PW: {verdict} ({입장 변경 여부} | ...)
  근거 불충분 마킹: {마킹된 에이전트 or "없음"}
  합의: {K}/3

### 최종 결론: {수정 예약|수정 보류|수정 불필요}
{미검증 태그 있을 경우 명시}
```

### Step 4-2: AskUserQuestion으로 사용자 확인 (하드 블락)

```
auto-walkthrough 분석 완료 — 사용자 확인 필요

방문한 함수: {total}개
  - 수정 예약 (에이전트 만장일치): {예약}개
  - 수정 보류 (만장일치 미달): {보류}개
  - 수정 불필요: {불필요}개 (⚠️ 미검증: {미검증_수}개 포함)

수정 예약된 함수:
  {함수명} — {수정 요약}

자세한 내용은 .omc/walkthroughs/auto-{slug}-summary.md를 확인하세요.
수정을 진행할까요?
```

**선택지:**
1. **"전체 수정 진행"** → Phase 5로 이동 (모든 예약 수정 실행 + 테스트 검증)
2. **"선택적 수정: {번호}"** → 지정된 항목만 수정 (예: "선택적 수정: 1,3,5")
   - 사용자가 번호를 콤마로 구분하여 승인할 항목만 선택
   - 미선택 항목은 "사용자 보류"로 기록
3. **"수정 없이 문서만 유지"** → 코드 수정 없이 종료 (문서는 저장됨)

**테스트 없는 함수 수정 시 안내:**
Phase 5에서 `has_tests: false`인 함수를 수정할 때:
```
⚠️ {func()}에 대한 테스트가 없습니다. 수정 후 회귀를 감지할 수 없습니다.
테스트를 먼저 작성하시겠습니까?
  1. 테스트 먼저 작성 후 수정 진행
  2. 테스트 없이 수정 진행
```

---

## Phase 5: 수정 실행 + 테스트 검증

**목적:** 사용자가 "수정 진행"을 승인하면, 수정 예약된 모든 함수에 Edit 도구로 수정을 적용하고, 각 수정 직후 테스트를 실행하여 검증한다.

### Step 5-0: 테스트 명령 탐지

수정 전에 프로젝트의 테스트 실행 명령을 탐지한다:
```
Glob("**/pytest.ini|**/pyproject.toml|**/setup.cfg") → Python: pytest 명령 사용
Glob("**/package.json") → JavaScript: npm test / vitest 명령 확인
명령 탐지 실패 → 테스트 스킵, 수정 완료 후 사용자에게 수동 테스트 권고
```

탐지된 테스트 명령:
```
test_command = {탐지된 명령 or null}
test_scope = {수정 파일과 연관된 테스트 경로 — Grep으로 추적}
```

### Step 5-1: 수정 실행 루프 (함수별)

수정 예약된 함수를 순서대로 Edit 도구로 수정한다. **각 수정 후 즉시 테스트를 실행한다.**

```
수정 실행 중...

[1/{예약_수}] {function_name}() — {file}:{line_range}
  통합 수정안 적용...
  [Edit 도구 실행]
  수정 완료 → 테스트 실행: {test_command} {test_scope}

  ✅ 테스트 통과 ({N}개) → 다음 함수로 진행
  OR
  ❌ 테스트 실패 ({N}개 실패) → 재수정 시도
     - FH failures 중 수정 방향과 충돌하는 항목 재분석
     - 재수정 후 테스트 재실행
     - 재수정도 실패 → Edit 롤백 (git checkout {file} or 수동 복원)
       해당 함수를 "수정 실패 — 롤백됨"으로 문서에 기록 후 다음 함수로 진행

[2/{예약_수}] ...
```

**테스트 명령 없는 경우:** 수정만 적용하고 테스트 단계를 스킵한다. 완료 보고에 "테스트 미실행 — 수동 확인 권고" 표시.

### Step 5-2: 수정 완료 보고

모든 수정이 완료되면:

```
수정 완료

수정 성공: {수정_수}개
  {파일명} → {함수명}: {수정 요약} | 테스트: {통과_수}/{전체_수}

수정 실패 (롤백됨): {실패_수}개
  {함수명}: {실패 이유 한 줄}

수정 보류 (만장일치 미달): {보류_수}개
  {함수명}: {미납득 에이전트} — {이유}

저장된 파일:
  .omc/walkthroughs/auto-{slug}-summary.md (요약 보고서)
  .omc/walkthroughs/auto-{slug}-review.md (리뷰 로그)
```

</Steps>

<Tool_Usage>
- **Agent (병렬)**: 4에이전트 독립 리뷰 — ET/FH/PW/IA를 동시에 병렬 호출 (`subagent_type: "executor"`)
- **Agent (순차)**: 토론 라운드 — ET/FH/PW 3에이전트가 이전 에이전트 의견을 보고 입장 업데이트 (반드시 순차, IA는 불참)
- **Edit**: **Phase 5에서만** 사용 — 사용자 승인 후 수정 실행. Phase 3에서는 절대 사용 금지.
- **Bash (역할 위반 감지)**: 정규식 패턴 매칭으로 각 에이전트 응답 검사 (별도 LLM 호출 불필요)
- **Bash (테스트)**: Phase 5에서 각 Edit 직후 테스트 실행 (`pytest`, `vitest` 등)
- **Glob**: 코드베이스 파일 구조 탐색, 진입점 후보 탐색, 테스트 명령 탐지
- **Grep**: 함수 정의 위치 탐색, 함수 호출처 확인, IA용 caller/callee 시그니처 수집
- **Read**: 각 함수의 실제 코드 전체 읽기 (에이전트 프롬프트에 코드 전체 포함)
- **Bash**: `git log`, `git blame` 실행 — Phase 1에서 각 함수의 변경 이력 수집 (git 불가 시 스킵)
- **Write**: 요약 보고서/리뷰 로그 저장
- **ToolSearch**: Figma MCP 도구 스키마 로드
- **mcp__claude_ai_Figma__generate_diagram**: Figma FigJam flowchart 생성
- **AskUserQuestion**: (1) 진입점 파악 때만 (인수 없을 때), (2) Phase 4 사용자 확인 (하드 블락)

**병렬 vs 순차 규칙:**
- 독립 리뷰 단계: ET/FH/PW/IA **병렬** 호출 (4에이전트 동시)
- 토론 라운드: ET/FH/PW **순차** 호출 (라운드별 로테이션 순서 적용, IA 불참)
  - 라운드 1: ET → FH → PW
  - 라운드 2: FH → PW → ET
  - 라운드 3: PW → ET → FH

**에이전트 납득 기준:**
- ET: 코드가 의도한 동작을 정확히 수행하면 납득 (intent_gap이 없으면 납득)
- FH: severe 실패 시나리오 없고 보안 체크리스트 이슈 없으면 납득 (minor 이슈는 납득 가능)
- PW: 현재 구현이 합리적 수준이면 납득 (완벽하지 않아도 됨)
- IA: 모든 call_site에서 계약이 일치하면 납득

**합의 + 점수 기준:**
- ET/FH/PW 3/3 납득 + 평균 점수 ≥ 0.75 → 수정 불필요 (대형 함수면 미검증 태그 추가)
- 3/3 납득 + 평균 점수 < 0.75 → 점수 기반 토론 진입
- 1명 이상 미납득 → 토론 진입
- IA 미납득: 합의 판정에는 불포함, 단 수정 예약 시 interface_checks 필수 반영

**sycophancy 방지:**
- position_changed=true이면 changed_due_to.evidence에 코드 줄번호 또는 인용이 반드시 있어야 함
- 막연한 동의("다른 에이전트 의견에 동의함")는 [근거 불충분]으로 마킹하고 이전 verdict 유지

**역할 경계 위반 처리:**
- 위반 감지 즉시 1회 재시도 (위반 패턴 명시)
- 재시도도 위반이면 [⚠️ 역할 경계 위반 감지] 마킹 후 진행 (차단하지 않음)
</Tool_Usage>

<Examples>

<Good>
Execution Tracer 출력 — 올바른 형식:
```json
{
  "perspective": "ExecutionTracer",
  "verdict": "미납득",
  "score": 0.55,
  "execution_trace": {
    "entry": "run_match()에서 각 턴 종료 후 호출",
    "steps": [
      "1. REVIEW_SYSTEM_PROMPT + 발언 텍스트로 LLM 호출 — L88",
      "2. 응답을 json.loads()로 파싱 — L95",
      "3. violations 리스트에서 penalty 합산 — L102",
      "4. is_blocked 여부 결정 후 반환 — L108"
    ],
    "exit": "dict {penalties, penalty_total, review_result, is_blocked} 반환 / TimeoutError 발생 가능",
    "side_effects": []
  },
  "intent_gap": "L95: json.loads()가 JSON 외 응답(마크다운 코드블록 등)을 받으면 파싱 실패 — REVIEW_SYSTEM_PROMPT에 JSON 전용 응답을 명시하지 않음",
  "modification_needed": true,
  "modification_proposal": "REVIEW_SYSTEM_PROMPT에 'JSON만 출력하라' 명시 또는 L95에 json 파싱 방어 코드 추가",
  "reasoning": "프롬프트가 JSON 응답을 강제하지 않는데 파싱 코드는 JSON을 기대함 — 의도와 실제 동작 불일치"
}
```
Why good: STEPS 형식으로 실행 순서만 서술. "실패 가능성"이나 "대신" 같은 FH/PW 영역 표현 없음. intent_gap에 줄 번호 포함.
</Good>

<Good>
Failure Hunter 출력 — 올바른 형식:
```json
{
  "perspective": "FailureHunter",
  "verdict": "미납득",
  "score": 0.35,
  "failures": [
    {
      "id": "F1",
      "trigger": "LLM이 마크다운 코드블록(```json ... ```)으로 응답할 때",
      "path": "L95: json.loads(response) → JSONDecodeError",
      "outcome": "예외가 호출자까지 전파되어 해당 턴 전체가 실패",
      "evidence": "\"result = json.loads(self._client.generate(...))\" — L95",
      "severity": "severe"
    },
    {
      "id": "F2",
      "trigger": "네트워크 타임아웃으로 generate() 10초 이상 소요 시",
      "path": "L88: await self._client.generate() 무한 대기",
      "outcome": "전체 턴 루프 블로킹 — asyncio.gather 상대방 에이전트도 차단",
      "evidence": "\"await self._client.generate(prompt=user_content\" — L88, 타임아웃 인수 없음",
      "severity": "severe"
    }
  ],
  "security_checklist": {
    "input_validation": "OK",
    "auth": "해당없음",
    "secrets": "OK",
    "race_condition": "해당없음",
    "resource_leak": "해당없음",
    "infinite_loop": "해당없음",
    "memory": "해당없음",
    "timeout": "이슈 — L88 generate() 타임아웃 없음"
  },
  "modification_needed": true,
  "modification_proposal": "L95 json.loads try/except 추가 + L88 generate() timeout 파라미터 설정",
  "reasoning": "severe 2건 — 프로덕션에서 재현 가능한 실패 경로"
}
```
Why good: FAILURE_N 블록마다 trigger/path/outcome/evidence 4요소 모두 포함. 줄 번호와 코드 직접 인용. 동작 서술이나 개선 제안 없음.
</Good>

<Good>
Proposal Writer 출력 — 올바른 형식:
```json
{
  "perspective": "ProposalWriter",
  "verdict": "미납득",
  "score": 0.5,
  "proposals": [
    {
      "id": "P1",
      "current": "\"result = json.loads(self._client.generate(prompt=user_content))\" — L95",
      "problem": "LLM 응답이 JSON이 아닐 때 예외가 호출자까지 전파됨",
      "replace": "try:\n    result = json.loads(raw)\nexcept (json.JSONDecodeError, KeyError):\n    result = _review_fallback()",
      "benefit": "JSON 파싱 실패 시 fallback 처리로 턴 루프 중단 방지"
    },
    {
      "id": "P2",
      "current": "\"await self._client.generate(prompt=user_content, model=model_id)\" — L88",
      "problem": "타임아웃 파라미터 없어 무한 대기 가능",
      "replace": "await self._client.generate(prompt=user_content, model=model_id, timeout=settings.debate_review_timeout)",
      "benefit": "타임아웃 설정으로 asyncio.gather 블로킹 방지"
    }
  ],
  "modification_needed": true,
  "reasoning": "현재 코드에 명확한 개선 여지 2건 존재"
}
```
Why good: PROPOSAL_N 블록의 current 필드에 실제 코드 직접 인용. replace는 실행 가능한 코드. 실패 시나리오 서술 없음.
</Good>

<Good>
토론 라운드 sycophancy 방지 동작:
```
  [토론 라운드 2/3] (순서: FH → PW → ET)
  FH: 미납득 유지 — F1 severe 여전히 미수정 방향
  PW: 미납득 → 납득 변경
    changed_due_to:
      agent: "FH"
      evidence: "F2 L88 타임아웃 없음 — P2 provide()에 timeout= 추가로 동시에 해소됨. 내 P2가 FH F2를 커버함을 확인"
    [근거 충분 — 줄 번호 + 코드 경로 명시. 입장 변경 인정]
  ET: 미납득 → 납득 변경
    changed_due_to:
      agent: "FH, PW"
      evidence: "FH F1(L95 json.loads) + PW P1(try/except fallback)이 내가 발견한 intent_gap(L95 파싱 미방어)을 커버함"
    [근거 충분 — 입장 변경 인정]
  현재 합의: 1/3 (FH 미납득 유지) — 계속

  [토론 라운드 3/3] (순서: PW → ET → FH)
  PW: 납득 유지
  ET: 납득 유지
  FH: 미납득 → 납득 변경
    changed_due_to:
      agent: "PW"
      evidence: "P1 try/except + P2 timeout이 F1+F2 모두 방어. 제안된 수정이 적용되면 내 severe 시나리오 해소"
    [근거 충분 — 입장 변경 인정]
  현재 합의: 3/3 | 평균 점수: 0.78 → 합의 달성!
```
Why good: 각 입장 변경에 코드 줄번호와 구체적 근거가 명시됨. "다른 에이전트에 동의"만으로는 인정되지 않음.
</Good>

<Good>
역할 경계 위반 감지 동작:
```
ET 응답 수신 → 금지 패턴 검사:
  패턴 "더 나은" 발견: "더 나은 방법은 try/except를 추가하는 것입니다"
  → 1회 재시도: "이전 응답에서 '더 나은' 패턴이 감지됐습니다. Proposal Writer 영역이므로 제거하고 execution_trace 형식만 채워주세요."
  → 재시도 응답: execution_trace 형식만 포함, 위반 없음
  → 재시도 응답 채택
```
Why good: 위반 패턴을 명시하여 재시도. 차단이 아닌 정정으로 흐름 유지.
</Good>

<Good>
Phase 5 테스트 실패 롤백:
```
[2/3] review_turn() — orchestrator.py:L80-140
  통합 수정안 적용... [Edit 도구 실행]
  수정 완료 → 테스트 실행: pytest tests/unit/services/test_orchestrator.py -v

  ❌ 테스트 실패 (3개 실패)
    FAILED test_review_turn_timeout — TypeError: generate() got unexpected keyword argument 'timeout'
  → 재수정 시도: timeout 파라미터 이름 확인 (generate 시그니처 재스캔)
    [Grep으로 inference_client.py generate() 시그니처 확인 → timeout → request_timeout]
    [Edit 도구로 재수정]
  → 테스트 재실행: pytest ...
  ✅ 테스트 통과 (5개) → 다음 함수로 진행
```
Why good: 실패 시 원인을 분석하고 재수정. 재수정 성공. 롤백 없이 정상 진행.
</Good>

<Good>
대형 함수 미검증 태그:
```
[3/4] _run_parallel_turns() ━━━━━━━━━━━━━━━━━━━━ 완료 (수정 불필요)
  ET: 납득 (0.82)
  FH: 납득 (0.78)
  PW: 납득 (0.80)
  [⚠️ 미검증 가능성 — 복잡 함수(620줄): 에이전트가 모든 실행 경로를 탐색하지 못했을 수 있음]
```
Why good: 높은 점수이지만 620줄짜리 대형 함수라 리뷰의 한계를 투명하게 표시.
</Good>

<Bad>
Execution Tracer가 역할 경계를 침범:
```json
{
  "execution_trace": {
    "steps": [
      "1. json.loads()로 파싱 — L95",
      "2. 파싱 실패 시 JSONDecodeError 발생 가능 (← Failure Hunter 영역)",
      "3. 더 나은 방법은 try/except 추가 (← Proposal Writer 영역)"
    ]
  }
}
```
Why bad: STEPS 필드에 실패 시나리오(FH 영역)와 개선 제안(PW 영역)이 혼입됨. 이 패턴은 금지 패턴 감지로 잡혀 재시도됨.
</Bad>

<Bad>
sycophancy — 근거 없는 입장 변경:
```
ET: 미납득 → 납득 변경
  changed_due_to:
    agent: "FH, PW"
    evidence: "다른 에이전트들의 의견이 설득력 있음"
→ [근거 불충분 — 줄 번호나 코드 인용 없음. 입장 변경 미인정. 이전 verdict '미납득' 유지]
```
Why bad: "설득력 있음"은 구체적 근거가 아님. 어떤 줄의 어떤 코드 때문에 입장이 바뀌었는지 명시해야 함.
</Bad>

<Bad>
Phase 3에서 즉시 수정:
```
합의 달성 (3/3) → Edit 도구로 코드 수정  ← 절대 금지
```
Why bad: Phase 3에서 수정하면 안 된다. 합의가 나도 수정은 예약만 하고, Phase 4에서 사용자에게 보고한 뒤 Phase 5에서 실행한다.
</Bad>

</Examples>

<Escalation_And_Stop_Conditions>

### 탐색 범위 제한

- **함수 50개 초과:** 50개까지만 탐색하고 채팅 출력 후 계속 진행
- **탐색 깊이 5레벨 초과:** 해당 경로 탐색 중단

### 순환 참조

- A→B→A 패턴 감지 시: 경고 노드로 표시하고 탐색 중단
- 경고 노드는 Phase 3 리뷰 루프에서 스킵

### Agent 호출 실패

- **IA 실패 시**: 인터페이스 검증 없이 3에이전트만으로 진행 (합의 기준 변동 없음)
- **ET/FH/PW 중 1개 실패 시**: 해당 에이전트를 "오류" 처리, 나머지 2개로 진행 (합의 기준 2/2로 조정)
- **2개 이상 에이전트 실패 시**: 해당 함수를 스킵 처리

### 역할 경계 위반 처리

- 1회 재시도 후에도 위반이 잔존하면: [⚠️ 역할 경계 위반 감지 — 재시도 후에도 잔존] 마킹 후 진행
- 위반이 있어도 해당 에이전트를 차단하지 않는다 (투명하게 표시 후 진행)

### sycophancy 감지

- changed_due_to.evidence가 막연하면: [근거 불충분] 마킹 + 이전 verdict 유지
- 3라운드 내내 근거 불충분으로 마킹된 에이전트가 있으면 해당 에이전트의 최초 verdict를 사용

### 수정 충돌 (Phase 5)

- 여러 에이전트의 수정 방향이 상충할 경우: 마지막 토론 라운드의 통합 합의안 우선
- 통합이 불명확하면 다음 우선순위:
  1. **안전성**: FH failures (severity=severe) 방어가 포함된 방향 우선
  2. **변경 범위**: 수정 줄 수가 적은(최소 침습적) 방향 우선
  3. **점수 차이**: score가 낮은(문제를 심각하게 본) 에이전트 방향 우선
  4. **폴백**: PW → FH → ET 순

### 테스트 실패 처리 (Phase 5)

- 수정 후 테스트 실패 시: 1회 재수정 시도
- 재수정도 실패 시: git checkout 또는 수동 복원으로 롤백, "수정 실패 — 롤백됨" 기록 후 다음 함수 진행
- 테스트 명령 탐지 실패 시: 테스트 스킵 + 완료 보고에 "수동 확인 권고" 표시

### 사용자 "수정 없이 문서만 유지" 선택

- Phase 4에서 해당 선택 시 코드 수정 없이 바로 종료
- 문서(summary.md + review.md)는 저장된 상태로 유지

</Escalation_And_Stop_Conditions>

<Final_Checklist>
- [ ] `{{ARGUMENTS}}`를 파싱하여 진입점을 파악했는가 (인수 없을 때만 AskUserQuestion 사용)
- [ ] Glob/Grep/Read로 실제 코드를 스캔하여 콜 그래프를 추적했는가 (추측 금지)
- [ ] 각 함수의 줄 수를 측정하여 complexity_tier(small/medium/large)를 기록했는가
- [ ] complexity_tier에 따라 MAX_ROUNDS를 조정했는가 (large는 N+1)
- [ ] git log/blame으로 각 함수의 변경 이력을 수집했는가 (git 불가 시 스킵)
- [ ] available-deferred-tools에서 Figma MCP 가용 여부 확인 → ToolSearch → 호출 순서를 따랐는가
- [ ] 독립 리뷰 단계에서 ET/FH/PW/IA를 병렬 호출했는가
- [ ] 각 에이전트에 역할 전용 출력 템플릿을 적용했는가 (공통 JSON 스키마 아님)
- [ ] IA 프롬프트에 caller/callee 시그니처와 호출 코드 스니펫이 포함되었는가
- [ ] 각 에이전트 응답 후 역할 경계 위반 감지를 실행했는가 (패턴 매칭)
- [ ] 위반 감지 시 1회 재시도를 수행했는가
- [ ] 각 에이전트 프롬프트에 변경 이력(git metadata)이 포함되었는가
- [ ] 각 에이전트 프롬프트에 accumulated_findings (누적 발견사항)가 포함되었는가
- [ ] 토론 라운드에서 ET/FH/PW만 순차 호출하되 로테이션 순서를 적용했는가 (IA 불참)
- [ ] IA findings가 토론 라운드 에이전트 프롬프트에 컨텍스트로 포함되었는가
- [ ] 토론 중 입장 변경 시 changed_due_to.evidence를 검증했는가 (근거 불충분 마킹)
- [ ] 3/3 납득이어도 평균 점수 < 0.75이면 토론에 진입했는가
- [ ] 수정 불필요 판정에서 large 함수에 미검증 태그를 추가했는가
- [ ] Phase 3에서 Edit 도구를 사용하지 않았는가 (합의 달성해도 수정 예약만)
- [ ] 모든 함수 토론 완료 후 summary.md + review.md를 저장했는가
- [ ] Phase 4에서 AskUserQuestion으로 사용자 확인을 받았는가 (하드 블락)
- [ ] Phase 5 시작 전 테스트 명령을 탐지했는가
- [ ] Phase 5에서만 Edit 도구로 수정을 실행했는가 (사용자 승인 후)
- [ ] 각 Edit 직후 테스트를 실행했는가
- [ ] 테스트 실패 시 재수정 → 롤백 흐름을 따랐는가
- [ ] 수정 충돌 시 동적 우선순위(안전성 → 변경 범위 → 점수 차이 → 폴백)를 적용했는가
- [ ] IA 미납득 시 수정 예약에 interface_checks가 필수 반영되었는가
- [ ] 수정 보류 함수의 대립 내용이 summary.md에 기록되었는가
- [ ] 전체 리뷰 루프 중 AskUserQuestion을 사용하지 않았는가 (진입점 파악 + Phase 4 제외)
</Final_Checklist>

Task: {{ARGUMENTS}}
