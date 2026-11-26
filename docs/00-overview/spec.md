# Project Specification

## 1. Tech Stack

### Backend

- **Runtime & Language**: Node.js 22 LTS + TypeScript
- **Framework**: NestJS (REST API)
- **ORM & Migrations**: Prisma ORM
- **Validation**: `class-validator` + `class-transformer`
- **Testing**: Jest

### Frontend

- **Framework & Build Tool**: React + TypeScript + Vite
- **Routing**: React Router (permission-guarded routes via
  `<PermissionGuard />`)
- **State & Data Fetching**: Redux Toolkit (`@reduxjs/toolkit`, `react-redux`) +
  `axios`
- **Styling & UI**: Tailwind CSS

### Database

- **Primary Database**: PostgreSQL 16
- **Local Runtime**: Docker Compose (`postgres:16-alpine`)

### Auth

- **Mechanism**: Stateless JWT Access Token + HTTP-only Refresh Token
- **Password Hashing**: `bcrypt`
- **Authorization (RBAC)**: Permission-based RBAC (`ROLE_PERMISSIONS` mapping
  `CUSTOMER`, `STAFF`, `ADMIN` to atomic permissions)

## 2. Project Structure & Conventions

### 2.1 Repository Layout

```text
booking/
├── backend/                # NestJS REST API
│   ├── prisma/             # Schema, migrations, and seed
│   ├── src/
│   │   ├── common/         # Shared guards, decorators, filters
│   │   └── modules/        # Domain feature modules
│   ├── test/               # E2E tests
│   └── .env.example
├── frontend/               # React + TypeScript + Vite
│   ├── src/
│   │   ├── api/            # HTTP client & interceptors
│   │   ├── app/            # Redux store setup
│   │   ├── components/     # Shared UI components & layouts
│   │   ├── features/       # Domain feature slices & pages
│   │   └── routes/         # Routing & permission guards
│   └── .env.example
├── docs/                   # Project documentation & specs
├── docker-compose.yml      # Local multi-container orchestration
├── .env.example            # Root Docker Compose variables
├── .gitignore
└── README.md
```

### 2.2 Formatting & Conventions

- **Commit messages**: Max 72 characters per line; include issue reference
  `Refs #<issue-number>` (e.g., `Refs #123`).
- **Code & Markdown (`.ts`, `.js`, `.md`)**: Compatible with default VSCode
  Prettier extension.
- **TypeScript**: Enable `strict: true` in `tsconfig.json`.

## 3. Domain Workflows & RBAC Matrix

### 3.1 Intent, Scope & Business Rules

- **Goal**: Build a full-stack service-booking platform supporting customer
  discovery and reservation, staff catalog/schedule/booking operations, admin
  user governance, overbooking prevention, cancellation fee calculation, and
  operational reporting.
- **Non-Goals**:
  - Multi-night stays or arbitrary date-range bookings (e.g., hotel rooms).
  - Online payment gateway integration, invoicing, or multi-currency pricing.
  - Dynamic multi-tier cancellation policy administration.
- **Anti-Goals**:
  - Overbooking a schedule slot under concurrent booking requests.
  - Exposing another customer's bookings or permitting unauthorized actions.
  - Falling below `60%` unit-test coverage on backend services.
- **Bookable Entities (`PROVIDER` / `CATEGORY` 1:N `SERVICE` 1:N `SCHEDULE` 1:N
  `BOOKING`)**:
  - `Provider`: Vendor or facility with unique name, optional contact info, and
    active status; owns zero or more services.
  - `Category`: Service classification with unique `name`, unique `slug`
    (auto-derived from `name` and updated when `name` changes), and active
    status; categorizes zero or more services.
  - `Service`: Bookable offering belonging to one `Provider` and one `Category`,
    with name (unique per provider), optional description, `duration_minutes`
    (`> 0`, session length for catalog display and default schedule duration;
    edits apply to newly created schedules only), unit `price` (integer VND),
    default `capacity` (`>= 1`), `booking_mode`, and active status.
  - `Schedule`: Pre-created discrete time window (`start_time`–`end_time`)
    belonging to one `Service`, with `capacity` (`>= 1`, defaulting from
    `Service.capacity`), `booked_count` (`0..capacity`), and `is_active`.
    Availability state is derived: `DISABLED` if inactive, `FULL` if
    `booked_count = capacity`, otherwise `OPEN`.
  - `Booking`: Reservation linking one `CUSTOMER` and one `Schedule` (uniquely
    resolving `Service` and `Provider`), with requested `quantity` (`>= 1`),
    `is_exclusive`, `reserved_seats`, and snapshotted VND pricing.
- **Booking Modes & Exclusivity (`Service.booking_mode` +
  `Booking.is_exclusive`)**:
  - **`SHARED_ONLY`** (e.g., `Gym theo buổi`): Shared seat bookings only. Reject
    exclusive requests (`400`). Requires `booked_count + quantity <= capacity`;
    reserves `reserved_seats = quantity`; `total_price = unit_price * quantity`.
  - **`EXCLUSIVE_ONLY`** (e.g., private court or room): Private slot only
    (defaults exclusive; reject non-exclusive with `400`). Requires empty slot
    (`booked_count = 0`) and `1 <= quantity <= capacity`; reserves full
    `capacity`; flat pricing `total_price = unit_price`.
  - **`BOTH`** (e.g., `Spa 60 phút` 4-bed room): Supports shared bookings
    (`SHARED_ONLY` rules) and private buyout (requires `booked_count = 0` and
    `1 <= quantity <= capacity`, reserves full `capacity`, buyout pricing
    `total_price = unit_price * capacity`).
- **Overbooking Prevention**:
  - Booking creation requires an active, upcoming `Schedule`
    (`start_time > now`), active `Category`, `Service`, and `Provider`, and
    sufficient capacity for the selected booking mode; otherwise reject (`409`
    for capacity conflicts, `400` for past or inactive targets).
  - Seat reservation on creation (`PENDING`) and release on rejection or
    cancellation execute atomically so concurrent requests can never exceed
    `Schedule.capacity`.
- **Schedule & Catalog Mutation Rules**:
  - **Deactivation (`is_active = false`)**: Blocks _new_ bookings only; existing
    bookings remain intact and can still be processed by `STAFF`/`ADMIN` (or
    cancelled by `CUSTOMER` before `start_time`).
  - **Schedule Edits**: Block editing `start_time` or `end_time` (`409`) if any
    booking (any status) references the schedule. Block editing `capacity`
    (`409`) if any exclusive seat-holding booking (`PENDING`, `CONFIRMED`, or
    `COMPLETED`) exists. Otherwise reject `capacity < booked_count` (`400`).
  - **Service `booking_mode` Edits**: Apply to new bookings only; existing
    bookings remain consistent via snapshotted `is_exclusive` and
    `reserved_seats`.
  - **Immutable Ownership**: `Service.provider_id` and `Schedule.service_id`
    cannot be changed after creation (deactivate the old record and create a new
    one instead); `Service.category_id` remains editable.
  - **Schedule Overlap & Default Duration**: Parallel overlapping schedules
    under the same `Service` are allowed (e.g., multiple rooms or courts). When
    `end_time` is omitted on creation, it defaults to
    `start_time + service.duration_minutes`; explicit `end_time > start_time` is
    accepted.
- **Cancellation Policy (Fixed System-Wide Rules)**:
  - Customer cancels `PENDING`: `0%` fee.
  - Customer cancels `CONFIRMED` `>= 24 hours` before `start_time`: `0%` fee.
  - Customer cancels `CONFIRMED` `< 24 hours` before `start_time`: `50%` of
    `total_price` (rounded to the nearest integer VND).
  - Past `start_time`: Customer cancellation blocked (`400`).
  - `STAFF`/`ADMIN` rejection or cancellation: `0%` customer fee.
- **Staff State Transition Timing**:
  - `STAFF`/`ADMIN` transitions (`confirm`, `reject`, `cancel`, `complete`) have
    no wall-clock restriction.

### 3.2 Booking Lifecycle States

```mermaid
stateDiagram-v2
  [*] --> PENDING : Customer books (reserves seats)
  PENDING --> CONFIRMED : Staff/Admin confirms
  PENDING --> REJECTED : Staff/Admin rejects (0% fee, releases seats)
  PENDING --> CANCELLED : Customer or Staff/Admin cancels (0% fee)
  CONFIRMED --> COMPLETED : Staff/Admin completes
  CONFIRMED --> CANCELLED : Customer (policy fee) or Staff (0% fee) cancels
  REJECTED --> [*]
  CANCELLED --> [*]
  COMPLETED --> [*]
```

- **Terminal States**: `COMPLETED`, `CANCELLED`, `REJECTED` (further transitions
  return `409 Conflict`).
- **Seat Reservation**: `reserved_seats` is counted in `Schedule.booked_count`
  immediately upon entering `PENDING` and released upon `REJECTED` or
  `CANCELLED`.

### 3.3 Role Workflows

- **`CUSTOMER`**:
  - Register, login, refresh session, logout, and view own profile.
  - Browse, search, and filter active services (keyword, category, price range,
    provider, `booking_mode`); view service details and upcoming slots with
    remaining capacity (`capacity - booked_count`).
  - Create shared or exclusive bookings without exceeding remaining capacity.
  - View personal booking history (filterable by status and date).
  - Cancel own `PENDING` or `CONFIRMED` bookings subject to the fee policy.
- **`STAFF`**:
  - Manage providers, categories, services, and discrete service schedules (CRUD
    and enable/disable).
  - View all bookings and process state transitions (`confirm`, `reject`,
    `cancel`, `complete`).
  - View operational reports (detailed specification deferred).
- **`ADMIN`**:
  - Inherits all `STAFF` capabilities.
  - Manage user accounts: list users, update roles (`CUSTOMER`, `STAFF`,
    `ADMIN`), and activate/deactivate accounts (cannot self-deactivate or demote
    the last active `ADMIN`; deactivated accounts immediately lose API access
    and have active refresh sessions revoked).

### 3.4 Permission Matrix (`ROLE_PERMISSIONS`)

| Permission           | `Public` | `CUSTOMER` | `STAFF` | `ADMIN` |
| :------------------- | :------: | :--------: | :-----: | :-----: |
| `service:read`       |   Yes    |    Yes     |   Yes   |   Yes   |
| `service:manage`     |    No    |     No     |   Yes   |   Yes   |
| `provider:read`      |   Yes    |    Yes     |   Yes   |   Yes   |
| `provider:manage`    |    No    |     No     |   Yes   |   Yes   |
| `schedule:read`      |   Yes    |    Yes     |   Yes   |   Yes   |
| `schedule:manage`    |    No    |     No     |   Yes   |   Yes   |
| `booking:create`     |    No    |    Yes     |   No    |   No    |
| `booking:read_own`   |    No    |    Yes     |   No    |   No    |
| `booking:cancel_own` |    No    |    Yes     |   No    |   No    |
| `booking:read_all`   |    No    |     No     |   Yes   |   Yes   |
| `booking:process`    |    No    |     No     |   Yes   |   Yes   |
| `report:read`        |    No    |     No     |   Yes   |   Yes   |
| `user:manage`        |    No    |     No     |   No    |   Yes   |

## 4. V1 Data Model & Auth/User API Contract

### 4.1 Capacity & Concurrency Decision

`Schedule` stores `booked_count` for fast availability reads without aggregating
bookings on every listing query. All seat reservations, releases, and schedule
capacity/time edits execute transactionally under a pessimistic row lock on the
target `Schedule` to guarantee zero overbooking.

### 4.2 Logical Entity Relationship Diagram

This diagram defines the target V1 domain model across all 7 entities. Schema
migrations roll out incrementally by feature:

1. **Auth & Users**: `USER`, `REFRESH_TOKEN`
2. **Catalog**: `PROVIDER`, `CATEGORY`, `SERVICE`
3. **Scheduling & Booking**: `SCHEDULE`, `BOOKING`

```mermaid
erDiagram
  USER ||--o{ REFRESH_TOKEN : owns
  USER ||--o{ BOOKING : places
  USER |o--o{ BOOKING : processes
  USER |o--o{ BOOKING : cancels
  PROVIDER ||--o{ SERVICE : offers
  CATEGORY ||--o{ SERVICE : categorizes
  SERVICE ||--o{ SCHEDULE : has
  SCHEDULE ||--o{ BOOKING : "reserved by"

  USER {
    id id PK
    string email UK
    string full_name
    enum role "CUSTOMER | STAFF | ADMIN"
    boolean is_active
  }

  REFRESH_TOKEN {
    id id PK
    id user_id FK
    datetime expires_at
    datetime revoked_at "nullable"
  }

  PROVIDER {
    id id PK
    string name UK
    string address "nullable"
    string phone "nullable"
    boolean is_active
  }

  CATEGORY {
    id id PK
    string name UK
    string slug UK "derived from name"
    boolean is_active
  }

  SERVICE {
    id id PK
    id provider_id FK
    id category_id FK
    string name "unique per provider"
    string description "nullable"
    int duration_minutes "> 0"
    int price "VND >= 0"
    int capacity "default >= 1"
    enum booking_mode "SHARED_ONLY | EXCLUSIVE_ONLY | BOTH"
    boolean is_active
  }

  SCHEDULE {
    id id PK
    id service_id FK
    datetime start_time
    datetime end_time "> start_time"
    int capacity "defaulted from SERVICE >= 1"
    int booked_count "0..capacity"
    boolean is_active "availability derived: OPEN | FULL | DISABLED"
  }

  BOOKING {
    id id PK
    id customer_id FK
    id schedule_id FK
    id processed_by_id FK "nullable, last STAFF/ADMIN processor"
    id cancelled_by_id FK "nullable, cancelling USER"
    int quantity "guests/seats >= 1"
    boolean is_exclusive "true if EXCLUSIVE_ONLY or buyout"
    int reserved_seats "capacity if exclusive else quantity"
    enum status "PENDING | CONFIRMED | REJECTED | CANCELLED | COMPLETED"
    int unit_price "snapshot SERVICE.price (VND)"
    int total_price "snapshot total (VND)"
    int cancellation_fee "VND >= 0"
    datetime booked_at
    datetime cancelled_at "nullable"
  }
```

### 4.3 Domain Enums & Data Integrity Rules

- **Enums**:
  - `Role`: `CUSTOMER | STAFF | ADMIN`
  - `BookingMode`: `SHARED_ONLY | EXCLUSIVE_ONLY | BOTH`
  - `BookingStatus`: `PENDING | CONFIRMED | REJECTED | CANCELLED | COMPLETED`
- **Data Integrity Rules**:
  - **Uniqueness (case-insensitive)**: `User.email`, `Category.name`,
    `Category.slug`, `Provider.name`, and `(Service.provider_id, name)`.
  - **Domain Bounds**: Numeric, seat-capacity, pricing, and time bounds follow
    the business rules in Section 3.1 and Logical ERD annotations in Section
    4.2.
  - **Actor Attribution**: `processed_by_id` records the last `STAFF`/`ADMIN`
    who confirmed, rejected, or completed the booking (`null` while `PENDING` or
    when customer-cancelled from `PENDING`). `cancelled_by_id` and
    `cancelled_at` are set if and only if `status = CANCELLED`, without
    overwriting `processed_by_id`.
  - **Historical Snapshots**: `Booking` pricing (`unit_price`, `total_price`,
    `cancellation_fee`), `is_exclusive`, and `reserved_seats` are immutable
    snapshots independent of later `Service` or `Schedule` edits.
  - **Soft Deactivation**: Referenced catalog/schedule entities (`Category`,
    `Provider`, `Service`, `Schedule`) use soft deactivation
    (`is_active = false`) rather than hard deletion.
  - **Physical Boundary**: Passwords and refresh tokens are never stored in
    plaintext; all timestamps represent UTC. Technical storage columns (hashes,
    audit timestamps), physical types, DB check constraints, and query indexes
    are defined at schema implementation time.

### 4.4 Auth & User API Contract (`/api/v1` & Swagger `/api/docs`)

- **Conventions**:
  - Base path: `/api/v1`; OpenAPI documentation: `/api/docs`.
  - Input validation rejects unknown or malformed fields with `400 Bad Request`.
  - Timestamps: ISO-8601 UTC (`YYYY-MM-DDTHH:mm:ss.sssZ`).
  - Pagination: `page` (default `1`), `limit` (default `10`, max `50`); list
    response `{ data: T[], meta: { total, page, limit, totalPages } }`.
- **Standard Error Payload**:

  ```json
  {
    "statusCode": 409,
    "error": "Conflict",
    "code": "EMAIL_ALREADY_EXISTS",
    "message": "An account with this email already exists",
    "details": [],
    "timestamp": "2026-11-25T03:00:00.000Z",
    "path": "/api/v1/auth/register"
  }
  ```

- **Auth & Session Endpoints**:

<table>
  <thead>
    <tr>
      <th>Method &amp; Path</th>
      <th>Access</th>
      <th>Request Input</th>
      <th>Success Response</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><code>POST /api/v1/auth/register</code></td>
      <td>Public</td>
      <td><code>{ email, password, fullName }</code></td>
      <td>
        <code>201</code>: <code>{ user, accessToken }</code> +
        <code>Set-Cookie: refresh_token</code> (HttpOnly)
      </td>
    </tr>
    <tr>
      <td><code>POST /api/v1/auth/login</code></td>
      <td>Public</td>
      <td><code>{ email, password }</code></td>
      <td>
        <code>200</code>: <code>{ user, accessToken }</code> +
        <code>Set-Cookie: refresh_token</code> (HttpOnly)
      </td>
    </tr>
    <tr>
      <td><code>POST /api/v1/auth/refresh</code></td>
      <td>Cookie</td>
      <td><code>refresh_token</code> HttpOnly cookie</td>
      <td>
        <code>200</code>: <code>{ accessToken }</code> + rotated
        <code>refresh_token</code> cookie
      </td>
    </tr>
    <tr>
      <td><code>POST /api/v1/auth/logout</code></td>
      <td>Authenticated</td>
      <td><code>refresh_token</code> HttpOnly cookie</td>
      <td>
        <code>204</code>: revokes active refresh session &amp; clears cookie
      </td>
    </tr>
    <tr>
      <td><code>GET /api/v1/auth/me</code></td>
      <td>Authenticated</td>
      <td>Bearer <code>accessToken</code></td>
      <td>
        <code>200</code>:
        <code>{ id, email, fullName, role, permissions }</code>
      </td>
    </tr>
  </tbody>
</table>

- **User Management Endpoints**:

<table>
  <thead>
    <tr>
      <th>Method &amp; Path</th>
      <th>Access</th>
      <th>Request Input</th>
      <th>Success Response</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><code>GET /api/v1/users</code></td>
      <td><code>user:manage</code></td>
      <td>
        Query: <code>role?</code>, <code>isActive?</code>, <code>page</code>,
        <code>limit</code>
      </td>
      <td><code>200</code>: <code>{ data: UserSummary[], meta }</code></td>
    </tr>
    <tr>
      <td><code>PATCH /api/v1/users/:id/role</code></td>
      <td><code>user:manage</code></td>
      <td><code>{ role }</code></td>
      <td>
        <code>200</code>:
        <code>{ id, email, fullName, role, isActive }</code>
      </td>
    </tr>
    <tr>
      <td><code>PATCH /api/v1/users/:id/status</code></td>
      <td><code>user:manage</code></td>
      <td><code>{ isActive }</code></td>
      <td>
        <code>200</code>:
        <code>{ id, email, fullName, role, isActive }</code> (revokes active
        sessions if <code>false</code>)
      </td>
    </tr>
  </tbody>
</table>

- **Domain Endpoints** (Categories, Providers, Services, Schedules, Bookings,
  Reports): Defined per feature issue in subsequent iterations.

### 4.5 Delivery & Acceptance Criteria

- Idempotent seed data covering all 3 roles, sample categories/providers,
  services across all 3 `booking_mode` values, and upcoming schedules.
- Backend service unit test coverage `>= 60%`.
- Reproducible local full-stack startup via Docker Compose, documented in
  `README.md`.
