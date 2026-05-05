# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

> **핵심 지침**
> 1. **오버엔지니어링 금지** — 필요 이상의 추상화·구조화를 하지 않는다.
> 2. **모든 처리 결과에 대한 설명은 한국어로** — 코드 외 모든 응답은 한국어로 작성한다.

## 반드시 준수할 것

- 모든 대화는 한국어로
- 오버엔지니어링 금지 — 세 곳에서 반복되기 전까지 공통 추출 금지
- 간단한 코드에는 주석 금지, 복잡한 코드에만 주석 작성
- `any` 타입 사용 금지. 불가피한 경우 주석으로 이유 명시
- `console.log` 직접 사용 금지 — pino 로거 사용

## 현재 레포지토리 구성

이 저장소는 아직 구현 코드가 없고, 설계·문서 단계입니다.

```
tem-caltalk/
├── docs/               # 기획·설계 문서 (PRD, 도메인 정의, 아키텍처 등)
├── swagger/
│   └── swagger.json    # OpenAPI 3.0.3 API 명세 (구현의 계약서)
└── mockup/             # Swagger 기반 목 서버
    └── server.js
```

### 목 서버 실행

```bash
cd mockup
node server.js          # 기본 실행
npx nodemon server.js   # 핫 리로드
```

- 목 API: `http://localhost:3000/api/v1/...`
- Swagger UI: `http://localhost:3000/docs`

## 예정된 구현 구조

구현 시작 시 루트에 `backend/`와 `frontend/` 디렉토리를 분리하여 생성합니다.

**기술 스택**

| 영역 | 기술 |
|------|------|
| 백엔드 런타임 | Node.js LTS + TypeScript (strict) |
| 백엔드 프레임워크 | Express.js |
| 입력 검증 | zod |
| 인증 | JWT(HS256) + bcrypt (salt ≥ 12) |
| ORM | Prisma + PostgreSQL |
| 이벤트 버스 | Node.js EventEmitter (v2 → Redis 교체 경로) |
| 프론트엔드 | React 18 + TypeScript + Vite |
| 라우팅 | React Router v6 |
| 서버 상태 | TanStack Query v5 |
| 클라이언트 상태 | Zustand (인증 정보, 선택된 팀만) |
| HTTP 클라이언트 | Axios (인터셉터로 토큰 갱신 중앙화) |
| 테스트 (백엔드) | Jest + Supertest + Testcontainers |
| 테스트 (프론트) | Vitest + React Testing Library |
| E2E | Playwright |

**예정 백엔드 명령어 (backend/)**

```bash
npm run dev          # nodemon + ts-node 개발 서버
npm run build        # TypeScript 컴파일
npm run test         # Jest 단위 + 통합 테스트
npm run db:migrate:dev    # Prisma 마이그레이션 (개발)
npm run db:generate       # Prisma Client 재생성
npm run db:seed           # 초기 데이터 투입
```

## 아키텍처 핵심 원칙

### 도메인 구조 (5개)

```
auth  →  team  →  schedule
                →  chat  →  notification
                →  notification
```

- **auth**: 인증(누구인가)만 담당. userId만 외부 제공
- **team**: 인가(팀에서 무슨 권한인가). `TeamMembershipGuard` 인터페이스로만 노출
- 도메인 간 직접 import 금지 — `shared/events/` 이벤트 버스 또는 명시적 export 인터페이스만 허용
- ESLint `no-restricted-imports`로 강제

### 백엔드 레이어 (단방향 의존)

```
Router → Validator(zod) → Controller → Service → Repository → [PostgreSQL]
```

- **Service**: 트랜잭션 경계 책임자. 비즈니스 규칙 적용. `prisma.$transaction`으로 묶기
- **Repository**: Prisma 호출만. Service 역참조 금지
- **이벤트**: 트랜잭션 커밋 이후 발행 (트랜잭션 내 발행 금지)

### 팀 격리 불변 규칙 (BR-60)

URL의 `teamId`를 신뢰하지 않는다. **모든 팀 관련 요청**에서 인증된 사용자가 해당 팀에 속해 있는지 DB에서 확인. 단일 리소스 접근 시 `리소스.team_id → 사용자 team membership` 순서로 검증 (IDOR 방어).

### 감사 불변성

메시지·초대 이력은 삭제하지 않는다. 상태 전환으로만 관리 (BR-35, BR-90).

### Long Polling (채팅)

`GET /teams/{teamId}/chats/{date}/messages?after={lastMessageId}`

- 새 메시지 없으면 최대 20초 대기 후 빈 배열 반환
- 클라이언트는 응답 수신 즉시(빈 응답 포함) 다음 요청 재전송
- 네트워크 오류 시 3초 후 재시도, 최대 3회

### 낙관적 잠금 (일정 수정)

일정 수정 요청에 `version` 필드 필수. 현재 version과 불일치 시 409 반환 (OQ-06).

## API 응답 형식

```json
// 성공
{ "data": { ... }, "message": "..." }

// 실패 (일반)
{ "message": "이미 대기 중인 초대가 존재합니다", "code": "DUPLICATE_INVITATION" }

// 실패 (입력 검증)
{ "message": "입력값 검증 실패", "errors": [{ "field": "title", "message": "제목은 필수입니다" }] }
```

## 네이밍 컨벤션

| 대상 | 형식 |
|------|------|
| 백엔드 파일명 | `schedule.service.ts` (kebab-case) |
| 프론트엔드 컴포넌트 | `CalendarView.tsx` (PascalCase) |
| 프론트엔드 훅/유틸 | `use-calendar.ts` (kebab-case) |
| 클래스·타입·인터페이스 | `ScheduleService` (PascalCase) |
| 상수·Enum 값 | `INVITATION_RECEIVED` (UPPER_SNAKE_CASE) |
| DB 컬럼·테이블 | `created_at`, `user_team_roles` (snake_case) |

## 필수 테스트 게이트

Mock DB 금지 — Testcontainers로 실제 PostgreSQL 컨테이너 사용.

반드시 존재해야 하는 테스트:
- 팀 격리 경로 통합 테스트 (BR-60)
- 감사 불변성 통합 테스트 (BR-35, BR-90)
- 역할 × 행위 권한 매트릭스 (팀장/팀원 × 일정 CRUD)
- 경계값: `종료일시 == 시작일시` (BR-70), 만료된 초대 (OQ-05)

## 환경 변수

| 변수 | 설명 |
|------|------|
| `DATABASE_URL` | PostgreSQL 연결 문자열 |
| `JWT_SECRET` | JWT 서명 키 (최소 32자) |
| `JWT_EXPIRES_IN` | Access Token 만료 (기본: `15m`) |
| `REFRESH_TOKEN_EXPIRES_IN` | Refresh Token 만료 (기본: `7d`) |
| `CORS_ORIGIN` | 허용 도메인 |
| `LONG_POLL_TIMEOUT_MS` | Long Polling 최대 대기 (기본: `20000`) |

`.env.example`에 키 목록 유지. `.env.development`, `.env.production`, `.env.test`는 gitignore.
