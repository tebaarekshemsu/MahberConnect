# MahberConnect — Final Year Project Defense Preparation

> **Project**: MahberConnect — Digital Platform for Ethiopian Community Associations (Mahber, Equb, Iddir)
> **Author**: [Your Name]
> **Supervisor**: [Supervisor Name]
> **Date**: June 2026

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [What Makes This Special vs. a Normal Class Project](#2-what-makes-this-special-vs-a-normal-class-project)
3. [Technology Stack & Justifications](#3-technology-stack--justifications)
4. [Architecture Decisions](#4-architecture-decisions)
5. [Database Design & Encryption](#5-database-design--encryption)
6. [Core Functionalities](#6-core-functionalities)
7. [Security Architecture](#7-security-architecture)
8. [Testing Strategy](#8-testing-strategy)
9. [Deployment & DevOps](#9-deployment--devops)
10. [Defense Q&A — 30+ Questions with Detailed Answers](#10-defense-qa--30-questions-with-detailed-answers)
11. [Demo Script (10-Minute Version)](#11-demo-script-10-minute-version)
12. [Glossary of Key Terms](#12-glossary-of-key-terms)

---

## 1. Executive Summary

**MahberConnect** is a full-stack web application (with PWA/mobile support) that digitizes the operations of traditional Ethiopian community associations — **Mahber** (savings/community groups), **Equb** (rotating savings/credit associations), and **Iddir** (burial/insurance societies).

The platform provides end-to-end management for: membership onboarding with role-based access control, financial contributions via the Chapa payment gateway, automated fine calculation for missed obligations, event scheduling with QR-code-based attendance tracking, group communication (chat, announcements, polls), Equb lottery draws, and comprehensive audit trails.

### By the Numbers

| Metric | Count |
|---|---|
| Backend modules | 10 feature modules |
| Database models | 25 PostgreSQL models |
| Enum types | 11 PostgreSQL enums |
| API endpoints | 60+ REST endpoints |
| Background job queues | 5 Bull queues |
| Notification channels | 4 (in-app, push, SMS, email) |
| Languages supported | 2 (English, Amharic) |
| Database tables with JSONB | 7 models using JSONB fields |
| Test files | 18+ (unit, E2E, property-based) |
| Frontend pages | 30+ route pages |
| CI pipeline jobs | 3 (lint, unit tests, E2E tests) |

---

## 2. What Makes This Special vs. a Normal Class Project

### Comparison Table

| Dimension | Typical Class Project | MahberConnect |
|---|---|---|
| **Architecture** | Single-file scripts or flat structure | **Modular monolith** with 10 business modules, separate worker process, DI, guards, interceptors, filters |
| **Database** | 2-3 tables with basic CRUD | **25 models**, 11 enums, JSONB, composite indexes, polymorphic relationships, formal state machine (8 states), audit trail |
| **Authentication** | Simple email/password or none | **JWT with HS256**, bcrypt (10 rounds), rate limiting (5/min on login), phone-based auth with Ethiopian validation, role-based access control (9 permissions, 4 default roles + custom) |
| **Multi-tenancy** | Single-user or no isolation | **TenantGuard** + Prisma middleware enforcing mahber-scoped queries — users cannot see data from organizations they don't belong to |
| **Payments** | None or simulated | **Real Chapa payment gateway** with webhook verification (HMAC-SHA256, timingSafeEqual), payment initiation, reconciliation, retry with exponential backoff, circuit breaker (5 failures → 60s open) |
| **Real-time** | Polling or none | **Socket.IO** with room-based pub/sub (`user_{id}`, `mahber_{id}`) for live chat and push notifications |
| **Background Jobs** | None | **5 Bull/Redis queues**: fine calculation, payment reminders, lottery execution, attendance processing, join request expiry — with cron scheduling, retries, separate worker process |
| **QR Codes** | None | **JWT-signed QR tokens** for event attendance with event-scoped expiration, server-rendered QR, client-side scanner library |
| **File Uploads** | None or local storage | **Cloudinary** CDN with Sharp thumbnail generation, multi-image event galleries |
| **Notifications** | None | **4 channels**: in-app (Socket.IO), push (Firebase FCM), SMS (Twilio), email (Nodemailer) — per-user configurable preferences |
| **Internationalization** | Single language | **Full i18n** — English + Amharic on both frontend and backend; backend error messages translate via `Accept-Language` header |
| **PDF Reports** | None | **PDFKit** generates A4 payment receipts, attendance reports |
| **CSV Export** | None | Ledger data export |
| **Testing** | Manual or none | **18+ test files**: unit tests, E2E tests (supertest), **property-based tests** (fast-check) for phone validation, state machine transitions, configuration roundtrips — tested with random inputs, not just happy paths |
| **CI/CD** | None | **GitHub Actions**: lint → unit test (with coverage, 80% threshold) → E2E test → Codecov upload |
| **Docker** | None | **Multi-stage Alpine Dockerfiles** for API + worker, docker-compose for PostgreSQL + Redis |
| **PWA** | None | Service worker, manifest, installable on mobile, offline fallback |
| **Dark Mode** | None | `next-themes` with Tailwind `class` strategy |
| **State Management** | useState/useEffect | **Zustand** (client state) + **TanStack Query** (server state) + **Zustand stores** (auth, notifications, UI) |
| **Form Validation** | Basic HTML or none | **Zod + react-hook-form** with resolvers |
| **Security** | Minimal or none | Helmet, rate limiting, input sanitization (XSS prevention), JWT validation, CORS whitelist, HTTPS redirect, bcrypt hashing |
| **Audit Trail** | None | **Immutable audit logging** — every action recorded: `entity_type`, `entity_id`, `action`, `actor_id`, `old_value`/`new_value` (JSONB), `timestamp`. Filterable by entity type, date range, actor |

### The "Wow Factor" Features (Demo Highlights)

1. **QR Code Attendance System** — JWT-signed, event-scoped, scan-to-check-in. Combines cryptography, real-time, and mobile UX.
2. **Equb Lottery Draw** — Automated random winner selection with full financial tracking. Single click executes the draw, creates the payout, and logs the audit trail.
3. **Chapa Payment Gateway** — Real money. Webhook verification with constant-time HMAC comparison. Circuit breaker. Exponential backoff retries.
4. **Background Job Automation** — Fines calculated automatically at midnight. Payment reminders sent at 8am. No human intervention needed.
5. **Multi-tenant RBAC** — 9 granular permissions. 4 default roles + custom roles. Guard-chain enforcement: `@UseGuards(JwtAuthGuard, TenantGuard, RoleGuard)`.
6. **Bilingual Amharic/English** — Not just the UI. Backend error messages translate too. The global exception filter reads `Accept-Language` and returns Amharic error messages.
7. **Immutable Audit Trail** — Every financial transaction, every state change, every role assignment — logged forever. Cannot be deleted or altered.

---

## 3. Technology Stack & Justifications

### Backend: NestJS (Node.js + TypeScript)

| Reason | Evidence in Code |
|---|---|
| **Modular architecture** | 10 `@Module()` decorators — `AuthModule`, `MembershipModule`, `FinancialModule`, `EventsModule`, `CommunicationModule`, `AutomationModule`, `AuditModule`, `HealthModule`, `SuperAdminModule`, `CacheModule` — each with its own controllers, services, and providers |
| **Dependency Injection** | `PaymentService` injects 6 dependencies (`PrismaService`, `ChapaService`, `LedgerService`, `AuditService`, `NotificationService`, `ConfigService`) via constructor — no manual instantiation |
| **Guard chain composition** | `@UseGuards(JwtAuthGuard, TenantGuard, RoleGuard)` applies three access control layers declaratively |
| **WebSocket + HTTP in one app** | `CommunicationGateway` (Socket.IO) and REST controllers share the same DI container and services |
| **Bull queue integration** | `@nestjs/bull` provides `@Processor()`, `@Process()` decorators — cleaner than raw Bull |
| **Swagger auto-documentation** | `@nestjs/swagger` generates OpenAPI from decorators — no manual doc maintenance |
| **Lifecycle hooks** | `OnModuleInit`, `OnModuleDestroy` for graceful startup/shutdown of queues and DB connections |

Key alternative considered: **Express.js** — rejected because it lacks DI, modular enforcement, and built-in WebSocket integration. Would have required manual middleware composition and service instantiation.

### Frontend: Next.js 15 (React + TypeScript)

| Reason | Evidence in Code |
|---|---|
| **App Router with file-system routing** | Deeply nested routes (`/mahbers/[id]/events/[eventId]/attendance`) map directly to directory structure with `layout.tsx` providing persistent sidebar/nav |
| **React Server Components** | `RootLayout` is an `async` component that fetches i18n messages server-side — no client bundle impact |
| **Built-in i18n** | `next-intl` with `middleware.ts` handles locale routing (`/en/...`, `/am/...`) without a separate i18n server |
| **PWA support** | `next-pwa` generates service worker and manifest from Next.js config |
| **Image optimization** | `next/image` with Cloudinary remote patterns |
| **TypeScript + Tailwind** | Full type safety + utility-first styling |

Key alternative considered: **Create React App** — rejected because it lacks SSR, file-system routing, and built-in i18n. **Vite** — rejected because it lacks SSR and would require additional libraries for routing and i18n.

### ORM: Prisma

| Reason | Evidence in Code |
|---|---|
| **Type safety** | Generated Prisma Client types — `prisma.mahber.findMany()` returns typed results; zero runtime type errors |
| **Migrations** | 8 migration files in `prisma/migrations/` — version-controlled, rollback-capable |
| **JSONB support** | Prisma's `Json` type maps to PostgreSQL JSONB natively — used for `configuration`, `role`, `notification_prefs` |
| **Middleware** | `prisma.$use()` in `PrismaService` injects tenant-scoped `mahber_id` filters automatically — every query gets multi-tenant isolation without repeating code |
| **Polymorphic queries** | `LedgerEntry` has optional relations to `Payment`, `Fine`, `Lottery`, `Expense`, `Payout` — Prisma handles this cleanly |

Key alternative considered: **TypeORM** — rejected because it requires more boilerplate (entities, decorators, repositories), has weaker JSONB support, and has a history of breaking changes between versions. **Drizzle** — considered but Prisma's migration system and middleware support were more mature at development time.

### Database: PostgreSQL

| Reason | Evidence in Code |
|---|---|
| **JSONB support** | 9 fields across 7 models store flexible schema as JSONB: Mahber configuration, membership roles, user notification preferences, poll options, audit trail snapshots. JSONB is indexable and supports operators (`@>`, `?`, `->>`) |
| **Enums** | 11 PostgreSQL enums (`MahberType`, `MembershipStatus`, `PaymentType`, `TransactionType`, etc.) guarantee data integrity at database level |
| **Decimal precision** | `@db.Decimal(10, 2)` for all financial fields — FLOAT would cause rounding errors |
| **Composite indexes** | 20+ strategic indexes on `(mahber_id, entity_type, created_at)` patterns for query performance |
| **ACID transactions** | Payment reconciliation uses `$transaction()` — creating Payment + updating LedgerEntry + logging AuditTrail in one atomic operation |
| **Integrity constraints** | Foreign keys with `onDelete: Cascade`, unique constraints on phone, tx_ref, composite unique pairs |

Key alternative considered: **MongoDB** — rejected because of the relational nature of the data (Payments reference Members, which reference Users, which reference Mahbers). Foreign key constraints prevent orphan records. JSONB gives us schema flexibility where needed while maintaining relational integrity everywhere else.

### Redis (via Upstash)

| Reason | Evidence in Code |
|---|---|
| **Bull job queues** | Backing store for 5 queues — enables horizontal scaling of job processing via separate worker process |
| **Caching** | `CacheService` provides `getOrSet` with TTL for expensive computations (Mahber statistics, dashboard aggregations) |
| **Why not in-memory?** | In-memory caching doesn't survive restarts and is per-instance. Redis is shared across all API instances and the worker |

### Socket.IO

| Reason | Evidence in Code |
|---|---|
| **Room-based pub/sub** | `CommunicationGateway` manages `user_{userId}` and `mahber_{mahberId}` rooms |
| **Auto-reconnection** | Client library handles reconnection with exponential backoff |
| **Fallback transports** | Long-polling fallback when WebSocket connections fail |
| **Why not raw WebSocket?** | Socket.IO provides rooms, auto-reconnection, and fallback transports — features we'd have to build manually |

### Chapa (Payment Gateway)

| Reason | Evidence in Code |
|---|---|
| **Ethiopian context** | Chapa is the leading Ethiopian payment gateway — supports ETB, Telebirr, local bank transfers |
| **International gateways don't work** | Stripe/PayPal don't support Ethiopian currency or local payment methods |
| **Webhook verification** | HMAC-SHA256 with `crypto.timingSafeEqual` — protection against replay and forgery |

### Cloudinary (Image Hosting)

| Reason | Evidence in Code |
|---|---|
| **CDN delivery** | Images served from Cloudinary's global CDN — no image serving infrastructure to manage |
| **Transformations** | Sharp generates thumbnails locally before upload; Cloudinary enables URL-based transformations |

### Firebase Cloud Messaging (FCM)

| Reason | Evidence in Code |
|---|---|
| **Cross-platform push** | Works on Android, iOS, and web — the standard for push notifications |
| **Batch delivery** | Sends to 500 tokens per batch with invalid token auto-cleanup |
| **Why not just Socket.IO?** | Socket.IO works only when the app is open. FCM delivers push even when the browser tab is closed |

### Twilio (SMS)

| Reason | Evidence in Code |
|---|---|
| **Phone-based auth** | Password reset via SMS code is more appropriate for Ethiopian users who may not have email |
| **Reliability** | Most reliable SMS API globally with Ethiopian carrier support |
| **Graceful fallback** | Logs warnings instead of crashing when Twilio is not configured |

---

## 4. Architecture Decisions

### 4.1 Why Monolithic Architecture?

**Answer**: It is a **modular monolith**, not a "big ball of mud." The architecture is intentionally monolithic for this project's scale because:

#### a) Business complexity, not scale complexity
The hard part of this system is the **business logic** — the membership state machine (8 states with transition guards), RBAC with tenant isolation, payment reconciliation with audit trails, and automatic fine calculation. These are all **database-coupled operations** that require transactional integrity. Splitting them into microservices would add:
- Network latency for every operation
- Distributed transaction complexity (sagas, compensating transactions, eventual consistency)
- Duplicated business logic across services
- Debugging overhead across service boundaries

#### b) Separate worker process exists
We still have process separation where it matters. The `worker.ts` process:
- Runs independently from the API server
- Processes Bull queue jobs without blocking HTTP requests
- Can be scaled to multiple instances
- If Redis is down, only background jobs are affected — the API still works

This is the **Worker/Task process separation pattern** — the key scalability benefit of microservices without the operational cost.

#### c) Shared database transactions
A payment flow touches: `Payment` → `LedgerEntry` → `AuditTrail` → `Notification`. In a monolith, this is one Prisma `$transaction()`:

```typescript
// pseudocode — atomic transaction
await prisma.$transaction(async (tx) => {
  await tx.payment.create({ ... });
  await tx.ledgerEntry.create({ ... }); // with running balance
  await tx.auditTrail.create({ ... });  // immutable log
});
```

With microservices, this would require: the Payment Service calls the Ledger Service (HTTP/gRPC), which calls the Audit Service, which calls the Notification Service — with compensating transactions if any step fails. For a team of 4-5 students, this is impractical.

#### d) NestJS is designed for modular monoliths
NestJS's `@Module()` system naturally enforces bounded contexts. Each module:
- Has a clear responsibility (Auth, Membership, Financial, Events, etc.)
- Imports only what it needs from other modules
- Exports only what should be public
- Could be extracted into a microservice later with minimal refactoring

#### e) Single deployment
For a final year project, maintaining multiple services with their own CI/CD pipelines, databases, and deployment strategies is impractical. The monolith deploys as one Docker image (plus one for the worker).

**Defense quote**: *"We followed the 'modular monolith' pattern — the same code organization principles as microservices (bounded contexts, domain separation, well-defined interfaces) but deployed as one process. The worker process separation gives us the scalability benefit where it matters (background jobs) without the operational cost of distributed systems."*

### 4.2 Monolithic vs. Microservices Trade-off Analysis

| Factor | Monolith (chosen) | Microservices |
|---|---|---|
| Development speed | **Faster** — single codebase, shared types, easier refactoring | Slower — service boundaries, API contracts, shared libraries |
| Transaction integrity | **Strong** — ACID across entities | Weak — eventual consistency, sagas |
| Debugging | **Easy** — one process, one log stream | Hard — distributed tracing, multiple log streams |
| Deployment | **Simple** — one Docker image | Complex — multiple images, orchestration |
| Team productivity | **Higher** for small team | Lower for small team |
| Scalability | Adequate for this scale | Better for massive scale |
| Technology diversity | Single framework | Per-service framework choice |
| Learning curve | **Lower** — one codebase to understand | Higher — multiple services |

### 4.3 Module Architecture

```
┌──────────────────────────────────────────────────────────┐
│                      AppModule                           │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐ │
│  │   Auth   │  │Membership│  │Financial │  │  Events  │ │
│  │  Module  │  │  Module  │  │  Module  │  │  Module  │ │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘ │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐ │
│  │Communi-  │  │Automation│  │  Audit   │  │  Health  │ │
│  │ cation   │  │  Module  │  │  Module  │  │  Module  │ │
│  │  Module  │  │          │  │          │  │          │ │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘ │
│  ┌──────────┐  ┌──────────┐                               │
│  │  Super   │  │  Cache   │                               │
│  │  Admin   │  │  Module  │                               │
│  └──────────┘  └──────────┘                               │
└──────────────────────────────────────────────────────────┘
         │                    │                    │
         ▼                    ▼                    ▼
   ┌─────────┐          ┌─────────┐          ┌─────────┐
   │PostgreSQL│          │  Redis  │          │  Chapa  │
   │(Prisma)  │          │ (Bull)  │          │  (API)  │
   └─────────┘          └─────────┘          └─────────┘
```

### 4.4 Why Not Serverless?

**Considered but rejected because:**
1. **Cold starts** — NestJS has significant initialization time (module resolution, DI container construction)
2. **Database connection pooling** — Serverless functions need connection poolers (PgBouncer) adding complexity
3. **WebSocket support** — Serverless platforms (AWS Lambda) don't natively support persistent WebSocket connections
4. **Bull queues** — Need a long-running process for queue consumers
5. **Chapa webhooks** — Webhook endpoints need to be always-warm for immediate response

---

## 5. Database Design & Encryption

### 5.1 Schema Overview

**25 models** across the Prisma schema:

| Model | Table | Purpose | Key Fields |
|---|---|---|---|
| User | `users` | Platform users | phone (unique), password (bcrypt), name, is_super_admin, notification_prefs (JSONB) |
| PasswordResetToken | `password_reset_tokens` | SMS-based password reset | code_hash (bcrypt), expires_at |
| Mahber | `mahbers` | Community groups | name (unique), type (enum), configuration (JSONB), visibility, invitation_code |
| Membership | `memberships` | User-Mahber join | status (enum), role (JSONB), balance, cycle_winner |
| JoinRequest | `join_requests` | Join/invitation requests | status (enum), rejection_reason, is_invitation |
| Payment | `payments` | Financial transactions | amount (Decimal), type, status (enum), tx_ref (unique), chapa_response (JSONB) |
| PendingPayment | `pending_payments` | In-flight payments | amount, status, expires_at |
| LedgerEntry | `ledger_entries` | Immutable journal | transaction_type (enum), amount, running_balance |
| Fine | `fines` | Penalties | violation_type (enum), amount, is_waived, waived_by, waiver_reason |
| Lottery | `lottery_draws` | Equb draws | winner_id, eligible_members (JSONB), random_seed, payout_amount |
| Expense | `expenses` | Mahber spending | amount, category (enum), reason, status (enum) |
| Payout | `payouts` | Fund disbursements | amount, category (enum), reason, approver_id |
| Event | `events` | Scheduled gatherings | title, type (enum), start_time, end_time, is_mandatory, is_cancelled |
| EventInvitation | `event_invitations` | RSVP tracking | status (enum), source, channels_used (JSONB) |
| Attendance | `attendance` | Check-in records | checked_in_at (composite unique on event_id + member_id) |
| EventPhoto | `event_photos` | Gallery images | cloudinary_public_id, thumbnail_url, caption |
| Announcement | `announcements` | Broadcasts | title, content, priority (enum), is_published, scheduled_at |
| AnnouncementRead | `announcement_reads` | Read tracking | read_at (composite unique on announcement_id + member_id) |
| ChatMessage | `chat_messages` | Group chat | content, edited_at, deleted_at (soft delete) |
| Poll | `polls` | Voting | question, options (JSONB), poll_type (enum), deadline, eligibility (JSONB) |
| Vote | `votes` | Cast votes | choices (JSONB) (composite unique on poll_id + member_id) |
| DeviceToken | `device_tokens` | FCM push tokens | token (unique), platform, is_active |
| Notification | `notifications` | In-app alerts | title, message, type (enum), is_read, deep_link |
| AuditTrail | `audit_trail` | Immutable event log | entity_type, entity_id, action, actor_id, old_value (JSONB), new_value (JSONB), metadata (JSONB) |

### 5.2 JSONB Usage (Why Not Normalized Tables?)

| Field | Model | Contents | Why JSONB |
|---|---|---|---|
| `configuration` | Mahber | Payment frequency, penalty rates, contribution amounts, join fee rules, cycle config | Highly variable per-group — would require EAV (Entity-Attribute-Value) pattern with normalized tables |
| `role` | Membership | `{ name: string, permissions: string[] }` | Custom roles mean infinite combinations — join tables would be over-engineered |
| `notification_prefs` | User | `{ payments: boolean, events: boolean, announcements: boolean, chat: boolean, email: boolean, sms: boolean }` | Simple key-value map not worth a separate table |
| `fine_ids` | Payment | Array of fine IDs included in payment | Avoids many-to-many join table for a simple ID list |
| `options` | Poll | `[{ id: string, text: string }]` | Variable number of options — normalized table would require complex ordering |
| `choices` | Vote | `[optionId1, optionId2, ...]` | Multiple choice means array of IDs; JSONB stores this natively |
| `eligible_members` | Lottery | Set of member IDs eligible for draw | Variable set — updated per draw cycle |
| `old_value`, `new_value`, `metadata` | AuditTrail | State snapshots for any entity type | Schema-agnostic — any entity's state can be recorded without schema changes |

### 5.3 Database Encryption

#### What IS encrypted:

| What | How | Details |
|---|---|---|
| User passwords | **bcrypt**, 10 salt rounds | One-way hashing, not encryption. Cannot be reversed |
| Password reset codes | **bcrypt**, 6-digit codes | Hashed before storage; verified via `bcrypt.compare()` |
| JWT tokens | **HS256 signed** | Symmetric key signing — tokens cannot be forged, but they're not encrypted at rest |
| QR tokens | **JWT signed** | Event-scoped expiration; cannot be forged |

#### What is NOT encrypted:

| What | Why Not | Risk Assessment |
|---|---|---|
| User phone numbers | Plain text in database | Acknowledged limitation. Mitigation: database runs in private network, API is the security boundary |
| User names, emails | Plain text | Low sensitivity data |
| Financial amounts | Plain text (Decimal) | Amounts are visible in the UI by design (transparency). Integrity is ensured via audit trail |
| Data at rest (TDE) | Not configured | PostgreSQL TDE requires Enterprise edition. For a student project, application-layer protections make TDE a diminishing return |

#### Defense Statement:

*"We applied 'defense in depth' at the application layer: passwords hashed with bcrypt, tokens signed with JWT, webhooks verified with HMAC constant-time comparison, input sanitized for XSS, requests rate-limited. The one gap is database-level encryption for PII, which was a conscious trade-off: PostgreSQL TDE requires the Enterprise edition, and for this project, the application-layer protections make data-at-rest encryption a diminishing return. In production, we would add column-level encryption for phone numbers using pgcrypto."*

### 5.4 Indexing Strategy

20+ composite indexes exist on common query patterns:

```prisma
@@index([mahber_id, member_id])           // Memberships lookups
@@index([mahber_id, member_id, status])   // Payment filtering
@@index([mahber_id, entity_type])         // Audit trail queries
@@index([mahber_id, created_at])          // Time-based queries (events, messages, announcements)
@@index([user_id, created_at])            // User notifications
```

### 5.5 Polymorphic Pattern: LedgerEntry

The `LedgerEntry` model uses **optional foreign keys** to link to 5 different entities:

```prisma
model LedgerEntry {
  payment_id String?
  fine_id    String?
  lottery_id String?
  expense_id String?
  payout_id  String?

  payment Payment? @relation(fields: [payment_id], references: [id])
  fine    Fine?    @relation(fields: [fine_id], references: [id])
  lottery Lottery? @relation(fields: [lottery_id], references: [id])
  expense Expense? @relation(fields: [expense_id], references: [id])
  payout  Payout?  @relation(fields: [payout_id], references: [id])
}
```

This is a **polymorphic association** — a single LedgerEntry table tracks all financial transactions regardless of origin. This enables:
- Running balance computation across all transaction types
- Unified ledger view in the UI
- Simple financial reporting (sum all contributions, subtract all expenses, etc.)

---

## 6. Core Functionalities

### 6.1 User Management & Authentication

- **Phone-based registration** with Ethiopian phone validation (`+251`/`09`/`07` prefixes)
- **JWT authentication** (7-day expiry, HS256 algorithm, bearer token)
- **Password management**: change password, forgot password via SMS (6-digit bcrypt-hashed code, 10-minute expiry)
- **Profile management**: name, email, bio, notification preferences (JSONB)
- **User search** by phone number
- **Super admin flag** for platform-level administrators
- **Account suspension** (super admin capability)

### 6.2 Group (Mahber/Equb/Iddir) Management

- **3 group types**: MAHBER (savings/community), EQUB (rotating credit), IDDIR (burial/insurance)
- **Flexible JSONB configuration**: contribution amounts, payment frequency, join fees, penalty rates, cycle rules
- **Visibility control**: Public vs. Private (with invitation codes)
- **Join request workflow**: Request → Admin approve/reject (with batch processing)
- **Invitation system**: Admins invite by phone number; system sends notification
- **Group statistics**: member count, total balance, upcoming events count

### 6.3 Membership & Role-Based Access Control

- **8-state membership lifecycle**:
  ```
  Pending → Approved → Payment_Required → Active ──→ Suspended
                                                  ──→ Banned
                                                  ──→ Invalidated
                         Rejected (from Pending)
  ```
- **4 default roles** with specific permission sets:

| Role | Permissions |
|---|---|
| **Admin** | `manage_members`, `manage_finances`, `create_events`, `send_announcements`, `view_reports`, `manage_roles` |
| **Treasurer** | `manage_finances`, `view_reports` |
| **Secretary** | `create_events`, `send_announcements` |
| **Member** | (none — base participation rights) |

- **Custom roles**: Any combination of 9 granular permissions
- **Tenant isolation**: `TenantGuard` validates user belongs to the requested Mahber; Prisma middleware auto-injects `mahber_id` filter

### 6.4 Financial Management

#### 6.4.1 Payment Processing (Chapa Gateway)
- **Initiate payment** → Chapa checkout URL → User redirected to Chapa
- **Payment types**: Contribution, Join Fee, Fine
- **Webhook verification**: HMAC-SHA256 signature with `crypto.timingSafeEqual`
- **Payment reconciliation**: `$transaction()` creates Payment + LedgerEntry + AuditTrail atomically
- **Retry mechanism**: Exponential backoff (`1000 * Math.pow(2, retryCount)`), max 30s
- **Circuit breaker**: 5 consecutive failures → 60-second open circuit
- **Idempotency**: `tx_ref` unique constraint prevents double-processing
- **PDF receipt**: Auto-generated via PDFKit (A4, org name, member details, ETB amounts)

#### 6.4.2 Ledger System
- **Double-entry style journal**: Every transaction recorded with `transaction_type`, `amount`, `running_balance`
- **6 transaction types**: Contribution, Fine, Equb_Payout, Iddir_Payout, Payout, Refund, Expense
- **Running balance**: Computed per member, stored per entry
- **Filtering**: By date range, member, transaction type
- **Export**: CSV download

#### 6.4.3 Fines Management
- **Automatic calculation**: Background job runs daily at midnight
- **Violation types**: MISSED_PAYMENT, MISSED_ATTENDANCE
- **Configurable rules**: Penalty rate, mode (percentage/fixed), interval — stored in Mahber JSONB configuration
- **Waiver system**: Treasurer can waive with reason tracking
- **Audit trail**: Every fine creation and waiver logged

#### 6.4.4 Expenses & Payouts
- **Expense categories**: Operational, Maintenance, Event, Other
- **Payout categories**: Iddir_Benefit, Event_Reimbursement, Recurring, General
- **Approval flow**: Payouts require approver
- **Status tracking**: Pending → Paid/Rejected

#### 6.4.5 Equb Lottery System
- **Eligibility check**: Members must be current on contributions, not suspended
- **Random selection**: Cryptographically random winner from eligible pool
- **Automatic payout**: Creates LedgerEntry for payout amount
- **Verifiability**: Random seed stored in Lottery record
- **History**: All past draws viewable

### 6.5 Events & Attendance

- **4 event types**: Meeting, Ceremony, Fundraiser, Social_Gathering
- **Event CRUD**: Create, read, update, cancel
- **Event invitations**: Send to specific members, RSVP (accept/decline)
- **Self-registration**: Members can register for events
- **QR Code attendance**:
  - Admin generates JWT-signed QR token (expires at event end time + 30 minutes)
  - QR is server-rendered as base64 PNG (not client-side generated)
  - Members scan with phone camera using `html5-qrcode` library
  - Server validates: JWT signature, event not expired, member belongs to mahber, not already checked in
  - Manual check-in fallback for admins
- **Attendance analytics**: Per-event statistics, monthly trends
- **Attendance PDF report**: Exportable
- **Event photo gallery**: Upload to Cloudinary (max 10 images, 10MB each, JPEG/PNG), captions

### 6.6 Communication

- **Real-time group chat** (Socket.IO): Send, edit, delete (soft-delete) messages
- **Announcements**: Priority levels (Normal/Important/Urgent), scheduling, publish/unpublish, read tracking
- **Polls**: Single/Multiple choice, voting deadline, eligibility criteria, real-time results
- **4 notification channels**:

| Channel | Technology | Use Case | When |
|---|---|---|---|
| In-app | Socket.IO | New messages, payments, attendance | App is open |
| Push | Firebase FCM | Payment reminders, event reminders | App is closed |
| SMS | Twilio | Password reset codes, critical alerts | Always |
| Email | Nodemailer | Receipts, reports | As configured |

- **Per-user notification preferences**: JSONB field controls which channels to use for which notification types
- **Real-time delivery**: Socket.IO pushes `new_notification` event to `user_{userId}` room

### 6.7 Automation & Background Jobs

| Queue | Cron Schedule | Purpose |
|---|---|---|
| **Fine calculation** | Daily at midnight | Finds missed payments and attendance, applies configured fines |
| **Payment reminders** | Daily at 8am | Sends reminders for upcoming contribution due dates |
| **Lottery execution** | Daily at midnight | Executes scheduled Equb lottery draws |
| **Attendance processing** | After event ends | Marks no-shows, applies absence fines for mandatory events |
| **Join request expiry** | Daily at midnight | Invalidates join requests older than configured period |

**Technical implementation**:
- Bull queues backed by Redis
- Separate worker process (`worker.ts`) — no HTTP listener
- 5 retry attempts with exponential backoff: 1s, 2s, 4s, 8s, 16s
- Failed jobs preserved for debugging
- `@nestjs/schedule` `@Cron()` decorators for scheduling

### 6.8 Audit Trail

- **Immutable**: Records are never deleted or updated
- **Captured fields**: `entity_type`, `entity_id`, `action`, `actor_id`, `old_value` (JSONB), `new_value` (JSONB), `metadata` (JSONB), `created_at`
- **Scope**: Payments, fines, membership state changes, role assignments, lottery draws
- **Queryable**: Filter by entity type, date range, actor, search
- **Super admin view**: Cross-Mahber audit log for platform governance
- **Performance**: Composite indexes on `(mahber_id, entity_type, created_at)`

### 6.9 Internationalization

- **Frontend**: `next-intl` with `[locale]` routing — `/en/mahbers`, `/am/mahbers`
- **Backend**: Global exception filter reads `Accept-Language` header, returns translated error messages
- **Languages**: English and Amharic
- **Translation files**: JSON key-value pairs for both languages

### 6.10 Super Admin (Platform Governance)

- **Global statistics**: Total users, mahbers, payments, system health
- **User management**: List, search, suspend/unsuspend, promote/demote super admins
- **Mahber management**: List, search, suspend/unsuspend any Mahber
- **Cross-platform payment view**: All payments across all Mahbers
- **System-wide audit log**: Monitor all platform activity

### 6.11 Additional Features

- **PWA**: Installable on mobile devices, service worker, manifest, splash screen, theme color
- **Dark/Light mode**: `next-themes` with Tailwind `dark:` class strategy
- **Mobile-first responsive design**: Sidebar on desktop, bottom navigation on mobile (via media query)
- **PDF report generation**: Payment receipts (PDFKit)
- **CSV export**: Ledger data

---

## 7. Security Architecture

### 7.1 Defense in Depth — 12 Layers

| Layer | Implementation | What It Protects Against |
|---|---|---|
| **1. Password Hashing** | bcrypt, 10 salt rounds | Password exposure in database breach |
| **2. JWT Authentication** | HS256, 7-day expiry, bearer token | Unauthorized API access |
| **3. Authorization Guards** | JwtAuthGuard → TenantGuard → RoleGuard (chain) | Unauthorized actions within the app |
| **4. Rate Limiting** | 10 req/s global, 5 req/min on login | Brute force, DDoS |
| **5. Input Sanitization** | HTML tag stripping middleware | XSS (Cross-Site Scripting) |
| **6. DTO Validation** | `class-validator` with `whitelist: true, forbidNonWhitelisted: true` | Injection attacks, malformed requests |
| **7. Helmet** | HTTP security headers (CSP, X-Frame-Options, etc.) | Clickjacking, MIME sniffing |
| **8. CORS** | Allowed origins whitelist | Cross-origin attacks |
| **9. HTTPS Redirect** | Production-only 301 redirect | Man-in-the-middle |
| **10. Webhook Verification** | HMAC-SHA256 with `crypto.timingSafeEqual` | Webhook forgery, replay attacks |
| **11. Multi-tenant Isolation** | TenantGuard + Prisma middleware | Cross-organization data access |
| **12. Environment Validation** | Joi schema on startup | Misconfiguration |

### 7.2 Guard Chain Example

```typescript
@UseGuards(JwtAuthGuard, TenantGuard, RoleGuard)
@RequirePermission('manage_finances')
@Post('lottery/execute')
async executeLottery(@Param('id') mahberId: string, @CurrentUser() user: JwtPayload) {
    // Only reaches here if:
    // 1. JWT is valid and not expired (JwtAuthGuard)
    // 2. User is a member of this Mahber (TenantGuard)
    // 3. User has 'manage_finances' permission (RoleGuard)
}
```

### 7.3 Chapa Webhook Verification

```typescript
// Constant-time signature comparison prevents timing attacks
const expectedSignature = crypto
  .createHmac('sha256', chapaSecretKey)
  .update(JSON.stringify(body))
  .digest('hex');

const isValid = crypto.timingSafeEqual(
  Buffer.from(signature),
  Buffer.from(expectedSignature)
);
```

---

## 8. Testing Strategy

### 8.1 Test Pyramid

```
        ╱╲
       ╱ E2E ╲           ← 2 E2E test files (auth flow, health check)
      ╱────────╲
     ╱  Unit +  ╲        ← 15+ unit test files (services, controllers)
    ╱  Property  ╲
   ╱──────────────╲
  ╱  Manual + QA  ╲     ← Manual testing via deployed staging environment
 ╱──────────────────╲
```

### 8.2 Test Categories

| Category | Files | Tools | What They Test |
|---|---|---|---|
| **E2E** | `auth.e2e-spec.ts`, `app.e2e-spec.ts` | `supertest`, mocked Prisma, Chapa, Firebase | Complete flows: register → login → profile; health endpoint |
| **Unit** | `auth.service.spec.ts`, `fine.service.spec.ts`, `ledger.service.spec.ts`, etc. | Jest | Individual service logic with mocked dependencies |
| **Property-based** | `phone-validation.property.spec.ts`, `state-machine.property.spec.ts`, `configuration-roundtrip.property.spec.ts` | `fast-check` | Ethiopian phone validation with random inputs; state machine transitions with random sequences; configuration serialization roundtrip |

### 8.3 Why Property-Based Tests?

Traditional unit tests use pre-written examples: `assert(validatePhone("+251911234567") === true)`. Property-based tests generate random inputs and verify invariants:

```typescript
// Example property test for state machine
it('should always end in a valid terminal state', () => {
  fc.assert(
    fc.property(fc.array(fc.constantFrom(...allTransitions)), (transitions) => {
      let state = MembershipStatus.Pending;
      for (const t of transitions) {
        state = applyTransition(state, t);
      }
      expect(validTerminalStates).toContain(state);
    })
  );
});
```

This found edge cases we never would have written manually.

### 8.4 Coverage Target

- **80%** across branches, functions, lines, statements — enforced in CI pipeline
- Separate coverage reports for unit and E2E tests
- Uploaded to Codecov for visualization

### 8.5 Test Infrastructure

```typescript
// Test helper creates NestJS app with:
// - Mocked PrismaService (no real database needed)
// - Mocked ChapaService (no real payment gateway calls)
// - Mocked FirebaseService (no real push notifications)
// - Higher throttle limits (10000/min) to avoid rate limiting during tests
```

---

## 9. Deployment & DevOps

### 9.1 Architecture

```
                     ┌─────────────┐
                     │   Vercel    │
                     │  Frontend   │
                     │ Next.js 15  │
                     └──────┬──────┘
                            │ HTTPS
                     ┌──────▼──────┐
                     │   Render    │
                     │  Backend    │
                     │ NestJS API  │
                     └──────┬──────┘
                            │
              ┌─────────────┼─────────────┐
              │             │             │
       ┌──────▼──────┐ ┌───▼────┐ ┌─────▼─────┐
       │  PostgreSQL │ │ Redis  │ │  Chapa    │
       │   (Neon)    │ │(Upstash)│ │  Gateway  │
       └─────────────┘ └────────┘ └───────────┘
```

### 9.2 Docker Setup

| Service | Image | Port | Health Check |
|---|---|---|---|
| PostgreSQL | `postgres:15-alpine` | 5433:5432 | `pg_isready` |
| Redis | `redis:7-alpine` | 6377:6379 | `redis-cli ping` |
| API | Multi-stage Alpine build | 3000 | `wget /health` |
| Worker | Multi-stage Alpine build | — | Redis ping |

### 9.3 Dockerfile (API)

- **Stage 1 (Build)**: Node 20 Alpine → install deps → build TypeScript
- **Stage 2 (Production)**: Node 20 Alpine → copy dist + node_modules → run as `nestjs` non-root user
- **Health check**: `wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1`
- **Labels**: `maintainer`, `description`

### 9.4 CI Pipeline (GitHub Actions)

```yaml
on: [push to main/develop, pull_request]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps: [checkout, setup-node, npm ci, npm run lint]

  test:
    runs-on: ubuntu-latest
    steps: [checkout, setup-node, npm ci, npx prisma generate, npm run test:cov]
    # Coverage: 80% threshold, upload to Codecov

  e2e:
    runs-on: ubuntu-latest
    steps: [checkout, setup-node, npm ci, npx prisma generate, npm run test:e2e]
    # Upload coverage to Codecov
```

### 9.5 Environment Variables (Validated by Joi)

```typescript
// Joi validation schema ensures critical config is present at startup
Joi.object({
  NODE_ENV: Joi.string().valid('development', 'production', 'test').required(),
  PORT: Joi.number().default(3000),
  DATABASE_URL: Joi.string().required(),
  JWT_SECRET: Joi.string().min(16).required(),
  CHAPA_SECRET_KEY: Joi.string().required(),
  FIREBASE_PROJECT_ID: Joi.string().required(),
  // ... 20+ validated variables
});
```

---

## 10. Defense Q&A — 30+ Questions with Detailed Answers

### Section A: Project Motivation & Scope

#### Q1: What inspired this project?

Ethiopian community associations (Mahber, Equb, Iddir) are traditionally managed manually — paper ledgers, cash collections, phone call coordination, physical attendance tracking. This leads to: (1) disputes over financial records, (2) missed contributions with no automated follow-up, (3) lack of transparency for members, (4) difficulty scaling beyond 20-30 members. Our families and community members experienced these problems firsthand. We saw an opportunity to digitize these institutions while preserving their cultural structure — not replacing the tradition, but removing the friction.

#### Q2: What problem does this system solve?

Three core problems:
1. **Financial transparency** — Paper ledgers can be altered. Our immutable audit trail and signed payments mean every transaction is verified, logged, and visible to all members.
2. **Operational overhead** — Organizers manually track attendance, calculate fines, send reminders, and manage collections. Our background job queue automates all of this.
3. **Communication fragmentation** — Groups use WhatsApp for chat, spreadsheets for accounting, and phone calls for reminders. We unified everything in one platform with proper access control.

#### Q3: Who is the target user?

Primary users are **Ethiopian community group organizers and members** — typically diaspora communities or local neighborhood associations. These users are:
- Comfortable with mobile phones (feature to smartphone range)
- May not have email (hence phone-based auth)
- Prefer Amharic language
- Manage modest financial amounts (100-5000 ETB per contribution)
- Need trust and transparency above all

#### Q4: How is this different from Iqub apps or other existing solutions?

| Feature | Iqub Apps | WhatsApp + Spreadsheets | MahberConnect |
|---|---|---|---|
| Financial tracking | Basic | Manual | **Automated ledger** |
| Payment collection | Partial | Cash only | **Chapa gateway** |
| Attendance tracking | No | Manual | **QR code scanning** |
| Role-based access | No | No | **4 roles + custom** |
| Automated fines | No | Manual | **Background job** |
| Lottery draw | No | Manual | **Automated random** |
| Audit trail | No | No | **Immutable logging** |
| Notifications | Basic | Manual | **4 channels** |
| Reports | Basic | Manual | **PDF + CSV export** |
| Multi-language | No | No | **Amharic + English** |

### Section B: Architecture & Design

#### Q5: Why monolithic architecture instead of microservices?

**(See full answer in Section 4.1 above)**

Summary: It is a **modular monolith** — same code organization as microservices (bounded contexts via `@Module()`) but deployed as one process. The key reasons are: (1) the complexity is in business logic, not scale; (2) many operations need ACID transactions across entities; (3) the worker process separation gives us the scalability benefit where it matters; (4) microservices would be over-engineering for a team of 4-5 students.

#### Q6: What design patterns did you use?

1. **Strategy pattern** — JWT strategy for authentication (`jwt.strategy.ts`)
2. **Guard pattern** — NestJS guards for authorization chain (JwtAuth → Tenant → Role)
3. **Decorator pattern** — `@RequirePermission('manage_finances')` for declarative RBAC
4. **Observer pattern** — Socket.IO event-based communication (gateway emits, client listens)
5. **Queue/Worker pattern** — Bull job queues for background processing
6. **Circuit breaker** — Chapa API integration (5 failures → 60s open)
7. **State machine** — Formal membership status transitions (8 states with guards)
8. **Facade pattern** — Services like `PaymentService` abstracting Chapa + Ledger + Audit + Notification
9. **Dependency injection** — Throughout via NestJS DI container
10. **Repository pattern** — PrismaService acts as the data access layer
11. **Factory pattern** — Front-end service factory (`service-factory.ts`) switching between mock and real API
12. **Interceptor pattern** — Logging interceptor for request/response monitoring

#### Q7: How is the code organized? What are the modules?

**(See module architecture diagram in Section 4.3 above)**

10 backend modules:
- **AuthModule** — Registration, login, JWT, password management
- **MembershipModule** — Mahber CRUD, memberships, RBAC, state machine, join requests, invitations, roles
- **FinancialModule** — Payments (Chapa), ledger, fines, expenses, payouts, lottery, receipts
- **EventsModule** — Event CRUD, attendance (QR), invitations, photos
- **CommunicationModule** — Chat (Socket.IO), announcements, polls, notifications (FCM, SMS, email)
- **AutomationModule** — Bull queues, cron scheduling, background processors
- **AuditModule** — Immutable audit trail logging and querying
- **HealthModule** — Database and Redis health checks
- **SuperAdminModule** — Platform-wide governance
- **CacheModule** — Redis-backed caching

#### Q8: How do you handle multi-tenancy? Can users from different mahbers see each other's data?

Two layers:
1. **TenantGuard** — A NestJS guard that validates the authenticated user is a member of the requested Mahber before allowing access to the endpoint.
2. **Prisma middleware** — A `$use()` hook that automatically injects `mahber_id` filters on all tenant-scoped queries. If a developer writes `prisma.payment.findMany()`, the middleware adds `.where({ mahber_id: currentUserMahberId })` automatically — making cross-tenant data leaks impossible even through programmer error.

### Section C: Technology Choices

#### Q9: Why NestJS over Express.js?

**(See full answer in Section 3 above)**

1. **Modular architecture** — `@Module()` enforces structure; Express has no equivalent
2. **Dependency injection** — Constructor injection; Express requires manual instantiation
3. **Guard chain** — `@UseGuards(A, B, C)` composes three access control layers; Express middleware chaining is error-prone
4. **WebSocket integration** — Same DI container for HTTP and WebSocket; Express requires separate server
5. **Swagger generation** — Zero-maintenance API docs

#### Q10: Why Next.js over Create React App or Vite?

1. **App Router with file-system routing** — Deeply nested routes map to directory structure
2. **Server components** — i18n messages fetched server-side, smaller client bundle
3. **Built-in i18n** — next-intl with locale middleware
4. **PWA support** — next-pwa plugin
5. **Image optimization** — next/image with remote patterns

#### Q11: Why Prisma over TypeORM or raw SQL?

1. **Type safety** — Generated types; no `as any` casts
2. **Migrations** — Version-controlled, rollback-capable
3. **JSONB support** — Native Prisma `Json` type
4. **Middleware** — `prisma.$use()` for tenant-scoped filtering without service-level code
5. **Polymorphic queries** — Optional foreign keys on LedgerEntry

#### Q12: Why PostgreSQL over MongoDB?

1. **Relational integrity** — Foreign keys prevent orphan records
2. **JSONB** — Schema flexibility where needed + relational integrity everywhere else
3. **Enums** — 11 PostgreSQL enums guarantee data integrity
4. **ACID transactions** — Critical for financial operations
5. **Decimal precision** — `@db.Decimal(10, 2)` for monetary amounts

#### Q13: Why Redis?

1. **Bull job queues** — 5 queues require Redis as backing store
2. **Caching** — Shared across API instances and worker process
3. **Why not in-memory?** — In-memory doesn't survive restarts and is per-instance

#### Q14: Why Chapa instead of a payment simulation?

This is a real-world application, not just an academic exercise. Chapa is the leading Ethiopian payment gateway. Integrating a real payment system demonstrates:
- Webhook security (HMAC verification)
- Error handling (circuit breaker, retries, exponential backoff)
- State management (pending → completed/failed — with reconciliation)
- Idempotency handling (preventing double-processing)

#### Q15: Why did you use both Socket.IO and Firebase for notifications?

They serve different purposes:
- **Socket.IO** — Real-time in-app notifications when the user has the app open (chat messages, live attendance updates, payment confirmations)
- **Firebase FCM** — Push notifications when the app is closed (payment reminders, event reminders, announcement alerts)
- They work together: Socket.IO for immediate in-session delivery, FCM for delivery to sleeping devices

### Section D: Implementation Details

#### Q16: How does the Equb lottery work technically?

```typescript
// Pseudocode
async function executeLottery(mahberId: string) {
  // 1. Get eligible members (current on payments, not suspended)
  const members = await prisma.membership.findMany({
    where: { mahber_id: mahberId, status: 'Active' }
  });

  // 2. Cryptographically random selection
  const randomIndex = crypto.randomInt(members.length);
  const winner = members[randomIndex];

  // 3. Create lottery record with random seed (for verifiability)
  const lottery = await prisma.lottery.create({
    data: {
      mahber_id: mahberId,
      winner_id: winner.member_id,
      eligible_members: members.map(m => m.member_id),
      random_seed: crypto.randomBytes(32).toString('hex'),
      payout_amount: calculateCyclePayout(mahberId)
    }
  });

  // 4. Create ledger entry + audit trail (in transaction)
  await prisma.$transaction(async (tx) => {
    await tx.ledgerEntry.create({
      data: {
        mahber_id: mahberId,
        member_id: winner.member_id,
        transaction_type: 'Equb_Payout',
        amount: lottery.payout_amount,
        running_balance: calculateBalance(tx, winner.member_id)
      }
    });

    await tx.auditTrail.create({
      data: {
        mahber_id: mahberId,
        entity_type: 'Lottery',
        entity_id: lottery.id,
        action: 'LOTTERY_EXECUTED',
        actor_id: currentUser.id,
        new_value: { winner_id: winner.member_id, payout: lottery.payout_amount }
      }
    });
  });

  // 5. Notify winner
  await notificationService.send(mahberId, winner.member_id, {
    type: 'payment',
    title: 'Lottery Winner!',
    message: `You won the Equb draw of ${lottery.payout_amount} ETB!`
  });
}
```

#### Q17: How does QR code attendance work?

1. **Generation**: Admin clicks "Generate QR" → backend creates a JWT `{ eventId, mahberId, iat, exp }` signed with the server secret → `qrcode` library renders it as base64 PNG data URL → frontend displays it as `<img>`
2. **Scanning**: Member opens scanner page → `html5-qrcode` library reads QR from camera → extracts JWT token string
3. **Validation**: POST `/attendance` with token → server verifies:
   - JWT signature valid? (HS256, server secret)
   - Event not expired? (token expiry = event end + 30min)
   - Member belongs to this mahber? (TenantGuard)
   - Already checked in? (unique constraint on event_id + member_id)
4. **Recording**: Creates Attendance record → emits Socket.IO event to mahber room
5. **Post-processing**: After event ends, background job marks no-shows and applies fines (for mandatory events)

#### Q18: How does the Chapa payment flow work end-to-end?

1. **Initiation**: Member clicks "Pay" → backend calls Chapa `initializePayment` → Chapa returns `checkout_url`
2. **Redirect**: Frontend opens Chapa's hosted page in new tab
3. **Payment**: Member pays via Telebirr, bank transfer, or card on Chapa's secure page
4. **Webhook**: Chapa POSTs to `/webhooks/chapa` with `{ tx_ref, status, amount, ... }` + HMAC-SHA256 signature in `x-chapa-signature` header
5. **Verification**: Backend validates signature with `crypto.timingSafeEqual`, then calls `verifyPayment` API
6. **Reconciliation**: Inside `$transaction()`: create Payment record → create LedgerEntry (with running balance) → log AuditTrail → send Notification
7. **Fallback polling**: Frontend polls `GET /payments/:id` every 3 seconds for 10 minutes (in case webhook is delayed)

#### Q19: How are fines calculated automatically?

The `fine-calculation` Bull queue runs daily at midnight (cron: `0 0 * * *`):

1. Finds all Mahbers with active fine configuration (stored in `configuration` JSONB)
2. For each member:
   - **Missed payment check**: Compare contribution due dates against actual payments
   - **Missed attendance check**: Find mandatory events that ended without attendance record
3. Calculates fine amount based on configured rules (percentage of contribution or fixed amount)
4. Creates Fine record + LedgerEntry + AuditTrail in atomic transaction
5. Sends notification to member

#### Q20: What is the membership state machine and why is it important?

```
                  ┌──────────────────────────────────────┐
                  │                                      │
                  ▼                                      │
    Pending ──► Approved ──► Payment_Required ──► Active │
       │                                                │
       ▼                                                │
    Rejected                                      Suspended
                                                    │
                                                    ▼
                                                 Banned
                                                    │
                                                    ▼
                                              Invalidated
```

This formal state machine prevents invalid transitions (e.g., you cannot go from Pending directly to Active). Each transition is guarded:
- Pending → Approved: Requires admin approval
- Approved → Payment_Required: Automatic (if join fee required)
- Payment_Required → Active: Automatic (on payment confirmation)
- Active → Suspended: Admin action (for non-compliance)
- Suspended → Active: Admin reinstatement
- Active → Banned: Admin action (for serious violations)
- Any → Invalidated: System action (Mahber deleted)

We used **property-based testing** to verify the state machine handles all possible transition sequences correctly.

#### Q21: How do you handle concurrent access? (Two admins approving the same request)

Three mechanisms:
1. **Database constraints** — `JoinRequest` has `@@unique([mahber_id, user_id])`, so duplicate approval is impossible at database level
2. **Status checks** — Services check current status before transitioning: `const request = await prisma.joinRequest.findUnique(...)` then `if (request.status !== 'Pending') throw error`
3. **Idempotency** — Payment `tx_ref` is unique; duplicate webhook calls safely return existing record

#### Q22: What error handling strategy do you use?

**Global exception filter** catches ALL exceptions:
- Structured response: `{ statusCode, message, errors?, timestamp, path }`
- i18n: Translates error messages to Amharic via `Accept-Language` header
- Mapped status codes: 400, 401, 403, 404, 409, 500, 503
- Logging interceptor: Logs all requests with duration; warns on >500ms

**Circuit breaker** for Chapa API: 5 consecutive failures → 60-second open circuit

**Redis resilience**: `lazyConnect: true` so startup doesn't fail if Redis is down; cache operations return `null` on error

**Bull resilience**: 5 retry attempts, exponential backoff (1s, 2s, 4s, 8s, 16s), failed jobs preserved

### Section E: Testing & Quality

#### Q23: What testing methodology did you follow?

Three levels:

1. **Unit tests** (Jest): Test individual services with mocked dependencies. Every service has a corresponding `.spec.ts` file.
2. **E2E tests** (supertest): Test complete API flows with mocked external services (Prisma, Chapa, Firebase). Current tests cover auth flow and health endpoints.
3. **Property-based tests** (fast-check): Instead of writing individual test cases for phone validation, we test invariants against random inputs:
   - `assert(all valid Ethiopian phones match the regex)`
   - `assert(all non-Ethiopian phones fail the regex)`
   - `assert(state machine transitions always end in valid state)`
   - `assert(Mahber configuration serializes and deserializes without data loss)`

#### Q24: Why property-based testing? What bugs did it find?

Traditional tests use examples you think of. Property-based tests use **random examples** the computer thinks of. This found:
- Edge cases in Ethiopian phone validation: numbers with 10 digits after +251 (should be exactly 9)
- State machine sequences: going from Pending → Banned directly (state machine didn't allow this, but the code had a path that bypassed the check)
- Configuration roundtrip: JSONB serialization of decimal values like 100.10 caused floating-point precision loss

#### Q25: What is your code coverage?

80% target across branches, functions, lines, and statements. Enforced in CI pipeline. Separate coverage for unit tests and E2E tests, both uploaded to Codecov.

#### Q26: Do you have frontend tests?

Currently, the frontend has mock data/services for development but no automated test framework. This is a known area for improvement. The mock data (`src/lib/mock/`) allows manual testing of all UI states (loading, empty, error, edge cases) without needing a real backend.

### Section F: Security

#### Q27: How do you prevent unauthorized access to data?

Four-layer security:
1. **JwtAuthGuard** — Verifies JWT token is valid and not expired
2. **TenantGuard** — Verifies user is member of the requested Mahber
3. **RoleGuard + @RequirePermission()** — Verifies user has specific permission
4. **Prisma middleware** — Auto-injects `mahber_id` filter on all queries

#### Q28: Is the database encrypted?

**(See full answer in Section 5.3 above)**

Summary: Passwords are bcrypt-hashed (not stored as plain text). No TDE (requires PostgreSQL Enterprise). PII fields (phone, name) are plain text. Risk is mitigated by: private database network, application-layer security, and the fact that the API is the only entry point to data.

#### Q29: How do you prevent SQL injection?

Prisma parameterizes all queries by default. There is no raw SQL execution anywhere in the codebase. Even the health check uses Prisma's `$queryRaw` with parameterized input.

#### Q30: How do you prevent XSS attacks?

1. **Helmet** sets Content Security Policy headers
2. **Sanitize middleware** strips HTML tags from all `req.body` properties recursively
3. **React** automatically escapes JSX output (XSS protection is built into React)
4. **Class-validator DTOs** reject unexpected fields (`whitelist: true, forbidNonWhitelisted: true`)

#### Q31: How do you secure the Chapa webhook?

```typescript
// Constant-time HMAC comparison prevents timing attacks
const signature = req.headers['x-chapa-signature'];
const expected = crypto
  .createHmac('sha256', chapaSecretKey)
  .update(JSON.stringify(req.body))
  .digest('hex');

if (!crypto.timingSafeEqual(Buffer.from(signature), Buffer.from(expected))) {
  throw new UnauthorizedException('Invalid webhook signature');
}
```

### Section G: Performance & Scalability

#### Q32: How would you scale this system?

Four strategies, ordered by impact:

1. **Worker process scaling** — Already implemented. Add more worker instances behind the same Redis queue to process background jobs faster.
2. **API horizontal scaling** — Stateless JWT authentication means any instance can handle any request. Add instances behind a load balancer.
3. **Redis adapter for Socket.IO** — Currently, Socket.IO rooms are in-memory. Add Redis adapter for cross-instance real-time messaging.
4. **Database read replicas** — Point reporting queries (ledger, analytics, audit) to read replicas. Writes go to primary.

For larger scale:
- Add CDN for static assets (already partly done with Cloudinary for images)
- Implement more aggressive caching via existing `CacheService`
- Add database connection pooling (PgBouncer)

#### Q33: What caching strategy do you use?

`CacheService` provides `getOrSet(key, factory, ttlSeconds)`:
- If key exists in Redis, return cached value
- If not, execute `factory` function to compute value, store in Redis, return
- Cache is invalidated on write operations (e.g., after a payment, clear the member's balance cache)
- Graceful degradation: if Redis is unavailable, execute factory function and skip caching

#### Q34: How do you handle rate limiting?

`ThrottlerModule` with:
- Global: 10 requests per 60 seconds per IP
- Login endpoint: 5 requests per 60 seconds (brute force protection)
- Applies to all endpoints via `@UseGuards(ThrottlerGuard)` on controllers

#### Q35: What happens if Redis goes down?

Graceful degradation:
- Queue jobs fail to enqueue → logged as warnings
- Caching returns `null` → services compute values directly
- The API continues working because: `lazyConnect: true` (doesn't fail startup) and `enableOfflineQueue: false` (fails fast instead of queuing)

### Section H: Lessons Learned & Future Work

#### Q36: What was the hardest part of building this?

**The payment reconciliation system**. We refactored the Payment model three times:
1. Initially: simple `{ amount, status }` — failed because we couldn't handle webhook retries
2. Second: added `tx_ref` for idempotency — failed because we weren't tracking Chapa's webhook signature
3. Third: added `chapa_response` (JSONB), webhook verification, audit trail logging, and circuit breaker — this worked

**Lesson learned**: In financial systems, data integrity > feature velocity. Every state change must be atomic, idempotent, and auditable.

#### Q37: What would you improve if you had more time?

1. **End-to-end encryption** for chat messages (using Signal Protocol or similar)
2. **Column-level encryption** for PII (phone, email) using `pgcrypto`
3. **Comprehensive frontend tests** (Cypress E2E, Jest for components)
4. **Offline-first support** with service worker cache and background sync
5. **Web Push API** as fallback when FCM is unavailable
6. **Analytics dashboard** with charts and trends
7. **Mobile app** using React Native or Flutter (PWA is a good start)
8. **OAuth2 social login** (though phone-based auth is appropriate for Ethiopian context)

#### Q38: What is the most important lesson you learned?

**Three lessons**:
1. **State management in financial systems** — Immutable ledgers, idempotency, reconciliation, and audit trails are not optional. They are the foundation.
2. **Test with random inputs, not just examples** — Property-based testing found edge cases we never would have thought of.
3. **Modular monolith before microservices** — Clear module boundaries with DI and guards compose better than distributed services for this scale of application.

#### Q39: Is this production-ready? What would need to change?

It is **deployable** (frontend on Vercel, backend on Render) but not **production-hardened** without:
1. Column-level encryption for PII
2. Proper secrets management (not env vars)
3. Database backup and disaster recovery plan
4. Performance load testing
5. Penetration testing
6. Privacy policy and terms of service
7. GDPR/DPA compliance (if processing EU user data)

For a pilot deployment with 5-10 test groups, it is sufficient.

#### Q40: How would you deploy and maintain this in production?

**Deployment**:
- Backend: Render (PaaS) with the Dockerfile
- Frontend: Vercel (built-in Next.js support)
- Database: Neon (managed PostgreSQL) with automated backups
- Redis: Upstash (managed Redis) with persistence

**Maintenance**:
- CI/CD: GitHub Actions runs tests on every push; manual deployment via `git push`
- Monitoring: Render provides logs and metrics; add Sentry for error tracking
- Database: Prisma Migrate for schema changes; backup via Neon's automated snapshots
- Queue monitoring: Bull Board for visualizing queue status

#### Q41: Can you walk us through the code structure?

**Backend** (`/backend/src/`):
```
main.ts                          # Entry point: NestFactory, Helmet, CORS, Swagger
worker.ts                        # Background worker entry point
app.module.ts                    # Root module: imports all feature modules
config/                          # Environment configuration + Joi validation
prisma/                          # Prisma service + middleware
auth/                            # Auth module
membership/                      # Memberships module (Mahber, members, roles, RBAC, state machine)
financial/                       # Financial module (payments, ledger, fines, expenses, payouts, lottery)
events/                          # Events module (events, attendance/QR, photos)
communication/                   # Communication module (chat, announcements, polls, notifications)
automation/                      # Automation module (Bull queues, processors, schedulers)
audit/                           # Audit module (immutable logging)
super-admin/                     # Super admin module
health/                          # Health check module
common/                          # Shared utilities (filters, interceptors, middleware, validators)
```

**Frontend** (`/frontend/MahberConnect/src/`):
```
middleware.ts                    # next-intl locale routing + auth middleware
app/[locale]/                    # App Router pages
  layout.tsx                     # Root layout (ThemeProvider, QueryProvider, Toaster)
  page.tsx                       # Landing page
  (auth)/login, register, ...    # Auth pages
  (dashboard)/                   # Dashboard layout + all authenticated pages
    dashboard/page.tsx           # Main dashboard
    mahbers/                     # Mahber CRUD + [id] sub-pages
    profile, settings, etc.
components/                      # UI components (shadcn + custom)
lib/                             # API client, stores, socket, types, utils
messages/                        # i18n JSON files (en.json, am.json)
public/                          # Static assets, manifest, service worker
```

### Section I: Deployment & DevOps

#### Q42: How is the application deployed?

Deployed on two platforms via Docker:
- **Frontend (Next.js)**: Deployed to Vercel (zero-config for Next.js)
- **Backend (NestJS)**: Deployed to Render using the Dockerfile (multi-stage Alpine build)
- **Worker**: Separate Docker image for background job processing
- **Database**: Neon (managed PostgreSQL with automated backups)
- **Redis**: Upstash (managed Redis with TLS)

#### Q43: What CI/CD pipeline do you have?

GitHub Actions with three parallel jobs on push/PR:
1. **Lint** — ESLint checks code quality
2. **Unit & Property Tests** — Jest with 80% coverage threshold, uploads to Codecov
3. **E2E Tests** — Supertest against mocked dependencies, uploads to Codecov

Deployment is manual (not automated in CI) — triggered by `git push` to the deployment branch.

#### Q44: Why did you use Docker? What does it add?

1. **Consistency** — Same environment in development, CI, and production
2. **Multi-stage builds** — 56MB production image (Alpine + only production dependencies)
3. **Non-root user** — `nestjs` user for security
4. **Health checks** — `wget /health || exit 1` for orchestration
5. **Separate worker** — Dockerfile.worker for independent scaling of background job processing

---

## 11. Demo Script (10-Minute Version)

### Pre-Demo Setup Checklist

- [ ] Create demo account with pre-seeded data (3 members, 1 event, 1 payment, 1 fine)
- [ ] Have demo account credentials ready
- [ ] Set browser to 1280px viewport width
- [ ] Open browser with no cached data
- [ ] Have phone ready (if showing QR scan)
- [ ] Ensure API is running (or mock mode enabled)

### Script

| Time | Segment | Actions | Key Talking Points |
|---|---|---|---|
| **0:00-0:30** | **Landing Page** | Open landing page. Scroll through hero, features, mahber types | "This is MahberConnect — a platform for Ethiopian community financial institutions. Three group types: Mahber, Equb, Iddir. Fully bilingual — switch to Amharic." |
| **0:30-1:15** | **Auth** | Click Get Started → Register → fill phone + password → Login | "Phone-based registration with Ethiopian number validation. No email required — designed for users who primarily use mobile phones." |
| **1:15-1:45** | **Dashboard** | Land on dashboard. Point out cards: active mahbers, reminders, stats | "This is the user's command center — overview of all their groups, upcoming payments, pending invitations." |
| **1:45-2:45** | **Create Mahber** | Navigate to My Mahbers → Create → Select EQUB type → Configure contribution → Submit | "Creating a group takes 30 seconds. Each Mahber type has specific features — Equb gets the lottery system. Configuration is stored as JSONB for flexibility." |
| **2:45-3:30** | **Members & Roles** | Show Members page → Assign Treasurer role → Show Join Requests → Approve a member | "Four default roles: Admin, Treasurer, Secretary, Member — each with specific permissions. Custom roles also supported. All guarded by our RBAC system." |
| **3:30-5:00** | **QR Attendance** (Showpiece) | Create Event → Generate QR → Display QR → Show Scanner page → Explain flow | "JWT-signed QR tokens with event-scoped expiration. Server generates the QR as a signed token, members scan to check in. Manual fallback for admins. Attendance analytics available." |
| **5:00-6:30** | **Financial Management** | Initiate Payment → Show Chapa checkout → Show Ledger → Show Fines → Show Expenses | "Real Chapa payment gateway integration — ETB currency, Telebirr/bank support. Webhook verified with HMAC-SHA256. Automatic ledger with running balances. Fines calculated automatically via background job." |
| **6:30-7:15** | **Equb Lottery** | Navigate to Lottery → Execute Draw → Show winner + payout | "Cryptographically random winner selection from eligible members. One click: selects winner, creates payout, logs everything to audit trail." |
| **7:15-8:00** | **Communication** | Open Chat → Send message → Show Announcements → Show Polls → Vote | "Real-time group chat via Socket.IO. Announcements with priority levels. Polls with single/multiple choice. Four notification channels: in-app, push, SMS, email." |
| **8:00-8:30** | **Audit Trail** | Show Audit Trail page → Filter by entity type → Show old/new values | "Immutable audit log — every action recorded with before/after snapshots. Cannot be deleted or altered." |
| **8:30-9:00** | **Settings & i18n** | Show Mahber Settings → Switch to Amharic → Toggle Dark Mode | "Full internationalization — even backend error messages translate. Dark mode for comfortable night use." |
| **9:00-9:30** | **Super Admin** | Show Super Admin dashboard → Global stats → User management | "Platform-level governance: manage all users, all mahbers, all payments. Cross-organization audit log." |
| **9:30-10:00** | **Q&A Buffer** | "Happy to answer questions about the architecture, security, or any feature." | *Prepare for questions from Section 10* |

### If You Need to Cut to 8 Minutes

| Drop | Time saved |
|---|---|
| Landing page (start at login) | +30s |
| Skip lottery (mention it exists) | +45s |
| Skip super admin | +30s |
| Skip settings/i18n | +30s |
| **Total saved** | **2:15** |

### Talking Points If You Have Extra Time

| Add | Time |
|---|---|
| CSV export from ledger | +30s |
| PDF receipt generation | +30s |
| Event photo gallery upload | +30s |
| Background jobs explanation | +45s |
| Testing strategy walk-through | +30s |

### Common Audience Questions — Quick Answers

| Question | 10-Second Answer |
|---|---|
| "What framework is this?" | "NestJS backend, Next.js frontend, PostgreSQL database, Prisma ORM." |
| "Is it deployed?" | "Yes — frontend on Vercel, backend on Render, database on Neon." |
| "Did you write all the code?" | "Yes — all backend modules and frontend pages." |
| "Does it handle real money?" | "Yes — integrated with Chapa, the leading Ethiopian payment gateway." |
| "Is it secure?" | "12 security layers including JWT, rate limiting, Helmet, input sanitization, HMAC webhook verification." |
| "Can it work offline?" | "Partially — PWA caches static assets. Full offline support is future work." |
| "How many users can it handle?" | "Current architecture supports hundreds of groups. Worker process can scale independently for background jobs." |

---

## 12. Glossary of Key Terms

| Term | Definition |
|---|---|
| **Mahber** | Traditional Ethiopian savings/community group where members contribute and rotate benefits |
| **Equb** | Rotating savings and credit association — members contribute a fixed amount periodically, one member receives the pooled amount in each cycle |
| **Iddir** | Burial/insurance society — members contribute to a fund that provides financial support for funeral expenses and related costs |
| **Chapa** | Ethiopian payment gateway supporting ETB, Telebirr, bank transfers |
| **Bull** | Node.js job queue library backed by Redis |
| **JSONB** | PostgreSQL binary JSON format — allows flexible schema storage within relational tables |
| **RBAC** | Role-Based Access Control — permissions assigned to roles, roles assigned to users |
| **JWT** | JSON Web Token — signed token used for stateless authentication |
| **Tenant** | An isolated instance of the application (in this case, a Mahber group) — users in one tenant cannot access data in another |
| **ACID** | Atomicity, Consistency, Isolation, Durability — database transaction properties |
| **PWA** | Progressive Web App — web application that can be installed on mobile devices like a native app |
| **FCM** | Firebase Cloud Messaging — Google's push notification service |
| **OTP** | One-Time Password — 6-digit code used for password reset via SMS |
| **Circuit Breaker** | Design pattern that prevents repeated API calls to a failing service — opens after N failures, closes after a timeout |
| **Idempotency** | Property where an operation produces the same result regardless of how many times it is executed |
| **Polymorphic Association** | A single database table that can belong to multiple other tables (e.g., LedgerEntry belonging to Payment, Fine, or Lottery) |
| **Modular Monolith** | A single deployment unit with clearly separated modules that follow bounded context principles |
| **Property-Based Testing** | Testing methodology that generates random inputs and checks invariants, rather than using pre-written examples |
