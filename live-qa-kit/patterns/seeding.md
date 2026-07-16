# seeding.md — 상태 시딩·정리·정리검증 패턴

> 원칙 4(식별 가능·실데이터 불가침)와 원칙 5(치우고, 치웠음을 검증)를 코드로 강제하는 패턴.
> P2 스크립트 생성 시(phases/P2-artifacts.md), 그리고 P3 이후 모든 run의 [세팅]·[정리] 단계에서 로드하라.
> 스크립트 표준 런타임은 Node.js(qa/scripts/), 시크릿은 `.env.qa`의 `secret_ref` 키로만 접근한다(원칙 6).

---

## 1. 원칙 — 상태는 API로 시딩, 브라우저는 검증할 UI만

- 케이스의 [세팅]은 **API 직접 호출**로 만든다. UI로 상태를 만드는 것(폼 채우기·클릭 연쇄)은
  느리고 플래키하며, 세팅 실패가 케이스 실패로 오염된다. API 시딩은 빠르고 결정적이다.
- 브라우저(대화형 브라우저 도구든 Playwright든 — patterns/browser.md)는 **그 케이스가 검증하려는
  UI 표면에만** 쓴다. 예: "목록에 항목이 표시된다" 케이스 — 항목 3건은 API로 시딩하고,
  브라우저는 목록 화면 확인에만 사용한다.
- 예외: **생성 UI 자체가 검증 대상**인 케이스(예: B 카테고리의 등록 폼)는 UI로 생성한다 —
  단 그 결과 검증은 API·DB 교차로(원칙 2, patterns/evidence.md). 이때 UI로 생성된 리소스도
  아래 인벤토리 규약(§4)을 똑같이 따른다.
- 시딩 API 호출의 인증(토큰·세션)은 patterns/auth-roles.md 규약을 따른다.

## 2. 명명 규약 — 식별 접두 + run-id

- 생성하는 모든 리소스의 이름(또는 식별 가능한 필드)에:
  **`<test_prefix><run-id>-<case-id>-<일련번호>`** (예: `qa-20260716a-A-01-1`).
  - `test_prefix` = qa-config.yaml `data.test_prefix`(기본 `qa-`).
  - `<run-id>` = 현재 run 식별자(run 디렉터리 `runs/<env>-<run-id>/`의 그것과 동일).
    형식은 `YYYYMMDD`+소문자 일련(a,b,…) — 예 `20260716a`(발급 규칙: phases/P3-first-run.md §1).
  - `<case-id>` = `<카테고리>-<번호>`(A-01 식) — 인벤토리·evidence/<case-id>/·정리가 상호참조하는 키.
- 이름 필드가 없는 리소스는 식별 가능한 다른 필드(설명·메모 등)에 접두를 넣는다.
  그것도 불가능하면 인벤토리 기록(§4)만이 유일한 추적 수단이다 — 기록 누락은 곧 유령 잔여물이다.
- **접두 없는 리소스는 절대 수정·삭제하지 않는다.** qa-config `data.deny_list`(기계 차단)와
  이중 방어다 — 시딩·정리 스크립트도 deny-list 대상에 접근 금지.
- 생성은 qa-config `environments.<env>.data_boundary`(전용 네임스페이스·계정·스키마) 안에서만.

## 3. 시딩 스크립트 표준 (qa/scripts/, Node.js)

**호출 규약**
- 인자로 환경 이름(qa-config `environments`의 키)과 run-id를 받는다:
  `node scripts/seed-<대상>.mjs --env <env> --run-id <run-id>`
- 시크릿은 `.env.qa`에서 `secret_ref` 키 이름으로 로드한다. 실값을 하드코딩·로그 출력하지 마라(원칙 6).
- run당 생성량은 qa-config `risk.budget_per_run.resource_creations` 상한 내 — 초과 시 중단+에스컬레이션.

**완료 조건 = 후속 조회 성공 (생성 응답 2xx가 아니다)**
- 생성 API의 200/201은 "접수됐다"는 표시일 뿐이다. 특히 외부 시스템(저장소 생성·인덱싱·프로비저닝
  등)은 **비동기로 준비**되어 지연이 있다 — 시딩 직후 케이스를 실행하면 세팅 미완으로 오탐이 난다.
- 따라서 모든 시딩 스크립트는 **readiness 폴링을 내장**한다: 생성 후 후속 조회(조회 API 또는
  외부 시스템 직접 조회)가 성공해야 그 리소스의 시딩 완료다.

```js
// readiness 폴링 골격 — 타임아웃·백오프·킬스위치 내장
const POLL = { initialMs: 1000, factor: 1.5, maxIntervalMs: 10_000, timeoutMs: 60_000 };
// 외부 시스템 대상은 timeoutMs 상향(권장 120_000). 값 조정은 P4 회고에서.

async function waitReady(check, poll = POLL) {
  const start = Date.now();
  let interval = poll.initialMs;
  while (Date.now() - start < poll.timeoutMs) {
    if (stopRequested()) throw new Error("STOP: qa/.stop 감지");  // 킬스위치 — 루프마다 확인
    if (await check()) return true;            // 후속 조회 성공 = 시딩 완료
    await sleep(interval);
    interval = Math.min(interval * poll.factor, poll.maxIntervalMs);  // 지수 백오프
  }
  return false;                                // 타임아웃 = 시딩 실패(단, 인벤토리에는 이미 기록됨 — §4)
}

// 순서: 생성 요청 → 인벤토리 기록(즉시) → readiness 폴링 → 완료 판정
const res = await api.post("/api/<resource>", { name: `${prefix}${runId}-A-01-1` /* … */ });
inventory.add({ name: `${prefix}${runId}-A-01-1`, id: res.id, type: "db:<table>", case: "A-01" });
const ready = await waitReady(() => api.get(`/api/<resource>/${res.id}`).then(r => r.ok));
if (!ready) fail("SEED_TIMEOUT: 후속 조회 미확인 — 리소스는 인벤토리에 남아 정리 대상");
```

- **킬스위치**: 스크립트 내부의 모든 폴링·반복 루프는 `qa/.stop` 존재를 확인한다
  (케이스 경계 체크는 러너 담당 — phases/P5-unattended.md, 스크립트 내부 루프는 스크립트 담당).
- **멱등성**: 같은 run-id로 재실행되면 이미 존재하는 동명 리소스는 재사용하거나 정리 후
  재생성한다 — 중복 생성으로 인벤토리와 실물이 어긋나게 하지 마라.

## 4. 생성 인벤토리 — STATE 잔여물 대장과 동기

- **생성 요청을 보낸 즉시**(성공 확인 전에) 인벤토리에 기록한다. 요청이 타임아웃·에러로 끝나도
  서버에는 생성됐을 수 있다 — "생성됐을 수도 있는" 식별자를 기록해야 유령 잔여물이 안 생긴다.
- 스크립트는 기계용 인벤토리를 `runs/<env>-<run-id>/inventory.json`에 남긴다
  (정리·정리검증 스크립트의 대조 입력):

```json
[
  { "name": "qa-20260716a-A-01-1", "id": "<서버 발급 ID>", "type": "db:<table>",
    "external": "<외부 시스템 리소스 경로 | null>", "case": "A-01",
    "created_at": "<ISO8601>", "cleaned": false, "verified_at": null }
]
```

- 호출자(에이전트 또는 러너)는 인벤토리 변경을 **즉시 STATE.md의 시딩 잔여물 대장에 반영**한다
  (templates/STATE.template.md — 리소스 식별자·유형·생성 run-id·정리 여부).
  사람·다음 세션이 읽는 정본은 STATE의 대장, `inventory.json`은 스크립트용 사본이다.
  둘이 어긋나면 **합집합을 정리 대상으로**(보수적으로) 삼는다.

## 5. 정리 스크립트 표준 (qa/scripts/)

- **프로젝트 삭제 정책을 따른다**: soft-delete가 정책이면 soft로 지운다 — 임의로 영구삭제하지
  마라(원칙 5·8, 하드삭제가 정책인 프로젝트도 있다 — 판단 근거는 설계 문서). soft 잔재의 purge는
  qa-config `data.retention.softdelete_purge` 주기 정책에 맡긴다.
- 정리 방식은 qa-config `data.cleanup_method`(soft | hard)를 참조한다 — P1에서 확정된 프로젝트 정책값이다.
- 호출 규약은 시딩과 동일: `node scripts/cleanup.mjs --env <env> --run-id <run-id>`.
- **대상 결정은 2단**:
  1. **인벤토리 기반**(1차): `inventory.json`의 `cleaned: false` 항목을 의존 역순(자식→부모,
     외부 연동 해제→본체)으로 삭제.
  2. **접두 스윕**(2차): `<test_prefix><run-id>` 패턴으로 DB·외부 시스템을 검색해 인벤토리에
     없는 유령 잔여물을 탐지 — 발견 시 인벤토리·STATE 대장에 추가 기록 후 삭제.
- 접두 없는 리소스는 스윕에 걸려도 **절대 건드리지 않는다**(§2). deny-list 대상 접근 금지.
- 삭제 API 호출 실패(권한·정책·의존성)는 건너뛰지 말고 해당 항목에 사유를 기록하고
  에스컬레이션한다 — "정리 불가"를 조용히 넘어가면 원칙 5 위반이다.

## 6. 정리검증 — 검증 통과가 정리 완료의 정의

- **삭제 API 2xx는 정리 완료가 아니다.** 정리 완료의 정의는 단 하나 — **인벤토리 대조 검증 통과**:
  - **DB 잔여 0건**: 인벤토리의 각 식별자를 조회해 부재를 확인한다. soft-delete 정책이면
    "활성 조회 결과 0건(삭제 표시 상태)"이 잔여 0의 정의다 — 다음 run을 오염시키지 않는 상태.
  - **외부 시스템 잔여 0건**: 외부 시스템에 직접 조회해 부재를 확인한다. DB만 보고 "치웠다"고
    하지 마라 — 외부에 실물이 남는 것이 전형적 누락이다.
- 삭제 반영도 비동기일 수 있다 — 검증 조회 역시 §3의 폴링 규격(타임아웃·백오프·`.stop` 체크)을 쓴다.
- 검증 통과 항목: `inventory.json`의 `cleaned: true`·`verified_at` 기입 + STATE 잔여물 대장
  해당 행을 "완료(검증 타임스탬프)"로 갱신. **검증 못 한 항목이 하나라도 있으면 정리 미완**이다 —
  run 리포트(`runs/<env>-<run-id>/report.md`)에 미정리 항목과 사유를 기록하고 에스컬레이션한다.
- 검증에 쓴 확인 쿼리·조회 응답은 리포트에 남긴다(잔여 0건의 증거). 저장 전 redaction은
  patterns/evidence.md 규약을 따른다.

## 7. 중단 복구 — 세션이 죽은 뒤 재개하면 정리부터

- 재개 세션의 첫 작업(AGENT.md §2): `qa/STATE.md`의 시딩 잔여물 대장에서 **"미정리" 행을
  확인하고, 새 작업 전에 정리부터** 한다 — 해당 run-id로 정리 스크립트(§5) 실행 → 정리검증(§6)
  통과 → 대장 갱신. 그 후에야 새 run을 시작한다.
- `qa/.lock` 처리: 존재하고 신선하면 다른 세션 실행 중 — 시작을 거부하고 기록한다.
  **stale(갱신 없이 30분 경과)이면 죽은 세션이다** — 잔여물 정리·검증을 마친 뒤 lock을 인수한다.
- `inventory.json`이 없거나 손상됐으면: STATE 대장을 1차 근거로 쓰고, `<test_prefix>` +
  해당 run-id 패턴 스윕(§5의 2단)으로 잔여물을 재구성한다 — 접두가 확인된 것만 정리 대상이다.
- 정리 불가 항목(권한 상실·정책 충돌)은 대장에 사유를 남기고 사용자에게 에스컬레이션한다.
  임의 판단으로 "정리된 셈" 처리하지 마라 — 결정은 사람이 한다(원칙 10).
