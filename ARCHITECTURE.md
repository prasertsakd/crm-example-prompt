# Architecture

A developer guide for AI coder agents working on the Opportunity Pipeline Monitor.

## System Design
- **Frontend & Routing**: Next.js App Router (pages in `app/`, components in `components/`).
- **Database**: SQLite with Prisma (`prisma/schema.prisma`).
- **Business Logic**: Stored in `lib/services/` (not in UI components or route handlers).
- **Validation**: Request/form schemas in `lib/validation/`.
- **Testing**: Service/unit tests in `tests/services/`, E2E tests in `tests/e2e/`.

## Directory Structure
- `app/`: Next.js App Router routes (Dashboard, Opportunities, Companies, Contacts, Follow-ups, Settings).
- `components/`: UI components matching app sections, and shared `ui/` elements.
- `lib/`: Domain enums (`domain/`), database client (`db.ts`), UI/domain helpers (`display/`, `navigation/`), and core services (`services/`).
- `prisma/`: Database schema, migrations, and seed data.
- `tests/`: Vitest (unit/services) and Playwright (E2E) test files.

## Crucial Business Rules
1. **Authentication**: No authentication. Anyone can access all pages and mutate all data.
2. **Follow-ups**: Active queue sorted by `opportunities.nextFollowUpAt` (excluding Won/Lost; Hold shown when revive date is due).
3. **Activities**: Customer-facing activities must update `lastContactAt` (internal notes/meetings must not).
4. **Stage Transitions**: Handled via `stage-transition-service.ts`:
   - `Quotation Sent`: Requires amount, sent date, and next follow-up. Creates/updates Proposal.
   - `Won`: Requires amount, project start date, PO status, main contact, handover note.
   - `Lost` / `Hold`: Requires a reason.
   - Every transition writes a `StageHistory` log row.

## Developer Workflows
- **Database Change**: Edit `prisma/schema.prisma` -> Run `npm run db:migrate` -> Run `npm run db:generate`.
- **Adding/Changing API**: Define validation schema (`lib/validation/`) -> Write business logic in a service (`lib/services/`) -> Create a thin route handler (`app/api/**/route.ts`).
- **Adding/Changing UI**: Fetch data on server page (`app/**/page.tsx`) -> Delegate interactive features to client components (`components/`).

## Verification Checklist
Before completing any task, ensure these commands pass:
```bash
# Services & unit tests
npm test

# E2E Playwright tests
CI=1 npm run test:e2e

# Production build & lint
npm run build && npm run lint
```
