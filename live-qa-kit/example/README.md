# example/ — 가상 프로젝트 예시 세트

이 폴더는 가상 프로젝트 **"TeamBoard"**(팀 게시판 + 파일 공유 웹앱 — React 프론트, REST 백엔드, PostgreSQL, 사내 SSO, 외부 스토리지 API)에 킷을 적용했을 때 각 산출물이 **채워진 모습**을 보여준다.
TeamBoard는 실존하지 않는다 — 어떤 실제 프로젝트·회사와도 무관하며, 값은 전부 예시다(시크릿은 `secret_ref` 키 이름만, 실값 없음).
템플릿(templates/)의 빈 골격과 이 예시를 나란히 놓고 보면 "무엇을 어떻게 채우는가"가 보인다.

## 예시 타임라인

P0 발견(07-06) → [게이트1] 설정 승인(07-07) → 준비물 확보(07-07~07-10) → [게이트2] 케이스 승인(07-10) → P3 유인 1차 실행(07-13, `runs/stage-20260713a/`). 각 파일은 그 시점의 스냅샷이다.

## 파일 안내

| 파일 | 무엇의 예시인가 | 대응 골격·규정 |
|---|---|---|
| discovery-report.md | P0 산출물 — 파악한 것/못 한 것·위험 후보·능력 프로파일·준비물 초안 | phases/P0-discovery.md §7 목차 |
| qa-config.yaml | P1 산출물 — [게이트1]로 확정된 계약 | templates/qa-config.template.yaml |
| clearance.md | P1 산출물 — 준비물 대장(복붙 요청 문구 포함) | templates/clearance.template.md |
| cases-sample.md | P2 케이스 인벤토리 발췌 5건(15건 중) — B-01은 두 바인딩 병기 | phases/P2-artifacts.md §1.5 |
| gate-summary-예시.md | [게이트2] 제시문 — 커버리지 요약표 + 케이스 목록 | templates/gate-summary.template.md |
| run-report-예시.md | P3 1차 run 리포트 — 신호등 🟡·실패 1건(known-issue 후보)·오탐 방지 3관문 기록 | templates/run-report.template.md |

STATE.md·runbook.md·scripts/ 등 나머지 산출물은 이 예시에 없다 — 골격은 templates/, 생성 규정은 phases/를 보라.
