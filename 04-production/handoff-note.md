# Vantage Controls — Engineering Handoff

Status: working prototype backed by a real external Supabase project.
Written from the implemented code and the live database (inspected 2026-09-21). Where the
implementation differs from the Living PRD or the earlier plan, the discrepancy is flagged inline.

---

## 1. 60-Second Overview

Vantage Controls is an internal control-action tracker for a controls/internal-audit function.
A reviewer (Controls & Audit) raises a **control action** containing a structured Finding, a
Recommendation and a list of **required evidence** items, and assigns it to an **action owner**.
The owner uploads the requested documents, sends the action for review, and the reviewer either
approves it (action becomes Completed) or returns it with a required comment for rework. The
hypothesis is that structured finding + recommendation + explicit evidence checklist lets owners
complete assigned actions without back-and-forth. Implemented today: real email/password sign-in,
six people and eight seeded demo actions stored in an external Supabase Postgres database, real
document upload/download into a private storage bucket, the full create → work → submit → recall →
return → resubmit → approve workflow, comments, an append-only audit trail, and permissions
enforced in the database by RLS plus a status-transition trigger. Not implemented: user
administration, notifications, malware scanning, automated tests.

---

## 2. Start Here

Read in this order — about an hour to full orientation:

1. `src/features/control-actions/model/control-action.ts` — the domain type (`ControlAction`),
   status values, derived due-date logic, scope filters. Everything else speaks this shape.
2. `src/features/control-actions/data/actions-repository.ts` — the single read path: one Supabase
   query joining actions, profiles, evidence, comments and events into `ControlAction[]`.
3. `src/features/control-actions/state/control-action-store.ts` — the single write path: every
   workflow mutation lives here (create, upload, remove, submit, recall, return, approve, comment,
   edit), each followed by an audit event and a refresh.
4. `src/features/auth/state/session-store.ts` — who the signed-in user is: auth user → profile →
   reviewer capability.
5. `supabase/migrations/20260921133838_*.sql` — the schema, RLS policies and transition trigger.
   This file, not the frontend, is the real authorization boundary.
6. Then the screens: `src/features/dashboard/`, `action-detail/`, `reviewer-queue/`,
   `review-action/`, `create-action/`.

---

## 3. Architecture in Plain Language

**Frontend.** TanStack Start v1 (React 19, Vite, file-based TanStack Router), Tailwind v4,
shadcn/ui components in `src/components/ui`. Rendering is effectively client-side for data:
all Supabase reads happen in the browser after hydration.

**Structure.** `src/features/<feature>/` — each feature holds its screen, its hook(s) and its
components. Shared domain/data/state for actions lives in `src/features/control-actions/`
(`model/`, `data/`, `state/`, `components/`). Routes in `src/routes/` are thin: they set page
metadata and render a feature screen.

**Backend.** An **external** Supabase project — **Vantage Controls**, ref `tnfxxspxcsokdcozrrlf`
(also set in `supabase/config.toml`). No Lovable Cloud. There are **no server functions and no
API routes** in this app: the browser talks to Supabase directly with the publishable key, and
all authorization is done by Postgres RLS and triggers.

**Auth.** Supabase Auth email/password. `session-store.ts` subscribes to `onAuthStateChange`,
then loads the matching `profiles` row (`auth_user_id = auth.uid()`) and any `user_roles` rows.
`AuthGate` in `src/routes/__root.tsx` redirects signed-out users to `/auth`
(public paths: `/auth`, `/reset-password`).

**Data flow.** Screen → `useActions()` (a `useSyncExternalStore` store) → `fetchActions()` → one
PostgREST query → RLS filters rows → mapped to `ControlAction[]`. Mutations write directly to
tables, insert an `action_events` row, then call `refreshActions()` (full reload; no optimistic
updates, no realtime).

**Evidence storage.** Private bucket `evidence`, 25 MB limit, no public URLs. Uploads go straight
from the browser; downloads use a 60-second signed URL. Storage policies mirror the table rules.

---

## 4. Components & Responsibilities

Screens / routes
- `src/routes/index.tsx` → `src/features/dashboard/needs-attention-screen.tsx` — "Your open work"
  dashboard: scope toggle (My team / Mine), search, control-area and severity breakdown filters,
  the "needs attention" list, reviewer-only "New action" and "Reviewer queue" entry points.
  Logic in `use-needs-attention.ts`.
- `src/routes/actions/$actionId.tsx` → `features/action-detail/action-detail-screen.tsx` — owner
  workspace: finding, recommendation, evidence checklist with upload/remove, send for review /
  recall, comments, audit trail.
- `src/routes/actions/new.tsx` → `features/create-action/create-action-screen.tsx` (+
  `use-create-action-form.ts`) — reviewer-only creation form.
- `src/routes/review/index.tsx` → `features/reviewer-queue/reviewer-queue-screen.tsx` (+
  `use-reviewer-queue.ts`) — actions awaiting review, with loading/empty/error states.
- `src/routes/review/$actionId.tsx` → `features/review-action/review-action-screen.tsx` (+
  `use-review-decision.ts`) — evidence download, approve (required comment) or return (required
  comment).
- `src/routes/auth.tsx` — sign in + forgot password. `src/routes/reset-password.tsx` — set a new
  password from the emailed link.

Shared action components (`src/features/control-actions/components/`)
- `action-overview.tsx`, `action-status.tsx` (status badges/labels),
  `evidence-requirements.tsx` (checklist, upload, remove, download),
  `comments-panel.tsx`, `audit-trail-panel.tsx`,
  `action-information-editor.tsx` (reviewer edits to finding, recommendation, owner, reviewer,
  area, severity, due date).

Data / state
- `control-actions/data/actions-repository.ts` — reads, profile lookups, DB↔UI status mapping.
- `control-actions/data/evidence-storage.ts` — file validation, upload, delete, signed download.
- `control-actions/state/control-action-store.ts` — all mutations + audit logging.
- `auth/state/session-store.ts` — session, profile, reviewer capability, sign-in/out, reset.
- `auth/components/reviewer-guard.tsx` — `useReviewerAccess()` / `<ReviewerOnly>`.
- `auth/components/account-menu.tsx` — avatar menu with sign out.
- `src/integrations/supabase/client.ts` — generated browser client; `types.ts` — generated types.
  Other files in that folder (`client.server.ts`, `auth-middleware.ts`, `cron-auth.ts`,
  `auth-attacher.ts`) are template scaffolding and are **not used** by this app.

---

## 5. Data Model

All tables are in `public`, all have RLS enabled.

| Table | Purpose |
|---|---|
| `profiles` | A person. `id`, optional `auth_user_id` → `auth.users` (nullable), `full_name`, `initials`, `team` (default `Controls & Assurance`). Actions reference profiles, not auth users. |
| `user_roles` | System capabilities. `user_id` → `auth.users`, `role` enum `app_role` — currently only `'reviewer'`. |
| `control_actions` | The action. `ref` (unique, e.g. `001`), `title`, `area`, `severity` (CHECK: Critical/High/Medium/Low), `finding`, `recommendation`, `owner_id`/`reviewer_id`/`created_by` → `profiles`, `due_date`, `status` enum `action_status`. |
| `evidence_requirements` | One requested evidence item per row: `action_id`, `name`, `position`. |
| `evidence_files` | An uploaded document: `requirement_id`, `storage_path` (unique), `original_name`, `mime_type`, `size_bytes`, `uploaded_by` → `profiles`. |
| `action_comments` | `action_id`, `author_id` → `profiles`, `body`. Insert + select only. |
| `action_events` | Audit trail: `action_id`, `actor_id`, `actor_label`, `description`. Insert + select only — no update/delete grants or policies, so it is append-only. |

Relationships: action 1–N requirements 1–N files; action 1–N comments; action 1–N events.
Deleting an action cascades to requirements, files rows, comments and events (storage objects are
not cascaded — see risks).

**Status values.** Database enum: `in_progress`, `awaiting_review`, `completed`. The UI adds two
*derived* states that are never stored: `overdue` and `due-soon`. Mapping lives in
`dbStatusToUi()`/`statusFor()`: a `completed` action shows Completed, `awaiting_review` shows
Awaiting review, otherwise days-to-due decides — negative → Overdue, 0–5 days → Due soon,
else On track.

**Profile/auth relationship.** A profile exists whether or not the person can sign in. Only
profiles with `auth_user_id` set can log in. This was a deliberate demo decision so the eight
seeded actions keep their original owners and reviewers without creating six auth accounts.

**Seeded demo data** (fixed seed date 2026-09-21, migration `20260921134633_*.sql`): 6 profiles
(Dana Reyes, Priya Nair, Marcus Bell, Lena Okafor, Tomás Rivera, Aisha Khan — all one team),
8 actions with refs `001`–`008` and due dates 2026-09-18 → 2026-09-30, 17 evidence requirements,
8 placeholder PDF evidence files uploaded into the bucket, 8 comments and a full audit trail.
Live counts at inspection: 6 profiles, 8 actions (6 in progress, 2 awaiting review), 17
requirements, 10 evidence files / 10 storage objects, 8 comments, 31 events, 2 auth users.
(Counts above 8 files reflect testing after seeding — expected.)

---

## 6. Identity, Roles & Permissions

- **Profile ↔ account.** `profiles.auth_user_id` links a person to a Supabase Auth user. Unlinked
  profiles are assignable but cannot sign in.
- **Reviewer** is a *system capability* stored in `user_roles`, resolved server-side by
  `is_reviewer()` (via `has_role(auth.uid(), 'reviewer')`). It is never inferred from a name,
  email or label.
- **Action Owner is per-action**, not a global role: ownership is simply
  `control_actions.owner_id = my_profile_id()`. A reviewer can also own actions; the two are not
  mutually exclusive. There is no `owner` role and no Super Admin.

What each can do (enforced in SQL):
- **Read an action**: reviewers (all), the assigned owner, the assigned reviewer, or anyone whose
  team matches the owner's team (`can_read_action()`). With one seeded team, everyone reads
  everything — a demo simplification.
- **Create an action / edit its metadata / manage requirements**: reviewers only.
- **Owner writes**: upload and delete evidence files only while the action is `in_progress`
  (`is_action_owner()` + `action_is_editable()`), and change status only
  `in_progress ↔ awaiting_review`. The `enforce_action_update_rules` trigger raises an exception
  if an owner touches any other column or makes any other transition.
- **Comments**: only the assigned owner or reviewer of that action may insert, and `author_id`
  must be their own profile. **Events**: insert only, `actor_id` must be self.

**Frontend vs backend.** `ReviewerOnly` / `useReviewerAccess()` only hide and redirect; a
non-reviewer typing `/actions/new` is redirected, but the real protection is the RLS policy and
the trigger. Treat every frontend check as cosmetic.

---

## 7. Core Workflows

All writes are plain PostgREST calls from `control-action-store.ts`; authorization and the legal
set of transitions are enforced server-side by RLS policies and the
`enforce_action_update_rules` trigger.

1. **Create & assign** (reviewer) — `addAction()` generates the next `ref`, inserts the action
   with owner, reviewer, area, severity, due date, then inserts the evidence requirements.
   RLS requires `is_reviewer()` and `created_by = my_profile_id()`.
2. **Owner works the action** — uploads one or more documents per requirement
   (`attachFiles` → validate → storage upload → `evidence_files` insert → audit event), or removes
   a file (`removeFile` → row delete → storage object delete). Allowed only while `in_progress`.
3. **Submit for review** — `sendForReview()` sets `awaiting_review` and logs the event.
4. **Recall** — `recallFromReview()` returns it to `in_progress`; permitted by the trigger for the
   assigned owner while the action is awaiting review. There is **no server-side check that the
   reviewer has not yet decided**, beyond the fact that an approved action is `completed` and can
   no longer be recalled by the owner.
5. **Return to owner** (reviewer) — requires a non-empty comment in the UI; sets `in_progress`,
   inserts the comment, logs "Returned to … for rework". *Discrepancy:* the comment requirement is
   enforced only in `use-review-decision.ts`, not in the database.
6. **Rework & resubmit** — the owner edits evidence again (the action is editable once more) and
   submits again; history accumulates in the audit trail.
7. **Approve & complete** (reviewer) — confirmation step with a required final comment; sets
   `completed`, inserts the comment, logs "Review outcome: Approved · action marked completed".
   Completed is the status; Approved is the recorded outcome in the audit text.
8. **Comments & audit** — comments are free text from owner/reviewer; audit events are written by
   the client on every workflow operation and cannot be updated or deleted by any app role.
   *Discrepancy vs. plan:* audit events are **client-written**, not generated by a database
   trigger, so an event row is only guaranteed to exist for operations the app performs.

Each mutation is a sequence of separate statements (status update, comment insert, event insert).
They are **not wrapped in a single transaction/RPC**, so a failure mid-sequence can leave a status
change without its comment or event.

---

## 8. Evidence Handling & Security

- **Bucket** `evidence`: private (`public = false`), size limit 26,214,400 bytes (25 MB), no
  allowed-MIME list configured at the bucket level.
- **Storage policies** on `storage.objects`: `evidence_read` (anyone who can read the parent
  action), `evidence_owner_insert` and `evidence_owner_delete` (assigned owner, action editable).
- **Path**: `${actionId}/${requirementId}/${crypto.randomUUID()}-${safeName}`; the original
  filename is sanitised to `[A-Za-z0-9._-]`, truncated to the last 120 characters, and the real
  display name is stored in `evidence_files.original_name`. Paths are never user-chosen.
- **Validation** (`evidence-storage.ts`, client-side): extension allow-list
  pdf/doc/docx/xls/xlsx/csv/txt/png/jpg/jpeg; browser MIME must match the extension; non-empty;
  ≤ 25 MB; magic-number check on the first bytes for PDF/PNG/JPEG/OOXML/OLE2; for txt/csv the first
  64 KB must decode as text with no binary control characters, and CSV must contain a `,` or `;`.
- **Download**: `createSignedUrl(path, 60, { download })` — a 60-second link, triggered by an
  anchor click. Files are never publicly reachable.
- **Limitations**: content validation runs **in the browser only** — a crafted direct PostgREST/
  Storage call bypasses it; the bucket enforces only the size limit. There is **no malware
  scanning**. A file whose bytes match an allowed signature but contains malicious content is
  accepted. Deleting an action cascades the `evidence_files` rows but leaves the storage objects
  behind (no cleanup job).

---

## 9. Authentication

- **Sign in**: email + password at `/auth` (`signInWithPassword`). **Sign out** from the avatar
  account menu.
- **Password reset**: "Forgot password" sends `resetPasswordForEmail` with a redirect to
  `/reset-password`, which calls `updateUser({ password })`. Delivery uses Supabase's built-in
  email sender; end-to-end delivery has **not been verified** in this project.
- **No public signup** — there is no sign-up screen and no self-service account creation. Accounts
  are created manually in the Supabase dashboard.
- **Mapping**: after sign-in, the app looks up `profiles` by `auth_user_id` and `user_roles` by
  `user_id`. If no profile matches, the UI falls back to the email address and the user has no
  ownership anywhere — a linking mistake shows up as "sees nothing".
- **Demo accounts**: two auth users exist — Dana Reyes (reviewer capability) and Priya Nair
  (action owner, no capability). Marcus Bell, Lena Okafor, Tomás Rivera and Aisha Khan are
  profiles without accounts. Passwords are managed by the project owner in Supabase and are not
  stored in the repo or in this document.

---

## 10. What's Solid vs. Duct Tape

Solid (real, server-backed)
- Supabase Auth email/password sign-in, session persistence, password-reset flow wiring.
- Postgres schema with FKs, indexes, enums and a CHECK on severity.
- RLS on every table plus the `enforce_action_update_rules` trigger — the substantive
  authorization model, including reviewer capability via `user_roles`.
- Private evidence bucket with matching storage policies and signed downloads.
- Real file upload/download of real documents; persistence survives refresh and sign-out.
- Append-only `action_events` (no update/delete path for app roles).
- The complete workflow, verified end to end by the product owner as both personas.

Duct tape / demo-only
- **One team** (`Controls & Assurance`) for all six people, so team-based read access is
  effectively "everyone reads everything".
- **Seeded people and actions** with hardcoded UUIDs and fixed dates; four profiles have no
  login.
- `MY_TEAM` in `control-action.ts` is a **hardcoded name array** used by the dashboard scope
  filter, and scope/ownership comparisons in the UI are done by **display name**, not profile id.
  Duplicate names would break the UI (the database is unaffected).
- Mutations are **multi-statement, not atomic**; no RPC wrapper.
- Audit events are **written by the client**, not by triggers.
- "Comment required" for return/approve is a **UI rule only**.
- Every read reloads **all** actions the user can see; no pagination, caching, realtime or
  optimistic updates.
- `src/integrations/supabase/client.server.ts`, `auth-middleware.ts`, `cron-auth.ts` are unused
  template files.
- Root `__root.tsx` still carries the default "Lovable App" metadata (leaf routes have proper
  titles).
- Three migration files dated `2026092111*` are leftovers from an earlier, discarded backend and
  were never applied to this project.
- No user administration UI, no notifications/email on assignment or decision, no tests, no
  monitoring.

---

## 11. How to Run & Maintain It

Run
```
bun install
bun run dev      # http://localhost:8080
bun run build    # production build
```

Configuration — `.env` (values not reproduced here):
`VITE_SUPABASE_URL`, `VITE_SUPABASE_PUBLISHABLE_KEY`, `VITE_SUPABASE_PROJECT_ID` (plus non-prefixed
duplicates). Only the publishable key is used; there is no service-role usage in app code.
`supabase/config.toml` pins `project_id = "tnfxxspxcsokdcozrrlf"`.

Supabase setup that must exist in any new environment: the schema/policy migration, the seed
migration, the `evidence` bucket with its three storage policies, and at least one auth user
linked to a profile with a `user_roles` row of `'reviewer'`.

Migrations that matter (in `supabase/migrations/`):
- `20260921133838_*` — schema, helpers, RLS, trigger, indexes, grants.
- `20260921134633_*` — profiles, roles, demo actions, requirements, files, comments, events.
- `20260921135800_*`, `20260921140147_*` — later fixes, including relinking Dana's profile to a
  recreated auth account.
- `2026092111*` (three files) — legacy, never applied here; do not treat as current state.

Where to change things
- UI/behaviour of a screen → `src/features/<feature>/`.
- Domain rules, statuses, derived due labels → `control-actions/model/control-action.ts`.
- Reads/queries → `control-actions/data/actions-repository.ts`.
- Workflow writes and audit text → `control-actions/state/control-action-store.ts`.
- Permissions → the SQL policies/trigger (and mirror any visibility change in
  `auth/components/reviewer-guard.tsx`).
- Evidence rules → `control-actions/data/evidence-storage.ts` + storage policies + bucket config.
- Session/roles → `auth/state/session-store.ts`.

---

## 12. Risks & Known Limitations

- **Client-side-only file validation and no malware scanning** — the strongest evidence-security
  gap. The bucket enforces size only.
- **Non-atomic workflow operations** — a network failure between the status update and the
  comment/event insert leaves inconsistent history.
- **Client-written audit trail** — trustworthy only insofar as writes come from this app; rows
  cannot be altered afterwards, but a direct API caller could skip writing one.
- **Comment-required rules not enforced in the database.**
- **Name-based comparisons in the UI** (owner/reviewer/team) instead of profile ids.
- **Single-team model** makes read access broad; real deployments need real teams before the
  team rule means anything.
- **Email**: built-in Supabase sender, low rate limits, unverified deliverability, no custom
  domain; no notification emails at all.
- **User administration is manual** — creating an account, linking `auth_user_id` and granting
  `reviewer` are all dashboard/SQL operations. Deleting an auth user sets `auth_user_id` to null
  and silently removes the person's access (this already happened once).
- **Legacy migrations** in the repo do not reflect this database.
- **No tests, no monitoring, no error tracking** beyond the built-in error boundary; no
  pagination or realtime, so the dashboard will degrade with volume.
- Storage objects are orphaned if an action is deleted.

Not verified from the implementation: password-reset email delivery, behaviour at scale,
cross-browser/mobile behaviour, and any performance characteristic.

---

## 13. Recommended Next Engineering Steps

1. **Move workflow transitions into Postgres RPCs** (`submit_action`, `recall_action`,
   `return_action`, `approve_action`) that update the status, insert the comment and write the
   audit event in one transaction, enforce the required comment, and enforce that a recall is only
   possible before a decision. Then reduce `control-action-store.ts` to RPC calls.
2. **Write audit events in triggers** rather than from the client.
3. **Harden evidence intake**: re-validate type/size server-side (Edge Function or a storage
   webhook), set `allowed_mime_types` on the bucket, and add a malware-scanning step before a file
   becomes visible.
4. **Replace name-based logic with profile ids** in `control-action.ts` and `use-needs-attention.ts`,
   and drop the hardcoded `MY_TEAM` in favour of the team on the profile.
5. **Real teams and user administration**: a `teams` table or at least seeded distinct teams, and a
   minimal admin surface for inviting users, linking profiles and granting the reviewer capability.
6. **Notifications** on assignment, submission and decision.
7. **Housekeeping**: delete the three legacy migrations and the unused
   `src/integrations/supabase/*.server/middleware/cron` files, fix the root metadata, add
   pagination, and add a first test pass around the workflow and RLS.

---

## 14. Engineering Verification Checklist

- [ ] **Auth** — sign in as the reviewer account and the owner account; sign out; refresh stays
      signed in; an unknown URL while signed out redirects to `/auth`.
- [ ] **Reviewer vs owner** — reviewer sees "New action" and "Reviewer queue"; owner sees neither
      and is redirected when typing `/actions/new` and `/review`.
- [ ] **Backend authorization** — as the owner, call the API directly: insert into
      `control_actions` (must fail), update another user's action (must fail), change
      `finding` on your own action (trigger must raise), move your action straight from
      `in_progress` to `completed` (must fail).
- [ ] **Action creation** — reviewer creates an action with requirements; new `ref` increments; it
      appears for the assigned owner.
- [ ] **Evidence** — upload an allowed file and confirm it lands in the bucket under
      `actionId/requirementId/uuid-name`; reject a renamed `.exe`, a >25 MB file, and a binary
      file renamed `.txt`; download as the reviewer via the signed link; remove a file as the
      owner.
- [ ] **Submit / recall** — owner submits, action shows Awaiting review and appears in the queue;
      owner recalls and can edit evidence again; both events appear in the trail.
- [ ] **Return / resubmit** — reviewer returns with a comment (blank comment blocked), owner
      reworks and resubmits.
- [ ] **Approve / complete** — approve with the required final comment; status Completed, comment
      recorded under the reviewer, audit line records the Approved outcome.
- [ ] **Comments & audit** — comments show the correct author; audit ordering is newest first;
      confirm no update/delete path exists for `action_events`.
- [ ] **Persistence** — refresh, sign out and back in, and in a second browser: all state
      (status, files, comments, history) is unchanged.
