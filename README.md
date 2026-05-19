# Althena — Full Platform Technical Specification

**Platform:** Mental Health SaaS (B2C + B2B + Public Site)
**Domain:** althena.com
**Roles Covered:** Client · Corporate Client (B2B) · Corporate Consultant · Therapist · Admin · Super Admin · B2B Organization
**Stack:** Next.js 15 (App Router) · Turborepo · Supabase (PostgreSQL + Auth) · Prisma · TypeScript · Tailwind CSS
**Architecture:** Althena monorepo pattern (v6.x)
**Version:** 5.0 | **Date:** 2026-05-19

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
│   ├── auth/           # @repo/auth       — Supabase Auth config, middleware, helpers
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
│   ├── webhooks/       # @repo/webhooks   — Stripe webhook verification (Svix)
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
| `apps/api` | `api.althena.com` | Webhook receivers (Stripe), health endpoint |
| `apps/email` | dev-only | React Email template preview server |

### Key Pattern Differences vs Plain Next.js

| Pattern | Convention | Implementation |
|---|---|---|
| Auth | `@repo/auth` wraps Supabase Auth | Supabase Auth for all user roles; custom admin TOTP layer |
| Database | `@repo/database` exports `prisma` client | Supabase PostgreSQL + Prisma ORM |
| ORM | Prisma schema in `packages/database/prisma/schema.prisma` | Single schema covering all tables |
| Env vars | `packages/<pkg>/keys.ts` with `@t3-oss/env-nextjs` + Zod | Each package owns its env shape |
| Rate limiting | `@repo/security` wraps Arcjet | Arcjet for public APIs |
| Email | `@repo/email` + Resend | React Email templates, Handlebars for admin-editable templates |
| CMS | `@repo/cms` wraps Content Collections | Content Collections for blog/tests (MDX-based, type-safe) |
| Webhooks | `@repo/webhooks` — verification via Svix | `apps/api/app/webhooks/stripe/route.ts` |
| Payments | `@repo/payments` wraps Stripe | Stripe Connect Express (therapist payouts) + Stripe Billing (B2B invoicing) |
| Observability | `@repo/observability` — Sentry + BetterStack | Sentry for error tracking, BetterStack for log drain |
| Analytics | `@repo/analytics` — PostHog + GA4 | Consent-gated; EU data residency on PostHog |
| Storage | `@repo/storage` — Supabase Storage | Presigned URLs, Sharp processing via Edge Function |
| Notifications | `@repo/notifications` — Knock | In-app + email notification delivery |

### Auth Package (`@repo/auth`) — Supabase Auth

Supabase Auth замінює Clerk. Ролі зберігаються у `profiles.role` в Prisma та у Supabase Auth `app_metadata.role` (встановлюється лише server-side, не доступне клієнту для модифікації).

```typescript
// packages/auth/keys.ts
import { createEnv } from '@t3-oss/env-nextjs';
import { z } from 'zod';

export const keys = createEnv({
  server: {
    SUPABASE_SERVICE_ROLE_KEY: z.string().min(1),
  },
  client: {
    NEXT_PUBLIC_SUPABASE_URL: z.string().url(),
    NEXT_PUBLIC_SUPABASE_ANON_KEY: z.string().min(1),
  },
  runtimeEnv: {
    SUPABASE_SERVICE_ROLE_KEY: process.env.SUPABASE_SERVICE_ROLE_KEY,
    NEXT_PUBLIC_SUPABASE_URL: process.env.NEXT_PUBLIC_SUPABASE_URL,
    NEXT_PUBLIC_SUPABASE_ANON_KEY: process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY,
  },
});

// packages/auth/src/middleware.ts
import { createServerClient } from '@supabase/ssr';
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export async function authMiddleware(request: NextRequest) {
  const response = NextResponse.next();
  const supabase = createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    { cookies: { /* cookie helpers */ } }
  );

  const { data: { session } } = await supabase.auth.getSession();
  const role = session?.user?.app_metadata?.role as string | undefined;

  // Захист адмін-маршрутів
  if (request.nextUrl.pathname.startsWith('/admin')) {
    if (!role || !['admin', 'super_admin'].includes(role)) {
      return NextResponse.redirect(new URL('/unauthorized', request.url));
    }
  }

  // Захист авторизованих маршрутів
  if (!session && !isPublicRoute(request.nextUrl.pathname)) {
    return NextResponse.redirect(new URL('/sign-in', request.url));
  }

  return response;
}
```

### Environment Variable Pattern

```typescript
// packages/database/keys.ts
export const keys = createEnv({
  server: {
    DATABASE_URL: z.string().url().startsWith('postgresql://'),
    DIRECT_URL: z.string().url().optional(),
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
```

### Turborepo Pipeline

```json
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
| Client | `client` | Фізична особа, яка шукає терапію; бронює сесії; може реєструватись як B2B клієнт за кодом контракту | Supabase Auth (Email / Google OAuth) |
| Therapist | `therapist` | Ліцензований фахівець; управляє розкладом; стандартна участь | Supabase Auth (Email + перевірка документів) |
| Corporate Consultant | `corporate_consultant` | Терапевт, який додатково погодив участь у корпоративній програмі; той самий кабінет + маркер | Supabase Auth (той самий акаунт терапевта) |
| Admin | `admin` | Оператор платформи; управляє користувачами, контентом, білінгом | Supabase Auth (внутрішній email, TOTP 2FA) |
| Super Admin | `super_admin` | Повний доступ до платформи; може управляти адмінами | Supabase Auth (внутрішній SSO + апаратний ключ) |

> **Важливо щодо B2B клієнта:** Корпоративний клієнт — це звичайна фізична особа (роль `client`). Єдина відмінність — прив'язка до контракту організації та спосіб оплати. Кабінет, інтерфейс, права — ідентичні звичайному клієнту.

> **Важливо щодо Corporate Consultant:** Це звичайний терапевт, який погодив участь у корпоративній програмі. Окремої ролі немає — лише маркер `isCorporateConsultant: true` у `TherapistProfile`. Кабінет не змінюється; терапевт отримує повідомлення в "дзвіночок" після погодження участі адміністратором.

> **Щодо HR:** HR-менеджери компаній-партнерів не реєструються на платформі та не мають кабінету. Звіти про використання надсилає адміністратор Althena на email HR-а вручну (або автоматизовано в майбутньому).

Роль зберігається у Supabase Auth `app_metadata.role` (server-side only) та дублюється у `profiles.role` в Prisma.

#### 1.2 Admin Permission Tiers

| Tier | Enum | Може робити |
|---|---|---|
| Support Agent | `admin:support` | Переглядати користувачів, управляти суперечками, читати платежі |
| Content Moderator | `admin:moderator` | Перевіряти документи терапевтів, управляти відгуками, CMS |
| Finance Admin | `admin:finance` | Повний доступ до платежів/виплат, налаштування білінгу |
| Operations Admin | `admin:operations` | Все вище + налаштування платформи |
| Super Admin | `super_admin` | Все + управління адмінами + B2B enterprise конфігурація |

Tier зберігається у Supabase Auth `app_metadata.adminTier` та `profiles.adminTier`.

#### 1.3 Therapist Sub-states

| State | Enum | Опис |
|---|---|---|
| Pending Verification | `pending` | Подав документи, очікує перевірки адміністратором |
| Active | `active` | Повністю онбордований, приймає клієнтів |
| On Leave | `on_leave` | Тимчасово недоступний; нових бронювань немає |
| Suspended | `suspended` | Дія адміністратора; вхід заблокований, сесії скасовані |
| Rejected | `rejected` | Документи не прийнято; може перезавантажити |

#### 1.4 Corporate Client States (прив'язка до контракту)

| State | Опис |
|---|---|
| `pending_contract` | Ввів код контракту, система перевіряє |
| `active` | Підтверджений корпоративний клієнт, може бронювати сесії |
| `deactivated` | Адміністратор вимкнув доступ (залишок кредитів не витрачається) |

---

### 2. Main User Flows & State Machines

#### 2.1 Therapist Onboarding
```
REGISTER (Supabase Auth) → EMAIL_VERIFY
  → PROFILE_SETUP → CREDENTIAL_UPLOAD
  → ADMIN_REVIEW
    → [APPROVED] → AVAILABILITY_SETUP → STRIPE_CONNECT → ACTIVE
    → [REJECTED]  → NOTIFICATION_SENT (email + "дзвіночок") → RESUBMIT
    → [MORE_INFO_NEEDED] → THERAPIST_NOTIFIED → REPLY → RE_REVIEW
```

#### 2.2 Client Registration & First Booking
```
REGISTER (Supabase Auth) → EMAIL_VERIFY
  → INTAKE_FORM → THERAPIST_MATCH
  → VIEW_PROFILE → SELECT_SLOT → PAYMENT_CAPTURE (Stripe)
  → BOOKING_CONFIRMED → REMINDER (Knock) → SESSION → POST_SESSION_REVIEW
```

#### 2.3 B2B Corporate Client — Повний флоу

**Частина 1: Укладання угоди (поза платформою + дії адміністратора)**
```
1. Компанія (напр. "Альфа") вирішує співпрацювати → підписує контракт з Альтеною
2. Альфа переводить кошти на Альтену (напр. 1000 EUR)
3. ADMIN у адмін-панелі:
   a. Створює організацію (Organization) для Альфи
   b. Обирає тип контракту (впливає на логіку оплати при бронюванні):
      - Тип 1: 100% передоплата (кредити на рахунку орг., оплата списується з кредитів)
      - Тип 2: Часткова передоплата (частина з кредитів, частина доплачує клієнт)
      - Тип 3: Постоплата (клієнт платить сам, компанія отримує рахунок наприкінці)
   c. Нараховує кредити: 1 EUR = 1 кредит → 1000 EUR = 1000 credits
   d. Генерує унікальний код контракту (contractCode)
4. ADMIN надсилає contractCode HR-у компанії Альфа (email поза платформою)
5. HR анонсує співробітникам можливість скористатись платформою
```

**Частина 2: Реєстрація корпоративного клієнта**
```
EMPLOYEE → althena.com (головна)
  → Реєстрація як звичайний клієнт (Supabase Auth)
  → В особистому кабінеті: Settings → "Я корпоративний клієнт"
    → З'являється поле: "Код контракту"
    → Вводить contractCode
    → Система перевіряє:
      - Чи існує контракт з таким кодом
      - Чи контракт активний
      - Чи достатньо кредитів (тип 1 / тип 2)
    → [VALID] → ClientProfile.organizationId прив'язується
               → ClientProfile.contractType встановлюється
               → Підтвердження + інформація про доступні сесії
    → [INVALID / EXPIRED / NO_CREDITS] → Повідомлення про помилку
```

**Частина 3: Бронювання корпоративним клієнтом**
```
CORPORATE CLIENT → Обирає терапевта → Обирає вид консультації (відео / чат)
  → Обирає дату та час → Підтвердження бронювання
  
  Логіка оплати залежить від типу контракту:
  
  [Тип 1 — 100% передоплата]:
    → Перевірка балансу кредитів організації
    → [ENOUGH CREDITS] → Кредити резервуються в момент бронювання
                        → Після сесії: кредити списуються з гаманця організації
    → [NO CREDITS] → Повідомлення: "Ліміт вичерпано, зверніться до вашого HR"
  
  [Тип 2 — Часткова передоплата]:
    → Частина суми списується з кредитів організації
    → Залишок клієнт оплачує через Stripe (картка)
  
  [Тип 3 — Постоплата]:
    → Клієнт оплачує повну вартість через Stripe
    → Організації виставляється рахунок наприкінці місяця
  
  → BOOKING_CONFIRMED → REMINDER → SESSION → POST_SESSION_REVIEW
```

**Частина 4: Звітність**
```
ADMIN → Щомісяця формує звіт по організації (анонімізована статистика)
      → Надсилає на email HR-а Альфи
      
Майбутня автоматизація: Supabase Edge Function cron (1 раз на місяць)
```

#### 2.4 Corporate Consultant — Онбординг у програму

```
THERAPIST (active) → Бажає брати участь у корпоративній програмі
  → Надсилає запит через Settings ("Беру участь у корпоративній програмі")
  → ADMIN розглядає запит
    → [APPROVED] → TherapistProfile.isCorporateConsultant = true
                 → Терапевт отримує повідомлення в "дзвіночок":
                   "Вашу участь у корпоративній програмі підтверджено"
                 → Кабінет терапевта не змінюється
                 → Терапевт тепер видний корпоративним клієнтам (якщо орг. не обмежила пул)
    → [REJECTED] → Терапевт отримує повідомлення з причиною
```

> **Для терапевта участь у корпоративній програмі означає:** -10% до доходу від корпоративних сесій, але стабільний потік клієнтів. Різниця між корпоративним і звичайним клієнтом для терапевта не відображається в інтерфейсі — він обслуговує всіх однаково.

#### 2.5 Session Lifecycle State Machine
```
SCHEDULED ──► CONFIRMED ──► IN_PROGRESS ──► COMPLETED
     │              │                            │
     ▼              ▼                            ▼
 CANCELLED      CANCELLED                   NO_SHOW
     │
     ▼
REFUND_INITIATED ──► REFUNDED
```

#### 2.6 Admin Therapist Review Flow
```
THERAPIST подає документи
  → Knock повідомлення до черги адміна
  → ADMIN відкриває екран перевірки (admin.althena.com)
    → Переглядає документ (Supabase Storage signed URL, 15 хв)
    → [APPROVE] → therapist.status = active, email via Resend
    → [REJECT + reason] → терапевт отримує повідомлення, може перезавантажити
    → [REQUEST_INFO] → Knock повідомлення терапевту
```

#### 2.7 Dispute / Refund Flow
```
CLIENT піднімає суперечку по бронюванню
  → Knock повідомлення до черги адміна
  → ADMIN розглядає запис сесії + повідомлення
    → [APPROVE_REFUND] → Stripe refund via @repo/payments, виплата скоригована
    → [REJECT_DISPUTE] → Клієнт отримує повідомлення з причиною (Resend)
    → [PARTIAL_REFUND] → Вводиться кастомна сума
```

#### 2.8 Guest Conversion Funnels

**Funnel A — Guest → Client Registration**
```
althena.com (Home)
  → [CTA: "Знайти терапевта" або "Пройти тест"]
    ├── /therapists/[slug] → /sign-up?intent=book&therapistId=[id]
    │     → app.althena.com/bookings (після реєстрації)
    ├── /tests/[slug]/start → /tests/[slug]/result
    │     → /sign-up?intent=test_result&testId=[id]&band=[band]
    └── /about → /sign-up
```

**Funnel B — B2B Corporate Lead**
```
althena.com/for-business → форма заявки → POST /api/public/leads
  → Resend повідомлення команді продажів, запис lead_submissions створено
```

**Funnel C — Therapist Partner Acquisition**
```
althena.com/cooperation → /sign-up?role=therapist (або inline lead форма)
```

**Funnel D — Test Funnel (SEO + конверсія)**
```
althena.com/tests/[slug] → /tests/[slug]/start → /tests/[slug]/result
  → /sign-up?intent=save_test&sessionToken=[token]
```

#### 2.9 Anonymous / Authenticated Transitions

| Transition Point | Auth Required | Post-auth Redirect |
|---|---|---|
| Забронювати сесію | Required (Supabase Auth) | `app.althena.com/bookings` |
| Зберегти терапевта | Required | Повернення на ту ж сторінку |
| Розпочати тест | Not required | — |
| Переглянути результат тесту | Not required | — |
| Зберегти результат тесту | Required | Повернення до результату |
| Підписка на newsletter | Not required | — |
| Бізнес-заявка | Not required | — |
| Заявка на співпрацю (cooperation) | Not required (lead) / Required (full) | — |

#### 2.10 Test Execution State Machine (Guest Domain)
```
[INTRO]
  → POST /api/public/tests/[testId]/session
    → session_token в httpOnly cookie (24h)
      → [QUESTION_N] × questions
        → PATCH .../answer (debounced 300ms, sendBeacon on unload)
          → POST .../complete (score computed server-side)
            → [RESULT]

[QUESTION_N] → браузер закрито
  → сесія зберігається (in_progress)
  → при поверненні: перевірка httpOnly cookie
    → активна: [RESUME_PROMPT]
    → прострочена (>24h): [INTRO]

[RESULT]
  → authenticated: POST .../claim
  → guest: /sign-up?intent=save_test&sessionToken=[token]
  → "Поділитися": /tests/[slug]/result?band=[result_band] (без PII)
```

#### 2.11 Newsletter Subscription State Machine
```
[IDLE] → POST /api/public/newsletter
  → honeypot check → silent discard if triggered
  → Arcjet rate limit check (3/IP/hour)
  → INSERT newsletter_subscribers (always return 200, no enumeration)
    → Resend: double opt-in confirmation email (обов'язково для EU)
      → [PENDING] → користувач натискає підтвердження → [CONFIRMED]
      → token прострочено (24h) → [EXPIRED] → можна надіслати повторно

[CONFIRMED] → unsubscribe link → [UNSUBSCRIBED]
```

#### 2.12 Content Publishing (CMS)
```
DRAFT → [Schedule] → SCHEDULED → [cron Edge Function] → PUBLISHED
DRAFT → [Publish Now] → PUBLISHED
PUBLISHED → [Unpublish] → DRAFT
PUBLISHED → [Archive] → ARCHIVED (410 response)

On PUBLISHED: next.js revalidateTag + revalidatePath
```

#### 2.13 Мультимовність

Мова обирається користувачем на **головній сторінці** (перший вхід). Вибрана мова зберігається і дотримується на всіх подальших сторінках. **Змінити мову можна лише на головній сторінці** — навмисне обмеження для UX-консистентності.

```typescript
// packages/internationalization/src/index.ts
// Підтримувані мови: ua, en (розширення — на майбутнє)
// Зберігання: cookie 'althena_lang' (server-readable) + localStorage (fallback)
// Middleware читає cookie та додає відповідний prefix або header Accept-Language
```

---

### 3. Business Model

#### 3.1 B2C Model

| Компонент | Деталі |
|---|---|
| Вартість сесії | Встановлює терапевт (різні ціни для різних фахівців) |
| Комісія платформи | Налаштовується адміністратором % від завершеної сесії (за замовчуванням 20%) |
| Плата за скасування | Клієнт платить 50% при скасуванні <24h (налаштовується) |
| Платіжний процесор | Stripe (картка, Apple Pay, Google Pay) |
| Виплата терапевту | Щотижнево через Stripe Connect Express |

#### 3.2 B2B Model

| Компонент | Деталі |
|---|---|
| Типи контрактів | Тип 1: 100% передоплата · Тип 2: Часткова передоплата · Тип 3: Постоплата |
| Кредитна система | 1 EUR = 1 кредит; кредити зберігаються на рахунку організації |
| Оплата | Залежить від типу контракту (з кредитів, частково, або постоплата через Stripe) |
| Доступ до терапевтів | Всі `corporate_consultant` (або обмежений пул, якщо орг. налаштувала) |
| Конфіденційність | Ніколи не розкривати контент індивідуальних сесій; тільки агрегована статистика |
| Звітність | Адміністратор щомісяця надсилає анонімізований звіт на email HR-а |
| Комісія платформи | На 10% менший дохід терапевта від корпоративних сесій |

#### 3.3 Клієнтський профіль — гнучкість та конфіденційність

- **Фото/аватар:** Необов'язкове поле. Клієнт може завантажити реальне фото або будь-яке зображення.
- **Ім'я:** Клієнт може використовувати реальне ім'я або псевдонім. Може змінити в будь-який момент (сьогодні "Наталі", завтра "Ігор" — для сайту психологічної підтримки це нормально).
- **Відгуки:** Клієнт може залишити відгук як під своїм ім'ям реєстрації, так і анонімно.
- **Нагадування:** Клієнт може налаштувати частоту нагадувань та канали доставки (email, push тощо).

---

### 4. Domain Entities & DB Schema

**ORM:** Prisma · **Database:** Supabase (PostgreSQL) · **Schema location:** `packages/database/prisma/schema.prisma`

Supabase connection pooling via pgBouncer — встановити `DATABASE_URL` на pooler URL і `DIRECT_URL` на пряме з'єднання (потрібно для Prisma migrations).

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

// Три типи контрактів для корпоративних організацій
enum ContractType {
  full_prepay      // Тип 1: 100% передоплата — всі сесії з кредитів орг.
  partial_prepay   // Тип 2: Часткова передоплата — частина з кредитів, частина клієнт
  postpay          // Тип 3: Постоплата — клієнт платить, орг. отримує рахунок
}

enum OrgMemberStatus {
  pending_contract  // Ввів код контракту, перевіряється
  active            // Підтверджений корпоративний клієнт
  deactivated       // Адміністратор вимкнув доступ
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
  id                String     @id // Supabase Auth user ID (uuid)
  role              UserRole
  adminTier         AdminTier?
  displayName       String     // Може бути псевдонім — клієнт обирає сам
  avatarUrl         String?    // Необов'язкове; може бути будь-яке зображення
  phone             String?
  timezone          String     @default("UTC")
  dateOfBirth       DateTime?  // SENSITIVE — зашифровано на app layer
  preferredLanguage String     @default("uk") // 'uk' | 'en'
  isDeleted         Boolean    @default(false)
  createdAt         DateTime   @default(now())
  updatedAt         DateTime   @updatedAt
  lastSeenAt        DateTime?

  therapistProfile      TherapistProfile?
  clientProfile         ClientProfile?
  sentMessages          Message[]           @relation("SentMessages")
  notifications         Notification[]
  adminLogs             AdminAuditLog[]     @relation("AdminLogs")
  raisedDisputes        Dispute[]           @relation("RaisedDisputes")
  resolvedDisputes      Dispute[]           @relation("ResolvedDisputes")
  cancelledBookings     Booking[]           @relation("CancelledBookings")
  impersonations        AdminImpersonation[] @relation("ImpersonationTarget")
  impersonatedBy        AdminImpersonation[] @relation("ImpersonationActor")
  uploadedMedia         MediaAsset[]
  authoredBlogPosts     BlogAuthor?

  @@map("profiles")
}

model TherapistProfile {
  id                      String          @id
  profile                 Profile         @relation(fields: [id], references: [id])
  licenseNumber           String          // SENSITIVE — зашифровано
  licenseState            String?
  licenseExpiry           DateTime?
  licenseVerifiedAt       DateTime?
  specializations         String[]
  approaches              String[]
  languages               String[]
  bio                     String?
  yearsOfExperience       Int?
  sessionRateCents        Int
  stripeAccountId         String?         // SENSITIVE
  onboardingStatus        TherapistStatus @default(pending)
  verifiedById            String?
  rejectionReason         String?
  totalSessionsCompleted  Int             @default(0)
  avgRating               Decimal?        @db.Decimal(3, 2)
  
  // Корпоративна програма
  isCorporateConsultant        Boolean  @default(false)
  corporateConsultantApprovedAt DateTime?
  corporateConsultantApprovedById String?
  // Маркер потрібен для аналітики (кількість учасників у програмі, покриття)

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
  
  // B2B прив'язка (null = звичайний клієнт)
  organizationId         String?
  organization           Organization? @relation(fields: [organizationId], references: [id])
  orgMemberStatus        OrgMemberStatus?
  // Тип контракту успадковується від організації, але кешується тут для зручності
  contractType           ContractType?
  
  emergencyContactName   String?  // SENSITIVE — зашифровано
  emergencyContactPhone  String?  // SENSITIVE — зашифровано
  primaryConcern         String[]
  assignedTherapistId    String?
  
  // Налаштування нагадувань
  reminderFrequency      String?  // 'daily' | 'session_day' | 'custom'
  reminderChannels       String[] // 'email' | 'push'

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
  
  // B2B оплата
  isCoveredByOrg     Boolean       @default(false)  // Частина або вся сума списана з кредитів орг.
  orgCreditUsed      Int           @default(0)      // Кількість списаних кредитів (cents)
  
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

  @@map("sessions")
}

model SessionNote {
  id                  String           @id @default(cuid())
  bookingId           String
  booking             Booking          @relation(fields: [bookingId], references: [id])
  therapistId         String
  therapist           TherapistProfile @relation(fields: [therapistId], references: [id])
  content             String           // SENSITIVE — AES-256 зашифровано перед INSERT
  noteType            NoteType
  isSharedWithClient  Boolean          @default(false)
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
  content        String       // SENSITIVE — AES-256 зашифровано
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
  stripePaymentIntentId  String        @unique // SENSITIVE — ніколи не логується
  amountCents            Int
  currency               String        @default("eur")
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
  action     String
  targetType String?
  targetId   String?
  metadata   Json?
  ipAddress  String?
  createdAt  DateTime @default(now())

  @@map("admin_audit_logs")
}

model AdminImpersonation {
  id           String   @id @default(cuid())
  actorId      String
  actor        Profile  @relation("ImpersonationActor", fields: [actorId], references: [id])
  targetId     String
  target       Profile  @relation("ImpersonationTarget", fields: [targetId], references: [id])
  approvedById String?
  startedAt    DateTime @default(now())
  endedAt      DateTime?

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
  type      String   // 'booking_confirmed' | 'session_reminder' | 'corporate_approved' | 'review_request' | ...
  title     String?
  body      String?
  data      Json?
  isRead    Boolean  @default(false)
  createdAt DateTime @default(now())

  @@map("notifications")
}

model Review {
  id           String           @id @default(cuid())
  bookingId    String           @unique
  booking      Booking          @relation(fields: [bookingId], references: [id])
  clientId     String
  client       ClientProfile    @relation(fields: [clientId], references: [id])
  therapistId  String
  therapist    TherapistProfile @relation(fields: [therapistId], references: [id])
  rating       Int              // 1–5
  comment      String?
  isPublic     Boolean          @default(true)
  isAnonymous  Boolean          @default(false)  // Клієнт може залишити анонімний відгук
  isFlagged    Boolean          @default(false)
  flaggedReason String?
  createdAt    DateTime         @default(now())

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
  id                    String       @id @default(cuid())
  name                  String
  logoUrl               String?
  contractType          ContractType // Тип 1, 2 або 3 — впливає на логіку оплати
  contractCode          String       @unique // Унікальний код, який HR передає співробітникам
  creditBalance         Int          @default(0) // Залишок кредитів у cents
  stripeCustomerId      String?      // SENSITIVE (для постоплати / Тип 3)
  billingEmail          String       // Email HR/бухгалтерії для звітів та рахунків
  billingContactName    String?
  isActive              Boolean      @default(true)
  contractStart         DateTime?    @db.Date
  contractEnd           DateTime?    @db.Date
  createdAt             DateTime     @default(now())
  
  // Налаштування пулу терапевтів (null = всі corporate consultants)
  restrictToPool        Boolean      @default(false)

  members        OrganizationMember[]
  therapistPool  OrganizationTherapistPool[]
  invoices       OrganizationInvoice[]
  usageReports   OrganizationUsageReport[]
  bookings       Booking[]
  clientProfiles ClientProfile[]
  creditTransactions OrgCreditTransaction[]

  @@map("organizations")
}

// Лог транзакцій кредитів (поповнення та списання)
model OrgCreditTransaction {
  id             String       @id @default(cuid())
  organizationId String
  organization   Organization @relation(fields: [organizationId], references: [id])
  type           String       // 'top_up' | 'debit' | 'refund'
  amountCents    Int          // Позитивне = поповнення, Від'ємне = списання
  bookingId      String?      // Прив'язка до бронювання (для debit/refund)
  note           String?      // Коментар адміністратора
  createdById    String?      // Admin ID
  createdAt      DateTime     @default(now())

  @@map("org_credit_transactions")
}

model OrganizationMember {
  id             String          @id @default(cuid())
  organizationId String
  organization   Organization    @relation(fields: [organizationId], references: [id])
  clientId       String?         @unique
  client         ClientProfile?  @relation(fields: [clientId], references: [id])
  email          String
  status         OrgMemberStatus @default(pending_contract)
  joinedAt       DateTime?

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
  stripeInvoiceId String?       @unique // Для Тип 3 (постоплата)
  periodStart     DateTime      @db.Date
  periodEnd       DateTime      @db.Date
  amountCents     Int
  sessionsBilled  Int?
  status          InvoiceStatus @default(draft)
  dueDate         DateTime?     @db.Date
  paidAt          DateTime?
  sentToEmail     String?       // Email HR, на який надіслано звіт
  createdAt       DateTime      @default(now())

  @@map("organization_invoices")
}

model OrganizationUsageReport {
  id                   String       @id @default(cuid())
  organizationId       String
  organization         Organization @relation(fields: [organizationId], references: [id])
  reportMonth          DateTime     @db.Date
  totalMembers         Int
  activeMembers        Int          // k-анонімність: приховати якщо < 5
  totalSessions        Int
  avgSatisfactionScore Decimal?     @db.Decimal(3, 2)
  generatedAt          DateTime     @default(now())
  sentToEmail          String?      // Email HR, на який надіслано звіт

  @@map("organization_usage_reports")
}

// ─────────────────────────────────────────────
// PUBLIC / GUEST DOMAIN
// ─────────────────────────────────────────────

model BlogAuthor {
  id          String    @id @default(cuid())
  profileId   String?   @unique
  profile     Profile?  @relation(fields: [profileId], references: [id])
  fullName    String
  slug        String    @unique
  bio         String?
  avatarUrl   String?
  title       String?
  credentials String?
  createdAt   DateTime  @default(now())

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
  id                 String       @id @default(cuid())
  slug               String       @unique
  title              String
  excerpt            String?      @db.VarChar(300)
  content            String       // MDX or HTML
  coverImageUrl      String?
  authorId           String?
  author             BlogAuthor?  @relation(fields: [authorId], references: [id])
  categoryId         String?
  category           BlogCategory? @relation(fields: [categoryId], references: [id])
  tags               String[]
  status             BlogStatus   @default(draft)
  publishedAt        DateTime?
  scheduledFor       DateTime?
  readingTimeMinutes Int?
  seoTitle           String?
  seoDescription     String?
  ogImageUrl         String?
  canonicalUrl       String?
  schemaType         String       @default("Article")
  viewCount          Int          @default(0)
  createdAt          DateTime     @default(now())
  updatedAt          DateTime     @updatedAt

  @@map("blog_posts")
}

model Test {
  id               String     @id @default(cuid())
  slug             String     @unique
  title            String
  description      String?
  instructions     String?
  coverImageUrl    String?
  category         String?
  estimatedMinutes Int?
  questionCount    Int?
  status           TestStatus @default(draft)
  isFeatured       Boolean    @default(false)
  scoringMethod    String     // 'sum'|'weighted_sum'|'subscale'
  resultConfig     Json
  seoTitle         String?
  seoDescription   String?
  ogImageUrl       String?
  schemaType       String     @default("MedicalWebPage")
  completionCount  Int        @default(0)
  publishedAt      DateTime?
  createdAt        DateTime   @default(now())
  updatedAt        DateTime   @updatedAt

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
  options      Json
  subscale     String?
  weight       Decimal  @default(1.0)
  createdAt    DateTime @default(now())

  @@map("test_questions")
}

model TestSession {
  id             String            @id @default(cuid())
  testId         String
  test           Test              @relation(fields: [testId], references: [id])
  sessionToken   String            @unique // httpOnly cookie — НІКОЛИ не логується
  userId         String?           // null якщо анонімний
  answers        Json              @default("{}")
  currentStep    Int               @default(1)
  status         TestSessionStatus @default(in_progress)
  score          Decimal?
  resultBand     String?
  resultSubscores Json?
  ipHash         String?           // SHA-256 від IP
  userAgent      String?
  completedAt    DateTime?
  expiresAt      DateTime?         // 24h для анонімних сесій
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
  source         String?
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
  intent      String?
  utmSource   String?
  utmMedium   String?
  utmCampaign String?
  status      String   @default("new") // 'new'|'contacted'|'qualified'|'closed'
  ipHash      String?
  createdAt   DateTime @default(now())

  @@map("lead_submissions")
}

model CooperationRequest {
  id                   String   @id @default(cuid())
  fullName             String
  email                String
  phone                String?
  specialization       String?
  licenseInfo          String?
  message              String?
  status               String   @default("new")
  convertedTherapistId String?
  createdAt            DateTime @default(now())

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

Prisma — основний рівень доступу до даних. Supabase RLS — захист в глибину. Критичні policies:

```sql
-- PHI: session_notes — тільки терапевт, який написав, або адмін
ALTER TABLE session_notes ENABLE ROW LEVEL SECURITY;
CREATE POLICY "therapist_owns_notes"
  ON session_notes FOR ALL
  USING (therapist_id = auth.uid()
    OR EXISTS (SELECT 1 FROM profiles WHERE id = auth.uid() AND role IN ('admin','super_admin')));

-- Public: blog_posts — тільки опубліковані
ALTER TABLE blog_posts ENABLE ROW LEVEL SECURITY;
CREATE POLICY "public_published_posts"
  ON blog_posts FOR SELECT
  USING (status = 'published' AND published_at <= now());

-- B2B: organizations — тільки admin або члени
CREATE POLICY "org_access"
  ON organizations FOR SELECT
  USING (EXISTS (SELECT 1 FROM profiles WHERE id = auth.uid() AND role IN ('admin','super_admin'))
    OR EXISTS (SELECT 1 FROM client_profiles WHERE id = auth.uid() AND organization_id = organizations.id));

-- Org credit transactions — тільки admin
CREATE POLICY "admin_only_credit_tx"
  ON org_credit_transactions FOR ALL
  USING (EXISTS (SELECT 1 FROM profiles WHERE id = auth.uid() AND role IN ('admin','super_admin')));

-- newsletter_subscribers: тільки INSERT для анонімних
CREATE POLICY "newsletter_insert_only"
  ON newsletter_subscribers FOR INSERT WITH CHECK (true);
```

---

### 5. Domain Boundaries

| Domain | Owns | Package | Depends On |
|---|---|---|---|
| **Identity & Auth** | Supabase Auth profile sync, `profiles` | `@repo/auth` | — |
| **Therapist Onboarding** | `therapist_profiles`, документи, черга перевірки | `@repo/database` | Identity |
| **Corporate Program** | `isCorporateConsultant`, approval flow | `@repo/database` | Identity, Therapist |
| **Scheduling** | `availability_schedules`, `exceptions`, обчислення слотів | `@repo/database` | Identity, Therapist |
| **Sessions** | `sessions`, `bookings`, Daily.co rooms | `@repo/database` | Scheduling |
| **Clinical Notes** | `session_notes` (encrypted) | `@repo/database` | Sessions, Identity |
| **Messaging** | `conversations`, `messages` (encrypted) | `@repo/realtime` | Identity |
| **Payments (B2C)** | `payments`, `payouts`, Stripe Connect | `@repo/payments` | Scheduling |
| **B2B Credits** | `org_credit_transactions`, `creditBalance`, логіка списання | `@repo/database` | Identity, Payments |
| **B2B Billing** | `organization_invoices`, Stripe Billing (Тип 3) | `@repo/payments` | B2B Credits |
| **Disputes** | `disputes`, refund logic | `@repo/payments` | Payments, Sessions |
| **Notifications** | In-app (Knock) + email (Resend) | `@repo/notifications`, `@repo/email` | All domains |
| **Reviews** | `reviews`, модерація, анонімні відгуки | `@repo/database` | Sessions |
| **Admin** | `admin_audit_logs`, `platform_settings`, інструменти | `apps/admin` | All domains |
| **Public / CMS** | Blog, tests, leads, newsletter | `@repo/cms`, `@repo/security` | Identity (optional) |
| **Analytics** | PostHog, materialized views | `@repo/analytics` | All domains |

---

### 6. Sensitive Data Inventory

| Field / Context | Classification | Handling |
|---|---|---|
| `session_notes.content` | PHI | AES-256 зашифровано перед Prisma INSERT, розшифровується на SELECT |
| `messages.content` | PHI | AES-256 зашифровано перед Prisma INSERT |
| `client_profiles.dateOfBirth` | PII | Зашифрований стовпець |
| `client_profiles.emergencyContact*` | PII | Зашифрований стовпець |
| `therapist_profiles.licenseNumber` | PII | Зашифрований стовпець |
| `therapist_profiles.stripeAccountId` | Financial PII | Prisma field-level access control + RLS |
| `organizations.stripeCustomerId` | Financial PII | Обмежений доступ |
| `payments.stripePaymentIntentId` | Financial | Ніколи не логується; тільки `@repo/payments` |
| `organizations.contractCode` | Operational | Не публікується в API; тільки admin може переглядати |
| `test_sessions.sessionToken` | Pseudo-PII | httpOnly cookie; ніколи не логується; auto-purge 24h для анонімних |
| `newsletter_subscribers.email` | PII | Service-role only SELECT; немає enumeration в API |
| Video stream | PHI | Daily.co DTLS-SRTP E2E encrypted |
| Credential documents | PII | Supabase Storage private bucket; 15-хв signed URL |
| `admin_audit_logs` | Operational | Immutable; read-only для super_admin; 7-річне зберігання |
| `ip_hash` (всі таблиці) | Pseudonymous | SHA-256 без зворотного словника |

---

### 7. RBAC Matrix

| Resource / Action | CLIENT | THERAPIST | CORPORATE_CONSULTANT | ADMIN (support) | ADMIN (finance) | ADMIN (operations) | SUPER_ADMIN |
|---|---|---|---|---|---|---|---|
| Переглядати власний профіль | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Редагувати власний профіль (ім'я/фото) | ✅ | ✅ | ✅ | ❌ | ❌ | ✅ | ✅ |
| Прив'язати код контракту | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Переглядати профілі інших | ❌ | власні клієнти | власні клієнти | ✅ read | ✅ read | ✅ | ✅ |
| Suspend / ban user | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ |
| Бронювати сесію | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Підтвердити / скасувати бронювання | ❌ | власні | власні | ✅ | ❌ | ✅ | ✅ |
| Створювати нотатки | ❌ | власні сесії | власні сесії | ❌ | ❌ | ❌ | ❌ |
| Читати нотатки | власні (якщо shared) | власні | власні | ✅ audit | ❌ | ✅ | ✅ |
| Надсилати повідомлення | ✅ | власним клієнтам | власним клієнтам | ❌ | ❌ | ❌ | ❌ |
| Налаштовувати розклад | ❌ | власний | власний | ❌ | ❌ | ❌ | ❌ |
| Затверджувати терапевта | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ |
| Затверджувати участь у корп. програмі | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ |
| Переглядати платежі | власні | власні | власні | ✅ read | ✅ | ✅ | ✅ |
| Видавати refund | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ |
| Поповнити кредити організації | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ |
| Переглянути кредитний баланс орг. | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ | ✅ |
| Управляти налаштуваннями платформи | ❌ | ❌ | ❌ | ❌ | partial | ✅ | ✅ |
| Управляти адмін-акаунтами | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |
| Переглядати звіти орг. | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ | ✅ |
| Залишати відгук (анонімно або з ім'ям) | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Переглядати audit logs | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ read | ✅ |
| Проходити тести | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Управляти блогом / тестами (CMS) | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ |
| Переглядати lead заявки | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ | ✅ |

---

## PHASE 1 — CORE PLATFORM {#phase-1}

---

### 8. Pages & Routes

#### `apps/web` — Public Routes (althena.com)
```
/                                 → Landing page (вибір мови)
/about
/pricing
/for-business
/cooperation
/therapists                       → Публічний каталог
/therapists/[slug]
/blog
/blog/[slug]
/blog/category/[categorySlug]
/tests
/tests/[slug]
/tests/[slug]/start
/tests/[slug]/question/[step]
/tests/[slug]/result
/sign-in                          → Supabase Auth Sign In
/sign-up                          → Supabase Auth Sign Up
/sign-up/client
/sign-up/therapist
/terms · /privacy · /cookies
```

#### `apps/app` — Authenticated Routes (app.althena.com)

**Client (включно з корпоративним клієнтом):**
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
/settings/preferences             → Ім'я, фото, нагадування, канали
/settings/corporate               → Прив'язка коду контракту (B2B)
```

**Therapist (включно з corporate consultant):**
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
/settings/corporate-program       → Запит на участь у корп. програмі (якщо ще не затверджено)
/onboarding
/onboarding/profile
/onboarding/credentials
/onboarding/availability
/onboarding/payout
```

**Video (isolated layout):**
```
/video/[roomId]                   → Daily.co кімната сесії, без sidebar
```

#### `apps/admin` — Admin Routes (admin.althena.com)
```
/overview
/users · /users/[userId]
/therapists · /therapists/[id] · /therapists/pending
/therapists/corporate-program     → Управління учасниками корп. програми
/clients
/sessions · /sessions/[bookingId]
/payments · /payments/[paymentId]
/payouts
/disputes · /disputes/[disputeId]
/organizations · /organizations/new
/organizations/[orgId]
/organizations/[orgId]/members
/organizations/[orgId]/credits    → Поповнення кредитів, лог транзакцій
/organizations/[orgId]/reports    → Формування та надсилання звітів на email HR
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
POST /webhooks/stripe             → payment/payout events → @repo/webhooks verify
POST /webhooks/daily              → Daily.co session events
```

---

### 9. Sidebar Navigation

#### 9.1 Client Sidebar (`apps/app` — role: client)
```
🌿 Althena          [Avatar + displayName]
─────────────────────────────────
🏠  Головна
🔍  Знайти терапевта
📅  Мої сесії
💬  Повідомлення         [лічильник непрочитаних — realtime]
💳  Оплата
─────────────────────────────────
⚙️  Налаштування
❓  Допомога
─────────────────────────────────
Footer (тільки для корпоративного клієнта):
🏢  [Назва організації] · Тип: [контракт]
    Кредитів: [залишок] (тільки для Тип 1/2)
```

> **Примітка:** Кабінет корпоративного клієнта ідентичний звичайному клієнту. Єдине додаткове — секція в Settings для введення коду контракту та footer-рядок з інформацією про організацію.

#### 9.2 Therapist Sidebar
```
🌿 Althena          [Avatar]
                    ● Online ▾
─────────────────────────────────
ГОЛОВНЕ
📊  Огляд
📅  Календар
👥  Мої клієнти
🎯  Сесії
💬  Повідомлення         [лічильник непрочитаних]
─────────────────────────────────
ФІНАНСИ
💰  Заробіток
📈  Аналітика
─────────────────────────────────
АКАУНТ
⚙️  Налаштування
   ↳ Профіль · Розклад · Виплати · Сповіщення · Корп. програма
─────────────────────────────────
❓  Допомога · 🚪 Вийти
```

> **Різниця для corporate consultant:** кабінет **не змінюється**. В Settings з'являється пункт "Корпоративна програма" (статус участі). Після затвердження — повідомлення в "дзвіночок".

#### 9.3 Admin Sidebar (`apps/admin`)
```
🌿 Althena ADMIN     [Avatar]
Роль: Operations Admin
─────────────────────────────────
ОГЛЯД          📊 Дашборд
КОРИСТУВАЧІ    👥 Всі · 🩺 Терапевти [badge] · 👤 Клієнти
ОПЕРАЦІЇ       📅 Сесії · ⚠️ Суперечки [badge] · ⭐ Відгуки
ФІНАНСИ        💳 Платежі · 💸 Виплати
B2B            🏢 Організації
КОНТЕНТ        📝 Блог · 🧪 Тести · 📥 Заявки
АНАЛІТИКА      📈 Дохід · Сесії · Терапевти · B2B
ПЛАТФОРМА      ⚙️ Налаштування · 👮 Адміни (super_admin) · 📋 Audit Log
─────────────────────────────────
🚪 Вийти
```

---

### 10. Forms

Всі форми використовують **react-hook-form** + **Zod** схеми. Валідація спільна між клієнтом і сервером.

#### 10.1 Therapist Registration
```typescript
const TherapistRegistrationSchema = z.object({
  email:           z.string().email().max(254),
  password:        z.string().min(8).regex(/^(?=.*[A-Z])(?=.*\d)(?=.*[!@#$%^&*])/),
  confirmPassword: z.string(),
  displayName:     z.string().min(2).max(100),
  acceptTerms:     z.literal(true),
  acceptPrivacy:   z.literal(true),
}).refine(d => d.password === d.confirmPassword, {
  message: 'Паролі не збігаються', path: ['confirmPassword'],
});
```

#### 10.2 Client Profile Settings (ім'я + фото)
```typescript
const ClientProfileSchema = z.object({
  displayName: z.string().min(1).max(60),
  // Може бути реальне ім'я або псевдонім — без обмежень на формат
  avatarUrl:   z.string().url().optional().or(z.literal('')),
});
```

#### 10.3 Corporate Contract Code (прив'язка клієнта до організації)
```typescript
const ContractCodeSchema = z.object({
  contractCode: z.string().min(6).max(32).regex(/^[A-Z0-9\-]+$/i),
});
// Сервер перевіряє: існування організації, активність контракту,
// достатність кредитів (для Тип 1/2), унікальність email клієнта в орг.
```

#### 10.4 Create Organization (Admin)
```typescript
const CreateOrganizationSchema = z.object({
  name:             z.string().min(2).max(100),
  contractType:     z.enum(['full_prepay', 'partial_prepay', 'postpay']),
  contractCode:     z.string().min(6).max(32).regex(/^[A-Z0-9\-]+$/i),
  initialCredits:   z.number().int().min(0), // EUR → cents (1 EUR = 100 credits в cents)
  billingEmail:     z.string().email(),
  billingContactName: z.string().min(2),
  contractStart:    z.date(),
  contractEnd:      z.date().optional(),
  restrictToPool:   z.boolean().default(false),
});
```

#### 10.5 Top Up Org Credits (Admin)
```typescript
const OrgCreditTopUpSchema = z.object({
  organizationId: z.string().cuid(),
  amountEur:      z.number().positive(), // EUR → конвертується в cents
  note:           z.string().min(5).max(500), // Наприклад: "Поповнення за договором №123 від 01.05.2026"
});
```

#### 10.6 Therapist Profile Setup
```typescript
const TherapistProfileSchema = z.object({
  displayName:       z.string().min(2).max(60),
  bio:               z.string().min(50).max(800),
  specializations:   z.array(z.string()).min(1),
  approaches:        z.array(z.string()).min(1),
  languages:         z.array(z.string()).min(1),
  yearsOfExperience: z.number().int().min(0).max(50),
  sessionRateCents:  z.number().int().min(3000).max(50000),
  sessionTypes:      z.array(z.enum(['video','chat'])).min(1),
  timezone:          z.string().min(1),
  phone:             z.string().regex(/^\+?[\d\s\-().]{7,20}$/).optional(),
});
```

#### 10.7 Availability Schedule
```typescript
const AvailabilitySlotSchema = z.object({
  dayOfWeek:           z.number().int().min(0).max(6),
  startTime:           z.string().regex(/^([0-1]\d|2[0-3]):[0-5]\d$/),
  endTime:             z.string().regex(/^([0-1]\d|2[0-3]):[0-5]\d$/),
  slotDurationMinutes: z.enum(['30','50','60','90']).transform(Number),
}).refine(d => d.endTime > d.startTime, { message: 'endTime має бути після startTime' });
```

#### 10.8 Session Note
```typescript
const SessionNoteSchema = z.object({
  noteType:           z.enum(['soap','progress','intake','discharge']),
  content:            z.string().min(20).max(10000),
  isSharedWithClient: z.boolean().default(false),
  subjective:  z.string().optional(),
  objective:   z.string().optional(),
  assessment:  z.string().optional(),
  plan:        z.string().optional(),
});
```

#### 10.9 Review (з опцією анонімності)
```typescript
const ReviewSchema = z.object({
  rating:      z.number().int().min(1).max(5),
  comment:     z.string().min(10).max(1000).optional(),
  isPublic:    z.boolean().default(true),
  isAnonymous: z.boolean().default(false),
  // Якщо isAnonymous = true → відображається як "Анонімний користувач"
  // Якщо isAnonymous = false → відображається displayName клієнта
});
```

#### 10.10 Reminder Settings (Клієнт)
```typescript
const ReminderSettingsSchema = z.object({
  reminderFrequency: z.enum(['session_day_only', 'day_before', 'both', 'none']),
  reminderChannels:  z.array(z.enum(['email', 'push'])).min(1),
});
```

#### 10.11 Platform Settings (Admin)
```typescript
const PlatformSettingsSchema = z.object({
  commissionRatePercent:         z.number().min(0).max(50).multipleOf(0.5),
  corporateCommissionDiscount:   z.number().min(0).max(30), // % знижки від комісії для corporate consultant
  cancellationWindowHours:       z.number().int().min(1).max(72),
  cancellationFeePercent:        z.number().min(0).max(100),
  payoutSchedule:                z.enum(['daily','weekly','biweekly','monthly']),
  sessionDurationOptions:        z.array(z.number().int()),
  defaultSessionDuration:        z.number().int(),
  supportEmail:                  z.string().email(),
  platformName:                  z.string().min(1).max(100),
});
```

#### 10.12 Admin Invite (Super Admin)
```typescript
const AdminInviteSchema = z.object({
  email:    z.string().email(),
  fullName: z.string().min(2).max(100),
  tier:     z.enum(['support','moderator','finance','operations']),
});
```

---

### 11. Tables & Lists

Всі списки використовують **cursor-based pagination** (без OFFSET). Фільтрація через URL search params з Zod coercion.

#### 11.1 Therapist List (`/therapists`)
Columns: Avatar + Ім'я | Спеціалізації | Статус | Сесій | Рейтинг | Корп. програма | Приєднався | Дії
Filters: Статус, Спеціалізація, Мова, Тільки корп. консультанти
Actions: Переглянути · Затвердити · Призупинити · Повідомлення

#### 11.2 Corporate Program Queue (`/therapists/corporate-program`)
Columns: Ім'я | Email | Дата запиту | Статус | Дії
Actions: Затвердити (isCorporateConsultant = true) · Відхилити

#### 11.3 Organizations (`/organizations`)
Columns: Назва | Тип контракту | Кредитний баланс | Активних клієнтів | Сесій цього місяця | Статус | Дії
Filters: Тип контракту, Статус

#### 11.4 Org Credits Log (`/organizations/[orgId]/credits`)
Columns: Дата | Тип (поповнення/списання) | Сума | Бронювання | Примітка | Адмін
Sorting: `createdAt` desc
Actions: Поповнити кредити (кнопка → форма OrgCreditTopUpSchema)

#### 11.5 Org Members (`/organizations/[orgId]/members`)
Columns: Ім'я (displayName) | Email | Статус | Сесій всього | Приєднався | Дії
Actions: Деактивувати · Змінити код контракту (генерує новий contractCode для орг.)

#### 11.6 All Users (`/users`)
Columns: Avatar + Ім'я | Роль | Email | Статус | Остання активність | Дії
Filters: Роль, Статус, Дата реєстрації
Search: full-text по `displayName`, `email`

#### 11.7 Payments (`/payments`)
Columns: Дата | Клієнт | Терапевт | Сума | Комісія | Кредити орг. | Статус | Дії
Filters: Статус, Діапазон дат, Організація

#### 11.8 Reviews (`/reviews`)
Columns: Дата | Клієнт | Терапевт | Рейтинг | Коментар | Публічний | Анонімний | Помічений | Дії
Filters: Помічені, Діапазон рейтингу, Публічні/Приватні, Анонімні
Actions: Зняти помітку · Видалити · Зробити приватним

#### 11.9 Audit Log (`/audit-log`)
Columns: Час | Адмін | Дія | Тип цілі | ID цілі | IP-адреса | Деталі
Sorting: `createdAt` desc (immutable)
Export: CSV

---

### 12. API Endpoints

#### `apps/app` — Authenticated REST API (Next.js Route Handlers)

Всі маршрути вимагають Supabase Auth session. Перевірка ролі через `@repo/auth` middleware helper.

```
Profiles
GET    /api/me
PATCH  /api/me
DELETE /api/me                      → Supabase Auth delete + Prisma soft-delete

Therapists (public browse — без auth)
GET    /api/therapists              → filter, search, paginate
GET    /api/therapists/[id]         → публічний профіль
GET    /api/therapists/[id]/reviews
GET    /api/therapists/me           → власний профіль (therapist auth)
PATCH  /api/therapists/me

Corporate Program (therapist)
POST   /api/therapists/me/corporate-program/request → запит на участь
DELETE /api/therapists/me/corporate-program         → вихід з програми

Availability
GET    /api/availability/[therapistId]
POST   /api/availability
DELETE /api/availability/[id]
GET    /api/availability/[therapistId]/slots?date=YYYY-MM-DD
POST   /api/availability/exceptions
DELETE /api/availability/exceptions/[id]

Bookings
GET    /api/bookings
POST   /api/bookings               → B2B перевірка кредитів → Stripe Payment Intent (або резервування кредитів)
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
POST   /api/notes                  → зашифровано перед Prisma INSERT
PATCH  /api/notes/[id]
DELETE /api/notes/[id]

Messages
GET    /api/conversations
POST   /api/conversations
GET    /api/conversations/[id]/messages
POST   /api/conversations/[id]/messages  → зашифровано перед INSERT
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
POST   /api/disputes               → клієнт піднімає суперечку
GET    /api/disputes/[id]
PATCH  /api/disputes/[id]/resolve  → тільки admin

Reviews
POST   /api/reviews                → тільки клієнт, після сесії
GET    /api/reviews?therapistId=[id]
PATCH  /api/reviews/[id]/flag

B2B — клієнтська сторона
POST   /api/corporate/link         → введення contractCode, прив'язка до організації
DELETE /api/corporate/link         → від'язатися від організації
GET    /api/corporate/balance      → баланс кредитів (тільки для Тип 1/2)
```

#### `apps/admin` — Admin API

Всі вимагають admin JWT + TOTP 2FA session check.

```
GET/PATCH  /api/admin/users/[id] + /suspend + /restore + DELETE
GET/PATCH  /api/admin/therapists/[id] + /approve + /reject + /suspend
GET        /api/admin/therapists/pending
GET        /api/admin/therapists/corporate-program (черга запитів на участь)
PATCH      /api/admin/therapists/[id]/corporate-program/approve
PATCH      /api/admin/therapists/[id]/corporate-program/reject
GET        /api/admin/sessions · /payments · /payouts
PATCH      /api/admin/payouts/[id]/retry
GET/PATCH  /api/admin/disputes/[id]/resolve
GET/PATCH  /api/admin/reviews/[id]/remove + /unflag
GET/POST/GET /api/admin/organizations + /[id]
POST       /api/admin/organizations/[id]/credits/top-up
GET        /api/admin/organizations/[id]/credits/transactions
POST       /api/admin/organizations/[id]/contract-code/regenerate
POST       /api/admin/organizations/[id]/reports/send  → надіслати звіт на email HR
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

#### Public API (в `apps/web`)

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
POST /api/public/tests/[testId]/claim     ← Supabase Auth required
POST /api/public/blog/[slug]/view
GET  /api/public/faq?category=[category]
```

---

### 13. Real-time Functionality

**Provider:** Supabase Realtime через `@repo/realtime`

```typescript
// packages/realtime/src/channels.ts

// Повідомлення в розмові
export function messageChannel(conversationId: string, handler: (msg: Message) => void) {
  return supabase.channel(`conversation:${conversationId}`)
    .on('postgres_changes', {
      event: 'INSERT', table: 'messages',
      filter: `conversation_id=eq.${conversationId}`
    }, handler);
}

// Сповіщення користувача
export function notificationChannel(userId: string, handler: (n: Notification) => void) {
  return supabase.channel(`notifications:${userId}`)
    .on('postgres_changes', {
      event: 'INSERT', table: 'notifications',
      filter: `user_id=eq.${userId}`
    }, handler);
}

// Оновлення бронювань (терапевт)
export function bookingChannel(therapistId: string, handler: (b: Booking) => void) {
  return supabase.channel(`bookings:${therapistId}`)
    .on('postgres_changes', {
      event: 'UPDATE', table: 'bookings',
      filter: `therapist_id=eq.${therapistId}`
    }, handler);
}

// Черги адміністратора (значки)
export function adminQueueChannel(handlers: { dispute: () => void; pending: () => void; corporateProgram: () => void }) {
  return supabase.channel('admin:queues')
    .on('postgres_changes', { event: '*', table: 'disputes' }, handlers.dispute)
    .on('postgres_changes', {
      event: '*', table: 'therapist_profiles',
      filter: `onboarding_status=eq.pending`
    }, handlers.pending)
    .on('postgres_changes', {
      event: 'UPDATE', table: 'therapist_profiles',
      filter: `is_corporate_consultant=eq.false`
    }, handlers.corporateProgram);
}
```

**Video:** Daily.co per-booking rooms. Кімната створюється server-side при `POST /api/sessions/[bookingId]/start`.

**Scheduled jobs (Supabase Edge Functions):**

| Job | Розклад | Дія |
|---|---|---|
| Нагадування про бронювання | Cron: щогодини | T-24h, T-1h email через Resend |
| Запит відгуку після сесії | Cron: щогодини | T+1h запит відгуку клієнту |
| Щотижнева виплата терапевту | Cron: пн 09:00 | Зведення виплат |
| Генерація рахунку B2B (Тип 3) | Cron: 1-го числа 06:00 | Stripe Invoice для postpay організацій |
| Звіт використання орг. | Cron: 1-го числа 07:00 | Обчислення → `organization_usage_reports` |
| Очищення анонімних сесій тестів | Cron: щодня 03:00 | DELETE test_sessions WHERE анонімна + прострочена |

---

### 14. File Functionality

**Provider:** `@repo/storage` wraps Supabase Storage

| Тип файлу | Bucket | Доступ | Макс розмір | CDN | Обробка |
|---|---|---|---|---|---|
| Аватар | `avatars` | Public | 5 MB | ✅ | Sharp → 400×400 WebP |
| Документи терапевта | `credentials` | Private, signed URL 15 хв | 10 MB | ❌ | None |
| Вкладення повідомлень | `attachments` | Private, signed URL 1h | 20 MB | ❌ | None |
| Логотипи організацій | `org-assets` | Public | 2 MB | ✅ | Sharp → 200×200 WebP |
| Обкладинки блогу/тестів | `public-media` | Public | 5 MB | ✅ | Sharp → 400w/800w/1200w WebP + 1200×630 OG |
| OG-зображення | `public-media/og` | Public | 2 MB | ✅ | next/og ImageResponse (Edge) |
| Email templates | `email-templates` | Admin-only | 500 KB | ❌ | None |
| Платформ. активи | `platform-assets` | Public | 10 MB | ✅ | None |

**Upload flow:**
1. Клієнт запитує presigned URL у `@repo/storage`
2. Клієнт завантажує безпосередньо в Supabase Storage
3. Supabase Edge Function тригериться при завантаженні → Sharp обробка → варіанти збережені
4. Запис `media_assets` створюється через Prisma з `blurhash`
5. API відповіді посилаються на signed URLs, отримані під час читання — ніколи сирі storage URLs

---

### 15. Role-Based Navigation Components

```typescript
// packages/design-system/src/providers.tsx
export function DesignSystemProvider({ children, role }: Props) {
  return (
    <SupabaseAuthProvider>
      <ThemeProvider>
        <RoleProvider role={role}>
          {children}
        </RoleProvider>
      </ThemeProvider>
    </SupabaseAuthProvider>
  );
}

// Кожен app компонує власну shell
// apps/app — ClientShell або TherapistShell на основі profile.role з Supabase Auth
// apps/admin — AdminShell з tier-filtered nav
```

**Supabase Auth Middleware (shared in `@repo/auth`):**

```typescript
// packages/auth/src/middleware.ts
import { createServerClient } from '@supabase/ssr';
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

const PUBLIC_ROUTES = [
  '/', '/about', '/pricing', '/for-business', '/cooperation',
  '/sign-in', '/sign-up',
  '/api/public',
  '/terms', '/privacy', '/cookies',
];

function isPublicRoute(pathname: string) {
  return PUBLIC_ROUTES.some(r => pathname === r || pathname.startsWith(r + '/'));
}

export async function authMiddleware(request: NextRequest) {
  const response = NextResponse.next();
  
  const supabase = createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll: () => request.cookies.getAll(),
        setAll: (cookiesToSet) => {
          cookiesToSet.forEach(({ name, value, options }) =>
            response.cookies.set(name, value, options)
          );
        },
      },
    }
  );

  const { data: { session } } = await supabase.auth.getSession();
  const role = session?.user?.app_metadata?.role as string | undefined;
  const pathname = request.nextUrl.pathname;

  if (!session && !isPublicRoute(pathname)) {
    return NextResponse.redirect(new URL('/sign-in', request.url));
  }

  if (pathname.startsWith('/admin') && !['admin', 'super_admin'].includes(role ?? '')) {
    return NextResponse.redirect(new URL('/unauthorized', request.url));
  }

  return response;
}
```

---

## PHASE 2 — PUBLIC SITE (Guest Domain) {#phase-2}

---

### 16. Public Domain Entities

Див. Section 4 Prisma schema: `BlogPost`, `BlogAuthor`, `BlogCategory`, `Test`, `TestQuestion`, `TestSession`, `NewsletterSubscriber`, `LeadSubmission`, `CooperationRequest`, `FaqItem`, `ContentPage`, `MediaAsset`.

---

### 17. Route Architecture (`apps/web`)

```
apps/web/app/
├── (public)/
│   ├── layout.tsx                   ← PublicLayout (nav, footer, cookie banner, вибір мови)
│   ├── page.tsx                     → /                      [ISR: 3600s]
│   ├── about/page.tsx               → /about                 [ISR: 86400s]
│   ├── pricing/page.tsx             → /pricing               [ISR: 3600s]
│   ├── for-business/page.tsx        → /for-business          [ISR: 3600s]
│   ├── cooperation/page.tsx         → /cooperation           [ISR: 3600s]
│   ├── blog/
│   │   ├── layout.tsx
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
├── api/public/                      ← Public Route Handlers (без auth)
│   ├── newsletter/route.ts
│   ├── newsletter/confirm/route.ts
│   ├── newsletter/unsubscribe/route.ts
│   ├── leads/route.ts
│   ├── cooperation/route.ts
│   ├── tests/[testId]/session/route.ts
│   ├── tests/[testId]/answer/route.ts
│   ├── tests/[testId]/complete/route.ts
│   ├── tests/[testId]/claim/route.ts   ← Supabase Auth required
│   ├── blog/[slug]/view/route.ts
│   └── faq/route.ts
│
├── sitemap.ts                       → /sitemap.xml
├── robots.ts                        → /robots.txt
└── opengraph-image.tsx
```

---

### 18. Public Forms

#### 18.1 NewsletterForm
```typescript
const NewsletterSchema = z.object({
  email:       z.string().email().max(254),
  source:      z.enum(['footer', 'blog_inline', 'test_result', 'popup']),
  sourceSlug:  z.string().optional(),
  website:     z.string().max(0), // honeypot
});
// Arcjet: 3 запити/IP/годину
// Завжди повертати 200 — без enumeration email
// Double opt-in через Resend (обов'язково для EU)
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
  website:     z.string().max(0),
});
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
```

---

### 19. Public API Endpoints

Rate limiting через `@repo/security` (Arcjet) на всіх `/api/public/*` маршрутах.

```
POST /api/public/newsletter           → 3/IP/год
POST /api/public/leads                → 5/IP/год
POST /api/public/cooperation          → 3/IP/год
POST /api/public/tests/*/session      → 10/IP/год
PATCH /api/public/tests/*/answer      → 120/IP/год
```

---

### 20. SEO Architecture

Реалізовано через `@repo/seo`. Стратегія metadata, JSON-LD та sitemap — без змін від попередньої версії специфікації.

#### JSON-LD Schema by Page

| Page | Schema Types |
|---|---|
| `/` | `WebSite` + `Organization` + `MedicalOrganization` |
| `/blog/[slug]` | `Article` або `MedicalWebPage` + `BreadcrumbList` |
| `/tests/[slug]` | `MedicalWebPage` + `FAQPage` |
| `/therapists/[slug]` | `Person` + `MedicalBusiness` + `BreadcrumbList` |
| `/for-business` | `WebPage` + `Service` |
| `/about` | `AboutPage` + `Organization` |

---

### 21. State Machines (Guest Domain)

Див. Section 2.10 (Test Execution), 2.11 (Newsletter), 2.12 (Content Publishing).

---

### 22. Frontend Architecture & Rendering Strategy

#### Server vs Client Component

| Component | Тип | Причина |
|---|---|---|
| Blog post content | Server | SEO-critical, LCP |
| Newsletter form | Client | Form state, Arcjet |
| Test question step | Client | Interactive, localStorage |
| Therapist catalog filters | Client | URL search params |
| Therapist card grid | Server | SEO-critical |
| Cookie consent banner | Client | localStorage check |
| Вибір мови (головна) | Client | cookie/localStorage |
| FAQ accordion | Client | open/close state |
| Home hero | Server | Above-fold, LCP-critical |
| Supabase Auth `<UserButton>` | Client | Supabase SDK requirement |

#### Rendering Strategy

| Route | Strategy | Примітки |
|---|---|---|
| `/` | ISR 3600s | Home/marketing |
| `/blog` | ISR 60s | |
| `/blog/[slug]` | ISR 300s + `generateStaticParams` | Top 100 by viewCount |
| `/tests/[slug]/start` | SSR | Session detection (httpOnly cookie) |
| `/tests/[slug]/question/[step]` | Client only | Interactive |
| `/tests/[slug]/result` | SSR, `noindex` | Session-specific |
| `/therapists` | SSR | Filter params vary |
| `/terms`, `/privacy`, `/cookies` | SSG | Never changes |

---

### 23. Analytics & Tracking Architecture

Реалізовано через `@repo/analytics` (PostHog + GA4 з consent gating).

#### Consent Tiers

```typescript
export type ConsentTier = 0 | 1 | 2; // 0: none, 1: analytics, 2: marketing

export function initAnalytics(tier: ConsentTier) {
  if (tier >= 1) {
    posthog.init(env.NEXT_PUBLIC_POSTHOG_KEY, {
      api_host: 'https://eu.posthog.com', // EU data residency
      capture_pageview: true,
    });
  }
  if (tier >= 2) {
    // GA4 full tracking
  }
}
```

---

## ADMIN PANEL DEEP DIVE {#admin-panel}

---

### 24. Admin Roles & Permission Architecture

- Тільки `super_admin` може запрошувати нових адмінів
- Invite flow: email + fullName + tier → Supabase Auth invitation → перший вхід → обов'язкове налаштування TOTP
- Session timeout: 4h inactivity
- Всі дії адміна логуються через Prisma в `AdminAuditLog` (immutable)
- `apps/admin` на `admin.althena.com` з окремими CSP headers

---

### 25. Admin Settings Module

#### 25.1 General Settings
```typescript
interface GeneralSettings {
  platformName:         string;
  platformTagline:      string;
  platformLogo:         File | string;
  supportEmail:         string;
  supportPhone?:        string;
  defaultTimezone:      string;
  defaultLanguage:      'uk' | 'en';
  maintenanceMode:      boolean;
  maintenanceBannerText?: string;
}
```

#### 25.2 Commission & Pricing
```typescript
interface CommissionSettings {
  defaultCommissionPercent:         number; // 0–50
  corporateConsultantDiscountPercent: number; // знижка для терапевтів у корп. програмі (зазвичай 10%)
  cancellationWindowHours:          number; // 1–72
  lateCancellationFeePercent:       number;
  noShowFeePercent:                 number;
}
```

#### 25.3 Payments & Payouts
```typescript
interface PaymentSettings {
  payoutSchedule:      'daily' | 'weekly' | 'biweekly' | 'monthly';
  payoutMinimumCents:  number;
  supportedCurrencies: string[];
  defaultCurrency:     string; // 'eur'
}
```

#### 25.4 Email Templates
Шаблони в `@repo/email` як React Email компоненти. Admin-редаговані subject + HTML override зберігаються в `PlatformSetting`.

Шаблони: `booking_confirmed` · `booking_cancelled` · `session_reminder_24h` · `session_reminder_1h` · `post_session_review_request` · `therapist_approved` · `therapist_rejected` · `corporate_consultant_approved` · `payout_sent` · `dispute_resolved` · `org_usage_report`

#### 25.5 B2B Configuration
```typescript
interface B2BSettings {
  b2bEnabled:               boolean;
  defaultContractTypes:     ContractType[];
  corporateCommissionDiscount: number; // %
  usageReportFrequency:     'monthly'; // наразі тільки місячна
  privacyGuaranteeText:     string;   // показується корпоративним клієнтам
}
```

#### 25.6 Compliance
```typescript
interface ComplianceSettings {
  dataRetentionDays:              number; // default 2555 (7 років)
  auditLogRetentionDays:          number;
  gdprMode:                       boolean;
  cookieConsentRequired:          boolean;
}
```

---

### 26. Admin User Management

#### 26.1 Therapist Review Screen (`/therapists/[id]`)
- Огляд профілю (фото, bio, спеціалізації)
- Панель документів: номер ліцензії, регіон, термін дії, viewer Supabase Storage (signed URL 15 хв, modal)
- Статус участі в корпоративній програмі + кнопки "Затвердити участь" / "Відхилити"
- Дії: **Затвердити** | **Відхилити** (modal з причиною) | **Запросити інфо** | **Призупинити**

#### 26.2 User Detail Screen (`/users/[id]`)
- Профіль: avatar, displayName, email, роль, приєднався, остання активність, статус
- Role-specific panel: документи терапевта АБО org membership клієнта
- Дії: **Призупинити** | **Відновити** | **Видалити (GDPR)** | **Скинути пароль** | **Імітувати** (super_admin + погодження 2-го адміна)

#### 26.3 Corporate Program Management (`/therapists/corporate-program`)
- Список терапевтів з `isCorporateConsultant = false` та pending запитами
- Статистика: кількість active corporate consultants, покриття по регіонах/спеціалізаціях
- Дії: Затвердити · Відхилити · Переглянути профіль

---

### 27. B2B Organizations Module

#### 27.1 Organization Detail Tabs
1. **Огляд** — тип контракту, дати, кредитний баланс, лічильник активних клієнтів
2. **Клієнти** — список зареєстрованих корпоративних клієнтів, деактивація
3. **Кредити** — поповнення, лог транзакцій, поточний баланс
4. **Код контракту** — поточний код, кнопка "Перегенерувати" (при підозрі на витік — за запитом HR)
5. **Звіти** — формування та надсилання звітів на email HR
6. **Пул терапевтів** — обмежити/розширити список доступних corporate consultants
7. **Рахунки** — для Тип 3 (postpay): список Stripe Invoice

#### 27.2 B2B Session Coverage Logic

```typescript
// packages/database/src/b2b.ts
export async function checkAndReserveB2BCoverage(
  clientId: string,
  orgId: string,
  sessionRateCents: number,
): Promise<{
  canBook: boolean;
  orgCreditUsed: number;
  clientPaymentRequired: number;
  reason?: string;
}> {
  const [member, org] = await Promise.all([
    prisma.organizationMember.findFirst({ where: { clientId, organizationId: orgId } }),
    prisma.organization.findUnique({ where: { id: orgId } }),
  ]);

  if (!member || !org || !org.isActive) {
    return { canBook: false, orgCreditUsed: 0, clientPaymentRequired: sessionRateCents, reason: 'ORG_NOT_FOUND' };
  }

  if (member.status !== 'active') {
    return { canBook: false, orgCreditUsed: 0, clientPaymentRequired: sessionRateCents, reason: 'MEMBER_DEACTIVATED' };
  }

  switch (org.contractType) {
    case 'full_prepay':
      if (org.creditBalance >= sessionRateCents) {
        return { canBook: true, orgCreditUsed: sessionRateCents, clientPaymentRequired: 0 };
      }
      return { canBook: false, orgCreditUsed: 0, clientPaymentRequired: 0, reason: 'INSUFFICIENT_CREDITS' };

    case 'partial_prepay': {
      // Приклад: 50% з кредитів, 50% клієнт
      // Налаштовується адміністратором при укладанні договору (зберігається в Organization)
      const orgPortion = Math.floor(sessionRateCents * 0.5);
      const clientPortion = sessionRateCents - orgPortion;
      if (org.creditBalance >= orgPortion) {
        return { canBook: true, orgCreditUsed: orgPortion, clientPaymentRequired: clientPortion };
      }
      return { canBook: false, orgCreditUsed: 0, clientPaymentRequired: sessionRateCents, reason: 'INSUFFICIENT_CREDITS' };
    }

    case 'postpay':
      // Клієнт платить повну суму; організація отримує рахунок наприкінці місяця
      return { canBook: true, orgCreditUsed: 0, clientPaymentRequired: sessionRateCents };
  }
}
```

#### 27.3 Credit Top-Up Flow (Admin)

```typescript
// POST /api/admin/organizations/[id]/credits/top-up
// Вхід: { amountEur, note }
// Операції:
1. prisma.orgCreditTransaction.create({ type: 'top_up', amountCents: amountEur * 100, note, createdById: adminId })
2. prisma.organization.update({ creditBalance: { increment: amountEur * 100 } })
3. prisma.adminAuditLog.create({ action: 'org.credit_top_up', targetId: orgId, metadata: { amountEur, note } })
```

#### 27.4 Contract Code Regeneration

```typescript
// POST /api/admin/organizations/[id]/contract-code/regenerate
// Виклик за запитом HR (при підозрі на витік коду)
// Старий код стає недійсним; нові реєстрації за ним блокуються
// Вже зареєстровані клієнти (orgMemberStatus = 'active') залишаються прив'язаними
1. prisma.organization.update({ contractCode: generateUniqueCode() })
2. prisma.adminAuditLog.create({ action: 'org.contract_code_regenerated', ... })
3. Повернути новий код адміну (він надсилає HR-у поза платформою)
```

#### 27.5 B2B Usage Report Generation

```typescript
// POST /api/admin/organizations/[id]/reports/send
// Вручну або автоматично (cron 1-го числа місяця)
1. Агрегувати bookings за звітний місяць
2. Обчислити: totalSessions, activeMembers (≥1 сесія), avgRating
   → k-анонімність: якщо activeMembers < 5, приховати це поле
3. Зберегти: prisma.organizationUsageReport.create(...)
4. Надіслати email: resend.send({ to: org.billingEmail, template: 'org_usage_report', data: reportData })
5. Для Тип 3 (postpay): stripe.invoices.create({ customer: org.stripeCustomerId, ... })
```

---

## CROSS-CUTTING CONCERNS {#cross-cutting}

---

### Security

| Concern | Implementation |
|---|---|
| Автентифікація | Supabase Auth (email/password, Google OAuth, TOTP 2FA для адмінів) |
| Авторизація | `app_metadata.role` в Supabase Auth + server-side перевірки в Route Handlers |
| PHI encryption | AES-256 на app layer перед Prisma INSERT (нотатки, повідомлення, PII поля) |
| Ключі шифрування | Vercel environment variables — ротація через Vercel dashboard |
| Rate limiting | `@repo/security` (Arcjet) — bot detection + rate limiting на всіх public API |
| Webhook verification | `@repo/webhooks` (Svix) — Stripe signature verification |
| Ізоляція адмін-панелі | Окремий `apps/admin` на `admin.althena.com` зі strict CSP |
| HIPAA audit trail | `AdminAuditLog` — immutable, 7-річне зберігання |
| Contract code | Не публікується в API; адмін передає HR поза платформою; можна перегенерувати |
| Credit transactions | Тільки admin може нараховувати; лог незмінний |
| Документи терапевта | Supabase Storage private bucket, 15-хв signed URLs |
| Video E2E | Daily.co DTLS-SRTP |
| Headers | `X-Frame-Options: DENY`, `Strict-Transport-Security` через `@repo/next-config` |

### Performance

| Concern | Implementation |
|---|---|
| Build caching | Turborepo Remote Cache |
| List views | Cursor-based Prisma pagination |
| Slot computation | Supabase Edge Function → Vercel KV cache (5 хв на терапевта) |
| Admin analytics | Postgres materialized views, оновлюються щогодини |
| Blog/test pages | ISR з `generateStaticParams` |
| Images | Supabase CDN + `next/image` (AVIF/WebP) + blurhash placeholders |
| DB connections | Supabase pgBouncer pooler + Prisma connection management |

### Sprint Prioritization

| Sprint | Deliverable |
|---|---|
| 1 | Monorepo scaffold, Turborepo, core packages skeleton |
| 2 | `@repo/database` Prisma schema + migrations, `@repo/auth` Supabase Auth setup |
| 3 | `apps/app` + `apps/api` bootstrap, Supabase Auth session management |
| 4 | Therapist onboarding wizard + admin credential review |
| 5 | Availability engine + slot computation |
| 6 | Booking flow + session state machine + Stripe Payment Intent |
| 7 | B2B: Organization CRUD, contract code, credit top-up, coverage logic |
| 8 | B2B: Корпоративна реєстрація клієнтів (contractCode → прив'язка) |
| 9 | Corporate consultant: approval flow, маркер, повідомлення в "дзвіночок" |
| 10 | Video session integration (Daily.co) |
| 11 | Session notes (PHI encryption) |
| 12 | Messaging + Supabase Realtime |
| 13 | Stripe Connect Express (therapist payouts) |
| 14 | Client-side billing + payment methods |
| 15 | `@repo/notifications` (Knock) — in-app + email |
| 16 | B2B billing: Stripe Invoice (Тип 3), usage reports, email HR |
| 17 | Admin analytics dashboard |
| 18 | Disputes + refund flow |
| 19 | Reviews + модерація + анонімні відгуки |
| 20 | Налаштування нагадувань клієнта (frequency + channels) |
| 21 | Platform settings module |
| 22 | Admin management (super_admin), audit log, impersonation |
| 23 | `apps/web` public site: home, therapist directory, static pages |
| 24 | Blog CMS + ISR + admin CRUD |
| 25 | Psychological test engine |
| 26 | Newsletter + lead capture + `@repo/analytics` |
| 27 | `@repo/observability` (Sentry, BetterStack), bundle analysis |
| 28+ | Mobile PWA, push notifications, i18n (`@repo/i18n`) |

### High-Risk Areas

| Ризик | Область | Митигація |
|---|---|---|
| PHI encryption key rotation | Нотатки, повідомлення | App-layer rotation job; без pgcrypto |
| Stripe Connect complexity | Виплати терапевтам | Спочатку вручну, автоматизація — Sprint 13 |
| B2B credit race conditions | Списання кредитів | Prisma `$transaction` + оптимістичне блокування |
| Contract code витік | B2B безпека | Перегенерація через адміна; старий код миттєво стає недійсним |
| Slot race conditions | Бронювання | Prisma `$transaction` |
| GDPR data deletion | User data | Prisma soft-delete + anonymization pipeline |
| Anonymous org reports | B2B приватність | k-анонімність: приховати якщо `activeMembers < 5` |
| Test score manipulation | Test engine | Скор обчислюється тільки server-side |
| Newsletter email enumeration | Public API | Завжди повертати 200 з однаковим body |
| GDPR double opt-in | Newsletter EU | Resend confirmation email обов'язковий |
| Supabase Auth webhook replay | Auth sync | Перевіряти timestamp + підпис |
| ISR stale health content | Blog | `revalidatePath` при публікації (on-demand) |

---

_Spec version: 5.0 — May 2026_
_Стек: Next.js 15 · Turborepo · Supabase (PostgreSQL + Auth) · Prisma · Stripe · Daily.co_
_Auth: Supabase Auth (замінює Clerk у всіх компонентах)_
_B2B: Кредитна система + 3 типи контрактів + corporate consultant маркер_
_Validate all `[INFERRED]` sections against actual Figma screens before Sprint 1._
