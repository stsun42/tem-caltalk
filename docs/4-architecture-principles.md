# Team CalTalk 프로젝트 구조 설계 원칙

> **버전**: v1.2 | **작성일**: 2026-05-04 | **수정일**: 2026-05-05 | **기준 범위**: v1.0 (Modular Monolith)

---

## 1. 최상위 원칙 (모든 스택 공통)

### 1.1 도메인 중심 설계

코드 구조는 기술 계층이 아닌 **도메인 경계**를 기준으로 구성한다.

도메인 5개: `auth` · `team` · `schedule` · `chat` · `notification`

도메인 간 의존 방향:

```
auth  → 전역 userId만 제공 (인증 책임)
team  → schedule, chat, notification에 팀 소속·역할 인터페이스 제공 (인가 책임)
chat  → notification (메시지 저장 완료 이벤트)
```

**auth와 team의 책임 분리**: auth=인증(누구인가), team=인가(이 팀에서 무슨 권한인가). 팀별 role은 team 도메인이 소유한다. 다른 도메인은 team이 노출한 `TeamMembershipGuard` 인터페이스에만 의존하며, team의 내부 Service·Repository를 직접 참조하지 않는다.

### 1.2 Modular Monolith

v1은 **Modular Monolith** — 단일 프로세스이지만 도메인 모듈 간 직접 import를 금지한다.

- 도메인 간 직접 import 차단: ESLint `no-restricted-imports` 규칙으로 강제
- 도메인 간 통신: `shared/events/` 이벤트 버스 또는 명시적으로 export된 인터페이스만 허용
- 기능이 없으면 파일도 없다. 미래 확장을 위한 빈 파일·추상화 금지
- 세 곳에서 반복되기 전까지 공통 추출 금지
- 마이크로서비스 분리는 v2 이후 검토

### 1.3 팀 격리 불변 규칙

**모든 API 요청에서 팀 소속 여부 검증은 생략할 수 없다.**

URL의 `teamId`는 신뢰하지 않는다. 인증된 사용자가 해당 팀에 속해 있는지 항상 DB에서 확인한다 (BR-60).

단일 리소스(`/schedules/:id`) 접근 시에도 리소스의 `team_id` → 사용자 팀 소속 여부 순서로 검증한다 (IDOR 방어).

### 1.4 감사(Audit) 불변성

메시지, 초대 이력은 **삭제하지 않는다**. 읽음/상태 전환으로만 관리한다 (BR-35, BR-90).

---

## 2. 의존성 / 레이어 원칙

### 2.1 백엔드 레이어 구조

```
[HTTP 요청]
    ↓
Router (경로 매핑, 인증 미들웨어 적용)
    ↓
Validator (zod 스키마로 입력 검증, DTO 화이트리스트)
    ↓
Controller (요청 파싱, 응답 직렬화)
    ↓
Service (비즈니스 규칙, 트랜잭션 경계)
    ↓
Repository (Prisma Client 호출, Entity 매핑)
    ↓
[DB: PostgreSQL via Prisma]
```

**의존 방향은 단방향**: Router → Validator → Controller → Service → Repository

- **Validator**: zod 스키마로 런타임 입력 검증만 담당. 비즈니스 검증은 Service.
- **Service**: 트랜잭션 경계의 책임자. 여러 Repository를 호출하는 경우 `prisma.$transaction`으로 묶는다.
- **Repository**: Prisma Client를 통한 DB 접근. DB row → Domain Entity 매핑은 Repository 내부에서 처리. Service를 역참조하지 않는다.
- **Entity**: 도메인 모델. Prisma 생성 타입과 1:1 대응하지 않아도 된다.

### 2.2 프론트엔드 레이어 구조

```
Page (React Router v6 라우트 단위 컴포넌트)
    ↓
Feature (도메인별 UI 조합)
    ├── hooks/ (비즈니스 로직 보유자)
    └── components/ (도메인 결합 UI)
    ↓
Shared Component (도메인 무관 순수 UI)
    ↓
API Client (Axios) / Query (TanStack Query — 서버 상태) / Store (Zustand — 클라이언트 상태)
```

- 비즈니스 로직은 Feature의 hook에 위치한다
- `components/`는 도메인 무관 순수 UI. `features/*/components/`는 도메인 결합 UI
- 서버 상태(캘린더, 채팅)는 TanStack Query 캐시로, 클라이언트 상태(인증 정보, 선택된 팀)만 Zustand store로 관리

### 2.3 이벤트 기반 도메인 통신

도메인 간 부수 효과(Chat → Notification 등)는 **이벤트 발행** 방식으로 처리한다.

```typescript
// 이벤트는 트랜잭션 커밋 이후 발행 (트랜잭션 내 발행 금지)
await db.transaction(async (tx) => {
  const message = await chatRepository.save(tx, payload);
  return message;
});
eventBus.emit('message.created', { teamId, messageId }); // 트랜잭션 외부
```

- v1에서는 `EventBus` 인터페이스 뒤에 인메모리 EventEmitter 구현체를 둔다
- v2에서 Redis Pub/Sub 교체가 구현체 교체만으로 가능해야 한다
- 이벤트 처리 실패 시 로그 기록 + 메모리 재시도 큐로 추적
- 이벤트 페이로드는 최소 식별자(messageId, teamId)만 포함. 핸들러가 재조회
- 이벤트 카탈로그는 `shared/events/contracts.ts`에 타입으로 명시

---

## 3. 코드 / 네이밍 원칙

### 3.1 네이밍 컨벤션

| 대상 | 형식 | 예시 |
|------|------|------|
| 파일명 (백엔드) | kebab-case | `schedule.service.ts` |
| 파일명 (프론트엔드) | PascalCase (컴포넌트), kebab-case (유틸/훅) | `CalendarView.tsx`, `use-calendar.ts` |
| 클래스·타입·인터페이스 | PascalCase | `ScheduleService`, `InvitationStatus` |
| 함수·변수 | camelCase | `getSchedulesByDate`, `teamId` |
| 상수·Enum 값 | UPPER_SNAKE_CASE | `INVITATION_RECEIVED`, `MAX_RETRY` |
| DB 컬럼·테이블 | snake_case | `created_at`, `user_team_roles` |
| API 경로 | kebab-case, 복수 명사 | `/teams/{id}/schedules` |

### 3.2 파일 구성 규칙

- 한 파일은 하나의 책임만 가진다
- 백엔드: `{domain}.{layer}.ts` — `team.service.ts`, `team.repository.ts`
- 프론트엔드: 컴포넌트와 관련 훅은 같은 feature 디렉토리에 배치

### 3.3 API 응답 형식

성공/실패 모두 일관된 구조를 사용한다.

```json
// 성공
{ "data": { ... }, "message": "..." }
{ "data": [ ... ], "message": "..." }

// 실패 (일반 오류)
{ "message": "이미 대기 중인 초대가 존재합니다", "code": "DUPLICATE_INVITATION" }

// 실패 (입력값 검증 오류)
{ "message": "입력값 검증 실패", "errors": [ { "field": "title", "message": "제목은 필수입니다" } ] }
```

> `code` 필드는 선택적이며, 클라이언트가 오류 유형을 구분해야 할 때만 포함한다.

HTTP 상태 코드는 도메인 정의서의 규칙을 따른다 (400 / 401 / 403 / 404 / 405 / 409).

### 3.4 날짜/시간 처리

- 서버는 모든 시간을 **UTC**로 저장한다
- API 요청/응답은 **ISO 8601** 형식 사용 (`2026-05-10T09:00:00Z`)
- 날짜 전용 파라미터는 `YYYY-MM-DD` 형식 사용 (`?date=2026-05-10`)
- 클라이언트에서만 로컬 타임존으로 변환하여 표시한다

---

## 4. 테스트 / 품질 원칙

### 4.1 테스트 계층

| 계층 | 대상 | 도구 | 비율 (가이드라인) |
|------|------|------|---------|
| 단위 테스트 | Service 비즈니스 규칙 (BR-*), 순수 도메인 로직 | Jest (백엔드), Vitest + React Testing Library (프론트엔드) | ~60% |
| 통합 테스트 | Repository + 실제 DB, 권한 검증, 실시간 채팅 흐름 | Jest + Supertest + Testcontainers (PostgreSQL 컨테이너) | ~30% |
| E2E 테스트 | 핵심 사용자 흐름 (SC-01~SC-15) | Playwright | ~10% |

비율은 가이드라인. **도메인별로 조정한다** (auth/team 권한, chat 실시간은 통합 위주).

### 4.2 테스트 원칙

- Repository 계층 테스트: **Mock DB 금지**. Testcontainers로 PostgreSQL 컨테이너 사용
- 비율과 무관하게 반드시 존재해야 하는 **필수 테스트 게이트**:
  - 모든 팀 격리 경로(BR-60) 통합 테스트
  - 감사 불변성(BR-35, BR-90) 통합 테스트
  - 역할 × 행위 권한 매트릭스 (팀장/팀원 × 일정 CRUD) 테스트
- 도메인 이벤트: 발행 측/구독 측 각각 단위 테스트 필수
- API 응답 스키마는 zod로 정의하고 프론트/백이 공유하여 계약 불일치를 방지
- 경계값 테스트 필수: `종료일시 == 시작일시` (BR-70), 만료된 초대 (OQ-05)

### 4.3 핵심 테스트 케이스 (BR 기준)

| 규칙 | 테스트 내용 |
|------|-----------|
| BR-15 | 중복 이메일 회원가입 → 409 |
| BR-20 | 팀원의 일정 생성/수정/삭제 시도 → 403 |
| BR-25 | 중복 Pending 초대 생성 시도 → 409 |
| BR-35 | 메시지 수정/삭제 시도 → 405 |
| BR-60 | 미소속 팀 데이터 접근 → 403 |
| BR-70 | 종료일시 < 시작일시 → 400 / 종료일시 == 시작일시 → 200 |
| BR-80 | 팀장 1명인 팀에서 해당 팀장 제거 → 409 |
| BR-90 | 팀원 메시지 전송 후 팀장 알림 생성 확인 |

### 4.4 코드 품질

- TypeScript strict 모드 활성화
- ESLint + Prettier 설정을 CI에서 강제
- ESLint `no-restricted-imports`로 도메인 간 직접 import 차단
- `any` 타입 사용 금지. 불가피한 경우 주석으로 이유 명시

---

## 5. 설정 / 보안 / 운영 원칙

### 5.1 설정 관리

- 모든 환경 변수는 `.env` 파일로 관리. 코드에 하드코딩 금지
- `.env.example`에 필요한 키 목록을 유지. 실제 값은 포함하지 않는다
- 환경별 설정: `.env.development`, `.env.production`, `.env.test`
- 프로덕션 비밀은 AWS Secrets Manager 등 Secrets Manager 사용
- `JWT_SECRET`, DB 패스워드는 분기별 1회 로테이션
- pre-commit hook(gitleaks 등)으로 비밀 유입 방지

| 변수 | 설명 |
|------|------|
| `DATABASE_URL` | DB 연결 문자열 |
| `JWT_SECRET` | JWT 서명 키 (최소 32자, HS256) |
| `JWT_EXPIRES_IN` | Access Token 만료 시간 (기본값: `15m`) |
| `REFRESH_TOKEN_EXPIRES_IN` | Refresh Token 만료 시간 (기본값: `7d`) |
| `CORS_ORIGIN` | 허용 도메인 (프로덕션에서 명시적 설정) |
| `LONG_POLL_TIMEOUT_MS` | Long Polling 최대 대기 시간 (기본값: `20000`) |

### 5.2 보안 원칙

**인증 (JWT 정책)**
- Access Token: JWT(HS256), **15분 만료**, 클레임에 `sub/iss/aud/exp/iat/jti` 포함
- Refresh Token: 불투명 토큰, **7일 만료**, DB에 해시 저장, Rotation 적용
- 검증 시 허용 알고리즘 화이트리스트 강제 (`alg: none` 거부)
- 비밀번호 변경·계정 잠금·명시적 로그아웃 시 Refresh Token 폐기
- 토큰 저장: `HttpOnly` + `Secure` + `SameSite=Strict` 쿠키. `localStorage` 금지
- Access Token은 메모리(SPA)에만 저장. 쿠키는 Refresh Token 전용

**인가 (RBAC)**
- 미들웨어 체인: `authenticate → loadTeamMembership → authorize(role)`
- 역할: `TEAM_LEADER`, `TEAM_MEMBER` (팀별 독립)
- 서비스 계층에서 재검증: 모든 변경 작업에서 actor의 `team_id`, `role` 재확인
- 권한 변경(팀장 위임 등)은 트랜잭션 내에서 BR-80 불변식 검증

**비밀번호**
- bcrypt, salt rounds ≥ 12
- 에러 메시지에서 이메일 존재 여부를 노출하지 않는다 ("이메일 또는 비밀번호가 올바르지 않습니다")

**입력 검증**
- 모든 외부 입력(req.body, req.query, req.params)은 **zod 스키마**로 검증 후 핸들러 진입
- DTO 화이트리스트 적용. 클라이언트가 보낸 `role`, `is_leader` 등 권한 필드는 서버에서 무시 (Mass Assignment 방지)
- body 크기 제한: JSON `100KB`, 채팅 메시지 본문 `2KB`
- 채팅 메시지: React 기본 이스케이핑 의존, `dangerouslySetInnerHTML` 사용 금지
- 사용자 제공 URL: 렌더링 시 `rel="noopener noreferrer"` 적용
- redirect 파라미터: 동일 출처 화이트리스트만 허용

**속도 제한 (Rate Limiting)**
- 전역: IP당 분당 100 요청
- 로그인: IP당 + 이메일당 분당 5회, 실패 누적 시 점진적 지연
- 회원가입: IP당 시간당 5회
- 채팅 메시지 전송: 사용자당 초당 5건, 분당 60건
- 위반 시 429 응답 + `Retry-After` 헤더
- v1은 인메모리(단일 인스턴스), 수평 확장 시 Redis로 전환

**HTTP 보안 헤더**
- `helmet` 미들웨어 기본 적용
- HSTS: `max-age=31536000; includeSubDomains`
- X-Frame-Options: `DENY`
- Referrer-Policy: `strict-origin-when-cross-origin`
- CSP: `default-src 'self'` (단계적 강화)

**API 보안**
- 모든 상태 변경 요청에 인증 미들웨어 적용
- ORM의 Prepared Statement 사용. 동적 쿼리 문자열 직접 조합 금지
- 프로덕션 CORS: 명시적 화이트리스트. `*` 허용 금지
- HTTPS 전용 배포. TLS 1.2 이상

**팀 격리 (IDOR 방어)**
- 팀 관련 모든 엔드포인트에 `TeamMemberGuard` 미들웨어 적용
- 서비스 계층에서도 팀 소속 여부 재검증 (BR-60)
- 단일 리소스 접근: `리소스.team_id` → 사용자 team membership → (필요 시) owner_id 순서로 검증

### 5.3 감사 로그 (Audit Log)

운영 로그(morgan)와 **보안 감사 로그는 분리**한다.

**감사 로그 테이블 스키마**:
```sql
audit_logs (id, actor_id, actor_ip, action, target_type, target_id, metadata jsonb, created_at)
```
append-only 운영. 애플리케이션 계정에 UPDATE/DELETE 권한 미부여.

**기록 대상 이벤트**:
- 인증: `login_success`, `login_failure`, `logout`, `password_change`
- 인가 위반: `forbidden_team_access` (BR-60 위반)
- 권한 변경: `team_leader_assigned`, `team_member_added/removed`
- 데이터 변경: `schedule_created/updated/deleted` (before/after diff)
- 초대: `invitation_sent`, `invitation_accepted`, `invitation_declined`

**보존 및 보안**:
- 보존 기간: 1년
- 로그에 기록 금지 필드: `password`, `jwt`, `refresh_token`, `session_id` (기록 직전 redact)

### 5.4 DB 인덱스

반드시 생성해야 하는 인덱스:

```sql
-- 날짜별 채팅·일정 조회 최적화 (BR-40, BR-50)
CREATE INDEX idx_messages_team_date ON messages(team_id, chat_date);
CREATE INDEX idx_schedules_team_date ON schedules(team_id, start_at, end_at);

-- 미읽음 알림 카운트 최적화 (BR-90)
CREATE INDEX idx_notifications_user_read ON notifications(user_id, is_read);

-- 중복 초대 검증 (BR-25)
CREATE INDEX idx_invitations_team_user_status ON invitations(team_id, invitee_id, status);
```

마이그레이션은 **Prisma Migrate**로 관리한다 (`prisma migrate dev`). 생성된 SQL 파일은 `prisma/migrations/`에 커밋한다. 롤백이 필요한 경우 rollback 마이그레이션을 별도로 작성한다.

### 5.5 의존성 관리

- 주 1회 `npm audit` 자동 실행 (CI)
- Dependabot/Renovate로 의존성 업데이트 PR 자동화
- 미사용 의존성 제거 (`production` / `devDependencies` 분리 엄수)
- `package-lock.json` 반드시 커밋

### 5.6 운영 / 모니터링

- 구조화된 로깅: **pino** 사용 (JSON 형식). `console.log` 직접 사용 금지
- 요청 로그에는 `requestId`, `userId`, `teamId`, `method`, `path`, `statusCode`, `durationMs` 포함
- 헬스체크 엔드포인트: `GET /health` — DB 연결 상태 포함
- 모니터링 항목: CPU, 메모리, DB 연결 수, Long Polling 활성 연결 수

---

## 6. 백엔드 디렉토리 구조

```
backend/
├── src/
│   ├── app.ts                  # Express 앱 초기화, 미들웨어 등록
│   ├── server.ts               # 서버 실행 진입점 (app과 분리하여 테스트 용이)
│   │
│   ├── domains/                # 도메인별 코드 (핵심 비즈니스 로직)
│   │   ├── auth/
│   │   │   ├── auth.controller.ts
│   │   │   ├── auth.service.ts
│   │   │   ├── auth.repository.ts
│   │   │   ├── auth.dto.ts         # zod 스키마 (입력/출력)
│   │   │   ├── auth.entity.ts      # 도메인 모델
│   │   │   └── auth.types.ts
│   │   ├── team/
│   │   │   ├── team.controller.ts
│   │   │   ├── team.service.ts
│   │   │   ├── team.repository.ts
│   │   │   ├── team.dto.ts
│   │   │   ├── team.entity.ts
│   │   │   └── team.types.ts
│   │   ├── schedule/
│   │   │   ├── schedule.controller.ts
│   │   │   ├── schedule.service.ts
│   │   │   ├── schedule.repository.ts
│   │   │   ├── schedule.dto.ts
│   │   │   ├── schedule.entity.ts
│   │   │   └── schedule.types.ts
│   │   ├── chat/
│   │   │   ├── chat.controller.ts  # Long Polling 엔드포인트 포함
│   │   │   ├── chat.service.ts
│   │   │   ├── chat.repository.ts
│   │   │   ├── chat.dto.ts
│   │   │   ├── chat.entity.ts
│   │   │   └── chat.types.ts
│   │   └── notification/
│   │       ├── notification.controller.ts
│   │       ├── notification.service.ts
│   │       ├── notification.repository.ts
│   │       ├── notification.dto.ts
│   │       ├── notification.entity.ts
│   │       └── notification.types.ts
│   │
│   ├── shared/                 # 도메인 비의존 공통 코드
│   │   ├── middlewares/
│   │   │   ├── auth.middleware.ts       # JWT 검증
│   │   │   ├── rate-limit.middleware.ts # Rate Limiting
│   │   │   └── team-member.guard.ts     # 팀 소속 검증 (BR-60)
│   │   ├── errors/
│   │   │   ├── app-error.ts             # 커스텀 에러 클래스
│   │   │   └── error-handler.ts         # 전역 에러 미들웨어
│   │   ├── events/
│   │   │   ├── event-bus.interface.ts   # EventBus 인터페이스
│   │   │   ├── event-bus.ts             # 인메모리 EventEmitter 구현체
│   │   │   └── contracts.ts             # 이벤트 카탈로그 (타입 정의)
│   │   ├── logger/
│   │   │   └── logger.ts                # 구조화 로깅 (JSON)
│   │   ├── config/
│   │   │   └── env.ts                   # 환경변수 검증 (zod)
│   │   └── utils/
│   │       ├── date.utils.ts
│   │       └── pagination.utils.ts
│   │
│   ├── db/
│   │   ├── client.ts            # Prisma Client 싱글톤
│   │   └── seeds/               # 초기 데이터 (ts-node로 실행)
│   │
│   └── routes/
│       └── index.ts             # 전체 라우터 등록
│
├── prisma/
│   ├── schema.prisma            # Prisma 스키마 (모델 정의)
│   └── migrations/              # Prisma Migrate 자동 생성 (up/down 쌍)
│
├── tests/
│   ├── unit/                    # Service 단위 테스트
│   │   ├── schedule.service.test.ts
│   │   └── ...
│   ├── integration/             # Repository + Testcontainers PostgreSQL
│   │   └── ...
│   └── e2e/                     # 핵심 시나리오 E2E 테스트
│       └── ...
│
├── .env.example
├── package.json
└── tsconfig.json
```

---

## 7. 프론트엔드 디렉토리 구조

```
frontend/
├── src/
│   ├── main.tsx                 # 앱 진입점
│   ├── App.tsx                  # React Router v6 라우터 설정
│   │
│   ├── pages/                   # 라우트 단위 페이지 컴포넌트
│   │   ├── AuthPage.tsx         # 로그인 / 회원가입
│   │   ├── TeamListPage.tsx     # 팀 목록
│   │   ├── CalendarPage.tsx     # 월/주 캘린더 (SC-08, SC-09)
│   │   └── DailyViewPage.tsx    # 일일 뷰 + 채팅 (SC-10, SC-14)
│   │
│   ├── features/                # 도메인별 UI 기능 단위
│   │   ├── auth/
│   │   │   ├── components/
│   │   │   │   ├── LoginForm.tsx
│   │   │   │   └── SignupForm.tsx
│   │   │   ├── hooks/
│   │   │   │   └── use-auth.ts
│   │   │   └── queries/         # React Query 키/훅
│   │   │       └── auth.queries.ts
│   │   ├── team/
│   │   │   ├── components/
│   │   │   │   ├── TeamCard.tsx
│   │   │   │   └── InviteModal.tsx
│   │   │   ├── hooks/
│   │   │   │   └── use-team.ts
│   │   │   └── queries/
│   │   │       └── team.queries.ts
│   │   ├── calendar/
│   │   │   ├── components/
│   │   │   │   ├── MonthlyCalendar.tsx
│   │   │   │   ├── WeeklyCalendar.tsx
│   │   │   │   └── ScheduleForm.tsx
│   │   │   ├── hooks/
│   │   │   │   └── use-calendar.ts
│   │   │   └── queries/
│   │   │       └── schedule.queries.ts
│   │   ├── chat/
│   │   │   ├── components/
│   │   │   │   ├── ChatPanel.tsx
│   │   │   │   ├── MessageList.tsx
│   │   │   │   └── MessageInput.tsx
│   │   │   ├── hooks/
│   │   │   │   ├── use-long-polling.ts  # Long Polling 훅
│   │   │   │   └── use-chat.ts
│   │   │   └── queries/
│   │   │       └── chat.queries.ts
│   │   └── notification/
│   │       ├── components/
│   │       │   ├── NotificationBadge.tsx
│   │       │   └── NotificationList.tsx
│   │       ├── hooks/
│   │       │   └── use-notification.ts
│   │       └── queries/
│   │           └── notification.queries.ts
│   │
│   ├── components/              # 재사용 가능한 UI 원자 단위 (도메인 무관)
│   │   ├── Button.tsx
│   │   ├── Modal.tsx
│   │   ├── Badge.tsx
│   │   └── LoadingSpinner.tsx
│   │
│   ├── api/                     # 백엔드 HTTP 통신 클라이언트
│   │   ├── client.ts            # Axios 인스턴스, 인터셉터 (토큰 갱신 포함)
│   │   ├── auth.api.ts
│   │   ├── team.api.ts
│   │   ├── schedule.api.ts
│   │   ├── chat.api.ts
│   │   └── notification.api.ts
│   │
│   ├── store/                   # Zustand — 클라이언트 전역 상태 (인증 정보, 선택된 팀)
│   │   ├── auth.store.ts
│   │   └── team.store.ts
│   │
│   ├── hooks/                   # 도메인 무관 공용 훅
│   │   └── use-debounce.ts
│   │
│   ├── constants/               # 라우트 경로, 권한 상수
│   │   └── routes.ts
│   │
│   └── utils/
│       ├── date.utils.ts        # UTC ↔ 로컬 타임존 변환
│       └── error.utils.ts
│
├── tests/
│   └── ...
│
├── .env.example
├── index.html                   # Vite 진입점
├── vite.config.ts               # Vite 빌드 설정
├── vitest.config.ts             # Vitest 테스트 설정
├── package.json
└── tsconfig.json
```

---

## 부록: 원칙-규칙 대응표

| 원칙 섹션 | 근거 규칙 / 문서 |
|---------|--------------|
| 팀 격리 불변 (1.3) | BR-60, PRD 8.3 |
| IDOR 방어 (1.3) | BR-60, OWASP A01 |
| 감사 불변성 (1.4) | BR-35, PRD 3.3 |
| Modular Monolith (1.2) | PRD 8.5 |
| 이벤트 기반 통신 (2.3) | BR-90, 도메인 정의서 5절 |
| JWT Refresh Token (5.2) | PRD Q-006, OWASP A07 |
| Rate Limiting (5.2) | OWASP A07 |
| 감사 로그 (5.3) | BR-35, BR-90, OWASP A09 |
| HTTP 보안 헤더 (5.2) | OWASP A05 |
| Long Polling (7 chat) | PRD 8.1 |
| 복합 인덱스 (5.4) | PRD 8.2, 도메인 정의서 7절 |
| 낙관적 잠금 | OQ-06, PRD Q-002 |
