---
name: code-audit-interview
description: 코드베이스를 4개 차원(품질/보안/아키텍처/성능)으로 심층 스캔하고, 발견된 이슈를 하나씩 Q&A로 납득시킨 뒤 승인된 항목을 코드 수정까지 완료하는 코드 감사 스킬
argument-hint: "<감사할 파일 경로 또는 디렉토리> [--incremental] [--focus <dimension>]"
---

<Purpose>
Code Audit Interview는 지정한 파일/디렉토리를 4개 차원(코드 품질, 보안 취약점, 아키텍처/설계, 성능/효율)에서 자동 스캔하여 이슈를 발견하고, 이슈마다 하드 블락 Q&A로 사용자가 납득한 뒤에만 코드를 수정한다.

refactor-interview와의 차이:
- refactor-interview: 사용자가 "무엇을 바꿀지" 이미 알고 시작 → Q&A로 납득 후 실행
- code-audit-interview: Claude가 "무엇이 문제인지" 먼저 발견 → Q&A로 납득 후 수정

핵심 메커니즘:
1. Glob/Grep/Read로 대상을 스캔하여 4차원 이슈를 추출한다.
2. 이슈를 심각도(HIGH/MEDIUM/LOW) 순으로 정렬하여 목록을 제시한다.
3. 이슈마다 Before 코드 블록 + 문제 설명 + After 제안 코드 + 이유를 보여준다.
4. 하드 블락: "수정하겠습니다" 또는 "거짓양성 확인" 전까지 다음 이슈로 진행 불가.
5. "건너뛰기" 선택지는 존재하지 않는다.
6. 거짓양성 3회 반복 주장 시 "사용자 판단 승인"으로 로그하고 다음 이슈.
7. 승인된 이슈는 즉시 코드를 수정한다.
8. 전체 완료 후 감사 보고서를 생성한다.

출력: `.omc/plans/code-audit-{slug}.md`
</Purpose>

<Use_When>
- 특정 파일이나 디렉토리의 코드 품질을 종합 점검하고 싶을 때
- "코드 감사", "audit", "코드 리뷰 해줘", "이 코드 문제점 찾아줘" 같은 요청이 들어올 때
- 보안 취약점, 성능 병목, 설계 문제를 체계적으로 찾고 싶을 때
- 발견된 문제를 이해하고 납득한 뒤 수정하고 싶을 때 (자동 수정 대신 인터랙티브하게)
- PR 전 또는 신규 기능 구현 후 해당 코드를 점검하고 싶을 때
</Use_When>

<Do_Not_Use_When>
- 이미 무엇을 리팩토링할지 알고 있을 때 -- refactor-interview를 사용하세요
- 새 기능을 설계할 때 -- plan-interview를 사용하세요
- 빠른 단일 수정이 필요할 때 -- executor로 직접 실행하세요
- 사용자가 "그냥 고쳐줘", "질문 없이 수정해" 라고 명시한 경우 -- executor를 사용하세요
</Do_Not_Use_When>

<Why_This_Exists>
코드 리뷰 도구들은 문제를 발견하지만 "왜 이것이 문제인지", "어떻게 고쳐야 하는지"를 개발자가 이해하지 못한 채 넘어가게 만든다. Code Audit Interview는 이 문제를 구조적으로 해결한다:

- AI가 먼저 코드를 스캔하여 실제 이슈를 발견한다 (추측이 아닌 근거 기반).
- 각 이슈의 Before/After 실제 코드를 보여줘서 개발자가 직접 읽고 판단할 수 있게 한다.
- 하드 블락으로 이해 없는 수정을 방지한다.
- 거짓양성(false positive) 처리를 통해 Claude가 틀렸을 때 사용자가 정정할 수 있다.
</Why_This_Exists>

<Execution_Policy>
- 코드를 먼저 스캔한다 -- 개발자에게 묻기 전에 Glob/Grep/Read로 사실을 확인한다
- 한 번에 하나의 이슈만 다룬다 -- 여러 이슈를 동시에 설명하지 않는다
- Before/After 실제 코드 블록을 보여준다 -- Read 도구로 읽은 실제 코드 전체 (100줄도 생략 없이)
- 이슈 납득 없이 다음 이슈로 진행하지 않는다 (하드 블락)
- "건너뛰기" 선택지는 절대 제공하지 않는다
- 거짓양성 3회 반복 시에만 사용자 판단을 승인하고 넘어간다
- 승인된 이슈는 Q&A 직후 바로 코드를 수정한다
- 인터뷰 상태를 `state_write`로 저장하여 세션 중단/재개를 지원한다
</Execution_Policy>

<Steps>

## Phase 1: 코드 스캔 및 이슈 추출

**목적:** 대상 코드를 4개 차원에서 자동 분석하여 이슈 목록을 생성한다.

### Step 1-1: 입력 파싱 및 세션 확인

1. `{{ARGUMENTS}}`에서 감사 대상 경로와 플래그를 파싱한다:
   - 대상 경로 (파일 또는 디렉토리)
   - `--incremental`: 증분 감사 모드 (이전 감사 결과와 비교하여 신규 이슈만 표시)
   - `--focus <dimension>`: 차원별 포커스 (`security`, `performance`, `quality`, `architecture`)
     - 해당 차원의 이슈를 목록 상단에 배치 (다른 차원을 숨기지 않음, 정렬만 변경)
2. `{{ARGUMENTS}}`가 비어 있으면 `AskUserQuestion`으로 대상을 요청한다:
   - "어떤 파일 또는 디렉토리를 감사하시겠습니까?"
3. `state_read`로 기존 세션을 확인한다.
   - 진행 중인 세션이 있으면 `AskUserQuestion`으로 재개 여부를 묻는다:
     - "이전 코드 감사를 이어서 진행할까요?" → 마지막 ACTIVE 이슈부터 재개
     - "새로 시작할까요?" → 기존 상태 초기화

### Step 1-1b: 증분 감사 초기화 (`--incremental` 모드)

`--incremental` 플래그 사용 시:

1. `.omc/plans/code-audit-*.md` 파일을 검색하여 동일 대상의 이전 감사 보고서를 찾는다.
2. 이전 보고서에서 다음을 추출한다:
   - 이전에 발견된 이슈 목록 (파일, 줄, 제목)
   - "사용자 판단 승인 (거짓양성)"으로 처리된 패턴
3. state에 저장:
   ```json
   {
     "previous_audit_path": ".omc/plans/code-audit-engine.md",
     "previous_issues": ["하드코딩 시크릿 auth.py:45", ...],
     "filtered_patterns": ["print() 사용은 이 프로젝트에서 허용"]
   }
   ```
4. Phase 1-3에서 이슈 목록 생성 시:
   - 이전 감사에서 이미 발견+처리된 이슈는 `[기존]` 태그 표시
   - 이전에 거짓양성 처리된 패턴과 일치하는 이슈는 자동 필터링
   - 신규 이슈만 `[신규]` 태그로 강조

### Step 1-1c: 커스텀 룰셋 로드

`.omc/audit-rules.json` 파일이 존재하면 로드한다:

```json
{
  "ignore_patterns": ["print() 사용", "bare except"],
  "custom_rules": [
    {
      "pattern": "datetime\\.now\\(\\)",
      "message": "timezone-aware datetime 사용 권장 (datetime.now(tz=UTC))",
      "severity": "MEDIUM",
      "dimension": "quality"
    }
  ],
  "severity_overrides": {
    "N+1 쿼리": "HIGH",
    "Dead code": "LOW"
  }
}
```

- `ignore_patterns`: 이 패턴과 일치하는 이슈를 스캔에서 제외
- `custom_rules`: 추가 스캔 패턴 (Grep으로 검색)
- `severity_overrides`: 특정 이슈의 심각도를 강제 지정

### Step 1-2: 4차원 코드 스캔

Glob/Grep/Read로 대상 코드를 탐색하여 각 차원별 이슈를 찾는다:

**차원 1 — 코드 품질:**
- 큰 함수/클래스 (함수 200줄↑, 파일 500줄↑)
- 중복 코드 블록 (유사 패턴 3개↑)
- 깊은 중첩 (4단계↑ 들여쓰기)
- 과도한 파라미터 (5개↑)
- Dead code (import/호출 0건인 public 함수)
- 네이밍 불명확 (단일 문자 변수, 의미 없는 이름)

**차원 2 — 보안 취약점:**
- SQL 인젝션 패턴 (f-string SQL, 직접 문자열 포맷팅)
- 하드코딩된 시크릿 (password, key, secret, token 리터럴)
- 인증/인가 누락 (공개 엔드포인트에 권한 체크 없음)
- 안전하지 않은 역직렬화
- Path traversal 패턴 (`..` 비검증 사용)
- eval/exec 사용

**차원 3 — 아키텍처/설계:**
- 순환 의존 (A → B → A)
- 단일 책임 위반 (한 클래스/파일이 관련 없는 두 역할 담당)
- 인터페이스 일관성 위반 (같은 패턴의 다른 구현)
- 글로벌 상태 남용
- 레이어 경계 위반 (라우터에서 직접 DB 쿼리 등)

**차원 4 — 성능/효율:**
- N+1 쿼리 패턴 (루프 안 DB 호출)
- 불필요한 동기 블로킹 (async context에서 sync 호출)
- 캐시 미적용 (반복 호출되는 비싼 연산)
- 대용량 데이터 메모리 전체 로드 (페이지네이션/스트리밍 미사용)
- 불필요한 중첩 루프 (O(n²) 가능성)

### Step 1-3: 이슈 목록 생성 및 우선순위 정렬

**심각도 자동 조정 (스캔 후 적용):**

각 이슈에 대해 다음 조건으로 심각도를 자동 조정한다:

| 조건 | 조정 | 측정 방법 |
|------|------|-----------|
| 해당 함수에 테스트 커버리지 있음 | 심각도 1단계 하향 | Grep으로 test 파일에서 함수명 참조 확인 |
| 해당 파일이 최근 7일 내 변경됨 | 심각도 1단계 상향 | `git log --since="7 days ago" -- {file}` |
| `severity_overrides`에 해당 | 최종 심각도 강제 지정 | `.omc/audit-rules.json` 참조 |

- 두 조건이 동시에 충족되면 상쇄 (변경 없음)
- `severity_overrides`는 자동 조정보다 우선
- 조정 이유를 이슈 목록에 표시:
  ```
  [2] [HIGH→MEDIUM] 보안: db.py:123 — SQL 인젝션 패턴 (↓ 테스트 존재)
  [3] [LOW→MEDIUM] 성능: cache.py:45 — 캐시 미적용 (↑ 3일 전 변경)
  ```

**`--focus` 정렬:**
`--focus security` 사용 시 보안 이슈를 목록 상단에 배치하고, 나머지 차원은 기존 심각도 순서 유지.

발견된 이슈를 심각도 순으로 정렬한다:

| 심각도 | 기준 | 예시 |
|--------|------|------|
| **HIGH** | 보안 취약점, 데이터 손실 위험, 시스템 장애 가능 | SQL 인젝션, 하드코딩 시크릿 |
| **MEDIUM** | 유지보수성 심각 저하, 성능 병목, 설계 결함 | N+1 쿼리, 순환 의존, 신(God) 파일 |
| **LOW** | 코드 품질 개선, 사소한 비효율, 스타일 | Dead code, 과도한 파라미터 |

### Step 1-4: 스캔 결과 제시 및 확인

스캔 결과를 요약하여 사용자에게 제시한다:

```
코드 감사 결과:

대상: {path}
총 이슈: {total}개 (HIGH: {h}개, MEDIUM: {m}개, LOW: {l}개)

이슈 목록 (우선순위 순):
[1] [HIGH] 보안: auth.py:45 — 하드코딩된 JWT 시크릿
[2] [HIGH] 보안: db.py:123 — SQL 인젝션 패턴
[3] [MEDIUM] 성능: engine.py:234 — N+1 쿼리 루프
[4] [MEDIUM] 아키텍처: agent_service.py — 두 클래스 혼재 (단일 책임 위반)
[5] [LOW] 품질: utils.py:89 — Dead code (호출처 0건)
```

`AskUserQuestion`으로 목록을 확인받는다:
- "이 이슈 목록으로 진행할까요?"
- "특정 이슈를 제외하거나 추가해주세요"
- "HIGH 이슈만 먼저 진행할까요?"

### Step 1-5: 초기 상태 저장

`state_write`로 세션을 저장한다:

```json
{
  "active": true,
  "current_phase": "code-audit-interview",
  "state": {
    "audit_id": "<uuid>",
    "target": "<대상 경로>",
    "issues": [
      {
        "id": 1,
        "severity": "HIGH",
        "dimension": "security",
        "file": "auth.py",
        "lines": "45-52",
        "title": "하드코딩된 JWT 시크릿",
        "status": "PENDING"
      }
    ],
    "current_issue_index": 0,
    "audit_log": [],
    "false_positive_counts": {}
  }
}
```

---

## Phase 2: 이슈별 Q&A 루프

**목적:** 각 이슈를 하나씩 설명하고, 사용자가 납득한 뒤 코드를 수정한다.

### Step 2-1: 이슈 상세 제시

현재 이슈의 내용을 구체적으로 제시한다:

1. **문제 위치** — 파일명, 줄 번호:
   ```
   [2/5] [HIGH] 보안 — engine.py:123-145
   ```

2. **Before 코드 블록** — `Read` 도구로 해당 영역 전체를 읽어 직접 표시:
   - 100줄 이상이어도 생략(`...`) 없이 전체 표시

   ```python
   # Before: engine.py:123-145 (Read 도구 결과 — 전체)
   def get_user(user_id: str):
       query = f"SELECT * FROM users WHERE id = '{user_id}'"
       return db.execute(query)
   ```

3. **문제 설명** — 구체적으로 왜 이것이 이슈인지:
   ```
   SQL 인젝션 취약점: f-string으로 SQL을 직접 조합하면
   user_id에 "'; DROP TABLE users; --" 같은 값이 들어올 때
   DB가 손상됩니다.
   ```

4. **After 제안 코드** — Claude가 직접 작성:

   ```python
   # After: engine.py:123-145 (제안)
   def get_user(user_id: str):
       query = "SELECT * FROM users WHERE id = :user_id"
       return db.execute(query, {"user_id": user_id})
   ```

5. **Why** — 이 수정이 왜 필요한지:
   ```
   파라미터 바인딩을 사용하면 DB 드라이버가 사용자 입력을
   자동으로 이스케이프하여 SQL 인젝션을 원천 차단합니다.
   ```

### Step 2-2: 하드 블락 납득 확인

`AskUserQuestion`으로 사용자 결정을 받는다.

**AskUserQuestion 질문 형식:**
```
이슈 [{n}/{total}] [{severity}] {title} — 어떻게 하시겠습니까? [재설명: {round}회]
```

**선택지 (4개 고정, "건너뛰기" 절대 없음):**

1. **"수정하겠습니다."**
   → 이슈 상태: PENDING → APPROVED
   → 즉시 After 코드로 파일 수정 (Step 2-3)
   → 진행 표시기 `[ ]` → `[✓]`

2. **"거짓양성입니다. 실제 문제가 아닙니다."**
   → 거짓양성 카운터 +1
   → **1~2회:** Claude가 다른 근거 또는 예시를 제시하고 재질문
   → **3회:** "사용자 판단 승인"으로 기록 → 다음 이슈 (Step 2-5)
   → 진행 표시기 `[ ]` → `[○]` (사용자 판단 승인)

3. **"더 자세히 설명해주세요. {구체적 질문}"**
   → Claude가 추가 설명 (다른 예시, 실제 공격 시나리오, 성능 측정값 등)
   → 동일 이슈에서 Q&A 계속 (라운드 +1)

4. **"After 코드를 수정해주세요. {변경 요청}"**
   → 사용자 요청을 반영하여 After 코드 재작성
   → 수정된 코드를 다시 제시 후 납득 확인 (라운드 +1)

### Step 2-3: 코드 수정 실행 (APPROVED 즉시)

사용자가 "수정하겠습니다"를 선택하면 즉시 코드를 수정한다:

1. `Edit` 도구로 Before 코드를 After 코드로 교체
2. 수정 완료 메시지:
   ```
   ✓ engine.py:123-145 수정 완료 → 파라미터 바인딩 적용
   ```
3. 이슈 상태 → FIXED, `state_write`로 저장

### Step 2-4: 라운드 캡 규칙

| 심각도 | soft cap (경고) | hard cap (강제 진행) |
|--------|-----------------|---------------------|
| HIGH   | 5 라운드        | 8 라운드            |
| MEDIUM | 3 라운드        | 6 라운드            |
| LOW    | 2 라운드        | 4 라운드            |

**soft cap 도달 시:**
```
같은 이슈에서 {n}번 설명했습니다. 다른 방식으로 접근해볼까요?
```

**hard cap 도달 시:**
→ "미해결"로 기록하고 다음 이슈로 진행
→ 보고서에 미해결 이슈로 표시

### Step 2-5: 거짓양성 처리 상세

| 시도 | Claude 반응 |
|------|-------------|
| 1회 | 실제 공격 시나리오 또는 버그 재현 예시를 추가 제시 후 재질문 |
| 2회 | 관련 공식 문서 또는 업계 표준을 근거로 제시 후 재질문 |
| 3회 | "사용자 판단 승인 — 이 이슈를 거짓양성으로 기록합니다." → 다음 이슈 |

거짓양성 3회 시 보고서에 "사용자 판단 승인 (거짓양성)" 항목으로 기록.

### Step 2-6: 진행 표시기 업데이트

이슈 상태가 변경될 때마다 텍스트 진행 표시기를 출력한다:

```
감사 진행 상황 (5개 이슈):
[✓] [HIGH] 하드코딩 시크릿 → 수정 완료
[✓] [HIGH] SQL 인젝션 → 수정 완료
[▶] [MEDIUM] N+1 쿼리 → Q&A 진행 중 (2회)
[ ] [MEDIUM] 단일 책임 위반
[○] [LOW] Dead code → 사용자 판단 승인 (거짓양성)
```

기호 규칙:
| 기호 | 상태 | 의미 |
|------|------|------|
| `[✓]` | FIXED | 수정 완료 |
| `[▶]` | ACTIVE | 현재 Q&A 진행 중 |
| `[ ]` | PENDING | 대기 중 |
| `[○]` | USER_APPROVED | 사용자 판단 승인 (거짓양성) |
| `[!]` | UNRESOLVED | 미해결 (hard cap) |

### Step 2-7: 상태 저장

이슈 상태가 변경될 때마다 `state_write`로 저장한다.

---

## Phase 3: 보고서 생성

**목적:** 감사 완료 후 `.omc/plans/code-audit-{slug}.md`에 보고서를 생성한다.

### Step 3-1: 보고서 구성

```markdown
# {대상} 코드 감사 보고서

## 메타데이터
- 감사 일시: {date}
- 대상: {path}
- 총 이슈: {total}개
- 수정 완료: {fixed}개
- 사용자 판단 승인 (거짓양성): {user_approved}개
- 미해결: {unresolved}개

## 요약

| 차원 | 발견 | 수정 | 거짓양성 | 미해결 |
|------|------|------|----------|--------|
| 코드 품질 | | | | |
| 보안 취약점 | | | | |
| 아키텍처/설계 | | | | |
| 성능/효율 | | | | |

## 수정 완료 이슈

### [HIGH] {title}
- **파일:** {file}:{lines}
- **문제:** {description}
- **수정:** {fix_summary}

## 사용자 판단 승인 (거짓양성 확인)

### [LOW] {title}
- **파일:** {file}:{lines}
- **Claude 판단:** {claude_reasoning}
- **사용자 판단:** 거짓양성 — {rounds}회 확인 후 승인

## 미해결 이슈

### [MEDIUM] {title}
- **파일:** {file}:{lines}
- **이유:** {unresolved_reason}
- **권고:** {recommendation}

## 감사 로그
<details><summary>전체 Q&A 기록</summary>
...
</details>
```

### Step 3-2: 파일 저장

`Write` 도구로 `.omc/plans/code-audit-{slug}.md`에 저장한다.
- `{slug}`는 대상 경로를 kebab-case로 변환 (예: `backend-services-debate`)

### Step 3-3: 핸드오프

`AskUserQuestion`으로 후속 작업을 선택받는다:

1. **"추가 감사 진행"** — 다른 파일/디렉토리 감사
2. **"테스트 실행"** — 수정된 코드에 대한 테스트 실행 (`test-runner` 에이전트)
3. **"완료"** — 감사 세션 종료

</Steps>

<Tool_Usage>
- **AskUserQuestion**: 모든 납득 확인 및 선택에 사용. 하드 블락의 핵심 도구. 한 번에 하나의 질문만.
- **Glob**: 대상 디렉토리 파일 목록 확인
- **Grep**: 보안 패턴 검색, import 관계, 함수 정의, 코드 스멜 감지
- **Read**: Before 코드 블록 생성 (전체 함수/클래스, 생략 없이)
- **Edit**: 승인된 이슈의 코드 수정
- **Write**: 감사 보고서 저장 (`.omc/plans/`)
- **state_write / state_read**: 세션 저장/복원

코드 스캔 원칙:
- 개발자에게 묻기 전에 도구로 확인할 수 있는 사실은 반드시 먼저 확인한다
- Before 코드는 반드시 Read 도구로 읽은 실제 코드를 사용한다 (추측 금지)
- 보안 이슈는 실제 공격 시나리오를 예시로 설명한다
- 성능 이슈는 구체적 측정 가능한 영향으로 설명한다 ("느릴 수 있음" 대신 "N+1 쿼리로 100개 항목 조회 시 DB 호출 101회 발생")
</Tool_Usage>

<Examples>

<Good>
SQL 인젝션 이슈 제시 (실제 코드 블록 포함):
```
이슈 [2/5] [HIGH] 보안 — db.py:23-28

문제 위치: db.py:23-28

--- Before ---
def find_user(username: str):
    query = f"SELECT * FROM users WHERE username = '{username}'"
    return db.execute(query)

--- After (제안) ---
def find_user(username: str):
    query = "SELECT * FROM users WHERE username = :username"
    return db.execute(query, {"username": username})

문제: f-string SQL은 username에 "' OR '1'='1"이 들어오면
모든 사용자 데이터를 반환합니다. 파라미터 바인딩이 이를 차단합니다.
```
Why good: 실제 코드 블록, 구체적 공격 예시, 명확한 Before/After.
</Good>

<Good>
거짓양성 1회 처리:
```
사용자: "거짓양성입니다. 이 코드는 내부 API라 외부 접근 없습니다."
Claude: "내부 API도 SSRF(서버사이드 요청 위조)나 내부 공격자의 타깃이 됩니다.
실제로 2020년 Twitter 내부 해킹 사례에서 내부 도구의 권한 검증 부재가
원인이었습니다. 이 점을 고려해도 수정이 불필요하다고 판단하시나요?"
```
Why good: 사용자 주장을 무시하지 않고, 추가 근거로 재설명한다.
</Good>

<Good>
N+1 쿼리 이슈 구체적 수치 제시:
```
문제: engine.py:156-170의 루프에서 매 반복마다 DB를 호출합니다.
에이전트 100개 조회 시 DB 호출 101회 → 에이전트 1000개면 1001회.
현재 코드에서 match 목록 조회 시 실측 쿼리 수를 확인해보면:
[Grep 결과: loop 안에 select() 호출 3곳]
```
Why good: "느릴 수 있음" 대신 측정 가능한 수치를 제시한다.
</Good>

<Bad>
추측으로 이슈 설명:
```
이 코드가 좀 느릴 수 있습니다.
아마 캐싱을 추가하면 좋을 것 같습니다.
```
Why bad: Read 도구로 실제 코드를 확인하지 않고 추측했다. 구체적 근거가 없다.
</Bad>

<Bad>
건너뛰기 선택지 포함:
```
이슈를 어떻게 하시겠습니까?
1. 수정하겠습니다.
2. 거짓양성입니다.
3. 이건 나중에.  <-- 절대 금지
4. 스킵.         <-- 절대 금지
```
Why bad: "나중에"나 "스킵" 옵션은 이해 없는 무시를 허용한다.
</Bad>

</Examples>

<Escalation_And_Stop_Conditions>

### 이슈당 라운드 캡

| 심각도 | soft cap (경고) | hard cap (미해결 처리) |
|--------|-----------------|----------------------|
| HIGH   | 5 라운드        | 8 라운드             |
| MEDIUM | 3 라운드        | 6 라운드             |
| LOW    | 2 라운드        | 4 라운드             |

### 거짓양성 cap

- 3회 반복 주장 시 "사용자 판단 승인"으로 처리하고 다음 이슈

### 스캔 범위 제한

- **파일 50개 초과:** `AskUserQuestion`으로 범위 재조정 제안
- **총 이슈 30개 초과:** "이슈가 많습니다. HIGH 이슈부터 진행하고 MEDIUM/LOW는 이후 세션에서 할까요?" 제안

### 중단 조건

- **사용자가 "중단", "취소", "그만":** 즉시 중단, 현재 상태를 `state_write`로 저장
- **사용자가 "HIGH만 해줘":** 현재 세션을 HIGH 이슈로 필터링하여 진행

### 세션 재개

- `/code-audit-interview`를 다시 호출하면 `state_read`로 이전 상태 확인
- 이전 세션이 있으면 재개 여부를 묻고, 마지막 ACTIVE 이슈부터 Q&A 재개

</Escalation_And_Stop_Conditions>

<Final_Checklist>
- [ ] Phase 1에서 대상 경로를 Glob/Grep/Read로 스캔했는가
- [ ] 4개 차원(품질/보안/아키텍처/성능) 모두 스캔했는가
- [ ] 이슈 목록이 심각도 순으로 정렬되어 있는가
- [ ] 각 이슈에 Before 실제 코드 블록 (Read 도구 결과, 생략 없음)이 포함되었는가
- [ ] 각 이슈에 After 제안 코드가 포함되었는가
- [ ] 건너뛰기 선택지가 없는가
- [ ] 거짓양성 3회 cap 후 "사용자 판단 승인"으로 처리했는가
- [ ] 승인된 이슈가 즉시 코드로 수정되었는가
- [ ] 보고서가 `.omc/plans/code-audit-{slug}.md`에 생성되었는가
- [ ] 세션 상태가 `state_write`로 저장되었는가
</Final_Checklist>

<Advanced>

## 세션 상태 구조

```json
{
  "active": true,
  "current_phase": "code-audit-interview",
  "state": {
    "audit_id": "<uuid>",
    "target": "<대상 경로>",
    "current_issue_index": 2,
    "issues": [
      {
        "id": 1,
        "severity": "HIGH",
        "dimension": "security",
        "file": "auth.py",
        "lines": "45-52",
        "title": "하드코딩된 JWT 시크릿",
        "status": "FIXED",
        "round": 1,
        "fix_summary": "환경 변수로 이동"
      },
      {
        "id": 2,
        "severity": "HIGH",
        "dimension": "security",
        "file": "db.py",
        "lines": "23-28",
        "title": "SQL 인젝션 패턴",
        "status": "ACTIVE",
        "round": 2,
        "false_positive_count": 1
      }
    ],
    "audit_log": [
      {
        "issue_id": 1,
        "round": 1,
        "action": "APPROVED",
        "user_response": "수정하겠습니다"
      }
    ]
  }
}
```

## refactor-interview와의 비교

```
refactor-interview:
  - 사용자가 "무엇을 바꿀지" 이미 알고 시작
  - 변경 계획 → Q&A로 납득 → 실행
  - 이슈는 미리 알려진 리팩토링 대상

code-audit-interview:
  - Claude가 "무엇이 문제인지" 먼저 발견
  - 스캔 → 발견 → Q&A로 납득 → 수정
  - 이슈는 스캔 중 새로 발견됨

공통:
  - 하드 블락 Q&A
  - 라운드 캡 (심각도별)
  - 진행 표시기
  - state_write/state_read 세션 저장
  - Before/After 실제 코드 블록
```

## 이슈 발견 힌트 (Grep 패턴)

```bash
# 보안: 하드코딩 시크릿
grep -rn "password\s*=\s*['\"]" --include="*.py"
grep -rn "secret\s*=\s*['\"]" --include="*.py"
grep -rn "api_key\s*=\s*['\"]" --include="*.py"

# 보안: SQL 인젝션
grep -rn "f\"SELECT\|f'SELECT\|% (.*)" --include="*.py"

# 성능: 루프 안 쿼리
grep -n "for.*:\s*$" -A5 --include="*.py" | grep "db\.\|select\|execute"

# 품질: 큰 함수 (간접 지표)
grep -n "^def \|^    def " --include="*.py" | head -100
```

</Advanced>

Task: {{ARGUMENTS}}
