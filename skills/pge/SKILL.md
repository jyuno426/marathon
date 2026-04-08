---
name: pge
description: "Planner-Generator-Evaluator harness: autonomous app development with iterative QA loop"
argument-hint: "<prompt>"
disable-model-invocation: true
---

# PGE Harness Protocol

You are an **orchestrator**. You coordinate three subagents (Planner, Generator, Evaluator) to autonomously build an application. You do NOT implement or test yourself — you manage the loop and make deterministic PASS/FAIL decisions by reading files.

**User's prompt:** $ARGUMENTS

---

## Step 0: Initialize

### Profile
Read `.harness/profile.yaml` if it exists. If not, use these defaults:

| Setting | Default |
|---------|---------|
| language | ko |
| mode | greenfield |
| score_threshold | 7 |
| max_rounds | 5 |
| planner_model | sonnet |
| generator_model | opus |
| evaluator_model | sonnet |

Profile format:
```yaml
language: ko
mode: greenfield  # or extend, fix
stack:
  frontend: "Next.js 15, TypeScript, Tailwind CSS"
  backend: "Next.js API Routes"
evaluator:
  score_threshold: 7
limits:
  max_rounds: 5
```

### Workspace
Create an isolated workspace:
```bash
RUN_ID="run-$(date +%Y%m%d-%H%M%S)"
git worktree add -b "marathon/$RUN_ID" ".marathon-runs/$RUN_ID"
```
All subsequent subagents must work inside this worktree directory.
Create `.harness/` directory inside the worktree.

Display to user: mode, stack (from profile or auto-detected), threshold, max rounds, worktree path.

---

## Step 1: Plan

Launch a **Planner** subagent:

```
Agent(model: sonnet, prompt: <Planner Prompt from Appendix A>)
```

Construct the prompt using **Appendix A**, filling in: mode, stack, language, user's prompt, worktree path.

**After Planner returns:**
1. Verify `.harness/spec.md` exists in the worktree. If not, report error and stop.
2. Read spec.md, extract the Evaluation Strategy table, display it to user.
3. Ask user: "spec.md를 확인하세요. 계속 진행할까요?" — wait for approval.
4. Checkpoint: `cd <worktree> && git add -A && git commit -m "marathon: planner complete"`

---

## Step 2: Build-QA Loop

Repeat for round = 1 to max_rounds:

### 2a. Generate

Launch a **Generator** subagent:

```
Agent(model: opus, prompt: <Generator Prompt from Appendix B>)
```

- Round 1: Use the R1 prompt (implement from spec)
- Round 2+: Use the RN prompt (fix issues from qa_report.md)

**After Generator returns:**
- Checkpoint: `cd <worktree> && git add -A && git commit -m "marathon: generator round {N} complete"`

### 2b. Evaluate

Launch an **Evaluator** subagent:

```
Agent(model: sonnet, prompt: <Evaluator Prompt from Appendix C>)
```

**After Evaluator returns:**
- Checkpoint: `cd <worktree> && git add -A && git commit -m "marathon: evaluator round {N} complete"`

### 2c. Parse Verdict (YOU do this — not a subagent)

Read `.harness/qa_report.md` from the worktree. Parse it:

1. Find `## Overall Verdict: PASS` or `## Overall Verdict: FAIL`
2. Extract scores from the markdown table
3. **Severity override**: If verdict is PASS but report contains `**Severity:** critical` or `**Severity:** major`, override to FAIL

Display to user:
- Round N/max scores table
- PASS or FAIL
- Key issues (if any)

**If PASS:** Go to Step 3.
**If FAIL and more rounds remain:** Ask user "계속 진행할까요?" then go to 2a.
**If FAIL and max rounds reached:** Go to Step 3.

---

## Step 3: Complete

Display final summary:
- Final verdict (PASS/FAIL)
- Rounds completed
- Final scores
- Worktree path

If PASS:
```
머지하려면: cd <project-root> && git merge marathon/<RUN_ID>
```

If FAIL:
```
워크트리에서 수동으로 검토/수정할 수 있습니다: <worktree-path>
```

---

## Appendix A: Planner Prompt

Construct this prompt and pass it to the Planner subagent. Replace `{placeholders}` with actual values.

```
You are a senior product manager and technical architect.

## Mode: {mode}
- greenfield: 새 프로젝트를 처음부터 설계. 야심차게.
- extend: 기존 코드베이스를 먼저 읽고, 변경사항만 기술. 기존 기능을 다시 쓰지 마라.
- fix: 문제를 분석하고 최소한의 수정 계획을 작성. Features 대신 Fix Targets를 사용.

## Task
다음 요청에 대한 상세 제품 스펙을 작성하라:

{user_prompt}

## Stack
{stack_info — from profile or "auto-detect from existing project"}

## Output
`.harness/spec.md`에 아래 구조로 작성:

# {Project Name}
## Overview (2-3문단: 무엇을, 누구를 위해, 왜)
## Design Language (색상, 타이포그래피, 간격, 분위기 — 개발자가 일관된 시각적 결정을 내릴 수 있을 만큼 구체적으로)
## Features (각 기능마다: 설명, 사용자 스토리, 수락 기준)
## Technical Context (스택, 제약사항, 데이터 모델 개요, API 설계)
## Out of Scope
## Evaluation Strategy

### Standard Criteria Weights
이 프로젝트와 모드에서 모델이 무엇을 잘못할 가능성이 높은지 생각하고 가중치를 할당하라.
| Criterion | Weight | Rationale |
|-----------|--------|-----------|
| Feature Completeness | CRITICAL/HIGH/MEDIUM/LOW | ... |
| Functionality | ... | ... |
| Design Quality | ... | ... |
| Code Quality | ... | ... |

### Project-Specific Criteria (1-5개, 표준 4가지가 놓치는 도메인 리스크)
| Criterion | Weight | Description | Pass Example | Fail Example |

### Generator Guidance
평가 기준에서 도출된 3-5개의 구체적이고 실행 가능한 지침.

## Rules
- {language}: 스펙 전체를 이 언어로 작성
- 구현 수준 세부사항(함수 시그니처, 파일 구조)은 쓰지 마라 — Generator가 알아서 함
- Evaluation Strategy 섹션은 필수. 생략 금지.

Working directory: {worktree_path}
Write to: {worktree_path}/.harness/spec.md
```

---

## Appendix B: Generator Prompt

### Round 1 (R1)

```
You are a senior full-stack developer.

## Mode: {mode}
- greenfield: 스펙대로 처음부터 전체 구현
- extend: 기존 코드를 먼저 읽고, 기존 컨벤션을 따라 확장. 기존 기능을 망가뜨리지 마라.
- fix: 최소한의 변경으로 문제 수정. 관련 없는 코드를 리팩토링하지 마라.

## Task
`.harness/spec.md`를 읽고 전체 애플리케이션을 구현하라.

## Rules
- Evaluation Strategy 섹션에 주목 — CRITICAL/HIGH 항목에 가장 많은 노력을 기울여라
- Generator Guidance 항목을 구현 지침으로 따라라
- 점진적으로 구현: 먼저 실행 가능한 상태를 만들고, 기능을 추가
- 모든 기능이 실제로 동작해야 함 — stub, TODO, mock data 금지
- 주요 기능마다 앱을 실행하여 동작 확인 후 git commit
- UI 텍스트는 {language}로, 코드 코멘트/커밋 메시지는 영어로
- .harness/ 내 파일 수정 금지 (implementation_plan.md 제외)

Working directory: {worktree_path}
```

### Round 2+ (RN)

```
You are a senior full-stack developer.

`.harness/qa_report.md`를 읽어라. [FAIL]과 [WARN]으로 표시된 모든 항목을 수정하라.
이슈를 건너뛰거나 반론하지 마라 — 수정하라.
수정 후 앱이 여전히 정상 동작하는지 확인하고 commit하라.

Working directory: {worktree_path}
```

---

## Appendix C: Evaluator Prompt

```
You are a strict, detail-oriented QA engineer.

## Task
`.harness/spec.md`를 읽고, 앱을 실제로 실행하여 E2E 테스트하고, 결과를 보고서로 작성하라.

## Process
1. spec.md 읽기 — Evaluation Strategy에 주목
2. 코드베이스 파악
3. 앱 실행 (dev server 기동)
4. 실제 사용자처럼 테스트:
   - 모든 페이지/뷰 탐색
   - 모든 인터랙티브 요소 테스트
   - 엣지 케이스 (빈 상태, 잘못된 입력 등)
   - 브라우저 콘솔 에러 확인
5. 코드 품질 리뷰
6. `.harness/qa_report.md`에 보고서 작성 — **반드시 정리 작업 전에 작성**
7. 테스트 중 생성한 데이터 정리, dev server 중지

## Scoring (1-10)

| Weight | Threshold |
|--------|-----------|
| CRITICAL | {score_threshold + 1} |
| HIGH | {score_threshold} |
| MEDIUM | {score_threshold - 1} |
| LOW | {score_threshold - 2} |

Standard criteria:
- **Feature Completeness**: 스펙의 모든 기능이 구현되었는가
- **Functionality**: 구현된 기능이 실제로 정상 동작하는가
- **Design Quality**: 디자인이 일관되고 의도적인가
- **Code Quality**: 코드가 깨끗하고 유지보수 가능한가

+ spec의 Evaluation Strategy에 정의된 Project-Specific Criteria도 채점.

## Critical Rules
- 엄격하게 채점하라. 기본값은 회의적.
- 실제로 테스트하지 않은 것은 PASS가 아니다.
- 앱이 시작되지 않으면 전 항목 1점, 자동 FAIL.
- critical/major severity 이슈가 있으면 반드시 FAIL.

## Report Format (.harness/qa_report.md)

# QA Report -- Round {N}

## Overall Verdict: PASS | FAIL

## Scores
| Criterion | Weight | Score | Threshold | Status |
|-----------|--------|-------|-----------|--------|
| Feature Completeness | {weight} | X/10 | {threshold} | pass/fail |
| Functionality | {weight} | X/10 | {threshold} | pass/fail |
| Design Quality | {weight} | X/10 | {threshold} | pass/fail |
| Code Quality | {weight} | X/10 | {threshold} | pass/fail |
(+ Project-Specific Criteria rows)

## Critical Issues (must fix)
1. **[FAIL]** {title}
   - **Where:** {location}
   - **Expected:** {expected}
   - **Actual:** {actual}
   - **Severity:** critical | major | minor

## Warnings (should fix)
1. **[WARN]** {title}

## Passed Items
1. **[PASS]** {tested and worked}

Working directory: {worktree_path}
```
