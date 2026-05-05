# Team CalTalk 구현 실행계획

> **버전**: v1.0 | **작성일**: 2026-05-05 | **기준 범위**: v1.0 MVP (Modular Monolith)
> **기술 스택**: Node.js + TypeScript + Express + Prisma + PostgreSQL / React 18 + Vite + TanStack Query + Zustand

---

## 전체 태스크 개요

| 영역 | 태스크 수 | 예상 시간 |
|------|---------|---------|
| 데이터베이스 (DB) | 16개 | ~11시간 |
| 백엔드 (BE) | 34개 | ~108시간 |
| 프론트엔드 (FE) | 39개 | ~92시간 |
| **합계** | **89개** | **~211시간** |

---

## 태스크 간 전체 의존 흐름

```
DB-001 → DB-002 → DB-003 → DB-004~010 (병렬)
                              └→ DB-011 → DB-012 → DB-013 → DB-014, DB-016
DB-002 → DB-015

BE-001 → BE-003 → BE-007 → 도메인별 Repository (BE-011, BE-014, BE-017, BE-020, BE-023, BE-026)
       → BE-004 → BE-005 → BE-006 (EventBus)
                          → BE-008 → BE-009 → BE-010
       → BE-002 (병렬)

FE-001 → FE-002 → FE-003 → Feature별 API (FE-009, FE-013, FE-017, FE-023, FE-028)
                → FE-004
       → FE-006, FE-008 (병렬)
FE-003 + FE-004 → FE-005
```

---

## Part 1. 데이터베이스 (DB)

### DB-001 · 데이터베이스 환경 설정

**상세 작업**
- `backend/.env.example`: `DATABASE_URL`, `DATABASE_URL_TEST` 변수 정의
- `docker-compose.dev.yml`: postgres:16-alpine 서비스 구성 (포트 5432, 볼륨 마운트)
- `backend/src/shared/config/env.ts`: zod로 `DATABASE_URL` 환경변수 검증
- `.gitignore`에 `.env.development`, `.env.production`, `.env.test` 추가 (`.env.example` 제외)

**완료 조건**
- [ ] `.env.example`에 `DATABASE_URL`, `DATABASE_URL_TEST` 항목 존재
- [ ] `docker-compose.dev.yml` 실행 후 PostgreSQL 접속 성공
- [ ] 잘못된 `DATABASE_URL` 입력 시 앱 기동 실패 (zod 오류)
- [ ] `.gitignore`에 실제 `.env` 파일 포함, `.env.example`은 git 추적

**의존성**: 없음 | **예상 소요시간**: 1h

---

### DB-002 · Prisma 초기화 및 기본 설정

**상세 작업**
- `npm install prisma @prisma/client` 설치
- `npx prisma init --datasource-provider postgresql` 실행
- `prisma/schema.prisma` generator/datasource 블록 설정 (output 경로 명시)
- `package.json`에 DB 스크립트 6개 등록: `db:migrate:dev`, `db:migrate:deploy`, `db:migrate:reset`, `db:generate`, `db:studio`, `db:seed`
- `src/generated/prisma/` → `.gitignore` 추가

**완료 조건**
- [ ] `prisma/schema.prisma` 파일 존재, `postgresql` datasource 설정
- [ ] `npx prisma generate` 오류 없이 완료
- [ ] `package.json`에 DB 스크립트 6개 등록
- [ ] `prisma/migrations/` 디렉토리 git 추적

**의존성**: DB-001 | **예상 소요시간**: 0.5h

---

### DB-003 · Prisma Enum 정의

**상세 작업**
`prisma/schema.prisma`에 6개 enum 정의 (UPPER_SNAKE_CASE):
```
UserStatus      — ACTIVE | INACTIVE
TeamStatus      — ACTIVE | DELETED
TeamRole        — TEAM_LEADER | TEAM_MEMBER
InvitationStatus — PENDING | ACCEPTED | DECLINED | EXPIRED
ScheduleStatus  — SCHEDULED | DONE | CANCELLED
NotificationType — MESSAGE_RECEIVED | INVITATION_RECEIVED | INVITATION_RESPONDED
```

**완료 조건**
- [ ] 6개 enum 블록 모두 `schema.prisma`에 존재
- [ ] `npx prisma validate` 통과
- [ ] 도메인 정의서 2.1~2.6절 상태값과 1:1 일치

**의존성**: DB-002 | **예상 소요시간**: 0.5h

---

### DB-004 · User + RefreshToken 모델 정의

**상세 작업**
- `User` 모델: `id(uuid)`, `name`, `email(unique)`, `passwordHash`, `status(UserStatus)`, `createdAt`, `updatedAt` + 관계 필드
- `RefreshToken` 모델: `id`, `userId(FK)`, `tokenHash`, `expiresAt`, `revokedAt(nullable)`, `createdAt`
- 전체 `DateTime`에 `@db.Timestamptz` (UTC 저장 원칙)
- `@@map("users")`, `@@map("refresh_tokens")`

**완료 조건**
- [ ] `User.email` — `@unique` 제약 적용 (BR-15)
- [ ] `RefreshToken` — `User`와 `1:N` 관계, `onDelete: Cascade`
- [ ] 모든 `DateTime` 필드에 `@db.Timestamptz`
- [ ] `npx prisma validate` 통과

**의존성**: DB-003 | **예상 소요시간**: 0.5h

---

### DB-005 · Team + UserTeamRole 모델 정의

**상세 작업**
- `Team` 모델: `id`, `name`, `status(TeamStatus=ACTIVE)`, `createdAt`, `updatedAt`
- `UserTeamRole` 모델: `id`, `userId(FK)`, `teamId(FK)`, `role(TeamRole)`, `createdAt`
- `@@unique([userId, teamId])` — 팀당 역할 하나
- `@@index([teamId, role])` — 팀장 목록 조회 최적화 (BR-80)
- `onDelete: Cascade` 양쪽 FK

**완료 조건**
- [ ] `Team.status` 소프트 삭제 지원 (OQ-03)
- [ ] `UserTeamRole.@@unique([userId, teamId])` 제약 적용
- [ ] `@@index([teamId, role])` 정의
- [ ] `npx prisma validate` 통과

**의존성**: DB-003, DB-004 | **예상 소요시간**: 0.5h

---

### DB-006 · Invitation 모델 정의

**상세 작업**
- `Invitation` 모델: `id`, `teamId(FK)`, `inviterId(FK)`, `inviteeId(FK)`, `status(InvitationStatus=PENDING)`, `createdAt`, `expiresAt`
- `inviter`/`invitee` → named relation으로 `User` 분리 참조
- `@@index([teamId, inviteeId, status])` — BR-25 중복 초대 검증
- `@@index([inviteeId, status])` — 수신 초대 목록 조회
- `expiresAt` 기본값은 앱 서비스 레이어에서 계산 (OQ-05 지연 평가)

**완료 조건**
- [ ] `inviter`, `invitee` 두 FK가 별도 named relation으로 `User` 참조
- [ ] `@@index([teamId, inviteeId, status])` 정의
- [ ] `expiresAt` nullable 아님
- [ ] `npx prisma validate` 통과

**의존성**: DB-003, DB-004, DB-005 | **예상 소요시간**: 0.5h

---

### DB-007 · Schedule + ScheduleAssignee 모델 정의

**상세 작업**
- `Schedule` 모델: `id`, `teamId(FK)`, `createdBy(FK)`, `title`, `startAt`, `endAt`, `status(ScheduleStatus=SCHEDULED)`, **`version(Int=0)`**, `createdAt`, `updatedAt`
- `ScheduleAssignee` 모델: `scheduleId(FK)`, `userId(FK)` — `@@id([scheduleId, userId])` 복합 PK
- `@@index([teamId, startAt, endAt])` — 날짜 범위 조회 (BR-40)
- `ScheduleAssignee` 양쪽 FK `onDelete: Cascade` (BR-75 담당자 자동 제거)

**완료 조건**
- [ ] `Schedule.version Int @default(0)` 존재 (낙관적 잠금 OQ-06)
- [ ] `ScheduleAssignee.@@id([scheduleId, userId])` 복합 PK
- [ ] `@@index([teamId, startAt, endAt])` 정의
- [ ] `ScheduleAssignee` 양쪽 `onDelete: Cascade`
- [ ] `npx prisma validate` 통과

**의존성**: DB-003, DB-004, DB-005 | **예상 소요시간**: 0.75h

---

### DB-008 · Chat + Message 모델 정의

**상세 작업**
- `Chat` 모델: `id`, `teamId(FK)`, `chatDate(@db.Date)`, `createdAt`
- `@@unique([teamId, chatDate])` — 날짜별 채팅방 1개 (BR-50)
- `Message` 모델: `id`, `chatId(FK)`, `authorId(FK)`, `content(@db.Text)`, `createdAt`
- `Message`에 `updatedAt` **의도적 제외** (BR-35 불변성 선언)
- `@@index([chatId, createdAt])` — 시간순 조회
- `@@index([chatId, id])` — Long Polling `after={lastId}` 쿼리

**완료 조건**
- [ ] `Chat.chatDate` — `@db.Date` 타입 (시간 없음, BR-50)
- [ ] `@@unique([teamId, chatDate])` 제약 정의
- [ ] `Message` 모델에 `updatedAt` 필드 없음 (BR-35)
- [ ] `@@index([chatId, id])` 정의
- [ ] `npx prisma validate` 통과

**의존성**: DB-003, DB-004, DB-005 | **예상 소요시간**: 0.5h

---

### DB-009 · Notification 모델 정의

**상세 작업**
- `Notification` 모델: `id`, `recipientId(FK)`, `type(NotificationType)`, `referenceId(@db.Uuid — FK 없음, 다형성)`, `isRead(Boolean=false)`, `createdAt`
- `@@index([recipientId, isRead])` — 미읽음 카운트 최적화 (BR-90)
- `@@index([recipientId, createdAt])` — 알림 목록 최신순
- 알림 삭제 없음 (도메인 정의서 2.6절)

**완료 조건**
- [ ] `referenceId` — FK 없는 `String @db.Uuid` 단독 필드 (다형성)
- [ ] `@@index([recipientId, isRead])` 정의
- [ ] `isRead @default(false)` 설정
- [ ] `npx prisma validate` 통과

**의존성**: DB-003, DB-004 | **예상 소요시간**: 0.5h

---

### DB-010 · AuditLog 모델 정의

**상세 작업**
- `AuditLog` 모델: `id`, `actorId?(nullable FK)`, `actorIp?(@db.VarChar(45))`, `action(@db.VarChar(100))`, `targetType?`, `targetId?(@db.Uuid)`, `metadata?(@db.JsonB)`, `createdAt`
- `actorId` → `onDelete: SetNull` (사용자 삭제 후에도 이력 보존)
- `@@index([action, createdAt])`, `@@index([targetType, targetId])`
- 마이그레이션 SQL에 `REVOKE UPDATE, DELETE ON audit_logs` 수동 추가

**완료 조건**
- [ ] `actorId` nullable (`String?`)
- [ ] `metadata @db.JsonB` 타입
- [ ] `onDelete: SetNull` 관계 설정
- [ ] 마이그레이션 SQL에 `REVOKE UPDATE, DELETE` 포함
- [ ] `npx prisma validate` 통과

**의존성**: DB-003, DB-004 | **예상 소요시간**: 0.5h

---

### DB-011 · 인덱스 최적화 검토

**상세 작업**
아키텍처 원칙 5.4절 필수 인덱스 전체 존재 여부 확인:

| 인덱스 | 테이블 | 컬럼 | 목적 |
|--------|--------|------|------|
| `idx_messages_team_date` | `messages` (chats 조인) | `chat_id, created_at` | BR-40, BR-50 |
| `idx_schedules_team_date` | `schedules` | `team_id, start_at, end_at` | BR-40 |
| `idx_notifications_user_read` | `notifications` | `recipient_id, is_read` | BR-90 |
| `idx_invitations_team_user_status` | `invitations` | `team_id, invitee_id, status` | BR-25 |
| `idx_messages_chat_id` | `messages` | `chat_id, id` | Long Polling |
| `idx_user_team_roles_team_role` | `user_team_roles` | `team_id, role` | BR-80 |

**완료 조건**
- [ ] 아키텍처 원칙 5.4절 필수 4개 인덱스 모두 `schema.prisma`에 존재
- [ ] Long Polling 지원 `@@index([chatId, id])` 존재 (DB-008 확인)
- [ ] `npx prisma validate` 통과

**의존성**: DB-004~DB-010 | **예상 소요시간**: 0.75h

---

### DB-012 · 유니크 제약 및 Check Constraint

**상세 작업**
- Prisma `@@unique`: `users.email`, `user_team_roles.(userId,teamId)`, `chats.(teamId,chatDate)` 확인
- 마이그레이션 SQL에 Check Constraint 수동 추가:
  ```sql
  ALTER TABLE schedules ADD CONSTRAINT chk_schedule_end_after_start CHECK (end_at >= start_at);
  ALTER TABLE schedules ADD CONSTRAINT chk_schedule_title_not_empty CHECK (char_length(trim(title)) > 0);
  ALTER TABLE invitations ADD CONSTRAINT chk_invitation_expires_after_created CHECK (expires_at > created_at);
  ```

**완료 조건**
- [ ] 3개 `@@unique` 제약 마이그레이션에 반영
- [ ] `chk_schedule_end_after_start` Check Constraint 포함 (BR-70)
- [ ] `chk_schedule_title_not_empty` Check Constraint 포함
- [ ] DB에서 직접 `end_at < start_at` INSERT 시도 시 오류 발생 확인

**의존성**: DB-005~DB-008 | **예상 소요시간**: 0.5h

---

### DB-013 · 초기 마이그레이션 실행 및 검증

**상세 작업**
1. `npx prisma validate` + `npx prisma format`
2. `npx prisma migrate dev --name init_all_tables` 실행
3. 생성된 `migration.sql`에 `REVOKE`, Check Constraint SQL 수동 추가
4. `npx prisma migrate reset` → 전체 초기화 후 재적용 확인
5. 11개 테이블 존재 확인: `users, refresh_tokens, teams, user_team_roles, invitations, schedules, schedule_assignees, chats, messages, notifications, audit_logs`
6. `npx prisma generate` 실행

**완료 조건**
- [ ] `prisma/migrations/` 디렉토리에 마이그레이션 파일 존재
- [ ] `npx prisma migrate status` — `Database schema is up to date` 확인
- [ ] 11개 테이블 모두 DB에 생성
- [ ] Check Constraint 2개 DB에 존재
- [ ] `migration.sql` 파일 git 커밋

**의존성**: DB-003~DB-012 | **예상 소요시간**: 1h

---

### DB-014 · 시드 데이터 작성

**상세 작업**
`backend/src/db/seeds/seed.ts` 작성:
- 사용자 3명 (leader@test.com, member1@test.com, member2@test.com), bcrypt 해시 저장 (salt rounds=12)
- 팀 1개, UserTeamRole 2개 (LEADER, MEMBER)
- Invitation 1건 (PENDING)
- Schedule 3건 (SCHEDULED, DONE, CANCELLED), version=0
- ScheduleAssignee 1건
- Chat 1개 + Message 3건 (오늘 날짜)
- Notification 1건 (is_read=false)

**완료 조건**
- [ ] `npx prisma db seed` 성공
- [ ] 비밀번호 bcrypt 해시로 저장 (평문 아님)
- [ ] `npx prisma migrate reset` 후 자동 seed 재실행 성공
- [ ] 모든 테이블 데이터 정상 확인

**의존성**: DB-013 | **예상 소요시간**: 1.5h

---

### DB-015 · Prisma Client 싱글톤 구현

**상세 작업**
`backend/src/db/client.ts`:
- 개발 환경 HMR 중복 연결 방지를 위한 글로벌 캐싱 패턴
- `NODE_ENV=development` — `['query', 'error', 'warn']` 로그
- `NODE_ENV=production` — `['error']` 로그
- `SIGTERM` 핸들러에서 `prisma.$disconnect()` 등록
- `GET /health`에서 `prisma.$queryRaw\`SELECT 1\`` DB 상태 확인

**완료 조건**
- [ ] 싱글톤 패턴 구현 (`globalThis` 캐싱)
- [ ] `NODE_ENV` 별 로그 레벨 적용
- [ ] `GET /health` — `{ db: "connected" }` 응답
- [ ] `SIGTERM` 핸들러에서 `prisma.$disconnect()` 호출

**의존성**: DB-002 | **예상 소요시간**: 0.5h

---

### DB-016 · 마이그레이션 전략 문서화

**상세 작업**
운영 마이그레이션 절차, 롤백 전략, CI/CD 통합:
- 환경별 마이그레이션 명령 정리 (dev/ci/prod)
- 롤백: 새 마이그레이션으로 역방향 DDL 작성 원칙
- 컬럼 삭제 2단계 전략 (NULLABLE → 다음 배포 삭제)
- Testcontainers 통합 테스트 setup에 `migrate deploy` 자동 실행 코드

**완료 조건**
- [ ] 마이그레이션 운영 절차 문서화 (README 또는 CLAUDE.md)
- [ ] Testcontainers setup에 마이그레이션 자동 적용 코드 존재
- [ ] `prisma/migrations/` 모든 파일 git 포함

**의존성**: DB-013 | **예상 소요시간**: 1h

---

## Part 2. 백엔드 (BE)

### Phase 0 — 프로젝트 기반 설정

---

### BE-001 · 프로젝트 초기화 및 TypeScript 설정

**상세 작업**
- `package.json`: `dev`, `build`, `start`, `test:unit`, `test:integration`, `test:e2e` 스크립트
- `tsconfig.json`: `strict: true`, `target: ES2022`, `paths` 별칭 (`@domains/*`, `@shared/*`, `@db/*`)
- `.env.example`: `DATABASE_URL`, `JWT_SECRET`, `JWT_EXPIRES_IN`, `REFRESH_TOKEN_EXPIRES_IN`, `CORS_ORIGIN`, `LONG_POLL_TIMEOUT_MS`, `PORT` 키
- `nodemon.json` 또는 `ts-node-dev` 핫 리로드 설정

**완료 조건**
- [ ] `npm run dev` 실행 시 `src/server.ts` 기동
- [ ] TypeScript strict 모드에서 컴파일 오류 없음
- [ ] `.env.example`에 모든 환경변수 키 존재
- [ ] `dist/` 빌드 결과물 생성

**의존성**: 없음 | **예상 소요시간**: 2h

---

### BE-002 · ESLint + Prettier + 도메인 격리 규칙

**상세 작업**
- `eslint.config.js`: `@typescript-eslint/recommended`, `no-restricted-imports` (도메인 간 직접 import 차단), `no-explicit-any` error, `no-console` error
- `.prettierrc`: `printWidth: 100`, `singleQuote: true`, `trailingComma: all`
- Husky + lint-staged: pre-commit hook으로 staged 파일 자동 lint + format

**완료 조건**
- [ ] `domains/team`에서 `domains/auth` 직접 import → ESLint 오류
- [ ] `npm run lint` CI 통과
- [ ] pre-commit hook 자동 실행

**의존성**: BE-001 | **예상 소요시간**: 2h

---

### BE-003 · Prisma 스키마 및 초기 마이그레이션

> DB 파트의 DB-002~DB-013 작업 결과물을 백엔드 프로젝트에 통합

**상세 작업**
- DB-002~DB-013에서 생성한 `prisma/schema.prisma` 및 마이그레이션 파일을 `backend/` 구조에 배치
- `npx prisma generate`로 `src/generated/prisma/` 타입 파일 생성 확인

**완료 조건**
- [ ] `npx prisma validate` 오류 없음
- [ ] `npx prisma generate` 성공
- [ ] 모든 필수 인덱스 마이그레이션 SQL에 포함
- [ ] `Schedule.version` 낙관적 잠금 필드 존재

**의존성**: BE-001, DB-013 | **예상 소요시간**: 0.5h (DB 작업 전제)

---

### BE-004 · Express 앱 초기화 및 공통 미들웨어

**상세 작업**
`src/app.ts` (테스트 가능하도록 `src/server.ts`와 분리):
- `helmet()`: HSTS, X-Frame-Options: DENY, Referrer-Policy, CSP 설정
- `cors()`: `CORS_ORIGIN` 환경변수 기반 화이트리스트
- `express.json({ limit: '100kb' })` body 크기 제한
- pino-http 요청 로깅 (`requestId`, `userId`, `teamId`, `method`, `path`, `statusCode`, `durationMs`)
- `GET /health`: Prisma DB ping 포함
- 전역 라우터 및 에러 핸들러 등록 (마지막에 등록)

`src/server.ts`: `app.listen(PORT)`, SIGTERM/SIGINT graceful shutdown

**완료 조건**
- [ ] `GET /health` — `{ data: { status: "ok", db: "connected" } }`
- [ ] 100KB 초과 body → 413
- [ ] helmet 헤더 응답에 포함
- [ ] 테스트에서 `app` 직접 import + supertest 사용 가능

**의존성**: BE-001, BE-003 | **예상 소요시간**: 3h

---

### BE-005 · 공통 인프라 — 에러 클래스, 에러 핸들러, 로거

**상세 작업**
`shared/errors/app-error.ts`: `AppError` + 파생 클래스 (400/401/403/404/405/409/429)

`shared/errors/error-handler.ts`:
- `AppError` → `{ error: { code, message } }` 직렬화
- `ZodError` → 400 + 필드별 검증 오류
- Prisma P2002 → 409 자동 변환
- 예상치 못한 에러 → 500 (스택 프로덕션 비노출)

`shared/logger/logger.ts`: pino 싱글톤 (레벨: 환경변수 기반, 테스트: silent)

`shared/config/env.ts`: zod 스키마로 모든 환경변수 검증, 실패 시 앱 즉시 종료

**완료 조건**
- [ ] `ZodError` → 400 필드 오류 직렬화
- [ ] Prisma P2002 → 409 자동 변환 테스트 통과
- [ ] `JWT_SECRET` 미설정 시 앱 기동 실패
- [ ] `console.log` ESLint 오류 발생

**의존성**: BE-001, BE-004 | **예상 소요시간**: 3h

---

### BE-006 · EventBus 인프라 구현

**상세 작업**
`shared/events/event-bus.interface.ts`: `EventBus` 인터페이스 (emit, on)

`shared/events/contracts.ts` — 이벤트 카탈로그:
- `message.created`: `{ teamId, messageId, authorId, chatDate }`
- `invitation.sent`: `{ teamId, invitationId, inviteeId }`
- `invitation.responded`: `{ teamId, invitationId, inviterId, status }`

`shared/events/event-bus.ts`: Node.js EventEmitter 기반 인메모리 구현체
- 이벤트 발행은 **트랜잭션 커밋 이후** (아키텍처 원칙 2.3)
- 핸들러 실패 시 최대 3회 재시도 (지수 백오프)

**완료 조건**
- [ ] `eventBus.emit('message.created', payload)` 후 핸들러 호출 확인
- [ ] 핸들러 오류 시 3회 재시도 후 로그 기록 (단위 테스트)
- [ ] 이벤트 페이로드 TypeScript 컴파일 타임 검증
- [ ] Mock 구현체로 교체 가능 (테스트용)

**의존성**: BE-005 | **예상 소요시간**: 3h

---

### BE-007 · Prisma Client 싱글톤 및 DB 연결

> DB-015 작업 결과물을 백엔드에 통합

**상세 작업**
- `src/db/client.ts`: DB-015 싱글톤 패턴 구현
- `src/db/seeds/`: DB-014 시드 데이터 확인
- 테스트 환경에서 `.env.test` DB URL 사용

**완료 조건**
- [ ] `GET /health`에서 DB `connected` 상태 반환
- [ ] 테스트 환경에서 별도 DB URL 사용
- [ ] `npx prisma db seed` 성공

**의존성**: BE-003, BE-004 | **예상 소요시간**: 1h

---

### Phase 1 — 보안 미들웨어 및 인증 인프라

---

### BE-008 · JWT 인증 미들웨어 + Rate Limiting

**상세 작업**
`shared/middlewares/auth.middleware.ts`:
- `Authorization: Bearer <token>` 헤더 파싱
- `jsonwebtoken` — HS256만 허용, `alg: none` 거부
- 클레임 검증: `sub`, `iss`, `aud`, `exp`, `iat`, `jti`
- 만료 토큰 → 401 + code `TOKEN_EXPIRED`

`shared/middlewares/rate-limit.middleware.ts` (아키텍처 원칙 5.2):
- 전역: IP당 분당 100 요청
- 로그인: IP + 이메일당 분당 5회
- 회원가입: IP당 시간당 5회
- 채팅 메시지: 사용자당 초당 5건, 분당 60건
- 위반 시 429 + `Retry-After` 헤더

**완료 조건**
- [ ] 유효한 JWT → `req.user` 주입 확인
- [ ] `alg: none` 또는 잘못된 시그니처 → 401
- [ ] 만료 토큰 → 401 + `TOKEN_EXPIRED`
- [ ] 로그인 6회 시도 → 429 + `Retry-After` 헤더

**의존성**: BE-005 | **예상 소요시간**: 3h

---

### BE-009 · 팀 소속 검증 가드 (TeamMemberGuard)

**상세 작업**
`shared/middlewares/team-member.guard.ts`:
- `req.params.teamId` → DB에서 `UserTeamRole` 조회
- 미소속 → `ForbiddenError` (403, `FORBIDDEN_TEAM_ACCESS`)
- 성공 → `req.teamMembership = { teamId, role }` 주입
- 팀 `DELETED` 상태 → 403
- IDOR 방어: URL `teamId` 신뢰 안 함

미들웨어 체인: `authenticate → TeamMemberGuard → authorize(role)`

감사 로그: 403 발생 시 `forbidden_team_access` 이벤트 기록

**완료 조건**
- [ ] 미소속 팀 접근 → 403 + `FORBIDDEN_TEAM_ACCESS`
- [ ] 삭제된 팀 접근 → 403
- [ ] 정상 소속 → `req.teamMembership` 주입
- [ ] IDOR 방어: 타 사용자 `teamId` 요청 → 403 (통합 테스트)

**의존성**: BE-007, BE-008 | **예상 소요시간**: 3h

---

### BE-010 · RBAC 인가 미들웨어

**상세 작업**
`shared/middlewares/authorize.middleware.ts`:
- `req.teamMembership.role !== requiredRole` → `ForbiddenError` (403, `INSUFFICIENT_ROLE`)
- 서비스 레이어에서 actor role 재검증 원칙 문서화

**완료 조건**
- [ ] 팀원 계정으로 `TEAM_LEADER` 전용 엔드포인트 → 403 (BR-20)
- [ ] 팀장 계정으로 동일 → 통과
- [ ] 403 발생 시 `audit_logs`에 이벤트 삽입 확인

**의존성**: BE-009 | **예상 소요시간**: 2h

---

### Phase 2 — Auth 도메인

---

### BE-011 · Auth Repository

**상세 작업**
`src/domains/auth/auth.repository.ts`:
- `findUserByEmail`, `findUserById`, `createUser`
- `saveRefreshToken`, `findRefreshToken(tokenHash)`, `revokeRefreshToken`, `revokeAllUserRefreshTokens`
- DB row → `UserEntity`, `RefreshTokenEntity` 매핑

**완료 조건**
- [ ] `findUserByEmail` — 미존재 이메일 → `null`
- [ ] `createUser` — 중복 이메일 → Prisma P2002 (에러 핸들러에서 409)
- [ ] `revokeAllUserRefreshTokens` — `revoked_at` 업데이트
- [ ] Testcontainers PostgreSQL 통합 테스트 통과

**의존성**: BE-007 | **예상 소요시간**: 2h

---

### BE-012 · Auth Service

**상세 작업**
`src/domains/auth/auth.service.ts`:

**회원가입** (UC-00, BR-15): 중복 이메일 → 409, `bcrypt.hash(password, 12)`, User 생성

**로그인** (UC-01): 이메일 존재 여부 비노출, bcrypt 비교, Access Token (JWT HS256, `sub/iss/aud/exp/iat/jti` 클레임), Refresh Token (불투명, bcrypt 해시 저장), 감사 로그

**토큰 재발급**: Refresh Token 쿠키 → DB 조회 → 검증 → Rotation (기존 폐기 + 신규 발급)

**로그아웃** (UC-01): Refresh Token 폐기 + 쿠키 삭제, 감사 로그

**완료 조건**
- [ ] 중복 이메일 → 409 `DUPLICATE_EMAIL` (BR-15)
- [ ] 로그인 → Access Token body + Refresh Token `HttpOnly` 쿠키
- [ ] 미존재 이메일 → 401 `INVALID_CREDENTIALS` (이메일 노출 없음)
- [ ] 로그아웃 후 Refresh Token 재발급 시도 → 401
- [ ] Access Token 만료 + 유효 Refresh Token → 새 토큰 발급

**의존성**: BE-011 | **예상 소요시간**: 4h

---

### BE-013 · Auth Controller, DTO, Router

**상세 작업**
`src/domains/auth/auth.dto.ts`: Zod 스키마 (RegisterSchema, LoginSchema)

엔드포인트:
- `POST /auth/register` → signupRateLimit → 검증 → `authService.register()`
- `POST /auth/login` → loginRateLimit → 검증 → `authService.login()`
- `POST /auth/refresh` → `authService.refreshToken()`
- `POST /auth/logout` → authenticate → `authService.logout()`

**완료 조건**
- [ ] `POST /auth/register` — 형식 불일치 → 400
- [ ] `POST /auth/login` — 성공 시 `{ data: { accessToken, user } }`
- [ ] `POST /auth/refresh` — 쿠키 Refresh Token → 새 Access Token
- [ ] `POST /auth/logout` — 인증 미들웨어 통과 필수, 성공 204

**의존성**: BE-012, BE-008 | **예상 소요시간**: 2h

---

### Phase 3 — Team 도메인

---

### BE-014 · Team + Invitation Repository

**상세 작업**
`src/domains/team/team.repository.ts`: `createTeam`, `addTeamMember`, `findTeamById`, `findTeamsByUserId`, `findTeamMembership`, `findTeamMembers`, `countLeaders`

`src/domains/team/invitation.repository.ts`: `createInvitation`, `findPendingInvitation`, `findInvitationById`, `findInvitationsByRecipient`, `updateInvitationStatus`

**완료 조건**
- [ ] `createTeam` + `addTeamMember` 트랜잭션 단위 테스트 (Testcontainers)
- [ ] `findTeamsByUserId` — 소속 팀만 반환
- [ ] `countLeaders` — 팀장 수 정확히 반환

**의존성**: BE-007 | **예상 소요시간**: 3h

---

### BE-015 · Team Service

**상세 작업**
- **팀 생성** (UC-02): `prisma.$transaction` — Team + UserTeamRole(TEAM_LEADER) 원자적 삽입
- **팀원 초대** (UC-03, BR-25, BR-90): TEAM_LEADER 재검증, 중복 Pending 확인 → 409, 만료일 = now+7일, 트랜잭션 후 `invitation.sent` 이벤트 발행
- **초대 수락/거절** (UC-09): 만료 여부 검증, 수락 → `prisma.$transaction` (Accepted + UserTeamRole), `invitation.responded` 이벤트 발행
- `TeamMembershipGuard` 인터페이스 export

**완료 조건**
- [ ] 팀 생성 — 원자적 삽입, 실패 시 롤백
- [ ] 중복 Pending 초대 → 409 `DUPLICATE_INVITATION` (BR-25)
- [ ] 만료 초대 수락 → `EXPIRED_INVITATION`
- [ ] 초대 수락 후 `invitation.responded` 이벤트 발행 확인

**의존성**: BE-014, BE-006 | **예상 소요시간**: 5h

---

### BE-016 · Team Controller, DTO, Router

**상세 작업**
엔드포인트:
```
POST   /teams                       — 팀 생성 (인증)
GET    /teams                       — 내 팀 목록
GET    /teams/:teamId               — 팀 상세 (팀 멤버만)
GET    /teams/:teamId/members       — 팀원 목록
POST   /teams/:teamId/invitations   — 팀원 초대 (팀장만)
GET    /invitations                 — 내 수신 초대 목록
PATCH  /invitations/:id/respond     — 초대 수락/거절
```

**완료 조건**
- [ ] `POST /teams` — 201 + `{ data: { id, name, role: 'TEAM_LEADER' } }`
- [ ] `GET /teams` — 소속 팀만 반환
- [ ] 팀원 계정으로 `POST /teams/:teamId/invitations` → 403

**의존성**: BE-015, BE-009, BE-010 | **예상 소요시간**: 3h

---

### Phase 4 — Schedule 도메인

---

### BE-017 · Schedule Repository

**상세 작업**
- `createSchedule`, `findScheduleById`, `findSchedulesByTeamAndMonth`, `findSchedulesByTeamAndWeek`, `findSchedulesByTeamAndDate`, `updateSchedule`, `deleteSchedule`, `removeAssigneeFromAllSchedules`
- **낙관적 잠금** (OQ-06): `updateMany WHERE id=? AND version=?` → 영향 행 0이면 409 `VERSION_CONFLICT`

**완료 조건**
- [ ] 동시 업데이트 → 409 `VERSION_CONFLICT` (통합 테스트)
- [ ] `findSchedulesByTeamAndDate` — 해당 날짜에 걸치는 일정만 반환 (경계값)
- [ ] `removeAssigneeFromAllSchedules` — BR-75 담당자 자동 제거 확인

**의존성**: BE-007 | **예상 소요시간**: 3h

---

### BE-018 · Schedule Service

**상세 작업**
- **일정 생성** (UC-05, BR-20, BR-70, BR-75): TEAM_LEADER 재검증, `end_at >= start_at` 검증, 담당자 소속 검증, 트랜잭션, 감사 로그
- **일정 수정** (UC-06): IDOR 방어(리소스 team_id → 소속 → 역할), 낙관적 잠금, 감사 로그
- **일정 삭제** (UC-06): IDOR 방어, 감사 로그
- **캘린더 조회** (UC-04, BR-40): `getDailyData` — schedules + messages 단일 응답

**완료 조건**
- [ ] 팀원 일정 생성 → 403 (BR-20)
- [ ] `end_at < start_at` → 400 `INVALID_TIME_RANGE` (BR-70)
- [ ] `end_at == start_at` → 201 성공 (BR-70 경계값)
- [ ] 비소속 담당자 → 400 `INVALID_ASSIGNEE` (BR-75)
- [ ] 낙관적 잠금 충돌 → 409 `VERSION_CONFLICT` (OQ-06)
- [ ] `getDailyData` — `{ schedules[], messages[] }` 포함 (BR-40)

**의존성**: BE-017, BE-015 | **예상 소요시간**: 5h

---

### BE-019 · Schedule Controller, DTO, Router

**상세 작업**
`src/domains/schedule/schedule.dto.ts`: Zod 스키마 (Create/Update/MonthQuery/DailyQuery)

엔드포인트:
```
GET    /teams/:teamId/schedules?year=&month=        — 월간 조회
GET    /teams/:teamId/schedules?weekStart=&weekEnd=  — 주간 조회
POST   /teams/:teamId/schedules                      — 일정 생성 (팀장만)
PATCH  /teams/:teamId/schedules/:scheduleId          — 일정 수정 (팀장만)
DELETE /teams/:teamId/schedules/:scheduleId          — 일정 삭제 (팀장만)
```
- `UpdateSchedule` body에 `version` 필드 필수 (낙관적 잠금)
- 일일 뷰는 클라이언트에서 schedules + chats 요청을 조합하여 구성 (별도 daily 엔드포인트 없음, BR-40)

**완료 조건**
- [ ] `PATCH` — `version` 미포함 → 400
- [ ] 타 팀 `scheduleId` 수정 시도 → 403 (IDOR)

**의존성**: BE-018, BE-009, BE-010 | **예상 소요시간**: 3h

---

### Phase 5 — Chat 도메인

---

### BE-020 · Chat Repository (Long Polling 포함)

**상세 작업**
- `findOrCreateChat`, `createMessage`, `findMessagesByChatId`, `findMessagesByTeamAndDate`, `findMessageById`, `findMessagesAfter`
- `pollNewMessages(chatId, afterMessageId, timeoutMs)`:
  - 즉시 조회 → 있으면 즉시 반환
  - 없으면 `setInterval(1초)` 폴링 + `setTimeout(20초)` 타임아웃
  - `res.on('close')` 핸들러로 interval/timeout 정리 (메모리 누수 방지)

**완료 조건**
- [ ] `findOrCreateChat` — 동일 (teamId, chatDate) 2회 호출 시 동일 Chat 반환
- [ ] `pollNewMessages` — 20초 내 새 메시지 없으면 빈 배열 (통합 테스트)
- [ ] 새 메시지 삽입 즉시 반환 (통합 테스트)
- [ ] 클라이언트 연결 종료 시 polling 루프 정리 확인

**의존성**: BE-007 | **예상 소요시간**: 4h

---

### BE-021 · Chat Service

**상세 작업**
- **메시지 전송** (UC-07, BR-30, BR-35, BR-85, BR-90): `content.trim().length === 0` → 400, 2KB 제한, `prisma.$transaction` (findOrCreateChat + Message 삽입), 트랜잭션 후 `message.created` 이벤트 발행
- **메시지 조회** (BR-50): (팀 + 날짜) 단위 메시지만 반환
- **Long Polling** (PRD 8.1): `pollMessages` — 최대 20초, 빈 배열 포함 응답, `res.on('close')` 연동

**완료 조건**
- [ ] 빈 문자열 → 400 `EMPTY_MESSAGE_CONTENT` (BR-85)
- [ ] 공백만 있는 메시지 → 400 (BR-85)
- [ ] 전송 성공 → 201 + `{ messageId, content, createdAt }` (BR-30)
- [ ] `message.created` 이벤트 발행 후 팀장 알림 생성 확인 (BR-90)
- [ ] Long Polling 20초 후 빈 배열 반환 (PRD 8.1)

**의존성**: BE-020, BE-006 | **예상 소요시간**: 4h

---

### BE-022 · Chat Controller, DTO, Router

**상세 작업**
엔드포인트:
```
GET    /teams/:teamId/chats/:date/messages            — 전체 조회 (BR-50)
GET    /teams/:teamId/chats/:date/messages?after={id} — Long Polling (PRD 8.1)
POST   /teams/:teamId/chats/:date/messages            — 메시지 전송
```
- `PUT/DELETE /messages/:id` 엔드포인트 등록 없음 (BR-35)
- `chatMessageRateLimit` 미들웨어 적용

**완료 조건**
- [ ] Long Polling — `after` 파라미터 있을 때 이후 메시지만 반환
- [ ] Long Polling — 20초 타임아웃 후 빈 배열 200 반환
- [ ] 채팅 메시지 분당 61건 → 429

**의존성**: BE-021, BE-009 | **예상 소요시간**: 3h

---

### Phase 6 — Notification 도메인

---

### BE-023 · Notification Repository

**상세 작업**
`src/domains/notification/notification.repository.ts`:
- `createNotification`, `createBulkNotifications`, `findNotificationsByUser`, `findNotificationById`, `markAsRead`, `countUnread`

**완료 조건**
- [ ] `createBulkNotifications` — 팀장 복수 시 일괄 삽입 (BR-90)
- [ ] `markAsRead` — 본인 알림만 처리 (userId 검증)
- [ ] `countUnread` — `idx_notifications_user_read` 인덱스 활용

**의존성**: BE-007 | **예상 소요시간**: 2h

---

### BE-024 · Notification Service + 이벤트 핸들러

**상세 작업**
`src/domains/notification/notification.service.ts`:
- `getMyNotifications`, `getUnreadCount`
- `markAsRead`: 타인 알림 → ForbiddenError, 이미 읽음 → idempotent

이벤트 핸들러 등록 (BR-90):
- `message.created` → 팀의 모든 TEAM_LEADER에게 `MESSAGE_RECEIVED` 알림 (작성자가 팀장이면 제외)
- `invitation.sent` → 수신자에게 `INVITATION_RECEIVED`
- `invitation.responded` → 발신 팀장에게 `INVITATION_RESPONDED`

**완료 조건**
- [ ] 팀원 메시지 → 팀장 `MESSAGE_RECEIVED` 알림 생성 (BR-90)
- [ ] 팀장 메시지 → 알림 생성 안 됨
- [ ] 초대 발송 → `INVITATION_RECEIVED` 알림 생성
- [ ] 타인 알림 읽음 처리 → 403

**의존성**: BE-023, BE-006 | **예상 소요시간**: 4h

---

### BE-025 · Notification Controller, DTO, Router

**상세 작업**
엔드포인트:
```
GET    /notifications                          — 내 알림 목록
GET    /notifications/unread-count             — 미읽음 수
PATCH  /notifications/:id/read                — 읽음 처리
```

**완료 조건**
- [ ] `GET /notifications` — 본인 알림만 반환
- [ ] `PATCH /notifications/:id/read` — 타인 알림 → 403
- [ ] 미읽음 수 정확히 반환

**의존성**: BE-024, BE-008 | **예상 소요시간**: 2h

---

### Phase 7 — 감사 로그

---

### BE-026 · 감사 로그 Repository 및 서비스

**상세 작업**
`src/domains/audit/audit.repository.ts`:
- `log(data)` — fire-and-forget, 절대 throw 안 함, INSERT만 수행
- 민감정보 Redact: `password`, `jwt`, `refresh_token`, `session_id` → `[REDACTED]`

기록 대상 이벤트 (아키텍처 원칙 5.3):
- `login_success`, `login_failure`, `logout`
- `forbidden_team_access` (BR-60)
- `team_leader_assigned`, `schedule_created/updated/deleted`, `invitation_*`

**완료 조건**
- [ ] `audit_logs` UPDATE/DELETE 시도 시 DB 권한 오류
- [ ] 로그인 성공/실패 시 `audit_logs` 레코드 삽입
- [ ] `password` 필드 `[REDACTED]` 저장 확인

**의존성**: BE-007 | **예상 소요시간**: 3h

---

### Phase 8 — 라우터 통합

---

### BE-027 · 전체 라우터 통합 및 미들웨어 체인 완성

**상세 작업**
`src/routes/index.ts`:
- 미들웨어 순서: `globalRateLimit → helmet → cors → pinoHttp`
- 라우터 등록: `/auth`, `/teams` (authenticate), `/invitations` (authenticate), `/notifications` (authenticate)
- 팀 하위: `teamMemberGuard` → 하위 라우터 → `authorize('TEAM_LEADER')` 필요한 경로

**완료 조건**
- [ ] 전체 API 엔드포인트 등록 확인
- [ ] `GET /health` 인증 없이 접근 가능
- [ ] 미등록 경로 → 404 일관된 형식

**의존성**: BE-013, BE-016, BE-019, BE-022, BE-025 | **예상 소요시간**: 2h

---

### Phase 9 — 테스트

---

### BE-028 · 단위 테스트 — Auth, Team Service

**상세 작업** (Jest + ts-jest, `tests/unit/`)
- Auth: BR-15, 로그인 이메일 비노출, bcrypt 해시 확인, 토큰 폐기 로직
- Team: BR-25 중복 초대, 만료 초대 수락, 이벤트 emit 확인 (Mock EventBus)
- Repository는 Jest Mock으로 대체

**완료 조건**
- [ ] BR-15, BR-25 에러 케이스 단위 테스트 커버
- [ ] 이벤트 발행 Mock 인수 검증
- [ ] Service 레이어 커버리지 80% 이상
- [ ] `npm run test:unit` 30초 내 완료

**의존성**: BE-012, BE-015 | **예상 소요시간**: 4h

---

### BE-029 · 단위 테스트 — Schedule, Chat, Notification Service

**상세 작업**
- Schedule: BR-20, BR-70(경계값 `==` 허용), BR-75, OQ-06 낙관적 잠금, IDOR
- Chat: BR-85(빈 문자열/공백), BR-90 이벤트 emit, 트랜잭션
- Notification: 타인 알림 ForbiddenError, idempotent, 이벤트 핸들러 양방향 테스트

**완료 조건**
- [ ] BR-70 경계값(`==`) 테스트 통과
- [ ] BR-85 빈 문자열/공백 테스트 통과
- [ ] `message.created` 핸들러 발행/구독 단위 테스트 (아키텍처 원칙 4.2)

**의존성**: BE-018, BE-021, BE-024 | **예상 소요시간**: 5h

---

### BE-030 · 통합 테스트 — Repository 계층 (Testcontainers)

**상세 작업** (Jest + Testcontainers, `tests/integration/`)
- BeforeAll: PostgreSQL 컨테이너 기동 + `prisma migrate deploy`
- AfterEach: 데이터 초기화 (truncate 또는 트랜잭션 롤백)
- Auth: 중복 이메일 P2002, Refresh Token 폐기
- Team: 트랜잭션 원자성, 롤백 확인
- Schedule: 낙관적 잠금 동시성 (두 클라이언트 동일 version 수정)
- Chat: `findOrCreateChat` idempotent, `pollNewMessages` 타임아웃
- Notification: `createBulkNotifications`, `countUnread` 인덱스 활용

**완료 조건**
- [ ] Mock DB 없이 전체 Repository 통합 테스트 통과
- [ ] 낙관적 잠금 동시성 테스트 통과 (OQ-06)
- [ ] `npm run test:integration` CI 환경 통과

**의존성**: BE-011, BE-014, BE-017, BE-020, BE-023 | **예상 소요시간**: 6h

---

### BE-031 · 통합 테스트 — API 레이어 (권한·팀 격리)

**상세 작업** (Jest + Supertest + Testcontainers)

**필수 테스트 게이트** (아키텍처 원칙 4.2):

팀 격리 (BR-60):
- 팀A 사용자 → 팀B 일정/채팅 → 403
- URL `teamId` 변조 → 403 (IDOR)

감사 불변성:
- `PUT/DELETE /messages/:id` → 404 또는 405 (BR-35)
- 메시지 전송 → 팀장 알림 생성 (BR-90)

역할 × 행위 매트릭스:
| 행위 | 팀장 | 팀원 |
|------|------|------|
| `POST /schedules` | 201 | 403 |
| `PATCH /schedules/:id` | 200 | 403 |
| `DELETE /schedules/:id` | 204 | 403 |
| `POST /invitations` | 201 | 403 |
| `POST /chats/:date/messages` | 201 | 201 |

**완료 조건**
- [ ] 팀 격리 경로 전체 커버 (BR-60)
- [ ] 역할 × 행위 매트릭스 전체 케이스 통과
- [ ] BR-35, BR-70(경계값), BR-80, BR-90 통합 테스트 통과

**의존성**: BE-027, BE-030 | **예상 소요시간**: 6h

---

### BE-032 · E2E 테스트 — 핵심 사용자 흐름

**상세 작업** (Jest + Supertest, `tests/e2e/`)
- SC-01: 회원가입 → 로그인 → 팀 생성 → 팀원 초대 → 수락 → 역할 확인
- SC-02: 일정 생성 → 팀원 채팅 → 팀장 알림 → 일정 수정
- SC-03: Long Polling 시작 → 메시지 전송 → 즉시 응답 수신 (5초 내)
- SC-04: 미인증 API → 401, 타 팀 접근 → 403, 만료 토큰 → 401, Refresh → 새 토큰

**완료 조건**
- [ ] SC-01~SC-04 전체 통과
- [ ] 실제 PostgreSQL 컨테이너 사용
- [ ] Long Polling SC-03이 5초 내 응답 수신

**의존성**: BE-031 | **예상 소요시간**: 6h

---

### Phase 10 — 운영 준비

---

### BE-033 · 헬스체크 및 모니터링, Graceful Shutdown

**상세 작업**
`GET /health` 상세 응답: `{ status, timestamp, version, db: { status, latencyMs }, longPollingConnections }`

Long Polling 활성 연결 수 전역 카운터 추적 (`res.on('close')` 감소)

Graceful Shutdown (`src/server.ts`):
- `server.close()` → 신규 연결 거부
- 활성 Long Polling 연결에 빈 응답 즉시 전송
- `prisma.$disconnect()`

**완료 조건**
- [ ] DB 연결 끊긴 경우 → 503
- [ ] `SIGTERM` 수신 시 Long Polling 연결 drain 후 종료
- [ ] `longPollingConnections` 메트릭 로그에 포함

**의존성**: BE-004, BE-022 | **예상 소요시간**: 3h

---

### BE-034 · 시드 데이터 및 개발 환경 구성

> DB-014 작업 결과물 통합 및 백엔드 개발 환경 최종 확인

**완료 조건**
- [ ] `npx prisma db seed` 성공
- [ ] 시드 후 `GET /teams` → `개발팀` 반환
- [ ] Long Polling 즉시 테스트 가능

**의존성**: BE-007 | **예상 소요시간**: 1h

---

## Part 3. 프론트엔드 (FE)

### Phase 0 — 프로젝트 초기 설정

---

### FE-001 · Vite + React 18 + TypeScript 초기화

**상세 작업**
- `vite` + `react-ts` 템플릿으로 프로젝트 생성
- 핵심 패키지 설치: `react@18`, `react-dom@18`, `react-router-dom@6`, `@tanstack/react-query@5`, `zustand`, `axios`
- 테스트: `vitest`, `@testing-library/react`, `@testing-library/user-event`, `@testing-library/jest-dom`, `jsdom`
- `vite.config.ts`: path alias (`@/` → `src/`), 개발서버 프록시 (`/api` → `http://localhost:3000`)
- `vitest.config.ts`: jsdom 환경, jest-dom setup
- `tsconfig.json`: `strict: true`, `paths: { "@/*": ["./src/*"] }`
- `frontend/.env.example`: `VITE_API_BASE_URL` 키

**완료 조건**
- [ ] `npm run dev` 개발 서버 정상 기동
- [ ] `npm run build` TypeScript 오류 없이 성공
- [ ] `npm run test` Vitest 기본 실행 성공
- [ ] path alias `@/` import 가능

**의존성**: 없음 | **예상 소요시간**: 2h

---

### FE-002 · 디렉토리 구조 및 공통 타입 정의

**상세 작업**
- 전체 디렉토리 골격 생성 (`pages/`, `features/`, `components/`, `api/`, `store/`, `hooks/`, `constants/`, `utils/`)
- `src/types/api.types.ts`: `ApiResponse<T>`, `ApiError` 타입
- `src/types/domain.types.ts`: `User`, `Team`, `Schedule`, `Message`, `Notification`, `Invitation`, `UserRole`, `ScheduleStatus` 등
- `src/constants/routes.ts`: `ROUTES` 경로 상수
- `src/constants/query-keys.ts`: `QUERY_KEYS` TanStack Query 키 상수

**완료 조건**
- [ ] 전체 디렉토리 구조 생성
- [ ] 공통 타입이 ERD 엔티티 전체 커버
- [ ] `QUERY_KEYS` 상수로 쿼리 키 중앙 관리
- [ ] TypeScript strict 오류 없음

**의존성**: FE-001 | **예상 소요시간**: 1.5h

---

### FE-003 · Axios HTTP 클라이언트 (인터셉터 + 토큰 갱신)

**상세 작업**
`src/api/client.ts`:
- `axios.create()`: `baseURL: VITE_API_BASE_URL`, `withCredentials: true`
- **요청 인터셉터**: Zustand `auth.store`에서 `accessToken` 읽어 `Authorization: Bearer` 헤더 주입
- **응답 인터셉터 (401 처리)**: `/auth/refresh` 호출 → 성공 시 원 요청 재시도 (중복 갱신 방지: `isRefreshing` 플래그 + `failedQueue`), 실패 시 로그아웃 + redirect
- Access Token: Zustand 메모리 저장만 (`localStorage` 저장 금지)
- `src/utils/error.utils.ts`: `isAxiosError`, `getApiErrorMessage`

**완료 조건**
- [ ] Access Token 자동 헤더 주입
- [ ] 401 시 자동 갱신 후 원 요청 재시도
- [ ] 동시 401 요청 시 중복 갱신 방지 (큐 방식)
- [ ] 갱신 실패 시 로그인 redirect
- [ ] `localStorage` 토큰 저장 코드 없음

**의존성**: FE-001, FE-002 | **예상 소요시간**: 3h

---

### FE-004 · Zustand 스토어 (auth + team)

**상세 작업**
`src/store/auth.store.ts`:
- `accessToken: string | null` (메모리 저장)
- `user: { id, name, email } | null`
- `isAuthenticated: boolean`
- `setAuth`, `clearAuth`, `updateAccessToken`
- devtools middleware 적용

`src/store/team.store.ts`:
- `selectedTeamId: string | null`
- `selectedTeamRole: 'TEAM_LEADER' | 'TEAM_MEMBER' | null`
- `setSelectedTeam`, `clearSelectedTeam`

**완료 조건**
- [ ] `auth.store`: 메모리 저장, 3개 액션 동작
- [ ] `team.store`: teamId/role 저장 및 초기화
- [ ] devtools Redux DevTools 연동 확인

**의존성**: FE-001, FE-002 | **예상 소요시간**: 1.5h

---

### FE-005 · TanStack Query 및 React Router 설정

**상세 작업**
`src/main.tsx`: `QueryClientProvider`, `BrowserRouter`, `ReactQueryDevtools` (개발만)

`src/App.tsx` 라우트 구성:
```
/auth/login          → AuthPage
/auth/signup         → AuthPage
/teams               → TeamListPage (ProtectedRoute)
/teams/:teamId/calendar → CalendarPage (ProtectedRoute)
/teams/:teamId/daily    → DailyViewPage (ProtectedRoute)
*                    → redirect to /auth/login
```
- `ProtectedRoute`: `isAuthenticated` 확인, false → `/auth/login`
- `queryClient` 기본 옵션: `staleTime: 60000`, `retry: 1`, `refetchOnWindowFocus: false`

**완료 조건**
- [ ] 모든 라우트 렌더링 정상
- [ ] 비인증 상태 보호 라우트 접근 → redirect
- [ ] `QueryClientProvider`가 앱 전체 감쌈

**의존성**: FE-003, FE-004 | **예상 소요시간**: 2h

---

### Phase 1 — Shared 컴포넌트 라이브러리

---

### FE-006 · 공통 UI 컴포넌트

**상세 작업**
`src/components/` 하위:
- `Button.tsx`: `variant(primary|secondary|danger|ghost)`, `size(sm|md|lg)`, `isLoading` prop, 로딩 시 비활성화
- `Modal.tsx`: `createPortal`, ESC/배경 클릭 닫기, `aria-modal`, `aria-labelledby`
- `Badge.tsx`: `count`, `max(기본99)`, 초과 시 "99+" 표시
- `LoadingSpinner.tsx`: CSS animation
- `ErrorMessage.tsx`: `message: string | null`
- `ConfirmDialog.tsx`: Modal 기반, 확인/취소 콜백
- `FormField.tsx`: label, error, required, htmlFor
- `EmptyState.tsx`: 빈 목록 표시

**완료 조건**
- [ ] 모든 컴포넌트 TypeScript strict 오류 없음
- [ ] Modal 접근성 속성 적용
- [ ] `Button` isLoading 시 중복 클릭 방지
- [ ] 각 컴포넌트 단위 테스트
- [ ] `dangerouslySetInnerHTML` 사용 없음

**의존성**: FE-001, FE-002 | **예상 소요시간**: 4h

---

### FE-007 · 공통 레이아웃 및 네비게이션

**상세 작업**
- `src/components/Layout.tsx`: GNB + 콘텐츠 영역
- `src/components/GlobalNavBar.tsx`: 로고, 팀 선택 드롭다운, 알림 아이콘 + Badge, 로그아웃 버튼
- `src/hooks/use-debounce.ts`: `useDebounce<T>(value, delay)` 훅

**완료 조건**
- [ ] GNB 알림 미읽음 배지 표시
- [ ] 팀 선택 시 `team.store` 업데이트
- [ ] 로그아웃 → `auth.store.clearAuth()` + redirect

**의존성**: FE-005, FE-006 | **예상 소요시간**: 2.5h

---

### FE-008 · 날짜 유틸리티

**상세 작업**
`src/utils/date.utils.ts`:
- `formatLocalDateTime`, `formatLocalDate`, `formatLocalTime` (UTC → 로컬 변환)
- `getMonthDays`, `getWeekDays`, `formatDateParam`, `isSameDay`, `isToday`
- 서버 → ISO 8601 UTC 수신 → 로컬 타임존 표시 (아키텍처 원칙 3.4)

**완료 조건**
- [ ] UTC → 로컬 변환 정확
- [ ] `formatDateParam` → `YYYY-MM-DD` 형식
- [ ] 월 경계값 테스트 통과
- [ ] 단위 테스트 커버리지 100%

**의존성**: FE-001 | **예상 소요시간**: 2h

---

### Phase 2 — Auth Feature

---

### FE-009 · Auth API 클라이언트

**상세 작업**
`src/api/auth.api.ts`: `register`, `login`, `logout`, `refreshToken` 함수

**완료 조건**
- [ ] 4개 API 함수 TypeScript 타입 명시
- [ ] `refreshToken`이 인터셉터와 연동

**의존성**: FE-003 | **예상 소요시간**: 1h

---

### FE-010 · Auth Queries (TanStack Query mutations)

**상세 작업**
`src/features/auth/queries/auth.queries.ts`:
- `useSignupMutation`: 성공 → `/auth/login` navigate
- `useLoginMutation`: 성공 → `auth.store.setAuth()` + `/teams` navigate
- `useLogoutMutation`: 성공 → `clearAuth()` + 전체 캐시 clear + `/auth/login` navigate

**완료 조건**
- [ ] 로그인 성공 → `auth.store` 업데이트 + 이동
- [ ] 로그아웃 성공 → store 초기화 + 캐시 clear
- [ ] API 오류 메시지 화면 표시

**의존성**: FE-004, FE-009 | **예상 소요시간**: 1.5h

---

### FE-011 · Auth Hook (`use-auth`)

**완료 조건**
- [ ] 반환 객체로 모든 인증 상태·액션 접근 가능

**의존성**: FE-010 | **예상 소요시간**: 1h

---

### FE-012 · AuthPage + LoginForm + SignupForm 컴포넌트

**상세 작업**
- `LoginForm`: 이메일/비밀번호, 클라이언트 유효성 검사, 오류 메시지 표시
- `SignupForm`: 이름/이메일/비밀번호/확인, 최소 8자+영문+숫자+특수문자, 비밀번호 확인 일치
- 빈 필드 → 서버 요청 없이 클라이언트 차단

**완료 조건**
- [ ] 빈 필드 제출 시 클라이언트 오류 (서버 요청 없음)
- [ ] 비밀번호 형식 불일치 → 클라이언트 오류
- [ ] 409 중복 이메일 → 서버 오류 메시지 표시
- [ ] 로그인 성공 → `/teams` 이동
- [ ] 회원가입 성공 → `/auth/login` 이동

**의존성**: FE-006, FE-011 | **예상 소요시간**: 3h

---

### Phase 3 — Team Feature

---

### FE-013 · Team API 클라이언트

**상세 작업**
`src/api/team.api.ts`: `getTeams`, `createTeam`, `getTeamMembers`, `sendInvitation`, `getReceivedInvitations`, `acceptInvitation`, `declineInvitation`

**완료 조건**
- [ ] 모든 API 함수 TypeScript 타입 명시

**의존성**: FE-003 | **예상 소요시간**: 1.5h

---

### FE-014 · Team Queries

**상세 작업**
`src/features/team/queries/team.queries.ts`: `useTeamsQuery`, `useCreateTeamMutation`, `useTeamMembersQuery`, `useSendInvitationMutation`, `useReceivedInvitationsQuery`, `useAcceptInvitationMutation`, `useDeclineInvitationMutation`

**완료 조건**
- [ ] 팀 생성 성공 후 목록 자동 갱신 (invalidate)
- [ ] 초대 수락 후 팀 목록 + 초대 목록 갱신

**의존성**: FE-002, FE-013 | **예상 소요시간**: 2h

---

### FE-015 · Team Hook (`use-team`)

**완료 조건**
- [ ] `isLeader` 값이 `team.store` 기반으로 정확히 반환

**의존성**: FE-004, FE-014 | **예상 소요시간**: 1h

---

### FE-016 · TeamListPage + 팀 관련 컴포넌트

**상세 작업**
- `TeamListPage.tsx`: 팀 목록, 팀 생성 버튼, 수신 초대 목록
- `TeamCard.tsx`: 팀명, 역할 배지, 클릭 → 캘린더 이동, 팀장 → 초대 버튼
- `CreateTeamModal.tsx`: 팀 이름 입력, 빈 이름 → 클라이언트 차단
- `InviteModal.tsx`: 이메일 입력, 오류 처리 (409/404/400/403)
- `InvitationCard.tsx`: 초대 정보, 수락/거절 버튼

**완료 조건**
- [ ] 팀 생성 → 목록 자동 갱신
- [ ] 초대 발송 성공 → 모달 닫힘
- [ ] 팀원은 "팀원 초대" 버튼 미표시

**의존성**: FE-006, FE-007, FE-015 | **예상 소요시간**: 4h

---

### Phase 4 — Calendar Feature

---

### FE-017 · Schedule API 클라이언트

**상세 작업**
`src/api/schedule.api.ts`: `getMonthlySchedules`, `getWeeklySchedules`, `getDailyView`, `createSchedule`, `updateSchedule` (version 필수), `deleteSchedule`

**완료 조건**
- [ ] `updateSchedule`에 `version` 필드 포함
- [ ] `getDailyView` → schedules + messages 복합 응답 처리

**의존성**: FE-003 | **예상 소요시간**: 1.5h

---

### FE-018 · Schedule Queries

**상세 작업**
`src/features/calendar/queries/schedule.queries.ts`:
- `useMonthlySchedulesQuery`, `useWeeklySchedulesQuery`, `useDailyViewQuery`
- `useCreateScheduleMutation`, `useUpdateScheduleMutation` (409 처리), `useDeleteScheduleMutation`

**완료 조건**
- [ ] 일정 CRUD 후 캘린더 자동 갱신
- [ ] 409 충돌 사용자 친화적 메시지

**의존성**: FE-002, FE-017 | **예상 소요시간**: 2h

---

### FE-019 · Calendar Hook (`use-calendar`)

**상세 작업**
- 현재 날짜, 뷰 모드(월간/주간) 상태 관리
- 날짜별 일정 그룹핑 (`schedulesByDate: { 'YYYY-MM-DD': Schedule[] }`)
- 이전/다음 월, 이전/다음 주 네비게이션

**완료 조건**
- [ ] 월/주 전환 시 적절한 데이터 fetch
- [ ] `schedulesByDate` 그룹핑 정확

**의존성**: FE-008, FE-018 | **예상 소요시간**: 2h

---

### FE-020 · MonthlyCalendar 컴포넌트 (SC-08)

**상세 작업**
- 7열 그리드, 요일 헤더
- 날짜 셀: 날짜 숫자 + 일정 제목 (최대 3개, 초과 "+N")
- 오늘 강조, 이전/다음 달 dimmed
- 날짜 클릭 → `DailyViewPage` 이동

**완료 조건**
- [ ] 7열 그리드 정확 렌더링
- [ ] 일정 제목 날짜 셀 표시
- [ ] 이전/다음 월 이동 시 데이터 갱신

**의존성**: FE-006, FE-008, FE-019 | **예상 소요시간**: 3h

---

### FE-021 · WeeklyCalendar 컴포넌트 (SC-09)

**상세 작업**
- 7열 × 시간대 행 그리드
- 일정 블록: 시작/종료 시간 기준 위치 및 높이
- 이전/다음 주 이동

**완료 조건**
- [ ] 시간대 기준 일정 블록 배치 정확
- [ ] 이전/다음 주 이동 시 데이터 갱신

**의존성**: FE-006, FE-008, FE-019 | **예상 소요시간**: 4h

---

### FE-022 · ScheduleForm + CalendarPage (SC-11, SC-12, SC-13)

**상세 작업**
- `ScheduleForm.tsx`: 제목/시작일시/종료일시/담당자(팀원만), 클라이언트 검증 (BR-70, BR-75), 낙관적 잠금 409 처리
- `CalendarPage.tsx`: 월간/주간 탭, 팀장 전용 "일정 추가" 버튼
- `ScheduleDetailModal.tsx`: 일정 상세, 팀장 전용 수정/삭제 버튼, `ConfirmDialog`

**완료 조건**
- [ ] `end_at < start_at` → 클라이언트 오류 (BR-70)
- [ ] `end_at == start_at` → 허용 (BR-70 경계값)
- [ ] 팀원 생성/수정/삭제 버튼 미표시 (BR-20)
- [ ] 삭제 확인 다이얼로그 표시
- [ ] 409 에러 처리 (OQ-06)

**의존성**: FE-006, FE-015, FE-018, FE-020, FE-021 | **예상 소요시간**: 5h

---

### Phase 5 — Chat Feature

---

### FE-023 · Chat API 클라이언트

**상세 작업**
`src/api/chat.api.ts`:
- `sendMessage(teamId, date, body)`
- `pollMessages(teamId, date, lastMessageId, signal)`: timeout 25초, AbortSignal 지원

**완료 조건**
- [ ] `pollMessages` AbortSignal 지원
- [ ] Long Polling timeout 25초

**의존성**: FE-003 | **예상 소요시간**: 1.5h

---

### FE-024 · Long Polling 훅 (`use-long-polling`)

**상세 작업**
`src/features/chat/hooks/use-long-polling.ts`:
- `lastMessageId` ref 유지
- 응답 즉시 재요청 (빈 배열 포함)
- 오류 시 3초 대기 후 재시도, **최대 3회** (PRD 8.1)
- 언마운트 / `enabled: false` → `AbortController.abort()`
- `date`/`teamId` 변경 시 `lastMessageId` 초기화 + 재시작

**완료 조건**
- [ ] 새 메시지 수신 시 `onNewMessages` 즉시 호출
- [ ] 빈 응답 즉시 재요청
- [ ] 언마운트 시 요청 취소 (AbortSignal)
- [ ] 네트워크 오류 3회 초과 → `error` state + 폴링 중지
- [ ] date 변경 시 `lastMessageId` 초기화

**의존성**: FE-023 | **예상 소요시간**: 4h

---

### FE-025 · Chat Queries

**상세 작업**
`src/features/chat/queries/chat.queries.ts`:
- `useSendMessageMutation`: 낙관적 업데이트 (전송 즉시 표시), 실패 시 롤백

**완료 조건**
- [ ] 낙관적 업데이트 적용
- [ ] 전송 실패 시 롤백

**의존성**: FE-002, FE-023 | **예상 소요시간**: 1.5h

---

### FE-026 · Chat Hook (`use-chat`)

**상세 작업**
- 초기 메시지: `useDailyViewQuery` 결과
- Long Polling 수신 → 로컬 state append
- `handleSendMessage`: 빈 문자열/공백 차단 (BR-85), 낙관적 업데이트
- 새 메시지 시 자동 스크롤 (`messagesEndRef`)

**완료 조건**
- [ ] 빈 메시지/공백 전송 차단 (BR-85)
- [ ] 새 메시지 수신 시 자동 스크롤
- [ ] 낙관적 업데이트 → 실패 시 롤백

**의존성**: FE-024, FE-025 | **예상 소요시간**: 2.5h

---

### FE-027 · Chat 컴포넌트 + DailyViewPage (SC-10, SC-14)

**상세 작업**
- `DailyViewPage.tsx`: `useDailyViewQuery` — 좌측 일정 + 우측 채팅 (BR-40)
- `ChatPanel.tsx`: `use-chat` 연결
- `MessageList.tsx`: 작성자/시간/내용, 본인/타인 정렬, `dangerouslySetInnerHTML` 금지
- `MessageInput.tsx`: Enter 전송, Shift+Enter 줄바꿈, 빈 문자열 비활성화 (BR-85)
- `ScheduleList.tsx`: 시간순 정렬, 팀장 전용 수정/삭제

**완료 조건**
- [ ] 좌측 일정 + 우측 채팅 동시 표시 (BR-40)
- [ ] 메시지 전송 후 즉시 표시 (낙관적)
- [ ] 새 메시지 수신 시 자동 스크롤
- [ ] 빈 메시지 전송 버튼 비활성화 (BR-85)
- [ ] XSS 방어 (`dangerouslySetInnerHTML` 미사용)

**의존성**: FE-006, FE-018, FE-026 | **예상 소요시간**: 5h

---

### Phase 6 — Notification Feature

---

### FE-028 · Notification API 클라이언트

`src/api/notification.api.ts`: `getNotifications`, `getUnreadCount`, `markAsRead`

**의존성**: FE-003 | **예상 소요시간**: 0.5h

---

### FE-029 · Notification Queries

`src/features/notification/queries/notification.queries.ts`:
- `useNotificationsQuery`: `refetchInterval: 30000`
- `useUnreadCountQuery`: `refetchInterval: 30000`
- `useMarkAsReadMutation`: 낙관적 업데이트 (즉시 isRead: true)

**완료 조건**
- [ ] 30초 자동 갱신
- [ ] 읽음 처리 낙관적 업데이트 → 배지 즉시 감소

**의존성**: FE-002, FE-028 | **예상 소요시간**: 1.5h

---

### FE-030 · Notification Hook (`use-notification`)

알림 클릭 → 읽음 처리 + 유형별 화면 이동:
- `MESSAGE_RECEIVED` → 해당 날짜 채팅
- `INVITATION_*` → 팀 목록

**의존성**: FE-029 | **예상 소요시간**: 1.5h

---

### FE-031 · Notification 컴포넌트 (SC-15)

- `NotificationBadge.tsx`: 미읽음 0 → 배지 숨김
- `NotificationList.tsx`: 유형 아이콘, 읽음/미읽음 강조, 클릭 처리

**완료 조건**
- [ ] 미읽음 배지 정확 표시
- [ ] 알림 클릭 → 읽음 처리 + 이동
- [ ] 읽음 후 배지 즉시 갱신

**의존성**: FE-006, FE-030 | **예상 소요시간**: 2.5h

---

### Phase 7 — 테스트

---

### FE-032 · Auth Feature 테스트

- `LoginForm.test.tsx`: 빈 필드 차단, 401/409 에러 표시, 로딩 비활성화
- `SignupForm.test.tsx`: 비밀번호 형식/확인, 409 처리
- `use-auth.test.ts`: 로그인 → store 업데이트, 로그아웃 → clearAuth

**완료 조건**
- [ ] BR-15 (409) 테스트 통과
- [ ] 클라이언트 유효성 검사 테스트 통과
- [ ] MSW 또는 vi.mock으로 API 모킹

**의존성**: FE-011, FE-012 | **예상 소요시간**: 3h

---

### FE-033 · Calendar Feature 테스트

- `ScheduleForm.test.tsx`: BR-70(경계값), BR-75, BR-20, 낙관적 잠금 409
- `MonthlyCalendar.test.tsx`: 일정 표시, 빈 월, 월 이동
- `use-calendar.test.ts`: 날짜별 그룹핑

**완료 조건**
- [ ] BR-70 경계값(`==`) 통과
- [ ] BR-20 팀원 403 시나리오 통과

**의존성**: FE-019, FE-022 | **예상 소요시간**: 3h

---

### FE-034 · Chat Feature 테스트 (Long Polling 포함)

- `use-long-polling.test.ts`: 새 메시지 콜백, 빈 응답 재요청, AbortSignal 취소, 3회 초과 → 중지, date 변경 시 초기화
- `MessageInput.test.tsx`: 빈 문자열/공백 비활성화 (BR-85), Enter/Shift+Enter
- `use-chat.test.ts`: 낙관적 업데이트, 실패 롤백

**완료 조건**
- [ ] BR-85 테스트 통과
- [ ] BR-35 (수정/삭제 UI 없음) 확인
- [ ] Long Polling 재시도/AbortSignal 테스트 통과

**의존성**: FE-024, FE-026, FE-027 | **예상 소요시간**: 4h

---

### FE-035 · Notification Feature 테스트

- `NotificationBadge.test.tsx`: 미읽음 0 숨김, 100 → "99+"
- `use-notification.test.ts`: 읽음 처리 낙관적 업데이트, 유형별 navigate

**의존성**: FE-030, FE-031 | **예상 소요시간**: 2h

---

### FE-036 · 팀 격리 및 권한 테스트

- BR-60: 팀A → 팀B URL 접근 시 403 처리 UI
- BR-20: 팀원 계정 → 생성/수정/삭제 버튼 미표시, 팀장 → 표시

**완료 조건**
- [ ] BR-60 시나리오 통과
- [ ] BR-20 역할별 UI 노출 테스트 통과

**의존성**: FE-016, FE-022, FE-031 | **예상 소요시간**: 2.5h

---

### Phase 8 — 통합 마무리

---

### FE-037 · Error Boundary 및 전역 에러 처리

- `ErrorBoundary.tsx`: 런타임 오류 캐치 → 에러 UI
- `ApiErrorFallback.tsx`: 401/403/404/500/429 별 메시지
- `NetworkErrorBanner.tsx`: Long Polling 오류 시 상단 배너

**완료 조건**
- [ ] 런타임 오류 시 크래시 없이 에러 UI
- [ ] HTTP 에러별 적절한 메시지

**의존성**: FE-005, FE-006 | **예상 소요시간**: 2h

---

### FE-038 · 접근성 및 UX 개선

- 모든 인터랙티브 요소 `aria-label`/`aria-labelledby`
- Modal focus trap, 폼 에러 `aria-describedby`
- 로딩 `aria-busy`, `aria-live`
- 768px 이상 레이아웃 지원 (PRD — 데스크톱·태블릿 우선)

**완료 조건**
- [ ] Modal focus trap 동작
- [ ] 알림 배지 aria-label 적용
- [ ] 768px 이상 레이아웃 정상

**의존성**: FE-006, FE-007, FE-031 | **예상 소요시간**: 2h

---

### FE-039 · 환경변수 및 빌드 최적화

- `.env.development`, `.env.production`, `.env.example` 구성
- Vite `manualChunks`: 벤더 번들 분리
- `React.lazy` + `Suspense`: 페이지별 동적 import (AuthPage, TeamListPage, CalendarPage, DailyViewPage)
- `npm run build`: `tsc --noEmit && vite build`

**완료 조건**
- [ ] `npm run build` 타입 검사 포함 성공
- [ ] 코드 스플리팅 적용
- [ ] `.env.example`에 실제 값 없음

**의존성**: FE-005 | **예상 소요시간**: 2h

---

## 전체 태스크 요약표

### DB 영역

| ID | 태스크명 | 시간 | 의존성 |
|----|---------|------|--------|
| DB-001 | 데이터베이스 환경 설정 | 1h | — |
| DB-002 | Prisma 초기화 | 0.5h | DB-001 |
| DB-003 | Enum 정의 | 0.5h | DB-002 |
| DB-004 | User + RefreshToken 모델 | 0.5h | DB-003 |
| DB-005 | Team + UserTeamRole 모델 | 0.5h | DB-003, DB-004 |
| DB-006 | Invitation 모델 | 0.5h | DB-003~005 |
| DB-007 | Schedule + ScheduleAssignee 모델 | 0.75h | DB-003~005 |
| DB-008 | Chat + Message 모델 | 0.5h | DB-003~005 |
| DB-009 | Notification 모델 | 0.5h | DB-003, DB-004 |
| DB-010 | AuditLog 모델 | 0.5h | DB-003, DB-004 |
| DB-011 | 인덱스 최적화 | 0.75h | DB-004~010 |
| DB-012 | 유니크 + Check Constraint | 0.5h | DB-005~008 |
| DB-013 | 초기 마이그레이션 실행 | 1h | DB-003~012 |
| DB-014 | 시드 데이터 | 1.5h | DB-013 |
| DB-015 | Prisma Client 싱글톤 | 0.5h | DB-002 |
| DB-016 | 마이그레이션 전략 문서화 | 1h | DB-013 |
| **합계** | | **~11h** | |

### BE 영역

| ID | 태스크명 | Phase | 시간 | 의존성 |
|----|---------|-------|------|--------|
| BE-001 | 프로젝트 초기화 | 0 | 2h | — |
| BE-002 | ESLint + Prettier | 0 | 2h | BE-001 |
| BE-003 | Prisma 스키마 통합 | 0 | 0.5h | BE-001, DB-013 |
| BE-004 | Express 앱 초기화 | 0 | 3h | BE-001, BE-003 |
| BE-005 | 에러 클래스, 로거 | 0 | 3h | BE-001, BE-004 |
| BE-006 | EventBus 인프라 | 0 | 3h | BE-005 |
| BE-007 | Prisma Client 싱글톤 | 0 | 1h | BE-003, BE-004 |
| BE-008 | JWT 인증 미들웨어 + Rate Limit | 1 | 3h | BE-005 |
| BE-009 | TeamMemberGuard | 1 | 3h | BE-007, BE-008 |
| BE-010 | RBAC 인가 미들웨어 | 1 | 2h | BE-009 |
| BE-011 | Auth Repository | 2 | 2h | BE-007 |
| BE-012 | Auth Service | 2 | 4h | BE-011 |
| BE-013 | Auth Controller + Router | 2 | 2h | BE-012, BE-008 |
| BE-014 | Team + Invitation Repository | 3 | 3h | BE-007 |
| BE-015 | Team Service | 3 | 5h | BE-014, BE-006 |
| BE-016 | Team Controller + Router | 3 | 3h | BE-015, BE-009, BE-010 |
| BE-017 | Schedule Repository | 4 | 3h | BE-007 |
| BE-018 | Schedule Service | 4 | 5h | BE-017, BE-015 |
| BE-019 | Schedule Controller + Router | 4 | 3h | BE-018, BE-009, BE-010 |
| BE-020 | Chat Repository (Long Polling) | 5 | 4h | BE-007 |
| BE-021 | Chat Service | 5 | 4h | BE-020, BE-006 |
| BE-022 | Chat Controller + Router | 5 | 3h | BE-021, BE-009 |
| BE-023 | Notification Repository | 6 | 2h | BE-007 |
| BE-024 | Notification Service + 이벤트 핸들러 | 6 | 4h | BE-023, BE-006 |
| BE-025 | Notification Controller + Router | 6 | 2h | BE-024, BE-008 |
| BE-026 | 감사 로그 Repository + 서비스 | 7 | 3h | BE-007 |
| BE-027 | 전체 라우터 통합 | 8 | 2h | BE-013, BE-016, BE-019, BE-022, BE-025 |
| BE-028 | 단위 테스트 — Auth, Team | 9 | 4h | BE-012, BE-015 |
| BE-029 | 단위 테스트 — Schedule, Chat, Notification | 9 | 5h | BE-018, BE-021, BE-024 |
| BE-030 | 통합 테스트 — Repository | 9 | 6h | BE-011, BE-014, BE-017, BE-020, BE-023 |
| BE-031 | 통합 테스트 — API 레이어 | 9 | 6h | BE-027, BE-030 |
| BE-032 | E2E 테스트 | 9 | 6h | BE-031 |
| BE-033 | 헬스체크 + Graceful Shutdown | 10 | 3h | BE-004, BE-022 |
| BE-034 | 시드 데이터 개발 환경 | 10 | 1h | BE-007 |
| **합계** | | | **~108h** | |

### FE 영역

| ID | 태스크명 | Phase | 시간 | 의존성 |
|----|---------|-------|------|--------|
| FE-001 | Vite + React 초기화 | 0 | 2h | — |
| FE-002 | 디렉토리 + 공통 타입 | 0 | 1.5h | FE-001 |
| FE-003 | Axios 클라이언트 | 0 | 3h | FE-001, FE-002 |
| FE-004 | Zustand 스토어 | 0 | 1.5h | FE-001, FE-002 |
| FE-005 | TanStack Query + Router | 0 | 2h | FE-003, FE-004 |
| FE-006 | Shared 컴포넌트 | 1 | 4h | FE-001, FE-002 |
| FE-007 | 레이아웃 + GNB | 1 | 2.5h | FE-005, FE-006 |
| FE-008 | 날짜 유틸리티 | 1 | 2h | FE-001 |
| FE-009 | Auth API | 2 | 1h | FE-003 |
| FE-010 | Auth Queries | 2 | 1.5h | FE-004, FE-009 |
| FE-011 | use-auth | 2 | 1h | FE-010 |
| FE-012 | LoginForm + SignupForm | 2 | 3h | FE-006, FE-011 |
| FE-013 | Team API | 3 | 1.5h | FE-003 |
| FE-014 | Team Queries | 3 | 2h | FE-002, FE-013 |
| FE-015 | use-team | 3 | 1h | FE-004, FE-014 |
| FE-016 | TeamListPage + 컴포넌트 | 3 | 4h | FE-006, FE-007, FE-015 |
| FE-017 | Schedule API | 4 | 1.5h | FE-003 |
| FE-018 | Schedule Queries | 4 | 2h | FE-002, FE-017 |
| FE-019 | use-calendar | 4 | 2h | FE-008, FE-018 |
| FE-020 | MonthlyCalendar | 4 | 3h | FE-006, FE-008, FE-019 |
| FE-021 | WeeklyCalendar | 4 | 4h | FE-006, FE-008, FE-019 |
| FE-022 | ScheduleForm + CalendarPage | 4 | 5h | FE-006, FE-015, FE-018, FE-020, FE-021 |
| FE-023 | Chat API | 5 | 1.5h | FE-003 |
| FE-024 | use-long-polling | 5 | 4h | FE-023 |
| FE-025 | Chat Queries | 5 | 1.5h | FE-002, FE-023 |
| FE-026 | use-chat | 5 | 2.5h | FE-024, FE-025 |
| FE-027 | Chat 컴포넌트 + DailyViewPage | 5 | 5h | FE-006, FE-018, FE-026 |
| FE-028 | Notification API | 6 | 0.5h | FE-003 |
| FE-029 | Notification Queries | 6 | 1.5h | FE-002, FE-028 |
| FE-030 | use-notification | 6 | 1.5h | FE-029 |
| FE-031 | Notification 컴포넌트 | 6 | 2.5h | FE-006, FE-030 |
| FE-032 | Auth 테스트 | 7 | 3h | FE-011, FE-012 |
| FE-033 | Calendar 테스트 | 7 | 3h | FE-019, FE-022 |
| FE-034 | Chat 테스트 (Long Polling) | 7 | 4h | FE-024, FE-026, FE-027 |
| FE-035 | Notification 테스트 | 7 | 2h | FE-030, FE-031 |
| FE-036 | 팀 격리 + 권한 테스트 | 7 | 2.5h | FE-016, FE-022, FE-031 |
| FE-037 | Error Boundary | 8 | 2h | FE-005, FE-006 |
| FE-038 | 접근성 | 8 | 2h | FE-006, FE-007, FE-031 |
| FE-039 | 빌드 최적화 | 8 | 2h | FE-005 |
| **합계** | | | **~92h** | |

---

## 비즈니스 규칙 → 태스크 커버리지

| 규칙 | 설명 | DB | BE | FE |
|------|------|----|----|-----|
| BR-10 | 인증 필수 | — | BE-008 | FE-005 |
| BR-15 | 이메일 유일성 | DB-004, DB-012 | BE-012 | FE-012, FE-032 |
| BR-20 | 일정 수정 권한 | — | BE-018, BE-031 | FE-022, FE-033, FE-036 |
| BR-25 | 초대 중복 방지 | DB-006, DB-011 | BE-015, BE-031 | FE-016 |
| BR-30 | 변경 요청 흐름 | — | BE-021 | FE-026 |
| BR-35 | 메시지 불변성 | DB-008 | BE-022, BE-031 | FE-034 |
| BR-40 | 채팅-일정 연동 | DB-011 | BE-018, BE-019 | FE-027, FE-033 |
| BR-50 | 날짜별 채팅 격리 | DB-008, DB-011 | BE-021 | FE-026 |
| BR-60 | 팀 격리 | — | BE-009, BE-031 | FE-036 |
| BR-70 | 일정 시간 무결성 | DB-012 | BE-018, BE-029 | FE-022, FE-033 |
| BR-75 | 담당자 소속 검증 | DB-007 | BE-018, BE-029 | FE-022 |
| BR-80 | 팀장 최소 인원 | DB-005, DB-011 | BE-015, BE-031 | — |
| BR-85 | 메시지 내용 필수 | — | BE-021, BE-029 | FE-026, FE-034 |
| BR-90 | 알림 자동 생성 | DB-009, DB-011 | BE-024, BE-029 | FE-035 |

---

## 변경 이력

| 버전 | 날짜 | 내용 |
|------|------|------|
| v1.0 | 2026-05-05 | 최초 작성. DB 16개, BE 34개, FE 39개 태스크 분해. 비즈니스 규칙 커버리지 매핑 포함. |
