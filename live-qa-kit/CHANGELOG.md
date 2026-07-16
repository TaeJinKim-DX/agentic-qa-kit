# CHANGELOG — Live QA Kit

> **`kit_version`의 정본.** `qa-config.yaml`의 `meta.kit_version`·`STATE.md`의 버전 항목은
> 이 파일 최신 버전과 대조해 드리프트를 감지한다(불일치 시 전체 재인터뷰 금지 —
> 바뀐 질문만 diff 인터뷰, AGENT.md §2). 킷 파일 변경 시 반드시 여기에 엔트리를 추가하고
> 버전을 올린다.

## v0.5.1 — 2026-07-16

첫 도그푸딩(가상 아닌 실 프로젝트를 대상으로 P0→P2 완주)에서 발견한 갭 5건 보강. `config_schema_version: 1`(변경 없음).

- **P1 §0.1 사용자 부재 모드 신설** — 프리빌드·도그푸딩·CI처럼 응답할 사람이 없을 때: 판단 질문은 기본값, 사실 질문은 clearance 미확보 라우팅, 게이트는 "상정-정지"(승인 자작 금지). AGENT.md §4에 포인터.
- **P2 §1.4 오라클 드리프트·바이너리 규칙** — 상위 소스(문서)가 코드보다 낡으면 코드로 승격+stale 표기 / 열 수 없는 바이너리 스펙(.xlsx 등)은 clearance로 열람 요청 + 2순위 잠정 진행.
- **evidence.md §7 시간 경계(E) 검증 기법 신설** — 저장(UTC) vs 표시(로컬) 3단 대조·경계 시각 시딩·만료 off-by-one·상대 표기 절삭(실전 timezone 버그 2건의 노하우).
- **templates/kit-feedback.template.md 신설** — AGENT.md §0·P4가 참조하던 dangling 템플릿 보강.
- **qa-config `retention.softdelete_purge`** 주석 — `cleanup_method: soft`일 때만 의미(hard 프로젝트 혼동 방지).

## v0.5.0 — 2026-07-16

초기 골격(1차 설계안 v0.3 구현). `config_schema_version: 1`.

- 진입 2문서: `AGENT.md`(에이전트 진입점 — 위치 규칙·상태 재개·페이즈 라우팅·게이트 규칙·능력 적응), `README.md`(사람용 1페이지 — 체크리스트·시작법·FAQ·용어집)
- `principles.md` — 12원칙 + 원칙별 실전 체크법
- `phases/` — P0 발견 → P1 인터뷰 → P2 산출물 → P3 유인 실행 → P4 회고·코드화 → P5 무인 운용(게이트 4개, P3↔P4 순환 안정화, 점진 코드화)
- `patterns/` — browser(도구중립 스텝 어휘 + 대화형↔Playwright 두 바인딩)·evidence(증거 스키마·redaction)·seeding(readiness 폴링·정리검증)·auth-roles(세션 상태·역할 전환)
- `templates/` — qa-config·STATE·case-inventory·runbook·run-report·retro·gate-summary·known-issues·clearance
- `example/` — 가상 프로젝트 1세트(익명, 두 바인딩 병기)
- 핵심 계약: 케이스 ID `<카테고리>-<번호>` · run 디렉터리 `runs/<env>-<run-id>/` · 증거 `evidence/<case-id>/` · 시크릿 `secret_ref`+`.env.qa` · 킬스위치 `qa/.stop` · 락 `qa/.lock`(stale 30분)
