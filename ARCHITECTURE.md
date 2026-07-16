# Architecture

This document is a practical map for AI coder agents working on Opportunity Pipeline Monitor. It explains where behavior lives, which boundaries matter, and how to change the app without breaking CRM workflows.

## High-Level Shape

```text
Browser
  -> Next.js App Router pages and client components
  -> Next.js API route handlers
  -> lib/services business logic
  -> Prisma client
  -> SQLite
  -> Local attachment storage
```

The app is a server-rendered internal CRM. Pages usually load data on the server, then delegate interactive form behavior to focused client components.

## Source Layout

```text
app/
  api/                  API route handlers
  dashboard/            action dashboard
  opportunities/        opportunity list, board, create, edit, detail
  companies/            company list, create, edit, detail
  contacts/             contact list, create, edit, detail
  follow-ups/           follow-up queues
  reports/              role-aware reporting
  settings/             admin settings/user surfaces
  profile/              self-service profile and password

components/
  dashboard/            dashboard widgets
  opportunities/        opportunity forms, tables, board, detail tools
  companies/            company forms and company contact list
  contacts/             contact forms
  follow-ups/           follow-up table and tabs
  layout/               authenticated shell, sidebar, topbar
  settings/             admin user controls
  ui/                   small shared UI primitives

lib/
  db.ts                 Prisma client
  domain/               enums and business-rule helpers
  display/              label formatting
  navigation/           app navigation definitions
  opportunities/        opportunity UI/domain helpers
  services/             business logic and persistence workflows
  storage/              local file attachment storage
  validation/           request/form schemas

prisma/
  schema.prisma         database schema and enum source
  migrations/           migration history
  seed.ts               demo seed data

tests/
  services/             service-level behavior tests
  validation/           validation tests
  storage/              attachment storage tests
  ui/                   source-level UI regression tests
  e2e/                  Playwright acceptance tests
```

## Runtime Responsibilities

### Pages

Server-rendered protected pages live in `app/**/page.tsx`.

There is no authentication. Pages can be accessed directly.

### API Routes

Route handlers live in `app/api/**/route.ts`.

API routes do not require authentication. They should validate input with schemas from `lib/validation/**` and call service functions for business behavior.

### Services

Business rules belong in `lib/services/**`.

Important service ownership:

- `opportunity-service.ts`: opportunity CRUD, list filters, detail serialization, export
- `activity-service.ts`: activity timeline and last-contact updates
- `activity-workflow-service.ts`: guided activity/proposal flow
- `stage-transition-service.ts`: quotation, won, lost, hold transitions
- `follow-up-service.ts`: follow-up queues and task completion
- `dashboard-service.ts`: dashboard metrics and dashboard lists
- `permission-service.ts`: reusable role and visibility helpers
- `user-service.ts`: manage sales reps
- `attachment-service.ts`: proposal/activity attachment lifecycle
- `report-service.ts`: role-aware report data

When adding behavior that affects more than one route, put it in a service and test it there.

### Validation

Request and form schemas live in `lib/validation/**`. Keep parsing, coercion, and error messages close to the API/form boundary, then pass typed values into services.

### Domain Rules

`lib/domain/rules.ts` owns calculated behavior such as health, aging, warnings, and overdue checks. `lib/domain/enums.ts` and `prisma/schema.prisma` must stay aligned when enum choices change.

### Database

Prisma schema is the database source of truth. Main models:

- `User`
- `Company`
- `Contact`
- `Opportunity`
- `OpportunityContact`
- `Activity`
- `Proposal`
- `ProposalAttachment`
- `ActivityAttachment`
- `FollowUpTask`
- `StageHistory`

Use migrations for schema changes. Run Prisma generation after schema edits.

## No Authentication (Sales Reps Only)

The system has no login requirement.
Users (Sales Reps) are managed in the settings (`/settings/users`).
When creating or editing opportunities or tasks, the user selects a Sales Rep from the system to assign as the owner.
Anyone can view and mutate any data.

## Business Rules To Preserve

Follow-up:

- Main queue is based on `opportunities.nextFollowUpAt`.
- `follow_up_tasks` are granular assigned task rows.
- Won and Lost should not appear in active follow-up queues.
- Hold can reappear when revive date is due.

Activities:

- Customer-contact activities update `opportunities.lastContactAt`.
- Internal notes and internal meetings do not update `lastContactAt`.
- `contactType` is separate from `activityType`.

Stage transitions:

- Use dedicated transition endpoints and `stage-transition-service.ts`.
- Quotation Sent requires quotation amount, sent date, and next follow-up date.
- Quotation Sent creates or updates the latest quotation proposal with status Sent.
- Won requires final amount, expected project start date, PO status, main contact, and handover note.
- Lost requires lost reason.
- Hold requires hold reason.
- Every stage transition writes `StageHistory`.

Attachments:

- Proposal and activity attachments use local filesystem storage for the current MVP.
- Attachment metadata lives in Prisma.
- Do not introduce external object storage unless a task explicitly asks for it.

## UI Architecture

The product is an operational CRM, not a marketing site.

Preserve the current direction:

- dashboard-first app flow
- authenticated app shell with left sidebar
- dense, practical layouts
- zinc background with white cards/tables
- restrained colors
- `lucide-react` icons
- horizontally scrollable tables on mobile where needed
- no decorative hero pages, blobs, or gradients

Primary navigation is in `lib/navigation/app-nav.ts`.

Important routes:

- `/` dashboard
- `/dashboard`
- `/follow-ups`
- `/opportunities`
- `/opportunities/new`
- `/opportunities/board`
- `/opportunities/[id]`
- `/companies`
- `/companies/new`
- `/companies/[id]`
- `/contacts`
- `/contacts/new`
- `/contacts/[id]`
- `/reports`
- `/settings/users`
- `/settings/stages`
- `/settings/project-types`
- `/settings/lost-reasons`

## Next.js 16 Rule

This repo explicitly warns that this is not the older Next.js API surface. Before changing framework-sensitive code, read the relevant guide in:

```text
node_modules/next/dist/docs/
```

This applies to App Router pages, route handlers, metadata, server actions, cookies, redirects, config, runtime behavior, build output, and caching.

## Change Recipes

### Add Or Change A Page

1. Read the nearest existing `app/**/page.tsx`.
2. Use `readCurrentUser()` plus redirect for protected pages.
3. Fetch data through services where available.
4. Keep interactive parts in focused client components.
5. Add or update source-level UI tests or Playwright tests when behavior changes.

### Add Or Change An API

1. Add or update validation in `lib/validation/**`.
2. Put business behavior in `lib/services/**`.
3. Keep route handlers thin: auth, parse, call service, return response.
4. Add service tests and route tests where appropriate.

### Add Or Change A Business Rule

1. Find the owning service or `lib/domain/rules.ts`.
2. Add focused tests before or alongside the change.
3. Check all API routes and pages that consume the affected service.

### Add Or Change Schema

1. Edit `prisma/schema.prisma`.
2. Create a migration.
3. Run Prisma generate.
4. Update seed data if needed.
5. Update validation, services, UI, and tests together.

### Add Or Change UI Copy Or Labels

1. Prefer existing label helpers in `lib/display/**`.
2. Keep UI concise and operational.
3. Thai labels are allowed where the existing app already supports them.
4. Verify screenshots if glyph rendering or layout fit is a risk.

## Testing Strategy

Run focused tests while developing, then the full verification set before reporting completion.

Focused examples:

```bash
npm test -- tests/services/activity-service.test.ts
npm test -- tests/services/stage-transition-service.test.ts
npm test -- tests/validation.test.ts
CI=1 npx playwright test tests/e2e/auth-and-routes.spec.ts
```

Full completion check:

```bash
npm test
npm run build
npm run lint
```

Known benign warning:

```text
MODULE_TYPELESS_PACKAGE_JSON for tailwind.config.ts during build
```

## Deployment Shape

For small companies, deploy as a standard Node.js application or use a platform like Vercel, Render, or a basic VPS.
The database is a local SQLite file (`dev.db`).

- Ensure the database file path is accessible and writable in the production environment.
- Use `npm run build` followed by `npm start` to run the production server.

## Agent Completion Checklist

Before handing work back:

- The change is scoped to the task.
- API routes return clear validation errors.
- Stage and follow-up side effects are handled by services.
- Prisma schema, migrations, seed, validation, services, UI, and tests are aligned when data changes.
- Focused tests were run for touched behavior.
- `npm test`, `npm run build`, and `npm run lint` were run or the blocker is clearly reported.
