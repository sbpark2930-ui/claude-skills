---
name: harness-designer
description: 프로젝트를 스캔하고 settings.json 전체를 구조화된 Q&A, 리스크 기반 승인, deepMerge 정렬 적용, 정적 검증으로 설계하는 통합 하네스 설계 스킬
argument-hint: "[--global] [--interactive] [--scope hooks|permissions|env|model|plugins] [--preset <type>] [--check] [--test-hooks]"
---

# Harness Designer

<Purpose>
프로젝트 코드베이스를 4개 영역으로 자동 스캔하고, settings.json(hooks, permissions, env, model, plugins)에 대한 최적 설정을 진단/제안/승인/적용/검증하는 컨설팅형 스킬.

- **update-config과의 차이**: update-config은 단일 키 변경 도구. harness-designer는 전체 설정 설계.
- **omc-setup과의 차이**: omc-setup은 statusLine/model 등 초기 글로벌 설정 관리. harness-designer는 프로젝트별 hooks/permissions/env 중심.
- **출력물**: 적용된 settings + 백업 파일 + 검증 리포트
</Purpose>

<Use_When>
- 새 프로젝트에서 Claude Code 하네스를 처음 설정할 때
- 기존 설정을 점검하고 개선하고 싶을 때
- "harness", "하네스", "settings 설계", "hook 설정", "권한 설정" 키워드 사용 시
- hooks/permissions를 체계적으로 구성하고 싶을 때
- 프로젝트 특성에 맞는 최적 설정을 자동 추천받고 싶을 때
</Use_When>

<Do_Not_Use_When>
- statusLine 변경만 필요할 때 (omc-setup 사용)
- 단일 키 변경만 필요할 때 (update-config 사용)
- plugin hooks.json 수정이 필요할 때 (수동 편집 필요)
- settings.json 구조를 이미 잘 알고 직접 편집하려는 경우
</Do_Not_Use_When>

<Why_This_Exists>
Claude Code의 settings.json은 hooks, permissions, env, model, plugins 등 여러 도메인으로 구성되어 있고, 잘못된 hook이나 permission은 전체 세션을 망가뜨릴 수 있다. 수동 편집은 오류가 발생하기 쉽고, 프로젝트 특성에 맞는 최적 설정을 파악하려면 코드베이스 분석이 필요하다. 이 스킬은 자동 스캔 + 인터뷰로 최적 설정을 도출하고, 리스크 기반 승인으로 안전하게 적용한다.
</Why_This_Exists>

<Execution_Policy>
- 코드베이스 사실은 스캔으로 수집 (사용자에게 묻지 않음). 질문은 선호도/의도 확인용으로만
- 리스크 기반 승인: LOW=일괄 요약, MEDIUM/HIGH=항목별 승인, --interactive=모두 항목별
- deepMerge 정렬 병합 전략:
  - 배열: REPLACE (기존 배열을 새 배열로 교체)
  - 예외: permissions.allow와 permissions.deny는 APPEND (기존 항목 유지 + 신규 추가, 중복 제거)
  - 객체: deep merge (재귀적 병합)
  - 스칼라: replace
- 적용 전 반드시 백업 생성
- 적용 후 정적 검증만 수행 (hook 실행 절대 금지 -- 세션 상태 변조 위험)
- state_write/state_read로 세션 상태 유지 (중단 시 재개 가능)
</Execution_Policy>

<Target_File_Resolution>
| 추천 유형 | 대상 파일 | 플래그 |
|-----------|-----------|--------|
| 프로젝트 추천 (기본) | `.claude/settings.local.json` | (없음) |
| 글로벌 추천 | `~/.claude/settings.json` | `--global` |
| 플러그인 hooks.json | **절대 수정하지 않음** | N/A |
</Target_File_Resolution>

<Risk_Classification>
| 도메인 | 리스크 | 승인 방식 | 이유 |
|--------|--------|-----------|------|
| `hooks` | HIGH | 항목별 | Hook은 Node.js 스크립트를 실행하며, 잘못된 hook은 세션을 망가뜨림 |
| `permissions.deny` | HIGH | 항목별 | Deny 규칙은 도구 접근을 차단; 잘못된 규칙은 필요한 도구를 차단 |
| `permissions.allow` | MEDIUM | 항목별 | Allow 규칙은 도구 접근을 허용; 보안 관련이지만 덜 파괴적 |
| `env` | LOW | 일괄 요약 | 환경변수는 코드가 소비할 때까지 비활성 |
| `model` | LOW | 일괄 요약 | 모델 선택은 품질/비용에 영향하지만 쉽게 되돌림 가능 (omc-setup과 겹침 있음) |
| `plugins` | LOW | 일괄 요약 | 플러그인 활성화는 되돌림 가능 |

`--interactive` 플래그: 설정 시 모든 항목이 리스크와 무관하게 항목별 승인으로 전환.
</Risk_Classification>

<Steps>

## Phase 1: 4-Area Project Scan

### Step 1-1: 입력 파싱 + 세션 확인

`{{ARGUMENTS}}`에서 플래그를 파싱한다:
- `--global`: 글로벌 settings.json 대상
- `--interactive`: 모든 항목 개별 승인
- `--scope <domain>`: 특정 도메인만 스캔/추천 (hooks, permissions, env, model, plugins)
- `--preset <type>`: 프리셋 기반 빠른 설정 (Phase 1 스캔을 프리셋으로 대체)
  - 사용 가능: `nextjs`, `python-fastapi`, `rust-tauri`, `monorepo`, `minimal`
- `--check`: Drift 감지 모드 (스캔만 실행, 변경 적용 없음)
- `--test-hooks`: Phase 6에서 hook 안전 테스트 활성화

기존 세션 확인: `state_read(mode="harness-designer")`. 기존 세션이 있으면 재개 여부를 AskUserQuestion으로 확인.

### Step 1-1b: 프리셋 모드 (`--preset` 사용 시)

프리셋이 지정되면 Phase 1 스캔을 프리셋 기본값으로 대체한다:

| 프리셋 | permissions.allow 기본값 | hooks 기본값 | env 기본값 |
|--------|------------------------|-------------|-----------|
| `nextjs` | `Bash(npm:*)`, `Bash(npx:*)` | PostToolUse: eslint | `NODE_ENV=development` |
| `python-fastapi` | `Bash(pytest:*)`, `Bash(uvicorn:*)` | PostToolUse: ruff | `PYTHONPATH=.` |
| `rust-tauri` | `Bash(cargo:*)`, `Bash(npm:*)` | PostToolUse: cargo clippy | - |
| `monorepo` | `Bash(turbo:*)`, `Bash(pnpm:*)` | - | - |
| `minimal` | `Bash(npm test:*)` | - | - |

프리셋으로 커버되지 않는 영역(보안 패턴, 워크플로우)은 여전히 스캔한다.
프리셋 값은 Phase 4에서 사용자가 수정할 수 있다.

### Step 1-1c: Drift 감지 모드 (`--check` 사용 시)

Phase 1 스캔만 실행하고 현재 설정과 비교한다:

```
Harness Designer -- Drift 리포트

현재 설정 vs 코드베이스 상태:

| 항목 | 설정 상태 | 코드베이스 상태 | Drift |
|------|-----------|----------------|-------|
| vitest | permissions에 없음 | package.json에 감지됨 | ⚠️ 누락 |
| .env.local | deny에 없음 | 파일 존재 | ⚠️ 보안 위험 |
| eslint hook | 설정됨 | eslint.config.js 존재 | ✓ 일치 |
| cargo | permissions에 있음 | Cargo.toml 없음 | ⚠️ 불필요 |

권장 조치:
  1. vitest 권한 추가 (permissions.allow)
  2. .env.local deny 규칙 추가 (permissions.deny)
  3. cargo 권한 제거 (더 이상 사용하지 않음)
```

Drift 감지 후 종료 (변경 적용 없음). "수정하시겠습니까?" 선택 시 일반 모드로 전환.

### Step 1-2: Scan Area 1 -- 프로젝트 구조

`explore` 에이전트(haiku)를 사용하여 프로젝트 구조를 파악한다:

- **Glob 패턴**: `**/package.json`, `**/Cargo.toml`, `**/pyproject.toml`, `**/*.config.*`, `**/tsconfig.json`, `**/vite.config.*`, `**/webpack.config.*`
- **Grep 패턴**: `"scripts"` (in package.json), `"test"`, `"lint"`, `"build"`, `"format"`
- **감지 항목**: 언어/프레임워크, 빌드 시스템, 테스트 프레임워크, 모노레포 구조
- **출력**: 추론된 도구 권한 (예: `Bash(npm test:*)`, `Bash(cargo build:*)`, `Bash(npx eslint:*)`)

### Step 1-3: Scan Area 2 -- 보안 패턴

- **Glob 패턴**: `**/.env*`, `**/*.pem`, `**/*.key`, `**/credentials*`, `**/secrets*`
- **Grep 패턴**: `API_KEY`, `SECRET`, `PASSWORD`, `TOKEN`, `PRIVATE_KEY` (소스 코드 내)
- **감지 항목**: 민감 파일 경로, 하드코딩된 시크릿, 인증 패턴
- **출력**: 추천 deny 규칙 (예: `Read(.env*)`), 필요한 환경변수 제안

### Step 1-4: Scan Area 3 -- 기존 설정

대상 파일들을 모두 읽는다:
- `.claude/settings.json` (프로젝트, 공유/커밋됨)
- `.claude/settings.local.json` (프로젝트, 로컬)
- `~/.claude/settings.json` (글로벌)
- `~/.claude/settings.local.json` (글로벌 로컬)

**분석 항목**:
- 이미 설정된 도메인과 값
- 프로젝트/글로벌 간 충돌
- 누락된 추천 설정
- **출력**: 갭 분석 (현재 vs 추천)

### Step 1-5: Scan Area 4 -- 워크플로우 패턴

- **Glob 패턴**: `.github/workflows/*`, `Makefile`, `scripts/*`, `.husky/*`, `.pre-commit-config.yaml`
- **Grep 패턴**: `pre-commit`, `lint-staged`, `eslint`, `prettier`, `cargo fmt`, `cargo clippy`
- **감지 항목**: CI/CD 설정, pre-commit 패턴, 테스트/린트/빌드 명령어, 포맷터
- **출력**: 추천 hooks (PreToolUse: 위험 명령어 차단, PostToolUse: 자동 포맷/린트)

### Step 1-6: 스캔 요약 + 갭 감지

스캔 결과를 테이블로 요약한다:

```
Harness Designer -- 프로젝트 스캔 결과

| 영역 | 발견 항목 | 추천 수 | 갭 |
|------|-----------|---------|-----|
| 프로젝트 구조 | Node.js + Rust (Tauri) | 8 permissions | - |
| 보안 패턴 | .env 파일 2개 발견 | 2 deny rules | - |
| 기존 설정 | permissions 42개 존재 | 3 신규 추가 | model 미설정 |
| 워크플로우 | eslint + cargo clippy 감지 | 2 hooks | CI 설정 없음 |
```

**갭 감지**: 스캔으로 파악할 수 없는 항목이 있으면 Phase 2로 이동. 모든 정보가 충분하면 Phase 2를 건너뛰고 Phase 3로.

state에 스캔 결과를 저장: `state_write(mode="harness-designer", state={scan_results: {...}, phase: "scan_complete"})`

## Phase 2: 보완 Q&A

갭이 감지된 경우에만 실행한다. 한 번에 하나의 질문만 한다.

**갭 트리거 예시**:
- 테스트 프레임워크를 감지했지만 테스트 실행 명령어를 모르는 경우 → "테스트 실행 명령어가 무엇인가요?"
- CI/CD 설정이 없는 경우 → "CI/CD를 사용하고 계신가요? 어떤 서비스인가요?"
- 특수한 빌드 프로세스가 있는 경우 → "빌드 전에 실행해야 하는 커스텀 스크립트가 있나요?"

AskUserQuestion을 사용하되, 스캔 결과에서 이미 알고 있는 정보는 옵션으로 미리 제시한다.

## Phase 3: 추천 생성

스캔 결과 + Q&A 답변을 기반으로 도메인별 추천을 생성한다.

### 3-1: 도메인별 추천 생성

각 도메인에 대해 추천 목록을 생성한다. 각 추천에는:
- **도메인**: hooks / permissions.allow / permissions.deny / env / model / plugins
- **키**: 구체적 설정 키
- **현재값**: 기존 값 또는 "미설정"
- **제안값**: 추천 값 (JSON 형태)
- **이유**: 왜 이 설정을 추천하는지 (스캔 근거 포함)
- **리스크**: HIGH / MEDIUM / LOW

### 3-2: 추천 요약 제시

리스크 티어별로 그룹화하여 요약 테이블을 보여준다:

```
Harness Designer -- 추천 요약

| # | 도메인 | 키 | 리스크 | 설명 |
|---|--------|-----|--------|------|
| 1 | hooks.PreToolUse | rm -rf 차단 | HIGH | 위험 명령어 실행 전 확인 |
| 2 | hooks.PostToolUse | eslint 자동실행 | HIGH | 코드 변경 후 자동 린트 |
| 3 | permissions.deny | Read(.env*) | HIGH | 환경변수 파일 읽기 차단 |
| 4 | permissions.allow | Bash(npm test:*) | MEDIUM | 테스트 실행 허용 |
| 5 | env.NODE_ENV | development | LOW | 개발 환경 설정 |
| 6 | model | sonnet | LOW | 기본 모델 (omc-setup 겹침) |

HIGH: 2개 | MEDIUM: 1개 | LOW: 2개
```

## Phase 4: 리스크 기반 승인

### Step 4-1: LOW 리스크 일괄 승인

LOW 리스크 항목을 요약 테이블로 보여준다:

```
LOW 리스크 추천 (env, model, plugins):

| # | 도메인 | 현재값 | 제안값 | 이유 |
|---|--------|--------|--------|------|
| 1 | env.NODE_ENV | 미설정 | "development" | 개발 환경 표준 |
| 2 | model | 미설정 | "sonnet" | 비용 대비 성능 균형 (omc-setup 겹침) |
```

AskUserQuestion:
- "모든 LOW 리스크 변경 승인" -- 전부 승인
- "개별 검토" -- 각 항목을 하나씩 확인
- "전부 건너뛰기" -- LOW 리스크 항목 모두 스킵

### Step 4-2: MEDIUM 리스크 항목별 승인

각 MEDIUM 리스크 항목에 대해:

```
[4/6] permissions.allow | MEDIUM 리스크

현재: ["Bash(npm run dev)", ...]  (42개 항목)
제안: "Bash(npm test:*)" 추가 (APPEND)
이유: package.json에 test 스크립트가 감지됨. 테스트 실행 권한 필요.
```

AskUserQuestion:
- "승인" -- 승인 목록에 추가
- "건너뛰기" -- 이 항목 스킵
- "수정" -- 사용자가 대안 값 제공
- "상세 설명" -- 추가 컨텍스트 표시

### Step 4-3: HIGH 리스크 항목별 승인

각 HIGH 리스크 항목에 대해:

```
[1/6] hooks.PreToolUse | HIGH 리스크

현재: 미설정
제안:
  {
    "matcher": "Bash(rm -rf*)",
    "hooks": [{
      "type": "command",
      "command": "echo '위험: rm -rf 명령어 감지. 계속하시겠습니까?'"
    }]
  }
이유: 프로젝트에 중요한 소스 파일이 있음. 실수로 삭제 방지.
```

AskUserQuestion:
- "승인" -- 승인 목록에 추가
- "건너뛰기" -- 이 항목 스킵
- "수정" -- 사용자가 대안 값 제공
- "상세 설명" -- 추가 컨텍스트 표시

### Step 4-4: --interactive 오버라이드

`--interactive` 플래그가 설정된 경우, LOW 리스크 항목도 Step 4-2/4-3과 동일한 항목별 승인 절차를 따른다.

### Step 4-4b: 팀 공유 vs 개인 설정 분류

각 승인된 추천에 대해 자동으로 팀/개인 분류를 제안한다:

**팀 공유 (`.claude/settings.json` — 커밋됨):**
- 프로젝트 스크립트 permissions (npm test, cargo build 등)
- 코드 품질 hooks (eslint, prettier, clippy)
- 프로젝트별 deny 규칙 (.env 보호 등)

**개인 설정 (`.claude/settings.local.json` — 로컬):**
- model 선호 (sonnet vs opus)
- 개인 토큰이 포함된 env 변수
- 개인 워크플로우 hooks

```
팀/개인 분류 제안:

팀 공유 (.claude/settings.json):
  [A] permissions.allow: "Bash(npm test:*)" — 프로젝트 공통
  [A] hooks.PostToolUse: eslint — 팀 코드 품질
  [A] permissions.deny: "Read(.env*)" — 보안

개인 설정 (.claude/settings.local.json):
  [A] model: "sonnet" — 개인 선호
  [A] env.OPENAI_API_KEY: "..." — 개인 토큰

이 분류가 맞나요?
```

AskUserQuestion으로 분류를 확인받는다:
- "이 분류로 적용"
- "분류 수정: {변경 요청}"
- "전부 로컬에 저장"

### Step 4-5: 진행 표시

승인 과정에서 진행 상태를 보여준다:
`[A] 승인됨, [S] 건너뜀, [M] 수정됨, [>] 현재, [ ] 대기`

예: `[A][A][S][>][ ][ ]  (3/6 완료)`

### Step 4-6: 충돌 처리

제안값이 기존값과 충돌할 때:
- 양쪽 값을 나란히 표시
- AskUserQuestion: "기존 유지" / "추천 사용" / "커스텀 값 입력"
- permissions.allow/deny의 경우: APPEND이므로 "기존 + 신규 병합" 옵션 추가

state에 승인 결과 저장.

## Phase 5: 적용

### Step 5-1: 백업 생성

적용 전 현재 파일을 백업한다:
```
cp .claude/settings.local.json .claude/settings.local.json.backup.{YYYYMMDD-HHmmss}
```
백업 경로를 state에 저장.

### Step 5-2: 드라이런 diff 표시

현재 vs 적용 후 JSON을 diff 형태로 보여준다:

```diff
{
  "permissions": {
    "allow": [
      "Bash(npm run dev)",
+     "Bash(npm test:*)",
+     "Bash(npx eslint:*)"
    ],
    "deny": [
+     "Read(.env*)"
    ]
  },
+ "hooks": {
+   "PreToolUse": [...]
+ },
+ "env": {
+   "NODE_ENV": "development"
+ }
}
```

### Step 5-3: 최종 확인

AskUserQuestion:
- "이 변경사항 적용" -- 적용 진행
- "돌아가서 수정" -- Phase 4로 돌아감
- "취소" -- 적용하지 않고 종료

### Step 5-4: deepMerge 정렬 병합 + 쓰기

병합 전략 참조 테이블:

| 값 유형 | 전략 | 예시 |
|---------|------|------|
| 객체 | Deep merge | `{a:1, b:2}` + `{b:3, c:4}` = `{a:1, b:3, c:4}` |
| 배열 (일반) | REPLACE | `["a","b"]` + `["c"]` = `["c"]` |
| `permissions.allow` | APPEND | `["Bash(npm:*)"]` + `["Read(*)"]` = `["Bash(npm:*)", "Read(*)"]` (중복 제거) |
| `permissions.deny` | APPEND | `["Bash(rm:*)"]` + `["Write(/etc)"]` = `["Bash(rm:*)", "Write(/etc)"]` (중복 제거) |
| 스칼라 | Replace | `"sonnet"` + `"opus"` = `"opus"` |

대상 파일에 JSON.stringify(result, null, 2)로 쓴다.

### Step 5-5: JSON 파싱 검증

쓴 직후 파일을 다시 읽고 JSON.parse가 성공하는지 확인. 실패 시 즉시 롤백.

## Phase 6: 정적 검증

**절대 hook을 실행하지 않는다.** Hook은 Node.js 스크립트를 실행하며 세션 상태를 변조한다.

### Step 6-1: JSON 파싱 재검증

파일을 다시 읽고 JSON으로 파싱한다. 실패 시 자동 롤백 + 오류 리포트.

### Step 6-2: JSON 스키마 검증

알려진 키의 타입을 검증한다:
- `permissions.allow`: string 배열이어야 함
- `permissions.deny`: string 배열이어야 함
- `hooks.*` 항목: `matcher` (string)와 `hooks` (배열) 필드 필요
- `env` 값: string이어야 함
- `model`: string이어야 함
- `enabledPlugins`: object이어야 함

### Step 6-3: Hook 스크립트 파일 존재 확인

각 hook 항목에서 참조된 스크립트 경로를 추출하고, 해당 파일이 디스크에 존재하는지 확인한다.
- 존재: "Hook 스크립트 확인: /path/to/script.js"
- 미존재: "경고: Hook 스크립트를 찾을 수 없음: /path/to/script.js" (경고만, 자동 롤백 아님)

### Step 6-4: Timeout 범위 검증

Hook의 timeout 값이 있는 경우:
- 양의 정수인지 확인
- 1-300초 범위 내인지 확인
- 범위 초과 시 경고

### Step 6-4b: Hook 안전 테스트 (`--test-hooks` 사용 시)

**절대 프로젝트 파일에 대해 hook을 실행하지 않는다.** 임시 디렉토리에서만 테스트한다.

1. 임시 디렉토리 생성: `mktemp -d`
2. 최소 테스트 파일 생성: `echo "test" > {tmpdir}/test.txt`
3. 각 hook을 임시 파일 대상으로 실행:
   ```bash
   cd {tmpdir} && timeout 10 node -e "{hook_command}" 2>&1
   ```
4. 결과 수집:

```
Hook 안전 테스트 결과:

| Hook | 타입 | 결과 | 소요 시간 |
|------|------|------|-----------|
| PreToolUse: rm 차단 | command | PASS | 0.1s |
| PostToolUse: eslint | command | FAIL — eslint 미설치 | 0.3s |
| PostToolUse: cargo fmt | command | TIMEOUT (10s) | 10.0s |
```

5. 실패한 hook에 대해 경고:
   - FAIL: "이 hook이 실행 환경에서 동작하지 않을 수 있습니다. 계속 적용할까요?"
   - TIMEOUT: "이 hook이 timeout 범위를 초과합니다. timeout 값을 조정할까요?"

### Step 6-5: 검증 리포트

```
Harness Designer -- 검증 리포트

대상: .claude/settings.local.json
백업: .claude/settings.local.json.backup.20260406-143022

적용된 변경사항:
  [A] hooks.PreToolUse: 매처 2개 추가 (HIGH)
  [A] permissions.allow: 규칙 5개 추가 (MEDIUM)
  [A] permissions.deny: 규칙 1개 추가 (HIGH)
  [S] model: 건너뜀 (현재값 유지)
  [M] env.NODE_ENV: "test"로 수정 (LOW)

정적 검증:
  JSON 파싱: PASS
  스키마 검증: PASS
  Hook 스크립트 존재:
    모든 참조 스크립트 확인됨
  Timeout 범위: PASS (모두 1-300초 범위 내)

롤백: "rollback harness-designer" 로 백업 복원 가능
```

### Step 6-6: 핸드오프

AskUserQuestion:
- "완료" -- 세션 종료
- "롤백" -- 백업에서 복원
- "추가 조정" -- Phase 4로 돌아가서 추가 수정

state 정리: `state_write(mode="harness-designer", state={...status: "complete"})`

</Steps>

<Tool_Usage>
- `explore` 에이전트 (haiku): Phase 1 프로젝트 구조 파악
- `Glob`: 파일 패턴 매칭 (package.json, .env, CI 설정 등)
- `Grep`: 코드 내 패턴 검색 (시크릿, 스크립트 명령어 등)
- `Read`: settings.json 파일 읽기, 백업 파일 읽기
- `Write`: settings.json 적용, 백업 생성
- `AskUserQuestion`: 보완 Q&A, 항목별 승인, 최종 확인
- `state_write` / `state_read`: 세션 상태 유지/재개
- `Bash`: 파일 존재 확인 (`test -f`), JSON 파싱 검증
</Tool_Usage>

<Examples>
<Good>
스캔 후 적절한 추천:
```
프로젝트에서 package.json의 "test": "vitest" 스크립트를 감지했습니다.

[추천] permissions.allow에 "Bash(npx vitest:*)" 추가 (MEDIUM 리스크)
이유: 테스트 실행 권한이 없으면 매번 수동 승인 필요.
```
</Good>

<Good>
리스크 기반 일괄 승인:
```
LOW 리스크 추천 3개:
| env.NODE_ENV | "development" | 개발 환경 표준 |
| model | "sonnet" | 비용/성능 균형 |
| plugins.hud | true | HUD 활성화 |

[모든 LOW 리스크 승인] [개별 검토] [전부 건너뛰기]
```
</Good>

<Good>
충돌 감지 및 처리:
```
permissions.allow 충돌 감지:

기존: ["Bash(npm run dev)"]
제안: ["Bash(npm run dev)", "Bash(npm test:*)"]  (APPEND)

→ 기존 항목은 유지하고 "Bash(npm test:*)"만 추가합니다.
```
</Good>

<Bad>
사용자에게 코드베이스 사실을 묻는 경우:
```
"어떤 테스트 프레임워크를 사용하시나요?"
```
왜 나쁜가: package.json을 스캔하면 알 수 있는 정보. 스캔 먼저, 질문은 선호도만.
</Bad>

<Bad>
Hook을 실행하여 검증하는 경우:
```
"추가된 PreToolUse hook을 테스트하기 위해 rm 명령어를 실행해봅니다..."
```
왜 나쁜가: Hook 실행은 세션 상태를 변조할 수 있어 절대 금지. 정적 검증만 수행.
</Bad>

<Bad>
배열을 Union으로 병합하는 경우:
```
permissions.allow: 기존 42개 + 신규 5개 = 47개 (union)
```
왜 나쁜가: OMC의 deepMerge는 배열을 REPLACE한다. permissions.allow/deny만 APPEND 예외.
</Bad>
</Examples>

<Escalation_And_Stop_Conditions>
- **스캔 실패**: 프로젝트 파일을 읽을 수 없으면 해당 영역 건너뛰고 Q&A로 보완
- **settings 파일 없음**: 첫 실행 시 새 파일 생성 (빈 JSON `{}`)
- **JSON 파싱 실패**: 즉시 롤백 + 오류 리포트
- **사용자 취소**: "취소", "중단", "stop" 시 현재 상태 저장 후 종료
- **20+ 추천 항목**: 도메인별 요약을 먼저 보여주고, 우선순위 높은 것부터 처리
- **모든 항목 건너뜀**: "모든 추천을 건너뛰셨습니다. 변경 없이 종료합니다." + 스캔 결과만 저장
</Escalation_And_Stop_Conditions>

<Final_Checklist>
- [ ] Phase 1: 4개 영역 스캔 완료
- [ ] Phase 2: 갭 감지 시 보완 Q&A 진행
- [ ] Phase 3: 각 추천에 도메인, 현재값, 제안값, 이유, 리스크 포함
- [ ] Phase 4: LOW=일괄, MEDIUM/HIGH=항목별 승인
- [ ] Phase 4: --interactive 시 모든 항목 개별 승인
- [ ] Phase 4: 충돌 감지 및 선택지 제공
- [ ] Phase 5: 적용 전 백업 생성
- [ ] Phase 5: 드라이런 diff 표시
- [ ] Phase 5: deepMerge 정렬 병합 (배열=REPLACE, permissions=APPEND)
- [ ] Phase 6: JSON 파싱 검증
- [ ] Phase 6: 스키마 검증 (타입 체크)
- [ ] Phase 6: Hook 스크립트 존재 확인
- [ ] Phase 6: Timeout 범위 확인 (1-300초)
- [ ] Phase 6: Hook 절대 실행하지 않음
- [ ] Phase 6: 검증 리포트 출력
- [ ] 롤백 옵션 제공
- [ ] 프로젝트 레벨 기본, --global로 글로벌 선택 가능
- [ ] plugin hooks.json 절대 수정하지 않음
- [ ] statusLine 추천 제외 (omc-setup 영역)
</Final_Checklist>

<Advanced>

## 세션 상태 구조

```json
{
  "active": true,
  "current_phase": "harness-designer",
  "state": {
    "session_id": "<uuid>",
    "flags": {
      "global": false,
      "interactive": false,
      "scope": null
    },
    "scan_results": {
      "project_structure": {...},
      "security_patterns": {...},
      "existing_settings": {...},
      "workflow_patterns": {...}
    },
    "recommendations": [...],
    "approvals": {
      "approved": [...],
      "skipped": [...],
      "modified": [...]
    },
    "backup_path": null,
    "target_file": ".claude/settings.local.json",
    "phase": "scan_complete|qa|recommend|approve|apply|verify|complete"
  }
}
```

## 설정 도메인 참조 (statusLine 제외)

| 도메인 | settings.json 키 | 값 타입 | 병합 전략 |
|--------|-------------------|---------|-----------|
| hooks | `hooks.PreToolUse`, `hooks.PostToolUse`, `hooks.Notification`, `hooks.Stop` | 배열 of objects | REPLACE |
| permissions.allow | `permissions.allow` | 배열 of strings | APPEND |
| permissions.deny | `permissions.deny` | 배열 of strings | APPEND |
| env | `env` | object of string values | Deep merge |
| model | `model` | string | Replace |
| plugins | `enabledPlugins` | object | Deep merge |

## 스캔 패턴 힌트

| 영역 | Glob | Grep |
|------|------|------|
| 구조 | `**/package.json`, `**/Cargo.toml` | `"scripts"`, `"test"` |
| 보안 | `**/.env*`, `**/*.pem` | `API_KEY`, `SECRET`, `TOKEN` |
| 설정 | `.claude/settings*.json` | - |
| 워크플로우 | `.github/workflows/*`, `Makefile` | `pre-commit`, `eslint`, `prettier` |

</Advanced>

Task: {{ARGUMENTS}}
