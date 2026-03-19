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
/code-walkthrough backend/app/main.py
/code-walkthrough  (진입점 후보 자동 탐지)
```

**특징:**
- 코드를 절대 수정하지 않음 — 이해만이 목적
- Glob/Grep/Read로 실제 코드를 스캔하여 콜 그래프 추적 (추측 금지)
- tldraw → Figma MCP → Mermaid 순으로 폴백하여 시각화
- 각 함수마다 하드 블락 Q&A — "다음" 선택 없이 진행 불가
- `.omc/walkthroughs/`에 이해 요약 문서 저장

---

### `/code-audit-interview`

코드베이스를 4개 차원(품질/보안/아키텍처/성능)으로 심층 스캔하고, 발견된 이슈를 하나씩 Q&A로 납득시킨 뒤 승인된 항목을 코드 수정까지 완료하는 코드 감사 스킬.

**사용 예시:**
```
/code-audit-interview backend/app/services/
/code-audit-interview backend/app/api/auth.py
```

**특징:**
- 4차원 자동 스캔: 코드 품질 / 보안 취약점 / 아키텍처 / 성능
- 이슈마다 Before/After 실제 코드 블록 + 문제 설명 제시
- 하드 블락: "수정하겠습니다" 전까지 다음 이슈 진행 불가
- 거짓양성 3회 주장 시 "사용자 판단 승인"으로 처리
- 승인된 이슈 즉시 코드 수정
- `.omc/plans/code-audit-{slug}.md`에 감사 보고서 저장

---

### `/refactor-interview`

기존 코드를 스캔하여 현황 파악 후 모듈별 Q&A로 변경사항을 납득한 뒤 적용하는 리팩토링 인터뷰 스킬.

**사용 예시:**
```
/refactor-interview backend/app/services/engine.py
/refactor-interview backend/app/services/
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

---

## 3가지 스킬의 관계

```
code-walkthrough     →  코드를 이해한다 (수정 없음)
        ↓ (이해 완료)
code-audit-interview →  이슈를 발견하고 수정한다 (Claude가 문제 탐지)
refactor-interview   →  이미 아는 변경을 납득하고 실행한다 (사용자가 방향 지정)
```

| 스킬 | 목적 | 코드 수정 | 시작점 |
|------|------|-----------|--------|
| code-walkthrough | 이해 | ❌ | 진입 함수/파일 |
| code-audit-interview | 이슈 발견 + 수정 | ✅ | 파일/디렉토리 |
| refactor-interview | 리팩토링 납득 + 실행 | ✅ | 파일/디렉토리 |
