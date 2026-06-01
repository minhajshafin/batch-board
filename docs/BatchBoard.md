Full operations platform for the thousands of coaching centers in BD (admission coaching, HSC, O-levels) that run entirely on paper and WhatsApp groups.

**The real problem:** BD has tens of thousands of coaching centers. They manage attendance, fees, results, and batch scheduling via notebooks and WhatsApp. Zero software penetration.

**Stack:** Next.js + Supabase (Postgres + Auth + Edge Functions + Storage) + SSL Wireless SMS + jsPDF

---

## Key Features

- Batch/class management (subject, teacher, schedule, capacity, fee cycle)
- Teacher management (profile, subject expertise, batch assignment, invite flow)
- Student enrollment + fee tracking (paid/due/overdue) with per-center invoice numbers
- Attendance marking (teacher-facing mobile view, offline-capable)
- Per-class session tracking for `per_class` fee billing (enrollment-date-aware)
- SMS alert to parents on absence or fee due (opt-in, prepaid credits, rate-limited)
- Exam management + result entry + rank calculation per batch
- Monthly fee collection report (PDF), attendance sheet (PDF), student result card (PDF)

---

## Core Architecture

**Frontend:** Next.js 14 (App Router) + TypeScript + Tailwind + shadcn/ui
**Backend:** Supabase Edge Functions (Deno)
**Database:** Supabase (Postgres with RLS)
**Auth:** Supabase Auth (JWT — built-in)
**SMS:** SSL Wireless (BD-local, cheapest per SMS, ~৳0.35/SMS)
**PDF:** jsPDF + jspdf-autotable + SolaimanLipi Bengali font
**File Storage:** Supabase Storage (student photos, documents)
**Deployment:** Vercel (frontend) + Supabase (Edge Functions, DB, Storage)

**Why Supabase Edge Functions over Express:**
Eliminates the need for a separate Railway-hosted backend. Edge Functions run on Deno globally close to users, share the same Supabase project, and access the DB natively. No separate infrastructure to maintain at MVP stage. Functions trigger SMS and generate PDFs without a persistent server.

**Why Supabase Auth over NextAuth:**
Multi-tenant RLS (Row Level Security) is native to Supabase. Each coaching center's data isolation becomes a Postgres policy, not application logic. This is the architecturally correct call.

---

## Database Schema

### Multi-Tenancy Model

Every table has `center_id`. Supabase RLS policies enforce isolation at DB level — no center can ever read another center's data even if an API call leaks.

```sql
-- TENANTS
create table centers (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  address text,
  phone text,
  logo_url text,
  invoice_prefix text not null default 'BB', -- per-center prefix for invoices e.g. "ABC"
  sms_balance integer default 0,             -- prepaid SMS credits
  sms_daily_cap integer default 100,         -- max SMS per center per day
  sms_opted_in boolean default false,        -- must be explicitly opted in
  invoice_counter integer default 0,         -- sequential invoice number per center
  created_at timestamptz default now()
);

-- USERS & ROLES
create table profiles (
  id uuid primary key references auth.users(id),
  center_id uuid references centers(id),
  full_name text not null,
  phone text,
  role text check (role in ('owner', 'admin', 'teacher')),
  subjects text[],                           -- subject expertise for teacher role
  is_active boolean default true,
  created_at timestamptz default now()
);

-- SUBJECTS
create table subjects (
  id uuid primary key default gen_random_uuid(),
  center_id uuid references centers(id),
  name text not null,                        -- "Physics", "English"
  code text                                  -- "PHY", "ENG"
);

-- BATCHES
create table batches (
  id uuid primary key default gen_random_uuid(),
  center_id uuid references centers(id),
  name text not null,                        -- "HSC-26 Morning"
  subject_id uuid references subjects(id),
  teacher_id uuid references profiles(id),
  schedule jsonb,                            -- {days: ["Sat","Mon"], time: "08:00"}
  capacity integer,
  fee_amount numeric(10,2),
  fee_cycle text check (fee_cycle in ('monthly', 'per_class')),
  status text default 'active' check (status in ('active', 'archived')),
  created_at timestamptz default now()
);

-- STUDENTS
create table students (
  id uuid primary key default gen_random_uuid(),
  center_id uuid references centers(id),
  full_name text not null,
  phone text,                                -- student's own phone
  guardian_phone text,                       -- for SMS alerts
  photo_url text,
  class_level text,                          -- "HSC", "SSC", "O-Level"
  created_at timestamptz default now()
);

-- ENROLLMENTS (student ↔ batch M:M)
create table enrollments (
  id uuid primary key default gen_random_uuid(),
  center_id uuid references centers(id),
  student_id uuid references students(id),
  batch_id uuid references batches(id),
  enrolled_at timestamptz default now(),
  status text default 'active' check (status in ('active', 'dropped')),
  unique(student_id, batch_id)
);

-- CLASS SESSIONS (tracks each physical class held — needed for per_class fee billing)
create table class_sessions (
  id uuid primary key default gen_random_uuid(),
  center_id uuid references centers(id),
  batch_id uuid references batches(id),
  session_date date not null,
  topic text,                                -- optional: "Chapter 3 - Kinematics"
  created_by uuid references profiles(id),
  created_at timestamptz default now(),
  unique(batch_id, session_date)
);

-- ATTENDANCE
create table attendance (
  id uuid primary key default gen_random_uuid(),
  center_id uuid references centers(id),
  batch_id uuid references batches(id),
  session_id uuid references class_sessions(id),
  student_id uuid references students(id),
  class_date date not null,
  status text check (status in ('present', 'absent', 'late')),
  marked_by uuid references profiles(id),
  created_at timestamptz default now(),
  unique(batch_id, student_id, class_date)
);

-- EXAMS
create table exams (
  id uuid primary key default gen_random_uuid(),
  center_id uuid references centers(id),
  batch_id uuid references batches(id),
  name text not null,                        -- "Monthly Test - April 2026"
  exam_date date,
  total_marks numeric(6,2) not null,
  created_at timestamptz default now()
);

-- RESULTS
-- NOTE: batch_id is intentionally denormalized (derivable from exam_id → exams.batch_id).
-- Kept for query performance — avoids join when listing results by batch.
-- Application code must ensure results.batch_id == exams.batch_id on insert.
create table results (
  id uuid primary key default gen_random_uuid(),
  center_id uuid references centers(id),
  exam_id uuid references exams(id),
  batch_id uuid references batches(id),
  student_id uuid references students(id),
  obtained_marks numeric(6,2),
  rank integer,                              -- calculated on bulk insert
  created_at timestamptz default now(),
  unique(exam_id, student_id)
);

-- FEES
create table fee_records (
  id uuid primary key default gen_random_uuid(),
  center_id uuid references centers(id),
  student_id uuid references students(id),
  batch_id uuid references batches(id),
  invoice_number text,                       -- "{invoice_prefix}-{counter}" e.g. "ABC-0042"
  amount numeric(10,2) not null,
  due_date date not null,                    -- when this fee becomes due (works for both monthly and per_class)
  month_year text,                           -- "2026-05" for monthly cycle (NULL for per_class)
  session_id uuid references class_sessions(id), -- for per_class cycle (NULL for monthly)
  paid_at timestamptz,
  payment_method text check (payment_method in ('cash', 'bkash', 'nagad', 'bank')),
  received_by uuid references profiles(id),
  status text default 'due' check (status in ('due', 'paid', 'waived')),
  notes text,                                -- e.g., "partial waiver approved"
  created_at timestamptz default now()
);

-- SMS LOG
create table sms_logs (
  id uuid primary key default gen_random_uuid(),
  center_id uuid references centers(id),
  student_id uuid references students(id),   -- nullable; enables "all SMS for this student" queries
  recipient_phone text,
  message text,
  type text check (type in ('absence', 'fee_due', 'result', 'custom')),
  status text check (status in ('sent', 'failed', 'pending')),
  sent_at timestamptz default now()
);
```

---

### RLS: Center ID Lookup Function (Avoiding Circular Reference)

The `profiles` table itself needs an RLS policy, but a naive `select center_id from profiles where id = auth.uid()` policy *on profiles* creates a circular reference — Postgres will recurse. Fix: a `SECURITY DEFINER` function that bypasses RLS to look up `center_id`, used in all policies.

```sql
-- SECURITY DEFINER function: runs with the privileges of the function owner (superuser),
-- bypassing RLS. This is safe because it only returns a single scalar value (center_id)
-- for the authenticated user. It cannot be abused to read arbitrary rows.
create or replace function get_my_center_id()
returns uuid
language sql
stable
security definer
set search_path = public
as $$
  select center_id from profiles where id = auth.uid()
$$;
```

### RLS Policy Pattern

Apply to every table including `profiles` and `centers`:

```sql
-- PROFILES (uses the SECURITY DEFINER function to avoid circular reference)
alter table profiles enable row level security;

create policy "profiles_isolation" on profiles
  using (center_id = get_my_center_id());

-- Allow insert for users who don't have a profile yet (during onboarding)
create policy "profiles_insert_own" on profiles
  for insert with check (id = auth.uid());

-- CENTERS (must also have RLS — otherwise any authenticated user can query any center)
alter table centers enable row level security;

create policy "centers_isolation" on centers
  using (id = get_my_center_id());

-- Allow insert during onboarding (before profile exists, so center_id lookup fails)
-- Onboarding creates center + profile in a transaction via service role Edge Function.
-- Client-side reads are still isolated by the using() policy.

-- ALL OTHER TABLES (standard pattern)
alter table subjects enable row level security;
create policy "center_isolation" on subjects
  using (center_id = get_my_center_id());

-- Repeat for: batches, students, enrollments, class_sessions, attendance,
-- exams, results, fee_records, sms_logs
-- (identical pattern: center_id = get_my_center_id())
```

**Key points:**
- `get_my_center_id()` is `SECURITY DEFINER` + `STABLE` — it bypasses RLS for the lookup but is safe because it only returns a scalar for `auth.uid()`.
- `set search_path = public` prevents search_path hijacking.
- `centers` has its own policy — without it, any authenticated user can read any center row.
- Onboarding (center creation + first profile) uses service role in the Edge Function because no profile/center exists yet. All subsequent operations go through RLS.

---

### Invoice Number Generation (Per-Center Prefix)

Each center has its own `invoice_prefix` (e.g., "ABC") and a monotonic counter. The trigger uses the center's prefix, not a hardcoded string.

```sql
create or replace function assign_invoice_number()
returns trigger language plpgsql as $$
declare
  new_counter integer;
  prefix text;
begin
  update centers
    set invoice_counter = invoice_counter + 1
    where id = NEW.center_id
    returning invoice_counter, invoice_prefix into new_counter, prefix;
  NEW.invoice_number := prefix || '-' || lpad(new_counter::text, 4, '0');
  return NEW;
end;
$$;

create trigger trg_assign_invoice
  before insert on fee_records
  for each row execute function assign_invoice_number();
```

Gaps in the sequence are acceptable (e.g., after a rolled-back transaction). Duplicates are not — the `UPDATE ... RETURNING` is atomic under row-level locking.

---

### Teacher Invite: Post-Signup Profile Creation

When an owner invites a teacher, the flow is:

1. Owner calls `center-setup/invite-teacher` Edge Function with `{email, full_name, role: 'teacher', subjects}`.
2. Edge Function (service role) calls `supabase.auth.admin.inviteUserByEmail()` with `data: { center_id, full_name, role, subjects }` in the invite metadata.
3. Teacher receives invite email, signs up via Supabase Auth.
4. A Postgres trigger on `auth.users` insert (or a Supabase Auth hook) creates the `profiles` row using the metadata:

```sql
-- Database trigger on auth.users (applied via migration)
create or replace function handle_new_user()
returns trigger language plpgsql
security definer
set search_path = public
as $$
begin
  insert into profiles (id, center_id, full_name, phone, role, subjects)
  values (
    NEW.id,
    (NEW.raw_user_meta_data->>'center_id')::uuid,
    coalesce(NEW.raw_user_meta_data->>'full_name', ''),
    NEW.phone,
    coalesce(NEW.raw_user_meta_data->>'role', 'teacher'),
    case
      when NEW.raw_user_meta_data->'subjects' is not null
      then array(select jsonb_array_elements_text(NEW.raw_user_meta_data->'subjects'))
      else '{}'::text[]
    end
  );
  return NEW;
end;
$$;

create trigger on_auth_user_created
  after insert on auth.users
  for each row execute function handle_new_user();
```

**For owner self-signup (onboarding):** The `center-setup` Edge Function (service role) creates the center row first, then sets `center_id` in the user's metadata, then the trigger above creates the profile. This is done in a single transaction within the Edge Function.

---

## Supabase Edge Functions (API Layer)

### Auth Pattern: Two-Client Model

Edge Functions use **two Supabase clients** with different privilege levels:

```typescript
// supabase/functions/_shared/clients.ts

import { createClient } from "https://esm.sh/@supabase/supabase-js@2";

// USER CLIENT: ANON_KEY + user's JWT. All queries go through RLS.
// Use for: reading/writing user-scoped data (batches, students, attendance, fees, results).
export function createUserClient(authHeader: string) {
  return createClient(
    Deno.env.get("SUPABASE_URL")!,
    Deno.env.get("SUPABASE_ANON_KEY")!,
    { global: { headers: { Authorization: authHeader } } }
  );
}

// ADMIN CLIENT: SERVICE_ROLE_KEY. Bypasses RLS entirely.
// Use ONLY for: operations that legitimately need to bypass RLS.
// Specifically: SMS balance decrement, invoice counter update, auth admin API,
// center creation during onboarding, cross-center admin queries.
export function createAdminClient() {
  return createClient(
    Deno.env.get("SUPABASE_URL")!,
    Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!
  );
}
```

**Rules:**
- Default to `createUserClient()` — it respects RLS. A center can never accidentally read another center's data.
- Use `createAdminClient()` only for explicitly documented privileged operations:
  - `center-setup`: creates center + profile (no profile exists yet, so RLS lookup fails)
  - `sms-send`: decrements `centers.sms_balance` (UPDATE on `centers` via service role)
  - `fee-pay`: triggers invoice counter update (handled by DB trigger, but fee status update via user client is fine)
  - `report-*`: service role for PDF generation reads if needed (though user client works too since RLS is correct)
  - Auth admin operations: `inviteUserByEmail`, user metadata updates

**Each function pattern:**
```typescript
// supabase/functions/<name>/index.ts
import { serve } from "https://deno.land/std/http/server.ts";
import { createUserClient, createAdminClient } from "../_shared/clients.ts";

serve(async (req) => {
  const authHeader = req.headers.get("Authorization");
  if (!authHeader) return new Response("Unauthorized", { status: 401 });

  // User-scoped client (RLS enforced)
  const supabase = createUserClient(authHeader);

  // Verify user is authenticated
  const { data: { user }, error } = await supabase.auth.getUser();
  if (error || !user) return new Response("Unauthorized", { status: 401 });

  // ... function logic using `supabase` (user client)
  // ... use createAdminClient() ONLY for privileged operations
});
```

### Function List

```
supabase/functions/
├── _shared/
│   └── clients.ts                # Two-client pattern (user + admin)
├── center-setup/                 # Onboarding + teacher invite (uses admin client)
├── batch-manage/                 # CRUD for batches (user client)
├── student-manage/               # CRUD for students + enrollment (user client)
├── session-manage/               # Create/list class sessions (user client)
├── attendance-bulk/              # Bulk mark attendance (user client) + SMS trigger (admin for sms_balance)
├── fee-generate/                 # Auto-generate fee records with due_date (user client)
├── fee-pay/                      # Mark fee as paid (user client; invoice assigned by DB trigger)
├── fee-overdue/                  # List overdue fees: due_date < today (user client)
├── exam-manage/                  # Create exams, enter results, calculate ranks (user client)
├── sms-send/                     # Send SMS via SSL Wireless (admin client for balance decrement)
├── sms-logs/                     # Retrieve SMS history (user client)
├── report-monthly/               # Monthly fee collection PDF (user client)
├── report-attendance/            # Batch attendance sheet PDF (user client)
└── report-result-card/           # Student result card PDF (user client)
```

---

### SMS Rate Limiting

`sms-send` enforces two limits to prevent accidental SMS credit drain:

1. **Per-call cap:** A single invocation can send at most **20 SMS** (configurable). Prevents a loop bug from draining the entire balance in one request.
2. **Daily cap per center:** `centers.sms_daily_cap` (default 100). The function counts `sms_logs` rows for the center where `sent_at::date = today` and rejects if the cap is reached.
3. **Balance check:** `centers.sms_balance > 0` before each send. Decrement is atomic (`UPDATE ... SET sms_balance = sms_balance - 1 WHERE sms_balance > 0 RETURNING sms_balance`). If the RETURNING returns no rows, balance is exhausted — abort.

```typescript
// Inside sms-send Edge Function (pseudocode)
const PER_CALL_MAX = 20;

if (recipients.length > PER_CALL_MAX) {
  return new Response(`Cannot send more than ${PER_CALL_MAX} SMS in one call`, { status: 400 });
}

// Check daily cap
const { count } = await adminClient
  .from("sms_logs")
  .select("id", { count: "exact", head: true })
  .eq("center_id", centerId)
  .gte("sent_at", todayStart);

const { data: center } = await adminClient
  .from("centers")
  .select("sms_daily_cap, sms_balance")
  .eq("id", centerId)
  .single();

if (count + recipients.length > center.sms_daily_cap) {
  return new Response("Daily SMS cap reached", { status: 429 });
}

if (center.sms_balance < recipients.length) {
  return new Response("Insufficient SMS credits", { status: 402 });
}

// Send each SMS, decrement balance atomically per send
for (const recipient of recipients) {
  const { data: updated } = await adminClient.rpc("decrement_sms_balance", { p_center_id: centerId });
  if (!updated) break; // balance hit zero mid-batch
  // ... call SSL Wireless API, insert sms_logs row
}
```

```sql
-- Helper RPC for atomic balance decrement
create or replace function decrement_sms_balance(p_center_id uuid)
returns boolean language plpgsql as $$
begin
  update centers
    set sms_balance = sms_balance - 1
    where id = p_center_id and sms_balance > 0;
  return found;
end;
$$;
```

---

### Fee Overdue Logic (Unified via `due_date`)

`fee_records.due_date` is the single source of truth for overdue checks, regardless of fee cycle type.

**Monthly fees:** `due_date` = first day of the next month after `month_year` (e.g., for `month_year = "2026-05"`, `due_date = 2026-06-01`).

**Per-class fees:** `due_date` = session date + 7 days (or configurable grace period).

**Overdue query:**
```sql
select fr.*, s.full_name as student_name, b.name as batch_name
from fee_records fr
join students s on s.id = fr.student_id
join batches b on b.id = fr.batch_id
where fr.center_id = get_my_center_id()
  and fr.status = 'due'
  and fr.due_date < current_date
order by fr.due_date asc;
```

No branching logic needed — both fee types use `due_date`.

---

### Per-Class Fee Billing: Enrollment Date Filtering

When generating per-class fees, only sessions *after* the student's enrollment date are billable:

```sql
-- Generate per-class fees for a batch in a given month
insert into fee_records (center_id, student_id, batch_id, amount, due_date, session_id, status)
select
  cs.center_id,
  e.student_id,
  cs.batch_id,
  b.fee_amount,
  cs.session_date + interval '7 days',     -- due 7 days after session
  cs.id,
  'due'
from class_sessions cs
join enrollments e on e.batch_id = cs.batch_id
  and e.status = 'active'
  and e.enrolled_at::date <= cs.session_date  -- only sessions AFTER enrollment
join batches b on b.id = cs.batch_id
where cs.batch_id = $1                       -- batch parameter
  and cs.session_date between $2 and $3      -- month range
  and b.fee_cycle = 'per_class'
on conflict do nothing;                       -- idempotent: skip already-generated fees
```

---

## Project Structure

```
batch-board/
├── docs/
│   ├── BatchBoard.md              # This document
│   └── plan.md                    # Development timeline
│
├── supabase/
│   ├── migrations/
│   │   ├── 001_initial_schema.sql # All tables
│   │   ├── 002_rls_policies.sql   # get_my_center_id() + all RLS policies
│   │   ├── 003_invoice_trigger.sql# Per-center invoice number function + trigger
│   │   ├── 004_auth_trigger.sql   # handle_new_user() for teacher invite flow
│   │   ├── 005_sms_helpers.sql    # decrement_sms_balance() RPC
│   │   └── 006_indexes.sql        # Performance indexes
│   ├── functions/
│   │   ├── _shared/
│   │   │   └── clients.ts         # Two-client pattern (user + admin)
│   │   ├── center-setup/index.ts
│   │   ├── batch-manage/index.ts
│   │   ├── student-manage/index.ts
│   │   ├── session-manage/index.ts
│   │   ├── attendance-bulk/index.ts
│   │   ├── fee-generate/index.ts
│   │   ├── fee-pay/index.ts
│   │   ├── fee-overdue/index.ts
│   │   ├── exam-manage/index.ts
│   │   ├── sms-send/index.ts
│   │   ├── sms-logs/index.ts
│   │   ├── report-monthly/index.ts
│   │   ├── report-attendance/index.ts
│   │   └── report-result-card/index.ts
│   └── seed.sql                   # Dev seed data
│
├── frontend/                      # Next.js App Router
│   ├── app/
│   │   ├── (auth)/
│   │   │   ├── login/page.tsx
│   │   │   └── setup/page.tsx     # Center onboarding
│   │   ├── (dashboard)/
│   │   │   ├── layout.tsx         # Sidebar + auth guard
│   │   │   ├── dashboard/page.tsx # Overview stats
│   │   │   ├── batches/
│   │   │   │   ├── page.tsx       # Batch list
│   │   │   │   └── [id]/page.tsx  # Batch detail + students + sessions
│   │   │   ├── students/
│   │   │   │   ├── page.tsx
│   │   │   │   └── [id]/page.tsx  # Student profile + fee + attendance history
│   │   │   ├── teachers/
│   │   │   │   └── page.tsx       # Teacher list + invite + batch assignment
│   │   │   ├── attendance/page.tsx# Mark attendance (teacher mobile view)
│   │   │   ├── fees/page.tsx      # Fee management + invoice list
│   │   │   ├── results/page.tsx   # Exam management + result entry + leaderboard
│   │   │   ├── sms/page.tsx       # SMS compose + logs
│   │   │   └── reports/page.tsx   # Generate + download PDFs
│   │   └── layout.tsx
│   ├── components/
│   │   ├── ui/                    # shadcn components
│   │   ├── attendance/
│   │   │   └── AttendanceGrid.tsx # Bulk mark UI (offline-capable)
│   │   ├── fees/
│   │   │   ├── FeeTable.tsx
│   │   │   └── InvoiceModal.tsx   # Printable invoice/receipt
│   │   ├── results/
│   │   │   └── ResultLeaderboard.tsx
│   │   └── shared/
│   │       ├── Sidebar.tsx
│   │       └── DataTable.tsx      # Reusable sortable/filterable table
│   ├── lib/
│   │   ├── supabase.ts            # Browser + server Supabase clients
│   │   ├── api.ts                 # Edge Function caller with JWT inject
│   │   └── utils/
│   │       ├── phone.ts           # BD phone normalization (+8801XXXXXXXXX)
│   │       └── invoice.ts         # Invoice number display helpers
│   └── middleware.ts              # Route protection
│
├── .env.example
└── README.md
```

---

## Feature Roadmap

### MVP — Weeks 1–3 (Core Operations)

| Week | Features |
|---|---|
| **1** | Supabase setup + migrations + RLS + `get_my_center_id()`, Auth + center onboarding, Subjects + Batch CRUD, Teacher invite flow |
| **2** | Student enrollment, Class session tracking, Attendance grid (offline-capable), Fee generation (monthly + per-class with enrollment date filter) |
| **3** | Payment marking + per-center invoice numbers, SMS absence alert (rate-limited), Monthly collection PDF, Deploy to Vercel + Supabase |

**MVP definition:** A teacher walks into class, creates a session, marks attendance in 2 minutes, and a guardian gets an SMS if their child is absent. Owner can see who hasn't paid fees this month and print a collection report. Invoice number with center prefix appears on every payment. Ship this.

---

### V1 Extensions — Weeks 4–6 (Polish + Exam Module)

| Week | Features |
|---|---|
| **4** | Exam management UI, Result entry (bulk), Rank auto-calculation, Result card PDF |
| **5** | Student profile page (full history), Attendance report PDF, SMS fee reminders (bulk), SMS credit balance + daily cap display |
| **6** | CI/CD setup, E2E tests (Playwright), Security audit (RLS + two-client tests), Performance review, Staging deploy |

---

### V2 — Post-Launch (Real Users)

- Fee waiver + partial payment support
- WhatsApp notification (via WhatsApp Business API — higher open rate than SMS in BD)
- Student self-service portal (view own attendance + fee status via phone number OTP)
- Center analytics dashboard (revenue trend, attendance trend, batch performance comparison)
- Mobile-optimized PWA (teachers marking attendance on phone in classroom with service worker)
- bKash payment link generation per fee record
- Multi-branch support (one owner, multiple center locations)

---

## Hard Constraints to Design Around Now

- **SMS cost:** SSL Wireless charges ~৳0.35/SMS. Build SMS as opt-in per center (`sms_opted_in`), not default-on. Display SMS credit balance prominently in dashboard. Deduct atomically via `decrement_sms_balance()`. Enforce per-call cap (20) and daily cap (`sms_daily_cap`, default 100).

- **Offline attendance:** Teachers in BD often have flaky internet. Build the `AttendanceGrid` with optimistic UI + local queue (`localStorage` + retry on reconnect). Session creation and attendance submission must work offline-first.

- **Bengali names:** DB, UI, and PDF pipeline must handle Unicode end-to-end. Test with "মিনহাজুল" early. jsPDF needs SolaimanLipi loaded explicitly via `jspdf-customfonts`. Validate in Week 1.

- **Phone number format:** BD numbers come as `01XXXXXXXXX` and `+8801XXXXXXXXX`. Normalize to `+8801XXXXXXXXX` at input in `lib/utils/phone.ts`. SSL Wireless expects E.164 format.

- **Per-class fee billing:** `class_sessions` is the source of truth. `fee_records.session_id` links a fee to the specific class. Only sessions where `session_date >= enrollment.enrolled_at` are billable. The `fee-generate` Edge Function enforces this filter.

- **Due date for overdue:** `fee_records.due_date` is the universal overdue indicator. Monthly fees: `due_date = first of next month`. Per-class fees: `due_date = session_date + 7 days`. The `fee-overdue` function simply queries `due_date < today AND status = 'due'`.

- **Invoice integrity:** `invoice_counter` increments atomically in Postgres via trigger — never in application code. Each center has its own `invoice_prefix` (e.g., "ABC") stored on the `centers` row. Gaps in the sequence are acceptable; duplicates are not.

- **RLS always on:** Every table has RLS enabled — including `centers` and `profiles`. All policies use `get_my_center_id()` (a `SECURITY DEFINER` function) to avoid circular reference on `profiles`. Edge Functions use the user client (ANON_KEY + JWT) for all user-scoped queries. The admin client (SERVICE_ROLE_KEY) is used only for: center creation during onboarding, SMS balance decrement, and auth admin API calls (teacher invites).

- **Denormalization:** `results.batch_id` is intentionally denormalized (derivable from `exams.batch_id`). Kept for query speed. Application code must ensure consistency on insert; a CHECK constraint or trigger can enforce `results.batch_id = (select batch_id from exams where id = results.exam_id)` if drift is a concern.
