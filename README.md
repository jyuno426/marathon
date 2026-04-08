# Marathon

Long-running autonomous development harness for Claude Code.

Multi-agent orchestration patterns that go the distance.

## Quick Start

```bash
# 로컬 테스트
claude --plugin-dir /path/to/marathon

# 사용
/marathon:pge "할일 관리 앱을 만들어줘. React + Vite로."
```

## Patterns

### PGE (Planner → Generator → Evaluator)

3-agent loop: 프롬프트를 스펙으로 확장하고, 코드를 구현하고, E2E 테스트로 검증하여 품질 기준을 충족할 때까지 반복.

```
/marathon:pge "블로그 플랫폼. Next.js, MDX 지원, 다크모드"
```

**흐름:**
1. **Planner** (Sonnet) — 프롬프트를 상세 스펙으로 확장 → `.harness/spec.md`
2. **Generator** (Opus) — 스펙 기반 전체 구현
3. **Evaluator** (Sonnet) — 실제 앱 실행 + E2E 테스트 → `.harness/qa_report.md`
4. FAIL이면 피드백 반영하여 반복 (최대 5라운드)

**특징:**
- Git worktree로 격리된 실행 환경
- 라운드마다 git commit 체크포인트
- 구조화된 점수 체계 (가중치별 threshold)
- Human-in-the-loop (스펙 확인, 라운드 승인)

## Profile (선택)

프로젝트에 `.harness/profile.yaml`을 두면 설정을 커스터마이즈할 수 있습니다:

```yaml
language: ko
mode: greenfield  # greenfield | extend | fix
stack:
  frontend: "Next.js 15, TypeScript, Tailwind CSS"
  backend: "Next.js API Routes"
evaluator:
  score_threshold: 7
limits:
  max_rounds: 5
```

없으면 디폴트로 실행됩니다.

## Install

### Local (개발/테스트)
```bash
git clone https://github.com/jyuno426/marathon.git
claude --plugin-dir ./marathon
```

### Marketplace (예정)
```
/plugin install marathon
```

## License

MIT
