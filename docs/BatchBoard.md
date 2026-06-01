Full operations platform for the thousands of coaching centers in BD (admission coaching, HSC, O-levels) that run entirely on paper and WhatsApp groups.

**The real problem:** BD has tens of thousands of coaching centers. They manage attendance, fees, results, and batch scheduling via notebooks and WhatsApp. Zero software penetration.

**Stack:** Next.js + Node/Express + Postgres + NextAuth + SSL Wireless SMS + jsPDF

##### Key features:

- Batch/class management (subject, teacher, schedule, capacity)
- Student enrollment + fee tracking (paid/due/overdue)
- Attendance marking (teacher-facing mobile view)
- SMS alert to parents on absence or fee due
- Result entry + rank calculation per batch
- Monthly fee collection report (PDF)

### Core Architecture

**Frontend:** Next.js 14 (App Router) + TypeScript + Tailwind + shadcn/ui
**Backend:** Node.js + Express.js (separate service)
**Database:** Supabase (Postgres)
**Auth:** Supabase Auth (built-in — skip NextAuth complexity)
**SMS:** SSL Wireless (BD-local, cheapest per SMS)
**PDF:** jsPDF + jspdf-autotable
**File Storage:** Supabase Storage (student photos, documents)
**Deployment:** Vercel (frontend) + Railway (Express backend)

**Why Supabase Auth over NextAuth here:** Multi-tenant RLS (Row Level Security) is native to Supabase. Each coaching center's data isolation becomes a Postgres policy, not application logic. This is the architecturally correct call.

### Database Schema

#### Multi-Tenancy Model

Every table has `center_id`. Supabase RLS policies enforce isolation at DB level — no center can ever read another center's data even if API leaks.

```
-- TENANTS
create table centers (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  address text,
  phone text,
  logo_url text,
  sms_balance integer default 0,        -- prepaid SMS credits
  created_at timestamptz default now()
);

-- USERS & ROLES
create table profiles (
  id uuid primary key references auth.users(id),
  center_id uuid references centers(id),
  full_name text not null,
  phone text,
  role text check (role in ('owner', 'admin', 'teacher')),
  created_at timestamptz default now()
);

-- SUBJECTS
create table subjects (
  id uuid primary key default gen_random_uuid(),
  center_id uuid references centers(id),
  name text not null,                    -- "Physics", "English"
  code text                              -- "PHY", "ENG"
);

-- BATCHES
create table batches (
  id uuid primary key default gen_random_uuid(),
  center_id uuid references centers(id),
  name text not null,                    -- "HSC-26 Morning"
  subject_id uuid references subjects(id),
  teacher_id uuid references profiles(id),
  schedule jsonb,                        -- {days: ["Sat","Mon"], time: "08:00"}
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
  phone text,                            -- student's own phone
  guardian_phone text,                   -- for SMS alerts
  photo_url text,
  class_level text,                      -- "HSC", "SSC", "O-Level"
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

-- ATTENDANCE
create table attendance (
  id uuid primary key default gen_random_uuid(),
  center_id uuid references centers(id),
  batch_id uuid references batches(id),
  student_id uuid references students(id),
  class_date date not null,
  status text check (status in ('present', 'absent', 'late')),
  marked_by uuid references profiles(id),
  created_at timestamptz default now(),
  unique(batch_id, student_id, class_date)
);

-- FEES
create table fee_records (
  id uuid primary key default gen_random_uuid(),
  center_id uuid references centers(id),
  student_id uuid references students(id),
  batch_id uuid references batches(id),
  amount numeric(10,2) not null,
  month_year text,                       -- "2026-05" for monthly
  paid_at timestamptz,
  payment_method text check (payment_method in ('cash', 'bkash', 'nagad', 'bank')),
  received_by uuid references profiles(id),
  status text default 'due' check (status in ('due', 'paid', 'waived')),
  created_at timestamptz default now()
);

-- RESULTS
create table results (
  id uuid primary key default gen_random_uuid(),
  center_id uuid references centers(id),
  batch_id uuid references batches(id),
  student_id uuid references students(id),
  exam_name text not null,               -- "Monthly Test - April"
  total_marks numeric(6,2),
  obtained_marks numeric(6,2),
  exam_date date,
  created_at timestamptz default now()
);

-- SMS LOG
create table sms_logs (
  id uuid primary key default gen_random_uuid(),
  center_id uuid references centers(id),
  recipient_phone text,
  message text,
  type text check (type in ('absence', 'fee_due', 'result', 'custom')),
  status text check (status in ('sent', 'failed', 'pending')),
  sent_at timestamptz default now()
); 
```

RLS Policy (example for attendance):

```
alter table attendance enable row level security;

create policy "center_isolation" on attendance
  using (center_id = (
    select center_id from profiles where id = auth.uid()
  ));
```
Apply identical policy pattern to every table.

### API Spec (Express Backend)

#### Auth (handled by Supabase — no custom endpoints needed)

Pass Supabase JWT in `Authorization: Bearer <token>` header. Backend validates via Supabase admin SDK.

#### Endpoints

```
-- CENTER
POST   /api/center/setup              # Onboarding: create center + owner profile

-- BATCHES
GET    /api/batches                   # List all batches for center
POST   /api/batches                   # Create batch
PATCH  /api/batches/:id               # Update batch
DELETE /api/batches/:id               # Archive batch (soft delete)

-- STUDENTS
GET    /api/students                  # List students (filter: batch_id, status)
POST   /api/students                  # Register student
PATCH  /api/students/:id              # Update student info
POST   /api/students/:id/enroll       # Enroll in batch
PATCH  /api/enrollments/:id/drop      # Drop from batch

-- ATTENDANCE
GET    /api/attendance?batch_id&date  # Get attendance for batch+date
POST   /api/attendance/bulk           # Mark attendance (array of {student_id, status})
GET    /api/attendance/report?student_id&month  # Student attendance %

-- FEES
GET    /api/fees?batch_id&month_year  # Fee status for batch+month
POST   /api/fees/generate             # Auto-generate fee records for batch+month
PATCH  /api/fees/:id/pay              # Mark fee as paid
GET    /api/fees/overdue              # All overdue fees across center

-- RESULTS
POST   /api/results/bulk              # Enter results for exam
GET    /api/results?batch_id&exam     # Results with rank calculation

-- SMS
POST   /api/sms/absence               # Send absence alert (triggered post-attendance)
POST   /api/sms/fee-reminder          # Send fee due reminders (bulk)
POST   /api/sms/custom                # Custom message to batch/student list
GET    /api/sms/logs                  # SMS history + delivery status

-- REPORTS
GET    /api/reports/monthly-collection  # Fee collection summary → PDF
GET    /api/reports/attendance-sheet    # Batch attendance sheet → PDF
GET    /api/reports/result-card/:student_id  # Student result card → PDF
```

### Project Structure

```
batchboard/
├── frontend/                          # Next.js App Router
│   ├── app/
│   │   ├── (auth)/
│   │   │   ├── login/page.tsx
│   │   │   └── setup/page.tsx         # Center onboarding
│   │   ├── (dashboard)/
│   │   │   ├── layout.tsx             # Sidebar + auth guard
│   │   │   ├── dashboard/page.tsx     # Overview stats
│   │   │   ├── batches/
│   │   │   │   ├── page.tsx           # Batch list
│   │   │   │   └── [id]/page.tsx      # Batch detail + students
│   │   │   ├── students/
│   │   │   │   ├── page.tsx
│   │   │   │   └── [id]/page.tsx      # Student profile + history
│   │   │   ├── attendance/page.tsx    # Mark attendance (teacher view)
│   │   │   ├── fees/page.tsx          # Fee management
│   │   │   ├── results/page.tsx       # Result entry + leaderboard
│   │   │   ├── sms/page.tsx           # SMS compose + logs
│   │   │   └── reports/page.tsx       # Generate + download PDFs
│   │   └── layout.tsx
│   ├── components/
│   │   ├── ui/                        # shadcn components
│   │   ├── attendance/
│   │   │   └── AttendanceGrid.tsx     # Bulk mark UI
│   │   ├── fees/
│   │   │   └── FeeTable.tsx
│   │   └── shared/
│   │       ├── Sidebar.tsx
│   │       └── DataTable.tsx          # Reusable table component
│   ├── lib/
│   │   ├── supabase.ts                # Client + server clients
│   │   └── api.ts                     # Axios instance with JWT inject
│   └── middleware.ts                  # Route protection
│
├── backend/                           # Express service
│   ├── src/
│   │   ├── middleware/
│   │   │   ├── auth.ts                # Supabase JWT verification
│   │   │   └── centerGuard.ts         # Inject center_id from profile
│   │   ├── routes/
│   │   │   ├── batches.ts
│   │   │   ├── students.ts
│   │   │   ├── attendance.ts
│   │   │   ├── fees.ts
│   │   │   ├── results.ts
│   │   │   ├── sms.ts
│   │   │   └── reports.ts
│   │   ├── services/
│   │   │   ├── smsService.ts          # SSL Wireless integration
│   │   │   ├── pdfService.ts          # jsPDF report generation
│   │   │   └── feeService.ts          # Fee generation + overdue logic
│   │   └── index.ts
│   └── package.json
```

### Feature Roadmap

#### MVP (Build this — 3 weeks)

|Week|Features|
|---|---|
|**1**|Supabase setup + RLS policies, Auth + center onboarding, Batch + student CRUD|
|**2**|Attendance marking (bulk grid UI), Fee record generation + payment marking, Basic dashboard stats|
|**3**|SMS absence alert (1 trigger only), Monthly fee collection PDF, Deploy (Vercel + Railway)|

**MVP definition:** A teacher can walk into class, mark attendance in 2 minutes, and a guardian gets an SMS if their child is absent. Owner can see who hasn't paid fees this month. That's it. Ship this.

---

#### V2 (Post-MVP, post-job search — or if you get real users)

- Result entry + rank leaderboard per batch
- Multi-teacher role management
- Fee waiver + partial payment support
- WhatsApp notification (via WhatsApp Business API — higher open rate than SMS in BD)
- Student self-service portal (view own attendance + fee status via phone number login)
- Center analytics dashboard (revenue trend, attendance trend, batch performance comparison)
- Mobile-optimized PWA (teachers marking attendance on phone in classroom)
- bKash payment link generation per fee record

---

### Hard Constraints to Design Around Now

- **SMS cost:** SSL Wireless charges ~৳0.35/SMS. Build SMS as opt-in per center, not default-on. Add SMS credit balance display in dashboard.
- **Offline attendance:** Teachers in BD often have flaky internet. Build the attendance grid with optimistic UI + local queue (use `localStorage` + retry on reconnect). Don't make it dependent on real-time DB writes.
- **Bengali names:** Ensure your DB and PDF pipeline handles Unicode correctly end-to-end. Test with "মিনহাজুল" early — jsPDF needs a Bengali font loaded explicitly (use `jspdf-customfonts` with SolaimanLipi).
- **Phone number format:** BD numbers come as `01XXXXXXXXX` and `+8801XXXXXXXXX`. Normalize to `+8801XXXXXXXXX` at input layer in a single util function. SSL Wireless expects E.164 format.

---
