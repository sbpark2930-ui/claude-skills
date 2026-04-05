# claude-skills

[oh-my-claudecode](https://github.com/sgwoonjung/oh-my-claudecode) 전용 스킬 모음입니다.

## 설치

```bash
npx skills add sbpark2930-ui/claude-skills --yes --global
```

> **전제 조건:** oh-my-claudecode가 설치되어 있어야 합니다.

---

## 스킬 목록

### `/code-walkthrough`

코드베이스의 특정 진입 함수(또는 파일)에서 시작해 콜 그래프를 추적하고 Mermaid 다이어그램으로 전체 흐름을 시각화한 뒤, 함수 하나씩 AI가 설명하고 사용자가 Q&A하는 인터랙티브 코드 워크스루 스킬.

**사용 예시:**
```
/code-walkthrough backend/app/services/engine.py:run_match
/code-walkthrough backend/app/main.py --comprehension-score
/code-walkthrough  (진입점 후보 자동 탐지)
```

**특징:**
- 코드를 절대 수정하지 않음 — 이해만이 목적
- Glob/Grep/Read로 실제 코드를 스캔하여 콜 그래프 추적 (추측 금지)
- tldraw → Figma MCP → Mermaid 순으로 폴백하여 시각화
- 각 함수마다 하드 블락 Q&A — "다음" 선택 없이 진행 불가
- `.omc/walkthroughs/`에 이해 요약 문서 저장

**v2 고도화:**
- `--comprehension-score`: 함수별 이해도 3단계 스코어링(명확/질문있음/이해불가) + 히트맵 생성
- 북마크 기능: "여기 중요" → 중요 함수 메모와 함께 저장, 세션 종료 시 북마크 요약
- Conditional Branch 추적: if/else, try/catch 분기점에서 "정상 경로" vs "에러 경로" 선택
- 스킬 핸드오프: 워크스루 중 문제 발견 시 code-audit-interview로 즉시 전환

---

### `/code-audit-interview`

코드베이스를 4개 차원(품질/보안/아키텍처/성능)으로 심층 스캔하고, 발견된 이슈를 하나씩 Q&A로 납득시킨 뒤 승인된 항목을 코드 수정까지 완료하는 코드 감사 스킬.

**사용 예시:**
```
/code-audit-interview backend/app/services/
/code-audit-interview backend/app/api/auth.py --incremental
/code-audit-interview backend/ --focus security
```

**특징:**
- 4차원 자동 스캔: 코드 품질 / 보안 취약점 / 아키텍처 / 성능
- 이슈마다 Before/After 실제 코드 블록 + 문제 설명 제시
- 하드 블락: "수정하겠습니다" 전까지 다음 이슈 진행 불가
- 거짓양성 3회 주장 시 "사용자 판단 승인"으로 처리
- 승인된 이슈 즉시 코드 수정
- `.omc/plans/code-audit-{slug}.md`에 감사 보고서 저장

**v2 고도화:**
- `--incremental`: 증분 감사 — 이전 감사 결과와 비교하여 신규 이슈만 표시, 거짓양성 패턴 자동 필터
- `--focus <dimension>`: 차원별 포커스 (security/performance/quality/architecture) — 해당 차원 이슈를 상단 배치
- 커스텀 룰셋: `.omc/audit-rules.json`에 ignore 패턴, 커스텀 규칙, 심각도 오버라이드 정의
- 심각도 자동 조정: 테스트 있으면 1단계 하향, 최근 7일 내 변경이면 1단계 상향

---

### `/refactor-interview`

기존 코드를 스캔하여 현황 파악 후 모듈별 Q&A로 변경사항을 납득한 뒤 적용하는 리팩토링 인터뷰 스킬.

**사용 예시:**
```
/refactor-interview backend/app/services/engine.py
/refactor-interview backend/app/services/ --phased
```

**특징:**
- 코드 자동 스캔 후 현황 리포트 생성 (파일 크기, 함수 수, 코드 스멜, 의존 관계)
- 리팩토링을 모듈 단위로 분해 (추출/이동/통합/인터페이스 변경/삭제)
- 모듈별 Before/After 코드 블록 + 대안 비교 제시
- 납득 전 다음 모듈 진행 불가 (하드 블락)
- 백트래킹 지원 (이전 모듈로 돌아가 결정 수정)
- 영향 범위 분석: 변경 파일, 영향 테스트, 롤백 전략
- 완료 후 ralph/team/autopilot으로 실행 핸드오프
- `.omc/plans/refactor-interview-{slug}.md`에 계획 문서 저장

**v2 고도화:**
- 안전성 점수 (0~10): 테스트 커버리지, 참조 횟수, 최근 변경일 기반 — 낮은 점수 모듈은 추가 확인 강제
- 상세 롤백 시나리오: 모듈별 git revert 명령어 자동 생성, 독립/의존 모듈 구분
- 영향 범위 Mermaid 시각화: 빨강(직접 수정), 주황(import 변경), 노랑(테스트 변경)
- `--phased`: 3단계 마이그레이션 계획 (Phase A: 인터페이스 추출 → Phase B: 구현 이동 → Phase C: 레거시 제거)

---

### `/auto-walkthrough`

코드베이스 콜 그래프를 4에이전트(Execution Tracer, Failure Hunter, Proposal Writer, Interface Auditor)가 자동으로 리뷰하고, 토론을 통해 합의된 수정안을 문서로 제시한 뒤 사용자 확인 후 코드를 실제로 수정하는 자동 리뷰-협의-수정 스킬.

**사용 예시:**
```
/auto-walkthrough backend/app/services/engine.py:run_match
/auto-walkthrough backend/app/services/engine.py --sort severity
/auto-walkthrough backend/app/services/ --rounds 5
```

**특징:**
- 4에이전트 병렬 독립 리뷰 → 역할 경계 위반 감지 → 토론 → 만장일치 합의
- 코드 수정은 사용자 승인 후에만 실행 (Phase 3까지 수정 금지)
- git blame/log로 변경 이력 수집, 조건부 diff로 과거 실수 반복 방지
- `.omc/walkthroughs/auto-{slug}-summary.md`에 요약 보고서 저장

**v2 고도화:**
- `--sort severity`: 수정 예약 목록을 심각도 순으로 정렬
- 선택적 승인: "1,3,5번만 수정" 같이 개별 항목 승인/거부
- 테스트 커버리지 사전 확인: 테스트 없는 함수에 `[테스트 없음]` 태그, 수정 시 테스트 작성 제안
- 토론 압축 뷰: 합의 포인트 + 대립 포인트만 추출한 요약

---

### `/plan-interview`

프로젝트를 3~7개 모듈로 분해하고, 각 모듈의 설계를 하나씩 납득시킨 뒤 실행 가능한 계획 문서를 생성하는 설계 인터뷰 스킬.

**사용 예시:**
```
/plan-interview "실시간 채팅 기능 추가"
/plan-interview .omc/specs/deep-interview-chat.md  (spec 파일 입력)
```

**특징:**
- deep-interview가 "무엇을 만들지"라면, plan-interview는 "어떻게 만들지"
- 모듈별 복잡도(LOW/MEDIUM/HIGH)에 따라 Q&A 깊이 자동 조절
- 납득 전 다음 모듈 진행 불가 (하드 블락), 건너뛰기 없음
- 백트래킹 지원, 라운드 캡 (soft/hard)
- `.omc/plans/plan-interview-{slug}.md`에 계획 문서 저장

**v2 고도화:**
- 템플릿 모듈 감지: CRUD API, 인증/JWT, 상태관리, DB 마이그레이션 패턴 자동 인식 → 빠른 확인 모드
- 인터페이스 계약: 모듈별 input/output을 TypeScript 타입으로 정의, 모듈 간 호환성 검증
- 모듈 의존 그래프: Mermaid로 의존 관계 시각화 + 위상 정렬 기반 구현 순서 자동 제안
- 복잡도 추적 테이블: 예상 vs 실제 난이도 기록 (구현 후 피드백 루프용)

---

### `/harness-designer`

프로젝트를 4개 영역으로 스캔하고, settings.json(hooks, permissions, env, model, plugins)의 최적 설정을 진단/제안/승인/적용/검증하는 하네스 설계 스킬.

**사용 예시:**
```
/harness-designer
/harness-designer --preset nextjs
/harness-designer --check
/harness-designer --global --interactive
```

**특징:**
- 4영역 자동 스캔: 프로젝트 구조, 보안 패턴, 기존 설정, 워크플로우
- 리스크 기반 승인: LOW=일괄, MEDIUM/HIGH=항목별
- deepMerge 정렬 병합 + 정적 검증 (hook 실행 금지)
- 적용 전 백업, 드라이런 diff 표시, 롤백 옵션

**v2 고도화:**
- `--preset <type>`: 프리셋 5종 (nextjs, python-fastapi, rust-tauri, monorepo, minimal)
- `--check`: Drift 감지 모드 — 현재 설정과 코드베이스 상태 비교, 불일치 리포트
- 팀/개인 설정 자동 분류: `.claude/settings.json` (팀 공유) vs `.claude/settings.local.json` (개인)
- `--test-hooks`: 임시 디렉토리에서 hook 안전 테스트 (프로젝트 파일 미접촉)

---

### `/deep-dive`

3-lane trace(원인 조사) → deep-interview(요구사항 결정화)를 연결하는 2단계 파이프라인 스킬.

**사용 예시:**
```
/deep-dive "프로덕션 DAG가 간헐적으로 실패한다"
/deep-dive "인증 흐름을 개선하고 싶다" --lanes 4
```

**특징:**
- trace: N개 병렬 가설 lane으로 원인 조사, 근거 수집, 반박 라운드
- 3-point injection: trace 결과를 interview 초기화에 주입 (중복 탐사 방지)
- deep-interview 프로토콜 참조 (복제 아님)
- `.omc/specs/deep-dive-{slug}.md`에 spec 저장

**v2 고도화:**
- `--lanes N`: 적응형 lane 수 (2~5) — 단순 문제 2 lane, 분산/비동기 이슈 5 lane
- Lane 완료 시 중간 보고: 핵심 발견 + 신뢰도 즉시 출력
- 이전 trace 참조: 동일 영역의 과거 trace를 자동 탐색하여 중복 조사 방지
- 증거 품질 스코어링 (0.0~1.0): 코드 직접 확인 1.0, 가설 0.2 — 약한 증거 lane 경고

---

### `/deep-interview`

소크라테스식 Q&A로 모호한 아이디어를 명확한 spec으로 결정화하는 인터뷰 스킬. 수학적 ambiguity gating으로 준비도를 측정한다.

**사용 예시:**
```
/deep-interview "작업 관리 앱을 만들고 싶다"
/deep-interview --quick "로그인 기능 추가"
```

**특징:**
- 한 번에 하나의 질문, 가장 약한 차원 타겟팅
- Ambiguity scoring: Goal 40% + Constraints 30% + Criteria 30%
- Challenge agents: Contrarian(R4), Simplifier(R6), Ontologist(R8)
- 20% 이하로 떨어지면 spec 생성, 실행 브릿지 (ralplan/autopilot/ralph/team)

**v2 고도화:**
- 응답 품질 감지: "아무거나", "모르겠다" 같은 low-quality 응답 자동 감지 → 차원 전환 + 재시도
- 시나리오 기반 질문 모드: 추상적 질문 대신 "사용자 A가 X를 하면..." 같은 구체적 시나리오
- Round 10+ 페르소나 검증: 최종 사용자/운영팀/보안팀 관점에서 spec gap 탐지
- Ontology ER 다이어그램: entity 3개 이상 시 Mermaid erDiagram 자동 생성 + 매 라운드 업데이트
- Spec 버전 스냅샷: Round 5/10/15 자동 저장 + 방향 전환 시 자동 저장, "이전 버전과 비교해줘" 지원

---

### Learned Skills (omc-learned/)

#### `figma-module-diagram`
코드베이스의 모듈/클래스/메서드 구조를 FigJam 플로우차트로 ���성. `ClassName.method()` 형식으로 정확한 코드 위치를 표기.

**v2 고도화:** `--depth N` 탐색 깊이 제한, `--diff` git 변경 파일 강조, 2-Level 줌 (개요→상세), tldraw 폴백 체인

#### `code-walkthrough` (learned)
다이어그램이나 모듈 구조를 기준으로 코드 흐름을 순서대로 설명하고, 사용자와 문답하며 문제를 발견하고 즉시 개선하는 인터랙티브 리뷰 세션.

**v2 고도화:** 이해도 3단계 스코어링, 북마크 & 메모, conditional branch 분기 추적, auto-walkthrough/audit 핸드오프

---

## 스킬 관계도

```
이해하기                    설계하기                    실행하기
─────────                  ─────────                  ─────────
code-walkthrough           deep-interview             autopilot
  │ (이해 완료)              │ (spec 완성)              ralph
  ├→ code-audit-interview   ├→ plan-interview          team
  │   (이슈 발견+수정)       │   (설계 납득)
  └→ auto-walkthrough       ├→ deep-dive
      (4에이전트 자동 리뷰)    │   (trace→interview)
                            └→ harness-designer
                                (settings 설계)

조사하기                    리팩토링하기
─────────                  ─────────
deep-dive                  refactor-interview
  (trace → interview)        (현황 스캔 → 모듈별 납득)
```

| 스킬 | 목적 | 코드 수정 | 시작점 |
|------|------|-----------|--------|
| code-walkthrough | 이해 | - | 진입 함수/파일 |
| auto-walkthrough | 자동 리뷰 + 수정 | O (승인 후) | 진입 함수/파일 |
| code-audit-interview | 이슈 발견 + 수정 | O | 파일/디렉토리 |
| refactor-interview | 리팩토링 납득 + 실행 | O | 파일/디렉토리 |
| plan-interview | 설계 납득 | - | 프로젝트 설명/spec |
| deep-interview | 요구사항 결정화 | - | 모호한 아이디어 |
| deep-dive | 원인 조사 → 요구사항 | - | 문제/탐색 대상 |
| harness-designer | settings 설계 | O (settings) | 프로젝트 |

---

## 공통 메커니즘

모든 인터뷰 스킬이 공유하는 핵심 패턴:

- **하드 블락 Q&A**: 납득/확인 없이 다음 단계 진행 불가
- **건너뛰기 없음**: "나중에", "스킵" 선택지가 존재하지 않음
- **라운드 캡**: 복잡도별 soft cap (경고) + hard cap (강제 진행 + 위험 표시)
- **세션 저장/재개**: `state_write`/`state_read`로 중단 후 재개 가능
- **실행 핸드오프**: 완료 후 ralph/team/autopilot 등으로 Skill() 호출
