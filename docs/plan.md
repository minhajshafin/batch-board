# BatchBoard — Waterfall Plan (6 weeks)

Build BatchBoard MVP: Next.js frontend (TypeScript), Express backend (JavaScript), Supabase Postgres with RLS. Deliver: onboarding/auth, batch & student CRUD, attendance grid (offline-capable), fee generation & payment, SMS absence trigger, monthly collection PDF.

## Phases & Timeline

### Week 1 — Requirements
- Confirm Supabase & SSL Wireless credentials.
- Produce env variable checklist and acceptance criteria.
- Deliverables: Requirements doc, env checklist, acceptance tests.
- Gate: Stakeholder sign-off.

### Week 2 — High-Level Design
- Finalize DB schema + RLS policies; draft `supabase/migrations/001_initial_schema.sql`.
- Produce OpenAPI spec and UI wireframes (login, setup, attendance, fees, reports).
- Deliverables: migration SQL (draft), API spec, wireframes.
- Gate: Design review passed.

### Weeks 3–4 — Backend Implementation
- Week 3:
  - Scaffold backend, implement `auth` middleware (Supabase JWT), center onboarding route.
  - Basic CRUD: centers, batches, students.
  - Apply migrations to dev Supabase.
- Week 4:
  - Implement `POST /attendance/bulk` with opt-in absence SMS trigger.
  - Implement fee endpoints (`POST /fees/generate`, `PATCH /fees/:id/pay`, `GET /fees/overdue`).
  - Implement `smsService` (stub + SSL Wireless adapter) and `pdfService` (jsPDF + SolaimanLipi).
- Deliverables: running backend on dev, seeded test DB, unit tests for core handlers.
- Gate: Backend integration tests green.

### Weeks 4–5 — Frontend Implementation
- Week 4 (parallel): Supabase client setup, login + onboarding pages, protected dashboard layout, basic batch/student pages.
- Week 5: `AttendanceGrid` component with local queue & sync, fees UI, SMS UI, reports download.
- Deliverables: authenticated frontend exercising backend endpoints.
- Gate: E2E smoke flows pass.

### Week 6 — Verification & Hardening
- Run full acceptance tests, add CI (lint/test/build), add Dockerfile (backend), security checks (no service keys in frontend).
- Deliverables: CI workflows, staging migration, e2e tests.
- Gate: All acceptance tests pass.

### End of Week 6 — Deploy & Handoff
- Deploy frontend (Vercel) and backend (Railway), sanity-check SMS and PDF generation.
- Deliverables: production URLs, runbook for SMS credits and Supabase roles.

## Phase Dependencies
- Flow: Requirements → Design → Backend → Frontend → Verification → Deployment.
- Supabase migrations must be applied before backend validation.
- SMS credentials can be stubbed until integration/testing.

## Actionable Files (priority)
- `supabase/migrations/001_initial_schema.sql`
- `backend/src/index.js`, `backend/src/middleware/auth.js`
- `backend/src/routes/{center,batches,students,attendance,fees,sms,reports}.js`
- `backend/src/services/{smsService,pdfService}.js`
- `frontend/lib/supabase.ts`, `frontend/app/(auth)/login/page.tsx`, `frontend/components/AttendanceGrid.tsx`
- `.env.example`, `frontend/.env.local.example`, `backend/.env.example`
- CI / deploy: `Dockerfile`, `vercel.json`, GitHub Actions workflows

## Verification & Tests
- Acceptance tests defined in Requirements; run in Week 6.
- Migration test: apply migrations to a test Supabase instance and validate RLS by simulating different `auth.uid()`.
- E2E smoke script: onboarding → batch → student → attendance → SMS stub → fees → PDF.

## Risks & Mitigations
- Missing credentials: use stubs and clear env templates.
- RLS misconfiguration: add migration tests and integration checks.
- Bengali font PDF issues: include SolaimanLipi early and test with Bengali names.