# AGENTS.md

이 저장소는 "Claude Code와 Codex로 대규모 개발하기" 가이드 문서 저장소이며, Project A(문서화 프로젝트)의 레퍼런스 구현이기도 합니다.

## 구조
- `docs/design/curriculum-design.md` — 전체 설계서. 모든 콘텐츠의 기준 문서.
- `modules/<NN-name>/` — 레슨. 파일명 `NN-M-<slug>.md`, 형식은 `docs/design/lesson-template.md`.
- `projects/<X-name>/` — 프로젝트 마일스톤. 파일명 `<X><N>-<slug>.md`.
- `templates/` — 학습자가 복사해 쓰는 템플릿.
- `docs/workflow/README.md` — 실제 개발의 단계·산출물·완료 조건을 연결하는 실행 가이드.
- `docs/reference/official-development-guides.md` — 공식 권고와 이 저장소 운영 규칙의 구분.

## 작업 절차
- 가이드 콘텐츠를 바꾸기 전에 설계서와 해당 레슨·프로젝트 README를 읽습니다.
- 개발 절차·산출물·프롬프트 관련 변경은 `docs/workflow/README.md`와 `templates/README.md`의 연결도 확인합니다.
- 루트 지침은 이 문서 저장소용입니다. 학습자의 애플리케이션에는 `templates/AGENTS.md.template`, `templates/CLAUDE.md.template`, `templates/development-workflow.md`를 적용합니다.
- 공식 가이드에서 확인한 기능과 이 저장소가 제안한 운영 규칙을 구분합니다. 확인일·원문 링크·문서 확인 또는 실제 실행 여부를 남깁니다.
- 예제의 예상 결과를 실제 테스트 통과 기록으로 쓰지 않습니다. 미실행 검증은 미실행으로 보고합니다.
- 완료 전 `git diff --check`를 실행하고, 변경한 문서의 상대 링크·산출물 경로·상태 표·두 도구 레시피의 일관성을 확인합니다. 이 저장소에 없는 `make verify`를 실행했다고 보고하지 않습니다.

## 작성 규칙
- 본문은 한국어, 도구·명령어·파일명은 원문 그대로 씁니다.
- 본문·설명·프롬프트 예제의 문체는 합니다체로 통일합니다. 요청문은 “하십시오” 또는 “해 주십시오”처럼 격식 있는 높임말을 사용합니다.
- 모든 레슨은 따라하기에 Claude Code 레시피와 Codex 레시피를 함께 제공합니다.
- 도구 특화 내용에는 front matter의 `last-verified` 날짜를 갱신합니다.
- 확인하지 않은 CLI 옵션이나 기능을 지어내지 않습니다. 불확실하면 `TODO(verify)`로 표시합니다.
- 레슨/마일스톤을 추가하면 해당 모듈·프로젝트 README의 상태 표를 갱신합니다.
