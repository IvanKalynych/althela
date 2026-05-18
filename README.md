# Althena — Full Platform Technical Specification

**Platform:** Mental Health SaaS (B2C + B2B + Public Site)
**Domain:** althena.com
**Roles Covered:** Client · B2B Member · Therapist · Admin · Super Admin · B2B Organization
**Stack:** Next.js 15 (App Router) · Turborepo · Supabase (PostgreSQL) · Prisma · TypeScript · Tailwind CSS
**Architecture:** Althena monorepo pattern (v6.x)
**Version:** 4.0 | **Date:** 2026-05-18


## TABLE OF CONTENTS

- [MONOREPO ARCHITECTURE](#monorepo)
- [PHASE 0 — PRODUCT & SYSTEM FOUNDATION](#phase-0)
  1. [User Roles & Sub-states](#1-user-roles--sub-states)
  2. [Main User Flows & State Machines](#2-main-user-flows--state-machines)
  3. [Business Model (B2C + B2B)](#3-business-model)
  4. [Domain Entities & DB Schema (Prisma)](#4-domain-entities--db-schema)
  5. [Domain Boundaries](#5-domain-boundaries)
  6. [Sensitive Data Inventory](#6-sensitive-data-inventory)
  7. [RBAC Matrix](#7-rbac-matrix)
- [PHASE 1 — CORE PLATFORM](#phase-1)
  8. [Pages & Routes (all roles + public)](#8-pages--routes)
  9. [Sidebar Navigation (per role)](#9-sidebar-navigation)
  10. [Forms — fields, validation, types](#10-forms)
  11. [Tables & Lists](#11-tables--lists)
  12. [API Endpoints](#12-api-endpoints)
  13. [Real-time Functionality](#13-real-time-functionality)
  14. [File Functionality](#14-file-functionality)
  15. [Role-Based Navigation Components](#15-role-based-navigation-components)
- [PHASE 2 — PUBLIC SITE (Guest Domain)](#phase-2)
  16. [Public Domain Entities](#16-public-domain-entities)
  17. [Route Architecture (App Router)](#17-route-architecture)
  18. [Public Forms](#18-public-forms)
  19. [Public API Endpoints](#19-public-api-endpoints)
  20. [SEO Architecture](#20-seo-architecture)
  21. [State Machines (Guest Domain)](#21-state-machines-guest-domain)
  22. [Frontend Architecture & Rendering Strategy](#22-frontend-architecture--rendering-strategy)
  23. [Analytics & Tracking Architecture](#23-analytics--tracking-architecture)
- [ADMIN PANEL DEEP DIVE](#admin-panel)
  24. [Admin Roles & Permission Architecture](#24-admin-roles--permission-architecture)
  25. [Admin Settings Module](#25-admin-settings-module)
  26. [Admin User Management](#26-admin-user-management)
  27. [B2B Organizations Module](#27-b2b-organizations-module)
- [CROSS-CUTTING CONCERNS](#cross-cutting)

---

## MONOREPO ARCHITECTURE {#monorepo}

Althena follows the **Althena v6.x monorepo pattern** managed by Turborepo. Each app is independently deployable; packages are shared libraries imported as `@repo/<name>`.

### Repository Structure

```
althena/
├── apps/
│   ├── web/            # Public marketing site — althena.com (port 3001)
│   ├── app/            # Authenticated client/therapist app — app.althena.com (port 3000)
│   ├── admin/          # Admin panel — admin.althena.com (port 3002)
│   ├── api/            # Webhook handlers, health checks — api.althena.com (port 3003)
│   └── email/          # React Email template previews (port 3004)
│
├── packages/
│   ├── auth/           # @repo/auth       — Clerk (or Better Auth) config, middleware
│   ├── database/       # @repo/database   — Prisma client, schema, migrations
│   ├── design-system/  # @repo/design-system — shadcn/ui, Tailwind tokens, providers
│   ├── email/          # @repo/email      — React Email templates via Resend
│   ├── cms/            # @repo/cms        — Blog/test content, Content Collections
│   ├── payments/       # @repo/payments   — Stripe client, Connect, Billing
│   ├── analytics/      # @repo/analytics  — PostHog + GA4 with consent gating
│   ├── observability/  # @repo/observability — Sentry, BetterStack logging
│   ├── security/       # @repo/security   — Arcjet rate limiting, bot protection
│   ├── seo/            # @repo/seo        — generateMetadata helpers, JSON-LD
│   ├── storage/        # @repo/storage    — Supabase Storage client, presigned URLs
│   ├── realtime/       # @repo/realtime   — Supabase Realtime channel helpers
│   ├── notifications/  # @repo/notifications — In-app notifications (Knock)
│   ├── feature-flags/  # @repo/feature-flags — Feature flag management
│   ├── webhooks/       # @repo/webhooks   — Stripe + Clerk webhook verification (Svix)
│   ├── internationalization/ # @repo/i18n — Multi-language (i18next)
│   └── next-config/    # @repo/next-config — Shared next.config.ts base
│
├── turbo.json
├── biome.jsonc         # Linting + formatting (replaces ESLint + Prettier)
├── package.json
└── tsconfig.json
```

### App Deployment URLs

| App | Subdomain | Notes |
|---|---|---|
| `apps/web` | `althena.com` | Public marketing site, blog, tests, therapist directory |
| `apps/app` | `app.althena.com` | Authenticated platform (client + therapist dashboards) |
| `apps/admin` | `admin.althena.com` | Admin panel — separate CSP, 2FA enforced |
| `apps/api` | `api.althena.com` | Webhook receivers (Stripe, Clerk), health endpoint |
| `apps/email` | dev-only | React Email template preview server |

### Key Pattern Differences vs Plain Next.js

| Pattern | Althena convention | Althena implementation |
|---|---|---|
| Auth | `@repo/auth` wraps Clerk | Clerk for client/therapist; custom admin TOTP layer |
| Database | `@repo/database` exports `prisma` client | Supabase PostgreSQL + Prisma ORM |
| ORM | Prisma schema in `packages/database/prisma/schema.prisma` | Single schema covering all tables |
| Env vars | `packages/<pkg>/keys.ts` with `@t3-oss/env-nextjs` + Zod | Each package owns its env shape |
| Rate limiting | `@repo/security` wraps Arcjet | Arcjet for public APIs; Clerk for auth endpoints |
| Email | `@repo/email` + Resend | React Email templates, Handlebars for admin-editable templates |
| CMS | `@repo/cms` wraps Content Collections or BaseHub | Content Collections for blog/tests (MDX-based, type-safe) |
| Webhooks | `@repo/webhooks` — verification via Svix | `apps/api/app/webhooks/stripe/route.ts`, `apps/api/app/webhooks/clerk/route.ts` |
| Payments | `@repo/payments` wraps Stripe | Stripe Connect Express (therapist payouts) + Stripe Billing (B2B invoicing) |
| Observability | `@repo/observability` — Sentry + BetterStack | Sentry for error tracking, BetterStack for log drain |
| Analytics | `@repo/analytics` — PostHog + GA4 | Consent-gated; EU data residency on PostHog |
| Storage | `@repo/storage` — Supabase Storage | Presigned URLs, Sharp processing via Edge Function |
| Notifications | `@repo/notifications` — Knock | In-app + email notification delivery |

### Environment Variable Pattern

Each package exposes a `keys.ts` with Zod validation. Apps compose them in `env.ts`:

```typescript
// packages/auth/keys.ts
import { createEnv } from '@t3-oss/env-nextjs';
import { z } from 'zod';

export const keys = createEnv({
  server: {
    CLERK_SECRET_KEY: z.string().min(1).startsWith('sk_'),
  },
  client: {
    NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY: z.string().min(1).startsWith('pk_'),
    NEXT_PUBLIC_CLERK_SIGN_IN_URL: z.string().default('/sign-in'),
    NEXT_PUBLIC_CLERK_SIGN_UP_URL: z.string().default('/sign-up'),
  },
  runtimeEnv: {
    CLERK_SECRET_KEY: process.env.CLERK_SECRET_KEY,
    NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY: process.env.NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY,
    NEXT_PUBLIC_CLERK_SIGN_IN_URL: process.env.NEXT_PUBLIC_CLERK_SIGN_IN_URL,
    NEXT_PUBLIC_CLERK_SIGN_UP_URL: process.env.NEXT_PUBLIC_CLERK_SIGN_UP_URL,
  },
});

// packages/database/keys.ts
export const keys = createEnv({
  server: {
    DATABASE_URL: z.string().url().startsWith('postgresql://'),
    DIRECT_URL: z.string().url().optional(), // for Supabase connection pooling
  },
  runtimeEnv: {
    DATABASE_URL: process.env.DATABASE_URL,
    DIRECT_URL: process.env.DIRECT_URL,
  },
});

// apps/app/env.ts — composes all package envs
import { keys as authKeys }     from '@repo/auth/keys';
import { keys as dbKeys }       from '@repo/database/keys';
import { keys as paymentsKeys } from '@repo/payments/keys';
import { keys as analyticsKeys } from '@repo/analytics/keys';
// ... etc
```

### Turborepo Pipeline

```json
// turbo.json
{
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": [".next/**", "!.next/cache/**", "dist/**"]
    },
    "dev": { "cache": false, "persistent": true },
    "lint": {},
    "type-check": { "dependsOn": ["^build"] },
    "db:generate": { "cache": false },
    "db:migrate": { "cache": false },
    "db:push": { "cache": false }
  }
}
```

---

## PHASE 0 — PRODUCT & SYSTEM FOUNDATION {#phase-0}

---

### 1. User Roles & Sub-states

#### 1.1 Primary Roles

| Role | Internal Enum | Description | Auth Method |
|---|---|---|---|
| Client | `client` | End-user seeking therapy; books sessions | Clerk (Email / Google OAuth) |
| Therapist | `therapist` | Licensed professional; manages caseload | Clerk (Email + credential verification) |
| Admin | `admin` | Platform operator; manages users, content, billing | Clerk (Internal email, TOTP 2FA enforced) |
| Super Admin | `super_admin` | Full platform access; can manage admins | Clerk (Internal SSO + hardware key) |
| B2B Member | `b2b_member` | Employee of an org with corporate plan | Clerk (SSO via org IdP or email) |

Role stored in **Clerk `publicMetadata.role`** and mirrored to `profiles.role` in Prisma on Clerk webhook sync.

#### 1.2 Admin Permission Tiers

| Tier | Enum | Can Do |
|---|---|---|
| Support Agent | `admin:support` | View users, manage disputes, read-only on payments |
| Content Moderator | `admin:moderator` | Review therapist credentials, manage reviews, CMS |
| Finance Admin | `admin:finance` | Full payment/payout access, billing config |
| Operations Admin | `admin:operations` | All of the above + platform settings |
| Super Admin | `super_admin` | Everything + admin management + B2B enterprise config |

Tier stored in `Clerk publicMetadata.adminTier` and `profiles.adminTier`.

#### 1.3 Therapist Sub-states

| State | Enum | Description |
|---|---|---|
| Pending Verification | `pending` | Submitted credentials, awaiting admin review |
| Active | `active` | Fully onboarded, accepting clients |
| On Leave | `on_leave` | Temporarily unavailable; no new bookings |
| Suspended | `suspended` | Admin action; login blocked, sessions cancelled |
| Rejected | `rejected` | Credentials not accepted; can resubmit |

#### 1.4 B2B Organization Member States

| State | Description |
|---|---|
| `invited` | Invitation email sent, not yet registered |
| `active` | Registered and can book sessions |
| `deactivated` | HR/admin removed access; data retained |

---

### 2. Main User Flows & State Machines

#### 2.1 Therapist Onboarding
```
REGISTER (Clerk) → CLERK_WEBHOOK_SYNC → EMAIL_VERIFY
  → PROFILE_SETUP → CREDENTIAL_UPLOAD
  → ADMIN_REVIEW
    → [APPROVED] → AVAILABILITY_SETUP → STRIPE_CONNECT → ACTIVE
    → [REJECTED]  → NOTIFICATION_SENT → RESUBMIT
    → [MORE_INFO_NEEDED] → THERAPIST_NOTIFIED → REPLY → RE_REVIEW
```

#### 2.2 Client Registration & First Booking
```
REGISTER (Clerk) → CLERK_WEBHOOK_SYNC → EMAIL_VERIFY
  → INTAKE_FORM → THERAPIST_MATCH
  → VIEW_PROFILE → SELECT_SLOT → PAYMENT_CAPTURE (Stripe)
  → BOOKING_CONFIRMED → REMINDER (Knock) → SESSION → POST_SESSION_REVIEW
```

#### 2.3 B2B Employee Onboarding
```
ORG_ADMIN_INVITES_EMPLOYEE → CLERK_MAGIC_LINK_EMAIL
  → EMPLOYEE_REGISTERS → SELECTS_THERAPIST (org-approved pool)
  → BOOKS_SESSION → BILLED_TO_ORG_ACCOUNT (Stripe Billing)
  → USAGE_REPORTED_TO_ORG_DASHBOARD (anonymized)
```

#### 2.4 Session Lifecycle State Machine
```
SCHEDULED ──► CONFIRMED ──► IN_PROGRESS ──► COMPLETED
     │              │                            │
     ▼              ▼                            ▼
 CANCELLED      CANCELLED                   NO_SHOW
     │
     ▼
REFUND_INITIATED ──► REFUNDED
```

#### 2.5 Admin Therapist Review Flow
```
THERAPIST submits credentials
  → Knock notification to admin queue
  → ADMIN opens review screen (admin.althena.com)
    → Views credential document (Supabase Storage signed URL, 15 min)
    → Checks license against external registry (Nursys/FSMB, optional)
    → [APPROVE] → therapist.status = active, Resend email sent
    → [REJECT + reason] → therapist notified, can resubmit
    → [REQUEST_INFO] → Knock message sent to therapist
```

#### 2.6 Dispute / Refund Flow
```
CLIENT raises dispute on booking
  → Knock notification to admin queue
  → ADMIN reviews session record + messages
    → [APPROVE_REFUND] → Stripe refund via @repo/payments, payout adjusted
    → [REJECT_DISPUTE] → Client notified with reason (Resend)
    → [PARTIAL_REFUND] → Custom amount entered
```

#### 2.7 Guest Conversion Funnels

**Funnel A — Guest → Client Registration**
```
althena.com (Home)
  → [CTA: "Find a Therapist" or "Take the Test"]
    ├── /therapists/[slug] → /sign-up?intent=book&therapistId=[id]
    │     → app.althena.com/bookings (post-registration)
    ├── /tests/[slug]/start → /tests/[slug]/result
    │     → /sign-up?intent=test_result&testId=[id]&band=[band]
    └── /about → /sign-up
```

**Funnel B — B2B Corporate Lead**
```
althena.com/for-business → lead form → api.althena.com/leads (POST)
  → Resend notification to sales team, lead_submissions record created
```

**Funnel C — Therapist Partner Acquisition**
```
althena.com/cooperation → /sign-up?role=therapist (or inline lead form)
```

**Funnel D — Test Funnel (SEO + conversion)**
```
althena.com/tests/[slug] → /tests/[slug]/start → /tests/[slug]/result
  → /sign-up?intent=save_test&sessionToken=[token]
```

#### 2.8 Anonymous / Authenticated Transitions

| Transition Point | Auth Required | Post-auth Redirect |
|---|---|---|
| Book a session | Required (Clerk) | `app.althena.com/bookings` |
| Save a therapist | Required | Return to same page |
| Start a test | Not required | — |
| View test result | Not required | — |
| Save test result | Required | Return to result |
| Newsletter signup | Not required | — |
| Business inquiry | Not required | — |
| Cooperation apply | Not required (lead) / Required (full) | — |

#### 2.9 Test Execution State Machine (Guest Domain)
```
[INTRO]
  → POST /api/public/tests/[testId]/session
    → session_token in httpOnly cookie (24h)
      → [QUESTION_N] × questions
        → PATCH .../answer (debounced 300ms, sendBeacon on unload)
          → POST .../complete (score computed server-side)
            → [RESULT]

[QUESTION_N] → browser closed
  → session persists (in_progress)
  → on return: check httpOnly cookie
    → active: [RESUME_PROMPT]
    → expired (>24h): [INTRO]

[RESULT]
  → authenticated: POST .../claim
  → guest: /sign-up?intent=save_test&sessionToken=[token]
  → "Share": /tests/[slug]/result?band=[result_band] (no PII)
```

#### 2.10 Newsletter Subscription State Machine
```
[IDLE] → POST /api/public/newsletter
  → honeypot check → silent discard if triggered
  → Arcjet rate limit check (3/IP/hour)
  → INSERT newsletter_subscribers (always return 200, no enumeration)
    → Resend: double opt-in confirmation email (mandatory for EU)
      → [PENDING] → user clicks confirm → [CONFIRMED]
      → token expired (24h) → [EXPIRED] → resend option

[CONFIRMED] → unsubscribe link → [UNSUBSCRIBED]
```

#### 2.11 Content Publishing (CMS)
```
DRAFT → [Schedule] → SCHEDULED → [cron Edge Function] → PUBLISHED
DRAFT → [Publish Now] → PUBLISHED
PUBLISHED → [Unpublish] → DRAFT
PUBLISHED → [Archive] → ARCHIVED (410 response)

On PUBLISHED: next.js revalidateTag + revalidatePath
```

---

### 3. Business Model

#### 3.1 B2C Model

| Component | Details |
|---|---|
| Session Fee | Therapist-set rate per session ($60–$300) |
| Platform Commission | Configurable % per completed session (default 20%) |
| Cancellation Fee | Client pays 50% if cancelled <24h (configurable) |
| Payment Processor | Stripe (card, Apple Pay, Google Pay) |
| Payout | Weekly via Stripe Connect Express |
| Subscription (optional) | Discounted session bundles |

#### 3.2 B2B Model

| Component | Details |
|---|---|
| Plan Types | Per-seat monthly · Session-credit pack · Unlimited (enterprise) |
| Billing Entity | Organization pays platform via Stripe Billing (NET-30) |
| Session Coverage | N sessions/month per employee seat |
| Overage | Employee pays out-of-pocket above plan limit |
| Invoicing | Monthly Stripe Invoice → `organization_invoices` record |
| Employee Privacy | Never share individual session content; aggregate stats only |
| Therapist Pool | Org can restrict employees to curated therapist list |
| SSO | SAML/OIDC via Clerk Organizations |
| Reporting | Org admin dashboard: anonymized utilization, avg satisfaction |

---

### 4. Domain Entities & DB Schema

**ORM:** Prisma · **Database:** Supabase (PostgreSQL) · **Schema location:** `packages/database/prisma/schema.prisma`

Supabase connection pooling via pgBouncer — set `DATABASE_URL` to the pooler URL and `DIRECT_URL` to the direct connection (required for Prisma migrations).

```prisma
// packages/database/prisma/schema.prisma

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider  = "postgresql"
  url       = env("DATABASE_URL")
  directUrl = env("DIRECT_URL")
}

// ─────────────────────────────────────────────
// ENUMS
// ─────────────────────────────────────────────

enum UserRole {
  client
  therapist
  admin
  super_admin
  b2b_member
}

enum AdminTier {
  support
  moderator
  finance
  operations
}

enum TherapistStatus {
  pending
  active
  on_leave
  suspended
  rejected
}

enum BookingStatus {
  scheduled
  confirmed
  in_progress
  completed
  cancelled
  no_show
}

enum SessionType {
  video
  audio
  chat
}

enum NoteType {
  soap
  progress
  intake
  discharge
}

enum PaymentStatus {
  pending
  succeeded
  failed
  refunded
  partially_refunded
}

enum PayoutStatus {
  pending
  paid
  failed
}

enum DisputeStatus {
  open
  under_review
  resolved_refund
  resolved_no_action
}

enum OrgPlan {
  per_seat
  credit_pack
  unlimited
  enterprise
}

enum OrgMemberRole {
  member
  org_admin
}

enum OrgMemberStatus {
  invited
  active
  deactivated
}

enum InvoiceStatus {
  draft
  open
  paid
  void
  uncollectible
}

enum BlogStatus {
  draft
  scheduled
  published
  archived
}

enum TestStatus {
  draft
  published
  archived
}

enum TestSessionStatus {
  in_progress
  completed
  abandoned
}

enum LeadType {
  business_inquiry
  cooperation_request
  general_contact
}

enum NewsletterStatus {
  pending
  confirmed
  unsubscribed
  bounced
}

// ─────────────────────────────────────────────
// CORE PLATFORM
// ─────────────────────────────────────────────

model Profile {
  id                String    @id // Clerk user ID
  role              UserRole
  adminTier         AdminTier?
  fullName          String
  avatarUrl         String?
  phone             String?
  timezone          String    @default("UTC")
  dateOfBirth       DateTime? // SENSITIVE — encrypted at app layer
  preferredLanguage String    @default("en")
  isDeleted         Boolean   @default(false)
  createdAt         DateTime  @default(now())
  updatedAt         DateTime  @updatedAt
  lastSeenAt        DateTime?

  therapistProfile      TherapistProfile?
  clientProfile         ClientProfile?
  sentMessages          Message[]         @relation("SentMessages")
  notifications         Notification[]
  adminLogs             AdminAuditLog[]   @relation("AdminLogs")
  raisedDisputes        Dispute[]         @relation("RaisedDisputes")
  resolvedDisputes      Dispute[]         @relation("ResolvedDisputes")
  cancelledBookings     Booking[]         @relation("CancelledBookings")
  impersonations        AdminImpersonation[] @relation("ImpersonationTarget")
  impersonatedBy        AdminImpersonation[] @relation("ImpersonationActor")
  uploadedMedia         MediaAsset[]
  authoredBlogPosts     BlogAuthor?

  @@map("profiles")
}

model TherapistProfile {
  id                      String          @id
  profile                 Profile         @relation(fields: [id], references: [id])
  licenseNumber           String          // SENSITIVE — encrypted
  licenseState            String?
  licenseExpiry           DateTime?
  licenseVerifiedAt       DateTime?
  specializations         String[]
  approaches              String[]
  languages               String[]
  bio                     String?
  yearsOfExperience       Int?
  sessionRateCents        Int
  acceptsInsurance        Boolean         @default(false)
  insuranceProviders      String[]
  stripeAccountId         String?         // SENSITIVE
  onboardingStatus        TherapistStatus @default(pending)
  verifiedById            String?
  rejectionReason         String?
  totalSessionsCompleted  Int             @default(0)
  avgRating               Decimal?        @db.Decimal(3, 2)

  bookings          Booking[]
  sessionNotes      SessionNote[]
  conversations     Conversation[]
  payouts           Payout[]
  reviews           Review[]
  orgPools          OrganizationTherapistPool[]
  availabilityRules AvailabilitySchedule[]
  availabilityExceptions AvailabilityException[]

  @@map("therapist_profiles")
}

model ClientProfile {
  id                     String   @id
  profile                Profile  @relation(fields: [id], references: [id])
  organizationId         String?
  organization           Organization? @relation(fields: [organizationId], references: [id])
  emergencyContactName   String?  // SENSITIVE — encrypted
  emergencyContactPhone  String?  // SENSITIVE — encrypted
  insuranceMemberId      String?  // SENSITIVE — encrypted
  primaryConcern         String[]
  assignedTherapistId    String?
  sessionsRemainingThisMonth Int?

  bookings      Booking[]
  conversations Conversation[]
  reviews       Review[]
  orgMembership OrganizationMember?

  @@map("client_profiles")
}

model AvailabilitySchedule {
  id                  String           @id @default(cuid())
  therapistId         String
  therapist           TherapistProfile @relation(fields: [therapistId], references: [id])
  dayOfWeek           Int              // 0–6
  startTime           String           // HH:mm
  endTime             String           // HH:mm
  slotDurationMinutes Int              @default(50)
  isActive            Boolean          @default(true)

  @@map("availability_schedules")
}

model AvailabilityException {
  id          String           @id @default(cuid())
  therapistId String
  therapist   TherapistProfile @relation(fields: [therapistId], references: [id])
  date        DateTime         @db.Date
  reason      String?
  createdAt   DateTime         @default(now())

  @@map("availability_exceptions")
}

model Booking {
  id                 String        @id @default(cuid())
  clientId           String
  client             ClientProfile @relation(fields: [clientId], references: [id])
  therapistId        String
  therapist          TherapistProfile @relation(fields: [therapistId], references: [id])
  organizationId     String?
  organization       Organization? @relation(fields: [organizationId], references: [id])
  scheduledAt        DateTime
  durationMinutes    Int           @default(50)
  status             BookingStatus @default(scheduled)
  sessionType        SessionType   @default(video)
  cancellationReason String?
  cancelledById      String?
  cancelledBy        Profile?      @relation("CancelledBookings", fields: [cancelledById], references: [id])
  cancelledAt        DateTime?
  rateCents          Int
  platformFeeCents   Int
  isCoveredByOrg     Boolean       @default(false)
  createdAt          DateTime      @default(now())
  updatedAt          DateTime      @updatedAt

  session    Session?
  notes      SessionNote[]
  payment    Payment?
  dispute    Dispute?
  review     Review?

  @@map("bookings")
}

model Session {
  id                   String   @id @default(cuid())
  bookingId            String   @unique
  booking              Booking  @relation(fields: [bookingId], references: [id])
  roomUrl              String?
  startedAt            DateTime?
  endedAt              DateTime?
  durationActualSeconds Int?
  recordingUrl         String?  // SENSITIVE
  transcriptUrl        String?  // SENSITIVE

  @@map("sessions")
}

model SessionNote {
  id                  String           @id @default(cuid())
  bookingId           String
  booking             Booking          @relation(fields: [bookingId], references: [id])
  therapistId         String
  therapist           TherapistProfile @relation(fields: [therapistId], references: [id])
  content             String           // SENSITIVE — AES-256 encrypted before INSERT
  noteType            NoteType
  isSharedWithClient  Boolean          @default(false)
  // SOAP subfields stored in content JSON when noteType = soap
  createdAt           DateTime         @default(now())
  updatedAt           DateTime         @updatedAt

  @@map("session_notes")
}

model Conversation {
  id          String           @id @default(cuid())
  clientId    String
  client      ClientProfile    @relation(fields: [clientId], references: [id])
  therapistId String
  therapist   TherapistProfile @relation(fields: [therapistId], references: [id])
  lastMessageAt DateTime?
  isArchived  Boolean          @default(false)

  messages Message[]

  @@map("conversations")
}

model Message {
  id             String       @id @default(cuid())
  conversationId String
  conversation   Conversation @relation(fields: [conversationId], references: [id])
  senderId       String
  sender         Profile      @relation("SentMessages", fields: [senderId], references: [id])
  content        String       // SENSITIVE — AES-256 encrypted
  attachmentUrl  String?
  isRead         Boolean      @default(false)
  sentAt         DateTime     @default(now())
  deletedAt      DateTime?

  @@map("messages")
}

model Payment {
  id                     String        @id @default(cuid())
  bookingId              String        @unique
  booking                Booking       @relation(fields: [bookingId], references: [id])
  stripePaymentIntentId  String        @unique // SENSITIVE — never logged
  amountCents            Int
  currency               String        @default("usd")
  status                 PaymentStatus @default(pending)
  refundAmountCents      Int?
  refundReason           String?
  createdAt              DateTime      @default(now())

  @@map("payments")
}

model Payout {
  id               String           @id @default(cuid())
  therapistId      String
  therapist        TherapistProfile @relation(fields: [therapistId], references: [id])
  stripeTransferId String?          // SENSITIVE
  amountCents      Int
  periodStart      DateTime         @db.Date
  periodEnd        DateTime         @db.Date
  status           PayoutStatus     @default(pending)
  paidAt           DateTime?

  @@map("payouts")
}

model AdminAuditLog {
  id         String   @id @default(cuid())
  adminId    String
  admin      Profile  @relation("AdminLogs", fields: [adminId], references: [id])
  action     String   // e.g. 'therapist.approve', 'user.suspend'
  targetType String?  // 'therapist' | 'client' | 'booking'
  targetId   String?
  metadata   Json?
  ipAddress  String?
  createdAt  DateTime @default(now())

  @@map("admin_audit_logs")
}

model AdminImpersonation {
  id         String   @id @default(cuid())
  actorId    String
  actor      Profile  @relation("ImpersonationActor", fields: [actorId], references: [id])
  targetId   String
  target     Profile  @relation("ImpersonationTarget", fields: [targetId], references: [id])
  approvedById String?
  startedAt  DateTime @default(now())
  endedAt    DateTime?

  @@map("admin_impersonations")
}

model Dispute {
  id             String        @id @default(cuid())
  bookingId      String        @unique
  booking        Booking       @relation(fields: [bookingId], references: [id])
  raisedById     String
  raisedBy       Profile       @relation("RaisedDisputes", fields: [raisedById], references: [id])
  reason         String
  status         DisputeStatus @default(open)
  resolutionNote String?
  resolvedById   String?
  resolvedBy     Profile?      @relation("ResolvedDisputes", fields: [resolvedById], references: [id])
  resolvedAt     DateTime?
  createdAt      DateTime      @default(now())

  @@map("disputes")
}

model Notification {
  id        String   @id @default(cuid())
  userId    String
  user      Profile  @relation(fields: [userId], references: [id])
  type      String   // See notification type enum in docs
  title     String?
  body      String?
  data      Json?
  isRead    Boolean  @default(false)
  createdAt DateTime @default(now())

  @@map("notifications")
}

model Review {
  id          String           @id @default(cuid())
  bookingId   String           @unique
  booking     Booking          @relation(fields: [bookingId], references: [id])
  clientId    String
  client      ClientProfile    @relation(fields: [clientId], references: [id])
  therapistId String
  therapist   TherapistProfile @relation(fields: [therapistId], references: [id])
  rating      Int              // 1–5
  comment     String?
  isPublic    Boolean          @default(true)
  isFlagged   Boolean          @default(false)
  flaggedReason String?
  createdAt   DateTime         @default(now())

  @@map("reviews")
}

model PlatformSetting {
  key       String   @id
  value     Json
  updatedBy String?
  updatedAt DateTime @updatedAt

  @@map("platform_settings")
}

// ─────────────────────────────────────────────
// B2B
// ─────────────────────────────────────────────

model Organization {
  id                       String    @id @default(cuid())
  clerkOrgId               String?   @unique // Clerk Organizations ID
  name                     String
  domain                   String?
  logoUrl                  String?
  planType                 OrgPlan
  seatsPurchased           Int?
  sessionsPerSeatPerMonth  Int?
  stripeCustomerId         String?   // SENSITIVE
  billingEmail             String
  billingContactName       String?
  ssoProvider              String?   // 'saml' | 'oidc' | null
  ssoConfig                Json?     // SENSITIVE — encrypted at app layer
  isActive                 Boolean   @default(true)
  contractStart            DateTime? @db.Date
  contractEnd              DateTime? @db.Date
  createdAt                DateTime  @default(now())

  members        OrganizationMember[]
  therapistPool  OrganizationTherapistPool[]
  invoices       OrganizationInvoice[]
  usageReports   OrganizationUsageReport[]
  bookings       Booking[]
  clientProfiles ClientProfile[]

  @@map("organizations")
}

model OrganizationMember {
  id                    String          @id @default(cuid())
  organizationId        String
  organization          Organization    @relation(fields: [organizationId], references: [id])
  clientId              String?         @unique
  client                ClientProfile?  @relation(fields: [clientId], references: [id])
  email                 String
  role                  OrgMemberRole   @default(member)
  status                OrgMemberStatus @default(invited)
  sessionsUsedThisMonth Int             @default(0)
  invitedAt             DateTime?
  joinedAt              DateTime?

  @@unique([organizationId, email])
  @@map("organization_members")
}

model OrganizationTherapistPool {
  organizationId String
  organization   Organization    @relation(fields: [organizationId], references: [id])
  therapistId    String
  therapist      TherapistProfile @relation(fields: [therapistId], references: [id])
  addedAt        DateTime         @default(now())

  @@id([organizationId, therapistId])
  @@map("organization_therapist_pools")
}

model OrganizationInvoice {
  id              String        @id @default(cuid())
  organizationId  String
  organization    Organization  @relation(fields: [organizationId], references: [id])
  stripeInvoiceId String        @unique // SENSITIVE
  periodStart     DateTime      @db.Date
  periodEnd       DateTime      @db.Date
  amountCents     Int
  seatsBilled     Int?
  sessionsBilled  Int?
  status          InvoiceStatus @default(draft)
  dueDate         DateTime?     @db.Date
  paidAt          DateTime?
  createdAt       DateTime      @default(now())

  @@map("organization_invoices")
}

model OrganizationUsageReport {
  id                  String       @id @default(cuid())
  organizationId      String
  organization        Organization @relation(fields: [organizationId], references: [id])
  reportMonth         DateTime     @db.Date
  totalMembers        Int
  activeMembers       Int          // had ≥1 session (k-anonymity: suppress if <5)
  totalSessions       Int
  avgSatisfactionScore Decimal?    @db.Decimal(3, 2)
  generatedAt         DateTime     @default(now())

  @@map("organization_usage_reports")
}

// ─────────────────────────────────────────────
// PUBLIC / GUEST DOMAIN
// ─────────────────────────────────────────────

model BlogAuthor {
  id                  String    @id @default(cuid())
  profileId           String?   @unique
  profile             Profile?  @relation(fields: [profileId], references: [id])
  fullName            String
  slug                String    @unique
  bio                 String?
  avatarUrl           String?
  title               String?
  credentials         String?
  twitterUrl          String?
  linkedinUrl         String?
  createdAt           DateTime  @default(now())

  posts BlogPost[]

  @@map("blog_authors")
}

model BlogCategory {
  id             String        @id @default(cuid())
  slug           String        @unique
  name           String
  description    String?
  parentId       String?
  parent         BlogCategory? @relation("CategoryParent", fields: [parentId], references: [id])
  children       BlogCategory[] @relation("CategoryParent")
  seoTitle       String?
  seoDescription String?
  coverImageUrl  String?
  sortOrder      Int           @default(0)
  createdAt      DateTime      @default(now())

  posts BlogPost[]

  @@map("blog_categories")
}

model BlogPost {
  id               String       @id @default(cuid())
  slug             String       @unique
  title            String
  excerpt          String?      @db.VarChar(300)
  content          String       // MDX or HTML
  coverImageUrl    String?
  authorId         String?
  author           BlogAuthor?  @relation(fields: [authorId], references: [id])
  categoryId       String?
  category         BlogCategory? @relation(fields: [categoryId], references: [id])
  tags             String[]
  status           BlogStatus   @default(draft)
  publishedAt      DateTime?
  scheduledFor     DateTime?
  readingTimeMinutes Int?
  seoTitle         String?
  seoDescription   String?
  ogImageUrl       String?
  canonicalUrl     String?
  schemaType       String       @default("Article")
  viewCount        Int          @default(0)
  createdAt        DateTime     @default(now())
  updatedAt        DateTime     @updatedAt

  @@map("blog_posts")
}

model Test {
  id              String     @id @default(cuid())
  slug            String     @unique
  title           String
  description     String?
  instructions    String?
  coverImageUrl   String?
  category        String?
  estimatedMinutes Int?
  questionCount   Int?       // denormalized
  status          TestStatus @default(draft)
  isFeatured      Boolean    @default(false)
  scoringMethod   String     // 'sum'|'weighted_sum'|'subscale'
  resultConfig    Json       // { bands: [...], subscales: null }
  seoTitle        String?
  seoDescription  String?
  ogImageUrl      String?
  schemaType      String     @default("MedicalWebPage")
  completionCount Int        @default(0)
  publishedAt     DateTime?
  createdAt       DateTime   @default(now())
  updatedAt       DateTime   @updatedAt

  questions TestQuestion[]
  sessions  TestSession[]

  @@map("tests")
}

model TestQuestion {
  id           String   @id @default(cuid())
  testId       String
  test         Test     @relation(fields: [testId], references: [id], onDelete: Cascade)
  sortOrder    Int
  text         String
  questionType String   // 'single_choice'|'scale'|'boolean'
  options      Json     // [{ value: 0, label: "Not at all" }]
  subscale     String?
  weight       Decimal  @default(1.0)
  createdAt    DateTime @default(now())

  @@map("test_questions")
}

model TestSession {
  id             String            @id @default(cuid())
  testId         String
  test           Test              @relation(fields: [testId], references: [id])
  sessionToken   String            @unique // httpOnly cookie — NEVER log
  userId         String?           // null if anonymous
  answers        Json              @default("{}")
  currentStep    Int               @default(1)
  status         TestSessionStatus @default(in_progress)
  score          Decimal?
  resultBand     String?
  resultSubscores Json?
  ipHash         String?           // SHA-256 of IP
  userAgent      String?
  completedAt    DateTime?
  expiresAt      DateTime?         // 24h after created for anon sessions
  createdAt      DateTime          @default(now())

  @@index([sessionToken])
  @@index([userId])
  @@index([status, expiresAt])
  @@map("test_sessions")
}

model NewsletterSubscriber {
  id             String           @id @default(cuid())
  email          String           @unique
  status         NewsletterStatus @default(pending)
  confirmToken   String?          @unique
  confirmedAt    DateTime?
  source         String?          // 'footer'|'blog_inline'|'test_result'|'popup'
  sourceSlug     String?
  tags           String[]
  ipHash         String?
  createdAt      DateTime         @default(now())
  unsubscribedAt DateTime?

  @@map("newsletter_subscribers")
}

model LeadSubmission {
  id          String   @id @default(cuid())
  type        LeadType
  fullName    String
  email       String
  companyName String?
  companySize String?
  phone       String?
  message     String?
  intent      String?  // 'demo'|'pricing'|'partnership'|'other'
  utmSource   String?
  utmMedium   String?
  utmCampaign String?
  status      String   @default("new") // 'new'|'contacted'|'qualified'|'closed'
  ipHash      String?
  createdAt   DateTime @default(now())

  @@map("lead_submissions")
}

model CooperationRequest {
  id                    String   @id @default(cuid())
  fullName              String
  email                 String
  phone                 String?
  specialization        String?
  licenseInfo           String?
  message               String?
  status                String   @default("new")
  convertedTherapistId  String?
  createdAt             DateTime @default(now())

  @@map("cooperation_requests")
}

model FaqItem {
  id          String   @id @default(cuid())
  question    String
  answer      String
  category    String?
  sortOrder   Int      @default(0)
  isPublished Boolean  @default(true)
  createdAt   DateTime @default(now())

  @@map("faq_items")
}

model ContentPage {
  id             String   @id @default(cuid())
  slug           String   @unique
  title          String
  blocks         Json     @default("[]")
  seoTitle       String?
  seoDescription String?
  ogImageUrl     String?
  isPublished    Boolean  @default(false)
  publishedAt    DateTime?
  updatedAt      DateTime @updatedAt

  @@map("content_pages")
}

model MediaAsset {
  id           String   @id @default(cuid())
  storagePath  String
  publicUrl    String
  filename     String
  mimeType     String
  sizeBytes    Int?
  width        Int?
  height       Int?
  altText      String?
  blurhash     String?
  uploadedById String?
  uploadedBy   Profile? @relation(fields: [uploadedById], references: [id])
  createdAt    DateTime @default(now())

  @@map("media_assets")
}
```

### Supabase Row-Level Security

Prisma is the primary data access layer. Supabase RLS is a defense-in-depth layer. Critical policies:

```sql
-- PHI: session_notes — only therapist who wrote it, or admin
ALTER TABLE session_notes ENABLE ROW LEVEL SECURITY;
CREATE POLICY "therapist_owns_notes"
  ON session_notes FOR ALL
  USING (therapist_id = auth.uid()
    OR EXISTS (SELECT 1 FROM profiles WHERE id = auth.uid() AND role IN ('admin','super_admin')));

-- Public: blog_posts — published only
ALTER TABLE blog_posts ENABLE ROW LEVEL SECURITY;
CREATE POLICY "public_published_posts"
  ON blog_posts FOR SELECT
  USING (status = 'published' AND published_at <= now());

-- Public: tests — published only
CREATE POLICY "public_published_tests"
  ON tests FOR SELECT
  USING (status = 'published' AND published_at <= now());

-- test_sessions: own by token or user_id
CREATE POLICY "own_test_session"
  ON test_sessions FOR ALL
  USING (session_token = current_setting('app.session_token', true)
    OR user_id = auth.uid());

-- newsletter_subscribers: INSERT only for anon
CREATE POLICY "newsletter_insert_only"
  ON newsletter_subscribers FOR INSERT WITH CHECK (true);
```

---

### 5. Domain Boundaries

| Domain | Owns | Package | Depends On |
|---|---|---|---|
| **Identity & Auth** | Clerk profile sync, `profiles` | `@repo/auth` | — |
| **Therapist Onboarding** | `therapist_profiles`, credentials, review queue | `@repo/database` | Identity |
| **Scheduling** | `availability_schedules`, `exceptions`, slot computation | `@repo/database` | Identity, Therapist |
| **Sessions** | `sessions`, `bookings`, Daily.co rooms | `@repo/database` | Scheduling |
| **Clinical Notes** | `session_notes` (encrypted) | `@repo/database` | Sessions, Identity |
| **Messaging** | `conversations`, `messages` (encrypted) | `@repo/realtime` | Identity |
| **Payments (B2C)** | `payments`, `payouts`, Stripe Connect | `@repo/payments` | Scheduling |
| **B2B** | `organizations`, `org_members`, `org_invoices` | `@repo/payments`, `@repo/database` | Identity, Payments |
| **Disputes** | `disputes`, refund logic | `@repo/payments` | Payments, Sessions |
| **Notifications** | In-app (Knock) + email (Resend) | `@repo/notifications`, `@repo/email` | All domains |
| **Reviews** | `reviews`, moderation | `@repo/database` | Sessions |
| **Admin** | `admin_audit_logs`, `platform_settings`, tools | `apps/admin` | All domains |
| **Public / CMS** | Blog, tests, leads, newsletter | `@repo/cms`, `@repo/security` | Identity (optional) |
| **Analytics** | PostHog, materialized views | `@repo/analytics` | All domains |

---

### 6. Sensitive Data Inventory

| Field / Context | Classification | Handling |
|---|---|---|
| `session_notes.content` | PHI (HIPAA) | AES-256 encrypt before Prisma INSERT, decrypt on SELECT |
| `messages.content` | PHI | AES-256 encrypt before Prisma INSERT |
| `client_profiles.dateOfBirth` | PII | Encrypted column |
| `client_profiles.emergencyContact*` | PII | Encrypted column |
| `client_profiles.insuranceMemberId` | PHI | Encrypted column |
| `therapist_profiles.licenseNumber` | PII | Encrypted column |
| `therapist_profiles.stripeAccountId` | Financial PII | Prisma field-level access control + RLS |
| `organizations.ssoConfig` | Security credential | Application-layer encrypt before Prisma INSERT |
| `organizations.stripeCustomerId` | Financial PII | Restricted access |
| `payments.stripePaymentIntentId` | Financial | Never logged; `@repo/payments` only |
| `sessions.recordingUrl` | PHI | Consent required; Supabase Storage signed URL (1h) |
| `test_sessions.sessionToken` | Pseudo-PII | httpOnly cookie; never logged; 24h auto-purge for anon |
| `newsletter_subscribers.email` | PII | Service-role only SELECT; no enumeration in API |
| Video stream | PHI | Daily.co DTLS-SRTP E2E encrypted |
| Credential documents | PII | Supabase Storage private bucket; 15-min signed URL |
| `admin_audit_logs` | Operational | Immutable; read-only to super_admin; 7-year retention |
| `ip_hash` (all tables) | Pseudonymous | SHA-256 without reversible dictionary |

**⚠️ HIPAA:** BAAs required with Supabase (Pro+), Daily.co, Resend, PostHog before US production.

**⚠️ GDPR:** `test_sessions` anonymous data + `session_token` may constitute health data. 24h purge mandatory. Double opt-in mandatory for EU newsletter subscribers.

---

### 7. RBAC Matrix

| Resource / Action | CLIENT | B2B_MEMBER | THERAPIST | ADMIN (support) | ADMIN (finance) | ADMIN (operations) | SUPER_ADMIN |
|---|---|---|---|---|---|---|---|
| View own profile | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Edit own profile | ✅ | ✅ | ✅ | ❌ | ❌ | ✅ | ✅ |
| View other user profiles | ❌ | ❌ | own clients | ✅ read | ✅ read | ✅ | ✅ |
| Suspend / ban user | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ |
| Book session | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Confirm / cancel booking | ❌ | ❌ | own only | ✅ | ❌ | ✅ | ✅ |
| Create session note | ❌ | ❌ | own sessions | ❌ | ❌ | ❌ | ❌ |
| Read session notes | own (if shared) | own (if shared) | own only | ✅ audit | ❌ | ✅ | ✅ |
| Send message | ✅ | ✅ | own clients | ❌ | ❌ | ❌ | ❌ |
| Set availability | ❌ | ❌ | own only | ❌ | ❌ | ❌ | ❌ |
| Approve therapist | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ |
| View payments | own only | own only | own only | ✅ read | ✅ | ✅ | ✅ |
| Issue refund | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ |
| Manage platform settings | ❌ | ❌ | ❌ | ❌ | partial | ✅ | ✅ |
| Manage admin accounts | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |
| Create B2B organization | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ |
| View org usage reports | ❌ | org_admin only | ❌ | ✅ | ✅ | ✅ | ✅ |
| View audit logs | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ read | ✅ |
| Execute tests | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Manage blog / tests (CMS) | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ |
| View lead submissions | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ | ✅ |

---

## PHASE 1 — CORE PLATFORM {#phase-1}

---

### 8. Pages & Routes

#### `apps/web` — Public Routes (althena.com)
```
/                                 → Landing page
/about
/pricing
/for-business
/cooperation
/therapists                       → Public directory
/therapists/[slug]
/blog
/blog/[slug]
/blog/category/[categorySlug]
/tests
/tests/[slug]
/tests/[slug]/start
/tests/[slug]/question/[step]
/tests/[slug]/result
/sign-in                          → Clerk <SignIn />
/sign-up                          → Clerk <SignUp />
/sign-up/client                   → post-signup role selection
/sign-up/therapist
/sign-up/b2b                      → invite token accept
/terms · /privacy · /cookies
```

#### `apps/app` — Authenticated Routes (app.althena.com)

**Client:**
```
/                                 → Redirect → /home
/home
/find-therapist
/therapists/[slug]
/bookings
/bookings/[bookingId]
/messages
/messages/[conversationId]
/billing
/settings
/settings/preferences
/settings/organization            ← B2B only
/settings/organization/members
/settings/organization/reports
/settings/organization/billing
```

**Therapist:**
```
/overview
/calendar
/clients
/clients/[clientId]
/sessions
/sessions/[bookingId]
/sessions/[bookingId]/note
/messages
/messages/[convoId]
/earnings
/analytics
/settings
/settings/availability
/settings/payout
/settings/notifications
/onboarding
/onboarding/profile
/onboarding/credentials
/onboarding/availability
/onboarding/payout
```

**Video (isolated layout):**
```
/video/[roomId]                   → Daily.co session room, no sidebar
```

#### `apps/admin` — Admin Routes (admin.althena.com)
```
/overview
/users · /users/[userId]
/therapists · /therapists/[id] · /therapists/pending
/clients
/sessions · /sessions/[bookingId]
/payments · /payments/[paymentId]
/payouts
/disputes · /disputes/[disputeId]
/organizations · /organizations/new
/organizations/[orgId]
/organizations/[orgId]/members · /billing
/reviews
/analytics · /analytics/revenue · /sessions · /therapists · /b2b
/settings · /settings/general · /commission · /payments · /email
/settings/notifications · /b2b · /compliance
/blog · /blog/new · /blog/[postId]
/tests · /tests/new · /tests/[testId]
/media
/leads · /leads/[leadId]
/admins                           ← super_admin only
/admins/new · /admins/[adminId]
/audit-log
```

#### `apps/api` — API Webhook Routes (api.althena.com)
```
GET  /health
POST /webhooks/clerk              → Clerk user sync → prisma upsert
POST /webhooks/stripe             → payment/payout events → @repo/webhooks verify
POST /webhooks/daily              → Daily.co session events
```

---

### 9. Sidebar Navigation

#### 9.1 Client Sidebar (`apps/app` — client role)
```
🌿 Althena          [Clerk UserButton]
─────────────────────────────────
🏠  Home
🔍  Find a Therapist
📅  My Sessions
💬  Messages              [unread count — realtime]
💳  Billing
─────────────────────────────────
⚙️  Settings
❓  Help & Support
─────────────────────────────────
Footer (B2B member only):
🏢  [Org Name] Plan · 2 / 4 sessions used
```

#### 9.2 Therapist Sidebar
```
🌿 Althena          [Clerk UserButton]
                    ● Online ▾
─────────────────────────────────
MAIN
📊  Overview
📅  Calendar
👥  My Clients
🎯  Sessions
💬  Messages              [unread count]
─────────────────────────────────
FINANCE
💰  Earnings
📈  Analytics
─────────────────────────────────
ACCOUNT
⚙️  Settings
   ↳ Profile · Availability · Payout · Notifications
─────────────────────────────────
❓  Help · 🚪 Log Out
```

Status toggle (avatar menu): 🟢 Active · 🟡 On Leave · ⛔ Do Not Disturb

#### 9.3 Admin Sidebar (`apps/admin`)
```
🌿 Althena ADMIN     [Clerk UserButton]
Role: Operations Admin
─────────────────────────────────
OVERVIEW       📊 Dashboard
USER MGMT      👥 All Users · 🩺 Therapists [pending badge] · 👤 Clients
OPERATIONS     📅 Sessions · ⚠️ Disputes [badge] · ⭐ Reviews
FINANCE        💳 Payments · 💸 Payouts
B2B            🏢 Organizations
CONTENT        📝 Blog · 🧪 Tests · 📥 Leads
ANALYTICS      📈 Revenue · Sessions · Therapists · B2B
PLATFORM       ⚙️ Settings · 👮 Admins (super_admin only) · 📋 Audit Log
─────────────────────────────────
🚪 Log Out
```

Admin sidebar visibility by tier: same as previously specified (support/moderator/finance/operations/super_admin columns).

---

### 10. Forms

All forms use **react-hook-form** + **Zod** schemas. Validation shared between client and server via the same Zod schema imported from `@repo/database` or a dedicated `@repo/validators` package.

#### 10.1 Therapist Registration
```typescript
const TherapistRegistrationSchema = z.object({
  email:          z.string().email().max(254),
  password:       z.string().min(8).regex(/^(?=.*[A-Z])(?=.*\d)(?=.*[!@#$%^&*])/),
  confirmPassword: z.string(),
  fullName:       z.string().min(2).max(100),
  acceptTerms:    z.literal(true),
  acceptPrivacy:  z.literal(true),
}).refine(d => d.password === d.confirmPassword, {
  message: 'Passwords do not match', path: ['confirmPassword'],
});
```

#### 10.2 Therapist Profile Setup
```typescript
const TherapistProfileSchema = z.object({
  displayName:       z.string().min(2).max(60),
  bio:               z.string().min(50).max(800),
  specializations:   z.array(z.string()).min(1),
  approaches:        z.array(z.string()).min(1),
  languages:         z.array(z.string()).min(1),
  yearsOfExperience: z.number().int().min(0).max(50),
  sessionRateCents:  z.number().int().min(3000).max(50000),
  sessionTypes:      z.array(z.enum(['video','audio','chat'])).min(1),
  timezone:          z.string().min(1),
  phone:             z.string().regex(/^\+?[\d\s\-().]{7,20}$/).optional(),
  acceptsInsurance:  z.boolean(),
  insuranceProviders: z.array(z.string()).optional(),
});
```

#### 10.3 Credential Upload
```typescript
const CredentialSchema = z.object({
  licenseNumber:   z.string().min(4).max(20).regex(/^[A-Z0-9]+$/i),
  licenseState:    z.string().length(2),       // ISO 3166-2 US state code
  licenseExpiry:   z.date().min(new Date()),
  npiNumber:       z.string().length(10).optional(),
  // Files validated via Zod .instanceof(File) on client,
  // mime + size enforced on server via @repo/storage
});
```

#### 10.4 Availability Schedule
```typescript
const AvailabilitySlotSchema = z.object({
  dayOfWeek:          z.number().int().min(0).max(6),
  startTime:          z.string().regex(/^([0-1]\d|2[0-3]):[0-5]\d$/),
  endTime:            z.string().regex(/^([0-1]\d|2[0-3]):[0-5]\d$/),
  slotDurationMinutes: z.enum(['30','50','60','90']).transform(Number),
}).refine(d => d.endTime > d.startTime, { message: 'endTime must be after startTime' });
```

#### 10.5 Session Note
```typescript
const SessionNoteSchema = z.object({
  noteType:          z.enum(['soap','progress','intake','discharge']),
  content:           z.string().min(20).max(10000),
  isSharedWithClient: z.boolean().default(false),
  // SOAP subfields (required when noteType === 'soap')
  subjective:  z.string().optional(),
  objective:   z.string().optional(),
  assessment:  z.string().optional(),
  plan:        z.string().optional(),
});
```

#### 10.6 Create Organization (Admin)
```typescript
const CreateOrganizationSchema = z.object({
  name:                    z.string().min(2).max(100),
  domain:                  z.string().optional(),
  planType:                z.enum(['per_seat','credit_pack','unlimited','enterprise']),
  seatsPurchased:          z.number().int().min(1).max(10000).optional(),
  sessionsPerSeatPerMonth: z.number().int().min(1).max(20).optional(),
  billingEmail:            z.string().email(),
  billingContactName:      z.string().min(2),
  contractStart:           z.date(),
  contractEnd:             z.date().optional(),
  stripeCustomerId:        z.string().optional(),
});
```

#### 10.7 Invite Organization Member (bulk)
```typescript
const InviteMemberSchema = z.object({
  emails: z.array(z.string().email()).min(1).max(50),
  role:   z.enum(['member','org_admin']).default('member'),
});
```

#### 10.8 Platform Settings (Admin)
```typescript
const PlatformSettingsSchema = z.object({
  commissionRatePercent:    z.number().min(0).max(50).multipleOf(0.5),
  cancellationWindowHours:  z.number().int().min(1).max(72),
  cancellationFeePercent:   z.number().min(0).max(100),
  therapistMinRateCents:    z.number().int().min(0),
  therapistMaxRateCents:    z.number().int().min(0),
  payoutSchedule:           z.enum(['daily','weekly','biweekly','monthly']),
  sessionDurationOptions:   z.array(z.number().int()),
  defaultSessionDuration:   z.number().int(),
  supportEmail:             z.string().email(),
  platformName:             z.string().min(1).max(100),
});
```

#### 10.9 Dispute Resolution (Admin)
```typescript
const DisputeResolutionSchema = z.object({
  status:           z.enum(['resolved_refund','resolved_no_action']),
  resolutionNote:   z.string().min(20),
  refundType:       z.enum(['full','partial']).optional(),
  refundAmountCents: z.number().int().min(1).optional(),
}).refine(d => {
  if (d.status === 'resolved_refund' && !d.refundType) return false;
  if (d.refundType === 'partial' && !d.refundAmountCents) return false;
  return true;
});
```

#### 10.10 Admin Invite (Super Admin)
```typescript
const AdminInviteSchema = z.object({
  email:    z.string().email(),
  fullName: z.string().min(2).max(100),
  tier:     z.enum(['support','moderator','finance','operations']),
});
```

---

### 11. Tables & Lists

All list views use **cursor-based pagination** (no OFFSET). Implemented via Prisma `cursor` + `take`. Filtering via URL search params with Zod coercion.

#### 11.1 Therapist List (`/therapists`)
Columns: Avatar + Name | Specializations | Status | Sessions | Avg Rating | Joined | Actions
Filters: Status (multi), Specialization, Language, Verified/Unverified
Sorting: `joinedAt` desc (default), `fullName`, `totalSessionsCompleted`, `avgRating`
Actions: View · Approve · Suspend · Message

#### 11.2 Pending Verification Queue (`/therapists/pending`)
Columns: Name | License # | State | Submitted | Days Waiting | Actions
Sorting: `createdAt` asc (oldest first)
Actions: Review (credential modal) · Approve · Reject

#### 11.3 All Users (`/users`)
Columns: Avatar + Name | Role | Email | Status | Last Active | Joined | Actions
Filters: Role, Status, Joined date range
Search: full-text on `fullName`, `email`

#### 11.4 All Sessions (`/sessions`)
Columns: Date/Time | Client | Therapist | Type | Duration | Status | Amount | Actions
Filters: Status, Type, Date range, Amount range

#### 11.5 Payments (`/payments`)
Columns: Date | Client | Therapist | Amount | Platform Fee | Status | Stripe ID | Actions
Filters: Status, Date range, Amount range, Org (B2B)
Summary row: Total revenue, total fees, total refunded (filtered period)

#### 11.6 Disputes Queue (`/disputes`)
Columns: ID | Client | Therapist | Booking Date | Reason (truncated) | Status | Opened | Actions
Filters: Status (open/under_review/resolved)
Sorting: `createdAt` asc (oldest open first)

#### 11.7 Organizations (`/organizations`)
Columns: Logo + Name | Plan | Seats | Active Members | Sessions This Month | MRR | Status | Actions
Filters: Plan type, Status

#### 11.8 Org Members (`/organizations/[orgId]/members`)
Columns: Name | Email | Role | Status | Sessions Used | Remaining | Joined | Actions
Actions: Deactivate · Make org_admin · Resend invite

#### 11.9 Audit Log (`/audit-log`)
Columns: Timestamp | Admin | Action | Target Type | Target ID | IP Address | Details
Sorting: `createdAt` desc (immutable)
Export: CSV (Server Action → Prisma query → stream)

#### 11.10 Reviews (`/reviews`)
Columns: Date | Client | Therapist | Rating | Comment (truncated) | Public | Flagged | Actions
Filters: Flagged only, Rating range, Public/Private
Actions: Unflag · Remove (soft delete) · Make private

#### 11.11 Lead Submissions (`/leads`)
Columns: Date | Name | Email | Company | Type | Intent | Status | Actions
Filters: Type, Status, Date range
Actions: Update status · View detail · Convert to org

#### 11.12 Blog Posts (`/blog`)
Columns: Title | Category | Status | Author | Published | Views | Actions
Filters: Status, Category, Author
Actions: Edit · Publish/Unpublish · Archive · Delete

---

### 12. API Endpoints

#### `apps/app` — Authenticated REST API (Next.js Route Handlers)

All routes require Clerk session. Role check via `@repo/auth` middleware helper.

```
Auth (handled by Clerk — no custom endpoints needed for login/logout)
POST /api/webhooks/clerk           → apps/api (role sync to Prisma)

Profiles
GET    /api/me
PATCH  /api/me
DELETE /api/me                     → Clerk account delete + Prisma soft-delete

Therapist (public browse — no auth)
GET    /api/therapists             → filter, search, paginate
GET    /api/therapists/[id]        → public profile
GET    /api/therapists/[id]/reviews
GET    /api/therapists/me          → own profile (therapist auth)
PATCH  /api/therapists/me

Availability
GET    /api/availability/[therapistId]
POST   /api/availability
DELETE /api/availability/[id]
GET    /api/availability/[therapistId]/slots?date=YYYY-MM-DD
POST   /api/availability/exceptions
DELETE /api/availability/exceptions/[id]

Bookings
GET    /api/bookings
POST   /api/bookings               → B2B coverage check → Stripe Payment Intent
GET    /api/bookings/[id]
PATCH  /api/bookings/[id]/confirm
PATCH  /api/bookings/[id]/cancel
PATCH  /api/bookings/[id]/no-show

Sessions
POST   /api/sessions/[bookingId]/start  → Daily.co room creation
PATCH  /api/sessions/[bookingId]/end
GET    /api/sessions/[bookingId]

Session Notes
GET    /api/notes?bookingId=[id]
POST   /api/notes                  → encrypt before Prisma INSERT
PATCH  /api/notes/[id]
DELETE /api/notes/[id]

Messages
GET    /api/conversations
POST   /api/conversations
GET    /api/conversations/[id]/messages
POST   /api/conversations/[id]/messages  → encrypt before INSERT
PATCH  /api/conversations/[id]/read

Clients (Therapist view)
GET    /api/clients
GET    /api/clients/[id]
GET    /api/clients/[id]/sessions
GET    /api/clients/[id]/notes

Payments
POST   /api/payments/intent        → Stripe Payment Intent
POST   /api/stripe/connect/onboard → Stripe Connect Express onboarding link
GET    /api/stripe/connect/status
GET    /api/payouts
GET    /api/payouts/[id]

Notifications
GET    /api/notifications
PATCH  /api/notifications/[id]/read
PATCH  /api/notifications/read-all

Analytics (Therapist)
GET    /api/analytics/overview
GET    /api/analytics/sessions
GET    /api/analytics/clients

Disputes
POST   /api/disputes               → client raises dispute
GET    /api/disputes/[id]
PATCH  /api/disputes/[id]/resolve  → admin only

Reviews
POST   /api/reviews                → client only, post-session
GET    /api/reviews?therapistId=[id]
PATCH  /api/reviews/[id]/flag

B2B Organizations
GET    /api/organizations/[id]
PATCH  /api/organizations/[id]
GET    /api/organizations/[id]/members
POST   /api/organizations/[id]/members/invite
PATCH  /api/organizations/[id]/members/[memberId]
DELETE /api/organizations/[id]/members/[memberId]
GET    /api/organizations/[id]/usage
GET    /api/organizations/[id]/invoices
GET    /api/organizations/[id]/therapist-pool
POST   /api/organizations/[id]/therapist-pool
DELETE /api/organizations/[id]/therapist-pool/[therapistId]
```

#### `apps/admin` — Admin API

All require admin JWT + TOTP 2FA session check.

```
GET/PATCH  /api/admin/users/[id] + /suspend + /restore + DELETE
GET/PATCH  /api/admin/therapists/[id] + /approve + /reject + /suspend
GET        /api/admin/therapists/pending
GET        /api/admin/sessions · /payments · /payouts
PATCH      /api/admin/payouts/[id]/retry
GET/PATCH  /api/admin/disputes/[id]/resolve
GET/PATCH  /api/admin/reviews/[id]/remove + /unflag
GET/POST/GET /api/admin/organizations + /[id]
GET        /api/admin/analytics/overview + /revenue + /sessions + /therapists + /b2b
GET/PATCH  /api/admin/settings
GET        /api/admin/audit-log
-- CMS
GET/POST/PATCH/DELETE /api/admin/blog + /[id]/publish + /unpublish
GET/POST/PATCH        /api/admin/tests + /[id]
GET/PATCH             /api/admin/leads + /[id]/status
POST                  /api/admin/media/upload
-- Super admin only
GET/POST/PATCH/DELETE /api/admin/admins + /invite + /[id]/tier
```

#### `apps/api` — Webhooks

```
GET  /health                       → 200 { status: 'ok' }
POST /webhooks/clerk               → verify Svix signature → sync profile to Prisma
POST /webhooks/stripe              → verify Stripe signature → update payments/payouts
POST /webhooks/daily               → Daily.co session events → update sessions table
```

#### Public API (in `apps/web`)

```
POST /api/public/newsletter
GET  /api/public/newsletter/confirm?token=[token]
GET  /api/public/newsletter/unsubscribe?token=[token]
POST /api/public/leads
POST /api/public/cooperation
POST /api/public/tests/[testId]/session
GET  /api/public/tests/[testId]/session?token=[token]
PATCH /api/public/tests/[testId]/answer
POST /api/public/tests/[testId]/complete
POST /api/public/tests/[testId]/claim     ← auth required
POST /api/public/blog/[slug]/view         ← fire-and-forget
GET  /api/public/faq?category=[category]
```

---

### 13. Real-time Functionality

**Provider:** Supabase Realtime via `@repo/realtime`

```typescript
// packages/realtime/src/channels.ts
import { createClient } from '@supabase/supabase-js';

// Per-conversation messages
export function messageChannel(conversationId: string, handler: (msg: Message) => void) {
  return supabase
    .channel(`conversation:${conversationId}`)
    .on('postgres_changes', {
      event: 'INSERT', table: 'messages',
      filter: `conversation_id=eq.${conversationId}`
    }, handler);
}

// Per-user notifications (augments Knock — fallback)
export function notificationChannel(userId: string, handler: (n: Notification) => void) {
  return supabase
    .channel(`notifications:${userId}`)
    .on('postgres_changes', {
      event: 'INSERT', table: 'notifications',
      filter: `user_id=eq.${userId}`
    }, handler);
}

// Booking updates (therapist)
export function bookingChannel(therapistId: string, handler: (b: Booking) => void) {
  return supabase
    .channel(`bookings:${therapistId}`)
    .on('postgres_changes', {
      event: 'UPDATE', table: 'bookings',
      filter: `therapist_id=eq.${therapistId}`
    }, handler);
}

// Admin queue badges
export function adminQueueChannel(handlers: { dispute: () => void; pending: () => void }) {
  return supabase
    .channel('admin:queues')
    .on('postgres_changes', { event: '*', table: 'disputes' }, handlers.dispute)
    .on('postgres_changes', {
      event: '*', table: 'therapist_profiles',
      filter: `onboarding_status=eq.pending`
    }, handlers.pending);
}

// Therapist presence
export function presenceChannel(userId: string, status: string) {
  return supabase
    .channel('presence:therapists')
    .on('presence', { event: 'sync' }, () => {})
    .track({ userId, status });
}
```

**Video:** Daily.co per-booking rooms. Room created server-side on `POST /api/sessions/[bookingId]/start`. Both parties receive short-lived Daily.co JWT tokens. `/video/[roomId]` is a full-screen isolated layout in `apps/app` — no sidebar chrome.

**Notifications:** Primary delivery via `@repo/notifications` (Knock). Supabase Realtime is the fallback for in-app badge updates.

**Scheduled jobs (Supabase Edge Functions):**

| Job | Schedule | Action |
|---|---|---|
| Booking reminder | Cron: every hour | T-24h, T-1h emails via Resend |
| Post-session review | Cron: every hour | T+1h review request to client |
| Weekly payout notification | Cron: Monday 09:00 | Payout summary to therapist |
| B2B invoice generation | Cron: 1st of month 06:00 | Aggregate bookings → Stripe Invoice |
| Org usage report | Cron: 1st of month 07:00 | Compute → `organization_usage_reports` |
| Anonymous session purge | Cron: daily 03:00 | DELETE test_sessions WHERE anon + expired |

---

### 14. File Functionality

**Provider:** `@repo/storage` wraps Supabase Storage

| File Type | Bucket | Access | Max Size | CDN | Processing |
|---|---|---|---|---|---|
| Avatar | `avatars` | Public | 5 MB | ✅ | Sharp → 400×400 WebP |
| Credential docs | `credentials` | Private, signed URL 15 min | 10 MB | ❌ | None |
| Message attachments | `attachments` | Private, signed URL 1h | 20 MB | ❌ | None |
| Session recordings | `recordings` | Private, signed URL 1h | 2 GB | ❌ | None |
| Org logos | `org-assets` | Public | 2 MB | ✅ | Sharp → 200×200 WebP |
| Blog/test covers | `public-media` | Public | 5 MB | ✅ | Sharp → 400w/800w/1200w WebP + 1200×630 OG |
| OG images | `public-media/og` | Public | 2 MB | ✅ | next/og ImageResponse (Edge) |
| Email templates | `email-templates` | Admin-only | 500 KB | ❌ | None |
| Platform assets | `platform-assets` | Public | 10 MB | ✅ | None |

**Upload flow:**
1. Client requests presigned URL from `@repo/storage`
2. Client uploads directly to Supabase Storage
3. Supabase Edge Function triggered on upload → Sharp processing → variants stored
4. `media_assets` record created via Prisma with `blurhash` (4×3 components)
5. API responses reference signed URLs fetched at read time — never raw storage URLs

---

### 15. Role-Based Navigation Components

```typescript
// packages/design-system/src/providers.tsx
export function DesignSystemProvider({ children, role }: Props) {
  return (
    <ClerkProvider>
      <ThemeProvider>
        <RoleProvider role={role}>
          {children}
        </RoleProvider>
      </ThemeProvider>
    </ClerkProvider>
  );
}

// Each app composes its own shell
// apps/app — ClientShell or TherapistShell based on Clerk publicMetadata.role
// apps/admin — AdminShell with tier-filtered nav
```

**Clerk Auth Middleware (shared in `@repo/auth`):**

```typescript
// packages/auth/src/middleware.ts
import { clerkMiddleware, createRouteMatcher } from '@clerk/nextjs/server';

const isPublicRoute = createRouteMatcher([
  '/', '/about', '/pricing', '/for-business', '/cooperation',
  '/therapists(.*)', '/blog(.*)', '/tests(.*)',
  '/sign-in(.*)', '/sign-up(.*)',
  '/api/public(.*)',
  '/terms', '/privacy', '/cookies',
]);

const isAdminRoute = createRouteMatcher(['/admin(.*)']);
const isTherapistRoute = createRouteMatcher(['/dashboard(.*)']);

export default clerkMiddleware(async (auth, req) => {
  if (!isPublicRoute(req)) {
    await auth.protect();
  }

  const { sessionClaims } = await auth();
  const role = sessionClaims?.publicMetadata?.role as string;

  if (isAdminRoute(req) && !['admin', 'super_admin'].includes(role)) {
    return Response.redirect(new URL('/unauthorized', req.url));
  }
  if (isTherapistRoute(req) && role !== 'therapist') {
    return Response.redirect(new URL('/unauthorized', req.url));
  }
});

export const config = {
  matcher: ['/((?!_next|[^?]*\\.(?:html?|css|js(?!on)|jpe?g|webp|png|gif|svg|ttf|woff2?|ico|csv|docx?|xlsx?|zip|webmanifest)).*)', '/(api|trpc)(.*)'],
};
```

---

## PHASE 2 — PUBLIC SITE (Guest Domain) {#phase-2}

---

### 16. Public Domain Entities

See Section 4 Prisma schema for full DDL: `BlogPost`, `BlogAuthor`, `BlogCategory`, `Test`, `TestQuestion`, `TestSession`, `NewsletterSubscriber`, `LeadSubmission`, `CooperationRequest`, `FaqItem`, `ContentPage`, `MediaAsset`.

**CMS approach:** `@repo/cms` uses **Content Collections** (MDX files in `packages/cms/content/`) for blog posts and test content — provides full TypeScript type safety, no external CMS dependency. Admin CMS UI in `apps/admin` reads/writes both the Prisma DB (for metadata) and the content files (for rich content). For dynamic per-user data (test sessions, leads, newsletter), Prisma is used directly.

---

### 17. Route Architecture (`apps/web`)

```
apps/web/app/
├── (public)/
│   ├── layout.tsx                   ← PublicLayout (nav, footer, cookie banner)
│   ├── page.tsx                     → /                      [ISR: 3600s]
│   ├── about/page.tsx               → /about                 [ISR: 86400s]
│   ├── pricing/page.tsx             → /pricing               [ISR: 3600s]
│   ├── for-business/page.tsx        → /for-business          [ISR: 3600s]
│   ├── cooperation/page.tsx         → /cooperation           [ISR: 3600s]
│   ├── blog/
│   │   ├── layout.tsx               ← BlogLayout (sidebar)
│   │   ├── page.tsx                 → /blog                  [ISR: 60s]
│   │   ├── [slug]/page.tsx          → /blog/[slug]           [ISR: 300s, generateStaticParams top 100]
│   │   └── category/[slug]/page.tsx → /blog/category/[slug] [ISR: 60s]
│   ├── tests/
│   │   ├── page.tsx                 → /tests                 [ISR: 300s]
│   │   └── [slug]/
│   │       ├── page.tsx             → /tests/[slug]          [ISR: 300s, generateStaticParams]
│   │       ├── start/page.tsx       → /tests/[slug]/start    [SSR — session detection]
│   │       ├── question/[step]/page.tsx → /tests/[slug]/question/[step] [Client Component]
│   │       └── result/page.tsx      → /tests/[slug]/result   [SSR, noindex]
│   ├── therapists/
│   │   ├── page.tsx                 → /therapists            [SSR — filter params vary]
│   │   └── [slug]/page.tsx          → /therapists/[slug]     [ISR: 300s]
│   └── (static)/
│       ├── terms/page.tsx · privacy/page.tsx · cookies/page.tsx  [SSG]
│
├── api/public/                      ← Public Route Handlers (no auth)
│   ├── newsletter/route.ts
│   ├── newsletter/confirm/route.ts
│   ├── newsletter/unsubscribe/route.ts
│   ├── leads/route.ts
│   ├── cooperation/route.ts
│   ├── tests/[testId]/session/route.ts
│   ├── tests/[testId]/answer/route.ts
│   ├── tests/[testId]/complete/route.ts
│   ├── tests/[testId]/claim/route.ts   ← Clerk auth required
│   ├── blog/[slug]/view/route.ts
│   └── faq/route.ts
│
├── sitemap.ts                       → /sitemap.xml
├── robots.ts                        → /robots.txt
└── opengraph-image.tsx              → dynamic OG images via next/og
```

---

### 18. Public Forms

All use Zod schemas + Arcjet rate limiting (`@repo/security`). Honeypot field (`name="website"`, visually hidden via CSS) on every form.

#### 18.1 NewsletterForm
```typescript
const NewsletterSchema = z.object({
  email:       z.string().email().max(254),
  source:      z.enum(['footer', 'blog_inline', 'test_result', 'popup']),
  sourceSlug:  z.string().optional(),
  website:     z.string().max(0), // honeypot
});
// Arcjet: 3 requests/IP/hour
// Always return 200 { message } — never expose email existence
// Side effect: INSERT newsletter_subscribers, send Resend double opt-in email (EU mandatory)
```

#### 18.2 BusinessInquiryForm
```typescript
const BusinessInquirySchema = z.object({
  fullName:    z.string().min(2).max(100),
  email:       z.string().email().max(254),
  companyName: z.string().min(1).max(200),
  companySize: z.enum(['1-10','11-50','51-200','201-1000','1000+']),
  phone:       z.string().regex(/^\+?[\d\s\-().]{7,20}$/).optional(),
  intent:      z.enum(['demo','pricing','partnership','other']),
  message:     z.string().min(10).max(1000).optional(),
  utmSource:   z.string().max(100).optional(),
  utmMedium:   z.string().max(100).optional(),
  utmCampaign: z.string().max(100).optional(),
  website:     z.string().max(0),
});
// Arcjet: 5 requests/IP/hour
// On success: inline thank-you, no redirect
```

#### 18.3 CooperationForm
```typescript
const CooperationSchema = z.object({
  fullName:       z.string().min(2).max(100),
  email:          z.string().email().max(254),
  phone:          z.string().optional(),
  specialization: z.string().max(200).optional(),
  licenseInfo:    z.string().max(500).optional(),
  message:        z.string().min(20).max(1000),
  website:        z.string().max(0),
});
// Arcjet: 3 requests/IP/hour
```

---

### 19. Public API Endpoints

Rate limiting via `@repo/security` (Arcjet) on all `/api/public/*` routes.

```
Arcjet limits:
  POST /api/public/newsletter         → 3/IP/hour
  POST /api/public/leads              → 5/IP/hour
  POST /api/public/cooperation        → 3/IP/hour
  POST /api/public/tests/*/session    → 10/IP/hour
  PATCH /api/public/tests/*/answer    → 120/IP/hour

POST /api/public/newsletter
  Body: NewsletterSchema
  Response: 200 { message } (always 200, no enumeration)
  Side effect: Prisma INSERT, Resend confirmation email

GET  /api/public/newsletter/confirm?token=[token]
  Response: redirect to /?newsletter=confirmed

GET  /api/public/newsletter/unsubscribe?token=[token]
  Response: unsubscribe confirmation page

POST /api/public/leads
  Body: BusinessInquirySchema
  Response: 201 { id } | 400 | 429

POST /api/public/cooperation
  Body: CooperationSchema
  Response: 201 { id } | 400 | 429

POST /api/public/tests/[testId]/session
  Body: { sessionToken? }
  Response: 201 { sessionToken, currentStep, status }
  Side effect: Prisma INSERT, httpOnly cookie set (24h)

GET  /api/public/tests/[testId]/session?token=[token]
  Response: { id, currentStep, status, answersCount, expiresAt }

PATCH /api/public/tests/[testId]/answer
  Body: { sessionToken, questionId, answerValue }
  Response: 200 { nextStep }

POST /api/public/tests/[testId]/complete
  Body: { sessionToken }
  Response: 200 { score, resultBand, resultDescription, ctaVariant }
  Side effect: server-side score, Prisma UPDATE, INCREMENT completionCount

POST /api/public/tests/[testId]/claim
  Auth: Clerk required
  Body: { sessionToken }
  Response: 200 | 404 | 409 (already claimed)
  Side effect: UPDATE test_sessions SET userId = clerkUserId

POST /api/public/blog/[slug]/view
  Response: 200 (always; Prisma increment, non-blocking)

GET  /api/public/faq?category=[cat]
  Response: { items: FaqItem[] }
  Cache-Control: public, max-age=3600
```

---

### 20. SEO Architecture

Implemented via `@repo/seo`:

```typescript
// packages/seo/src/index.ts — exported helpers
export { createMetadata } from './metadata';
export { createJsonLd, blogPostJsonLd, testJsonLd, therapistJsonLd } from './json-ld';
export { createSitemap } from './sitemap';
```

#### Metadata Strategy

```typescript
// apps/web/app/(public)/blog/[slug]/page.tsx
import { createMetadata } from '@repo/seo';

export async function generateMetadata({ params }): Promise<Metadata> {
  const post = await prisma.blogPost.findUnique({ where: { slug: params.slug } });
  if (!post) return {};
  return createMetadata({
    title: post.seoTitle ?? `${post.title} | Althena`,
    description: post.seoDescription ?? post.excerpt ?? '',
    canonical: post.canonicalUrl ?? `https://althena.com/blog/${post.slug}`,
    openGraph: {
      type: 'article',
      images: [{ url: post.ogImageUrl ?? post.coverImageUrl ?? '', width: 1200, height: 630 }],
      publishedTime: post.publishedAt?.toISOString(),
    },
  });
}
```

#### JSON-LD Schema by Page

| Page | Schema Types |
|---|---|
| `/` | `WebSite` + `Organization` + `MedicalOrganization` |
| `/blog` | `WebPage` + `ItemList` |
| `/blog/[slug]` | `Article` or `MedicalWebPage` + `BreadcrumbList` |
| `/tests` | `WebPage` + `ItemList` |
| `/tests/[slug]` | `MedicalWebPage` + `FAQPage` |
| `/therapists/[slug]` | `Person` + `MedicalBusiness` + `BreadcrumbList` |
| `/for-business` | `WebPage` + `Service` |
| `/about` | `AboutPage` + `Organization` |

#### OG Image Generation (Edge Runtime)

```typescript
// apps/web/app/(public)/blog/[slug]/opengraph-image.tsx
import { ImageResponse } from 'next/og';
export const runtime = 'edge';
export default async function OGImage({ params }: { params: { slug: string } }) {
  const post = await prisma.blogPost.findUnique({ where: { slug: params.slug } });
  return new ImageResponse(
    <div style={{ display: 'flex', width: '100%', height: '100%', background: '#0f172a' }}>
      <span style={{ color: 'white', fontSize: 48 }}>{post?.title}</span>
    </div>,
    { width: 1200, height: 630 }
  );
}
```

#### Sitemap (`apps/web/app/sitemap.ts`)

```typescript
export default async function sitemap(): Promise<MetadataRoute.Sitemap> {
  const [posts, tests, therapists] = await Promise.all([
    prisma.blogPost.findMany({ where: { status: 'published' }, select: { slug: true, updatedAt: true } }),
    prisma.test.findMany({ where: { status: 'published' }, select: { slug: true, updatedAt: true } }),
    prisma.therapistProfile.findMany({ where: { onboardingStatus: 'active' }, select: { id: true } }),
  ]);
  return [
    { url: 'https://althena.com', changeFrequency: 'daily', priority: 1.0 },
    { url: 'https://althena.com/blog', changeFrequency: 'daily', priority: 0.9 },
    { url: 'https://althena.com/tests', changeFrequency: 'weekly', priority: 0.8 },
    ...posts.map(p => ({ url: `https://althena.com/blog/${p.slug}`, lastModified: p.updatedAt, priority: 0.7 })),
    ...tests.map(t => ({ url: `https://althena.com/tests/${t.slug}`, priority: 0.7 })),
    ...therapists.map(t => ({ url: `https://althena.com/therapists/${t.id}`, priority: 0.6 })),
    // EXCLUDED: /tests/*/start, /tests/*/question/*, /tests/*/result
    // EXCLUDED: /sign-in, /sign-up, /api/*
  ];
}
```

---

### 21. State Machines (Guest Domain)

See Section 2.9 (Test Execution), 2.10 (Newsletter), and 2.11 (Content Publishing) for full state machines.

#### Progressive Test Answer Saving

```typescript
// apps/web/app/(public)/tests/[slug]/question/[step]/page.tsx (Client Component)
const saveAnswer = useDebouncedCallback(async (questionId: string, value: number) => {
  try {
    await fetch(`/api/public/tests/${testId}/answer`, {
      method: 'PATCH',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ sessionToken, questionId, answerValue: value }),
    });
  } catch {
    // Queue in localStorage for retry
    const queue = JSON.parse(localStorage.getItem('althena_answer_queue') ?? '[]');
    queue.push({ questionId, value, timestamp: Date.now() });
    localStorage.setItem('althena_answer_queue', JSON.stringify(queue));
  }
}, 300);

// Flush queue on next answer or via sendBeacon on unload
window.addEventListener('beforeunload', () => {
  navigator.sendBeacon(`/api/public/tests/${testId}/answer`, JSON.stringify(pendingAnswer));
});
```

---

### 22. Frontend Architecture & Rendering Strategy

#### Server vs Client Component Decisions

| Component | Type | Reason |
|---|---|---|
| Blog post content | Server | No interactivity, SEO-critical, LCP |
| Blog share buttons | Client | `navigator.share` / clipboard |
| Newsletter form | Client | Form state, Arcjet via Route Handler |
| Test question step | Client | Interactive, localStorage, state machine |
| Test progress bar | Client | Local state |
| Therapist catalog filters | Client | URL search params manipulation |
| Therapist card grid | Server | SEO-critical |
| Cookie consent banner | Client | `localStorage` check |
| Lead capture form | Client | Form state, validation |
| FAQ accordion | Client | open/close state |
| Home hero | Server | Above-fold, LCP-critical |
| Clerk `<UserButton>` | Client | Clerk SDK requirement |

#### Rendering Strategy

| Route (apps/web) | Strategy | Notes |
|---|---|---|
| `/` | ISR 3600s | Home/marketing |
| `/blog` | ISR 60s | New posts appear within 60s |
| `/blog/[slug]` | ISR 300s + `generateStaticParams` | Top 100 by viewCount at build |
| `/tests` | ISR 300s | |
| `/tests/[slug]` | ISR 300s + `generateStaticParams` | All published tests |
| `/tests/[slug]/start` | SSR | Session detection (httpOnly cookie) |
| `/tests/[slug]/question/[step]` | Client only (`'use client'`) | Interactive |
| `/tests/[slug]/result` | SSR, `noindex` | Session-specific |
| `/therapists` | SSR | Filter params vary |
| `/therapists/[slug]` | ISR 300s | |
| `/for-business`, `/cooperation` | ISR 3600s | |
| `/about` | ISR 86400s | |
| `/terms`, `/privacy`, `/cookies` | SSG | Never changes |

#### Caching Layers

```
Browser:
  Static assets (JS/CSS): Cache-Control: public, max-age=31536000, immutable
  FAQ API:                 Cache-Control: public, max-age=3600

Vercel Edge Cache:
  ISR pages: CDN-served, invalidated by revalidatePath/revalidateTag on admin publish

Next.js Data Cache:
  Prisma queries in Server Components: tagged via next() options
  Invalidated on admin publish via revalidateTag('blog-posts'), revalidateTag('tests')

Supabase:
  Connection pooling via pgBouncer (poolerUrl in DATABASE_URL)
  Direct connection for migrations (DIRECT_URL)
  Slot computation: Edge Function result cached 5 min per therapist in Vercel KV

Turborepo Remote Cache:
  Build artifacts cached — CI/CD pipeline reuses unchanged package builds
```

---

### 23. Analytics & Tracking Architecture

Implemented via `@repo/analytics` which wraps PostHog + GA4 with consent gating.

#### Event Taxonomy

```typescript
// packages/analytics/src/events.ts
type AnalyticsEvent =
  | { event: 'page_viewed';             props: { page: string; referrer: string } }
  | { event: 'cta_clicked';            props: { ctaId: string; page: string; section: string } }
  | { event: 'article_viewed';         props: { slug: string; category: string } }
  | { event: 'article_scrolled';       props: { slug: string; depthPercent: 25|50|75|100 } }
  | { event: 'test_started';           props: { testSlug: string; category: string } }
  | { event: 'test_question_answered'; props: { testSlug: string; step: number; total: number } }
  | { event: 'test_abandoned';         props: { testSlug: string; step: number } }
  | { event: 'test_completed';         props: { testSlug: string; resultBand: string } }
  | { event: 'test_cta_clicked';       props: { testSlug: string; resultBand: string; ctaVariant: string } }
  | { event: 'newsletter_submitted';   props: { source: string } }
  | { event: 'newsletter_confirmed';   props: {} }
  | { event: 'lead_submitted';         props: { type: string; intent: string } }
  | { event: 'register_started';       props: { intent?: string; role: string } }
  | { event: 'therapist_viewed';       props: { therapistId: string; source: string } }
  | { event: 'booking_cta_clicked';    props: { therapistId: string; page: string } }
```

#### UTM Capture

```typescript
// packages/analytics/src/utm.ts
export function captureUTM(): void {
  const params = new URLSearchParams(window.location.search);
  const utm = (['source','medium','campaign','content','term'] as const)
    .reduce<Record<string, string>>((acc, key) => {
      const val = params.get(`utm_${key}`);
      if (val) acc[`utm_${key}`] = val;
      return acc;
    }, {});
  if (Object.keys(utm).length > 0) {
    sessionStorage.setItem('althena_utm', JSON.stringify(utm));
  }
}
export const getUTM = (): Record<string, string> =>
  JSON.parse(sessionStorage.getItem('althena_utm') ?? '{}');
```

#### Consent Tiers

```typescript
// packages/analytics/src/consent.ts
export type ConsentTier = 0 | 1 | 2; // 0: none, 1: analytics, 2: marketing

export function initAnalytics(tier: ConsentTier) {
  if (tier >= 1) {
    posthog.init(env.NEXT_PUBLIC_POSTHOG_KEY, {
      api_host: 'https://eu.posthog.com',  // EU data residency
      capture_pageview: true,
    });
  }
  if (tier >= 2) {
    // GA4 full tracking
    // Facebook Pixel (if used)
  }
}
```

---

## ADMIN PANEL DEEP DIVE {#admin-panel}

---

### 24. Admin Roles & Permission Architecture

- Only `super_admin` can invite new admins (via Clerk Organizations or custom invite flow)
- Invite flow: email + fullName + tier → Clerk invitation → first login → mandatory TOTP setup
- Session timeout: 4h inactivity (configured via Clerk session settings for admin app)
- All admin actions logged via Prisma to `AdminAuditLog` (immutable — no UPDATE/DELETE policies)
- Suspicious IP detected via Clerk's session security features → re-authentication
- `apps/admin` served on `admin.althena.com` with separate CSP headers in `next.config.ts`

---

### 25. Admin Settings Module

#### 25.1 General Settings
```typescript
interface GeneralSettings {
  platformName: string;
  platformTagline: string;
  platformLogo: File | string;
  supportEmail: string;
  supportPhone?: string;
  defaultTimezone: string;         // IANA
  defaultLanguage: string;         // ISO 639-1
  maintenanceMode: boolean;
  maintenanceBannerText?: string;
}
```

#### 25.2 Commission & Pricing
```typescript
interface CommissionSettings {
  defaultCommissionPercent: number;     // 0–50, step 0.5
  therapistMinRateCents: number;
  therapistMaxRateCents: number;
  cancellationWindowHours: number;      // 1–72
  lateCancellationFeePercent: number;   // 0–100
  noShowFeePercent: number;
}
```

#### 25.3 Payments & Payouts
```typescript
interface PaymentSettings {
  stripePublishableKey: string;         // displayed masked
  stripeWebhookSecret: string;          // masked, stored in Vercel env
  payoutSchedule: 'daily'|'weekly'|'biweekly'|'monthly';
  payoutMinimumCents: number;
  supportedCurrencies: string[];
  defaultCurrency: string;
}
```

#### 25.4 Email Templates
Editable via `apps/admin` CMS UI. Templates stored in `@repo/email` as React Email components with Handlebars-style variable slots. Admin-editable subject + HTML override stored in `PlatformSetting` as JSON.

Templates: `booking_confirmed` · `booking_cancelled` · `session_reminder_24h` · `session_reminder_1h` · `post_session_review_request` · `therapist_approved` · `therapist_rejected` · `payout_sent` · `dispute_opened` · `dispute_resolved` · `org_member_invite` · `password_reset`

Each has: subject · HTML body · plain text fallback · **Test send** button.

#### 25.5 B2B Configuration
```typescript
interface B2BSettings {
  b2bEnabled: boolean;
  defaultPlanTypes: OrgPlan[];
  contractTemplateUrl?: string;
  defaultSessionsPerSeat: number;
  orgTrialDays: number;
  usageReportFrequency: 'daily'|'weekly'|'monthly';
  privacyGuaranteeText: string;         // shown to employees
}
```

#### 25.6 Compliance
```typescript
interface ComplianceSettings {
  hipaaMode: boolean;                           // stricter logging, BAA flow
  dataRetentionDays: number;                    // default 2555 (7 years)
  sessionRecordingEnabled: boolean;
  sessionRecordingRequiresConsent: boolean;
  auditLogRetentionDays: number;                // min 2555 for HIPAA
  gdprMode: boolean;                            // data export / right-to-delete
  cookieConsentRequired: boolean;
}
```

---

### 26. Admin User Management

#### 26.1 Therapist Review Screen (`/therapists/[id]`)
- Profile overview (photo, bio, specializations)
- Credential panel: license number, state, expiry, Supabase Storage document viewer (signed URL 15 min, modal)
- Status history timeline: each status change + admin who acted + timestamp
- Actions: **Approve** | **Reject** (reason modal) | **Request More Info** (Knock message) | **Suspend**
- Sessions count, reviews summary, Stripe Connect status badge

#### 26.2 User Detail Screen (`/users/[id]`)
- Profile card: avatar, name, email, role, joined, last active, status
- Role-specific panel: therapist credentials OR client intake OR org membership
- Session history (last 10, paginated link)
- Payment history (last 10)
- Active disputes
- Audit trail for this user
- Actions: **Suspend** | **Restore** | **Delete (GDPR)** — soft-delete + anonymization | **Reset Password** (Clerk) | **Impersonate** (super_admin only — requires 2nd admin approval, logged to `AdminImpersonation`)

#### 26.3 Admin List (`/admins`) — super_admin only
Columns: Name | Email | Tier | 2FA Enabled | Last Login | Created | Actions
Actions: Edit Tier (Clerk publicMetadata update) · Suspend (Clerk org ban) · Remove
Create: "Invite Admin" → `AdminInviteSchema` → Clerk invitation

---

### 27. B2B Organizations Module

#### 27.1 Organization Detail Tabs
1. **Overview** — plan, contract dates, seats, active usage meter
2. **Members** — invite (bulk), deactivate, promote to org_admin
3. **Therapist Pool** — add/remove therapists from `OrganizationTherapistPool`
4. **Billing** — invoices list, plan config, Stripe customer portal link
5. **Usage Reports** — monthly anonymized utilization (k-anonymity ≥5 members enforced)
6. **Settings** — SSO (Clerk Organizations SAML/OIDC), custom branding, domain

#### 27.2 B2B Session Coverage Logic

```typescript
// packages/database/src/b2b.ts
export async function checkB2BCoverage(clientId: string, orgId: string) {
  const [member, org] = await Promise.all([
    prisma.organizationMember.findFirst({ where: { clientId, organizationId: orgId } }),
    prisma.organization.findUnique({ where: { id: orgId } }),
  ]);
  if (!member || !org) throw new Error('Member or org not found');
  return {
    isCovered: member.sessionsUsedThisMonth < (org.sessionsPerSeatPerMonth ?? 0),
    sessionsUsed: member.sessionsUsedThisMonth,
    sessionsLimit: org.sessionsPerSeatPerMonth ?? 0,
  };
  // isCovered = false → client pays out of pocket via Stripe
  // isCovered = true → booking.isCoveredByOrg = true → billed on org invoice
}
```

#### 27.3 Org Admin View (within `apps/app`)

```
/settings/organization          — plan overview, seat count, sessions used
/settings/organization/members  — invite/deactivate members
/settings/organization/reports  — anonymized usage (k-anon: suppress if <5 active)
/settings/organization/billing  — invoices, Stripe customer portal link
```

Org admins see only aggregate data — never individual session content, notes, or message threads.

#### 27.4 B2B Invoice Generation

Supabase Edge Function (cron: 1st of month, 06:00 UTC):

```typescript
// Idempotency key: `${orgId}:${periodStart}` passed to Stripe Invoice create
1. prisma.booking.findMany({ where: { isCoveredByOrg: true, organizationId, scheduledAt: { gte: periodStart, lt: periodEnd } } })
2. prisma.organizationInvoice.create({ data: { ... } })
3. stripe.invoices.create({ customer: org.stripeCustomerId, ... }) → @repo/payments
4. resend.send({ to: org.billingEmail, template: 'org_invoice' }) → @repo/email
5. prisma.organizationInvoice.update({ stripeInvoiceId: invoice.id })
```

#### 27.5 SSO Configuration (Clerk Organizations)

```typescript
// Clerk Organizations provides SAML/OIDC SSO out of the box
// Admin UI at apps/admin/app/organizations/[orgId]/settings
// On SSO setup: admin links Clerk Organization to Althena org record via clerkOrgId

interface SSOConfig {
  provider: 'saml' | 'oidc';
  // SAML — configured via Clerk dashboard
  entryPoint?: string;
  issuer?: string;
  cert?: string;          // SENSITIVE — stored in Clerk, not Prisma
  // OIDC — configured via Clerk dashboard
  clientId?: string;
  clientSecret?: string;  // SENSITIVE — stored in Clerk, not Prisma
  discoveryUrl?: string;
}
// On SSO login: Clerk auto-enrolls user → Clerk webhook fires → sync to OrganizationMember
```

---

## CROSS-CUTTING CONCERNS {#cross-cutting}

---

### Security

| Concern | Implementation |
|---|---|
| Authentication | Clerk (session tokens, TOTP 2FA for admins, hardware key for super_admin) |
| Authorization | Clerk `publicMetadata.role` + server-side checks in Route Handlers |
| PHI encryption | AES-256 at application layer before Prisma INSERT (notes, messages, PII fields) |
| Encryption keys | Vercel environment variables (not Prisma schema) — rotate via Vercel dashboard |
| Rate limiting | `@repo/security` (Arcjet) — bot detection + rate limiting on all public APIs |
| Webhook verification | `@repo/webhooks` (Svix) — Stripe + Clerk signature verification |
| Admin panel isolation | Separate `apps/admin` app on `admin.althena.com` with strict CSP |
| HIPAA audit trail | `AdminAuditLog` — immutable, 7-year retention, RLS `FOR SELECT` only |
| Test session privacy | `sessionToken` never logged; 24h auto-purge for anonymous sessions |
| SSO secrets | Stored in Clerk Organizations — never in Prisma / plaintext env |
| Admin impersonation | Requires 2nd admin approval + logged to `AdminImpersonation` |
| Credential documents | Supabase Storage private bucket, 15-min signed URLs, admin-only |
| Video E2E | Daily.co DTLS-SRTP |
| Headers | `X-Frame-Options: DENY`, `Strict-Transport-Security`, configured via `@repo/next-config` |

### Performance

| Concern | Implementation |
|---|---|
| Build caching | Turborepo Remote Cache — only rebuild changed packages |
| List views | Cursor-based Prisma pagination (no OFFSET) |
| Slot computation | Supabase Edge Function → Vercel KV cache (5 min per therapist) |
| Admin analytics | Postgres materialized views, refreshed hourly via Edge Function |
| Org usage reports | Precomputed nightly via Edge Function cron |
| Blog/test pages | ISR with `generateStaticParams` (top 100 by viewCount) |
| Images | Supabase CDN + `next/image` (AVIF/WebP) + blurhash placeholders |
| Blog view counter | Non-blocking `navigator.sendBeacon` → Prisma increment |
| DB connections | Supabase pgBouncer pooler (poolerUrl) + Prisma connection management |
| Bundle analysis | `@repo/next-config` with `withAnalyzer()` wrapper for CI bundle reports |
| Observability | `@repo/observability` (Sentry) — error tracking; BetterStack — log drain |

### Sprint Prioritization

| Sprint | Deliverable |
|---|---|
| 1 | Monorepo scaffold (Althena init), Turborepo, core packages skeleton |
| 2 | `@repo/database` Prisma schema + migrations, `@repo/auth` Clerk setup |
| 3 | `apps/app` + `apps/api` bootstrap, Clerk webhooks → profile sync |
| 4 | Therapist onboarding wizard (`apps/app`) + admin credential review (`apps/admin`) |
| 5 | Availability engine + slot computation (Edge Function + Vercel KV cache) |
| 6 | Booking flow + session state machine + Stripe Payment Intent |
| 7 | Video session integration (Daily.co) + `/video/[roomId]` isolated layout |
| 8 | Session notes (PHI encryption before Prisma INSERT) |
| 9 | Messaging + Supabase Realtime (`@repo/realtime`) |
| 10 | Stripe Connect Express (therapist payouts) |
| 11 | Client-side billing + payment methods (Stripe Elements) |
| 12 | `@repo/notifications` (Knock) — in-app + email reminders (Resend) |
| 13 | B2B organizations: creation, member invite, coverage logic |
| 14 | B2B billing: Stripe Billing, monthly invoicing Edge Function, org admin dashboard |
| 15 | Admin analytics dashboard (all sub-pages, materialized views) |
| 16 | Disputes + refund flow |
| 17 | Reviews + moderation queue |
| 18 | Platform settings module (all tabs) |
| 19 | Admin management (super_admin), audit log, impersonation |
| 20 | `apps/web` public site: home, therapist directory, static pages |
| 21 | Blog CMS: `@repo/cms` Content Collections + ISR + admin CRUD |
| 22 | Psychological test engine: admin creation + anonymous execution + result |
| 23 | Newsletter + lead capture + `@repo/analytics` consent management |
| 24 | B2B SSO (Clerk Organizations SAML/OIDC) |
| 25 | `@repo/observability` full setup (Sentry, BetterStack), bundle analysis |
| 26+ | Mobile PWA, push notifications, recording consent flow, i18n (`@repo/i18n`) |

### High-Risk Areas

| Risk | Area | Mitigation |
|---|---|---|
| PHI encryption key rotation | Notes, messages | App-layer rotation job + Prisma migration; no pgcrypto dependency |
| Stripe Connect complexity | Therapist payouts | Start manual (admin-initiated), automate Sprint 10 |
| B2B billing reconciliation | Org invoices | Stripe Billing + idempotency key `orgId:periodStart` |
| SAML/OIDC misconfiguration | B2B SSO | Use Clerk Organizations (managed IdP integration) |
| Slot race conditions | Booking creation | Prisma `$transaction` + `SELECT FOR UPDATE` equivalent via optimistic locking |
| HIPAA BAA gap | All PHI vendors | BAAs: Supabase Pro+, Daily.co, Resend, PostHog (EU) before US production |
| Admin impersonation abuse | Super admin | 2nd admin approval required; every session logged in `AdminImpersonation` |
| Credential forgery | Therapist onboarding | Nursys / FSMB license verification API integration (Sprint 4+) |
| GDPR data deletion | User data | Prisma soft-delete + anonymization pipeline; no hard DELETE |
| Anonymous org reports | B2B privacy | k-anonymity: suppress if `activeMembers < 5` |
| Test score manipulation | Test engine | Score computed server-side only in Route Handler; never on client |
| Newsletter email enumeration | Public API | Always return 200 with identical message body |
| Duplicate content (test results) | SEO | `noindex` all `/tests/*/result` pages; `noindex` in `robots.ts` |
| ISR stale health content | Blog | Use `revalidatePath` on publish (on-demand), not time-based only |
| GDPR double opt-in | Newsletter EU | Resend confirmation email mandatory; single opt-in blocked for EU IPs |
| Anonymous test health data | GDPR | 24h auto-purge Edge Function + `expiresAt` column enforced |
| Turborepo cache poisoning | CI/CD | Sign Turborepo Remote Cache tokens; use Vercel's managed cache |
| Clerk webhook replay attacks | Auth sync | Verify Svix signature + timestamp window (5 min) in `apps/api` |

---

_Spec version: 4.0 — May 2026_
_Aligned to Althena v6.x monorepo pattern, Turborepo, Prisma, Clerk, Arcjet, Resend, Knock, Stripe._
_Validate all `[INFERRED]` sections against actual Figma screens before Sprint 1._
_Domain: althena.com (currently offline — spec context only)_# Althena — Full Platform Technical Specification

**Platform:** Mental Health SaaS (B2C + B2B + Public Site)
**Domain:** althena.com
**Roles Covered:** Client · B2B Member · Therapist · Admin · Super Admin · B2B Organization
**Stack:** Next.js 15 (App Router) · Turborepo · Supabase (PostgreSQL) · Prisma · TypeScript · Tailwind CSS
**Architecture:** Althena monorepo pattern (v6.x)
**Version:** 4.0 | **Date:** 2026-05-18

> ⚠️ **Figma Access Note:** Built from Figma node contexts `3990:77497` (Therapist Dashboard), `4016:76749` (Admin Dashboard), `3990-77440` (Guest Site). Sections marked `[INFERRED]` must be validated against actual screens before implementation.

---

## TABLE OF CONTENTS

- [MONOREPO ARCHITECTURE](#monorepo)
- [PHASE 0 — PRODUCT & SYSTEM FOUNDATION](#phase-0)
  1. [User Roles & Sub-states](#1-user-roles--sub-states)
  2. [Main User Flows & State Machines](#2-main-user-flows--state-machines)
  3. [Business Model (B2C + B2B)](#3-business-model)
  4. [Domain Entities & DB Schema (Prisma)](#4-domain-entities--db-schema)
  5. [Domain Boundaries](#5-domain-boundaries)
  6. [Sensitive Data Inventory](#6-sensitive-data-inventory)
  7. [RBAC Matrix](#7-rbac-matrix)
- [PHASE 1 — CORE PLATFORM](#phase-1)
  8. [Pages & Routes (all roles + public)](#8-pages--routes)
  9. [Sidebar Navigation (per role)](#9-sidebar-navigation)
  10. [Forms — fields, validation, types](#10-forms)
  11. [Tables & Lists](#11-tables--lists)
  12. [API Endpoints](#12-api-endpoints)
  13. [Real-time Functionality](#13-real-time-functionality)
  14. [File Functionality](#14-file-functionality)
  15. [Role-Based Navigation Components](#15-role-based-navigation-components)
- [PHASE 2 — PUBLIC SITE (Guest Domain)](#phase-2)
  16. [Public Domain Entities](#16-public-domain-entities)
  17. [Route Architecture (App Router)](#17-route-architecture)
  18. [Public Forms](#18-public-forms)
  19. [Public API Endpoints](#19-public-api-endpoints)
  20. [SEO Architecture](#20-seo-architecture)
  21. [State Machines (Guest Domain)](#21-state-machines-guest-domain)
  22. [Frontend Architecture & Rendering Strategy](#22-frontend-architecture--rendering-strategy)
  23. [Analytics & Tracking Architecture](#23-analytics--tracking-architecture)
- [ADMIN PANEL DEEP DIVE](#admin-panel)
  24. [Admin Roles & Permission Architecture](#24-admin-roles--permission-architecture)
  25. [Admin Settings Module](#25-admin-settings-module)
  26. [Admin User Management](#26-admin-user-management)
  27. [B2B Organizations Module](#27-b2b-organizations-module)
- [CROSS-CUTTING CONCERNS](#cross-cutting)

---

## MONOREPO ARCHITECTURE {#monorepo}

Althena follows the **Althena v6.x monorepo pattern** managed by Turborepo. Each app is independently deployable; packages are shared libraries imported as `@repo/<name>`.

### Repository Structure

```
althena/
├── apps/
│   ├── web/            # Public marketing site — althena.com (port 3001)
│   ├── app/            # Authenticated client/therapist app — app.althena.com (port 3000)
│   ├── admin/          # Admin panel — admin.althena.com (port 3002)
│   ├── api/            # Webhook handlers, health checks — api.althena.com (port 3003)
│   └── email/          # React Email template previews (port 3004)
│
├── packages/
│   ├── auth/           # @repo/auth       — Clerk (or Better Auth) config, middleware
│   ├── database/       # @repo/database   — Prisma client, schema, migrations
│   ├── design-system/  # @repo/design-system — shadcn/ui, Tailwind tokens, providers
│   ├── email/          # @repo/email      — React Email templates via Resend
│   ├── cms/            # @repo/cms        — Blog/test content, Content Collections
│   ├── payments/       # @repo/payments   — Stripe client, Connect, Billing
│   ├── analytics/      # @repo/analytics  — PostHog + GA4 with consent gating
│   ├── observability/  # @repo/observability — Sentry, BetterStack logging
│   ├── security/       # @repo/security   — Arcjet rate limiting, bot protection
│   ├── seo/            # @repo/seo        — generateMetadata helpers, JSON-LD
│   ├── storage/        # @repo/storage    — Supabase Storage client, presigned URLs
│   ├── realtime/       # @repo/realtime   — Supabase Realtime channel helpers
│   ├── notifications/  # @repo/notifications — In-app notifications (Knock)
│   ├── feature-flags/  # @repo/feature-flags — Feature flag management
│   ├── webhooks/       # @repo/webhooks   — Stripe + Clerk webhook verification (Svix)
│   ├── internationalization/ # @repo/i18n — Multi-language (i18next)
│   └── next-config/    # @repo/next-config — Shared next.config.ts base
│
├── turbo.json
├── biome.jsonc         # Linting + formatting (replaces ESLint + Prettier)
├── package.json
└── tsconfig.json
```

### App Deployment URLs

| App | Subdomain | Notes |
|---|---|---|
| `apps/web` | `althena.com` | Public marketing site, blog, tests, therapist directory |
| `apps/app` | `app.althena.com` | Authenticated platform (client + therapist dashboards) |
| `apps/admin` | `admin.althena.com` | Admin panel — separate CSP, 2FA enforced |
| `apps/api` | `api.althena.com` | Webhook receivers (Stripe, Clerk), health endpoint |
| `apps/email` | dev-only | React Email template preview server |

### Key Pattern Differences vs Plain Next.js

| Pattern | Althena convention | Althena implementation |
|---|---|---|
| Auth | `@repo/auth` wraps Clerk | Clerk for client/therapist; custom admin TOTP layer |
| Database | `@repo/database` exports `prisma` client | Supabase PostgreSQL + Prisma ORM |
| ORM | Prisma schema in `packages/database/prisma/schema.prisma` | Single schema covering all tables |
| Env vars | `packages/<pkg>/keys.ts` with `@t3-oss/env-nextjs` + Zod | Each package owns its env shape |
| Rate limiting | `@repo/security` wraps Arcjet | Arcjet for public APIs; Clerk for auth endpoints |
| Email | `@repo/email` + Resend | React Email templates, Handlebars for admin-editable templates |
| CMS | `@repo/cms` wraps Content Collections or BaseHub | Content Collections for blog/tests (MDX-based, type-safe) |
| Webhooks | `@repo/webhooks` — verification via Svix | `apps/api/app/webhooks/stripe/route.ts`, `apps/api/app/webhooks/clerk/route.ts` |
| Payments | `@repo/payments` wraps Stripe | Stripe Connect Express (therapist payouts) + Stripe Billing (B2B invoicing) |
| Observability | `@repo/observability` — Sentry + BetterStack | Sentry for error tracking, BetterStack for log drain |
| Analytics | `@repo/analytics` — PostHog + GA4 | Consent-gated; EU data residency on PostHog |
| Storage | `@repo/storage` — Supabase Storage | Presigned URLs, Sharp processing via Edge Function |
| Notifications | `@repo/notifications` — Knock | In-app + email notification delivery |

### Environment Variable Pattern

Each package exposes a `keys.ts` with Zod validation. Apps compose them in `env.ts`:

```typescript
// packages/auth/keys.ts
import { createEnv } from '@t3-oss/env-nextjs';
import { z } from 'zod';

export const keys = createEnv({
  server: {
    CLERK_SECRET_KEY: z.string().min(1).startsWith('sk_'),
  },
  client: {
    NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY: z.string().min(1).startsWith('pk_'),
    NEXT_PUBLIC_CLERK_SIGN_IN_URL: z.string().default('/sign-in'),
    NEXT_PUBLIC_CLERK_SIGN_UP_URL: z.string().default('/sign-up'),
  },
  runtimeEnv: {
    CLERK_SECRET_KEY: process.env.CLERK_SECRET_KEY,
    NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY: process.env.NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY,
    NEXT_PUBLIC_CLERK_SIGN_IN_URL: process.env.NEXT_PUBLIC_CLERK_SIGN_IN_URL,
    NEXT_PUBLIC_CLERK_SIGN_UP_URL: process.env.NEXT_PUBLIC_CLERK_SIGN_UP_URL,
  },
});

// packages/database/keys.ts
export const keys = createEnv({
  server: {
    DATABASE_URL: z.string().url().startsWith('postgresql://'),
    DIRECT_URL: z.string().url().optional(), // for Supabase connection pooling
  },
  runtimeEnv: {
    DATABASE_URL: process.env.DATABASE_URL,
    DIRECT_URL: process.env.DIRECT_URL,
  },
});

// apps/app/env.ts — composes all package envs
import { keys as authKeys }     from '@repo/auth/keys';
import { keys as dbKeys }       from '@repo/database/keys';
import { keys as paymentsKeys } from '@repo/payments/keys';
import { keys as analyticsKeys } from '@repo/analytics/keys';
// ... etc
```

### Turborepo Pipeline

```json
// turbo.json
{
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": [".next/**", "!.next/cache/**", "dist/**"]
    },
    "dev": { "cache": false, "persistent": true },
    "lint": {},
    "type-check": { "dependsOn": ["^build"] },
    "db:generate": { "cache": false },
    "db:migrate": { "cache": false },
    "db:push": { "cache": false }
  }
}
```

---

## PHASE 0 — PRODUCT & SYSTEM FOUNDATION {#phase-0}

---

### 1. User Roles & Sub-states

#### 1.1 Primary Roles

| Role | Internal Enum | Description | Auth Method |
|---|---|---|---|
| Client | `client` | End-user seeking therapy; books sessions | Clerk (Email / Google OAuth) |
| Therapist | `therapist` | Licensed professional; manages caseload | Clerk (Email + credential verification) |
| Admin | `admin` | Platform operator; manages users, content, billing | Clerk (Internal email, TOTP 2FA enforced) |
| Super Admin | `super_admin` | Full platform access; can manage admins | Clerk (Internal SSO + hardware key) |
| B2B Member | `b2b_member` | Employee of an org with corporate plan | Clerk (SSO via org IdP or email) |

Role stored in **Clerk `publicMetadata.role`** and mirrored to `profiles.role` in Prisma on Clerk webhook sync.

#### 1.2 Admin Permission Tiers

| Tier | Enum | Can Do |
|---|---|---|
| Support Agent | `admin:support` | View users, manage disputes, read-only on payments |
| Content Moderator | `admin:moderator` | Review therapist credentials, manage reviews, CMS |
| Finance Admin | `admin:finance` | Full payment/payout access, billing config |
| Operations Admin | `admin:operations` | All of the above + platform settings |
| Super Admin | `super_admin` | Everything + admin management + B2B enterprise config |

Tier stored in `Clerk publicMetadata.adminTier` and `profiles.adminTier`.

#### 1.3 Therapist Sub-states

| State | Enum | Description |
|---|---|---|
| Pending Verification | `pending` | Submitted credentials, awaiting admin review |
| Active | `active` | Fully onboarded, accepting clients |
| On Leave | `on_leave` | Temporarily unavailable; no new bookings |
| Suspended | `suspended` | Admin action; login blocked, sessions cancelled |
| Rejected | `rejected` | Credentials not accepted; can resubmit |

#### 1.4 B2B Organization Member States

| State | Description |
|---|---|
| `invited` | Invitation email sent, not yet registered |
| `active` | Registered and can book sessions |
| `deactivated` | HR/admin removed access; data retained |

---

### 2. Main User Flows & State Machines

#### 2.1 Therapist Onboarding
```
REGISTER (Clerk) → CLERK_WEBHOOK_SYNC → EMAIL_VERIFY
  → PROFILE_SETUP → CREDENTIAL_UPLOAD
  → ADMIN_REVIEW
    → [APPROVED] → AVAILABILITY_SETUP → STRIPE_CONNECT → ACTIVE
    → [REJECTED]  → NOTIFICATION_SENT → RESUBMIT
    → [MORE_INFO_NEEDED] → THERAPIST_NOTIFIED → REPLY → RE_REVIEW
```

#### 2.2 Client Registration & First Booking
```
REGISTER (Clerk) → CLERK_WEBHOOK_SYNC → EMAIL_VERIFY
  → INTAKE_FORM → THERAPIST_MATCH
  → VIEW_PROFILE → SELECT_SLOT → PAYMENT_CAPTURE (Stripe)
  → BOOKING_CONFIRMED → REMINDER (Knock) → SESSION → POST_SESSION_REVIEW
```

#### 2.3 B2B Employee Onboarding
```
ORG_ADMIN_INVITES_EMPLOYEE → CLERK_MAGIC_LINK_EMAIL
  → EMPLOYEE_REGISTERS → SELECTS_THERAPIST (org-approved pool)
  → BOOKS_SESSION → BILLED_TO_ORG_ACCOUNT (Stripe Billing)
  → USAGE_REPORTED_TO_ORG_DASHBOARD (anonymized)
```

#### 2.4 Session Lifecycle State Machine
```
SCHEDULED ──► CONFIRMED ──► IN_PROGRESS ──► COMPLETED
     │              │                            │
     ▼              ▼                            ▼
 CANCELLED      CANCELLED                   NO_SHOW
     │
     ▼
REFUND_INITIATED ──► REFUNDED
```

#### 2.5 Admin Therapist Review Flow
```
THERAPIST submits credentials
  → Knock notification to admin queue
  → ADMIN opens review screen (admin.althena.com)
    → Views credential document (Supabase Storage signed URL, 15 min)
    → Checks license against external registry (Nursys/FSMB, optional)
    → [APPROVE] → therapist.status = active, Resend email sent
    → [REJECT + reason] → therapist notified, can resubmit
    → [REQUEST_INFO] → Knock message sent to therapist
```

#### 2.6 Dispute / Refund Flow
```
CLIENT raises dispute on booking
  → Knock notification to admin queue
  → ADMIN reviews session record + messages
    → [APPROVE_REFUND] → Stripe refund via @repo/payments, payout adjusted
    → [REJECT_DISPUTE] → Client notified with reason (Resend)
    → [PARTIAL_REFUND] → Custom amount entered
```

#### 2.7 Guest Conversion Funnels

**Funnel A — Guest → Client Registration**
```
althena.com (Home)
  → [CTA: "Find a Therapist" or "Take the Test"]
    ├── /therapists/[slug] → /sign-up?intent=book&therapistId=[id]
    │     → app.althena.com/bookings (post-registration)
    ├── /tests/[slug]/start → /tests/[slug]/result
    │     → /sign-up?intent=test_result&testId=[id]&band=[band]
    └── /about → /sign-up
```

**Funnel B — B2B Corporate Lead**
```
althena.com/for-business → lead form → api.althena.com/leads (POST)
  → Resend notification to sales team, lead_submissions record created
```

**Funnel C — Therapist Partner Acquisition**
```
althena.com/cooperation → /sign-up?role=therapist (or inline lead form)
```

**Funnel D — Test Funnel (SEO + conversion)**
```
althena.com/tests/[slug] → /tests/[slug]/start → /tests/[slug]/result
  → /sign-up?intent=save_test&sessionToken=[token]
```

#### 2.8 Anonymous / Authenticated Transitions

| Transition Point | Auth Required | Post-auth Redirect |
|---|---|---|
| Book a session | Required (Clerk) | `app.althena.com/bookings` |
| Save a therapist | Required | Return to same page |
| Start a test | Not required | — |
| View test result | Not required | — |
| Save test result | Required | Return to result |
| Newsletter signup | Not required | — |
| Business inquiry | Not required | — |
| Cooperation apply | Not required (lead) / Required (full) | — |

#### 2.9 Test Execution State Machine (Guest Domain)
```
[INTRO]
  → POST /api/public/tests/[testId]/session
    → session_token in httpOnly cookie (24h)
      → [QUESTION_N] × questions
        → PATCH .../answer (debounced 300ms, sendBeacon on unload)
          → POST .../complete (score computed server-side)
            → [RESULT]

[QUESTION_N] → browser closed
  → session persists (in_progress)
  → on return: check httpOnly cookie
    → active: [RESUME_PROMPT]
    → expired (>24h): [INTRO]

[RESULT]
  → authenticated: POST .../claim
  → guest: /sign-up?intent=save_test&sessionToken=[token]
  → "Share": /tests/[slug]/result?band=[result_band] (no PII)
```

#### 2.10 Newsletter Subscription State Machine
```
[IDLE] → POST /api/public/newsletter
  → honeypot check → silent discard if triggered
  → Arcjet rate limit check (3/IP/hour)
  → INSERT newsletter_subscribers (always return 200, no enumeration)
    → Resend: double opt-in confirmation email (mandatory for EU)
      → [PENDING] → user clicks confirm → [CONFIRMED]
      → token expired (24h) → [EXPIRED] → resend option

[CONFIRMED] → unsubscribe link → [UNSUBSCRIBED]
```

#### 2.11 Content Publishing (CMS)
```
DRAFT → [Schedule] → SCHEDULED → [cron Edge Function] → PUBLISHED
DRAFT → [Publish Now] → PUBLISHED
PUBLISHED → [Unpublish] → DRAFT
PUBLISHED → [Archive] → ARCHIVED (410 response)

On PUBLISHED: next.js revalidateTag + revalidatePath
```

---

### 3. Business Model

#### 3.1 B2C Model

| Component | Details |
|---|---|
| Session Fee | Therapist-set rate per session ($60–$300) |
| Platform Commission | Configurable % per completed session (default 20%) |
| Cancellation Fee | Client pays 50% if cancelled <24h (configurable) |
| Payment Processor | Stripe (card, Apple Pay, Google Pay) |
| Payout | Weekly via Stripe Connect Express |
| Subscription (optional) | Discounted session bundles |

#### 3.2 B2B Model

| Component | Details |
|---|---|
| Plan Types | Per-seat monthly · Session-credit pack · Unlimited (enterprise) |
| Billing Entity | Organization pays platform via Stripe Billing (NET-30) |
| Session Coverage | N sessions/month per employee seat |
| Overage | Employee pays out-of-pocket above plan limit |
| Invoicing | Monthly Stripe Invoice → `organization_invoices` record |
| Employee Privacy | Never share individual session content; aggregate stats only |
| Therapist Pool | Org can restrict employees to curated therapist list |
| SSO | SAML/OIDC via Clerk Organizations |
| Reporting | Org admin dashboard: anonymized utilization, avg satisfaction |

---

### 4. Domain Entities & DB Schema

**ORM:** Prisma · **Database:** Supabase (PostgreSQL) · **Schema location:** `packages/database/prisma/schema.prisma`

Supabase connection pooling via pgBouncer — set `DATABASE_URL` to the pooler URL and `DIRECT_URL` to the direct connection (required for Prisma migrations).

```prisma
// packages/database/prisma/schema.prisma

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider  = "postgresql"
  url       = env("DATABASE_URL")
  directUrl = env("DIRECT_URL")
}

// ─────────────────────────────────────────────
// ENUMS
// ─────────────────────────────────────────────

enum UserRole {
  client
  therapist
  admin
  super_admin
  b2b_member
}

enum AdminTier {
  support
  moderator
  finance
  operations
}

enum TherapistStatus {
  pending
  active
  on_leave
  suspended
  rejected
}

enum BookingStatus {
  scheduled
  confirmed
  in_progress
  completed
  cancelled
  no_show
}

enum SessionType {
  video
  audio
  chat
}

enum NoteType {
  soap
  progress
  intake
  discharge
}

enum PaymentStatus {
  pending
  succeeded
  failed
  refunded
  partially_refunded
}

enum PayoutStatus {
  pending
  paid
  failed
}

enum DisputeStatus {
  open
  under_review
  resolved_refund
  resolved_no_action
}

enum OrgPlan {
  per_seat
  credit_pack
  unlimited
  enterprise
}

enum OrgMemberRole {
  member
  org_admin
}

enum OrgMemberStatus {
  invited
  active
  deactivated
}

enum InvoiceStatus {
  draft
  open
  paid
  void
  uncollectible
}

enum BlogStatus {
  draft
  scheduled
  published
  archived
}

enum TestStatus {
  draft
  published
  archived
}

enum TestSessionStatus {
  in_progress
  completed
  abandoned
}

enum LeadType {
  business_inquiry
  cooperation_request
  general_contact
}

enum NewsletterStatus {
  pending
  confirmed
  unsubscribed
  bounced
}

// ─────────────────────────────────────────────
// CORE PLATFORM
// ─────────────────────────────────────────────

model Profile {
  id                String    @id // Clerk user ID
  role              UserRole
  adminTier         AdminTier?
  fullName          String
  avatarUrl         String?
  phone             String?
  timezone          String    @default("UTC")
  dateOfBirth       DateTime? // SENSITIVE — encrypted at app layer
  preferredLanguage String    @default("en")
  isDeleted         Boolean   @default(false)
  createdAt         DateTime  @default(now())
  updatedAt         DateTime  @updatedAt
  lastSeenAt        DateTime?

  therapistProfile      TherapistProfile?
  clientProfile         ClientProfile?
  sentMessages          Message[]         @relation("SentMessages")
  notifications         Notification[]
  adminLogs             AdminAuditLog[]   @relation("AdminLogs")
  raisedDisputes        Dispute[]         @relation("RaisedDisputes")
  resolvedDisputes      Dispute[]         @relation("ResolvedDisputes")
  cancelledBookings     Booking[]         @relation("CancelledBookings")
  impersonations        AdminImpersonation[] @relation("ImpersonationTarget")
  impersonatedBy        AdminImpersonation[] @relation("ImpersonationActor")
  uploadedMedia         MediaAsset[]
  authoredBlogPosts     BlogAuthor?

  @@map("profiles")
}

model TherapistProfile {
  id                      String          @id
  profile                 Profile         @relation(fields: [id], references: [id])
  licenseNumber           String          // SENSITIVE — encrypted
  licenseState            String?
  licenseExpiry           DateTime?
  licenseVerifiedAt       DateTime?
  specializations         String[]
  approaches              String[]
  languages               String[]
  bio                     String?
  yearsOfExperience       Int?
  sessionRateCents        Int
  acceptsInsurance        Boolean         @default(false)
  insuranceProviders      String[]
  stripeAccountId         String?         // SENSITIVE
  onboardingStatus        TherapistStatus @default(pending)
  verifiedById            String?
  rejectionReason         String?
  totalSessionsCompleted  Int             @default(0)
  avgRating               Decimal?        @db.Decimal(3, 2)

  bookings          Booking[]
  sessionNotes      SessionNote[]
  conversations     Conversation[]
  payouts           Payout[]
  reviews           Review[]
  orgPools          OrganizationTherapistPool[]
  availabilityRules AvailabilitySchedule[]
  availabilityExceptions AvailabilityException[]

  @@map("therapist_profiles")
}

model ClientProfile {
  id                     String   @id
  profile                Profile  @relation(fields: [id], references: [id])
  organizationId         String?
  organization           Organization? @relation(fields: [organizationId], references: [id])
  emergencyContactName   String?  // SENSITIVE — encrypted
  emergencyContactPhone  String?  // SENSITIVE — encrypted
  insuranceMemberId      String?  // SENSITIVE — encrypted
  primaryConcern         String[]
  assignedTherapistId    String?
  sessionsRemainingThisMonth Int?

  bookings      Booking[]
  conversations Conversation[]
  reviews       Review[]
  orgMembership OrganizationMember?

  @@map("client_profiles")
}

model AvailabilitySchedule {
  id                  String           @id @default(cuid())
  therapistId         String
  therapist           TherapistProfile @relation(fields: [therapistId], references: [id])
  dayOfWeek           Int              // 0–6
  startTime           String           // HH:mm
  endTime             String           // HH:mm
  slotDurationMinutes Int              @default(50)
  isActive            Boolean          @default(true)

  @@map("availability_schedules")
}

model AvailabilityException {
  id          String           @id @default(cuid())
  therapistId String
  therapist   TherapistProfile @relation(fields: [therapistId], references: [id])
  date        DateTime         @db.Date
  reason      String?
  createdAt   DateTime         @default(now())

  @@map("availability_exceptions")
}

model Booking {
  id                 String        @id @default(cuid())
  clientId           String
  client             ClientProfile @relation(fields: [clientId], references: [id])
  therapistId        String
  therapist          TherapistProfile @relation(fields: [therapistId], references: [id])
  organizationId     String?
  organization       Organization? @relation(fields: [organizationId], references: [id])
  scheduledAt        DateTime
  durationMinutes    Int           @default(50)
  status             BookingStatus @default(scheduled)
  sessionType        SessionType   @default(video)
  cancellationReason String?
  cancelledById      String?
  cancelledBy        Profile?      @relation("CancelledBookings", fields: [cancelledById], references: [id])
  cancelledAt        DateTime?
  rateCents          Int
  platformFeeCents   Int
  isCoveredByOrg     Boolean       @default(false)
  createdAt          DateTime      @default(now())
  updatedAt          DateTime      @updatedAt

  session    Session?
  notes      SessionNote[]
  payment    Payment?
  dispute    Dispute?
  review     Review?

  @@map("bookings")
}

model Session {
  id                   String   @id @default(cuid())
  bookingId            String   @unique
  booking              Booking  @relation(fields: [bookingId], references: [id])
  roomUrl              String?
  startedAt            DateTime?
  endedAt              DateTime?
  durationActualSeconds Int?
  recordingUrl         String?  // SENSITIVE
  transcriptUrl        String?  // SENSITIVE

  @@map("sessions")
}

model SessionNote {
  id                  String           @id @default(cuid())
  bookingId           String
  booking             Booking          @relation(fields: [bookingId], references: [id])
  therapistId         String
  therapist           TherapistProfile @relation(fields: [therapistId], references: [id])
  content             String           // SENSITIVE — AES-256 encrypted before INSERT
  noteType            NoteType
  isSharedWithClient  Boolean          @default(false)
  // SOAP subfields stored in content JSON when noteType = soap
  createdAt           DateTime         @default(now())
  updatedAt           DateTime         @updatedAt

  @@map("session_notes")
}

model Conversation {
  id          String           @id @default(cuid())
  clientId    String
  client      ClientProfile    @relation(fields: [clientId], references: [id])
  therapistId String
  therapist   TherapistProfile @relation(fields: [therapistId], references: [id])
  lastMessageAt DateTime?
  isArchived  Boolean          @default(false)

  messages Message[]

  @@map("conversations")
}

model Message {
  id             String       @id @default(cuid())
  conversationId String
  conversation   Conversation @relation(fields: [conversationId], references: [id])
  senderId       String
  sender         Profile      @relation("SentMessages", fields: [senderId], references: [id])
  content        String       // SENSITIVE — AES-256 encrypted
  attachmentUrl  String?
  isRead         Boolean      @default(false)
  sentAt         DateTime     @default(now())
  deletedAt      DateTime?

  @@map("messages")
}

model Payment {
  id                     String        @id @default(cuid())
  bookingId              String        @unique
  booking                Booking       @relation(fields: [bookingId], references: [id])
  stripePaymentIntentId  String        @unique // SENSITIVE — never logged
  amountCents            Int
  currency               String        @default("usd")
  status                 PaymentStatus @default(pending)
  refundAmountCents      Int?
  refundReason           String?
  createdAt              DateTime      @default(now())

  @@map("payments")
}

model Payout {
  id               String           @id @default(cuid())
  therapistId      String
  therapist        TherapistProfile @relation(fields: [therapistId], references: [id])
  stripeTransferId String?          // SENSITIVE
  amountCents      Int
  periodStart      DateTime         @db.Date
  periodEnd        DateTime         @db.Date
  status           PayoutStatus     @default(pending)
  paidAt           DateTime?

  @@map("payouts")
}

model AdminAuditLog {
  id         String   @id @default(cuid())
  adminId    String
  admin      Profile  @relation("AdminLogs", fields: [adminId], references: [id])
  action     String   // e.g. 'therapist.approve', 'user.suspend'
  targetType String?  // 'therapist' | 'client' | 'booking'
  targetId   String?
  metadata   Json?
  ipAddress  String?
  createdAt  DateTime @default(now())

  @@map("admin_audit_logs")
}

model AdminImpersonation {
  id         String   @id @default(cuid())
  actorId    String
  actor      Profile  @relation("ImpersonationActor", fields: [actorId], references: [id])
  targetId   String
  target     Profile  @relation("ImpersonationTarget", fields: [targetId], references: [id])
  approvedById String?
  startedAt  DateTime @default(now())
  endedAt    DateTime?

  @@map("admin_impersonations")
}

model Dispute {
  id             String        @id @default(cuid())
  bookingId      String        @unique
  booking        Booking       @relation(fields: [bookingId], references: [id])
  raisedById     String
  raisedBy       Profile       @relation("RaisedDisputes", fields: [raisedById], references: [id])
  reason         String
  status         DisputeStatus @default(open)
  resolutionNote String?
  resolvedById   String?
  resolvedBy     Profile?      @relation("ResolvedDisputes", fields: [resolvedById], references: [id])
  resolvedAt     DateTime?
  createdAt      DateTime      @default(now())

  @@map("disputes")
}

model Notification {
  id        String   @id @default(cuid())
  userId    String
  user      Profile  @relation(fields: [userId], references: [id])
  type      String   // See notification type enum in docs
  title     String?
  body      String?
  data      Json?
  isRead    Boolean  @default(false)
  createdAt DateTime @default(now())

  @@map("notifications")
}

model Review {
  id          String           @id @default(cuid())
  bookingId   String           @unique
  booking     Booking          @relation(fields: [bookingId], references: [id])
  clientId    String
  client      ClientProfile    @relation(fields: [clientId], references: [id])
  therapistId String
  therapist   TherapistProfile @relation(fields: [therapistId], references: [id])
  rating      Int              // 1–5
  comment     String?
  isPublic    Boolean          @default(true)
  isFlagged   Boolean          @default(false)
  flaggedReason String?
  createdAt   DateTime         @default(now())

  @@map("reviews")
}

model PlatformSetting {
  key       String   @id
  value     Json
  updatedBy String?
  updatedAt DateTime @updatedAt

  @@map("platform_settings")
}

// ─────────────────────────────────────────────
// B2B
// ─────────────────────────────────────────────

model Organization {
  id                       String    @id @default(cuid())
  clerkOrgId               String?   @unique // Clerk Organizations ID
  name                     String
  domain                   String?
  logoUrl                  String?
  planType                 OrgPlan
  seatsPurchased           Int?
  sessionsPerSeatPerMonth  Int?
  stripeCustomerId         String?   // SENSITIVE
  billingEmail             String
  billingContactName       String?
  ssoProvider              String?   // 'saml' | 'oidc' | null
  ssoConfig                Json?     // SENSITIVE — encrypted at app layer
  isActive                 Boolean   @default(true)
  contractStart            DateTime? @db.Date
  contractEnd              DateTime? @db.Date
  createdAt                DateTime  @default(now())

  members        OrganizationMember[]
  therapistPool  OrganizationTherapistPool[]
  invoices       OrganizationInvoice[]
  usageReports   OrganizationUsageReport[]
  bookings       Booking[]
  clientProfiles ClientProfile[]

  @@map("organizations")
}

model OrganizationMember {
  id                    String          @id @default(cuid())
  organizationId        String
  organization          Organization    @relation(fields: [organizationId], references: [id])
  clientId              String?         @unique
  client                ClientProfile?  @relation(fields: [clientId], references: [id])
  email                 String
  role                  OrgMemberRole   @default(member)
  status                OrgMemberStatus @default(invited)
  sessionsUsedThisMonth Int             @default(0)
  invitedAt             DateTime?
  joinedAt              DateTime?

  @@unique([organizationId, email])
  @@map("organization_members")
}

model OrganizationTherapistPool {
  organizationId String
  organization   Organization    @relation(fields: [organizationId], references: [id])
  therapistId    String
  therapist      TherapistProfile @relation(fields: [therapistId], references: [id])
  addedAt        DateTime         @default(now())

  @@id([organizationId, therapistId])
  @@map("organization_therapist_pools")
}

model OrganizationInvoice {
  id              String        @id @default(cuid())
  organizationId  String
  organization    Organization  @relation(fields: [organizationId], references: [id])
  stripeInvoiceId String        @unique // SENSITIVE
  periodStart     DateTime      @db.Date
  periodEnd       DateTime      @db.Date
  amountCents     Int
  seatsBilled     Int?
  sessionsBilled  Int?
  status          InvoiceStatus @default(draft)
  dueDate         DateTime?     @db.Date
  paidAt          DateTime?
  createdAt       DateTime      @default(now())

  @@map("organization_invoices")
}

model OrganizationUsageReport {
  id                  String       @id @default(cuid())
  organizationId      String
  organization        Organization @relation(fields: [organizationId], references: [id])
  reportMonth         DateTime     @db.Date
  totalMembers        Int
  activeMembers       Int          // had ≥1 session (k-anonymity: suppress if <5)
  totalSessions       Int
  avgSatisfactionScore Decimal?    @db.Decimal(3, 2)
  generatedAt         DateTime     @default(now())

  @@map("organization_usage_reports")
}

// ─────────────────────────────────────────────
// PUBLIC / GUEST DOMAIN
// ─────────────────────────────────────────────

model BlogAuthor {
  id                  String    @id @default(cuid())
  profileId           String?   @unique
  profile             Profile?  @relation(fields: [profileId], references: [id])
  fullName            String
  slug                String    @unique
  bio                 String?
  avatarUrl           String?
  title               String?
  credentials         String?
  twitterUrl          String?
  linkedinUrl         String?
  createdAt           DateTime  @default(now())

  posts BlogPost[]

  @@map("blog_authors")
}

model BlogCategory {
  id             String        @id @default(cuid())
  slug           String        @unique
  name           String
  description    String?
  parentId       String?
  parent         BlogCategory? @relation("CategoryParent", fields: [parentId], references: [id])
  children       BlogCategory[] @relation("CategoryParent")
  seoTitle       String?
  seoDescription String?
  coverImageUrl  String?
  sortOrder      Int           @default(0)
  createdAt      DateTime      @default(now())

  posts BlogPost[]

  @@map("blog_categories")
}

model BlogPost {
  id               String       @id @default(cuid())
  slug             String       @unique
  title            String
  excerpt          String?      @db.VarChar(300)
  content          String       // MDX or HTML
  coverImageUrl    String?
  authorId         String?
  author           BlogAuthor?  @relation(fields: [authorId], references: [id])
  categoryId       String?
  category         BlogCategory? @relation(fields: [categoryId], references: [id])
  tags             String[]
  status           BlogStatus   @default(draft)
  publishedAt      DateTime?
  scheduledFor     DateTime?
  readingTimeMinutes Int?
  seoTitle         String?
  seoDescription   String?
  ogImageUrl       String?
  canonicalUrl     String?
  schemaType       String       @default("Article")
  viewCount        Int          @default(0)
  createdAt        DateTime     @default(now())
  updatedAt        DateTime     @updatedAt

  @@map("blog_posts")
}

model Test {
  id              String     @id @default(cuid())
  slug            String     @unique
  title           String
  description     String?
  instructions    String?
  coverImageUrl   String?
  category        String?
  estimatedMinutes Int?
  questionCount   Int?       // denormalized
  status          TestStatus @default(draft)
  isFeatured      Boolean    @default(false)
  scoringMethod   String     // 'sum'|'weighted_sum'|'subscale'
  resultConfig    Json       // { bands: [...], subscales: null }
  seoTitle        String?
  seoDescription  String?
  ogImageUrl      String?
  schemaType      String     @default("MedicalWebPage")
  completionCount Int        @default(0)
  publishedAt     DateTime?
  createdAt       DateTime   @default(now())
  updatedAt       DateTime   @updatedAt

  questions TestQuestion[]
  sessions  TestSession[]

  @@map("tests")
}

model TestQuestion {
  id           String   @id @default(cuid())
  testId       String
  test         Test     @relation(fields: [testId], references: [id], onDelete: Cascade)
  sortOrder    Int
  text         String
  questionType String   // 'single_choice'|'scale'|'boolean'
  options      Json     // [{ value: 0, label: "Not at all" }]
  subscale     String?
  weight       Decimal  @default(1.0)
  createdAt    DateTime @default(now())

  @@map("test_questions")
}

model TestSession {
  id             String            @id @default(cuid())
  testId         String
  test           Test              @relation(fields: [testId], references: [id])
  sessionToken   String            @unique // httpOnly cookie — NEVER log
  userId         String?           // null if anonymous
  answers        Json              @default("{}")
  currentStep    Int               @default(1)
  status         TestSessionStatus @default(in_progress)
  score          Decimal?
  resultBand     String?
  resultSubscores Json?
  ipHash         String?           // SHA-256 of IP
  userAgent      String?
  completedAt    DateTime?
  expiresAt      DateTime?         // 24h after created for anon sessions
  createdAt      DateTime          @default(now())

  @@index([sessionToken])
  @@index([userId])
  @@index([status, expiresAt])
  @@map("test_sessions")
}

model NewsletterSubscriber {
  id             String           @id @default(cuid())
  email          String           @unique
  status         NewsletterStatus @default(pending)
  confirmToken   String?          @unique
  confirmedAt    DateTime?
  source         String?          // 'footer'|'blog_inline'|'test_result'|'popup'
  sourceSlug     String?
  tags           String[]
  ipHash         String?
  createdAt      DateTime         @default(now())
  unsubscribedAt DateTime?

  @@map("newsletter_subscribers")
}

model LeadSubmission {
  id          String   @id @default(cuid())
  type        LeadType
  fullName    String
  email       String
  companyName String?
  companySize String?
  phone       String?
  message     String?
  intent      String?  // 'demo'|'pricing'|'partnership'|'other'
  utmSource   String?
  utmMedium   String?
  utmCampaign String?
  status      String   @default("new") // 'new'|'contacted'|'qualified'|'closed'
  ipHash      String?
  createdAt   DateTime @default(now())

  @@map("lead_submissions")
}

model CooperationRequest {
  id                    String   @id @default(cuid())
  fullName              String
  email                 String
  phone                 String?
  specialization        String?
  licenseInfo           String?
  message               String?
  status                String   @default("new")
  convertedTherapistId  String?
  createdAt             DateTime @default(now())

  @@map("cooperation_requests")
}

model FaqItem {
  id          String   @id @default(cuid())
  question    String
  answer      String
  category    String?
  sortOrder   Int      @default(0)
  isPublished Boolean  @default(true)
  createdAt   DateTime @default(now())

  @@map("faq_items")
}

model ContentPage {
  id             String   @id @default(cuid())
  slug           String   @unique
  title          String
  blocks         Json     @default("[]")
  seoTitle       String?
  seoDescription String?
  ogImageUrl     String?
  isPublished    Boolean  @default(false)
  publishedAt    DateTime?
  updatedAt      DateTime @updatedAt

  @@map("content_pages")
}

model MediaAsset {
  id           String   @id @default(cuid())
  storagePath  String
  publicUrl    String
  filename     String
  mimeType     String
  sizeBytes    Int?
  width        Int?
  height       Int?
  altText      String?
  blurhash     String?
  uploadedById String?
  uploadedBy   Profile? @relation(fields: [uploadedById], references: [id])
  createdAt    DateTime @default(now())

  @@map("media_assets")
}
```

### Supabase Row-Level Security

Prisma is the primary data access layer. Supabase RLS is a defense-in-depth layer. Critical policies:

```sql
-- PHI: session_notes — only therapist who wrote it, or admin
ALTER TABLE session_notes ENABLE ROW LEVEL SECURITY;
CREATE POLICY "therapist_owns_notes"
  ON session_notes FOR ALL
  USING (therapist_id = auth.uid()
    OR EXISTS (SELECT 1 FROM profiles WHERE id = auth.uid() AND role IN ('admin','super_admin')));

-- Public: blog_posts — published only
ALTER TABLE blog_posts ENABLE ROW LEVEL SECURITY;
CREATE POLICY "public_published_posts"
  ON blog_posts FOR SELECT
  USING (status = 'published' AND published_at <= now());

-- Public: tests — published only
CREATE POLICY "public_published_tests"
  ON tests FOR SELECT
  USING (status = 'published' AND published_at <= now());

-- test_sessions: own by token or user_id
CREATE POLICY "own_test_session"
  ON test_sessions FOR ALL
  USING (session_token = current_setting('app.session_token', true)
    OR user_id = auth.uid());

-- newsletter_subscribers: INSERT only for anon
CREATE POLICY "newsletter_insert_only"
  ON newsletter_subscribers FOR INSERT WITH CHECK (true);
```

---

### 5. Domain Boundaries

| Domain | Owns | Package | Depends On |
|---|---|---|---|
| **Identity & Auth** | Clerk profile sync, `profiles` | `@repo/auth` | — |
| **Therapist Onboarding** | `therapist_profiles`, credentials, review queue | `@repo/database` | Identity |
| **Scheduling** | `availability_schedules`, `exceptions`, slot computation | `@repo/database` | Identity, Therapist |
| **Sessions** | `sessions`, `bookings`, Daily.co rooms | `@repo/database` | Scheduling |
| **Clinical Notes** | `session_notes` (encrypted) | `@repo/database` | Sessions, Identity |
| **Messaging** | `conversations`, `messages` (encrypted) | `@repo/realtime` | Identity |
| **Payments (B2C)** | `payments`, `payouts`, Stripe Connect | `@repo/payments` | Scheduling |
| **B2B** | `organizations`, `org_members`, `org_invoices` | `@repo/payments`, `@repo/database` | Identity, Payments |
| **Disputes** | `disputes`, refund logic | `@repo/payments` | Payments, Sessions |
| **Notifications** | In-app (Knock) + email (Resend) | `@repo/notifications`, `@repo/email` | All domains |
| **Reviews** | `reviews`, moderation | `@repo/database` | Sessions |
| **Admin** | `admin_audit_logs`, `platform_settings`, tools | `apps/admin` | All domains |
| **Public / CMS** | Blog, tests, leads, newsletter | `@repo/cms`, `@repo/security` | Identity (optional) |
| **Analytics** | PostHog, materialized views | `@repo/analytics` | All domains |

---

### 6. Sensitive Data Inventory

| Field / Context | Classification | Handling |
|---|---|---|
| `session_notes.content` | PHI (HIPAA) | AES-256 encrypt before Prisma INSERT, decrypt on SELECT |
| `messages.content` | PHI | AES-256 encrypt before Prisma INSERT |
| `client_profiles.dateOfBirth` | PII | Encrypted column |
| `client_profiles.emergencyContact*` | PII | Encrypted column |
| `client_profiles.insuranceMemberId` | PHI | Encrypted column |
| `therapist_profiles.licenseNumber` | PII | Encrypted column |
| `therapist_profiles.stripeAccountId` | Financial PII | Prisma field-level access control + RLS |
| `organizations.ssoConfig` | Security credential | Application-layer encrypt before Prisma INSERT |
| `organizations.stripeCustomerId` | Financial PII | Restricted access |
| `payments.stripePaymentIntentId` | Financial | Never logged; `@repo/payments` only |
| `sessions.recordingUrl` | PHI | Consent required; Supabase Storage signed URL (1h) |
| `test_sessions.sessionToken` | Pseudo-PII | httpOnly cookie; never logged; 24h auto-purge for anon |
| `newsletter_subscribers.email` | PII | Service-role only SELECT; no enumeration in API |
| Video stream | PHI | Daily.co DTLS-SRTP E2E encrypted |
| Credential documents | PII | Supabase Storage private bucket; 15-min signed URL |
| `admin_audit_logs` | Operational | Immutable; read-only to super_admin; 7-year retention |
| `ip_hash` (all tables) | Pseudonymous | SHA-256 without reversible dictionary |

**⚠️ HIPAA:** BAAs required with Supabase (Pro+), Daily.co, Resend, PostHog before US production.

**⚠️ GDPR:** `test_sessions` anonymous data + `session_token` may constitute health data. 24h purge mandatory. Double opt-in mandatory for EU newsletter subscribers.

---

### 7. RBAC Matrix

| Resource / Action | CLIENT | B2B_MEMBER | THERAPIST | ADMIN (support) | ADMIN (finance) | ADMIN (operations) | SUPER_ADMIN |
|---|---|---|---|---|---|---|---|
| View own profile | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Edit own profile | ✅ | ✅ | ✅ | ❌ | ❌ | ✅ | ✅ |
| View other user profiles | ❌ | ❌ | own clients | ✅ read | ✅ read | ✅ | ✅ |
| Suspend / ban user | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ |
| Book session | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Confirm / cancel booking | ❌ | ❌ | own only | ✅ | ❌ | ✅ | ✅ |
| Create session note | ❌ | ❌ | own sessions | ❌ | ❌ | ❌ | ❌ |
| Read session notes | own (if shared) | own (if shared) | own only | ✅ audit | ❌ | ✅ | ✅ |
| Send message | ✅ | ✅ | own clients | ❌ | ❌ | ❌ | ❌ |
| Set availability | ❌ | ❌ | own only | ❌ | ❌ | ❌ | ❌ |
| Approve therapist | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ |
| View payments | own only | own only | own only | ✅ read | ✅ | ✅ | ✅ |
| Issue refund | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ |
| Manage platform settings | ❌ | ❌ | ❌ | ❌ | partial | ✅ | ✅ |
| Manage admin accounts | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |
| Create B2B organization | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ |
| View org usage reports | ❌ | org_admin only | ❌ | ✅ | ✅ | ✅ | ✅ |
| View audit logs | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ read | ✅ |
| Execute tests | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Manage blog / tests (CMS) | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ |
| View lead submissions | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ | ✅ |

---

## PHASE 1 — CORE PLATFORM {#phase-1}

---

### 8. Pages & Routes

#### `apps/web` — Public Routes (althena.com)
```
/                                 → Landing page
/about
/pricing
/for-business
/cooperation
/therapists                       → Public directory
/therapists/[slug]
/blog
/blog/[slug]
/blog/category/[categorySlug]
/tests
/tests/[slug]
/tests/[slug]/start
/tests/[slug]/question/[step]
/tests/[slug]/result
/sign-in                          → Clerk <SignIn />
/sign-up                          → Clerk <SignUp />
/sign-up/client                   → post-signup role selection
/sign-up/therapist
/sign-up/b2b                      → invite token accept
/terms · /privacy · /cookies
```

#### `apps/app` — Authenticated Routes (app.althena.com)

**Client:**
```
/                                 → Redirect → /home
/home
/find-therapist
/therapists/[slug]
/bookings
/bookings/[bookingId]
/messages
/messages/[conversationId]
/billing
/settings
/settings/preferences
/settings/organization            ← B2B only
/settings/organization/members
/settings/organization/reports
/settings/organization/billing
```

**Therapist:**
```
/overview
/calendar
/clients
/clients/[clientId]
/sessions
/sessions/[bookingId]
/sessions/[bookingId]/note
/messages
/messages/[convoId]
/earnings
/analytics
/settings
/settings/availability
/settings/payout
/settings/notifications
/onboarding
/onboarding/profile
/onboarding/credentials
/onboarding/availability
/onboarding/payout
```

**Video (isolated layout):**
```
/video/[roomId]                   → Daily.co session room, no sidebar
```

#### `apps/admin` — Admin Routes (admin.althena.com)
```
/overview
/users · /users/[userId]
/therapists · /therapists/[id] · /therapists/pending
/clients
/sessions · /sessions/[bookingId]
/payments · /payments/[paymentId]
/payouts
/disputes · /disputes/[disputeId]
/organizations · /organizations/new
/organizations/[orgId]
/organizations/[orgId]/members · /billing
/reviews
/analytics · /analytics/revenue · /sessions · /therapists · /b2b
/settings · /settings/general · /commission · /payments · /email
/settings/notifications · /b2b · /compliance
/blog · /blog/new · /blog/[postId]
/tests · /tests/new · /tests/[testId]
/media
/leads · /leads/[leadId]
/admins                           ← super_admin only
/admins/new · /admins/[adminId]
/audit-log
```

#### `apps/api` — API Webhook Routes (api.althena.com)
```
GET  /health
POST /webhooks/clerk              → Clerk user sync → prisma upsert
POST /webhooks/stripe             → payment/payout events → @repo/webhooks verify
POST /webhooks/daily              → Daily.co session events
```

---

### 9. Sidebar Navigation

#### 9.1 Client Sidebar (`apps/app` — client role)
```
🌿 Althena          [Clerk UserButton]
─────────────────────────────────
🏠  Home
🔍  Find a Therapist
📅  My Sessions
💬  Messages              [unread count — realtime]
💳  Billing
─────────────────────────────────
⚙️  Settings
❓  Help & Support
─────────────────────────────────
Footer (B2B member only):
🏢  [Org Name] Plan · 2 / 4 sessions used
```

#### 9.2 Therapist Sidebar
```
🌿 Althena          [Clerk UserButton]
                    ● Online ▾
─────────────────────────────────
MAIN
📊  Overview
📅  Calendar
👥  My Clients
🎯  Sessions
💬  Messages              [unread count]
─────────────────────────────────
FINANCE
💰  Earnings
📈  Analytics
─────────────────────────────────
ACCOUNT
⚙️  Settings
   ↳ Profile · Availability · Payout · Notifications
─────────────────────────────────
❓  Help · 🚪 Log Out
```

Status toggle (avatar menu): 🟢 Active · 🟡 On Leave · ⛔ Do Not Disturb

#### 9.3 Admin Sidebar (`apps/admin`)
```
🌿 Althena ADMIN     [Clerk UserButton]
Role: Operations Admin
─────────────────────────────────
OVERVIEW       📊 Dashboard
USER MGMT      👥 All Users · 🩺 Therapists [pending badge] · 👤 Clients
OPERATIONS     📅 Sessions · ⚠️ Disputes [badge] · ⭐ Reviews
FINANCE        💳 Payments · 💸 Payouts
B2B            🏢 Organizations
CONTENT        📝 Blog · 🧪 Tests · 📥 Leads
ANALYTICS      📈 Revenue · Sessions · Therapists · B2B
PLATFORM       ⚙️ Settings · 👮 Admins (super_admin only) · 📋 Audit Log
─────────────────────────────────
🚪 Log Out
```

Admin sidebar visibility by tier: same as previously specified (support/moderator/finance/operations/super_admin columns).

---

### 10. Forms

All forms use **react-hook-form** + **Zod** schemas. Validation shared between client and server via the same Zod schema imported from `@repo/database` or a dedicated `@repo/validators` package.

#### 10.1 Therapist Registration
```typescript
const TherapistRegistrationSchema = z.object({
  email:          z.string().email().max(254),
  password:       z.string().min(8).regex(/^(?=.*[A-Z])(?=.*\d)(?=.*[!@#$%^&*])/),
  confirmPassword: z.string(),
  fullName:       z.string().min(2).max(100),
  acceptTerms:    z.literal(true),
  acceptPrivacy:  z.literal(true),
}).refine(d => d.password === d.confirmPassword, {
  message: 'Passwords do not match', path: ['confirmPassword'],
});
```

#### 10.2 Therapist Profile Setup
```typescript
const TherapistProfileSchema = z.object({
  displayName:       z.string().min(2).max(60),
  bio:               z.string().min(50).max(800),
  specializations:   z.array(z.string()).min(1),
  approaches:        z.array(z.string()).min(1),
  languages:         z.array(z.string()).min(1),
  yearsOfExperience: z.number().int().min(0).max(50),
  sessionRateCents:  z.number().int().min(3000).max(50000),
  sessionTypes:      z.array(z.enum(['video','audio','chat'])).min(1),
  timezone:          z.string().min(1),
  phone:             z.string().regex(/^\+?[\d\s\-().]{7,20}$/).optional(),
  acceptsInsurance:  z.boolean(),
  insuranceProviders: z.array(z.string()).optional(),
});
```

#### 10.3 Credential Upload
```typescript
const CredentialSchema = z.object({
  licenseNumber:   z.string().min(4).max(20).regex(/^[A-Z0-9]+$/i),
  licenseState:    z.string().length(2),       // ISO 3166-2 US state code
  licenseExpiry:   z.date().min(new Date()),
  npiNumber:       z.string().length(10).optional(),
  // Files validated via Zod .instanceof(File) on client,
  // mime + size enforced on server via @repo/storage
});
```

#### 10.4 Availability Schedule
```typescript
const AvailabilitySlotSchema = z.object({
  dayOfWeek:          z.number().int().min(0).max(6),
  startTime:          z.string().regex(/^([0-1]\d|2[0-3]):[0-5]\d$/),
  endTime:            z.string().regex(/^([0-1]\d|2[0-3]):[0-5]\d$/),
  slotDurationMinutes: z.enum(['30','50','60','90']).transform(Number),
}).refine(d => d.endTime > d.startTime, { message: 'endTime must be after startTime' });
```

#### 10.5 Session Note
```typescript
const SessionNoteSchema = z.object({
  noteType:          z.enum(['soap','progress','intake','discharge']),
  content:           z.string().min(20).max(10000),
  isSharedWithClient: z.boolean().default(false),
  // SOAP subfields (required when noteType === 'soap')
  subjective:  z.string().optional(),
  objective:   z.string().optional(),
  assessment:  z.string().optional(),
  plan:        z.string().optional(),
});
```

#### 10.6 Create Organization (Admin)
```typescript
const CreateOrganizationSchema = z.object({
  name:                    z.string().min(2).max(100),
  domain:                  z.string().optional(),
  planType:                z.enum(['per_seat','credit_pack','unlimited','enterprise']),
  seatsPurchased:          z.number().int().min(1).max(10000).optional(),
  sessionsPerSeatPerMonth: z.number().int().min(1).max(20).optional(),
  billingEmail:            z.string().email(),
  billingContactName:      z.string().min(2),
  contractStart:           z.date(),
  contractEnd:             z.date().optional(),
  stripeCustomerId:        z.string().optional(),
});
```

#### 10.7 Invite Organization Member (bulk)
```typescript
const InviteMemberSchema = z.object({
  emails: z.array(z.string().email()).min(1).max(50),
  role:   z.enum(['member','org_admin']).default('member'),
});
```

#### 10.8 Platform Settings (Admin)
```typescript
const PlatformSettingsSchema = z.object({
  commissionRatePercent:    z.number().min(0).max(50).multipleOf(0.5),
  cancellationWindowHours:  z.number().int().min(1).max(72),
  cancellationFeePercent:   z.number().min(0).max(100),
  therapistMinRateCents:    z.number().int().min(0),
  therapistMaxRateCents:    z.number().int().min(0),
  payoutSchedule:           z.enum(['daily','weekly','biweekly','monthly']),
  sessionDurationOptions:   z.array(z.number().int()),
  defaultSessionDuration:   z.number().int(),
  supportEmail:             z.string().email(),
  platformName:             z.string().min(1).max(100),
});
```

#### 10.9 Dispute Resolution (Admin)
```typescript
const DisputeResolutionSchema = z.object({
  status:           z.enum(['resolved_refund','resolved_no_action']),
  resolutionNote:   z.string().min(20),
  refundType:       z.enum(['full','partial']).optional(),
  refundAmountCents: z.number().int().min(1).optional(),
}).refine(d => {
  if (d.status === 'resolved_refund' && !d.refundType) return false;
  if (d.refundType === 'partial' && !d.refundAmountCents) return false;
  return true;
});
```

#### 10.10 Admin Invite (Super Admin)
```typescript
const AdminInviteSchema = z.object({
  email:    z.string().email(),
  fullName: z.string().min(2).max(100),
  tier:     z.enum(['support','moderator','finance','operations']),
});
```

---

### 11. Tables & Lists

All list views use **cursor-based pagination** (no OFFSET). Implemented via Prisma `cursor` + `take`. Filtering via URL search params with Zod coercion.

#### 11.1 Therapist List (`/therapists`)
Columns: Avatar + Name | Specializations | Status | Sessions | Avg Rating | Joined | Actions
Filters: Status (multi), Specialization, Language, Verified/Unverified
Sorting: `joinedAt` desc (default), `fullName`, `totalSessionsCompleted`, `avgRating`
Actions: View · Approve · Suspend · Message

#### 11.2 Pending Verification Queue (`/therapists/pending`)
Columns: Name | License # | State | Submitted | Days Waiting | Actions
Sorting: `createdAt` asc (oldest first)
Actions: Review (credential modal) · Approve · Reject

#### 11.3 All Users (`/users`)
Columns: Avatar + Name | Role | Email | Status | Last Active | Joined | Actions
Filters: Role, Status, Joined date range
Search: full-text on `fullName`, `email`

#### 11.4 All Sessions (`/sessions`)
Columns: Date/Time | Client | Therapist | Type | Duration | Status | Amount | Actions
Filters: Status, Type, Date range, Amount range

#### 11.5 Payments (`/payments`)
Columns: Date | Client | Therapist | Amount | Platform Fee | Status | Stripe ID | Actions
Filters: Status, Date range, Amount range, Org (B2B)
Summary row: Total revenue, total fees, total refunded (filtered period)

#### 11.6 Disputes Queue (`/disputes`)
Columns: ID | Client | Therapist | Booking Date | Reason (truncated) | Status | Opened | Actions
Filters: Status (open/under_review/resolved)
Sorting: `createdAt` asc (oldest open first)

#### 11.7 Organizations (`/organizations`)
Columns: Logo + Name | Plan | Seats | Active Members | Sessions This Month | MRR | Status | Actions
Filters: Plan type, Status

#### 11.8 Org Members (`/organizations/[orgId]/members`)
Columns: Name | Email | Role | Status | Sessions Used | Remaining | Joined | Actions
Actions: Deactivate · Make org_admin · Resend invite

#### 11.9 Audit Log (`/audit-log`)
Columns: Timestamp | Admin | Action | Target Type | Target ID | IP Address | Details
Sorting: `createdAt` desc (immutable)
Export: CSV (Server Action → Prisma query → stream)

#### 11.10 Reviews (`/reviews`)
Columns: Date | Client | Therapist | Rating | Comment (truncated) | Public | Flagged | Actions
Filters: Flagged only, Rating range, Public/Private
Actions: Unflag · Remove (soft delete) · Make private

#### 11.11 Lead Submissions (`/leads`)
Columns: Date | Name | Email | Company | Type | Intent | Status | Actions
Filters: Type, Status, Date range
Actions: Update status · View detail · Convert to org

#### 11.12 Blog Posts (`/blog`)
Columns: Title | Category | Status | Author | Published | Views | Actions
Filters: Status, Category, Author
Actions: Edit · Publish/Unpublish · Archive · Delete

---

### 12. API Endpoints

#### `apps/app` — Authenticated REST API (Next.js Route Handlers)

All routes require Clerk session. Role check via `@repo/auth` middleware helper.

```
Auth (handled by Clerk — no custom endpoints needed for login/logout)
POST /api/webhooks/clerk           → apps/api (role sync to Prisma)

Profiles
GET    /api/me
PATCH  /api/me
DELETE /api/me                     → Clerk account delete + Prisma soft-delete

Therapist (public browse — no auth)
GET    /api/therapists             → filter, search, paginate
GET    /api/therapists/[id]        → public profile
GET    /api/therapists/[id]/reviews
GET    /api/therapists/me          → own profile (therapist auth)
PATCH  /api/therapists/me

Availability
GET    /api/availability/[therapistId]
POST   /api/availability
DELETE /api/availability/[id]
GET    /api/availability/[therapistId]/slots?date=YYYY-MM-DD
POST   /api/availability/exceptions
DELETE /api/availability/exceptions/[id]

Bookings
GET    /api/bookings
POST   /api/bookings               → B2B coverage check → Stripe Payment Intent
GET    /api/bookings/[id]
PATCH  /api/bookings/[id]/confirm
PATCH  /api/bookings/[id]/cancel
PATCH  /api/bookings/[id]/no-show

Sessions
POST   /api/sessions/[bookingId]/start  → Daily.co room creation
PATCH  /api/sessions/[bookingId]/end
GET    /api/sessions/[bookingId]

Session Notes
GET    /api/notes?bookingId=[id]
POST   /api/notes                  → encrypt before Prisma INSERT
PATCH  /api/notes/[id]
DELETE /api/notes/[id]

Messages
GET    /api/conversations
POST   /api/conversations
GET    /api/conversations/[id]/messages
POST   /api/conversations/[id]/messages  → encrypt before INSERT
PATCH  /api/conversations/[id]/read

Clients (Therapist view)
GET    /api/clients
GET    /api/clients/[id]
GET    /api/clients/[id]/sessions
GET    /api/clients/[id]/notes

Payments
POST   /api/payments/intent        → Stripe Payment Intent
POST   /api/stripe/connect/onboard → Stripe Connect Express onboarding link
GET    /api/stripe/connect/status
GET    /api/payouts
GET    /api/payouts/[id]

Notifications
GET    /api/notifications
PATCH  /api/notifications/[id]/read
PATCH  /api/notifications/read-all

Analytics (Therapist)
GET    /api/analytics/overview
GET    /api/analytics/sessions
GET    /api/analytics/clients

Disputes
POST   /api/disputes               → client raises dispute
GET    /api/disputes/[id]
PATCH  /api/disputes/[id]/resolve  → admin only

Reviews
POST   /api/reviews                → client only, post-session
GET    /api/reviews?therapistId=[id]
PATCH  /api/reviews/[id]/flag

B2B Organizations
GET    /api/organizations/[id]
PATCH  /api/organizations/[id]
GET    /api/organizations/[id]/members
POST   /api/organizations/[id]/members/invite
PATCH  /api/organizations/[id]/members/[memberId]
DELETE /api/organizations/[id]/members/[memberId]
GET    /api/organizations/[id]/usage
GET    /api/organizations/[id]/invoices
GET    /api/organizations/[id]/therapist-pool
POST   /api/organizations/[id]/therapist-pool
DELETE /api/organizations/[id]/therapist-pool/[therapistId]
```

#### `apps/admin` — Admin API

All require admin JWT + TOTP 2FA session check.

```
GET/PATCH  /api/admin/users/[id] + /suspend + /restore + DELETE
GET/PATCH  /api/admin/therapists/[id] + /approve + /reject + /suspend
GET        /api/admin/therapists/pending
GET        /api/admin/sessions · /payments · /payouts
PATCH      /api/admin/payouts/[id]/retry
GET/PATCH  /api/admin/disputes/[id]/resolve
GET/PATCH  /api/admin/reviews/[id]/remove + /unflag
GET/POST/GET /api/admin/organizations + /[id]
GET        /api/admin/analytics/overview + /revenue + /sessions + /therapists + /b2b
GET/PATCH  /api/admin/settings
GET        /api/admin/audit-log
-- CMS
GET/POST/PATCH/DELETE /api/admin/blog + /[id]/publish + /unpublish
GET/POST/PATCH        /api/admin/tests + /[id]
GET/PATCH             /api/admin/leads + /[id]/status
POST                  /api/admin/media/upload
-- Super admin only
GET/POST/PATCH/DELETE /api/admin/admins + /invite + /[id]/tier
```

#### `apps/api` — Webhooks

```
GET  /health                       → 200 { status: 'ok' }
POST /webhooks/clerk               → verify Svix signature → sync profile to Prisma
POST /webhooks/stripe              → verify Stripe signature → update payments/payouts
POST /webhooks/daily               → Daily.co session events → update sessions table
```

#### Public API (in `apps/web`)

```
POST /api/public/newsletter
GET  /api/public/newsletter/confirm?token=[token]
GET  /api/public/newsletter/unsubscribe?token=[token]
POST /api/public/leads
POST /api/public/cooperation
POST /api/public/tests/[testId]/session
GET  /api/public/tests/[testId]/session?token=[token]
PATCH /api/public/tests/[testId]/answer
POST /api/public/tests/[testId]/complete
POST /api/public/tests/[testId]/claim     ← auth required
POST /api/public/blog/[slug]/view         ← fire-and-forget
GET  /api/public/faq?category=[category]
```

---

### 13. Real-time Functionality

**Provider:** Supabase Realtime via `@repo/realtime`

```typescript
// packages/realtime/src/channels.ts
import { createClient } from '@supabase/supabase-js';

// Per-conversation messages
export function messageChannel(conversationId: string, handler: (msg: Message) => void) {
  return supabase
    .channel(`conversation:${conversationId}`)
    .on('postgres_changes', {
      event: 'INSERT', table: 'messages',
      filter: `conversation_id=eq.${conversationId}`
    }, handler);
}

// Per-user notifications (augments Knock — fallback)
export function notificationChannel(userId: string, handler: (n: Notification) => void) {
  return supabase
    .channel(`notifications:${userId}`)
    .on('postgres_changes', {
      event: 'INSERT', table: 'notifications',
      filter: `user_id=eq.${userId}`
    }, handler);
}

// Booking updates (therapist)
export function bookingChannel(therapistId: string, handler: (b: Booking) => void) {
  return supabase
    .channel(`bookings:${therapistId}`)
    .on('postgres_changes', {
      event: 'UPDATE', table: 'bookings',
      filter: `therapist_id=eq.${therapistId}`
    }, handler);
}

// Admin queue badges
export function adminQueueChannel(handlers: { dispute: () => void; pending: () => void }) {
  return supabase
    .channel('admin:queues')
    .on('postgres_changes', { event: '*', table: 'disputes' }, handlers.dispute)
    .on('postgres_changes', {
      event: '*', table: 'therapist_profiles',
      filter: `onboarding_status=eq.pending`
    }, handlers.pending);
}

// Therapist presence
export function presenceChannel(userId: string, status: string) {
  return supabase
    .channel('presence:therapists')
    .on('presence', { event: 'sync' }, () => {})
    .track({ userId, status });
}
```

**Video:** Daily.co per-booking rooms. Room created server-side on `POST /api/sessions/[bookingId]/start`. Both parties receive short-lived Daily.co JWT tokens. `/video/[roomId]` is a full-screen isolated layout in `apps/app` — no sidebar chrome.

**Notifications:** Primary delivery via `@repo/notifications` (Knock). Supabase Realtime is the fallback for in-app badge updates.

**Scheduled jobs (Supabase Edge Functions):**

| Job | Schedule | Action |
|---|---|---|
| Booking reminder | Cron: every hour | T-24h, T-1h emails via Resend |
| Post-session review | Cron: every hour | T+1h review request to client |
| Weekly payout notification | Cron: Monday 09:00 | Payout summary to therapist |
| B2B invoice generation | Cron: 1st of month 06:00 | Aggregate bookings → Stripe Invoice |
| Org usage report | Cron: 1st of month 07:00 | Compute → `organization_usage_reports` |
| Anonymous session purge | Cron: daily 03:00 | DELETE test_sessions WHERE anon + expired |

---

### 14. File Functionality

**Provider:** `@repo/storage` wraps Supabase Storage

| File Type | Bucket | Access | Max Size | CDN | Processing |
|---|---|---|---|---|---|
| Avatar | `avatars` | Public | 5 MB | ✅ | Sharp → 400×400 WebP |
| Credential docs | `credentials` | Private, signed URL 15 min | 10 MB | ❌ | None |
| Message attachments | `attachments` | Private, signed URL 1h | 20 MB | ❌ | None |
| Session recordings | `recordings` | Private, signed URL 1h | 2 GB | ❌ | None |
| Org logos | `org-assets` | Public | 2 MB | ✅ | Sharp → 200×200 WebP |
| Blog/test covers | `public-media` | Public | 5 MB | ✅ | Sharp → 400w/800w/1200w WebP + 1200×630 OG |
| OG images | `public-media/og` | Public | 2 MB | ✅ | next/og ImageResponse (Edge) |
| Email templates | `email-templates` | Admin-only | 500 KB | ❌ | None |
| Platform assets | `platform-assets` | Public | 10 MB | ✅ | None |

**Upload flow:**
1. Client requests presigned URL from `@repo/storage`
2. Client uploads directly to Supabase Storage
3. Supabase Edge Function triggered on upload → Sharp processing → variants stored
4. `media_assets` record created via Prisma with `blurhash` (4×3 components)
5. API responses reference signed URLs fetched at read time — never raw storage URLs

---

### 15. Role-Based Navigation Components

```typescript
// packages/design-system/src/providers.tsx
export function DesignSystemProvider({ children, role }: Props) {
  return (
    <ClerkProvider>
      <ThemeProvider>
        <RoleProvider role={role}>
          {children}
        </RoleProvider>
      </ThemeProvider>
    </ClerkProvider>
  );
}

// Each app composes its own shell
// apps/app — ClientShell or TherapistShell based on Clerk publicMetadata.role
// apps/admin — AdminShell with tier-filtered nav
```

**Clerk Auth Middleware (shared in `@repo/auth`):**

```typescript
// packages/auth/src/middleware.ts
import { clerkMiddleware, createRouteMatcher } from '@clerk/nextjs/server';

const isPublicRoute = createRouteMatcher([
  '/', '/about', '/pricing', '/for-business', '/cooperation',
  '/therapists(.*)', '/blog(.*)', '/tests(.*)',
  '/sign-in(.*)', '/sign-up(.*)',
  '/api/public(.*)',
  '/terms', '/privacy', '/cookies',
]);

const isAdminRoute = createRouteMatcher(['/admin(.*)']);
const isTherapistRoute = createRouteMatcher(['/dashboard(.*)']);

export default clerkMiddleware(async (auth, req) => {
  if (!isPublicRoute(req)) {
    await auth.protect();
  }

  const { sessionClaims } = await auth();
  const role = sessionClaims?.publicMetadata?.role as string;

  if (isAdminRoute(req) && !['admin', 'super_admin'].includes(role)) {
    return Response.redirect(new URL('/unauthorized', req.url));
  }
  if (isTherapistRoute(req) && role !== 'therapist') {
    return Response.redirect(new URL('/unauthorized', req.url));
  }
});

export const config = {
  matcher: ['/((?!_next|[^?]*\\.(?:html?|css|js(?!on)|jpe?g|webp|png|gif|svg|ttf|woff2?|ico|csv|docx?|xlsx?|zip|webmanifest)).*)', '/(api|trpc)(.*)'],
};
```

---

## PHASE 2 — PUBLIC SITE (Guest Domain) {#phase-2}

---

### 16. Public Domain Entities

See Section 4 Prisma schema for full DDL: `BlogPost`, `BlogAuthor`, `BlogCategory`, `Test`, `TestQuestion`, `TestSession`, `NewsletterSubscriber`, `LeadSubmission`, `CooperationRequest`, `FaqItem`, `ContentPage`, `MediaAsset`.

**CMS approach:** `@repo/cms` uses **Content Collections** (MDX files in `packages/cms/content/`) for blog posts and test content — provides full TypeScript type safety, no external CMS dependency. Admin CMS UI in `apps/admin` reads/writes both the Prisma DB (for metadata) and the content files (for rich content). For dynamic per-user data (test sessions, leads, newsletter), Prisma is used directly.

---

### 17. Route Architecture (`apps/web`)

```
apps/web/app/
├── (public)/
│   ├── layout.tsx                   ← PublicLayout (nav, footer, cookie banner)
│   ├── page.tsx                     → /                      [ISR: 3600s]
│   ├── about/page.tsx               → /about                 [ISR: 86400s]
│   ├── pricing/page.tsx             → /pricing               [ISR: 3600s]
│   ├── for-business/page.tsx        → /for-business          [ISR: 3600s]
│   ├── cooperation/page.tsx         → /cooperation           [ISR: 3600s]
│   ├── blog/
│   │   ├── layout.tsx               ← BlogLayout (sidebar)
│   │   ├── page.tsx                 → /blog                  [ISR: 60s]
│   │   ├── [slug]/page.tsx          → /blog/[slug]           [ISR: 300s, generateStaticParams top 100]
│   │   └── category/[slug]/page.tsx → /blog/category/[slug] [ISR: 60s]
│   ├── tests/
│   │   ├── page.tsx                 → /tests                 [ISR: 300s]
│   │   └── [slug]/
│   │       ├── page.tsx             → /tests/[slug]          [ISR: 300s, generateStaticParams]
│   │       ├── start/page.tsx       → /tests/[slug]/start    [SSR — session detection]
│   │       ├── question/[step]/page.tsx → /tests/[slug]/question/[step] [Client Component]
│   │       └── result/page.tsx      → /tests/[slug]/result   [SSR, noindex]
│   ├── therapists/
│   │   ├── page.tsx                 → /therapists            [SSR — filter params vary]
│   │   └── [slug]/page.tsx          → /therapists/[slug]     [ISR: 300s]
│   └── (static)/
│       ├── terms/page.tsx · privacy/page.tsx · cookies/page.tsx  [SSG]
│
├── api/public/                      ← Public Route Handlers (no auth)
│   ├── newsletter/route.ts
│   ├── newsletter/confirm/route.ts
│   ├── newsletter/unsubscribe/route.ts
│   ├── leads/route.ts
│   ├── cooperation/route.ts
│   ├── tests/[testId]/session/route.ts
│   ├── tests/[testId]/answer/route.ts
│   ├── tests/[testId]/complete/route.ts
│   ├── tests/[testId]/claim/route.ts   ← Clerk auth required
│   ├── blog/[slug]/view/route.ts
│   └── faq/route.ts
│
├── sitemap.ts                       → /sitemap.xml
├── robots.ts                        → /robots.txt
└── opengraph-image.tsx              → dynamic OG images via next/og
```

---

### 18. Public Forms

All use Zod schemas + Arcjet rate limiting (`@repo/security`). Honeypot field (`name="website"`, visually hidden via CSS) on every form.

#### 18.1 NewsletterForm
```typescript
const NewsletterSchema = z.object({
  email:       z.string().email().max(254),
  source:      z.enum(['footer', 'blog_inline', 'test_result', 'popup']),
  sourceSlug:  z.string().optional(),
  website:     z.string().max(0), // honeypot
});
// Arcjet: 3 requests/IP/hour
// Always return 200 { message } — never expose email existence
// Side effect: INSERT newsletter_subscribers, send Resend double opt-in email (EU mandatory)
```

#### 18.2 BusinessInquiryForm
```typescript
const BusinessInquirySchema = z.object({
  fullName:    z.string().min(2).max(100),
  email:       z.string().email().max(254),
  companyName: z.string().min(1).max(200),
  companySize: z.enum(['1-10','11-50','51-200','201-1000','1000+']),
  phone:       z.string().regex(/^\+?[\d\s\-().]{7,20}$/).optional(),
  intent:      z.enum(['demo','pricing','partnership','other']),
  message:     z.string().min(10).max(1000).optional(),
  utmSource:   z.string().max(100).optional(),
  utmMedium:   z.string().max(100).optional(),
  utmCampaign: z.string().max(100).optional(),
  website:     z.string().max(0),
});
// Arcjet: 5 requests/IP/hour
// On success: inline thank-you, no redirect
```

#### 18.3 CooperationForm
```typescript
const CooperationSchema = z.object({
  fullName:       z.string().min(2).max(100),
  email:          z.string().email().max(254),
  phone:          z.string().optional(),
  specialization: z.string().max(200).optional(),
  licenseInfo:    z.string().max(500).optional(),
  message:        z.string().min(20).max(1000),
  website:        z.string().max(0),
});
// Arcjet: 3 requests/IP/hour
```

---

### 19. Public API Endpoints

Rate limiting via `@repo/security` (Arcjet) on all `/api/public/*` routes.

```
Arcjet limits:
  POST /api/public/newsletter         → 3/IP/hour
  POST /api/public/leads              → 5/IP/hour
  POST /api/public/cooperation        → 3/IP/hour
  POST /api/public/tests/*/session    → 10/IP/hour
  PATCH /api/public/tests/*/answer    → 120/IP/hour

POST /api/public/newsletter
  Body: NewsletterSchema
  Response: 200 { message } (always 200, no enumeration)
  Side effect: Prisma INSERT, Resend confirmation email

GET  /api/public/newsletter/confirm?token=[token]
  Response: redirect to /?newsletter=confirmed

GET  /api/public/newsletter/unsubscribe?token=[token]
  Response: unsubscribe confirmation page

POST /api/public/leads
  Body: BusinessInquirySchema
  Response: 201 { id } | 400 | 429

POST /api/public/cooperation
  Body: CooperationSchema
  Response: 201 { id } | 400 | 429

POST /api/public/tests/[testId]/session
  Body: { sessionToken? }
  Response: 201 { sessionToken, currentStep, status }
  Side effect: Prisma INSERT, httpOnly cookie set (24h)

GET  /api/public/tests/[testId]/session?token=[token]
  Response: { id, currentStep, status, answersCount, expiresAt }

PATCH /api/public/tests/[testId]/answer
  Body: { sessionToken, questionId, answerValue }
  Response: 200 { nextStep }

POST /api/public/tests/[testId]/complete
  Body: { sessionToken }
  Response: 200 { score, resultBand, resultDescription, ctaVariant }
  Side effect: server-side score, Prisma UPDATE, INCREMENT completionCount

POST /api/public/tests/[testId]/claim
  Auth: Clerk required
  Body: { sessionToken }
  Response: 200 | 404 | 409 (already claimed)
  Side effect: UPDATE test_sessions SET userId = clerkUserId

POST /api/public/blog/[slug]/view
  Response: 200 (always; Prisma increment, non-blocking)

GET  /api/public/faq?category=[cat]
  Response: { items: FaqItem[] }
  Cache-Control: public, max-age=3600
```

---

### 20. SEO Architecture

Implemented via `@repo/seo`:

```typescript
// packages/seo/src/index.ts — exported helpers
export { createMetadata } from './metadata';
export { createJsonLd, blogPostJsonLd, testJsonLd, therapistJsonLd } from './json-ld';
export { createSitemap } from './sitemap';
```

#### Metadata Strategy

```typescript
// apps/web/app/(public)/blog/[slug]/page.tsx
import { createMetadata } from '@repo/seo';

export async function generateMetadata({ params }): Promise<Metadata> {
  const post = await prisma.blogPost.findUnique({ where: { slug: params.slug } });
  if (!post) return {};
  return createMetadata({
    title: post.seoTitle ?? `${post.title} | Althena`,
    description: post.seoDescription ?? post.excerpt ?? '',
    canonical: post.canonicalUrl ?? `https://althena.com/blog/${post.slug}`,
    openGraph: {
      type: 'article',
      images: [{ url: post.ogImageUrl ?? post.coverImageUrl ?? '', width: 1200, height: 630 }],
      publishedTime: post.publishedAt?.toISOString(),
    },
  });
}
```

#### JSON-LD Schema by Page

| Page | Schema Types |
|---|---|
| `/` | `WebSite` + `Organization` + `MedicalOrganization` |
| `/blog` | `WebPage` + `ItemList` |
| `/blog/[slug]` | `Article` or `MedicalWebPage` + `BreadcrumbList` |
| `/tests` | `WebPage` + `ItemList` |
| `/tests/[slug]` | `MedicalWebPage` + `FAQPage` |
| `/therapists/[slug]` | `Person` + `MedicalBusiness` + `BreadcrumbList` |
| `/for-business` | `WebPage` + `Service` |
| `/about` | `AboutPage` + `Organization` |

#### OG Image Generation (Edge Runtime)

```typescript
// apps/web/app/(public)/blog/[slug]/opengraph-image.tsx
import { ImageResponse } from 'next/og';
export const runtime = 'edge';
export default async function OGImage({ params }: { params: { slug: string } }) {
  const post = await prisma.blogPost.findUnique({ where: { slug: params.slug } });
  return new ImageResponse(
    <div style={{ display: 'flex', width: '100%', height: '100%', background: '#0f172a' }}>
      <span style={{ color: 'white', fontSize: 48 }}>{post?.title}</span>
    </div>,
    { width: 1200, height: 630 }
  );
}
```

#### Sitemap (`apps/web/app/sitemap.ts`)

```typescript
export default async function sitemap(): Promise<MetadataRoute.Sitemap> {
  const [posts, tests, therapists] = await Promise.all([
    prisma.blogPost.findMany({ where: { status: 'published' }, select: { slug: true, updatedAt: true } }),
    prisma.test.findMany({ where: { status: 'published' }, select: { slug: true, updatedAt: true } }),
    prisma.therapistProfile.findMany({ where: { onboardingStatus: 'active' }, select: { id: true } }),
  ]);
  return [
    { url: 'https://althena.com', changeFrequency: 'daily', priority: 1.0 },
    { url: 'https://althena.com/blog', changeFrequency: 'daily', priority: 0.9 },
    { url: 'https://althena.com/tests', changeFrequency: 'weekly', priority: 0.8 },
    ...posts.map(p => ({ url: `https://althena.com/blog/${p.slug}`, lastModified: p.updatedAt, priority: 0.7 })),
    ...tests.map(t => ({ url: `https://althena.com/tests/${t.slug}`, priority: 0.7 })),
    ...therapists.map(t => ({ url: `https://althena.com/therapists/${t.id}`, priority: 0.6 })),
    // EXCLUDED: /tests/*/start, /tests/*/question/*, /tests/*/result
    // EXCLUDED: /sign-in, /sign-up, /api/*
  ];
}
```

---

### 21. State Machines (Guest Domain)

See Section 2.9 (Test Execution), 2.10 (Newsletter), and 2.11 (Content Publishing) for full state machines.

#### Progressive Test Answer Saving

```typescript
// apps/web/app/(public)/tests/[slug]/question/[step]/page.tsx (Client Component)
const saveAnswer = useDebouncedCallback(async (questionId: string, value: number) => {
  try {
    await fetch(`/api/public/tests/${testId}/answer`, {
      method: 'PATCH',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ sessionToken, questionId, answerValue: value }),
    });
  } catch {
    // Queue in localStorage for retry
    const queue = JSON.parse(localStorage.getItem('althena_answer_queue') ?? '[]');
    queue.push({ questionId, value, timestamp: Date.now() });
    localStorage.setItem('althena_answer_queue', JSON.stringify(queue));
  }
}, 300);

// Flush queue on next answer or via sendBeacon on unload
window.addEventListener('beforeunload', () => {
  navigator.sendBeacon(`/api/public/tests/${testId}/answer`, JSON.stringify(pendingAnswer));
});
```

---

### 22. Frontend Architecture & Rendering Strategy

#### Server vs Client Component Decisions

| Component | Type | Reason |
|---|---|---|
| Blog post content | Server | No interactivity, SEO-critical, LCP |
| Blog share buttons | Client | `navigator.share` / clipboard |
| Newsletter form | Client | Form state, Arcjet via Route Handler |
| Test question step | Client | Interactive, localStorage, state machine |
| Test progress bar | Client | Local state |
| Therapist catalog filters | Client | URL search params manipulation |
| Therapist card grid | Server | SEO-critical |
| Cookie consent banner | Client | `localStorage` check |
| Lead capture form | Client | Form state, validation |
| FAQ accordion | Client | open/close state |
| Home hero | Server | Above-fold, LCP-critical |
| Clerk `<UserButton>` | Client | Clerk SDK requirement |

#### Rendering Strategy

| Route (apps/web) | Strategy | Notes |
|---|---|---|
| `/` | ISR 3600s | Home/marketing |
| `/blog` | ISR 60s | New posts appear within 60s |
| `/blog/[slug]` | ISR 300s + `generateStaticParams` | Top 100 by viewCount at build |
| `/tests` | ISR 300s | |
| `/tests/[slug]` | ISR 300s + `generateStaticParams` | All published tests |
| `/tests/[slug]/start` | SSR | Session detection (httpOnly cookie) |
| `/tests/[slug]/question/[step]` | Client only (`'use client'`) | Interactive |
| `/tests/[slug]/result` | SSR, `noindex` | Session-specific |
| `/therapists` | SSR | Filter params vary |
| `/therapists/[slug]` | ISR 300s | |
| `/for-business`, `/cooperation` | ISR 3600s | |
| `/about` | ISR 86400s | |
| `/terms`, `/privacy`, `/cookies` | SSG | Never changes |

#### Caching Layers

```
Browser:
  Static assets (JS/CSS): Cache-Control: public, max-age=31536000, immutable
  FAQ API:                 Cache-Control: public, max-age=3600

Vercel Edge Cache:
  ISR pages: CDN-served, invalidated by revalidatePath/revalidateTag on admin publish

Next.js Data Cache:
  Prisma queries in Server Components: tagged via next() options
  Invalidated on admin publish via revalidateTag('blog-posts'), revalidateTag('tests')

Supabase:
  Connection pooling via pgBouncer (poolerUrl in DATABASE_URL)
  Direct connection for migrations (DIRECT_URL)
  Slot computation: Edge Function result cached 5 min per therapist in Vercel KV

Turborepo Remote Cache:
  Build artifacts cached — CI/CD pipeline reuses unchanged package builds
```

---

### 23. Analytics & Tracking Architecture

Implemented via `@repo/analytics` which wraps PostHog + GA4 with consent gating.

#### Event Taxonomy

```typescript
// packages/analytics/src/events.ts
type AnalyticsEvent =
  | { event: 'page_viewed';             props: { page: string; referrer: string } }
  | { event: 'cta_clicked';            props: { ctaId: string; page: string; section: string } }
  | { event: 'article_viewed';         props: { slug: string; category: string } }
  | { event: 'article_scrolled';       props: { slug: string; depthPercent: 25|50|75|100 } }
  | { event: 'test_started';           props: { testSlug: string; category: string } }
  | { event: 'test_question_answered'; props: { testSlug: string; step: number; total: number } }
  | { event: 'test_abandoned';         props: { testSlug: string; step: number } }
  | { event: 'test_completed';         props: { testSlug: string; resultBand: string } }
  | { event: 'test_cta_clicked';       props: { testSlug: string; resultBand: string; ctaVariant: string } }
  | { event: 'newsletter_submitted';   props: { source: string } }
  | { event: 'newsletter_confirmed';   props: {} }
  | { event: 'lead_submitted';         props: { type: string; intent: string } }
  | { event: 'register_started';       props: { intent?: string; role: string } }
  | { event: 'therapist_viewed';       props: { therapistId: string; source: string } }
  | { event: 'booking_cta_clicked';    props: { therapistId: string; page: string } }
```

#### UTM Capture

```typescript
// packages/analytics/src/utm.ts
export function captureUTM(): void {
  const params = new URLSearchParams(window.location.search);
  const utm = (['source','medium','campaign','content','term'] as const)
    .reduce<Record<string, string>>((acc, key) => {
      const val = params.get(`utm_${key}`);
      if (val) acc[`utm_${key}`] = val;
      return acc;
    }, {});
  if (Object.keys(utm).length > 0) {
    sessionStorage.setItem('althena_utm', JSON.stringify(utm));
  }
}
export const getUTM = (): Record<string, string> =>
  JSON.parse(sessionStorage.getItem('althena_utm') ?? '{}');
```

#### Consent Tiers

```typescript
// packages/analytics/src/consent.ts
export type ConsentTier = 0 | 1 | 2; // 0: none, 1: analytics, 2: marketing

export function initAnalytics(tier: ConsentTier) {
  if (tier >= 1) {
    posthog.init(env.NEXT_PUBLIC_POSTHOG_KEY, {
      api_host: 'https://eu.posthog.com',  // EU data residency
      capture_pageview: true,
    });
  }
  if (tier >= 2) {
    // GA4 full tracking
    // Facebook Pixel (if used)
  }
}
```

---

## ADMIN PANEL DEEP DIVE {#admin-panel}

---

### 24. Admin Roles & Permission Architecture

- Only `super_admin` can invite new admins (via Clerk Organizations or custom invite flow)
- Invite flow: email + fullName + tier → Clerk invitation → first login → mandatory TOTP setup
- Session timeout: 4h inactivity (configured via Clerk session settings for admin app)
- All admin actions logged via Prisma to `AdminAuditLog` (immutable — no UPDATE/DELETE policies)
- Suspicious IP detected via Clerk's session security features → re-authentication
- `apps/admin` served on `admin.althena.com` with separate CSP headers in `next.config.ts`

---

### 25. Admin Settings Module

#### 25.1 General Settings
```typescript
interface GeneralSettings {
  platformName: string;
  platformTagline: string;
  platformLogo: File | string;
  supportEmail: string;
  supportPhone?: string;
  defaultTimezone: string;         // IANA
  defaultLanguage: string;         // ISO 639-1
  maintenanceMode: boolean;
  maintenanceBannerText?: string;
}
```

#### 25.2 Commission & Pricing
```typescript
interface CommissionSettings {
  defaultCommissionPercent: number;     // 0–50, step 0.5
  therapistMinRateCents: number;
  therapistMaxRateCents: number;
  cancellationWindowHours: number;      // 1–72
  lateCancellationFeePercent: number;   // 0–100
  noShowFeePercent: number;
}
```

#### 25.3 Payments & Payouts
```typescript
interface PaymentSettings {
  stripePublishableKey: string;         // displayed masked
  stripeWebhookSecret: string;          // masked, stored in Vercel env
  payoutSchedule: 'daily'|'weekly'|'biweekly'|'monthly';
  payoutMinimumCents: number;
  supportedCurrencies: string[];
  defaultCurrency: string;
}
```

#### 25.4 Email Templates
Editable via `apps/admin` CMS UI. Templates stored in `@repo/email` as React Email components with Handlebars-style variable slots. Admin-editable subject + HTML override stored in `PlatformSetting` as JSON.

Templates: `booking_confirmed` · `booking_cancelled` · `session_reminder_24h` · `session_reminder_1h` · `post_session_review_request` · `therapist_approved` · `therapist_rejected` · `payout_sent` · `dispute_opened` · `dispute_resolved` · `org_member_invite` · `password_reset`

Each has: subject · HTML body · plain text fallback · **Test send** button.

#### 25.5 B2B Configuration
```typescript
interface B2BSettings {
  b2bEnabled: boolean;
  defaultPlanTypes: OrgPlan[];
  contractTemplateUrl?: string;
  defaultSessionsPerSeat: number;
  orgTrialDays: number;
  usageReportFrequency: 'daily'|'weekly'|'monthly';
  privacyGuaranteeText: string;         // shown to employees
}
```

#### 25.6 Compliance
```typescript
interface ComplianceSettings {
  hipaaMode: boolean;                           // stricter logging, BAA flow
  dataRetentionDays: number;                    // default 2555 (7 years)
  sessionRecordingEnabled: boolean;
  sessionRecordingRequiresConsent: boolean;
  auditLogRetentionDays: number;                // min 2555 for HIPAA
  gdprMode: boolean;                            // data export / right-to-delete
  cookieConsentRequired: boolean;
}
```

---

### 26. Admin User Management

#### 26.1 Therapist Review Screen (`/therapists/[id]`)
- Profile overview (photo, bio, specializations)
- Credential panel: license number, state, expiry, Supabase Storage document viewer (signed URL 15 min, modal)
- Status history timeline: each status change + admin who acted + timestamp
- Actions: **Approve** | **Reject** (reason modal) | **Request More Info** (Knock message) | **Suspend**
- Sessions count, reviews summary, Stripe Connect status badge

#### 26.2 User Detail Screen (`/users/[id]`)
- Profile card: avatar, name, email, role, joined, last active, status
- Role-specific panel: therapist credentials OR client intake OR org membership
- Session history (last 10, paginated link)
- Payment history (last 10)
- Active disputes
- Audit trail for this user
- Actions: **Suspend** | **Restore** | **Delete (GDPR)** — soft-delete + anonymization | **Reset Password** (Clerk) | **Impersonate** (super_admin only — requires 2nd admin approval, logged to `AdminImpersonation`)

#### 26.3 Admin List (`/admins`) — super_admin only
Columns: Name | Email | Tier | 2FA Enabled | Last Login | Created | Actions
Actions: Edit Tier (Clerk publicMetadata update) · Suspend (Clerk org ban) · Remove
Create: "Invite Admin" → `AdminInviteSchema` → Clerk invitation

---

### 27. B2B Organizations Module

#### 27.1 Organization Detail Tabs
1. **Overview** — plan, contract dates, seats, active usage meter
2. **Members** — invite (bulk), deactivate, promote to org_admin
3. **Therapist Pool** — add/remove therapists from `OrganizationTherapistPool`
4. **Billing** — invoices list, plan config, Stripe customer portal link
5. **Usage Reports** — monthly anonymized utilization (k-anonymity ≥5 members enforced)
6. **Settings** — SSO (Clerk Organizations SAML/OIDC), custom branding, domain

#### 27.2 B2B Session Coverage Logic

```typescript
// packages/database/src/b2b.ts
export async function checkB2BCoverage(clientId: string, orgId: string) {
  const [member, org] = await Promise.all([
    prisma.organizationMember.findFirst({ where: { clientId, organizationId: orgId } }),
    prisma.organization.findUnique({ where: { id: orgId } }),
  ]);
  if (!member || !org) throw new Error('Member or org not found');
  return {
    isCovered: member.sessionsUsedThisMonth < (org.sessionsPerSeatPerMonth ?? 0),
    sessionsUsed: member.sessionsUsedThisMonth,
    sessionsLimit: org.sessionsPerSeatPerMonth ?? 0,
  };
  // isCovered = false → client pays out of pocket via Stripe
  // isCovered = true → booking.isCoveredByOrg = true → billed on org invoice
}
```

#### 27.3 Org Admin View (within `apps/app`)

```
/settings/organization          — plan overview, seat count, sessions used
/settings/organization/members  — invite/deactivate members
/settings/organization/reports  — anonymized usage (k-anon: suppress if <5 active)
/settings/organization/billing  — invoices, Stripe customer portal link
```

Org admins see only aggregate data — never individual session content, notes, or message threads.

#### 27.4 B2B Invoice Generation

Supabase Edge Function (cron: 1st of month, 06:00 UTC):

```typescript
// Idempotency key: `${orgId}:${periodStart}` passed to Stripe Invoice create
1. prisma.booking.findMany({ where: { isCoveredByOrg: true, organizationId, scheduledAt: { gte: periodStart, lt: periodEnd } } })
2. prisma.organizationInvoice.create({ data: { ... } })
3. stripe.invoices.create({ customer: org.stripeCustomerId, ... }) → @repo/payments
4. resend.send({ to: org.billingEmail, template: 'org_invoice' }) → @repo/email
5. prisma.organizationInvoice.update({ stripeInvoiceId: invoice.id })
```

#### 27.5 SSO Configuration (Clerk Organizations)

```typescript
// Clerk Organizations provides SAML/OIDC SSO out of the box
// Admin UI at apps/admin/app/organizations/[orgId]/settings
// On SSO setup: admin links Clerk Organization to Althena org record via clerkOrgId

interface SSOConfig {
  provider: 'saml' | 'oidc';
  // SAML — configured via Clerk dashboard
  entryPoint?: string;
  issuer?: string;
  cert?: string;          // SENSITIVE — stored in Clerk, not Prisma
  // OIDC — configured via Clerk dashboard
  clientId?: string;
  clientSecret?: string;  // SENSITIVE — stored in Clerk, not Prisma
  discoveryUrl?: string;
}
// On SSO login: Clerk auto-enrolls user → Clerk webhook fires → sync to OrganizationMember
```

---

## CROSS-CUTTING CONCERNS {#cross-cutting}

---

### Security

| Concern | Implementation |
|---|---|
| Authentication | Clerk (session tokens, TOTP 2FA for admins, hardware key for super_admin) |
| Authorization | Clerk `publicMetadata.role` + server-side checks in Route Handlers |
| PHI encryption | AES-256 at application layer before Prisma INSERT (notes, messages, PII fields) |
| Encryption keys | Vercel environment variables (not Prisma schema) — rotate via Vercel dashboard |
| Rate limiting | `@repo/security` (Arcjet) — bot detection + rate limiting on all public APIs |
| Webhook verification | `@repo/webhooks` (Svix) — Stripe + Clerk signature verification |
| Admin panel isolation | Separate `apps/admin` app on `admin.althena.com` with strict CSP |
| HIPAA audit trail | `AdminAuditLog` — immutable, 7-year retention, RLS `FOR SELECT` only |
| Test session privacy | `sessionToken` never logged; 24h auto-purge for anonymous sessions |
| SSO secrets | Stored in Clerk Organizations — never in Prisma / plaintext env |
| Admin impersonation | Requires 2nd admin approval + logged to `AdminImpersonation` |
| Credential documents | Supabase Storage private bucket, 15-min signed URLs, admin-only |
| Video E2E | Daily.co DTLS-SRTP |
| Headers | `X-Frame-Options: DENY`, `Strict-Transport-Security`, configured via `@repo/next-config` |

### Performance

| Concern | Implementation |
|---|---|
| Build caching | Turborepo Remote Cache — only rebuild changed packages |
| List views | Cursor-based Prisma pagination (no OFFSET) |
| Slot computation | Supabase Edge Function → Vercel KV cache (5 min per therapist) |
| Admin analytics | Postgres materialized views, refreshed hourly via Edge Function |
| Org usage reports | Precomputed nightly via Edge Function cron |
| Blog/test pages | ISR with `generateStaticParams` (top 100 by viewCount) |
| Images | Supabase CDN + `next/image` (AVIF/WebP) + blurhash placeholders |
| Blog view counter | Non-blocking `navigator.sendBeacon` → Prisma increment |
| DB connections | Supabase pgBouncer pooler (poolerUrl) + Prisma connection management |
| Bundle analysis | `@repo/next-config` with `withAnalyzer()` wrapper for CI bundle reports |
| Observability | `@repo/observability` (Sentry) — error tracking; BetterStack — log drain |

### Sprint Prioritization

| Sprint | Deliverable |
|---|---|
| 1 | Monorepo scaffold (Althena init), Turborepo, core packages skeleton |
| 2 | `@repo/database` Prisma schema + migrations, `@repo/auth` Clerk setup |
| 3 | `apps/app` + `apps/api` bootstrap, Clerk webhooks → profile sync |
| 4 | Therapist onboarding wizard (`apps/app`) + admin credential review (`apps/admin`) |
| 5 | Availability engine + slot computation (Edge Function + Vercel KV cache) |
| 6 | Booking flow + session state machine + Stripe Payment Intent |
| 7 | Video session integration (Daily.co) + `/video/[roomId]` isolated layout |
| 8 | Session notes (PHI encryption before Prisma INSERT) |
| 9 | Messaging + Supabase Realtime (`@repo/realtime`) |
| 10 | Stripe Connect Express (therapist payouts) |
| 11 | Client-side billing + payment methods (Stripe Elements) |
| 12 | `@repo/notifications` (Knock) — in-app + email reminders (Resend) |
| 13 | B2B organizations: creation, member invite, coverage logic |
| 14 | B2B billing: Stripe Billing, monthly invoicing Edge Function, org admin dashboard |
| 15 | Admin analytics dashboard (all sub-pages, materialized views) |
| 16 | Disputes + refund flow |
| 17 | Reviews + moderation queue |
| 18 | Platform settings module (all tabs) |
| 19 | Admin management (super_admin), audit log, impersonation |
| 20 | `apps/web` public site: home, therapist directory, static pages |
| 21 | Blog CMS: `@repo/cms` Content Collections + ISR + admin CRUD |
| 22 | Psychological test engine: admin creation + anonymous execution + result |
| 23 | Newsletter + lead capture + `@repo/analytics` consent management |
| 24 | B2B SSO (Clerk Organizations SAML/OIDC) |
| 25 | `@repo/observability` full setup (Sentry, BetterStack), bundle analysis |
| 26+ | Mobile PWA, push notifications, recording consent flow, i18n (`@repo/i18n`) |

### High-Risk Areas

| Risk | Area | Mitigation |
|---|---|---|
| PHI encryption key rotation | Notes, messages | App-layer rotation job + Prisma migration; no pgcrypto dependency |
| Stripe Connect complexity | Therapist payouts | Start manual (admin-initiated), automate Sprint 10 |
| B2B billing reconciliation | Org invoices | Stripe Billing + idempotency key `orgId:periodStart` |
| SAML/OIDC misconfiguration | B2B SSO | Use Clerk Organizations (managed IdP integration) |
| Slot race conditions | Booking creation | Prisma `$transaction` + `SELECT FOR UPDATE` equivalent via optimistic locking |
| HIPAA BAA gap | All PHI vendors | BAAs: Supabase Pro+, Daily.co, Resend, PostHog (EU) before US production |
| Admin impersonation abuse | Super admin | 2nd admin approval required; every session logged in `AdminImpersonation` |
| Credential forgery | Therapist onboarding | Nursys / FSMB license verification API integration (Sprint 4+) |
| GDPR data deletion | User data | Prisma soft-delete + anonymization pipeline; no hard DELETE |
| Anonymous org reports | B2B privacy | k-anonymity: suppress if `activeMembers < 5` |
| Test score manipulation | Test engine | Score computed server-side only in Route Handler; never on client |
| Newsletter email enumeration | Public API | Always return 200 with identical message body |
| Duplicate content (test results) | SEO | `noindex` all `/tests/*/result` pages; `noindex` in `robots.ts` |
| ISR stale health content | Blog | Use `revalidatePath` on publish (on-demand), not time-based only |
| GDPR double opt-in | Newsletter EU | Resend confirmation email mandatory; single opt-in blocked for EU IPs |
| Anonymous test health data | GDPR | 24h auto-purge Edge Function + `expiresAt` column enforced |
| Turborepo cache poisoning | CI/CD | Sign Turborepo Remote Cache tokens; use Vercel's managed cache |
| Clerk webhook replay attacks | Auth sync | Verify Svix signature + timestamp window (5 min) in `apps/api` |

---
