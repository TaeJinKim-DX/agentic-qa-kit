# cases-sample.md — TeamBoard 케이스 발췌 (예시 · 15건 중 5건)

> 가상 프로젝트 예시다(example/README.md). [게이트2] 승인된 인벤토리 15건(전체 목록·커버리지 표는
> gate-summary-예시.md) 중 카테고리별 대표 5건. 포맷은 phases/P2-artifacts.md §1.5 —
> `[세팅]→[실행]→[실제값 검증]→[정리]` + 증거 요구사항. 스텝 어휘·두 바인딩 대응은 patterns/browser.md,
> 시딩·정리 규약은 patterns/seeding.md. `<run-id>`는 실행 시점에 치환된다(예: 20260713a).
> 케이스 ID(`<카테고리>-<번호>`)는 evidence/·runner/·known-issues.md의 공통 키 — 확정 후 변경 금지.

---

## A-01 — 제목 없이 게시글 등록이 거부된다

| 필드 | 값 |
|---|---|
| 우선순위 | 상 |
| 출처 | docs/spec/posts.md("title 필수, 1~200자") |
| 오라클 출처 | 기능 문서 |
| 환경 제약 | stage |
| 증거 요구사항 | request.json(400 응답) · screenshot.png(에러 메시지) · db-check.json(미생성 확인) |
| 상태 | 미실행 |
| 러너 편입 | 보류 |

[세팅]
- member 세션 로드(qa/.auth/member.state.json — patterns/auth-roles.md). 시딩 없음.

[실행]
- navigate: /boards/test-board/new
- fill: "본문" 입력란 = qa-<run-id>-A-01 본문(제목은 비워 둔다)
- click: "등록" 버튼

[실제값 검증]
- expect-network: POST /api/posts → 400, body.error.code = "TITLE_REQUIRED"
- expect-text: "제목을 입력해 주세요"
- assert-db: posts(본문 = qa-<run-id>-A-01 본문) → 0건(생성되지 않았음)

[정리]
- 정리 없음(생성 실패가 기대 결과 — [실제값 검증]의 0건 확인이 곧 잔여 없음 증명)

---

## A-02 — 제목 길이 경계(200자 허용 / 201자 거부)

| 필드 | 값 |
|---|---|
| 우선순위 | 중 |
| 출처 | docs/spec/posts.md("title 1~200자") |
| 오라클 출처 | 기능 문서 |
| 환경 제약 | stage |
| 증거 요구사항 | request-200.json · request-201.json · db-check.json · cleanup-check.json |
| 상태 | 미실행 |
| 러너 편입 | 보류 |

[세팅]
- 경계값은 API 레벨이 정확하다 — member 인증으로 api-call 직접 검증(UI 경유 성공 경로는 B-01 담당).
- 제목 2종 생성: `qa-<run-id>-A-02-` 접두 + 채움 문자로 정확히 200자 / 201자.

[실행]
- api-call: POST /api/posts { title: <200자>, board: "test-board" }
- api-call: POST /api/posts { title: <201자>, board: "test-board" }

[실제값 검증]
- 200자 → 201 Created, body.title 길이 = 200
- 201자 → 400, body.error.code = "TITLE_TOO_LONG"
- assert-db: 200자 글 1건 존재 + 201자 글 0건

[정리]
- api-call: DELETE /api/posts/<200자 글 id> (soft — 프로젝트 정책, CL-05)
- 정리검증: 활성 조회(deleted_at IS NULL) 0건 확인 → 잔여물 대장 "완료" 갱신(patterns/seeding.md §6)

---

## B-01 — 게시글 작성(대표 성공 경로) ★ 두 바인딩 병기

| 필드 | 값 |
|---|---|
| 우선순위 | 상 |
| 출처 | docs/spec/posts.md |
| 오라클 출처 | 기능 문서 |
| 환경 제약 | stage |
| 증거 요구사항 | request.json(201 응답) · screenshot.png(상세 화면) · db-check.json(1건 존재) |
| 상태 | 미실행 |
| 러너 편입 | 보류 |

[세팅]
- member 세션 로드. 시딩 없음 — **생성 UI 자체가 검증 대상**이라 UI로 생성한다(patterns/seeding.md §1 예외).
  UI로 생성된 리소스도 STATE.md 잔여물 대장에 즉시 기록.

[실행]
- navigate: /boards/test-board/new
- fill: "제목" 입력란 = qa-<run-id>-B-01-1 제목
- fill: "본문" 입력란 = 라이브 QA 생성 글(자동 정리 대상)
- click: "등록" 버튼
- wait-for: url ~ /boards/test-board/posts/

[실제값 검증]
- expect-network: POST /api/posts → 201, body.title = "qa-<run-id>-B-01-1 제목"
- extract: 생성 글 ID ← 응답 body.id → $POST_ID
- expect-text: "qa-<run-id>-B-01-1 제목"
- assert-db: posts(id = $POST_ID, deleted_at IS NULL) → 1건
- screenshot: post-created

### 바인딩 1 — 대화형 브라우저 도구

1. URL 이동: `<base_url>/boards/test-board/new` → 접근성 트리로 작성 폼 로드 확인.
2. "제목" 입력란 클릭 → 전체 선택 → `qa-<run-id>-B-01-1 제목` **실제 타이핑**
   (JS 값 주입 금지 — controlled input, patterns/browser.md §3.1). "본문"도 동일.
3. "등록" 버튼 — 클릭 전 안전 검사(deny-list·파괴적 문구 아님 확인 — §3.5) → 클릭.
4. 네트워크 요청 조회: POST /api/posts 의 상태 201·body.title 확인 → redaction 후
   evidence/B-01/request.json 저장.
5. 접근성 트리(또는 페이지 텍스트)에서 상세 화면 제목 확인 → 스크린샷을 evidence/B-01/screenshot.png로
   (타임아웃 시 접근성 트리 + 네트워크 로그로 대체 — §3.3).
6. 셸: `node scripts/db-check.mjs --table posts --id $POST_ID` → evidence/B-01/db-check.json
   (assert-db 는 어느 바인딩이든 브라우저 밖 Node 스크립트 — §1).

### 바인딩 2 — Playwright (발췌 — 증거 저장 유틸·재시도는 생략)

```js
import { test, expect } from '@playwright/test';

const RUN_ID = process.env.QA_RUN_ID;                     // 예: 20260713a
const TITLE = `qa-${RUN_ID}-B-01-1 제목`;

test.use({ storageState: 'qa/.auth/member.state.json' }); // 세션 재사용 — patterns/auth-roles.md

test('B-01 게시글 작성', async ({ page }) => {
  await page.goto('/boards/test-board/new');

  await page.getByLabel('제목').fill(TITLE);
  await expect(page.getByLabel('제목')).toHaveValue(TITLE); // fill 반영 검증 — browser.md §3.1
  await page.getByLabel('본문').fill('라이브 QA 생성 글(자동 정리 대상)');

  const resPromise = page.waitForResponse(r =>
    r.url().includes('/api/posts') && r.request().method() === 'POST');
  await page.getByRole('button', { name: '등록' }).click();
  const res = await resPromise;

  expect(res.status()).toBe(201);                          // expect-network
  const body = await res.json();
  expect(body.title).toBe(TITLE);
  // redaction 후 evidence/B-01/request.json 저장(patterns/evidence.md)

  await page.waitForURL(/\/boards\/test-board\/posts\//);  // wait-for
  await expect(page.getByText(TITLE)).toBeVisible();       // expect-text
  await page.screenshot({ path: `runs/stage-${RUN_ID}/evidence/B-01/screenshot.png` });

  // assert-db — node scripts/db-check.mjs --table posts --id <body.id> → db-check.json
});
```

[정리]
- api-call: DELETE /api/posts/$POST_ID (soft — 프로젝트 정책, CL-05)
- 정리검증: 활성 조회 0건 확인 → 잔여물 대장 "완료(검증 시각)" 갱신(원칙 5)

---

## D-01 — 타인 글 수정 차단(UI + API 레벨)

| 필드 | 값 |
|---|---|
| 우선순위 | 상 |
| 출처 | docs/spec/permissions.md("수정은 작성자 본인 또는 admin만") |
| 오라클 출처 | 기능 문서 |
| 환경 제약 | stage |
| 증거 요구사항 | request.json(403 응답) · db-check.json(무변경 확인) · screenshot.png(수정 버튼 부재) |
| 상태 | 미실행 |
| 러너 편입 | 보류 |

[세팅]
- admin 인증으로 시딩: `node scripts/seed-post.mjs --env stage --run-id <run-id>` →
  글 `qa-<run-id>-D-01-1`(작성자 = admin) 생성, readiness 폴링 통과 확인.
- 생성 즉시 STATE.md 잔여물 대장 기록(patterns/seeding.md §4). member 세션 로드.

[실행 — UI 레벨]
- navigate: /boards/test-board/posts/$POST_ID
- expect-text: 상세 화면에 "수정" 버튼 없음   # 부재 검증 — 가장 가까운 어휘 + 주석(browser.md 표기 규약)
- screenshot: no-edit-button

[실행 — API 레벨(권한 경계의 정본 — UI 버튼 숨김만으로는 차단이 아니다)]
- api-call: PUT /api/posts/$POST_ID { title: "변조 시도" } — member 인증으로 직접 호출

[실제값 검증]
- API 응답 = 403, body.error.code = "FORBIDDEN"
- assert-db: posts(id = $POST_ID).title = "qa-<run-id>-D-01-1"(무변경)

[정리]
- admin 인증 api-call: DELETE /api/posts/$POST_ID (soft) → 활성 조회 0건 검증 → 잔여물 대장 "완료" 갱신

---

## E-01 — 공유 링크 만료일의 자정 경계 표시(저장 UTC vs 표시 KST)

| 필드 | 값 |
|---|---|
| 우선순위 | 중 |
| 출처 | docs/spec/files.md("만료일은 Asia/Seoul 기준으로 표시") + 시간 경계 의무(phases/P2-artifacts.md §1.3 — timezone.storage ≠ display) |
| 오라클 출처 | 기능 문서 |
| 환경 제약 | stage |
| 증거 요구사항 | request.json(share 응답 — 저장값) · screenshot.png(표시 화면) · db-check.json |
| 상태 | 미실행 |
| 러너 편입 | 보류 |

[세팅]
- member 인증 시딩: `node scripts/seed-file.mjs --env stage --run-id <run-id>` →
  파일 `qa-<run-id>-E-01-1` 업로드(스토리지 readiness 폴링 포함), 잔여물 대장 기록.
- api-call: POST /api/files/$FILE_ID/share { expires_at: "2026-07-20T00:00:00+09:00" } —
  **KST 자정 정각**(= UTC 전날 15:00). 자정 경계값이라 UTC 날짜와 KST 날짜가 갈린다.

[실행]
- navigate: /files/$FILE_ID
- wait-for: 공유 정보 영역 표시

[실제값 검증]
- expect-network: GET /api/files/$FILE_ID/share → 200, body.expires_at = "2026-07-19T15:00:00Z"(UTC 저장 확인)
- expect-text: 만료일 "2026-07-20"(KST 변환 표시 — "2026-07-19"로 보이면 자정 경계에서 하루 밀린 것)
- assert-db: share_links(file_id = $FILE_ID).expires_at = 2026-07-19 15:00:00+00

[정리]
- api-call: DELETE /api/files/$FILE_ID (soft — 공유 링크 함께 비활성)
- 정리검증: DB 활성 조회 0건 + 앱 경유 파일 접근 불가 확인. 스토리지 실물은 purge 정책(30d)이
  제거 — soft-delete 정책에서 "잔여 0건" = 다음 run을 오염시키지 않는 상태(patterns/seeding.md §6)
