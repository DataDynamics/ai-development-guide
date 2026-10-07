# AGENTS.md

이 저장소는 "Claude Code와 Codex로 대규모 개발하기" 가이드 문서 저장소이며, Project A(문서화 프로젝트)의 레퍼런스 구현이기도 하다.

## 구조
- `docs/design/curriculum-design.md` — 전체 설계서. 모든 콘텐츠의 기준 문서.
- `modules/<NN-name>/` — 레슨. 파일명 `NN-M-<slug>.md`, 형식은 `docs/design/lesson-template.md`.
- `projects/<X-name>/` — 프로젝트 마일스톤. 파일명 `<X><N>-<slug>.md`.
- `templates/` — 학습자가 복사해 쓰는 템플릿.

## 작성 규칙
- 본문은 한국어, 도구·명령어·파일명은 원문 그대로 쓴다.
- 문체는 "~한다" 체로 통일한다.
- 모든 레슨은 따라하기에 Claude Code 레시피와 Codex 레시피를 함께 제공한다.
- 도구 특화 내용에는 front matter의 `last-verified` 날짜를 갱신한다.
- 확인하지 않은 CLI 옵션이나 기능을 지어내지 않는다. 불확실하면 `TODO(verify)`로 표시한다.
- 레슨/마일스톤을 추가하면 해당 모듈·프로젝트 README의 상태 표를 갱신한다.
