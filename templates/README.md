---
title: 실무 템플릿 적용 순서
last-verified: 2026-10-08
verification-method: official-documentation
---

# 실무 템플릿 적용 순서

템플릿은 [단계별 실행 가이드](../docs/workflow/README.md)에 따라 **실제로 개발할 저장소**에 복사합니다. 이 문서 저장소의 루트 지침은 가이드 집필용이므로 애플리케이션 지침으로 그대로 복사하지 않습니다.

`<...>`는 프로젝트 값으로 바꾸고, 필요 없는 항목은 이유와 함께 제거합니다. 기존 파일은 덮어쓰지 않고 비교해서 합칩니다. 문서 링크만 추가했다고 에이전트가 링크 본문까지 자동으로 읽는 것은 아닙니다. `AGENTS.md`에서 필요한 파일을 읽도록 명시합니다.

| 순서 | 원본 | 대상 저장소의 기본 경로 | 작성·갱신 시점 |
|---|---|---|---|
| 1 | [AGENTS.md.template](./AGENTS.md.template) | `AGENTS.md` | 프로젝트 규칙·실제 검증 명령이 정해질 때 |
| 1 | [CLAUDE.md.template](./CLAUDE.md.template) | `CLAUDE.md` | 공통 지침을 Claude Code에 연결할 때 |
| 1 | [development-workflow.md](./development-workflow.md) | `docs/development-workflow.md` | 초기 도입, 팀 절차 변경 시 |
| 2 | [prd.md](./prd.md) | `docs/prd/<product>.md` | 문제·범위·수용 기준 정의 시 |
| 3 | [architecture.md](./architecture.md) | `docs/architecture/overview.md` | 책임·의존성·데이터 흐름 설계 시 |
| 3 | [adr.md](./adr.md) | `docs/adr/NNNN-<decision>.md` | 되돌리기 어려운 결정이나 대안 선택 시 |
| 3 | [verification-plan.md](./verification-plan.md) | `docs/verification/strategy.md` | 검증 환경·명령·판정 기준 확정 시 |
| 4 | [task-spec.md](./task-spec.md) | `docs/backlog/tasks/TASK-NNN-<slug>.md` | 실행할 작업을 분리할 때 |
| 5 | [execution-plan.md](./execution-plan.md) | `docs/plans/TASK-NNN.md` | 복잡한 Task의 구현 전·재계획 시 |
| 5 | [task-prompt.md](./task-prompt.md) | `docs/prompts/TASK-NNN.md` | 도구에 전달할 실행 지시 구성 시 |
| 6 | [verification-report.md](./verification-report.md) | `docs/verification/TASK-NNN.md` | 기준선·구현 후·리뷰 수정 후 검증 시 |
| 7 | [review-checklist.md](./review-checklist.md) | `docs/review-checklist.md` | 리뷰 기준 도입·변경 시 |
| 7 | [handoff-note.md](./handoff-note.md) | `docs/handoffs/TASK-NNN.md` | 세션 종료·도구 교체·미완료 인계 시 |
| 8 | [release-checklist.md](./release-checklist.md) | `docs/releases/<release>.md` | 실제 배포가 포함된 작업의 배포 전후 |

계약 파일은 언어·통신 방식에 맞는 OpenAPI, JSON Schema, 타입 또는 메시지 스키마로 작성합니다. 하나의 범용 계약 템플릿을 강제하지 않습니다. 구조 예제는 [TaskFlow 적용 예제](../docs/workflow/taskflow-example.md)를 참고합니다.

## 복사 후 확인

1. `AGENTS.md`의 개요·경로·명령을 실제 프로젝트 기준으로 채웁니다.
2. `CLAUDE.md`의 `@AGENTS.md`가 같은 디렉터리의 파일을 가리키는지 확인합니다.
3. 공통 절차와 현재 Task를 읽게 한 뒤, 두 도구에 적용한 규칙의 경로와 검증 명령을 설명하게 합니다.
4. 명령을 직접 실행합니다. 실패·미실행·아직 없는 명령은 통과로 기록하지 않습니다.
5. 작은 작업 한 건으로 적용을 확인한 뒤 다음 기능으로 확대합니다.
