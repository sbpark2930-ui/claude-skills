---
name: refactor-interview
description: 기존 코드를 스캔하여 현황 파악 후 모듈별 Q&A로 변경사항을 납득한 뒤 적용
argument-hint: "<리팩토링 대상 파일/모듈/디렉토리>"
---

<Purpose>
Refactor Interview는 기존 코드를 자동 스캔하여 현황을 파악하고, 리팩토링 변경 사항을 모듈 단위로 설명하여 개발자가 모든 변경을 납득한 뒤에만 실행으로 넘어가는 인터뷰 스킬이다. plan-interview가 "새로 어떻게 만들지"를 납득시킨다면, refactor-interview는 "기존 코드를 왜/어떻게 바꾸는지"를 납득시킨다.

핵심 메커니즘:
1. 대상 코드를 Glob/Grep/Read로 자동 스캔하여 현황 리포트를 생성한다.
2. 리팩토링을 논리적 모듈(독립적으로 변경+테스트 가능한 단위)로 분해한다.
3. 각 모듈에 복잡도(LOW/MEDIUM/HIGH)를 부여하고 Before/After를 보여주며 Q&A한다.
4. 모듈마다 하드 블락을 건다 -- "납득했습니다" 전에는 절대 다음 모듈로 넘어가지 않는다.
5. "건너뛰기", "나중에", "스킵" 옵션은 존재하지 않는다.
6. 영향 범위 분석(테스트, 의존 관계, 롤백)까지 확인한 뒤 계획 문서를 생성한다.

출력: `.omc/plans/refactor-interview-{slug}.md`
</Purpose>

<Use_When>
- 기존 코드를 리팩토링하기 전에 변경 사항을 이해하고 싶을 때
- "refactor interview", "리팩토링 인터뷰", "코드 정리 전에 설명해줘" 같은 요청이 들어올 때
- 큰 파일을 분리하거나, 모듈 구조를 변경하거나, 인터페이스를 재설계할 때
- 리팩토링의 영향 범위를 파악하고 안전한 변경 순서를 확인하고 싶을 때
- 코드 어시스턴트가 제안한 리팩토링을 이해하지 못하고 넘기는 상황을 방지하고 싶을 때
</Use_When>

<Do_Not_Use_When>
- 리팩토링 범위가 명확하고 변경 사항을 이미 이해하고 있을 때 -- executor로 직접 실행하세요
- 단순 이름 변경이나 포맷팅 같은 기계적 변경 -- ast_grep_replace 또는 직접 실행하세요
- 새 기능을 설계할 때 -- plan-interview를 사용하세요
- 사용자가 "그냥 리팩토링해줘", "질문 없이 진행해" 라고 명시한 경우 -- 의사를 존중하세요
</Do_Not_Use_When>

<Why_This_Exists>
AI 코드 어시스턴트는 리팩토링 계획을 즉시 만들 수 있다. 문제는 개발자가 "왜 이 코드가 이렇게 바뀌는지" 이해하지 못한 채 "좋아 보이네요"라고 넘기는 것이다. 나중에 의도치 않은 동작 변경이나 회귀 버그가 발생해도 원인을 추적할 수 없다.

Refactor Interview는 이 문제를 구조적으로 해결한다:
- 코드를 먼저 스캔하여 현황을 객관적으로 파악한다.
- 각 변경 모듈의 Before/After를 구체적으로 보여주고, 왜 이 변경이 필요한지 설명한다.
- 납득 전에는 물리적으로 다음 모듈로 진행할 수 없다 (하드 블락).
- 영향 범위를 분석하여 어떤 테스트가 깨질 수 있는지, 롤백 방법은 무엇인지 명시한다.
</Why_This_Exists>

<Execution_Policy>
- 코드를 먼저 스캔한다 -- 개발자에게 묻기 전에 Glob/Grep/Read로 사실을 확인한다
- 한 번에 하나의 모듈만 다룬다 -- 여러 변경을 동시에 설명하지 않는다
- Before/After를 구체적으로 보여준다 -- Read 도구로 읽은 실제 코드 블록 (100줄 이상도 생략 없이)
- 모듈 납득 없이 다음 모듈로 진행하지 않는다 (하드 블락)
- "건너뛰기" 옵션은 절대 제공하지 않는다
- 복잡도에 맞게 Q&A 깊이를 조절한다 (LOW는 간략, HIGH는 심층)
- 라운드 캡을 준수한다 -- soft cap에서 경고, hard cap에서 강제 진행 + 위험 표시
- 대화 중에는 텍스트 진행 표시기를 사용한다 -- Mermaid는 최종 계획 문서에만 포함
- 인터뷰 상태를 `state_write`로 저장하여 세션 중단/재개를 지원한다
- 영향 범위(깨질 테스트, 의존 파일, 롤백 방법)를 반드시 분석한다
</Execution_Policy>

<Steps>

## Phase 1: 코드베이스 스캔 및 현황 파악

Glob/Grep/Read로 대상 코드를 탐색하여 현황 리포트를 생성한다:

```
코드 현황 리포트:

대상 범위: {directory_or_file}
총 파일: {count}개
총 줄 수: {total_lines}줄

파일별 요약:
- engine.py: 1,716줄, 15개 함수, 3개 클래스

발견된 문제점:
1. engine.py가 1,716줄로 과대 (신 파일)
2. _execute_debate()와 _execute_multi()에 중복 로직 존재
```

`AskUserQuestion`으로 스캔 범위를 확인받는다.

---

## Phase 2: 리팩토링 모듈 분해

리팩토링을 논리적 단위(모듈)로 분해하고 복잡도를 부여한다:

**변경 유형:**
- **추출 (Extract)**: 큰 함수/클래스에서 분리
- **이동 (Move)**: 책임 재배치
- **통합 (Merge)**: 중복 코드 합치기
- **인터페이스 변경 (Interface)**: API 시그니처 수정
- **삭제 (Delete)**: dead code 제거

**복잡도:**
| 복잡도 | 기준 | Q&A 깊이 |
|--------|------|----------|
| **LOW** | 단순 이름 변경, dead code 삭제 | 간략 (1~2 라운드) |
| **MEDIUM** | 함수 추출, 책임 이동 | 표준 (최대 5 라운드) |
| **HIGH** | 인터페이스 변경, 대규모 구조 변경 | 심층 (최대 10 라운드) |

---

## Phase 3: 모듈별 변경 Q&A 루프

각 모듈마다:
1. 현재 코드의 문제점 (파일명, 줄 번호로 특정)
2. Before 코드 블록 (Read 도구로 읽은 실제 코드, 생략 없이)
3. After 코드 블록 (Claude가 직접 작성)
4. Why (왜 이 변경이 필요한지)
5. 대안 비교 (2개 이상일 때 A vs B)
6. 위험 요소 명시 (깨질 수 있는 동작, 영향받는 테스트, 롤백 방법)

**선택지 (5개 고정, "건너뛰기" 절대 없음):**

1. **"납득했습니다. 다음 모듈로 진행해주세요."** → CONFIRMED
2. **"아직 이해가 안 됩니다. {질문}"** → 추가 설명
3. **"대안을 다시 비교해주세요."** → A vs B 재제시
4. **"변경 계획을 수정해주세요: {변경 요청}"** → 계획 수정
5. **"이전 모듈 [{name}]으로 돌아가고 싶습니다."** → 백트래킹

**라운드 캡:**
| 복잡도 | soft cap (경고) | hard cap (강제 진행) |
|--------|-----------------|---------------------|
| LOW    | 2 라운드        | 4 라운드            |
| MEDIUM | 5 라운드        | 8 라운드            |
| HIGH   | 7 라운드        | 10 라운드           |

**진행 표시기:**
```
[✓] 모듈 1: engine.py 분리 [HIGH] (납득 완료, 5라운드)
[▶] 모듈 2: 중복 로직 통합 [MEDIUM] (Q&A 진행 중, 라운드 2/5)
[ ] 모듈 3: dead code 삭제 [LOW]
[!] 모듈 4: 인터페이스 정리 [MEDIUM] (위험: 8라운드 강제 진행)
```

**백트래킹:** 이전 모듈로 돌아가면 해당 모듈 CONFIRMED → ACTIVE, 이후 모듈 → PENDING 리셋

---

## Phase 4: 영향 범위 분석

- 직접 수정/신규 생성/삭제되는 파일 목록
- 영향받는 테스트 (import 경로 변경, 시그니처 변경 등)
- 안전한 변경 순서 (의존 관계 기반)
- 롤백 전략 (git 커밋 단위)

---

## Phase 5: 계획 문서 생성

`.omc/plans/refactor-interview-{slug}.md`에 저장:
- Before/After Mermaid 다이어그램
- 모듈별 변경 계획 (변경 유형, 문제점, 수용 기준)
- 변경 순서 및 영향 범위
- 전체 Q&A 인터뷰 로그

---

## Phase 6: 실행 핸드오프

`AskUserQuestion`으로 실행 방법을 선택받는다:

1. **"ralph로 실행"** → `Skill("oh-my-claudecode:ralph")` 호출
2. **"team으로 병렬 실행"** → `Skill("oh-my-claudecode:team")` 호출
3. **"autopilot으로 실행"** → `Skill("oh-my-claudecode:autopilot")` 호출
4. **"계획만 저장 (실행 보류)"** → 계획 파일 경로 출력 후 종료

**IMPORTANT:** 실행 옵션 선택 시 반드시 `Skill()`로 해당 스킬을 호출한다.

</Steps>

<Tool_Usage>
- **AskUserQuestion**: 모든 납득 확인 및 변경 선택에 사용. 하드 블락의 핵심 도구.
- **Glob**: 코드베이스 파일 구조 탐색
- **Grep**: 패턴 검색 -- import 관계, class/def 정의, 코드 스멜 감지
- **Read**: 코드 내용 확인 (Before 코드 블록 생성)
- **Write**: 최종 계획 문서를 `.omc/plans/`에 저장
- **state_write / state_read**: 인터뷰 상태 저장/복원
- **Skill()**: 실행 핸드오프 시 ralph/team/autopilot 호출
</Tool_Usage>

<Examples>

<Good>
Read 도구로 실제 코드를 읽어 Before/After 블록으로 제시:
```
모듈: "engine.py 분리" [HIGH], 라운드 1/7

현재 문제: engine.py:342-485 -- _execute_debate() 함수가 143줄로 과대

--- Before: engine.py:342-485 ---
async def _execute_debate(self, match, agent_a, agent_b, ...):
    for round_num in range(match.total_rounds):
        response_a = await self._call_llm(agent_a, ...)
        review = await self.orchestrator.review_turn(...)
        await publish_event(match.id, "turn", {...})
        ...

--- After: turn_executor.py (신규 파일) ---
class TurnExecutor:
    async def execute_turn(self, agent, prompt, ...) -> TurnResult:
        response = await self._call_llm(agent, prompt)
        review = await self.orchestrator.review_turn(response)
        await self._publish(match_id, "turn", {...})
        return TurnResult(response=response, review=review)

이유: 책임 분리로 각 모듈을 독립적으로 테스트 가능하게 만듭니다.
```
</Good>

<Bad>
건너뛰기 옵션을 포함:
```
모듈 변경을 이해하셨나요?
1. 납득했습니다.
2. 아직 이해가 안 됩니다.
3. 이건 나중에 하겠습니다.  <-- 절대 금지
```
</Bad>

</Examples>

<Escalation_And_Stop_Conditions>

- **파일 50개 초과:** 범위 재조정 권장
- **총 코드 10,000줄 초과:** 핵심 모듈 우선 처리
- **모듈 10개 초과:** 모듈 합치기 또는 단계 분리 권장
- **사용자 "중단/취소/그만":** 즉시 중단, 상태 저장

</Escalation_And_Stop_Conditions>

<Final_Checklist>
- [ ] Phase 1에서 코드 스캔이 완료되었는가 (파일 크기, 함수 수, 의존 관계, 코드 스멜)
- [ ] 모든 모듈이 CONFIRMED 또는 FORCED 상태인가
- [ ] FORCED 모듈이 있다면 계획 문서에 표기했는가
- [ ] 영향 범위 분석(Phase 4)이 완료되었는가
- [ ] 계획 문서가 `.omc/plans/refactor-interview-{slug}.md`에 생성되었는가
- [ ] 계획 문서에 Before/After Mermaid 다이어그램이 포함되었는가
- [ ] 실행 핸드오프에서 Skill()로 선택된 스킬을 호출했는가 (직접 실행 금지)
- [ ] 대화 중에 Mermaid를 출력하지 않고 텍스트 진행 표시기만 사용했는가
</Final_Checklist>

Task: {{ARGUMENTS}}
