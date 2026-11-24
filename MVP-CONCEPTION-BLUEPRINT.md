# Virtual Queue MVP — Conception and Implementation Blueprint

This document is the shared source of truth for the first working version of the virtual queue platform. It is deliberately limited to an MVP that two developers can complete and demonstrate reliably.

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
- Reconnection and ticket recovery using a private public token
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
| US-01 | As a customer, I can scan a QR code and join without an account. | A valid open queue creates exactly one ticket and returns its private token. |
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
| `/ticket/:publicToken` | Customer live ticket | Token holder |
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
        uuid counter_id FK
        string public_token_hash UK
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
        uuid actor_user_id FK
        string type
        json metadata
        datetime created_at
    }
```

### Database rules

- Unique constraint: `(queue_id, number_date, sequence_number)` or store a separate daily counter table.
- Index tickets on `(queue_id, status, joined_at)`.
- Store only a hash of the customer public token; return the raw token once.
- Never derive authority from `display_number`; use UUIDs internally.
- Record every meaningful transition in `ticket_events`.
- Use UTC timestamps in storage and localize only in the UI.

## 10. Concurrency-safe Call Next

Two staff members may click at the same time. The selection and update must happen in one database transaction using a locked row.

```mermaid
sequenceDiagram
    actor A as Staff A
    actor B as Staff B
    participant API
    participant DB as PostgreSQL
    participant IO as Socket.IO

    par Simultaneous requests
        A->>API: POST /queues/:id/call-next
        B->>API: POST /queues/:id/call-next
    end
    API->>DB: BEGIN; select oldest WAITING<br/>FOR UPDATE SKIP LOCKED
    DB-->>API: Ticket A-001
    API->>DB: Set A-001 CALLED; COMMIT
    API->>DB: BEGIN; select oldest unlocked WAITING
    DB-->>API: Ticket A-002
    API->>DB: Set A-002 CALLED; COMMIT
    API->>IO: Emit queue and ticket updates
    IO-->>A: Updated snapshot
    IO-->>B: Updated snapshot
```

Prisma may require a parameterized raw query for `FOR UPDATE SKIP LOCKED`. Keep that query inside the ticket service and cover it with an integration test.

## 11. REST API contract

All responses use JSON. Errors use:

```json
{
  "error": {
    "code": "QUEUE_CLOSED",
    "message": "This queue is currently closed."
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
| `POST` | `/api/public/queues/:slug/tickets` | Public + rate limit | Join and receive raw public token |
| `GET` | `/api/public/tickets/:token` | Token | Get own ticket, position, and queue summary |
| `POST` | `/api/public/tickets/:token/cancel` | Token | Cancel own waiting ticket |
| `GET` | `/api/staff/queues/:queueId/snapshot` | Staff | Full staff queue state |
| `POST` | `/api/staff/queues/:queueId/open` | Staff | Open queue |
| `POST` | `/api/staff/queues/:queueId/close` | Staff | Stop new joins |
| `POST` | `/api/staff/queues/:queueId/call-next` | Staff | Atomically call oldest waiting ticket |
| `POST` | `/api/staff/tickets/:ticketId/recall` | Staff | Repeat call notification |
| `POST` | `/api/staff/tickets/:ticketId/start` | Staff | Mark serving and assign counter |
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

| Event | Room | Payload |
|---|---|---|
| `queue:summary-updated` | Public | `{ waitingCount, calledTickets }` |
| `queue:snapshot-updated` | Staff | `StaffQueueSnapshot` |
| `ticket:updated` | Ticket | `CustomerTicketView` |
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
    Socket-->>Client: ticket:updated / queue:...updated
    Client->>API: Refetch if event version is newer
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
    Web->>API: POST ticket with idempotency key
    API->>DB: Transaction: increment daily number<br/>and create WAITING ticket
    DB-->>API: Ticket A-001
    API-->>Web: Raw public token and ticket view
    Web->>Web: Store token in localStorage
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
    IO-->>Customer: ticket:updated
    IO-->>Kiosk: queue:summary-updated
    IO-->>Dashboard: queue:snapshot-updated
    Staff->>API: POST start
    Staff->>API: POST complete
```

## 14. Business rules and edge cases

- Joining a closed queue returns `409 QUEUE_CLOSED`.
- A customer cannot hold two active tickets for the same queue in the same browser. This is a usability rule, not a complete identity guarantee.
- Ticket creation accepts an idempotency key to prevent duplicates caused by double taps or retries.
- `call-next` returns `409 QUEUE_EMPTY` when no ticket is waiting.
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
- Generate customer tokens with a cryptographically secure random generator.
- Store token hashes, never raw customer tokens, in the database.
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
- Duplicate join request with the same idempotency key
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
- [ ] Ticket creation with daily display number and private token
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
    Next --> Live[All three screens update live]
    Live --> Serve[Staff starts and completes service]
    Serve --> Persist[History remains after restart]
```

## 22. Decisions required before coding

Record the answers in short files under `docs/decisions/`:

1. Will a staff session use a server session store or a signed short-lived JWT cookie?
2. Does the MVP need multiple counters, or will it seed one counter initially?
3. What are the exact recall and no-show rules?
4. Does a customer explicitly tap **Join**, or should opening the QR URL create a ticket immediately? Explicit confirmation is safer for avoiding accidental tickets.
5. Where will the demonstration be deployed, and does that environment support WebSockets and managed PostgreSQL?

Do not add Redis, subscriptions, custom forms, or analytics until the walking skeleton and concurrency tests work reliably.
