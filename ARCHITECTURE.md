# Architecture Map & AI Guide

This document is a practical reference map for AI coder agents working on the Opportunity Pipeline Monitor. It explains key boundaries, file responsibilities, and change recipes.

## High-Level Architecture
```text
Browser -> Next.js Pages & Client Components (UI)
        -> Next.js API Routes (app/api/)
        -> Business Services (lib/services/)
        -> Prisma Client & SQLite (dev.db)
```
The app has no authentication system. All operations are accessible directly, with Sales Reps represented by the `User` model.

---

## Directory Structure
- `app/`: Next.js App Router folders representing views (dashboard, opportunities, companies, contacts, follow-ups, settings).
- `components/`: UI components matching the app routes, including shared primitives under `ui/` and shell layout under `layout/`.
- `lib/`: Core logic:
  - `services/`: Primary business services and DB operations.
  - `domain/`: Business enums (`enums.ts`) and validation rule helpers (`rules.ts`).
  - `validation/`: Zod schemas for input validation.
  - `display/` & `navigation/`: Formatting and sidebar configuration.
- `prisma/`: Prisma schema (`schema.prisma`), migrations, and seed scripts.
- `tests/`: Organized into `services/` (Vitest unit/integration) and `e2e/` (Playwright tests).

---

## Service Responsibilities (`lib/services/`)
Put business rules in these services rather than API routes or UI components:
- `opportunity-service.ts`: Opportunity CRUD, filtering, and CSV export.
- `activity-service.ts`: Tracks timeline and handles customer contact updates.
- `stage-transition-service.ts`: Handles business logic for stage transitions (Won, Lost, Hold, Quotation Sent).
- `follow-up-service.ts`: Manages the follow-up tasks and opportunity queue.
- `dashboard-service.ts`: Gathers metrics, due follow-ups, and risk cards.
- `user-service.ts`: Handles Sales Rep (User) CRUD.

---

## Database Schema (Prisma Models)
The primary models defined in `prisma/schema.prisma` are:
- `User`: Sales Rep owner.
- `Company`: Holds multiple contacts and opportunities.
- `Contact`: Details for leads/customers.
- `Opportunity`: The main pre-sales tracking unit (includes stage, temperature, health, owner, company, value).
- `OpportunityContact`: Mapping table to link multiple contacts to an opportunity (one marked as primary).
- `Activity` & `ActivityAttachment`: Actions/notes taken with customers.
- `Proposal` & `ProposalAttachment`: Documents and quotes sent.
- `FollowUpTask`: Actionable check-ins.
- `StageHistory`: Chronological logs of stage changes.

---

## Crucial Business Rules to Preserve
- **Follow-up Queues**: Sourced from `opportunities.nextFollowUpAt`. Closed opportunities (Won/Lost) are excluded. Hold opportunities appear when their revive date is due.
- **Last Contact updates**: Only customer-facing activities (external meetings, emails, etc.) update `opportunities.lastContactAt`. Internal meetings and notes must not update this field.
- **Stage Transitions**:
  - `Quotation Sent`: Requires quotation amount, sent date, and next follow-up. Automatically creates/updates the quotation proposal.
  - `Won`: Requires final amount, expected start date, PO status, main contact, and handover note.
  - `Lost` & `Hold`: Require reasons.
  - Every stage change must log a `StageHistory` record.

---

## Developer Recipes

### 1. Database Schema Changes
1. Edit `prisma/schema.prisma`.
2. Generate migration: `npm run db:migrate`.
3. Re-generate Prisma Client: `npm run db:generate`.
4. Update seeds in `prisma/seed.ts` if needed.

### 2. Creating or Modifying APIs
1. Define/update Zod schema in `lib/validation/`.
2. Implement or modify logic in `lib/services/`.
3. Keep the API route handler (`app/api/.../route.ts`) thin—only validate input, call the service, and return JSON.

### 3. Creating or Modifying Pages
1. Fetch initial data on the server component page (`app/**/page.tsx`).
2. Pass data down to client components (`"use client"`) for forms or interactive elements.
3. Keep style aligned with the CRM aesthetic: zinc/neutral colors, clean white panels, Lucide icons, and no decorative hero blobs or gradients.

---

## Verification & Testing
Always verify changes locally with the following commands before finalizing:
```bash
# Run unit and service-level tests
npm test

# Run E2E tests
CI=1 npm run test:e2e

# Verify production build and linter
npm run build && npm run lint
```
