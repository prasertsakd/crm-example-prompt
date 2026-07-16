# Opportunity Pipeline Monitor - Build Brief For AI Coder Agents

This document is the master prompt and product brief for building the app from an empty folder. Use it together with `ARCHITECTURE.md`. The two files are intended to be enough for a new AI coder agent to recreate the project without reading any prior repository history.

## Master Agent Prompt

```text
You are an AI coder agent building Opportunity Pipeline Monitor from scratch.

Create a production-quality internal CRM-style web application for pre-sales opportunity tracking. The app must help a sales/engineering team track companies, contacts, opportunities, activities, proposals/quotations, follow-up queues, pipeline health, and won/lost/hold transitions.

Use:
- Next.js App Router with TypeScript
- React
- SQLite
- Prisma
- Tailwind CSS
- lucide-react icons
- Zod validation
- Vitest for unit/service tests
- Playwright for E2E tests

Important:
- If using a modern or unfamiliar Next.js version, read the local Next.js docs before writing framework-sensitive code.
- Build the actual app as the first screen after login. Do not create a marketing landing page.
- Keep business rules in services, not in UI components.
- Write tests for services, validation, and core E2E routes.
- Before claiming completion, run npm test, npm run build, and npm run lint.
```

## Product Goal

Build an internal opportunity pipeline monitor for pre-sales work. The system tracks prospective work before project execution begins.

The app must answer these questions quickly:

- What must be followed up today?
- Which opportunities are overdue or at risk?
- Where is each opportunity in the sales pipeline?
- Which opportunities have gone quiet?
- What has the team updated recently?
- Which proposal, quotation, PO, or handover action is next?

This is not a full project management system. The scope is pre-sales, proposal/quotation, follow-up, close status, and handover preparation after winning.

## Users (Sales Reps)

The system has no authentication or login. Anyone can access all pages and mutate any data. The `User` model represents a Sales Rep in the system. When creating or editing opportunities, the user selects a Sales Rep from a list to assign as the owner.

## Core Data Model

```text
User
Company
  -> Contact
  -> Opportunity
       -> OpportunityContact
       -> Activity
       -> Proposal
       -> FollowUpTask
       -> StageHistory
       -> ProposalAttachment
       -> ActivityAttachment
```

Minimum model requirements:

- Users represent Sales Reps (can be added/edited in settings).
- Companies can have many contacts and opportunities.
- Contacts belong to one company.
- Opportunities belong to one company and one owner user.
- Opportunities may link many contacts, with one primary contact.
- Activities belong to an opportunity and company, optionally to a contact.
- Proposals belong to an opportunity and may have attachments.
- Follow-up tasks belong to opportunities and are assigned to users.
- Stage history records every stage transition.

## Pipeline Stages

Use these stages exactly as user-facing labels:

```text
New Lead
Qualification
Initial Presentation
Requirement Gathering
Proposal Preparing
Quotation Sent
Follow-up / Negotiation
Verbal Confirm / Waiting PO
Won
Lost
Hold
```

Primary flow:

```text
New Lead
-> Qualification
-> Initial Presentation
-> Requirement Gathering
-> Proposal Preparing
-> Quotation Sent
-> Follow-up / Negotiation
-> Verbal Confirm / Waiting PO
-> Won
```

Alternative close flow:

```text
Quotation Sent / Follow-up / Negotiation / Waiting PO
-> Lost or Hold
```

## Opportunity Dimensions

Each opportunity needs three status dimensions:

- `stage`: sales process position.
- `temperature`: `Hot`, `Warm`, `Cold`, `Dormant`.
- `health`: `Good`, `Warning`, `Risk`, `Blocked`, `Closed`.

Health should be derived or updated from stage, last contact, next follow-up, and close status. Closed opportunities should use `Closed`.

## Required Business Rules

Follow-up:

- Main follow-up queue is based on `opportunities.nextFollowUpAt`.
- Won and Lost opportunities do not appear in active follow-up queues.
- Hold opportunities can appear when revive date is due.
- Follow-up tasks are granular assigned task rows and do not replace the main opportunity follow-up date.

Activities:

- Customer-contact activities update `opportunities.lastContactAt`.
- Internal notes and internal meetings must not update `lastContactAt`.
- `activityType` and `contactType` are separate fields.

Stage transitions:

- Use dedicated service functions and route endpoints for stage changes.
- Quotation Sent requires quotation amount, sent date, and next follow-up date.
- Quotation Sent creates or updates a quotation proposal with status `Sent`.
- Won requires final amount, expected project start date, PO status, main contact, and handover note.
- Lost requires lost reason.
- Hold requires hold reason and may include revive date and next action.
- Every stage transition writes a `StageHistory` row.

Authorization:

- No authentication is required. All users have full access to view and mutate all records.

## Required App Routes

Main app:

- `/`: action-first dashboard (default route)
- `/dashboard`: action-first dashboard
- `/follow-ups`: follow-up queues
- `/opportunities`: opportunity table/list
- `/opportunities/new`: create opportunity
- `/opportunities/board`: pipeline board by stage
- `/opportunities/[id]`: opportunity detail with timeline, proposals, actions
- `/opportunities/[id]/edit`: edit opportunity
- `/companies`: company list
- `/companies/new`: create company
- `/companies/[id]`: company detail
- `/companies/[id]/edit`: edit company
- `/contacts`: contact list
- `/contacts/new`: create contact
- `/contacts/[id]`: contact detail
- `/contacts/[id]/edit`: edit contact
- `/reports`: pipeline reports
- `/settings/users`: manage sales reps (users)
- `/settings/stages`: read-only stage settings
- `/settings/project-types`: read-only project type settings
- `/settings/lost-reasons`: read-only lost reason settings

Core APIs:

- CRUD for companies, contacts, opportunities, users.
- Activity create/update/delete and opportunity timeline.
- Proposal create/update/delete and attachments.
- Stage transition endpoints: change stage, mark won, mark lost, mark hold.
- Follow-up list, update, complete.
- Dashboard summary, follow-up today, risk opportunities, pipeline summary, recent activities.
- CSV export for opportunities, restricted by role.

## Dashboard Requirements

The dashboard is the first authenticated screen. It should be action-first, not decorative.

Include:

- KPI cards: open opportunities, due today, overdue, pipeline value.
- Follow-up today panel.
- Risk opportunities panel.
- Pipeline summary by stage.
- Recent activity feed.
- Quick links to opportunities, board, follow-ups, companies, contacts, reports.

## Opportunity Detail Requirements

Opportunity detail is the most important workflow screen.

Include:

- Compact deal summary.
- Company, owner, value, probability, next action, next follow-up, last contact.
- Primary actions for stage changes, quote sent, won, lost, hold.
- Activity timeline with customer-contact indicators.
- Proposal/quotation list and attachment links.
- Follow-up tasks.
- Related contacts with primary contact indicator.
- Stage history.

## UI Direction

Make the app feel like a practical operations CRM:

- Dense but readable.
- Sidebar app shell.
- Restrained palette.
- Zinc/neutral background.
- White panels, tables, and form surfaces.
- Cards should be simple and not overly rounded.
- Use `lucide-react` icons.
- Tables may scroll horizontally on mobile.
- Avoid marketing heroes, decorative gradients, blobs, and promotional copy.
- Avoid visible instructional text that explains obvious UI behavior.

## Suggested Implementation Order

1. Scaffold Next.js TypeScript app with Tailwind, linting, Vitest, Playwright.
2. Add Prisma, SQLite config, and environment files.
3. Implement schema, migrations, and seed users/data.
4. Implement shared app shell and sidebar navigation.
5. Implement validation schemas and domain enums/rules.
6. Implement services for companies, contacts, opportunities, activities, stage transitions, follow-ups, dashboard, reports, users, attachments.
7. Implement API routes as thin auth/validation/service wrappers.
8. Implement main pages and client form components.
9. Add service tests, validation tests, and E2E route/workflow tests.
10. Run full verification and fix failures.

## Environment

Use these environment variables:

```text
DATABASE_URL="file:./dev.db"
AUTH_SECRET=replace-with-at-least-32-characters
ATTACHMENT_STORAGE_DIR=./storage/attachments
```

Demo users to seed:

```text
admin / Admin12345!
manager / Manager12345!
owner / Owner12345!
viewer / Viewer12345!
```

## Required Commands

The final project should support:

```bash
npm run dev
npm run build
npm run start
npm run lint
npm test
npm run test:e2e
npm run db:generate
npm run db:migrate
npm run db:seed
```

## Verification Standard

Before reporting the project complete:

```bash
npm test
npm run build
npm run lint
CI=1 npm run test:e2e
```

At minimum, tests should cover:

- opportunity CRUD
- activity last-contact behavior
- stage transition requirements
- follow-up queue filtering
- dashboard service data
- user management safeguards
- attachment metadata/storage behavior

## Completion Definition

The build is complete when:

- The app starts at `/dashboard` (or `/`).
- Anyone can see all opportunities.
- Anyone can mutate all records.
- Company, contact, opportunity CRUD works.
- Opportunity detail supports activities, proposals, follow-ups, stage transitions, won/lost/hold.
- Dashboard, follow-ups, board, reports, settings routes render.
- App starts successfully with local SQLite.
- Full verification commands pass.
