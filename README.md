# ai-development-guide

**Claude Code와 Codex로 대규모 개발하기** — 따라하기와 숙제로 익히는 실전 가이드

작은 기능부터 수만 라인 규모의 프로젝트까지 AI 코딩 에이전트를 적용하는 방법을 다룹니다.
모든 레슨은 **개념 → 따라하기 → 숙제 → 회고** 순서로 진행합니다.
초보자는 각 모듈의 PNG 그림과 읽는 순서로 핵심 개념을 먼저 익힙니다. [21개 그림 목록](./docs/design/illustrations.md)에서 주제별 레슨과 제안서용 이미지를 찾습니다.

## 실제 개발에 바로 적용하기

먼저 [AI로 개발하는 단계별 실행 가이드](./docs/workflow/README.md)를 따릅니다. 단계별 입력·산출물·완료 조건과 Claude Code·Codex에서 사용할 프롬프트를 제공합니다.

1. **현황과 지침**: 대상 저장소를 조사하고 `AGENTS.md`·`CLAUDE.md`·공통 개발 절차를 연결합니다.
2. **요구사항과 설계**: PRD·수용 기준·디렉터리 책임·계약·필요한 ADR을 작성합니다.
3. **환경과 작업 분해**: 실제 검증 명령을 구성하고 의존성 순서로 Task를 만듭니다.
4. **구현과 검증**: Task별 탐색·계획·작은 변경·테스트를 반복하고 증거를 남깁니다.
5. **리뷰와 인계**: 변경을 교차 리뷰하고 통합·릴리스·핸드오프를 기록합니다.

| 필요 | 시작할 문서 |
|---|---|
| 처음부터 끝까지 순서대로 진행합니다 | [단계별 실행 가이드](./docs/workflow/README.md) |
| 산출물을 어떻게 채우는지 봅니다 | [TaskFlow 기능 하나의 명세·계획·검증 예제](./docs/workflow/taskflow-example.md) |
| 내 프로젝트에 두 도구의 지침을 적용합니다 | [템플릿 적용 순서](./templates/README.md)와 [공통 개발 절차](./templates/development-workflow.md) |
| 개발 요청을 정확한 프롬프트로 바꿉니다 | [02-5 개발용 프롬프트 엔지니어링](./modules/02-spec-driven-development/02-5-prompt-engineering.md) |
| OpenAI·Anthropic의 공식 근거를 확인합니다 | [공식 개발 가이드 적용 기준](./docs/reference/official-development-guides.md) |

이 저장소의 루트 지침은 **가이드 집필용**입니다. 애플리케이션을 개발할 때는 `templates/`의 지침을 대상 저장소에 적용합니다. TaskFlow 예제는 작성 예제이며, 실행 가능한 TaskFlow 서비스가 이 저장소에 포함된 것은 아닙니다.

## 구성

| 구분 | 내용 |
|---|---|
| [modules/](./modules) | 짧은 레슨 단위로 익히는 기법 (7개 모듈) |
| [projects/](./projects) | 모듈을 관통하는 장기 실습 프로젝트 (문서화 / 개발 / 레거시) |
| [templates/](./templates) | 지침, PRD, 아키텍처, ADR, Task, 프롬프트, 계획, 검증, 리뷰, 릴리스, 핸드오프 |
| [docs/workflow/](./docs/workflow) | 실제 프로젝트에 적용하는 단계별 절차와 산출물 예제 |
| [docs/design/](./docs/design) | 커리큘럼 설계서, 레슨·숙제 템플릿 |
| [docs/reference/tool-reference.md](./docs/reference/tool-reference.md) | Claude Code·Codex 명령어/설정 검증 레퍼런스 (레슨의 사실 기준) |

### 모듈
| # | 모듈 | 핵심 |
|---|---|---|
| 00 | [Foundations](./modules/00-foundations) | 에이전트 동작 원리, Claude vs Codex |
| 01 | [Environment Setup](./modules/01-environment-setup) | 메모리 파일, 권한, MCP, Skills/Hooks |
| 02 | [Spec-Driven Development](./modules/02-spec-driven-development) | PRD, ADR, 작업 분해, 인터페이스 우선, 프롬프트 엔지니어링 |
| 03 | [Agentic Workflow](./modules/03-agentic-workflow) | Plan→Implement→Verify, TDD, 컨텍스트 관리 |
| 04 | [Quality & Verification](./modules/04-quality-and-verification) | 검증 계층, 교차 리뷰, 보안 |
| 05 | [Scaling Up](./modules/05-scaling-up) | 병렬 에이전트, 오케스트레이션, CI 자동화 |
| 06 | [Team & Operations](./modules/06-team-and-operations) | 비용, 지표, 거버넌스 |

### 프로젝트
| # | 프로젝트 | 내용 |
|---|---|---|
| A | [문서화 프로젝트](./projects/A-documentation-project) | Docs-as-Code 문서 사이트 구축·운영 |
| B | [개발 프로젝트 TaskFlow](./projects/B-development-project) | 웹 + API + 워커 + CLI, 3~5만 라인 |
| C | [레거시 마이그레이션](./projects/C-legacy-migration-project) | 모르는 코드베이스의 대규모 변경 |

## 학습 경로
| 경로 | 대상 | 순서 | 기간 (주 5시간) |
|---|---|---|---|
| Fast Track | 실무자 | 01 → 02 → 03 → Project B (B0~B4) | 4주 |
| Full Track | 입문자 | 00 → 01 → A → 02 → 03 → 04 → B → 05 → 06 | 12주 |
| Lead Track | 리드/아키텍트 | 01 → 04 → 05 → 06 → Project C | 6주 |

## 숙제 제출
1. 저장소를 fork 하고 숙제마다 `hw/<모듈>-<레슨>` 브랜치를 만듭니다.
2. 결과물과 `submissions/<모듈>-<레슨>.md`([템플릿](./docs/design/homework-template.md))를 커밋합니다.
3. PR 본문에 에이전트 리뷰 결과와 자기 평가를 첨부합니다.

자세한 설계는 [커리큘럼 설계서](./docs/design/curriculum-design.md)를 참고합니다.
