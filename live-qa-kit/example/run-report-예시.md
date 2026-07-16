# run-report — runs/stage-20260713a/report.md (TeamBoard · 예시)

> 가상 프로젝트 예시다(example/README.md). P3 유인 1차 실행의 리포트를
> templates/run-report.template.md 의 섹션 순서·제목 그대로 채운 모습.
> 증거 경로는 이 run 디렉터리(`runs/stage-20260713a/`) 기준. redaction 확인 후 확정된 상태다.

---

## 3줄 요약 (맨 위 고정 — 비전공자용)

- **신호등**: 🟡 (🟢 조치 불필요 / 🟡 확인 필요 / 🔴 즉시 조치)
- **조치 필요**: 있음 — [게이트3]에서 known-issue 등록 승인 1건(E-01) + SKIP 1건(F-02) 재실행 확인.
- **한 문장**: 핵심 기능(게시글·댓글·파일)은 정상 동작하지만, 파일 공유 만료일이 자정 경계에서
  하루 이르게 표시되는 버그 1건을 찾았습니다.

## run 정보

| 항목 | 값 |
|---|---|
| run-id / 디렉터리 | 20260713a (`runs/stage-20260713a/`) |
| 환경 | stage (등급 stage) |
| 모드 | 유인(P3 1차) — 대화형 브라우저 도구 경로 |
| 시작–종료 | 2026-07-13 18:20 – 20:02 |
| 프리플라이트 | 통과(헬스·도구·인증·이전 잔여물·.lock — 킬스위치·환경 fingerprint는 무인 run 한정 항목이라 해당 없음) |

## 정량 요약 (고정 필드)

| 케이스 수 | PASS | FAIL | SKIP | 플래키 |
|---|---|---|---|---|
| 15 | 13 | 1 | 1 | 0 |

| 실패 3분류(잠정) | 제품 버그 | 테스트 버그 | 문서 갭 | 미분류 |
|---|---|---|---|---|
| 건수 | 1 | 0 | 0 | 0 |

| 항목 | 값 |
|---|---|
| 오탐 수 | 1 — C-01 실행 중 "새 글이 목록 최상단에 없다"를 버그로 의심 → 관문②(의도된 설계)에서 기각: docs/spec/posts.md에 기본 정렬 "고정글 우선 + 관련도순" 명시. C-01 PASS 유지, retro 오탐률 추세 입력 |
| 실행 시간 | 102분 (상한 — 첫 run, 이 실측이 이후 기준) |
| AUTH 이슈 | AUTH_EXPIRED: member(수명 12h) — 유인 재로그인 → 세션 재저장(qa/.auth/member.state.json) 후 D-03 재실행 PASS. 케이스 FAIL로 계상하지 않음(phases/P4-retro.md 분류 규칙) |
| 차단 발동 | 0건 |
| run당 예산 | 외부 호출 38/100 · 리소스 생성 14/20 |
| 정리 검증 | 잔여 0건 확인(DB+외부) 2026-07-13 20:02 — 생성 14건 전 건 soft-delete(프로젝트 정책, CL-05), 스토리지는 점검창 종료 후 폴링 재시도로 확인. 확인 쿼리·조회는 inventory.json(cleaned·verified_at)과 함께 run 디렉터리에 보존 |

## 커버리지 신선도

| 항목 | 값 |
|---|---|
| 마지막 디스커버리 기준 | discovery 스냅샷 — main@2f4c9ab (2026-07-06) |
| 이번 run 대상 | main@2f4c9ab (동일) |
| 판정 | 신선(기준 일치) |

## 케이스별 결과

| 케이스 ID | 제목 | 결과 | 증거 | 비고 |
|---|---|---|---|---|
| A-01 | 제목 없이 게시글 등록이 거부된다 | PASS | [evidence/A-01/](evidence/A-01/) | 400 TITLE_REQUIRED + DB 미생성 확인 |
| A-02 | 제목 길이 경계(200자 허용/201자 거부) | PASS | [evidence/A-02/](evidence/A-02/) | 200자 → 201 / 201자 → 400 TITLE_TOO_LONG + DB 교차 확인 |
| A-03 | 빈 댓글 등록이 거부된다 | PASS | [evidence/A-03/](evidence/A-03/) | 400 + DB 미생성 확인 |
| A-04 | 파일 형식·용량 제한 거부 | PASS | [evidence/A-04/](evidence/A-04/) | 허용 외 형식 415, 용량 초과 413 + 스토리지 미생성 확인 |
| B-01 | 게시글 작성(대표 성공 경로) | PASS | [evidence/B-01/](evidence/B-01/) | 201 + 상세 화면 표시 + DB 1건 존재 |
| B-02 | 댓글 작성 | PASS | [evidence/B-02/](evidence/B-02/) | 201 + 목록 반영 + DB 1건 존재 |
| B-03 | 파일 업로드·공유 링크 발급 | PASS | [evidence/B-03/](evidence/B-03/) | 201 + 공유 링크 발급 + DB·스토리지 교차 확인 |
| C-01 | 게시글 검색·댓글 수 표시 | PASS | [evidence/C-01/](evidence/C-01/) | 검색 결과가 조건과 일치(응답 body 대조) — 오탐 1건 경위는 정량 요약 |
| C-02 | 목록 정렬 | PASS | [evidence/C-02/](evidence/C-02/) | 스냅샷 기준과 일치 — 회귀 기준(정답 아님) |
| D-01 | 타인 글 수정 차단(UI+API) | PASS | [evidence/D-01/](evidence/D-01/) | 403 + DB 무변경 + 수정 버튼 부재 |
| D-02 | 타인 댓글 삭제 차단 | PASS | [evidence/D-02/](evidence/D-02/) | 403 + DB 무변경 |
| D-03 | 비공유 파일 타인 접근 차단 | PASS | [evidence/D-03/](evidence/D-03/) | 404(비노출 정책) — AUTH 이슈로 재로그인 후 재실행 PASS |
| E-01 | 공유 링크 만료일 자정 경계 표시 | FAIL | [evidence/E-01/](evidence/E-01/) | → 실패 상세 |
| F-01 | 업로드 파일 스토리지 실물 존재 | PASS | [evidence/F-01/](evidence/F-01/) | 테스트 버킷 직접 조회로 실물 존재 확인 |
| F-02 | 파일 삭제 시 외부 스토리지 접근 불가 연동 | SKIP | — | 외부 스토리지 API 공지 점검창과 겹쳐 외부 직접 조회 불가 — FAIL 오염 방지 SKIP, 다음 run 재실행(케이스·오라클 변경 없음 — 무판정 이월) |

- 전 PASS 건 증거 확보(증거 없는 PASS 없음 — 원칙 2). EXPECTED-FAIL 해당 없음(첫 run — known-issue 미등록 상태).

## 실패 상세 (FAIL마다 1블록 — 비전공자 읽기용)

### E-01 공유 링크 만료일 자정 경계 표시

| 항목 | 내용 |
|---|---|
| 무슨 뜻인가 | 만료일을 KST 자정 정각(2026-07-20T00:00:00+09:00)으로 설정하면 저장(UTC 2026-07-19T15:00:00Z)은 정확하지만 화면이 "2026-07-19"로 표시된다 — UTC 날짜를 그대로 표시해 사용자가 만료일을 하루 이르게 오해할 수 있다. 저장 데이터 훼손은 없고 표시 변환 결함이다 |
| 심각도 | 중간 — 데이터는 정상이나 사용자 오해 소지(우회 가능·표시 수준 이상) |
| 권장 다음 행동 | [게이트3]에서 known-issues.md 등록 승인 → 등록 시 expected-fail 표기(개발 수정 전까지 "예상된 실패"로 리포트, 클린 런 비차단) → 이 블록을 증거와 함께 담당 개발자에게 그대로 전달 |
| 3분류(잠정) | 제품 버그 — 확정은 retro.md에서 3관문 통과 후 |
| 증거 | [evidence/E-01/](evidence/E-01/) — request.json(UTC 저장값), screenshot.png("2026-07-19" 표시), db-check.json, request-2.json(재현 2회차) |

**오탐 방지 3관문 기록(원칙 8 — 통과해야 "제품 버그 후보"로 상정)**

1. 필드 실존 — `expires_at`은 server/openapi.yaml 스키마에 실존, 실제 응답에도 존재 확인. ✅
2. 의도된 설계 여부 — docs/spec/files.md에 "만료일은 Asia/Seoul 기준으로 표시" 명시 → 의도 위반. ✅
3. 재현 2회 — 동일 조건 재실행 2회 동일 결과(evidence/E-01/request-2.json). ✅

## 전회 대비 diff

첫 run — 전회 없음.

## known-issue (expected-fail)

첫 run — 해당 없음(qa/known-issues.md 미등록 상태). E-01은 게이트3에서 등록 상정.

## 개선제안

| 제안 | 유형 | 처리 |
|---|---|---|
| E-01 known-issue 등록(만료일 자정 경계 표시 — 제품 버그 후보, 3관문 통과) | [정책] | **게이트 상정** — 승인 전 미적용 |

---

- redaction 스윕: 0건 — Authorization/Cookie 헤더 제거 + `secret_patterns`(tb-tok-\*·stg-key-\* 포함)
  재스캔 매칭 0건 확인 후 리포트 확정(원칙 6, patterns/evidence.md §3).
