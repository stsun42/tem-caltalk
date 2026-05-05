# Team CalTalk 기술 아키텍처 다이어그램

> **버전**: v1.2 | **작성일**: 2026-05-05 | **수정일**: 2026-05-05 | **기준**: v1.0 Modular Monolith

---

## 0. 기술 스택 명세

| 영역 | 기술 | 비고 |
|------|------|------|
| **런타임** | Node.js LTS + TypeScript (strict) | 백엔드 |
| **백엔드 프레임워크** | Express.js | REST API, Long Polling |
| **ORM / 마이그레이션** | Prisma | schema.prisma 기반, Prisma Migrate |
| **데이터베이스** | PostgreSQL | 단일 인스턴스 (v1) |
| **인증** | JWT(HS256) — jsonwebtoken | Access 15분, Refresh 7일 |
| **비밀번호** | bcrypt (salt rounds ≥ 12) | |
| **입력 검증** | zod | API 레이어 런타임 검증 |
| **로깅** | pino | JSON 구조화 로깅 |
| **보안 헤더** | helmet | |
| **속도 제한** | express-rate-limit | |
| **이벤트 버스** | Node.js EventEmitter (인메모리) | v2에서 Redis Pub/Sub으로 교체 가능 |
| **프론트엔드 빌드** | Vite | |
| **프론트엔드 프레임워크** | React 18 + TypeScript | |
| **라우팅** | React Router v6 | |
| **서버 상태** | TanStack Query (React Query v5) | 캘린더, 채팅, 알림 |
| **클라이언트 상태** | Zustand | 인증 정보, 선택된 팀 |
| **HTTP 클라이언트** | Axios | 토큰 갱신 인터셉터 포함 |
| **백엔드 테스트** | Jest + Supertest + Testcontainers | 통합 테스트: PostgreSQL 컨테이너 |
| **프론트엔드 테스트** | Vitest + React Testing Library | |
| **E2E 테스트** | Playwright | |

---

## 1. 전체 시스템 구조

```mermaid
graph TD
    Browser["브라우저 (웹 클라이언트)"]

    subgraph Backend["백엔드 (Node.js + TypeScript · Express · Prisma)"]
        API["REST API\n(Express · zod · helmet)"]
        LongPoll["Long Polling\n채팅 엔드포인트"]
        EventBus["EventBus\n(인메모리)"]

        subgraph Domains["도메인 모듈"]
            Auth["auth"]
            Team["team"]
            Schedule["schedule"]
            Chat["chat"]
            Notification["notification"]
        end
    end

    DB[("PostgreSQL")]

    Browser -->|HTTP / Long Polling| API
    Browser -->|Long Polling| LongPoll
    API --> Domains
    LongPoll --> Chat
    Chat -->|이벤트 발행| EventBus
    EventBus -->|알림 생성| Notification
    Domains --> DB
```

---

## 2. 도메인 의존 관계

```mermaid
graph LR
    Auth["auth\n(인증)"]
    Team["team\n(인가·팀 관리)"]
    Schedule["schedule\n(일정)"]
    Chat["chat\n(채팅)"]
    Notification["notification\n(알림)"]

    Auth -->|userId 제공| Team
    Auth -->|userId 제공| Schedule
    Auth -->|userId 제공| Chat
    Auth -->|userId 제공| Notification

    Team -->|소속·역할 검증| Schedule
    Team -->|소속 검증| Chat
    Team -->|초대 이벤트| Notification

    Chat -->|message.created 이벤트| Notification
```

---

## 3. 백엔드 레이어 구조

```mermaid
graph TD
    HTTP["HTTP 요청"]
    Router["Router\n경로 매핑 · 미들웨어"]
    Validator["Validator\nzod 입력 검증"]
    Controller["Controller\n요청 파싱 · 응답"]
    Service["Service\n비즈니스 규칙 · 트랜잭션"]
    Repository["Repository\nDB 쿼리 · Entity 매핑"]
    DB[("PostgreSQL")]

    HTTP --> Router --> Validator --> Controller --> Service --> Repository --> DB
```

---

## 4. 프론트엔드 레이어 구조

```mermaid
graph TD
    Page["Page\n라우트 단위"]
    Feature["Feature\nhooks + components"]
    SharedUI["Shared Component\n순수 UI"]
    API["API Client\n(Axios)"]
    Query["TanStack Query\n서버 상태 캐시"]
    Store["Zustand\n클라이언트 상태"]

    Page --> Feature
    Feature --> SharedUI
    Feature --> Query
    Feature --> Store
    Query --> API
    Store --> API
```

---

## 5. 채팅 Long Polling 흐름

```mermaid
sequenceDiagram
    participant C as 클라이언트
    participant S as 서버
    participant DB as PostgreSQL

    C->>S: GET /teams/{teamId}/chats/{date}/messages?after={lastId}
    S->>DB: 새 메시지 조회
    alt 새 메시지 있음
        DB-->>S: messages[]
        S-->>C: 200 OK (즉시 반환)
    else 새 메시지 없음
        S->>S: 최대 20초 대기
        S-->>C: 200 OK (빈 배열)
    end
    C->>S: 즉시 재요청
```

---

## 6. 팀원 초대 흐름

```mermaid
sequenceDiagram
    participant L as 팀장
    participant S as 서버
    participant M as 팀원

    L->>S: POST /teams/{id}/invitations
    S->>S: Pending 초대 생성 (BR-25 검증)
    S->>S: INVITATION_RECEIVED 알림 생성
    S-->>L: 201 Created

    M->>S: PATCH /invitations/{id}/accept
    S->>S: 팀원 등록 (UserTeamRole)
    S->>S: INVITATION_RESPONDED 알림 생성
    S-->>M: 200 OK
```

---

## 7. ERD (Entity Relationship Diagram)

```mermaid
erDiagram
    User {
        uuid    id           PK
        string  name
        string  email        UK
        string  password_hash
        enum    status       "Active | Inactive"
        datetime created_at
        datetime updated_at
    }

    Team {
        uuid    id           PK
        string  name
        enum    status       "Active | Deleted"
        datetime created_at
        datetime updated_at
    }

    UserTeamRole {
        uuid    id           PK
        uuid    user_id      FK
        uuid    team_id      FK
        enum    role         "TEAM_LEADER | TEAM_MEMBER"
        datetime created_at
    }

    Invitation {
        uuid    id           PK
        uuid    team_id      FK
        uuid    inviter_id   FK
        uuid    invitee_id   FK
        enum    status       "Pending | Accepted | Declined | Expired"
        datetime created_at
        datetime expires_at
    }

    Schedule {
        uuid    id           PK
        uuid    team_id      FK
        uuid    created_by   FK
        string  title
        datetime start_at
        datetime end_at
        enum    status       "Scheduled | Done | Cancelled"
        int     version      "낙관적 잠금 (OQ-06)"
        datetime created_at
        datetime updated_at
    }

    ScheduleAssignee {
        uuid    schedule_id  FK
        uuid    user_id      FK
    }

    Chat {
        uuid    id           PK
        uuid    team_id      FK
        date    chat_date
        datetime created_at
    }

    Message {
        uuid    id           PK
        uuid    chat_id      FK
        uuid    author_id    FK
        string  content
        datetime created_at
    }

    Notification {
        uuid    id            PK
        uuid    recipient_id  FK
        enum    type          "MESSAGE_RECEIVED | INVITATION_RECEIVED | INVITATION_RESPONDED"
        uuid    reference_id  "원인 엔티티 ID"
        boolean is_read
        datetime created_at
    }

    AuditLog {
        uuid    id           PK
        uuid    actor_id     FK
        string  actor_ip
        string  action
        string  target_type
        uuid    target_id
        jsonb   metadata
        datetime created_at
    }

    User           ||--o{ UserTeamRole    : "소속"
    Team           ||--o{ UserTeamRole    : "구성"
    User           ||--o{ Invitation      : "발송 (inviter)"
    User           ||--o{ Invitation      : "수신 (invitee)"
    Team           ||--o{ Invitation      : "소속"
    Team           ||--o{ Schedule        : "소속"
    Team           ||--o{ Chat            : "소속"
    User           ||--o{ Schedule        : "생성 (created_by)"
    Schedule       ||--o{ ScheduleAssignee : "담당"
    User           ||--o{ ScheduleAssignee : "담당자"
    Chat           ||--o{ Message         : "포함"
    User           ||--o{ Message         : "작성"
    User           ||--o{ Notification    : "수신"
    User           ||--o{ AuditLog        : "행위자"
```

**주요 설계 결정:**

| 결정 | 내용 |
|------|------|
| Chat은 (team_id, chat_date) 복합 유니크 | 팀당 날짜별 채팅방 1개 보장 |
| Message는 chat_id 참조 | Chat 엔티티를 통해 팀·날짜 context 획득 |
| ScheduleAssignee 별도 테이블 | 담당자 0명 이상 허용 (BR-75) |
| Schedule.version 컬럼 | 낙관적 잠금 (OQ-06, 동시 편집 충돌 방지) |
| AuditLog.actor_id nullable 허용 | 시스템 이벤트(배치 등) 기록 가능 |
| UserTeamRole 팀별 독립 | 동일 사용자가 팀마다 다른 역할 보유 가능 |
| Notification.reference_id | Message 또는 Invitation ID를 다형성으로 참조 |
