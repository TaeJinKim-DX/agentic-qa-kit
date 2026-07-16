# runbook.md — 실행 런북 (도구 중립)

> P2(phases/P2-artifacts.md)에서 이 골격을 채워 `<프로젝트 루트>/qa/runbook.md`로 생성한다.
> 모든 절차는 **도구 중립 스텝 어휘**로 적는다 — 같은 런북이 대화형 브라우저 도구로도,
> Playwright 스크립트로도 실행된다. 어휘 정의와 두 바인딩 대응표(대화형 ↔ Playwright)의
> **정본은 patterns/browser.md** — 실행 시 그 대응표로 옮긴다.
> 셀렉터·대기 조건 등 바인딩 세부는 여기 적지 않는다 — 케이스 인벤토리
> (qa/cases/case-inventory.md)의 바인딩 메모에 기록한다(phases/P3-first-run.md).
> 절차가 현실과 어긋나면 문서 갭이다 — P4(phases/P4-retro.md)에서 이 파일을 직접 개선한다.

## 0. 스텝 어휘 (요약 — 정본은 patterns/browser.md, 이 어휘만 사용)

| 어휘 | 뜻 | 표기 예 |
|---|---|---|
| navigate | 화면/URL 이동 | `navigate: /events/new` |
| fill | 입력 필드 채우기 | `fill: title = "qa-<run-id>-회의"` |
| select | 드롭다운·옵션 선택 | `select: 공개 범위 = "팀 전체"` |
| click | 컨트롤 클릭 | `click: 저장 버튼` |
| press | 키 입력(Enter·Escape 등) | `press: Enter` |
| wait-for | 조건 대기(고정 대기 금지) | `wait-for: 목록에 새 행 표시` |
| expect-text | 화면 텍스트 검증 | `expect-text: "저장되었습니다"` |
| expect-network | 기대 요청·응답 확인 | `expect-network: POST /api/events → 400` |
| assert-db | DB 반영 확인 | `assert-db: events에 해당 행 0건` |
| assert-external | 외부 시스템 상태를 직접 조회해 기대값 대조 | `assert-external: <시스템>에 리소스 존재` |
| api-call | 브라우저 밖 직접 HTTP 호출(시딩·정리·외부시스템 교차검증) | `api-call: POST /api/events {…}` |
| screenshot | 화면 증거 저장 | `screenshot: 오류 안내 표시 상태` |
| extract | 화면·응답에서 값을 읽어 후속 스텝 변수로 | `extract: 생성 ID ← body.id → $ID` |

- 증거를 남기는 스텝(expect-network · assert-db · assert-external · screenshot)의 결과는
  `runs/<env>-<run-id>/evidence/<case-id>/`에 저장한다 — 스키마·redaction은 patterns/evidence.md.
- 케이스 경계마다 킬스위치 `qa/.stop`을 확인한다 — 존재하면 진행 중 케이스만 마치고 정리 후 중단.
- 실행 락 `qa/.lock`(stale 판정 30분)은 프리플라이트(1-1)에서 처리한다.

## 1. 공통 절차

### 1-1. 프리플라이트 (모든 run 공통 — 항목 정의는 phases/P3-first-run.md §1)

<!-- 5항목(헬스·도구·인증·이전 잔여물·.lock)을 이 프로젝트에서 어떻게 수행하는지 구체화한다. -->

1. 헬스: `<scripts/ 헬스체크 실행 방법 — 예: node scripts/healthcheck.mjs <환경>>`
2. 도구: 능력 프로파일(STATE.md)의 브라우저 경로 가동 확인 — 빈 페이지 `screenshot` 1장.
3. 인증 유효: 역할별 인증 필요 API 1콜 — `<예: expect-network: GET /api/me → 200>`.
   실패는 케이스 FAIL이 아니라 AUTH 이슈 — 1-2로 세션 재확보 후 재시도.
4. 이전 잔여물: STATE.md 잔여물 대장에 "미정리" 행 확인 → 있으면 1-3 정리부터.
5. `.lock`: `qa/.lock` 존재+신선(30분 이내)이면 시작 거부·기록. stale이면 잔여물 정리 후 인수.

통과 후 `qa/.lock` 생성 → run-id 발급 → `runs/<env>-<run-id>/` 생성.

### 1-2. 로그인 (역할별 — 규약은 patterns/auth-roles.md)

<!-- qa-config auth.roles의 역할마다 절차 1개. 세션 재사용이 기본, 만료 시에만 재로그인.
     시크릿 실값 기재 금지(원칙 6) — secret_ref(.env.qa 키 이름)만 적는다. -->

**`<역할명>`** (secret_ref: `<QA_..._CRED>`)
1. `qa/.auth/<role>.state.json`이 있고 만료 전이면 재사용 — 로그인 생략.
2. 없거나 만료:
   - `navigate: <로그인 화면>`
   - `fill: <계정 필드> = <.env.qa의 secret_ref 값>`
   - `click: 로그인` → `expect-network: <로그인 요청> → 2xx`
   - 세션 저장(만료시각 기록) — 로그·리포트에 세션 내용 인용 금지.
3. SSO면 `sso` / 캡차·2FA 등이면 `manual` (둘 다 유인 로그인→세션 저장 절차는 동일):
   사용자에게 수동 로그인을 요청하고 세션만 저장한다.

### 1-3. 정리 (run 종료 공통 — patterns/seeding.md)

1. STATE.md 잔여물 대장과 이번 run 생성 인벤토리 대조.
2. `<scripts/ 정리 스크립트 실행 방법>` — 삭제 방식은 프로젝트 정책(soft-delete가 정책이면 soft).
3. 정리검증: `assert-db: <잔여 확인 조회> → 0건` + `assert-external: <외부 잔여 확인> → 0건`.
4. 대장 갱신("완료(검증 시각)"). 잔여가 남으면 "미정리"로 기록 + 리포트 기재 + 사용자 보고.
5. `qa/.lock` 해제.

## 2. 케이스별 절차

<!-- 케이스 인벤토리(qa/cases/case-inventory.md)의 ID·순서와 1:1 대응. 블록 형식: -->

### `<ID>` — `<제목>`

- 전제: `<공통 1-2의 로그인 역할 / [세팅]에 필요한 시딩 스크립트>`
- 스텝:
  1. `<어휘>: <대상/입력>`
  2. `<…>`
- 검증(→ `evidence/<case-id>/`):
  1. `expect-network: <METHOD> <경로> → <상태코드·body 조건>`
  2. `assert-db: <조회> → <기대값>` (해당 시 `assert-external`도)
  3. `screenshot: <기대 화면 상태>`
- 정리: `<생성 리소스 정리 스텝 / 없음 — 잔여물 대장 확인만>`

<!-- 기입 예(가상 웹앱 — case-inventory.template.md의 예시 A-01과 짝):

### A-01 — 일정 등록: 필수값(title) 누락 거부

- 전제: member 로그인(공통 1-2). 시딩 없음.
- 스텝:
  1. navigate: /events/new
  2. fill: date = "<오늘+1일>"   (title은 비워 둔다)
  3. click: 저장 버튼
- 검증(→ evidence/A-01/):
  1. expect-network: POST /api/events → 400, body에 title 필드 오류
  2. screenshot: 필수값 오류 안내 표시 상태
  3. assert-db: events에 이번 시도로 생성된 행 0건
- 정리: 없음 — 잔여물 대장 확인만.
-->
