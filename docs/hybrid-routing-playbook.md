# Hybrid Routing Playbook

> 목적: manager-orchestrator가 Claude specialist와 Codex를 함께 사용할 때, 라우팅/핸드오프/검증/상태 업데이트를 일관되게 수행하기 위한 실행 지침.

## 1) Quick Start

1. 작업을 L0-L4로 판정한다.
2. 운영 모드(Mode A / Mode B / Mode B+Bootstrap)를 선언한다.
3. 레벨별 기본 경로를 선택한다.
4. Codex handoff가 필요하면 템플릿을 채운다.
5. `scripts/run-codex-handoff.sh`로 실행한다.
6. 결과를 git diff 기준으로 회수한다.
7. specialist 1차 검증 후 manager 최종 승인한다.
8. Claude Task 상태를 갱신한다.

모드 정의:
- Mode A: Agent Teams 기반 (`teams + tasks`)
- Mode B: Subagent Task 기반 (`tasks only`)
- Mode B+Bootstrap: P1에서 teams skeleton 생성

## 2) Level Scoring Sheet

| 항목 | 점수 가이드 |
|------|------------|
| 변경 파일 수 | 1-3: 0, 4-10: 2, 11+: 4 |
| 도메인 수(UI/API/DB/인프라) | 1: 0, 2: 2, 3+: 4 |
| 위험도 | 낮음: 0, 중간: 2, 높음: 4 |
| 검증 난이도 | smoke: 0, unit+integration: 2, e2e+security: 4 |
| 외부 영향도 | 내부만: 0, 사용자 노출: 2, 권한/결제/운영핵심: 4 |

레벨 매핑:
- 0-3점: L0
- 4-7점: L1
- 8-11점: L2
- 12-15점: L3
- 16-20점: L4

## 3) Routing Matrix

| Level | 기본 경로 | 병렬 | Codex |
|------|-----------|------|------|
| L0 | sonnet 단독 | 없음 | 사용 안 함 |
| L1 | sonnet + 빠른 검증 | 제한적 | 선택 |
| L2 | `opusplan` -> sonnet | 가능 | 권장 |
| L3 | 8-Phase full | 적극 권장 | 적극 권장 |
| L4 | L3 + 보안 게이트 | 적극 권장 | 필수 후보 |

## 4) Codex Handoff Template

```text
handoff_id: HOFF-YYYYMMDD-###
source_task_id:
level:
owner_agent:
objective:
in_scope_files:
- src/...
out_of_scope_files:
- src/...
acceptance_criteria:
- ...
verification_commands:
- npm run build
- npm test -- <scope>
expected_output_format:
- changed_files
- diff_summary
- test_results
- unresolved_risks
rollback_plan:
- git revert <commit> or reset branch to <sha>
```

## 4.1) Codex Run Command

```bash
cd /Users/jaebak/cocoding
scripts/run-codex-handoff.sh handoffs/HOFF-YYYYMMDD-001.txt .
```

실행 전 체크:
```bash
command -v codex >/dev/null && codex exec --help >/dev/null
```

## 5) Manager Prompt Skeleton (for Codex)

```text
You are executing handoff {handoff_id}.
Goal: {objective}

Constraints:
- In-scope only: {in_scope_files}
- Out-of-scope: {out_of_scope_files}
- Do not modify files outside scope.

Definition of Done:
{acceptance_criteria}

Verification commands:
{verification_commands}

Return format:
1) changed_files
2) diff_summary (1-3 lines per file)
3) test_results
4) unresolved_risks
```

## 6) Recovery Rules

- 결과 누락: 동일 handoff_id로 1회 재요청
- 테스트 실패: specialist가 원인 분류 후 재핸드오프
- 2회 연속 실패: Claude specialist direct path로 전환
- L4 반복 실패: 사용자 승인 대기 + 수동 리뷰

## 7) Agent-Monitor Sync Checklist

- 시작 시: 해당 Claude Task를 `in_progress`로 전이
- handoff 시작 시: `activeForm`을 handoff 상태로 갱신
- 결과 회수 후: `blockedBy`, `blocks` 최신화
- 최종 승인 시: Task를 `completed`로 전이

## 8) Git-Centric Retrieval Checklist

1. `codex/<handoff_id>` 브랜치 생성 여부 확인
2. `BASE=$(git merge-base main codex/<handoff_id>)`
3. `git diff --name-only "$BASE"...codex/<handoff_id>`
4. 파일별 핵심 diff 확인
5. 영향 범위 테스트 실행
6. 실패 로그 최소 수집
7. 승인/반려 결정 후 Task 반영

## 9) Example Scenario (L2)

상황:
- 대시보드 필터 기능 추가 (프론트만), 파일 5개 변경 예상
- 레벨 판정: L2
- 모드: Mode B(Subagent Task)

절차:
1. Manager가 Task를 `in_progress`로 전이
2. handoff 파일 생성 (`handoffs/HOFF-20260213-001.txt`)
3. `scripts/run-codex-handoff.sh` 실행
4. `codex/HOFF-20260213-001` 브랜치에서 결과 회수
5. frontend specialist 1차 검증
6. manager 최종 승인 후 Task `completed`
