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
