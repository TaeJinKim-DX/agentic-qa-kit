# patterns/evidence.md — 증거 스키마 · redaction · mock 판별 · 교차검증

> 원칙 1(실데이터)·2(모든 PASS엔 증거)·8(오탐 방지)을 실행 절차로 만든 문서.
> P3 유인 실행·P4 회고·P5 무인 러너에서 증거를 만들거나 판정할 때 로드하라.
> 규정은 **"무엇이 담겨야 하나"** 다 — 어떤 도구(대화형 브라우저 도구 / Playwright / HTTP 클라이언트)로
> 얻는지는 수단의 문제고, 스키마는 같다.

---

## 1. 증거 스키마 (도구 중립)

위치: `runs/<env>-<run-id>/evidence/<case-id>/` — 케이스 ID(`A-01` 식)는
cases/·runner/·known-issues.md·리포트가 상호참조하는 공통 키다.

| 파일 | 언제 필수 | 무엇이 담겨야 하나 |
|---|---|---|
| `request.json` | 네트워크 검증이 있는 모든 케이스 | 판정 대상 요청·상태코드·응답 **발췌** + 판정·근거 |
| `screenshot.png` | 선택 — UI 표시 자체가 검증 대상일 때 | 판정 시점 화면 |
| `db-check.json` | 저장 검증이 케이스 검증 항목일 때 | 저장소 반영 확인(조회·기대·실제·일치) |
| `external-check.json` | 외부 시스템 부작용 케이스(F 등) 해당 시 | 외부 시스템 교차 확인(조회·기대·실제·일치) — `assert-external` 스텝의 산출물(patterns/browser.md §1) |

케이스별 필수 파일은 케이스의 **증거 요구사항 필드**가 정한다(phases/P2-artifacts.md).
요구된 파일이 없으면 그 케이스는 PASS가 아니다(원칙 2).

### request.json

```json
{
  "case_id": "B-01",
  "captured_at": "2026-01-01T02:00:00+09:00",
  "request": {
    "method": "POST",
    "url": "https://stage.example.com/api/items",
    "body_excerpt": { "name": "qa-20260101a-B-01-1" }
  },
  "response": {
    "status": 201,
    "body_excerpt": { "id": 1042, "name": "qa-20260101a-B-01-1" }
  },
  "verdict": "PASS",
  "verdict_basis": "201 + 응답 id 실존 확인, db-check.json 일치"
}
```

- **발췌만.** 전체 body 덤프 금지 — 판정에 쓴 필드만 남겨라(redaction 면적과 노이즈를 줄인다).
- **헤더는 기본 미포함.** Authorization/Cookie는 어차피 저장 금지(§3). 다른 헤더가 판정 근거면 그 헤더만 발췌.
- `captured_at`은 타임존 포함 — 시간 경계 케이스(E)의 판정 근거다.
- `verdict_basis` 1줄 필수 — 오탐 방지 관문(§6) 통과 내용이 여기 들어간다.

### screenshot.png (선택)

- UI 표시 자체가 검증 대상일 때만 남긴다. "찍어두면 좋겠지"로 늘리지 마라 — redaction 대상만 는다.
- 화면에 시크릿·1회 노출 토큰·실데이터 개인정보가 보이는 상태로 저장 금지(§3).
- 스크린샷 실패(타임아웃 등) 시 폴백(접근성 트리/DOM 스냅샷 + 네트워크 캡처)은 patterns/browser.md.

### db-check.json / external-check.json

```json
{
  "case_id": "B-01",
  "checked_at": "2026-01-01T02:00:05+09:00",
  "target": "DB items 테이블",
  "lookup": "SELECT id, name FROM items WHERE name = 'qa-20260101a-B-01-1'",
  "expected": { "count": 1, "name": "qa-20260101a-B-01-1" },
  "actual":   { "count": 1, "name": "qa-20260101a-B-01-1" },
  "match": true
}
```

- `target` — 어디를 봤나(external-check는 외부 시스템 이름). `lookup` — 실제 실행한 조회(SQL·API 경로·CLI)를
  그대로 적어라. 사람이 같은 조회로 재현할 수 있어야 증거다.
- DB 직접 접근이 없으면 read-back API 조회로 대체하고 `lookup`에 그렇게 적어라 —
  단 그 API가 mock이 아닌지 먼저 확인(§4).
- 비동기 반영(큐·웹훅)은 즉시 조회 1회로 단정하지 마라 — 폴링·타임아웃(요령은 patterns/seeding.md의
  readiness 폴링과 동일) 후 최종값을 기록.

---

## 2. 수집 — 두 경로

**대화형 브라우저 도구 경로**
- 네트워크 요청 읽기 도구로 판정 대상 요청·응답을 확인 → 발췌를 직접 `request.json`으로 작성(저장 전 §3 redaction).
- 스크린샷 도구 → `screenshot.png`.
- db/external 확인은 Node.js 스크립트(qa/scripts/)로 조회하고 결과를 JSON으로 기록.

**Playwright 경로 — 수집 훅 스니펫**

```js
// qa/scripts/lib/evidence.js — 케이스 시작 전 attach, 판정 직후 finalize
const fs = require('node:fs');
const path = require('node:path');

// qa-config.yaml secret_patterns → RegExp 배열
function loadPatterns(qaConfig) {
  return (qaConfig.secret_patterns || []).map((p) => new RegExp(p, 'gi'));
}

function hit(re, s) { re.lastIndex = 0; return re.test(String(s)); }

// 키가 매칭되면 값 마스킹, 문자열 값이 매칭돼도 마스킹 (§3 절차 2)
function redact(node, patterns) {
  if (Array.isArray(node)) return node.map((v) => redact(v, patterns));
  if (node && typeof node === 'object') {
    return Object.fromEntries(Object.entries(node).map(([k, v]) =>
      [k, patterns.some((re) => hit(re, k)) ? '***REDACTED***' : redact(v, patterns)]));
  }
  if (typeof node === 'string' && patterns.some((re) => hit(re, node))) return '***REDACTED***';
  return node;
}

function attachEvidence(page, { runDir, caseId, isTarget, patterns }) {
  const dir = path.join(runDir, 'evidence', caseId);
  fs.mkdirSync(dir, { recursive: true });
  const captured = [];
  page.on('response', async (res) => {
    const req = res.request();
    if (!isTarget(req.method(), req.url())) return;   // 케이스가 판정에 쓰는 요청만
    let body = null;
    try { body = await res.json(); } catch { /* 비JSON은 발췌 생략 */ }
    captured.push({
      captured_at: new Date().toISOString(),
      // 헤더는 캡처 자체에서 제외 — Authorization/Cookie는 제거가 마스킹보다 안전(§3 절차 1)
      request: { method: req.method(), url: req.url(),
                 body_excerpt: req.postData() ? JSON.parse(req.postData()) : null },
      response: { status: res.status(), body_excerpt: body },
    });
  });
  return {
    // 저장은 이 함수 하나로만 — redact를 거치지 않는 우회 쓰기 금지(§3)
    finalize({ verdict, verdict_basis, excerpt }) {
      const last = captured.at(-1);
      if (!last) throw new Error(`evidence: no captured request for ${caseId}`);
      const record = { case_id: caseId, ...last, verdict, verdict_basis };
      if (excerpt) {                       // 판정에 쓴 필드만 남기는 발췌 함수(케이스별 정의)
        record.request.body_excerpt = excerpt(record.request.body_excerpt);
        record.response.body_excerpt = excerpt(record.response.body_excerpt);
      }
      fs.writeFileSync(path.join(dir, 'request.json'),
        JSON.stringify(redact(record, patterns), null, 2));
    },
    screenshot: () => page.screenshot({ path: path.join(dir, 'screenshot.png') }),
  };
}

module.exports = { loadPatterns, redact, attachEvidence };
```

db-check/external-check는 같은 `redact` 함수를 거쳐 저장하는 조회 스크립트로 작성한다(qa/scripts/).

---

## 3. redaction — 리포트 확정의 전제 게이트 (원칙 6)

모든 증거 파일은 **저장 전** 아래 3단계를 순서대로 거친다. redaction 미통과 파일이 하나라도 있으면
run 리포트를 확정하지 마라.

1. **헤더 제거** — `Authorization`·`Cookie`·`Set-Cookie`는 마스킹이 아니라 **아예 미포함**.
   캡처 단계에서 제외하는 것이 저장 후 지우는 것보다 안전하다.
2. **secret_patterns 마스킹** — qa-config.yaml `secret_patterns` 정규식으로 키·값을 전부 스캔,
   매칭되면 값을 `***REDACTED***`로. 프로젝트 고유 토큰 형식은 P0/P1에서 패턴으로 등록돼 있어야 한다 —
   빠진 형식을 발견하면 즉시 패턴 추가를 제안하라(config 변경은 게이트 사항, 원칙 10).
3. **1회 노출 토큰류 마스킹** — 생성 시 한 번만 노출되는 토큰·임시 비밀번호·초대코드 등은
   패턴 매칭 여부와 무관하게 값 전체를 마스킹. 존재·형식 검증이 케이스 목적이면
   `"토큰 존재, 길이 40"` 식 **메타데이터만** 남겨라. 이런 응답 필드를 발견하면 secret_patterns에
   등록해 기계 마스킹으로 승격하라.

**게이트 절차**
- 저장 경로 단일화: 증거 쓰기는 redact를 내장한 함수 하나(§2 `finalize`)로만. 우회 쓰기 금지.
- **최종 스윕**: report.md 확정 직전, `evidence/` 전체 파일을 secret_patterns로 재스캔 →
  매칭 0건 확인 → 결과를 리포트에 1줄 기록(예: `redaction 스윕: 0건`). 이 줄이 없는 리포트는 미확정이다.
- 스윕에서 발견 시: 즉시 마스킹으로 덮어쓰고, 같은 값이 로그·커밋·채팅 출력에 이미 노출됐는지 확인.
  노출됐으면 리포트에 기록하고 해당 시크릿 로테이션을 사용자에게 에스컬레이션.
- screenshot.png는 정규식 스윕이 닿지 않는다 — 시크릿이 보이는 화면은 찍지 말거나,
  스크린샷을 포기하고 request.json으로 대체하라.

---

## 4. mock 판별 — 이 증거는 진짜인가 (원칙 1)

증거를 남기기 전에 증거의 출처가 실시스템인지 확인하라. mock을 상대로 남긴 PASS는 검증이 아니다.

**mock·폴백 신호**
- 백엔드를 내려도(또는 네트워크가 실패해도) 화면이 그럴듯하다 → 하드코딩 폴백.
- 판정 대상 액션에서 네트워크 요청이 아예 안 나간다(정적 데이터 렌더).
- 토큰·ID가 `mock-`, `test-`, `dummy-`, `sample-` 패턴.
- 어떤 입력에도 같은 응답(상수 응답) / 응답 값이 프론트 코드 안 상수와 일치.
- 타임스탬프가 고정값이거나 매 실행 동일.

**실제 신호**
- 매 실행 값이 달라진다(타임스탬프·증가 ID).
- 요청 body를 바꾸면 응답이 그에 따라 달라진다.
- **외부 직접 조회 교차**: 생성됐다는 리소스를 UI 밖(DB·외부 시스템 API)에서 조회하면 실제로 존재한다.

**판별 절차 (의심 시)**
1. 네트워크 캡처로 요청이 실제로 나갔고 응답이 서버에서 왔는지 확인
   (대화형: 네트워크 읽기 도구 / Playwright: `page.on('response')`).
2. 입력을 바꿔 재실행 → 응답이 입력을 따라 변하는가.
3. 외부 직접 조회 교차(위) — 이것이 최종 판별이다. mock은 UI 밖에 리소스를 남기지 못한다.

**mock으로 판정되면**: 해당 표면의 케이스를 PASS로 남기지 말고 **환경 문제**로 기록 —
대상 환경이 mock 모드로 떠 있거나 base_url이 잘못됐을 수 있다. qa-config의 환경 정의를 재확인하고
사용자에게 에스컬레이션하라(케이스 FAIL로 세지 않는다).

---

## 5. 3중 교차검증 — 네트워크 → DB → 외부 (원칙 2)

부작용이 있는 케이스는 세 겹으로 확인한다. 순서 고정:

1. **네트워크**: 요청이 나갔고 기대 상태코드·응답으로 돌아왔다 → `request.json`
2. **DB**: 그 결과가 실제로 저장됐다 → `db-check.json`
3. **외부**: 외부 시스템 부작용이 있는 케이스면 그쪽 상태까지 → `external-check.json`

몇 겹이 필요한지는 케이스의 증거 요구사항이 정한다 — 읽기 케이스(C)는 네트워크만으로 족할 수 있고,
교차시스템 케이스(F)는 3겹 전부다. 각 겹의 결과는 해당 파일의 `match` 필드로 남는다.

**부분 성공 판정표**

| 네트워크 | DB | 외부 | 판정 |
|---|---|---|---|
| 기대 응답 | 반영 | 반영(또는 해당 없음) | PASS |
| 기대 응답 | **미반영** | — | FAIL — 전형적 가짜 성공(응답만 그럴듯) |
| 기대 응답 | 반영 | **미반영** | 비동기 지연 가능 — 폴링 후에도 없으면 FAIL |
| 기대 외 응답 | — | — | FAIL — 단, DB·외부를 마저 확인해 **부분 반영**(더 나쁜 상태) 여부를 기록 |

실패·거부 케이스(A·D)의 교차검증은 방향이 반대다: 거부된 요청이 **흔적을 남기지 않았음**(잔여 0건)을
db-check로 확인하라 — 4xx를 받고도 리소스가 생겼다면 그게 버그다.

---

## 6. 오탐 방지 3관문 — evidence 흐름에 배치 (원칙 8)

FAIL·"버그" 판정은 세 관문을 다 지나야 한다. 각 관문을 증거 작업 흐름의 고정 시점에 배치하라.

| 관문 | 시점 | 하는 일 | 기록 위치 |
|---|---|---|---|
| ① 실존 확인 | 판정이 FAIL로 기울 때 즉시(증거 확정 전) | 이상하다고 본 필드/엔드포인트가 API 스키마·코드에 **실제로 존재**하는가 — 없는 필드를 읽고 "값이 없는 버그"라 부르지 마라 | `request.json`의 `verdict_basis` |
| ② 의도 확인 | FAIL 잠정 판정 후, 리포트 기재 전 | 설계 문서·기능 명세·결정 기록 확인 — **의도된 설계**일 수 있다(예: 하드삭제가 정책인 프로젝트) | 리포트의 케이스 결과 노트(확인한 문서명 포함) |
| ③ 재현 2회 | 리포트 작성 전 | 새 식별자(접두+run-id)로 동일 케이스 재실행 — 2회 모두 재현될 때만 "버그" | 같은 `evidence/<case-id>/`에 2회차 증거(`request-2.json` 식 접미사) |

- 관문에서 걸리면 버그가 아니다: ①에서 필드 부재 → **테스트 버그**(케이스 수정) /
  ②에서 의도 확인 → **문서 갭**(케이스 기대값 수정) / ③에서 재현 안 됨 → **플래키**로 분류.
  실패 3분류와 후속 처리는 phases/P4-retro.md.
- 3관문을 통과한 버그도 known-issues.md 등록은 자동이 아니다 — **게이트3 승인 사항**(원칙 10).
  리포트에는 관문별 확인 결과를 판정 근거로 함께 적어 올려라.
- 무인 run(P5)에서는 관문 ②를 코드로 대신할 수 없다 — 러너는 FAIL을 "미확인 실패"로 표기만 하고,
  3관문 판정은 다음 유인 세션(또는 에이전트 회고)에서 수행한다.

## 7. 시간 경계(E) 검증 기법

`timezone.storage` ≠ `display`면 시간·날짜 버그가 조용히 숨는다(실전에서 두 번 터졌다:
만료일 계산이 시간대 경계에서 어긋난 것, 저장값을 UTC로 읽어 표시가 하루 이르게 나온 것).
E 케이스는 아래 기법으로 **저장 원값과 표시값을 분리 대조**한다.

- **저장 vs 표시 3단 대조**: ① API 응답의 날짜/시각 문자열(표시값) ② DB 원값(`db-check.json` —
  `TIMESTAMPTZ`는 UTC로 저장됨) ③ 외부 시스템이 있으면 그쪽 값(`external-check.json`).
  세 값이 **같은 순간을 가리키는지** 확인하라 — 표시값만 보면 UTC↔로컬 하루 밀림을 못 잡는다.
- **경계 시각 시딩**: 자정 직전(예: 로컬 23:50)·직후(00:10)·월말·윤년 2/29 등 **경계에 걸치는
  시각으로 데이터를 만든다**(patterns/seeding.md). 시각 주입 수단이 없으면 실제 그 시각을 기다리지
  말고 — 저장 시각을 직접 지정하는 시딩 API/DB 삽입을 쓰거나, 없으면 케이스에 "경계 시각 수동 필요"로 표기.
- **만료·상대 날짜**: "N일 후 만료" 같은 계산은 경계에서 off-by-one이 흔하다 — 계산식이 UTC/로컬
  중 무엇을 기준으로 하는지 코드에서 확인하고(오탐 방지 관문①), 저장된 만료값과 외부 시스템의 만료값을
  대조한다(둘이 하루 다르면 그게 버그다).
- **표시 포맷**: 상대 표기("3분 전")·날짜만 표기는 타임존 절삭이 숨는 자리다 — 원값(초 단위)과 함께 본다.
