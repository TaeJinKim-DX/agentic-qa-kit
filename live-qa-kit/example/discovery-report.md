# discovery-report.md — TeamBoard (P0 산출물 · 예시)

> 가상 프로젝트 예시다(example/README.md). 실제로는 `qa/discovery-report.md`로 생성된다 —
> 목차·순서는 phases/P0-discovery.md §7 그대로. 작성일 2026-07-06, 사용자에게 던진 질문 0건.

## 1. 요약

- 스택·기동법·API 표면(31 엔드포인트)·인증(사내 SSO)·DB·외부 스토리지 연동·타임존(저장 UTC ≠ 표시 Asia/Seoul)을 코드 근거로 파악했다.
- 파악 못 한 것 5건(stage URL·2FA 여부·DB 접속 계정·테스트 버킷·정리 정책) — P1 질문 재료다.
- 다음 단계: P1 인터뷰. deny-list 후보 3건·준비물 후보 6건을 초안으로 넘긴다.

## 2. 파악한 것

| # | 항목 | 파악 내용 | 근거 |
|---|---|---|---|
| 1 | 스택 | 프론트 React 18 + Vite(TypeScript), 백엔드 Node.js 20 + Express 4 + Prisma, npm. 프론트/백 분리(web/·server/) | web/package.json, server/package.json |
| 2 | 기동법 | ① `docker compose up -d db` ② `npm run migrate`(server/) ③ `npm run dev`(server/ :4000) ④ `npm run dev`(web/ :5173). 헬스 URL `GET /api/health` | README.md, docker-compose.yml, server/package.json |
| 3 | 라우트/API 표면 | 31 엔드포인트(GET 19 · non-GET 12) — server/openapi.yaml이 정본(코드와 대조 완료). 프론트 라우트 9개 | server/openapi.yaml, server/src/routes/, web/src/routes.tsx |
| 4 | 인증 | 사내 SSO(OIDC). 역할 admin/member 2종. 세션 쿠키 `tb-tok-*` 수명 12h. 캡차 없음 | server/src/auth/oidc.js, server/src/middleware/roles.js, server/src/auth/session.js |
| 5 | DB | PostgreSQL 16. posts·comments·files·share_links·users 등 9테이블, 시각 컬럼 전부 timestamptz. 삭제는 deleted_at 컬럼(soft-delete) | docker-compose.yml, server/prisma/schema.prisma |
| 6 | 외부 시스템 | ① 스토리지 API(파일 실물 저장 — 객체 생성·영구삭제 부작용, 키 `STORAGE_API_KEY`) ② SMTP 메일(공지 발송 부작용) | server/src/lib/storage-client.js, server/src/lib/mailer.js |
| 7 | 기존 QA/기능 문서 | 기능 명세 docs/spec/posts.md·files.md·permissions.md(케이스 1순위 소스), API 정본 server/openapi.yaml. 자동 테스트는 서버 유닛만(server/test/) — 라이브 QA 없음 | docs/spec/, server/test/ |
| 8 | 저장 vs 표시 타임존 | 저장 UTC(timestamptz + 서버 TZ=UTC), 표시 Asia/Seoul(formatKST) → **다름 — 시간 경계 케이스(E) 의무** | server/prisma/schema.prisma, web/src/lib/date.ts |

## 3. 파악 못 한 것

| 항목 | 시도한 것 | 알아내려면 필요한 것 |
|---|---|---|
| stage base_url | README "배포" 절에 `stage.teamboard.example` 언급 — 배포 설정 저장소가 분리되어 확정 불가 | 사용자 확인([필수입력] 후보 답으로 제시 — §8) |
| SSO 2FA 여부 | 코드에는 IdP 리다이렉트만 — 2FA는 IdP 정책이라 코드로 판별 불가 | IdP 운영 담당 확인(준비물 후보 — §6) |
| stage DB 접속 계정 | server/.env.example에 키 이름만(`DATABASE_URL`) — stage 접속 정보 없음 | DB 읽기 전용 계정 발급(준비물 후보 — assert-db 증거 수집용) |
| 스토리지 테스트 버킷 | storage-client.js는 버킷명을 env로 받음 — 테스트 전용 버킷 존재 여부 불명 | 스토리지 운영팀 확인(준비물 후보) |
| soft-delete purge 운영 주기 | 코드에 purge 배치 없음 — 운영 정책으로 추정 | 개발팀 확인(qa-config `data.retention` 결정) |

## 4. 위험 후보

**deny-list 초안 — non-GET 전수(12건)**

| 메서드·경로 | 하는 일 | 부작용 대상 | 불가역 |
|---|---|---|---|
| POST /api/auth/callback | SSO 콜백 | 세션 | |
| POST /api/posts | 게시글 생성 | DB | |
| PUT /api/posts/{id} | 게시글 수정 | DB | |
| DELETE /api/posts/{id} | 게시글 삭제(soft) | DB | |
| POST /api/posts/{id}/comments | 댓글 생성 | DB | |
| DELETE /api/comments/{id} | 댓글 삭제(soft) | DB | |
| POST /api/files | 파일 업로드 | DB + 스토리지(객체 생성) | |
| POST /api/files/{id}/share | 공유 링크 생성(만료일 지정) | DB | |
| DELETE /api/files/{id} | 파일 삭제(soft) | DB | |
| **DELETE /api/files/{id}/purge** | **파일 영구삭제 — 스토리지 객체까지 제거** | DB + 스토리지 | ★ 복구 불가 |
| **POST /api/notifications/announce** | **전체 공지 발송 — 전 구성원 메일** | SMTP(외부로 샘) | ★ 회수 불가 |
| **PATCH /api/users/{id}/role** | 역할 변경 | DB(실계정 오염 위험) | ★ 불가역 취급 권고 |

**UI 휴리스틱 후보**(대화형 경로 클릭 전 검사): "영구 삭제" 버튼, "전체 공지 발송" 버튼, "되돌릴 수 없습니다" 확인 다이얼로그.

**외부 부작용 목록**: 스토리지 객체 생성·삭제(과금 가능성), 메일 발송.

**prod 오인 위험 단서**: stage/prod가 같은 도메인 패턴(`*.teamboard.example`)이고 프론트에 환경 배지 없음 → 환경 fingerprint 검증 필요(`GET /api/health` 응답의 `env` 필드로 판별 가능 — server/src/routes/health.js).

**secret_patterns 후보**: `tb-tok-[A-Za-z0-9]{24}`(세션 토큰), `stg-key-[A-Za-z0-9]{32}`(스토리지 API 키).

## 5. 능력 프로파일

(정본은 qa/STATE.md — 여기는 사본)

| 능력 | 값 | 검증 |
|---|---|---|
| 브라우저 자동화 | 대화형 브라우저 도구 | about:blank 스크린샷 1장 성공 |
| 구조화 질문 UI | 없음(채팅 번호 질문) | 도구 목록 확인 |
| 헤드리스 재호출 수단 | 있음 — 호출 커맨드는 STATE.md에 기록(에이전트 CLI별 상이) | 무해한 1회 호출 성공 |
| 스케줄 수단 | cron | `crontab -l` 1회(등록 안 함 — 원칙 7) |
| Node.js | v20.11 | `node -v` |

## 6. 준비물 후보 (clearance 초안)

| 분류 | 후보 | 왜(어느 페이즈·케이스) | 요청 대상 추정 |
|---|---|---|---|
| 계정 | SSO 테스트 계정 2종(admin 1·member 1) | P3~ 전 케이스의 로그인 전제, 권한(D)의 축 | 플랫폼팀(IdP 운영) |
| 키 | 스토리지 API 키(테스트 버킷 스코프) | 교차검증(F)·시딩·정리검증 — P3~P5 | 스토리지 운영팀 |
| 접근 | stage 내부망(VPN) 개통 | 전 케이스 전제 — P2 UI 크롤링부터 | 인프라팀 |
| 접근 | stage DB 읽기 전용 계정 | assert-db 증거 — P3~P5 | TeamBoard 개발팀 |
| 외부권한 | 테스트 버킷의 객체 생성·삭제 권한 | 시딩·정리 — P3~P5(API 키 스코프에 포함 가능 — P1에서 확인) | 스토리지 운영팀 |
| 승인 | soft-delete purge 주기·테스트 데이터 정리 정책 | qa-config `data.retention` 확정 — P1 | TeamBoard 개발팀 |

## 7. 기동 시도 기록

- 타임박스 10분, 실소요 7분. `docker compose up -d db` → `npm run migrate` → server :4000 기동 → `GET /api/health` 200 확인 → web :5173 기동. 확인 후 프로세스 종료 완료.
- 단, 사내 IdP에 로컬 콜백 URL이 미등록이라 **로컬 로그인은 불가** — P1에서 대상이 stage로 정해지면 로컬 기동은 불필요(URL 헬스체크로 대체).

## 8. P1 인터뷰 후보 답

| 질문(P1) | 후보 답(근거) |
|---|---|
| [필수입력] 대상 환경 base_url | `https://stage.teamboard.example`(README "배포" 절 — 확정은 사용자) |
| [필수입력] 테스트 계정 | 현재 없음 — 발급 필요(준비물 후보, §6) |
| 역할 목록 | admin / member(server/src/middleware/roles.js) |
| deny-list | §4 불가역 3건(파일 영구삭제·전체 공지 발송·역할 변경) |
| test_prefix | `qa-`(기존 데이터와 충돌 없음 확인) |
| 타임존 | 저장 UTC / 표시 Asia/Seoul → E 의무 |
| 우선 기능 | 게시글·댓글·파일 공유(라우트 밀도·문서 커버 기준) |
| 정리 방식 | soft-delete(deleted_at) — purge 주기는 정책 확인 필요(§3) |
