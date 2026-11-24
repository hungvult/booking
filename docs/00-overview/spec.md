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
