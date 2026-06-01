# BatchBoard — Development Plan (8 Weeks)

Build BatchBoard: a multi-tenant coaching center management SaaS for Bangladesh.
Stack: Next.js 14 (App Router) frontend + Supabase (Postgres + Auth + Edge Functions + Storage).

---

## Overview

| Phase | Weeks | Focus |
|---|---|---|
| **Phase 1 — Foundation** | 1 | Supabase setup, schema, RLS (with `get_my_center_id()`), auth, env |
| **Phase 2 — Core Backend** | 2–3 | Edge Functions (two-client pattern): center, batches, students, sessions, attendance |
| **Phase 3 — Fees & SMS** | 4 | Fee generation (with `due_date`), per-center invoices, payment, SMS (rate-limited) |
| **Phase 4 — Frontend MVP** | 5 | Dashboard, batch/student/attendance/fees UI |
| **Phase 5 — Exam Module** | 6 | Exams, results, rank calculation, result card PDF |
| **Phase 6 — Polish & Deploy** | 7–8 | Reports, CI, RLS security tests, E2E tests, production deploy |

---

## Phase 1 — Foundation (Week 1)

### Goals
- Supabase project created and configured.
- All migrations applied to dev database.
- RLS enabled on **every table** (including `centers` and `profiles`).
- `get_my_center_id()` SECURITY DEFINER function deployed and tested.
- Auth flow working (signup, login, protected routes).
- Teacher invite post-signup hook deployed.
- Environment variable checklist complete.

### Tasks

**Supabase Setup**
- Create Supabase project; record `SUPABASE_URL`, `SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_ROLE_KEY`.
- Write `supabase/migrations/001_initial_schema.sql` — all tables:
  `centers` (with `invoice_prefix`, `sms_daily_cap`, `sms_opted_in`),
  `profiles`, `subjects`, `batches`, `students`, `enrollments`,
  `class_sessions`, `attendance`, `exams`, `results` (with denormalization comment),
  `fee_records` (with `due_date`, `session_id`), `sms_logs` (with `student_id`).
- Write `supabase/migrations/002_rls_policies.sql`:
  - `get_my_center_id()` — SECURITY DEFINER function (avoids circular reference on `profiles`).
  - RLS policies for ALL tables including `centers` and `profiles`.
  - `profiles_insert_own` policy (allows insert during onboarding).
- Write `supabase/migrations/003_invoice_trigger.sql`:
  - `assign_invoice_number()` trigger using `centers.invoice_prefix` (not hardcoded).
- Write `supabase/migrations/004_auth_trigger.sql`:
  - `handle_new_user()` trigger on `auth.users` — creates `profiles` row from invite metadata (`center_id`, `full_name`, `role`, `subjects`).
- Write `supabase/migrations/005_sms_helpers.sql`:
  - `decrement_sms_balance()` — atomic balance decrement RPC.
- Write `supabase/migrations/006_indexes.sql` — indexes on `center_id`, `batch_id`, `student_id`, `class_date`, `due_date`, `month_year`.
- Write `supabase/seed.sql` with one sample center (with `invoice_prefix`), 2 batches, 5 students, test profiles.
- Apply migrations and seed to dev Supabase instance.

**RLS Validation (Phase 1 Smoke Test)**
- After applying migrations, test with two separate auth users in different centers:
  - User A should see only Center A data.
  - User A should NOT be able to read Center B's `centers` row.
  - User A should NOT be able to read Center B's students, batches, etc.
  - Verify `get_my_center_id()` returns correct center for each user.
- This is a quick manual check; full automated RLS tests come in Phase 6.

**Next.js Scaffold**
- Init Next.js 14 app with TypeScript + Tailwind + shadcn/ui in `frontend/`.
- Configure `lib/supabase.ts` (browser client using `SUPABASE_ANON_KEY` + `@supabase/ssr`).
- Write `middleware.ts` (route protection: redirect unauthenticated users to `/login`).
- Create auth pages: `(auth)/login/page.tsx`, `(auth)/setup/page.tsx` (center onboarding).
- Write `lib/utils/phone.ts` — BD phone normalizer (`01XXXXXXXXX` → `+8801XXXXXXXXX`).
- Test Bengali Unicode: render "মিনহাজুল" in UI; verify DB round-trip.

**Edge Function Shared Code**
- Write `supabase/functions/_shared/clients.ts` — two-client pattern:
  - `createUserClient(authHeader)` — ANON_KEY + user JWT, RLS enforced.
  - `createAdminClient()` — SERVICE_ROLE_KEY, bypasses RLS.
- Document in code comments which operations use which client.

**Environment**
- Write `.env.example`, `frontend/.env.local.example`.
- Document all required vars: `SUPABASE_URL`, `SUPABASE_ANON_KEY` (frontend only), `SUPABASE_SERVICE_ROLE_KEY` (Edge Functions only — never in frontend bundle), SSL Wireless API key, app URL.

### Gate
- Migrations apply cleanly.
- Login → onboarding → dashboard redirect works.
- Bengali name stored and retrieved correctly.
- RLS smoke test passes: cross-center reads blocked on all tables including `centers` and `profiles`.
- Teacher invite → signup → profile auto-created with correct `center_id`.

---

## Phase 2 — Core Backend: Batches, Students, Sessions, Attendance (Weeks 2–3)

### Goals
- Edge Functions for center setup, batch CRUD, teacher management, student CRUD + enrollment, class session management, and bulk attendance.
- Frontend skeleton pages wired to live Supabase data.

### Week 2 Tasks

**Edge Functions**
- `center-setup/index.ts`:
  - POST `/setup` — create center (with `invoice_prefix`) + owner profile. Uses **admin client** (no profile exists yet, so RLS lookup fails). Single transaction.
  - POST `/invite-teacher` — calls `supabase.auth.admin.inviteUserByEmail()` with `data: { center_id, full_name, role: 'teacher', subjects }`. Uses **admin client** for auth admin API.
- `batch-manage/index.ts` — GET/POST/PATCH/DELETE: list, create, update, archive batches. Uses **user client** (RLS enforced).
- `student-manage/index.ts` — GET/POST/PATCH: list (filterable by batch), register, update student; POST enroll; PATCH drop enrollment. Uses **user client**.

**Frontend Pages**
- `(dashboard)/layout.tsx` — sidebar with nav: Dashboard, Batches, Students, Teachers, Attendance, Fees, Results, SMS, Reports.
- `(dashboard)/batches/page.tsx` — batch list with status badge, teacher name, student count.
- `(dashboard)/batches/[id]/page.tsx` — batch detail: enrolled students, session log, schedule.
- `(dashboard)/students/page.tsx` — student list with search, filter by batch.
- `(dashboard)/teachers/page.tsx` — teacher list, invite button (calls `center-setup/invite-teacher`), batch assignment display.

### Week 3 Tasks

**Edge Functions**
- `session-manage/index.ts` — POST: create class session (date, topic, batch); GET: list sessions for batch. Uses **user client**.
- `attendance-bulk/index.ts`:
  - POST: bulk mark attendance array `[{student_id, status}]`.
  - Creates session if not exists.
  - Returns list of absent student IDs for SMS trigger.
  - Uses **user client** for attendance writes.
  - For SMS trigger: calls `sms-send` internally, which uses **admin client** for `sms_balance` decrement.

**Frontend Pages**
- `(dashboard)/attendance/page.tsx` — teacher view: select batch → select date → mark grid.
- `components/attendance/AttendanceGrid.tsx` — offline-capable grid:
  - Load existing attendance from Supabase on mount.
  - Store changes in `localStorage` queue.
  - Submit bulk to `attendance-bulk` Edge Function.
  - Retry failed submits on reconnect.
  - Show sync status indicator.

### Gate
- Teacher can mark full class attendance; absent students appear in API response.
- Data persists in DB with correct `center_id`.
- All Edge Functions use user client by default; admin client only where documented.
- Cross-center isolation holds: User A's teacher cannot mark attendance in User B's batches.

---

## Phase 3 — Fees & SMS (Week 4)

### Goals
- Fee records auto-generated for a batch+month (or per session), with `due_date` set correctly.
- Per-class fees respect enrollment date (no billing for pre-enrollment sessions).
- Invoice numbers assigned automatically via DB trigger with per-center prefix.
- Payment marking with receipt display.
- SMS absence alert working end-to-end with rate limiting.

### Tasks

**Edge Functions**
- `fee-generate/index.ts` (uses **user client**):
  - For `monthly` batches: generate one `fee_record` per enrolled student for given `month_year`. Set `due_date = first day of next month`.
  - For `per_class` batches: generate one `fee_record` per enrolled student per `session_id` in the month. Set `due_date = session_date + 7 days`. **Filter: only sessions where `session_date >= enrollment.enrolled_at`**.
  - Skip if record already exists (idempotent — `ON CONFLICT DO NOTHING`).
- `fee-pay/index.ts` (uses **user client**) — PATCH `fee_records/:id`: set `status = 'paid'`, `paid_at`, `payment_method`, `received_by`. Invoice number is already assigned by DB trigger on insert.
- `fee-overdue/index.ts` (uses **user client**) — GET: all `fee_records` where `status = 'due'` **AND `due_date < current_date`**, grouped by student. Works uniformly for both monthly and per-class — no branching needed.
- `sms-send/index.ts` (uses **admin client** for `sms_balance` decrement + `sms_logs` insert):
  - Accepts `{type, recipients: [{phone, student_id, student_name, ...contextData}]}`.
  - **Rate limiting checks:**
    1. `recipients.length <= 20` (per-call cap) — reject with 400 if exceeded.
    2. Count today's `sms_logs` for center. If `count + recipients.length > sms_daily_cap` — reject with 429.
    3. `centers.sms_balance >= recipients.length` — reject with 402 if insufficient.
    4. `centers.sms_opted_in = true` — reject with 403 if SMS not opted in.
  - For each recipient: call `decrement_sms_balance()` RPC (atomic, returns false if balance hits zero). Build message template per type. Call SSL Wireless API with E.164 phone. Insert `sms_logs` row with `student_id`.
  - Returns `{sent: N, failed: N, results: [...]}`.
- `sms-logs/index.ts` (uses **user client**) — GET: paginated SMS history for center. Filterable by `student_id` (enabled by `sms_logs.student_id` column).

**Frontend Pages**
- `(dashboard)/fees/page.tsx`:
  - Tab 1: Generate fees (select batch + month → generate button).
  - Tab 2: Fee list (filter by batch/month/status) with Pay button.
  - Tab 3: Overdue view (fees where `due_date` has passed and `status = 'due'`).
- `components/fees/InvoiceModal.tsx` — printable A5 receipt with invoice number (shows center prefix, e.g., "ABC-0042"), student name, batch, amount, date, payment method, center name.
- `(dashboard)/sms/page.tsx`:
  - SMS credit balance + daily cap usage display.
  - Compose tab: select audience (batch / individual) + message type + send.
  - Log tab: history with status badges, filterable by student.

**Trigger SMS from Attendance:**
- `attendance-bulk` function: after saving, for each absent student with a `guardian_phone`, call `sms-send` inline (checks `sms_opted_in`, `sms_balance`, and daily cap).

### Gate
- Generate fees for a batch → pay one → invoice number with center prefix appears on receipt.
- Generate per-class fees → verify student enrolled after 3rd session only gets billed for sessions 4+.
- Mark student absent → guardian SMS delivered (or stubbed with log entry + `student_id`).
- Attempt >20 SMS in one call → rejected with 400.
- Attempt SMS with 0 balance → rejected with 402.
- Attempt SMS beyond daily cap → rejected with 429.

---

## Phase 4 — Frontend MVP Polish (Week 5)

### Goals
- All MVP screens complete and connected to live data.
- Dashboard stats meaningful.
- Mobile-optimized attendance view.

### Tasks

**Dashboard** (`(dashboard)/dashboard/page.tsx`)
- Stats cards: Total Students, Active Batches, Fees Collected This Month, Overdue Fees (by `due_date`), SMS Credits Remaining / Daily Cap Usage.
- Quick actions: Mark Today's Attendance, Generate Monthly Fees.
- Recent activity feed (last 5 payments, last SMS sent).

**Student Profile** (`(dashboard)/students/[id]/page.tsx`)
- Personal info + edit form.
- Enrolled batches list.
- Attendance history: monthly % per batch, last 30 days calendar heatmap.
- Fee history: all invoices with status badges, Pay button for due fees.
- SMS history for this student (via `sms_logs.student_id` filter).

**Batch Detail** (`(dashboard)/batches/[id]/page.tsx`)
- Edit batch info.
- Enrolled students list with attendance % column.
- Session log: list of all class sessions with date + topic.
- Fee summary: collection rate for current month.

**Reports** (`(dashboard)/reports/page.tsx`)
- Monthly Fee Collection PDF: calls `report-monthly` Edge Function.
- Attendance Sheet PDF: calls `report-attendance` Edge Function.

**Edge Functions (PDF)** — use **user client** (RLS filters data to current center)
- `report-monthly/index.ts`: query `fee_records` for batch+month → generate PDF table with jsPDF (center name header, student list, amount, status, invoice number with prefix). Embed SolaimanLipi for Bengali names.
- `report-attendance/index.ts`: query `attendance` for batch+month → generate grid PDF (students as rows, dates as columns, P/A/L cells).

### Gate
- End-to-end flow: onboarding → batch → student → session → attendance → SMS → fee → pay → PDF download.
- All screens render correctly on mobile.
- Student profile shows SMS history.
- Dashboard overdue count matches `fee-overdue` API.

---

## Phase 5 — Exam Module (Week 6)

### Goals
- Exams can be created per batch.
- Results entered in bulk.
- Ranks auto-calculated and stored.
- Student result card PDF generated.

### Tasks

**Edge Functions** — use **user client**
- `exam-manage/index.ts`:
  - POST `/exams` — create exam (name, date, total_marks, batch_id).
  - GET `/exams?batch_id` — list exams for batch.
  - POST `/results/bulk` — bulk insert `[{student_id, obtained_marks}]` for an exam; calculate and store `rank` (order by `obtained_marks DESC`, assign sequential rank). **Ensure `results.batch_id` matches `exams.batch_id`** (enforce consistency for the denormalized column).
  - GET `/results?exam_id` — results with rank, student name.
- `report-result-card/index.ts` — GET `?student_id`: all exam results for student → PDF result card (student name, photo if available, exam list, marks, rank, batch name).

**Frontend Pages**
- `(dashboard)/results/page.tsx`:
  - Select batch → select or create exam.
  - Bulk result entry grid: student name + marks input per row.
  - Leaderboard view: ranked list with marks, rank badge, percentage.
- Add result card PDF download button to `(dashboard)/students/[id]/page.tsx`.

### Gate
- Create exam → enter results for all students → ranks calculated correctly.
- `results.batch_id` always equals `exams.batch_id` — no drift.
- Result card PDF downloads with correct student data.

---

## Phase 6 — Polish, Security & Deploy (Weeks 7–8)

### Goals
- Production-ready: CI, RLS validation, E2E tests, security hardening.
- Deployed to Vercel + Supabase production.
- Runbook written.

### Week 7 Tasks

**Security & RLS Validation**
- Write automated RLS test script: apply migrations to Supabase local emulator; create two centers with separate users; for **every table** (including `centers` and `profiles`):
  - Assert User A can read their own data.
  - Assert User A **cannot** read User B's data (SELECT returns 0 rows, not an error).
  - Assert User A **cannot** insert/update/delete User B's data.
- Verify `get_my_center_id()` correctly returns NULL for users without a profile (no accidental data leakage via NULL center_id match).
- Verify no `SUPABASE_SERVICE_ROLE_KEY` in frontend bundle (grep Next.js build output).
- Verify Edge Functions using admin client are limited to: `center-setup`, `sms-send`, `attendance-bulk` (SMS trigger only).
- SMS rate limit tests:
  - Assert `sms-send` returns 402 if `sms_balance = 0`.
  - Assert `sms-send` returns 429 if daily cap reached.
  - Assert `sms-send` returns 400 if `recipients.length > 20`.
  - Assert `sms-send` returns 403 if `sms_opted_in = false`.
- Invoice trigger test: concurrent inserts to `fee_records` for the same center → assert no duplicate invoice numbers.

**Testing**
- Unit tests: `phone.ts` normalizer (edge cases: missing +88, 11-digit, 10-digit).
- Unit tests: `due_date` calculation (monthly: first of next month; per-class: session + 7 days).
- Unit tests: enrollment date filter (per-class fee generation skips pre-enrollment sessions).
- E2E (Playwright): smoke flows:
  1. Onboarding → set invoice prefix → create batch → enroll student.
  2. Owner invites teacher → teacher signs up → profile auto-created with correct center_id.
  3. Create session → mark attendance → SMS stub triggered → `sms_logs` has `student_id`.
  4. Generate monthly fees → verify `due_date` → pay fee → invoice shows center prefix.
  5. Generate per-class fees → verify enrollment filter → verify `due_date`.
  6. Create exam → enter results → verify ranks → verify `results.batch_id` consistency.
  7. Download monthly collection PDF.

**CI (GitHub Actions)**
- `lint-typecheck.yml`: ESLint + TypeScript check on PR.
- `test.yml`: Run unit tests + E2E against Supabase local emulator.
- `deploy-preview.yml`: Vercel preview URL on PR.

### Week 8 Tasks

**Performance Review**
- Add DB indexes for all common query patterns (verify with `EXPLAIN ANALYZE`).
- Ensure `get_my_center_id()` has minimal overhead (STABLE + indexed `profiles.id` PK lookup).
- Add `content-visibility: auto` to long student/fee lists.
- Verify Supabase Edge Function cold start times are acceptable (<500ms).

**Production Deploy**
- Apply all migrations to production Supabase instance.
- Set all env vars in Vercel production environment.
- **Verify `SUPABASE_SERVICE_ROLE_KEY` is NOT in any `NEXT_PUBLIC_` env var.**
- Deploy Edge Functions: `supabase functions deploy --project-ref <ref>`.
- Configure Supabase Storage bucket policies for student photos.
- Sanity check: full smoke flow on production.

**Runbook**
- How to top up SSL Wireless SMS credits.
- How to adjust `sms_daily_cap` per center.
- How to apply future DB migrations.
- How to add a new center admin (manual profile insert if needed).
- How the teacher invite flow works (metadata → auth trigger → profile).
- How invoice prefixes work and how to change one (update `centers.invoice_prefix`).
- Backup policy (Supabase automated backups; point-in-time recovery).

### Gate
- All E2E tests pass on staging.
- Automated RLS tests pass for all tables (including `centers`, `profiles`).
- SMS rate limit tests pass.
- CI green.
- Production URL live.

---

## Phase Dependencies

```
Phase 1 (Schema + RLS + Auth + get_my_center_id() + auth trigger)
  └── Phase 2 (Edge Functions: two-client pattern, batches, students, attendance)
        └── Phase 3 (Fees with due_date + enrollment filter, SMS with rate limiting)
              └── Phase 4 (Frontend MVP)
                    ├── Phase 5 (Exam Module)
                    └── Phase 6 (Polish + Deploy)
```

- SMS credentials can be stubbed (log-only) until Phase 3 integration testing.
- PDF generation (jsPDF + SolaimanLipi) should be tested with Bengali names in Phase 1 separately.
- Phases 5 and 6 can partially overlap (deploy infra can be set up during Week 6).

---

## Key Deliverables per Phase

| Phase | Deliverable |
|---|---|
| 1 | Migrations applied, `get_my_center_id()` working, RLS on ALL tables (including `centers`/`profiles`), auth trigger for teacher invite, Bengali test passing |
| 2 | Batch/student/attendance Edge Functions (user client by default), skeleton frontend pages |
| 3 | Fee generation with `due_date` + enrollment filter, per-center invoice prefix, payment flow, rate-limited SMS end-to-end, `sms_logs.student_id` populated |
| 4 | Complete MVP frontend, dashboard with overdue-by-due-date, PDFs, mobile attendance view |
| 5 | Exam + result module, rank calculation, `results.batch_id` consistency enforced, result card PDF |
| 6 | CI green, automated RLS tests (all tables), SMS rate limit tests, E2E smoke, production deployed, runbook |

---

## Risks & Mitigations

| Risk | Mitigation |
|---|---|
| Missing SSL Wireless credentials | Stub `sms-send` with `console.log` + `sms_logs` insert (including `student_id`); real key plugged in Phase 3 |
| RLS misconfiguration leaking cross-center data | `get_my_center_id()` SECURITY DEFINER avoids circular ref; RLS on `centers` and `profiles` explicitly; automated test in Phase 6 covers all tables |
| Service role key accidentally bypasses RLS | Two-client pattern enforced; code review checklist: every Edge Function must justify admin client usage. grep audit in Phase 6 |
| Bengali font in jsPDF | Test with SolaimanLipi in Phase 1; don't defer to PDF phase |
| Edge Function cold start latency | Keep functions lean; `get_my_center_id()` is STABLE (cacheable within transaction); use Supabase connection pooling (pgBouncer mode) |
| Per-class fee billing complexity | `due_date` column unifies overdue logic; enrollment date filter in `fee-generate` prevents pre-enrollment billing; tested in Phase 3 |
| Invoice number gaps after rollbacks | Acceptable (trigger on INSERT only); document in runbook |
| Invoice prefix drift (center changes prefix after invoices issued) | Document: changing `invoice_prefix` only affects future invoices; existing invoice numbers are immutable |
| Offline attendance sync conflicts | Use `unique(batch_id, student_id, class_date)` constraint; upsert on retry |
| SMS loop bug draining credits | Per-call cap (20), daily cap (`sms_daily_cap`), atomic `decrement_sms_balance()` that returns false at zero |
| Teacher invite profile not created | `handle_new_user()` DB trigger on `auth.users` INSERT ensures profile is created from metadata; tested in Phase 1 |
| `results.batch_id` drifts from `exams.batch_id` | Application code sets `batch_id` from the exam on insert; add CHECK constraint or trigger if drift is observed |