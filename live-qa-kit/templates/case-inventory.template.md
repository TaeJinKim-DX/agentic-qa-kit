# case-inventory.md — 케이스 인벤토리 (P2 산출물, [게이트2] 승인 대상)

> P2(phases/P2-artifacts.md)에서 이 골격을 채워 `<프로젝트 루트>/qa/cases/case-inventory.md`로 생성한다.
> [게이트2]는 이 문서의 **커버리지 요약표 + 케이스 목록**으로 상정한다 — 비전공자가 빈 칸/과밀을
> 눈으로 판단할 수 있게 유지하라. **케이스 ID = `<카테고리>-<번호>`(A-01 식)** — runbook.md ·
> `runs/<env>-<run-id>/evidence/<case-id>/` · runner/ · known-issues.md가 상호참조하는 공통 키다.
> 한번 부여한 ID는 바꾸지 않는다(케이스 제외 시 결번으로 남긴다).

## 헤더

| 항목 | 값 |
|---|---|
| 프로젝트 | `<qa-config.yaml meta.project_name>` |
| 대상 환경 | `<qa-config environments 키 — 예: stage>` |
| 총량 / 상한 | `<n>` / `<qa-config scope.case_cap>` |
| 정렬 | 실패 우선 — A → B → C → D → E → F (원칙 3) |
| 오라클 출처 우선순위 | 기능문서 > API 스키마 > 현행동작 스냅샷(회귀 기준 — 정답 아님) |
| 상태 집계 | 미실행 `<n>` · PASS `<n>` · FAIL `<n>` · SKIP `<n>` · expected-fail `<n>` |

### 커버리지 요약표 (기능 × 카테고리)

<!-- 행 = qa-config scope.priority_features 전부. 셀 = 케이스 수(케이스 ID 병기 권장).
     커버리지 규칙: 우선 기능마다 A·B·D 최소 1케이스(phases/P2-artifacts.md). 미달 칸은 ⚠ 표시 + 사유.
     E(데이터 정합): 저장 타임존 ≠ 표시 타임존이면 시간 경계 케이스 의무(qa-config timezone). -->

| 기능 \ 카테고리 | A 실패·경계 | B 핵심 액션 | C 목록·탐색 | D 권한 | E 데이터 정합 | F 교차시스템 | 계 |
|---|---|---|---|---|---|---|---|
| `<기능 1>` | `<1 (A-01)>` | | | | | | |
| `<기능 2>` | | | | | | | |
| **계** | | | | | | | `<n>` |

## 케이스 표준 블록

<!-- 아래 형식으로 실패 우선 순서대로 나열한다. 기입 규칙:
     - 우선순위: 상 / 중 / 하 — 상 = 이번 run에서 반드시, 하 = 시간 남으면.
     - 환경 제약: 실행 가능한 environments 키. 데이터 생성·변경이 있으면 prod 금지(grade: prod = 읽기전용).
     - 오라클 출처: 스냅샷 기반이면 반드시 "회귀 기준(정답 아님)" 병기 — 버그가 정답이 되는 것 방지.
     - 상태: 미실행 / PASS / FAIL / SKIP / expected-fail — 최신 run 기준으로 갱신.
       expected-fail은 known-issues.md의 이슈 ID를 병기(등록은 [게이트3] 승인 사항 — phases/P4-retro.md).
     - 러너 편입: 편입 기준 = 유인 연속 2회 PASS + 셀렉터 무수정(phases/P4-retro.md). 무인 범위 = 편입 케이스만.
     - 바인딩 메모: P3에서 학습한 셀렉터·대기 조건을 기록(phases/P3-first-run.md) — 러너 코드화의 재료.
       runbook.md는 도구 중립을 유지하고, 바인딩 세부는 여기에만 둔다. -->

### `<ID>` — `<제목>`

| 필드 | 값 |
|---|---|
| 우선순위 | `<상 / 중 / 하>` |
| 출처 | `<기존 QA/기능 문서 / API 명세·라우트 표면 / UI 크롤링 — phases/P2-artifacts.md §1.1>` |
| 환경 제약 | `<stage 전용 / 전 환경 / prod 읽기전용 가능 …>` |
| 오라클 출처 | `<기능문서 §x / API 스키마 <엔드포인트> / 현행동작 스냅샷 — 회귀 기준(정답 아님)>` |
| 상태 | `<미실행 / PASS / FAIL / SKIP / expected-fail(<이슈 ID>)>` |
| 러너 편입 | `<미편입 / 편입(YYYY-MM-DD)>` |
| 증거 요구사항 | `<PASS 판정에 필요한 증거 파일 목록 — 예: request.json · screenshot.png · db-check.json(스키마: patterns/evidence.md)>` |

**[세팅]** — 사전 상태·데이터 (시딩 규약: patterns/seeding.md, 스크립트: scripts/)
- `<필요 역할·로그인 세션 / 시딩 리소스(test_prefix + run-id 접두) / 없음>`

**[실행]** — 무엇을 하는가 (스텝 상세는 runbook.md의 동일 ID 블록)
- `<행위 요지>`

**[실제값 검증]** — 증거 요구 명시(원칙 2 — 증거 없는 PASS는 PASS가 아니다) → `evidence/<case-id>/`
- 네트워크: `<기대 요청·상태코드·응답 body 조건>` (request.json)
- 화면: `<기대 화면 상태>` (screenshot.png)
- DB/외부 시스템: `<반영 확인 조회와 기대값>` (db-check.json)

**[정리]** — 생성 리소스 정리 + 잔여 0건 검증(원칙 5, 잔여물 대장: STATE.md)
- `<정리 대상·검증 방법 / 생성 리소스 없음 — 대장 확인만>`

**바인딩 메모** (P3에서 기록)
- `<학습한 셀렉터·대기 조건·특이사항 — 없으면 비움>`

---

## 예시 — 가상 웹앱(팀 일정 관리)의 케이스 1건

### A-01 — 일정 등록: 필수값(title) 누락 거부

| 필드 | 값 |
|---|---|
| 우선순위 | 상 |
| 출처 | API 명세 `POST /api/events`(라우트 표면 — P0 추출) |
| 환경 제약 | stage 전용(생성 시도 수반 — prod 금지) |
| 오라클 출처 | API 스키마 `POST /api/events` — `title` required |
| 상태 | 미실행 |
| 러너 편입 | 미편입 |
| 증거 요구사항 | request.json(400 응답) · screenshot.png(오류 안내) · db-check.json(미생성 확인) |

**[세팅]**
- member 역할 로그인 세션 확보(patterns/auth-roles.md — `.auth/member.state.json` 재사용, 만료 시 재로그인).
- 시딩 불필요(사전 데이터 없음).

**[실행]**
- 일정 등록 화면에서 `title`을 비우고 `date`만 채워 저장을 시도한다.
- 스텝 상세: runbook.md `A-01` 블록.

**[실제값 검증]** → `evidence/A-01/`
- 네트워크: `POST /api/events` → **400** + 응답 body에 `title` 필드 오류 (request.json).
  프런트가 요청 자체를 차단하면 그 사실을 기록하고 API 직접 호출로 서버 측 거부를 확인한다.
- 화면: 필수값 오류 안내가 표시된다 (screenshot.png).
- DB: `events`에 이번 시도로 생성된 행 0건 (db-check.json).

**[정리]**
- 생성 리소스 없음 — STATE.md 잔여물 대장에 이 케이스 생성분 0건 확인만.

**바인딩 메모**
- (미실행 — P3에서 기록)
