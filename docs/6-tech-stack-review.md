# Team CalTalk 기술 스택 분석 및 선택

> **버전**: v1.0 | **작성일**: 2026-05-05 | **기준**: PRD v1.3 + 아키텍처 원칙

---

## 1. 요구사항 분석 요약

PRD에서 도출한 기술 선택의 핵심 제약 조건:

| 제약 | 내용 |
|------|------|
| **팀 규모** | 소규모 팀 5~10명, MVP 50개 팀 이하 |
| **실시간성** | 채팅 실시간 수신 필요 (단, WebSocket 불필요 — Long Polling으로 충분) |
| **보안** | JWT 인증, 팀 격리, 입력 검증, 감사 추적 |
| **개발 속도** | 2026년 Q3 MVP 출시, 소규모 팀 개발 |
| **확장성** | v1은 모놀리식, v2에서 마이크로서비스 분리 가능성 |
| **감사(Audit)** | 메시지 불변, 모든 변경 이력 추적 |

---

## 2. 백엔드 기술 스택

### 2.1 런타임 · 언어: Node.js LTS + TypeScript (strict)

**선택 이유**
- 프론트엔드와 동일 언어로 코드베이스 일관성 확보 (타입 공유 가능)
- Long Polling의 비동기 I/O 처리에 Node.js 이벤트 루프가 적합
- TypeScript strict 모드로 런타임 전 타입 버그 차단 → Audit 신뢰성 향상

**대안 비교**
| 대안 | 제외 이유 |
|------|---------|
| Python (FastAPI) | 팀이 TS 중심, 언어 혼용 비용 |
| Go | 생산성 대비 러닝커브 과다, 소규모 MVP 불필요 |
| Java (Spring) | 설정 오버헤드, MVP 속도에 비효율 |

---

### 2.2 웹 프레임워크: Express.js

**선택 이유**
- REST API + Long Polling 엔드포인트를 최소 설정으로 구현
- 미들웨어 생태계 풍부 (helmet, express-rate-limit 등 바로 사용)
- v1 트래픽 규모(소규모 팀 50개)에서 NestJS·Fastify 등의 추가 추상화는 오버엔지니어링

**대안 비교**
| 대안 | 제외 이유 |
|------|---------|
| NestJS | 구조가 강제되어 MVP 속도 저하, 규모 대비 과도 |
| Fastify | 성능 우위는 있으나 현 규모에서 차이 미미 |
| Hono | 생태계 미성숙, 팀 숙련도 리스크 |

---

### 2.3 입력 검증: zod

**선택 이유**
- TypeScript 타입 추론과 런타임 검증을 단일 스키마로 통합
- 스키마를 프론트엔드와 공유 가능 (향후 monorepo 전환 시 유리)
- SQL Injection·입력 오류를 API 레이어에서 조기 차단 (PRD 보안 요건)

---

### 2.4 인증: JWT(HS256) + bcrypt

**선택 이유**
- Access Token 15분 / Refresh Token 7일 전략으로 탈취 피해 최소화
- bcrypt salt rounds ≥ 12로 비밀번호 브루트포스 방어
- 서버 Stateless → 수평 확장 용이 (v2 대비)
- Refresh Token 서버 측 저장으로 로그아웃 시 즉시 무효화 가능 (Q-006 해결)

---

### 2.5 이벤트 버스: Node.js EventEmitter (인메모리)

**선택 이유**
- 채팅 메시지 발생 시 알림 생성 트리거가 단일 프로세스 내 처리로 충분
- v1 모놀리식 구조에서 Redis Pub/Sub 도입은 오버엔지니어링
- v2 스케일아웃 시 Redis Pub/Sub으로 교체 경로 명확

---

### 2.6 보안·모니터링 미들웨어

| 라이브러리 | 역할 | PRD 근거 |
|---------|------|---------|
| helmet | HTTP 보안 헤더 자동 설정 | 섹션 8.3 보안 |
| express-rate-limit | API 속도 제한 | 브루트포스 방어 |
| pino | JSON 구조화 로깅 | 감사 추적, 성능 모니터링 |

---

## 3. 데이터베이스 기술 스택

### 3.1 데이터베이스: PostgreSQL

**선택 이유**
- ACID 트랜잭션으로 일정 동시 편집 충돌(낙관적 잠금 via version 컬럼) 보장 (Q-002)
- JSONB 지원으로 AuditLog.metadata 유연한 저장
- (team_id, chat_date) 복합 유니크 제약, 복합 인덱스 → 채팅·일정 조회 최적화
- 성숙한 생태계, Prisma와 완벽 통합

**대안 비교**
| 대안 | 제외 이유 |
|------|---------|
| MySQL | JSONB 부재, Prisma와의 기능 호환성 열위 |
| MongoDB | 스키마 유연성 불필요, 트랜잭션 복잡성, 팀 격리 쿼리에 불리 |
| SQLite | 동시성 제한, v2 스케일아웃 불가 |

---

### 3.2 ORM / 마이그레이션: Prisma

**선택 이유**
- TypeScript-first: 스키마에서 타입 자동 생성 → 런타임 타입 오류 제거
- Prisma Migrate로 schema.prisma 단일 파일에서 마이그레이션 관리
- Prepared Statement 기본 적용 → SQL Injection 방어 (PRD 섹션 8.3)
- Testcontainers와 결합한 통합 테스트 용이

**대안 비교**
| 대안 | 제외 이유 |
|------|---------|
| TypeORM | 데코레이터 기반 설정 복잡, 타입 추론 품질 열위 |
| Drizzle | 생태계 초기 단계, 팀 숙련도 리스크 |
| 순수 SQL (node-postgres) | 타입 안전성 직접 관리 부담 |

---

## 4. 프론트엔드 기술 스택

### 4.1 빌드 도구: Vite

**선택 이유**
- ESM 기반 HMR로 개발 생산성 우수 (CRA 대비 10배 이상 빠른 시작)
- React + TypeScript 템플릿 공식 지원
- 번들 최적화 (코드 스플리팅, 트리쉐이킹) 기본 제공

---

### 4.2 UI 프레임워크: React 18 + TypeScript

**선택 이유**
- 캘린더 뷰 (월/주/일) + 채팅 UI의 복잡한 컴포넌트 트리에 React 컴포넌트 모델 최적
- Concurrent Features(Suspense, Transitions)로 Long Polling 응답 중 UI 블로킹 방지
- 생태계 최대 → 캘린더 라이브러리, 날짜 처리 등 선택지 풍부

---

### 4.3 라우팅: React Router v6

**선택 이유**
- 중첩 라우팅으로 팀 선택 → 캘린더 → 일일 뷰 계층 구조 표현 용이
- Loader/Action 패턴으로 데이터 페칭과 라우팅 통합 가능 (선택적 활용)

---

### 4.4 서버 상태 관리: TanStack Query (React Query v5)

**선택 이유**
- 캘린더(월/주/일), 채팅 메시지, 알림 데이터의 캐시·갱신·동기화를 선언적으로 처리
- Long Polling 구현 시 `refetchInterval` + `enabled` 패턴으로 간결하게 표현
- 낙관적 업데이트로 일정 수정 UX 개선 가능

---

### 4.5 클라이언트 상태 관리: Zustand

**선택 이유**
- 인증 정보(토큰, 사용자), 선택된 팀 등 전역 클라이언트 상태는 소량
- Redux 대비 보일러플레이트 최소화 → 오버엔지니어링 방지
- Context API 대비 리렌더링 최적화

---

### 4.6 HTTP 클라이언트: Axios

**선택 이유**
- 인터셉터로 Access Token 만료 시 Refresh Token 재발급 로직 중앙화
- 요청/응답 변환, 에러 처리 표준화
- Fetch API 대비 브라우저 호환성, 취소 토큰 처리 용이

---

## 5. 테스트 전략

| 레이어 | 도구 | 전략 |
|--------|------|------|
| **백엔드 통합 테스트** | Jest + Supertest + Testcontainers | 실제 PostgreSQL 컨테이너로 테스트 — mock DB 금지 (운영 불일치 방지) |
| **프론트엔드 단위 테스트** | Vitest + React Testing Library | 컴포넌트 행동 기반 테스트, 구현 세부사항 테스트 지양 |
| **E2E 테스트** | Playwright | 핵심 시나리오(팀 생성, 초대, 일정 CRUD, 채팅) 커버 |

---

## 6. 최종 기술 스택 요약

```
백엔드
├── 런타임:     Node.js LTS + TypeScript (strict)
├── 프레임워크: Express.js
├── 검증:       zod
├── 인증:       JWT(HS256) + bcrypt (salt ≥ 12)
├── 이벤트:     Node.js EventEmitter (인메모리, v2→ Redis)
├── 보안:       helmet + express-rate-limit
└── 로깅:       pino

데이터베이스
├── RDBMS: PostgreSQL
└── ORM:   Prisma (TypeScript-first, Migrate 내장)

프론트엔드
├── 빌드:        Vite
├── 프레임워크:  React 18 + TypeScript
├── 라우팅:      React Router v6
├── 서버 상태:   TanStack Query v5
├── 클라이언트:  Zustand
└── HTTP:        Axios

테스트
├── 백엔드: Jest + Supertest + Testcontainers
├── 프론트: Vitest + React Testing Library
└── E2E:    Playwright
```

---

## 7. 주요 설계 결정 요약

| 결정 | 근거 |
|------|------|
| WebSocket 미사용 → Long Polling | v1 소규모 트래픽에서 배포 복잡성 대비 효과 미미 |
| 모놀리식 구조 (v1) | 50개 팀 MVP 규모에서 분산 시스템 오버헤드 불필요 |
| 인메모리 EventEmitter | 단일 프로세스 내 알림 생성 충분, v2 Redis로 교체 경로 확보 |
| Prisma 선택 | TypeScript 타입 자동 생성 + SQL Injection 방어 동시 해결 |
| Testcontainers 실 DB 테스트 | mock/운영 불일치로 인한 장애 예방 |
