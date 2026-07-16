# Opportunity Pipeline Monitor

An internal pre-sales CRM-style web application for tracking sales opportunities, pipeline health, companies, contacts, activities, and follow-ups.

## Tech Stack
- **Framework**: Next.js App Router (TypeScript, React)
- **Database & ORM**: SQLite, Prisma
- **Styling**: Tailwind CSS, `lucide-react` icons
- **Validation & Testing**: Zod, Vitest (Unit/Service), Playwright (E2E)

## Data Model & Stages

```text
User (Sales Rep)
Company -> Contact
Company -> Opportunity
           -> OpportunityContact (with a primary contact)
           -> Activity (Customer-contact activities update lastContactAt)
           -> Proposal (and ProposalAttachment)
           -> FollowUpTask (granular tasks assigned to users)
           -> StageHistory (logs stage transitions)
```

### Pipeline Stages
- `New Lead` -> `Qualification` -> `Initial Presentation` -> `Requirement Gathering` -> `Proposal Preparing` -> `Quotation Sent` -> `Follow-up / Negotiation` -> `Verbal Confirm / Waiting PO` -> `Won`
- Alternative close states: `Lost` (requires reason), `Hold` (requires reason, optional revive date)

### Opportunity Dimensions
- **Stage**: (Listed above)
- **Temperature**: `Hot`, `Warm`, `Cold`, `Dormant`
- **Health**: `Good`, `Warning`, `Risk`, `Blocked`, `Closed` (Derived from stage, last contact, next follow-up)

## Core Business Rules
- **Authentication**: No login required. Anyone has full access. `User` represents a Sales Rep chosen as the owner.
- **Follow-up**: Active queue is based on `nextFollowUpAt` (Won/Lost are excluded; Hold appears when revive date is due).
- **Activities**: Only customer-facing activities update `lastContactAt` (Internal notes/meetings do not).
- **Stage Transitions**:
  - `Quotation Sent`: Requires amount, sent date, next follow-up. Automatically creates a Proposal.
  - `Won`: Requires final amount, expected project start date, PO status, main contact, handover note.
  - `Lost`: Requires lost reason.
  - `Hold`: Requires hold reason.
  - All transitions must write a `StageHistory` log.

## App Routes
- `/` or `/dashboard`: Action-first dashboard (open opportunities, due today, overdue, pipeline value).
- `/follow-ups`: Follow-up queues.
- `/opportunities`: Table/list, `/opportunities/new`, `/opportunities/board` (Kanban), `/opportunities/[id]`, `/opportunities/[id]/edit`.
- `/companies`: List, `/companies/new`, `/companies/[id]`, `/companies/[id]/edit`.
- `/contacts`: List, `/contacts/new`, `/contacts/[id]`, `/contacts/[id]/edit`.
- `/reports`: Pipeline reports.
- `/settings/users`: Manage Sales Reps.
- `/settings/[stages | project-types | lost-reasons]`: Read-only configurations.

## Development & Environment

### Environment Variables (`.env`)
```text
DATABASE_URL="file:./dev.db"
AUTH_SECRET=replace-with-at-least-32-characters
ATTACHMENT_STORAGE_DIR=./storage/attachments
```

### Required Commands
```bash
# Setup & Database
npm run db:generate
npm run db:migrate
npm run db:seed     # Seeds demo users: admin, manager, owner, viewer

# Development
npm run dev
npm run build
npm start
npm run lint

# Testing & Verification
npm test
npm run test:e2e
```

### Verification Standard
Before declaring the build complete, ensure these pass:
```bash
npm test && npm run build && npm run lint && CI=1 npm run test:e2e
```
