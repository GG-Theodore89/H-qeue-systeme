# Virtual Queue MVP — Conception and Implementation Blueprint

This document is the shared source of truth for the first working version of the virtual queue platform. It is deliberately limited to an MVP that two developers can complete and demonstrate reliably.

## How to use this document

- Sections 1–5 define **what** the MVP must do.
- Sections 6–18 define the architecture, data, API, realtime behavior, security, and tests.
- Sections 19–22 define teamwork, completion rules, and frozen implementation decisions.
- Sections 23–30 explain the classes/modules, their methods, and the exact file map.
- Section 31 is the step-by-step realization path; follow it in order.
- Sections 32–36 provide difficult-code recipes, authorization, errors, teamwork, and quality gates.

Before coding, both developers should read sections 5, 11, 12, 22, 25, and 31. During coding, treat the acceptance checks at the end of every realization step as merge gates.

## 1. MVP goal

An organization operates one virtual queue. A visitor scans a QR code, receives a ticket without creating an account, follows their position live, and is called by a staff member. The kiosk, staff dashboard, and visitor screen update without refreshing.

```mermaid
flowchart LR
    C[Customer phone] -->|Scan QR| K[Kiosk URL]
    K -->|Open join page| C
    C -->|Create ticket| API[Express API]
    S[Staff dashboard] -->|Call next / complete / no-show| API
    API --> DB[(PostgreSQL)]
    API --> IO[Socket.IO]
    IO --> C
    IO --> S
    IO --> K
```

## 2. Scope

### Included

- One organization and one queue seeded for the demonstration
- Staff authentication
- Kiosk with QR code and Now Serving display
- Anonymous customer ticket creation
- Unique daily ticket numbers such as `A-001`
- Customer view showing ticket status and people ahead
- Staff waiting list and current ticket
- Call next, recall, start service, complete, and no-show actions
- Real-time updates through Socket.IO
- Persistent ticket and event history in PostgreSQL
- Ticket access and reconnection using a browser-generated private token
- Responsive customer interface
- Basic automated tests

### Not included in the first MVP

- Subscriptions, billing, and plan limits
- Multiple organizations or multiple queues in the UI
- Redis
- Dynamic one-use QR codes
- Custom intake forms and routing
- VIP priority
- Push notifications
- Advanced analytics or machine-learning estimates
- Native mobile application

These exclusions are architectural extensions, not requirements for proving the core queue flow.

## 3. Actors and permissions

```mermaid
flowchart TB
    Customer --> CJ[Join queue]
    Customer --> CV[View own ticket and position]
    Customer --> CC[Cancel own waiting ticket]

    Staff --> Login[Log in]
    Staff --> WL[View waiting list]
    Staff --> Next[Call next]
    Staff --> Recall[Recall current ticket]
    Staff --> Start[Start service]
    Staff --> Finish[Complete or mark no-show]

    Kiosk --> QR[Display join QR]
    Kiosk --> Board[Display called/serving tickets]
```

| Actor | Authentication | Access |
|---|---|---|
| Customer | Private random ticket token | Only their ticket and public queue information |
| Staff | Email/password session or JWT in an HttpOnly cookie | Queue operations and waiting-ticket details |
| Kiosk | Public queue slug | QR code and public Now Serving data only |

## 4. Main user stories

| ID | Story | Acceptance criteria |
|---|---|---|
| US-01 | As a customer, I can scan a QR code and join without an account. | A valid open queue creates exactly one ticket for the browser-generated private token. |
| US-02 | As a customer, I can see my ticket and people ahead. | Position is calculated from waiting tickets created earlier and updates live. |
| US-03 | As staff, I can call the next customer. | The oldest waiting ticket is selected once, even if two requests arrive together. |
| US-04 | As a customer, I know when I am called. | My screen changes to `CALLED` without a refresh. |
| US-05 | As a kiosk viewer, I can see tickets being called. | The public board updates without exposing customer data. |
| US-06 | As staff, I can finish a service or record a no-show. | The status and timestamp are persisted and all screens update. |
| US-07 | As a customer, I can reopen my ticket. | The private token stored locally restores the current server state. |
| US-08 | As staff, I can safely reconnect. | Reloading the dashboard loads the authoritative queue state from the API. |

## 5. Ticket lifecycle

```mermaid
stateDiagram-v2
    [*] --> WAITING: Customer joins
    WAITING --> CANCELLED: Customer cancels
    WAITING --> CALLED: Staff calls next
    CALLED --> CALLED: Staff recalls
    CALLED --> SERVING: Customer arrives
    CALLED --> NO_SHOW: Customer does not arrive
    SERVING --> COMPLETED: Service finishes
    COMPLETED --> [*]
    CANCELLED --> [*]
    NO_SHOW --> [*]
```

Allowed transitions must be enforced by the backend. Clients never change status directly.

| Current state | Allowed next state |
|---|---|
| `WAITING` | `CALLED`, `CANCELLED` |
| `CALLED` | `SERVING`, `NO_SHOW`, `CALLED` (recall event) |
| `SERVING` | `COMPLETED` |
| Terminal states | None |

## 6. Technical architecture

Use a TypeScript monorepo. PostgreSQL is the source of truth. Socket.IO distributes changes but does not own state. On every reload or reconnect, the client obtains a fresh snapshot over REST.

```mermaid
flowchart TB
    subgraph Browser[React web application]
        CR[Customer routes]
        KR[Kiosk route]
        SR[Staff routes]
        SC[Shared components and Socket client]
    end

    subgraph Server[Node.js server]
        REST[Express REST controllers]
        AUTH[Authentication middleware]
        QS[Queue service]
        TS[Ticket service]
        SOCK[Socket.IO gateway]
        VAL[Request validation]
    end

    subgraph Data[Data layer]
        ORM[Prisma ORM]
        PG[(PostgreSQL)]
    end

    CR --> REST
    KR --> REST
    SR --> AUTH --> REST
    SC <--> SOCK
    REST --> VAL
    REST --> QS
    REST --> TS
    QS --> ORM
    TS --> ORM
    ORM --> PG
    QS --> SOCK
    TS --> SOCK
```

### Technology decisions

| Area | Choice |
|---|---|
| Monorepo | npm workspaces |
| Frontend | React, Vite, TypeScript, React Router, TanStack Query |
| Backend | Node.js, Express, TypeScript, Socket.IO |
| Validation | Zod shared between client and server where useful |
| Database | PostgreSQL with Prisma migrations |
| Authentication | Argon2 password hashing; signed JWT/session in Secure HttpOnly cookie |
| QR generation | A frontend QR component encoding the public join URL |
| Testing | Vitest, React Testing Library, Supertest, Playwright |
| Local services | Docker Compose for PostgreSQL |
| Formatting | ESLint and Prettier |

## 7. Repository structure

```text
virtual-queue/
├── apps/
│   ├── web/
│   │   ├── src/
│   │   │   ├── api/                 # HTTP client and query hooks
│   │   │   ├── components/          # Reusable UI elements
│   │   │   ├── features/
│   │   │   │   ├── customer/
│   │   │   │   ├── kiosk/
│   │   │   │   └── staff/
│   │   │   ├── pages/
│   │   │   ├── realtime/            # Socket connection and event handlers
│   │   │   ├── routes/
│   │   │   ├── styles/
│   │   │   ├── test/
│   │   │   └── main.tsx
│   │   ├── index.html
│   │   ├── package.json
│   │   └── vite.config.ts
│   └── api/
│       ├── src/
│       │   ├── config/
│       │   ├── db/
│       │   ├── middleware/
│       │   ├── modules/
│       │   │   ├── auth/
│       │   │   ├── queues/
│       │   │   └── tickets/
│       │   ├── realtime/
│       │   ├── errors/
│       │   ├── app.ts               # Express configuration
│       │   └── server.ts            # HTTP and Socket.IO startup
│       ├── prisma/
│       │   ├── schema.prisma
│       │   ├── migrations/
│       │   └── seed.ts
│       ├── test/
│       ├── package.json
│       └── tsconfig.json
├── packages/
│   └── shared/
│       ├── src/
│       │   ├── contracts/            # REST DTOs and Zod schemas
│       │   ├── events/               # Socket event names and payload types
│       │   └── types/
│       ├── package.json
│       └── tsconfig.json
├── docs/
│   ├── decisions/                    # Short architecture decision records
│   ├── wireframes/
│   └── api.md
├── e2e/
├── .env.example
├── .gitignore
├── docker-compose.yml
├── eslint.config.js
├── package.json
├── README.md
└── tsconfig.base.json
```

## 8. Routes and screens

| Route | Screen | Access |
|---|---|---|
| `/q/:queueSlug` | Join queue | Public |
| `/ticket` | Customer live ticket | Private token restored from localStorage |
| `/kiosk/:queueSlug` | QR and Now Serving board | Public |
| `/staff/login` | Staff login | Public |
| `/staff/queue/:queueId` | Staff dashboard | Staff only |

```mermaid
flowchart LR
    Join[Join page] -->|POST ticket| Ticket[Live ticket]
    Login[Staff login] --> Dashboard[Staff dashboard]
    Kiosk[Kiosk] --> Join
    Dashboard -->|Realtime event| Ticket
    Dashboard -->|Realtime event| Kiosk
```

## 9. Database model

```mermaid
erDiagram
    ORGANIZATION ||--o{ USER : employs
    ORGANIZATION ||--o{ QUEUE : owns
    QUEUE ||--o{ COUNTER : has
    QUEUE ||--o{ TICKET : contains
    USER ||--o{ TICKET_EVENT : performs
    TICKET ||--o{ TICKET_EVENT : records
    COUNTER ||--o{ TICKET : serves

    ORGANIZATION {
        uuid id PK
        string name
        datetime created_at
    }
    USER {
        uuid id PK
        uuid organization_id FK
        string email UK
        string password_hash
        string role
        datetime created_at
    }
    QUEUE {
        uuid id PK
        uuid organization_id FK
        string name
        string slug UK
        string prefix
        boolean is_open
        int last_daily_number
        date number_date
        datetime created_at
        datetime updated_at
    }
    COUNTER {
        uuid id PK
        uuid queue_id FK
        string name
        boolean is_active
    }
    TICKET {
        uuid id PK
        uuid queue_id FK
        uuid counter_id FK "nullable while WAITING"
        string public_token_hash UK
        string join_request_id
        date service_date
        int sequence_number
        string display_number
        string status
        datetime joined_at
        datetime called_at
        datetime service_started_at
        datetime finished_at
        datetime updated_at
    }
    TICKET_EVENT {
        uuid id PK
        uuid ticket_id FK
        uuid actor_user_id FK "nullable for customer/system"
        string type
        json metadata
        datetime created_at
    }
```

### Database rules

- Unique constraint: `(queue_id, service_date, sequence_number)`.
- Unique constraint: `(queue_id, join_request_id)` for idempotent join retries.
- Index tickets on `(queue_id, status, joined_at)`.
- Store only a hash of the customer token; the raw token remains in the browser and is never returned by the API.
- Never derive authority from `display_number`; use UUIDs internally.
- Record every meaningful transition in `ticket_events`.
- Use UTC timestamps in storage and localize only in the UI.
- Generate the queue's `service_date` in `Africa/Casablanca`; lock the queue row while resetting/incrementing `last_daily_number`.
- Order FIFO deterministically by `(joined_at, id)`.
- Add a partial unique database index on `counter_id` for statuses `CALLED` and `SERVING` so one counter cannot hold two active tickets.

## 10. Concurrency-safe Call Next

Two staff members on different counters may click at the same time. Selection and update happen in one transaction using a locked row. A test can create two counters even though the default demonstration seed contains one.

```mermaid
sequenceDiagram
    actor A as Staff A
    actor B as Staff B
    participant API
    participant DB as PostgreSQL
    participant IO as Socket.IO

    par Simultaneous requests
        A->>API: POST call-next {counterId: 1}
        B->>API: POST call-next {counterId: 2}
    end
    API->>DB: BEGIN; select oldest WAITING<br/>FOR UPDATE SKIP LOCKED
    DB-->>API: Ticket A-001
    API->>DB: Set A-001 CALLED at Counter 1; COMMIT
    API->>DB: BEGIN; select oldest unlocked WAITING
    DB-->>API: Ticket A-002
    API->>DB: Set A-002 CALLED at Counter 2; COMMIT
    API->>IO: Emit invalidation events after commit
    IO-->>A: queue:staff-changed
    IO-->>B: queue:staff-changed
```

Prisma may require a parameterized raw query for `FOR UPDATE SKIP LOCKED`. Keep that query inside the ticket repository and cover it with an integration test.

## 11. REST API contract

All responses use JSON. Errors use:

```json
{
  "error": {
    "code": "QUEUE_CLOSED",
    "message": "This queue is currently closed.",
    "requestId": "req_..."
  }
}
```

| Method | Path | Authentication | Purpose |
|---|---|---|---|
| `POST` | `/api/auth/login` | Public | Start staff session |
| `POST` | `/api/auth/logout` | Staff | End staff session |
| `GET` | `/api/auth/me` | Staff | Restore staff session |
| `GET` | `/api/public/queues/:slug` | Public | Public queue name, open state, and count |
| `GET` | `/api/public/queues/:slug/board` | Public | Called/serving display numbers |
| `POST` | `/api/public/queues/:slug/tickets` | Public + rate limit | Join using a client-generated token and idempotency key |
| `GET` | `/api/public/tickets/me` | `X-Ticket-Token` header | Get own ticket, position, and queue summary |
| `POST` | `/api/public/tickets/me/cancel` | `X-Ticket-Token` header | Cancel own waiting ticket |
| `GET` | `/api/staff/queues/:queueId/snapshot` | Staff | Full staff queue state |
| `POST` | `/api/staff/queues/:queueId/open` | Staff | Open queue |
| `POST` | `/api/staff/queues/:queueId/close` | Staff | Stop new joins |
| `POST` | `/api/staff/queues/:queueId/call-next` | Staff + `{ counterId }` | Atomically call oldest waiting ticket and assign its counter |
| `POST` | `/api/staff/tickets/:ticketId/recall` | Staff | Repeat call notification |
| `POST` | `/api/staff/tickets/:ticketId/start` | Staff | Change the called ticket to serving |
| `POST` | `/api/staff/tickets/:ticketId/complete` | Staff | Finish service |
| `POST` | `/api/staff/tickets/:ticketId/no-show` | Staff | Mark called ticket absent |

### Important response shapes

```ts
type CustomerTicketView = {
  displayNumber: string;
  status: "WAITING" | "CALLED" | "SERVING" | "COMPLETED" | "CANCELLED" | "NO_SHOW";
  peopleAhead: number | null;
  counterName: string | null;
  joinedAt: string;
  calledAt: string | null;
  queue: { name: string; isOpen: boolean };
};

type StaffQueueSnapshot = {
  queue: { id: string; name: string; isOpen: boolean };
  waiting: Array<{
    id: string;
    displayNumber: string;
    joinedAt: string;
  }>;
  active: Array<{
    id: string;
    displayNumber: string;
    status: "CALLED" | "SERVING";
    counterName: string | null;
  }>;
};
```

## 12. Socket.IO contract

Sockets notify clients that authoritative state changed. REST remains available for commands and recovery.

### Rooms

| Room | Members | Purpose |
|---|---|---|
| `queue:{queueId}:public` | Kiosk and customers | Public board/count changes |
| `queue:{queueId}:staff` | Authorized staff | Waiting and active list changes |
| `ticket:{ticketId}` | Owner of a validated public token | Private ticket changes |

### Server-to-client events

For the MVP, Socket.IO events are invalidation notifications. When an event arrives, TanStack Query refetches the authoritative REST view. This removes event-order/version ambiguity and makes reconnection recovery straightforward.

| Event | Room | Payload |
|---|---|---|
| `queue:public-changed` | Public | `{ queueId }` |
| `queue:staff-changed` | Staff | `{ queueId }` |
| `ticket:changed` | Ticket | `{ ticketId }` |
| `ticket:recalled` | Ticket | `{ displayNumber, recalledAt }` |

### Connection flow

```mermaid
sequenceDiagram
    participant Client
    participant API
    participant Socket
    participant DB

    Client->>API: Load authoritative snapshot
    API->>DB: Query current state
    DB-->>API: State
    API-->>Client: Snapshot
    Client->>Socket: Connect with staff session or ticket token
    Socket->>DB: Validate access
    Socket-->>Client: Joined authorized room
    Socket-->>Client: ticket:changed / queue:...changed
    Client->>API: Invalidate query and refetch current state
```

Never let a client choose arbitrary room names without server-side authorization.

## 13. Principal workflows

### Customer joins

```mermaid
sequenceDiagram
    actor Customer
    participant Web
    participant API
    participant DB
    participant IO as Socket.IO

    Customer->>Web: Scan kiosk QR
    Web->>API: GET public queue
    API-->>Web: Queue is open
    Customer->>Web: Tap Join queue
    Web->>Web: Generate customer token and join-request ID
    Web->>API: POST ticket with token body<br/>and Idempotency-Key header
    API->>DB: Transaction: increment daily number<br/>and create WAITING ticket
    DB-->>API: Ticket A-001
    API-->>Web: Customer ticket view
    Web->>Web: Keep the raw token in localStorage
    API->>IO: Broadcast updated counts/snapshot
    Web->>IO: Authenticate ticket room
```

### Staff serves a customer

```mermaid
sequenceDiagram
    actor Staff
    participant Dashboard
    participant API
    participant DB
    participant IO as Socket.IO
    participant Customer
    participant Kiosk

    Staff->>Dashboard: Call Next
    Dashboard->>API: POST call-next
    API->>DB: Lock oldest WAITING and set CALLED
    DB-->>API: Updated ticket
    API->>IO: Broadcast updates
    IO-->>Customer: ticket:changed, then REST refetch
    IO-->>Kiosk: queue:public-changed, then REST refetch
    IO-->>Dashboard: queue:staff-changed, then REST refetch
    Staff->>API: POST start
    Staff->>API: POST complete
```

## 14. Business rules and edge cases

- Joining a closed queue returns `409 QUEUE_CLOSED`.
- A customer cannot hold two active tickets for the same queue in the same browser. This is a usability rule, not a complete identity guarantee.
- Before joining, the browser generates a high-entropy customer token and an idempotency key. Retries reuse both values; the server stores only the token hash and `join_request_id`.
- `call-next` returns `409 QUEUE_EMPTY` when no ticket is waiting.
- `call-next` accepts a counter ID, assigns it while changing the ticket to `CALLED`, and returns `409 COUNTER_BUSY` if that counter already has a called/serving ticket.
- A ticket transition with the wrong current state returns `409 INVALID_TICKET_STATE`.
- Closing a queue prevents new joins but does not delete existing tickets.
- The queue date and daily numbering timezone must be configured explicitly; use `Africa/Casablanca` for the initial deployment.
- If Socket.IO disconnects, show a reconnecting indicator and refetch state after reconnection.
- If the customer token is lost, staff can still serve the ticket; the MVP does not attempt identity recovery.
- Do not automatically mark no-show. Staff makes that decision in the MVP.
- Do not delete operational records; use terminal statuses and event history.

## 15. Security baseline

- Hash passwords with Argon2id.
- Put staff authentication in a `Secure`, `HttpOnly`, `SameSite=Lax` cookie.
- Protect state-changing cookie-authenticated routes against CSRF.
- Validate every request with Zod and reject unknown fields.
- Rate-limit login and public ticket creation.
- Generate customer tokens with Web Crypto in the browser and validate their format on the server.
- Store token hashes, never raw customer tokens, in the database.
- Hash high-entropy searchable customer tokens with deterministic SHA-256; password hashing still uses Argon2id.
- Apply authorization on every staff action and Socket.IO room join.
- Do not expose customer tokens, database UUIDs, email addresses, or notes on the kiosk.
- Restrict CORS to configured frontend origins.
- Keep secrets in environment variables and never commit `.env`.
- Log action type and internal IDs, but not passwords or raw tokens.

## 16. Environment variables

```dotenv
NODE_ENV=development
PORT=3000
WEB_ORIGIN=http://localhost:5173
DATABASE_URL=postgresql://queue_user:queue_password@localhost:5432/virtual_queue
AUTH_SECRET=replace-with-a-long-random-secret
PUBLIC_APP_URL=http://localhost:5173
QUEUE_TIMEZONE=Africa/Casablanca
```

The committed `.env.example` contains names and safe examples only.

## 17. Testing strategy

```mermaid
flowchart TB
    E2E[Playwright end-to-end<br/>critical user journeys]
    INT[API and database integration<br/>transactions, auth, concurrency]
    UNIT[Unit and component tests<br/>rules, validation, UI states]
    E2E --> INT --> UNIT
```

### Required tests

- Ticket-state transition rules
- Ticket display-number generation
- Joining an open versus closed queue
- Duplicate join request with the same idempotency key and customer token
- Two simultaneous `call-next` requests select different tickets
- Empty queue behavior
- Customer token cannot access another ticket
- Unauthenticated staff operations are rejected
- Socket room authorization
- Customer page updates after Call Next
- Reloading restores the customer ticket
- Complete customer journey in Playwright

## 18. Observability and error handling

- Return stable machine-readable error codes.
- Use structured server logs with request ID, route, response status, duration, and internal actor ID.
- Include a health endpoint: `GET /health`.
- Include a readiness endpoint that checks PostgreSQL: `GET /ready`.
- Log every ticket transition in `ticket_events`.
- Show friendly customer errors and actionable staff errors.
- Do not treat a Socket disconnect as data loss; PostgreSQL remains authoritative.

## 19. Two-person delivery plan

Both developers review pull requests and understand the full flow. Ownership below indicates the lead, not exclusive knowledge.

```mermaid
gantt
    title Virtual Queue MVP — 6-week plan
    dateFormat  YYYY-MM-DD
    axisFormat  Week %W
    section Together
    Scope, stories, wireframes, contracts :a1, 2026-11-23, 5d
    Integration and demo rehearsals       :a2, after b4, 5d
    section Developer A lead
    Monorepo, database, migrations         :b1, 2026-11-30, 5d
    Ticket API and safe Call Next          :b2, after b1, 5d
    Auth, security, API integration tests  :b3, after b2, 5d
    Reliability and deployment docs        :b4, after b3, 5d
    section Developer B lead
    Routes, UI shell, shared components    :c1, 2026-11-30, 5d
    Customer and kiosk screens             :c2, after c1, 5d
    Staff dashboard and Socket.IO client   :c3, after c2, 5d
    Responsive UI and Playwright tests     :c4, after c3, 5d
```

Dates are a suggested starting schedule and can be shifted while preserving the order.

### Backlog in build order

#### Milestone 0 — Conception

- [ ] Agree on the included and excluded scope
- [ ] Draw low-fidelity wireframes for the five routes
- [ ] Confirm ticket lifecycle and business rules
- [ ] Confirm REST and Socket.IO contracts
- [ ] Create Git repository, project board, and pull-request rules

#### Milestone 1 — Foundation

- [ ] Create npm workspace and shared TypeScript configuration
- [ ] Create React/Vite application
- [ ] Create Express/Socket.IO server
- [ ] Add Docker Compose PostgreSQL service
- [ ] Add Prisma schema, first migration, and seed data
- [ ] Add lint, format, type-check, and test scripts
- [ ] Add CI for lint, type-check, and tests

#### Milestone 2 — Walking skeleton

- [ ] Public queue endpoint
- [ ] Ticket creation with daily display number and hashed private token
- [ ] Basic customer ticket screen
- [ ] Basic staff waiting list
- [ ] Transaction-safe Call Next
- [ ] Socket broadcasts to customer, kiosk, and staff
- [ ] Complete the first end-to-end flow with rough styling

#### Milestone 3 — Complete MVP behavior

- [ ] Staff login and authorization
- [ ] Recall, start, complete, no-show, and customer cancellation
- [ ] Queue open/close control
- [ ] Customer localStorage recovery
- [ ] Socket reconnection and REST resynchronization
- [ ] Kiosk QR and Now Serving board
- [ ] Empty, loading, disconnected, and error UI states

#### Milestone 4 — Quality and delivery

- [ ] Required unit and integration tests
- [ ] Concurrency test with simultaneous staff requests
- [ ] Playwright critical journey
- [ ] Mobile and accessibility review
- [ ] Security checklist review
- [ ] README setup and demonstration instructions
- [ ] Seeded demo account and queue
- [ ] Final demo rehearsal

## 20. Git workflow

- Protect `main`; merge through reviewed pull requests.
- Create short branches such as `feat/ticket-creation` or `fix/call-next-lock`.
- Keep pull requests small enough for the teammate to review the same day.
- Include migration files when the Prisma schema changes.
- Never edit an applied migration; create a new one.
- Before merging, run `lint`, `typecheck`, and `test`.
- Agree on API contracts before independently implementing client and server pieces.

## 21. Definition of Done

A backlog item is done only when:

- Its acceptance criteria pass.
- Server inputs and permissions are validated.
- Loading, empty, error, and success states are handled where relevant.
- Automated tests cover important behavior.
- No secrets or personal data are logged or committed.
- The teammate has reviewed the change.
- Documentation and shared contracts are updated.

The MVP is complete when a fresh machine can follow the README and demonstrate this flow:

```mermaid
flowchart LR
    Setup[Start app and seed database] --> Login[Staff logs in]
    Login --> Open[Staff opens queue]
    Open --> Scan[Customer scans kiosk QR]
    Scan --> Join[Customer receives ticket]
    Join --> Next[Staff calls next]
    Next --> Live[All three screens receive change events and refetch]
    Live --> Serve[Staff starts and completes service]
    Serve --> Persist[History remains after restart]
```

## 22. Frozen realization decisions

To prevent the two developers from implementing incompatible assumptions, use these defaults for the MVP and record them under `docs/decisions/`:

1. Seed one organization, one queue, one counter, and one staff user. Keep the schema ready for more counters.
2. A customer must explicitly tap **Join queue** after scanning; opening a URL never creates a ticket.
3. The browser generates one 32-byte base64url customer token plus one UUID join-request ID per attempt. It reuses them on a retry.
4. Staff authentication uses an eight-hour signed JWT in a `Secure`, `HttpOnly`, `SameSite=Lax` cookie. Logout clears it.
5. PostgreSQL is the only source of truth. REST handles commands and snapshots; Socket.IO only announces changes.
6. A counter can have at most one `CALLED` or `SERVING` ticket. The counter is assigned by **Call Next**.
7. Recall is allowed only for `CALLED`; it creates an event without changing ticket state.
8. No-show is a manual staff action. There is no automatic timer in the MVP.
9. Queue daily numbering uses the `Africa/Casablanca` calendar date, while timestamps are stored in UTC.
10. Frontend and API should be deployed under the same site where possible; the chosen host must support WebSockets and PostgreSQL.

Do not add Redis, subscriptions, custom forms, analytics, or multiple organizations until the walking skeleton and concurrency tests work reliably.

## 23. How to read the realization design

The following class diagrams describe **responsibilities, dependencies, fields, and method contracts**. They do not require every box to become a JavaScript `class`.

- Prisma models are data records, not active-record classes.
- Controllers and services may be plain exported objects or functions.
- Repository and publisher interfaces are useful boundaries for database and Socket.IO code.
- React uses function components and hooks; the frontend diagram is a module/component diagram expressed with Mermaid class notation.
- Create a separate file only when it has a clear responsibility. Do not generate empty files just because a diagram contains a box.

The architecture rule is more important than the `class` keyword:

```mermaid
flowchart LR
    HTTP[HTTP and Socket adapters] --> APP[Application services]
    APP --> PORTS[Repository and publisher interfaces]
    PORTS --> INFRA[Prisma and Socket.IO implementations]
    INFRA --> DB[(PostgreSQL)]
```

Dependencies point from the outside toward business behavior. Services never import controllers or React code.

## 24. Domain class diagram

```mermaid
classDiagram
    direction LR

    class Organization {
        +UUID id
        +string name
        +Date createdAt
        +Date updatedAt
    }

    class User {
        +UUID id
        +UUID organizationId
        +string email
        -string passwordHash
        +UserRole role
        +Date createdAt
        +Date updatedAt
    }

    class Queue {
        +UUID id
        +UUID organizationId
        +string name
        +string slug
        +string prefix
        +boolean isOpen
        +number lastDailyNumber
        +LocalDate numberDate
        +Date createdAt
        +Date updatedAt
    }

    class Counter {
        +UUID id
        +UUID queueId
        +string name
        +boolean isActive
        +Date createdAt
        +Date updatedAt
    }

    class Ticket {
        +UUID id
        +UUID queueId
        +UUID counterId
        -string publicTokenHash
        +string joinRequestId
        +LocalDate serviceDate
        +number sequenceNumber
        +string displayNumber
        +TicketStatus status
        +Date joinedAt
        +Date calledAt
        +Date serviceStartedAt
        +Date finishedAt
        +Date updatedAt
    }

    class TicketEvent {
        +UUID id
        +UUID ticketId
        +UUID actorUserId
        +TicketEventType type
        +Json metadata
        +Date createdAt
    }

    class TicketStatus {
        <<enumeration>>
        WAITING
        CALLED
        SERVING
        COMPLETED
        CANCELLED
        NO_SHOW
    }

    class UserRole {
        <<enumeration>>
        STAFF
    }

    class TicketEventType {
        <<enumeration>>
        CREATED
        CALLED
        RECALLED
        SERVICE_STARTED
        COMPLETED
        CANCELLED
        NO_SHOW
    }

    Organization "1" --> "many" User : employs
    Organization "1" --> "many" Queue : owns
    Queue "1" --> "many" Counter : has
    Queue "1" --> "many" Ticket : contains
    Counter "zero or one" --> "many" Ticket : serves
    Ticket "1" --> "many" TicketEvent : history
    User "zero or one" --> "many" TicketEvent : performs
    Ticket --> TicketStatus
    User --> UserRole
    TicketEvent --> TicketEventType
```

### Domain invariants

- A queue slug is unique.
- `counterId` is `null` while a ticket is waiting.
- `actorUserId` is `null` for customer and system events.
- `(queueId, serviceDate, sequenceNumber)` is unique.
- `(queueId, joinRequestId)` is unique.
- `publicTokenHash` is unique and the raw token is never persisted.
- Only one `CALLED` or `SERVING` ticket may occupy a counter.
- `displayNumber` is presentation data; authorization always uses UUIDs and credentials.
- A state change and its `TicketEvent` are committed in the same database transaction.

## 25. Backend application class diagram

```mermaid
classDiagram
    direction TB

    class AuthController {
        +login(req, res) Promise~void~
        +logout(req, res) void
        +me(req, res) Promise~void~
    }

    class PublicQueueController {
        +getQueue(req, res) Promise~void~
        +getBoard(req, res) Promise~void~
    }

    class PublicTicketController {
        +join(req, res) Promise~void~
        +getOwn(req, res) Promise~void~
        +cancelOwn(req, res) Promise~void~
    }

    class StaffQueueController {
        +getSnapshot(req, res) Promise~void~
        +open(req, res) Promise~void~
        +close(req, res) Promise~void~
        +callNext(req, res) Promise~void~
    }

    class StaffTicketController {
        +recall(req, res) Promise~void~
        +start(req, res) Promise~void~
        +complete(req, res) Promise~void~
        +markNoShow(req, res) Promise~void~
    }

    class AuthService {
        -UserRepository users
        -PasswordHasher passwords
        -SessionSigner sessions
        +login(email, password) Promise~StaffSession~
        +getUser(userId) Promise~StaffUserView~
        +verifySession(token) StaffPrincipal
    }

    class QueueService {
        -QueueRepository queues
        -RealtimePublisher realtime
        +getPublicQueue(slug) Promise~PublicQueueView~
        +getBoard(slug) Promise~PublicBoardView~
        +getStaffSnapshot(queueId, actor) Promise~StaffQueueSnapshot~
        +openQueue(queueId, actor) Promise~QueueView~
        +closeQueue(queueId, actor) Promise~QueueView~
    }

    class TicketService {
        -QueueRepository queues
        -TicketRepository tickets
        -CustomerTokenService tokens
        -TicketStateMachine states
        -RealtimePublisher realtime
        +joinQueue(slug, requestId, token) Promise~CustomerTicketView~
        +getCustomerTicket(token) Promise~CustomerTicketView~
        +cancelTicket(token) Promise~CustomerTicketView~
        +callNext(queueId, counterId, actor) Promise~StaffTicketView~
        +recall(ticketId, actor) Promise~StaffTicketView~
        +startService(ticketId, actor) Promise~StaffTicketView~
        +completeTicket(ticketId, actor) Promise~StaffTicketView~
        +markNoShow(ticketId, actor) Promise~StaffTicketView~
    }

    class TicketStateMachine {
        +canTransition(from, to) boolean
        +assertTransition(from, to) void
        +isTerminal(status) boolean
    }

    class CustomerTokenService {
        +validateFormat(rawToken) void
        +hash(rawToken) string
    }

    class PasswordHasher {
        <<interface>>
        +hash(password) Promise~string~
        +verify(hash, password) Promise~boolean~
    }

    class SessionSigner {
        <<interface>>
        +sign(user) string
        +verify(token) StaffPrincipal
    }

    class QueueRepository {
        <<interface>>
        +findBySlug(slug) Promise~QueueRecord~
        +findById(id) Promise~QueueRecord~
        +setOpenState(id, isOpen) Promise~QueueRecord~
        +getPublicBoard(id) Promise~PublicBoardView~
        +getStaffSnapshot(id) Promise~StaffQueueSnapshot~
        +staffCanManage(userId, queueId) Promise~boolean~
    }

    class TicketRepository {
        <<interface>>
        +createWithDailyNumber(input) Promise~TicketRecord~
        +findByTokenHash(hash) Promise~TicketRecord~
        +getCustomerView(ticketId) Promise~CustomerTicketView~
        +countAhead(ticketId) Promise~number~
        +cancelWaitingByToken(hash) Promise~TicketRecord~
        +callNextAtomic(queueId, counterId, actorId) Promise~TicketRecord~
        +recordRecall(ticketId, actorId) Promise~TicketRecord~
        +transitionAtomic(input) Promise~TicketRecord~
    }

    class UserRepository {
        <<interface>>
        +findByEmail(email) Promise~UserRecord~
        +findById(id) Promise~UserRecord~
    }

    class PrismaQueueRepository {
        -PrismaClient db
    }

    class PrismaTicketRepository {
        -PrismaClient db
    }

    class PrismaUserRepository {
        -PrismaClient db
    }

    class RealtimePublisher {
        <<interface>>
        +queuePublicChanged(queueId) Promise~void~
        +queueStaffChanged(queueId) Promise~void~
        +ticketChanged(ticketId) Promise~void~
        +ticketRecalled(ticketId, recalledAt) Promise~void~
    }

    class SocketRealtimePublisher {
        -SocketServer io
    }

    AuthController --> AuthService
    PublicQueueController --> QueueService
    PublicTicketController --> TicketService
    StaffQueueController --> QueueService
    StaffQueueController --> TicketService
    StaffTicketController --> TicketService

    AuthService --> UserRepository
    AuthService --> PasswordHasher
    AuthService --> SessionSigner
    QueueService --> QueueRepository
    QueueService --> RealtimePublisher
    TicketService --> QueueRepository
    TicketService --> TicketRepository
    TicketService --> CustomerTokenService
    TicketService --> TicketStateMachine
    TicketService --> RealtimePublisher

    PrismaQueueRepository ..|> QueueRepository
    PrismaTicketRepository ..|> TicketRepository
    PrismaUserRepository ..|> UserRepository
    SocketRealtimePublisher ..|> RealtimePublisher
```

### Responsibility rules

- **Controller:** translate HTTP input/output only; no Prisma calls and no queue rules.
- **Service:** enforce permissions, states, orchestration, and error codes.
- **Repository:** own Prisma queries, row locks, and database transactions.
- **Mapper:** convert database records into safe public DTOs.
- **Realtime publisher:** emit only after the repository transaction commits.
- **Middleware:** authenticate and validate before a controller runs.

For the first implementation, `PublicTicketController` and `StaffTicketController` may be exported functions in the same `ticket.controller.ts`. Split them only when the file becomes difficult to navigate.

## 26. Frontend module diagram

React uses functions and hooks rather than long-lived page classes, but the responsibilities can still be shown precisely:

```mermaid
classDiagram
    direction TB

    class HttpClient {
        +get(path, schema) Promise
        +post(path, body, schema) Promise
        +setTicketToken(token) void
    }

    class PublicQueueApi {
        +getQueue(slug) Promise~PublicQueueView~
        +joinQueue(slug, requestId, token) Promise~CustomerTicketView~
    }

    class CustomerTicketApi {
        +getOwnTicket(token) Promise~CustomerTicketView~
        +cancelOwnTicket(token) Promise~CustomerTicketView~
    }

    class StaffApi {
        +login(credentials) Promise~StaffUserView~
        +getSnapshot(queueId) Promise~StaffQueueSnapshot~
        +callNext(queueId, counterId) Promise~StaffTicketView~
        +recall(ticketId) Promise~StaffTicketView~
        +start(ticketId) Promise~StaffTicketView~
        +complete(ticketId) Promise~StaffTicketView~
        +markNoShow(ticketId) Promise~StaffTicketView~
    }

    class SocketClient {
        +connectStaff() void
        +connectTicket(token) void
        +joinPublicQueue(slug) void
        +onChanged(handler) unsubscribe
        +disconnect() void
    }

    class TicketStorage {
        +save(queueSlug, token, requestId) void
        +load(queueSlug) StoredTicket
        +clear(queueSlug) void
    }

    class JoinQueuePage {
        <<React page>>
        +loadQueue() void
        +handleJoin() Promise~void~
    }

    class CustomerTicketPage {
        <<React page>>
        +loadTicket() void
        +handleCancel() Promise~void~
        +handleReconnect() void
    }

    class KioskPage {
        <<React page>>
        +loadBoard() void
        +subscribeToQueue() void
    }

    class StaffDashboardPage {
        <<React page>>
        +loadSnapshot() void
        +handleCallNext(counterId) Promise~void~
        +handleTicketAction(action) Promise~void~
    }

    PublicQueueApi --> HttpClient
    CustomerTicketApi --> HttpClient
    StaffApi --> HttpClient
    JoinQueuePage --> PublicQueueApi
    JoinQueuePage --> TicketStorage
    CustomerTicketPage --> CustomerTicketApi
    CustomerTicketPage --> TicketStorage
    CustomerTicketPage --> SocketClient
    KioskPage --> PublicQueueApi
    KioskPage --> SocketClient
    StaffDashboardPage --> StaffApi
    StaffDashboardPage --> SocketClient
```

TanStack Query owns server-state caching. When Socket.IO announces a change, the relevant query is invalidated:

```text
ticket:changed       → invalidate customer-ticket query
queue:public-changed → invalidate public-board and public-queue queries
queue:staff-changed  → invalidate staff-snapshot query
reconnect            → invalidate every query used by that screen
```

## 27. Method behavior catalogue

| Method | Required behavior | Main errors |
|---|---|---|
| `AuthService.login` | Normalize email, find user, verify Argon2id hash, issue eight-hour cookie claims | `INVALID_CREDENTIALS`, `RATE_LIMITED` |
| `QueueService.getPublicQueue` | Return name, slug, open state, and waiting count only | `QUEUE_NOT_FOUND` |
| `QueueService.getBoard` | Return called/serving display numbers and counter names only | `QUEUE_NOT_FOUND` |
| `QueueService.openQueue` | Verify organization access, set `isOpen=true`, publish public/staff invalidations | `FORBIDDEN`, `QUEUE_NOT_FOUND` |
| `QueueService.closeQueue` | Stop new joins without deleting active tickets | `FORBIDDEN`, `QUEUE_NOT_FOUND` |
| `TicketService.joinQueue` | Validate token/request ID, ensure queue is open, create daily number and event atomically, publish | `QUEUE_CLOSED`, `VALIDATION_ERROR` |
| `TicketService.getCustomerTicket` | Hash token, locate ticket, count tickets ahead, map safe DTO | `TICKET_NOT_FOUND` |
| `TicketService.cancelTicket` | Permit `WAITING → CANCELLED`, add event, publish | `INVALID_TICKET_STATE` |
| `TicketService.callNext` | Check actor/counter, atomically lock FIFO ticket, assign counter, set `CALLED`, add event, publish | `QUEUE_EMPTY`, `COUNTER_BUSY`, `FORBIDDEN` |
| `TicketService.recall` | Require `CALLED`, append `RECALLED`; do not change status | `INVALID_TICKET_STATE` |
| `TicketService.startService` | Permit `CALLED → SERVING`, timestamp and add event | `INVALID_TICKET_STATE` |
| `TicketService.completeTicket` | Permit `SERVING → COMPLETED`, timestamp and add event | `INVALID_TICKET_STATE` |
| `TicketService.markNoShow` | Permit `CALLED → NO_SHOW`, timestamp and add event | `INVALID_TICKET_STATE` |
| `TicketStateMachine.assertTransition` | Reject every transition not present in the allowed-transition table | `INVALID_TICKET_STATE` |
| `CustomerTokenService.hash` | Produce deterministic SHA-256 for lookup; never log input | `VALIDATION_ERROR` |

## 28. Request realization path

Every command follows the same path:

```mermaid
sequenceDiagram
    participant UI as React UI
    participant R as Express Route
    participant M as Middleware
    participant C as Controller
    participant S as Application Service
    participant Repo as Repository
    participant DB as PostgreSQL
    participant RT as Realtime Publisher

    UI->>R: HTTP command
    R->>M: Authenticate and validate
    M->>C: Typed request
    C->>S: Call use-case method
    S->>Repo: Execute query or transaction
    Repo->>DB: Lock, read, and write
    DB-->>Repo: Committed result
    Repo-->>S: Domain result
    S->>RT: Announce change after commit
    S-->>C: Safe response DTO
    C-->>UI: JSON response
    RT-->>UI: Invalidation event
    UI->>R: Refetch authoritative snapshot
```

If Socket.IO emission fails after a commit, the database result stays valid. Connected clients recover by polling manually, refreshing, or refetching after reconnection.

### Use-case traceability matrix

| User action | Route | Service method | Repository method | Event | Main UI |
|---|---|---|---|---|---|
| View queue | `GET /public/queues/:slug` | `getPublicQueue` | `findBySlug` | None | `JoinQueuePage` |
| Join | `POST /public/queues/:slug/tickets` | `joinQueue` | `createWithDailyNumber` | All three change events | `JoinQueuePage` |
| Restore ticket | `GET /public/tickets/me` | `getCustomerTicket` | `findByTokenHash`, `countAhead` | None | `CustomerTicketPage` |
| Cancel | `POST /public/tickets/me/cancel` | `cancelTicket` | `cancelWaitingByToken` | All three change events | `CustomerTicketPage` |
| Staff snapshot | `GET /staff/queues/:id/snapshot` | `getStaffSnapshot` | `getStaffSnapshot` | None | `StaffDashboardPage` |
| Call next | `POST /staff/queues/:id/call-next` | `callNext` | `callNextAtomic` | All three change events | `StaffDashboardPage` |
| Recall | `POST /staff/tickets/:id/recall` | `recall` | `recordRecall` | `ticket:recalled` | `StaffDashboardPage` |
| Start | `POST /staff/tickets/:id/start` | `startService` | `transitionAtomic` | All three change events | `StaffDashboardPage` |
| Complete | `POST /staff/tickets/:id/complete` | `completeTicket` | `transitionAtomic` | All three change events | `StaffDashboardPage` |
| No-show | `POST /staff/tickets/:id/no-show` | `markNoShow` | `transitionAtomic` | All three change events | `StaffDashboardPage` |

## 29. Concrete backend file map

```text
apps/api/src/
├── config/
│   └── env.ts                       # Parse and validate process.env
├── db/
│   └── prisma.ts                    # One PrismaClient instance
├── errors/
│   ├── app-error.ts
│   └── error-codes.ts
├── middleware/
│   ├── error-handler.ts
│   ├── require-staff.ts
│   ├── require-queue-access.ts
│   ├── ticket-token.ts
│   ├── validate.ts
│   ├── request-id.ts
│   ├── origin-check.ts
│   └── rate-limit.ts
├── modules/
│   ├── auth/
│   │   ├── auth.routes.ts
│   │   ├── auth.controller.ts
│   │   ├── auth.service.ts
│   │   ├── auth.repository.ts
│   │   ├── auth.schemas.ts
│   │   ├── password-hasher.ts
│   │   └── session-signer.ts
│   ├── queues/
│   │   ├── queue.routes.ts
│   │   ├── queue.controller.ts
│   │   ├── queue.service.ts
│   │   ├── queue.repository.ts
│   │   ├── queue.mapper.ts
│   │   └── queue.schemas.ts
│   └── tickets/
│       ├── ticket.routes.ts
│       ├── ticket.controller.ts
│       ├── ticket.service.ts
│       ├── ticket.repository.ts
│       ├── ticket.mapper.ts
│       ├── ticket.schemas.ts
│       ├── ticket-state-machine.ts
│       ├── customer-token.service.ts
│       └── call-next.query.ts
├── realtime/
│   ├── socket-server.ts
│   ├── socket-auth.ts
│   ├── room-names.ts
│   └── realtime-publisher.ts
├── app.ts
└── server.ts
```

Start with one file per row above only when its behavior is introduced. In particular, repository files can initially call Prisma directly and later be split into smaller query files.

## 30. Concrete frontend file map

```text
apps/web/src/
├── api/
│   ├── http-client.ts
│   ├── api-error.ts
│   └── query-client.ts
├── components/
│   ├── Button.tsx
│   ├── ErrorMessage.tsx
│   ├── LoadingSpinner.tsx
│   ├── StatusBadge.tsx
│   └── ConnectionIndicator.tsx
├── features/
│   ├── customer/
│   │   ├── api/
│   │   │   ├── get-public-queue.ts
│   │   │   ├── join-queue.ts
│   │   │   ├── get-own-ticket.ts
│   │   │   └── cancel-own-ticket.ts
│   │   ├── components/
│   │   │   ├── JoinQueueCard.tsx
│   │   │   └── CustomerTicketCard.tsx
│   │   ├── hooks/
│   │   │   └── use-customer-ticket.ts
│   │   └── storage/
│   │       └── customer-ticket-storage.ts
│   ├── kiosk/
│   │   ├── api/get-board.ts
│   │   └── components/
│   │       ├── QueueQrCode.tsx
│   │       └── NowServingBoard.tsx
│   └── staff/
│       ├── api/
│       │   ├── auth.ts
│       │   ├── get-staff-snapshot.ts
│       │   └── ticket-actions.ts
│       ├── components/
│       │   ├── LoginForm.tsx
│       │   ├── WaitingList.tsx
│       │   ├── ActiveTicket.tsx
│       │   └── CounterControls.tsx
│       └── routes/ProtectedStaffRoute.tsx
├── pages/
│   ├── JoinQueuePage.tsx
│   ├── CustomerTicketPage.tsx
│   ├── KioskPage.tsx
│   ├── StaffLoginPage.tsx
│   └── StaffDashboardPage.tsx
├── realtime/
│   ├── socket-client.ts
│   ├── use-connection-status.ts
│   ├── use-ticket-events.ts
│   └── use-queue-events.ts
├── routes/router.tsx
├── styles/global.css
└── main.tsx
```

## 31. Detailed realization sequence

Build in vertical slices. A vertical slice completes one small behavior through contracts, backend, database, frontend, and tests:

```mermaid
flowchart LR
    Contract[Shared contract] --> Test[Failing service or integration test]
    Test --> Route[Route and controller]
    Route --> Service[Service and repository]
    Service --> DB[(PostgreSQL)]
    Service --> Client[Frontend API function]
    Client --> Screen[React screen]
    Screen --> Demo[Demonstrable behavior]
```

Do not build the entire backend and then begin the frontend. Integrate a small flow at the end of every slice.

```mermaid
flowchart TD
    S0[0. Freeze behavior and wireframes] --> S1[1. Create runnable workspace]
    S1 --> S2[2. Create and seed database]
    S2 --> S3[3. Define shared contracts]
    S3 --> S4[4. View public queue]
    S4 --> S5[5. Join and restore ticket]
    S5 --> S6[6. Add staff authentication]
    S6 --> S7[7. Staff snapshot and safe Call Next]
    S7 --> S8[8. Add realtime invalidation]
    S8 --> S9[9. Complete ticket lifecycle]
    S9 --> S10[10. Finish kiosk and UX]
    S10 --> S11[11. Reliability and delivery]
```

### Step 0 — Freeze behavior and wireframes

Create:

```text
docs/decisions/001-postgresql-source-of-truth.md
docs/decisions/002-authentication.md
docs/decisions/003-ticket-lifecycle.md
docs/wireframes/join-queue.png
docs/wireframes/customer-ticket.png
docs/wireframes/kiosk.png
docs/wireframes/staff-login.png
docs/wireframes/staff-dashboard.png
docs/api.md
```

Tasks:

- Copy the frozen decisions, endpoints, states, errors, and events into the appropriate documents.
- Sketch the five pages at mobile/desktop sizes as appropriate.
- Make every button correspond to exactly one API command.
- Agree on the counter-busy behavior and demonstration credentials.

Checkpoint: both developers can trace every button from screen to endpoint to allowed state transition.

### Step 1 — Create a runnable workspace

Create the root workspace, React/Vite app, Express app, shared package, Docker Compose PostgreSQL, TypeScript, ESLint, Prettier, and test runners.

Minimum runnable files:

```text
apps/api/src/app.ts
apps/api/src/server.ts
apps/api/src/config/env.ts
apps/web/src/main.tsx
apps/web/src/routes/router.tsx
packages/shared/src/index.ts
```

Add `GET /health`, environment validation, and root scripts for `dev`, `build`, `lint`, `typecheck`, and `test`.

Checkpoint: a fresh clone can start PostgreSQL, API, and web app by following the README; `/health` returns `200`.

### Step 2 — Create and seed the database

Implement Prisma models in dependency order: `Organization`, `User`, `Queue`, `Counter`, `Ticket`, `TicketEvent`. Add the constraints and indexes from the data-model section.

Seed predictably:

```text
Organization: Demo Organization
Queue: General, slug general, prefix A
Counter: Counter 1
Staff: staff@example.com
```

Make the seed idempotent so running it twice does not create duplicates.

Checkpoint: a blank database can be migrated and seeded, and all relationships are visible through a query or Prisma Studio.

### Step 3 — Define shared contracts

Create:

```text
packages/shared/src/types/ticket-status.ts
packages/shared/src/contracts/public-queue.contract.ts
packages/shared/src/contracts/customer-ticket.contract.ts
packages/shared/src/contracts/staff-snapshot.contract.ts
packages/shared/src/contracts/errors.contract.ts
packages/shared/src/events/socket-events.ts
```

If a value crosses HTTP or Socket.IO, define its public shape here. Never send Prisma records directly to browsers.

Checkpoint: both apps import the same contracts, and private fields such as `passwordHash` and `publicTokenHash` are absent.

### Step 4 — Vertical slice: view a public queue

Implement `GET /api/public/queues/:slug`, the queue mapper, frontend query, and `/q/:queueSlug` page.

Acceptance checks:

- Existing slug returns queue name, slug, open state, and waiting count.
- Unknown slug returns `404 QUEUE_NOT_FOUND`.
- Closed queues disable the Join button.
- No internal UUID or private data is returned.

Checkpoint: `/q/general` renders data loaded from PostgreSQL.

### Step 5 — Vertical slice: join and restore a ticket

Implement `POST /api/public/queues/:slug/tickets`, `GET /api/public/tickets/me`, customer token storage, and `/ticket`.

Join sequence:

1. Browser creates a 32-byte base64url token and UUID request ID.
2. Browser stores both before sending the request.
3. Browser sends the token in the body and request ID as `Idempotency-Key`.
4. Server validates both and hashes the token with SHA-256.
5. Transaction locks the queue row, resolves the local service date, resets/increments the daily number, inserts ticket and event, then commits.
6. Server returns a safe `CustomerTicketView`.
7. Browser navigates to `/ticket` and uses `X-Ticket-Token` for future requests.

Acceptance checks:

- Joining an open queue produces `A-001` with `WAITING`.
- A closed queue returns `409 QUEUE_CLOSED`.
- Retrying the same request ID and token returns the existing ticket without duplication.
- Two new requests receive consecutive unique numbers.
- Only the token hash appears in PostgreSQL.
- Reload restores the ticket from localStorage.

Checkpoint: join, see `A-001`, reload, and see the same ticket.

### Step 6 — Vertical slice: staff authentication

Implement login, logout, current-user endpoint, Argon2id verification, JWT cookie, staff middleware, protected React route, and login page.

Acceptance checks:

- Correct credentials set an HttpOnly cookie.
- Incorrect credentials return a generic error.
- Anonymous requests cannot access staff routes.
- Staff from another organization cannot manage the queue.
- Logout removes access and reload restores a valid session.
- Login is rate-limited.

Checkpoint: staff can enter the protected dashboard shell only after login.

### Step 7 — Vertical slice: snapshot and concurrency-safe Call Next

Implement:

```http
GET  /api/staff/queues/:queueId/snapshot
POST /api/staff/queues/:queueId/call-next
Content-Type: application/json

{ "counterId": "..." }
```

The transaction must check that the counter is active/free, lock the oldest waiting ticket with `FOR UPDATE SKIP LOCKED`, assign the counter, set `CALLED`, and insert its event before committing.

Acceptance checks:

- Oldest ticket is called first using `(joinedAt, id)` ordering.
- Empty queue returns `409 QUEUE_EMPTY`.
- Busy counter returns `409 COUNTER_BUSY`.
- A called ticket is never called as new again.
- Two simultaneous requests using different counters receive different tickets.
- Refreshing the dashboard restores database state.

Checkpoint: create three tickets and serve them in FIFO order. The simultaneous-call integration test is a gate for continuing.

### Step 8 — Vertical slice: real-time invalidation

Implement server rooms, socket authentication, publisher, client hooks, connection indicator, and query invalidation.

Rules:

1. Commit PostgreSQL first.
2. Emit change events after a successful commit.
3. Validate every room join on the server.
4. Refetch REST state when an event arrives.
5. Refetch all screen queries after reconnection.

Acceptance checks:

- Customer receives only their private ticket changes.
- Kiosk receives only public queue changes.
- Authorized staff receives staff invalidations.
- A ticket token cannot join another ticket room.
- Failed transactions emit nothing.
- Reconnection restores current state.

Checkpoint: show the customer, kiosk, and staff pages side by side; Call Next updates all three without a manual refresh.

### Step 9 — Vertical slice: complete the lifecycle

Implement cancellation, recall, start, complete, no-show, and queue open/close one at a time. For every transition: validate actor, validate current state, update ticket and event transactionally, commit, publish, return mapped DTO.

| Action | Valid state | Result |
|---|---|---|
| Cancel | `WAITING` | `CANCELLED` |
| Recall | `CALLED` | `RECALLED` event; remains `CALLED` |
| Start | `CALLED` | `SERVING` |
| Complete | `SERVING` | `COMPLETED` |
| No-show | `CALLED` | `NO_SHOW` |

Checkpoint: complete one normal journey, one cancellation, and one no-show; tests reject every invalid transition.

### Step 10 — Finish kiosk and user experience

Add the QR code `${PUBLIC_APP_URL}/q/general`, large called numbers, counter name, full-screen kiosk layout, mobile customer layout, loading/empty/error/terminal states, pending-button disabling, accessibility labels, keyboard support, and visible connection state.

Checkpoint: QR opens the correct join page; the kiosk exposes no UUIDs, tokens, email, or staff data; the customer flow is usable at mobile width.

### Step 11 — Reliability and delivery

Finish unit, integration, Socket authorization, and Playwright tests. Add request IDs, structured logs, `/ready`, production builds, clean-database migration test, restart test, environment documentation, and demo instructions.

```mermaid
flowchart LR
    Clone --> Install
    Install --> StartDB[Start PostgreSQL]
    StartDB --> Migrate
    Migrate --> Seed
    Seed --> Run[Run web and API]
    Run --> Demo[Complete queue journey]
    Demo --> Restart[Restart server]
    Restart --> Verify[Verify persisted state]
```

Checkpoint: a teammate can clone the repository and complete the entire demonstration using only the README.

## 32. Coding recipes for difficult behavior

### Ticket state machine

```ts
const allowedTransitions: Record<TicketStatus, readonly TicketStatus[]> = {
  WAITING: ["CALLED", "CANCELLED"],
  CALLED: ["SERVING", "NO_SHOW"],
  SERVING: ["COMPLETED"],
  COMPLETED: [],
  CANCELLED: [],
  NO_SHOW: [],
};

export class TicketStateMachine {
  canTransition(from: TicketStatus, to: TicketStatus): boolean {
    return allowedTransitions[from].includes(to);
  }

  assertTransition(from: TicketStatus, to: TicketStatus): void {
    if (!this.canTransition(from, to)) {
      throw new AppError("INVALID_TICKET_STATE", 409);
    }
  }

  isTerminal(status: TicketStatus): boolean {
    return allowedTransitions[status].length === 0;
  }
}
```

Recall is intentionally absent because it is an event, not a state transition.

### Idempotent ticket creation transaction

```ts
async function createWithDailyNumber(input: JoinInput): Promise<TicketRecord> {
  return db.$transaction(async (tx) => {
    const replay = await tx.ticket.findUnique({
      where: {
        queueId_joinRequestId: {
          queueId: input.queueId,
          joinRequestId: input.joinRequestId,
        },
      },
    });
    if (replay) return replay;

    const queue = await lockQueueRow(tx, input.queueId);
    if (!queue.isOpen) throw new AppError("QUEUE_CLOSED", 409);

    const serviceDate = dateInQueueTimezone("Africa/Casablanca");
    const sequenceNumber = nextDailyNumber(queue, serviceDate);
    await saveDailyNumber(tx, queue.id, serviceDate, sequenceNumber);

    const ticket = await tx.ticket.create({
      data: {
        queueId: queue.id,
        publicTokenHash: input.publicTokenHash,
        joinRequestId: input.joinRequestId,
        serviceDate,
        sequenceNumber,
        displayNumber: `${queue.prefix}-${String(sequenceNumber).padStart(3, "0")}`,
        status: "WAITING",
      },
    });

    await tx.ticketEvent.create({
      data: { ticketId: ticket.id, type: "CREATED" },
    });
    return ticket;
  });
}
```

`lockQueueRow`, `dateInQueueTimezone`, and `saveDailyNumber` are small helpers whose behavior must have integration/unit tests. The actual raw SQL identifiers must match the Prisma migration.

### Atomic Call Next transaction

```ts
async function callNextAtomic(
  queueId: string,
  counterId: string,
  actorId: string,
): Promise<TicketRecord> {
  return db.$transaction(async (tx) => {
    await assertCounterAvailable(tx, queueId, counterId);

    const rows = await tx.$queryRaw<TicketRecord[]>`
      SELECT *
      FROM "Ticket"
      WHERE "queueId" = ${queueId}
        AND "status" = 'WAITING'
      ORDER BY "joinedAt", "id"
      FOR UPDATE SKIP LOCKED
      LIMIT 1
    `;

    const next = rows[0];
    if (!next) throw new AppError("QUEUE_EMPTY", 409);

    const updated = await tx.ticket.update({
      where: { id: next.id },
      data: {
        status: "CALLED",
        counterId,
        calledAt: new Date(),
      },
    });

    await tx.ticketEvent.create({
      data: {
        ticketId: updated.id,
        actorUserId: actorId,
        type: "CALLED",
        metadata: { counterId },
      },
    });
    return updated;
  });
}
```

The service calls the realtime publisher only after `callNextAtomic` resolves successfully.

### Generic guarded transition

```ts
async function transitionAtomic(input: TransitionInput): Promise<TicketRecord> {
  return db.$transaction(async (tx) => {
    const result = await tx.ticket.updateMany({
      where: { id: input.ticketId, status: input.expectedStatus },
      data: {
        status: input.nextStatus,
        ...input.timestamps,
      },
    });

    if (result.count !== 1) {
      throw new AppError("INVALID_TICKET_STATE", 409);
    }

    await tx.ticketEvent.create({
      data: {
        ticketId: input.ticketId,
        actorUserId: input.actorId,
        type: input.eventType,
      },
    });

    return tx.ticket.findUniqueOrThrow({ where: { id: input.ticketId } });
  });
}
```

The conditional `expectedStatus` prevents two simultaneous actions from both succeeding.

## 33. Route authorization matrix

| Route group | Validation | Credential | Organization check | Rate limit |
|---|---|---|---|---|
| `POST /auth/login` | Body schema | None | No | Strict |
| Public queue reads | Params schema | None | No | General |
| Public ticket join | Params, body, idempotency header | None | No | Strict per IP/queue |
| Customer ticket read/cancel | Token-header format | Customer token | Ticket ownership by token hash | General |
| Staff queue reads/actions | Params/body schema | Staff JWT cookie | `user.organizationId == queue.organizationId` | General |
| Staff ticket actions | Params/body schema | Staff JWT cookie | Ticket queue belongs to staff organization | General |
| Public Socket room | Queue identifier | None | Public room only | Connection limit |
| Customer Socket room | Handshake token | Customer token | Exact ticket match | Connection limit |
| Staff Socket room | Cookie handshake | Staff JWT | Queue organization match | Connection limit |

Every state-changing cookie-authenticated request validates `Origin`. If frontend and API require cross-site cookies in deployment, add an explicit CSRF-token mechanism before release.

## 34. Standard error contract

```ts
type ApiError = {
  error: {
    code:
      | "VALIDATION_ERROR"
      | "UNAUTHENTICATED"
      | "FORBIDDEN"
      | "INVALID_CREDENTIALS"
      | "QUEUE_NOT_FOUND"
      | "QUEUE_CLOSED"
      | "QUEUE_EMPTY"
      | "COUNTER_BUSY"
      | "TICKET_NOT_FOUND"
      | "INVALID_TICKET_STATE"
      | "RATE_LIMITED"
      | "INTERNAL_ERROR";
    message: string;
    requestId: string;
    details?: Record<string, string[]>;
  };
};
```

Use `201` for new ticket creation, `200` for queries/actions/idempotent replay, `401` for missing or invalid credentials, `403` for wrong organization, `404` for absent resources, `409` for business conflicts, `422` for validation, and `429` for rate limiting.

## 35. Two-developer realization workflow

Split leadership by vertical slice rather than permanently assigning one person to all frontend work and the other to all backend work.

| Slice | Developer A lead | Developer B lead | Complete together |
|---|---|---|---|
| Foundation | API and database setup | Web and shared-package setup | Root scripts and README |
| Public queue | Backend endpoint | Join page | Contract and acceptance test |
| Join ticket | Transaction and tokens | Customer flow and storage | Integration test |
| Authentication | Auth service/middleware | Login and protected route | Permission review |
| Call Next | Locking query | Staff dashboard | Concurrency demonstration |
| Real time | Socket server | Socket client | Three-screen test |
| Lifecycle | Transition service | Staff/customer controls | Invalid-state tests |
| Delivery | API integration tests | Playwright/accessibility | Fresh-clone rehearsal |

```mermaid
flowchart LR
    Agree[Agree on contract] --> Implement[Implement independently]
    Implement --> PR[Open small pull request]
    PR --> Review[Teammate reviews]
    Review --> Run[Run the slice together]
    Run --> Merge[Merge after checkpoint passes]
```

For each feature, create or update artifacts in this order:

1. User story and acceptance criteria
2. Shared request/response schema
3. Error and event types
4. Failing service or integration test
5. Repository and service method
6. Controller, validation, and route
7. Frontend API function
8. React component or page
9. Realtime publisher/handler where required
10. End-to-end acceptance test
11. Documentation

## 36. Definition of Ready and realization gates

### Definition of Ready

A task enters implementation only when:

- Its user story and testable acceptance criteria exist.
- Valid and invalid ticket states are known.
- Request, response, errors, and authorization are defined.
- Its required predecessor slice is complete.
- A developer and reviewer are assigned.

### Definition of Done for each slice

- User-visible behavior works through all required layers.
- The server validates input, authentication, and authorization.
- Database writes are transactional where required.
- Success and important failure paths have automated tests.
- Loading, empty, error, disabled, and retry states are handled.
- Reload/reconnection recovery works where relevant.
- Contracts and documentation match the code.
- `lint`, `typecheck`, `test`, and `build` pass.
- The teammate reviewed it.
- Its checkpoint can be demonstrated without manually editing the database.

### Do not expand the product until all gates pass

- Full customer journey works.
- Simultaneous Call Next test passes.
- Three screens update live.
- Invalid ticket transitions are rejected.
- Staff HTTP routes and Socket rooms are protected.
- Customer tokens do not appear in URLs, logs, or database plaintext.
- Server restart preserves queue state.
- Fresh clone setup and demonstration work from the README.
