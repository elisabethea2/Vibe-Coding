# Vantage Controls — Living PRD

*Updated 2026-09-21. Reflects the implemented application backed by the external Supabase project "Vantage Controls" (project ref `tnfxxspxcsokdcozrrlf`). Supersedes the earlier prototype PRD.*

## Product Overview

Vantage Controls is a lightweight internal control-action tracking tool designed to support the remaining control-action workflow currently handled by a legacy application being decommissioned.

It supports the lifecycle from creating and assigning a control action through evidence submission, review, rework and completion. It serves two primary roles: Controls & Audit / Reviewers, who create and review actions, and Action Owners, who understand, execute and provide evidence for assigned actions.

The current version is a working application, not a prototype. The UI and workflow are unchanged from the validated prototype, but every piece of state — users, actions, evidence checklists, uploaded documents, comments and the audit trail — now lives in Supabase (PostgreSQL, Auth and Storage) with access rules enforced server-side.

## Problem

The current control-action tracking workflow is supported by a legacy application that is being decommissioned. Most of the application's original use cases have already been moved to other platforms that are better suited to support them, leaving control-action tracking as one of the remaining use cases.

Maintaining the legacy application solely for this residual workflow is not a sustainable long-term solution. The product need is therefore to provide a lightweight, fit-for-purpose way to support the control-action lifecycle without depending on the legacy application.

Within that broader problem, the product tests the following hypothesis: **structuring every control action around a clear Finding, a concrete Recommendation, and an explicit list of Required Evidence helps Action Owners understand and complete assigned actions without additional clarification.**

The prototype validated this hypothesis as a design; the current build makes it operational with real authentication, persistent data and real evidence documents.

## Users & jobs (primary user, job to be done)

Users sign in with email and password (Supabase Auth). There is **no public self-sign-up** — accounts are created internally in the Supabase dashboard, and each account is linked to a person in the `profiles` table. Two accounts exist today:

| User | Role label | Primary job to be done |
| --- | --- | --- |
| **Dana Reyes** | Controls & Audit · Reviewer | Create control actions from findings, assign an owner and required evidence, review submitted evidence, and approve (→ completed) or return actions for rework. Also acts as an Action Owner on actions assigned to her. |
| **Priya Nair** | Action Owner | See the actions assigned to her, understand what is required (Finding / Recommendation / Required Evidence), upload evidence files, comment, and send the action for review. |

The role model:

- **Reviewer** is a system capability stored in `user_roles` (not on the person record). Dana holds it; it is what makes "Reviewer queue" and "+ New action" visible and what grants create/edit rights in the database.
- **Action Owner** is not a global role. Ownership rights derive per action from `control_actions.owner_id = the signed-in person's profile`. A reviewer can also be an owner; the two are not mutually exclusive.
- People without an account (Marcus Bell, Lena Okafor, Tomás Rivera, Aisha Khan) exist as profiles so the demo actions keep their original owners and reviewers; they simply cannot sign in. Adding a person later is a one-row change.

Current permission rules (enforced by database RLS policies plus a trigger, with matching UI rules):

- **Reviewer** sees and can access "Reviewer queue" and "+ New action". Action Owners do not see these options and are redirected to the dashboard if they open `/review` or `/actions/new` directly. The check derives from the real reviewer capability in Supabase — never from names, emails or labels — and waits for the session to load so a reviewer is never bounced on refresh.
- **Action Owner** can upload/remove evidence files and use Send for review / Recall from review **only on actions they own**, and only while the action is editable (not awaiting review, not completed).
- **Reviewer of a specific action** can edit that action's information — Finding, Recommendation, Action Owner, Reviewer, Control Area, Severity, Due Date — on both the Action Detail page and the Review Action page, for any non-completed action. The reviewer **cannot** add, replace, or delete the owner's evidence.
- Approve / Return to owner are available only to the action's reviewer and only while the action is "Awaiting review".
- Comments can be added by any signed-in user on both the Action Detail and Review Action pages; the audit trail is insert-only.

## User Flow & Screen Map

### Controls & Audit / Reviewer

Sign in → Dashboard → Create Action → Action Detail → Reviewer Queue → Review Action → Approve & Complete

or:

Reviewer Queue → Review Action → Return to Owner → Owner reworks and resubmits → Review Action → Approve & Complete

### Action Owner

Sign in → Dashboard → Action Detail → Review Finding, Recommendation and Required Evidence → Upload Evidence → Send for Review

If returned:

Action Detail → Review return comment → Update evidence / action work → Resubmit for Review

### Main screens

- **Sign in** (`/auth`) — email/password sign-in with "Forgot password"; `/reset-password` completes an emailed reset link.
- **Dashboard** — overview of open work, filtering, search and action preview.
- **Create Action** — creates and assigns a new control action.
- **Action Detail** — primary workspace for understanding and completing an action.
- **Reviewer Queue** — shows actions awaiting the signed-in reviewer's attention.
- **Review Action** — evidence inspection, action editing and review decision.

## Scope (in / out)

### In scope (implemented)

- **Authentication**: Supabase email/password; sign-in screen with forgot-password email (built-in Supabase sender); password reset screen; an auth gate that redirects signed-out visitors to `/auth`; account menu (avatar, top-right) with the user's name, role label and Sign out. Sessions persist across refreshes; no public sign-up.
- **Dashboard ("Needs attention")**: bento-grid summary (Open actions / Overdue / Due soon), scope switcher (My team / Mine), search box (title, owner, area), interactive "By control area" and "By severity" breakdowns that act as combinable filters with chip indicators and a Clear filters button, a master action list sorted by due date, and a detail preview panel. Completed actions are excluded from the dashboard list.
- **Action Detail page** (`/actions/:id`): status, severity, reference, owner, reviewer, control area, due date; Finding and Recommendation panels; Required Evidence list with per-item Uploaded/Missing state, file counts, expandable file lists, upload and remove (owner only, editable states only); Send for review (enabled only when all evidence is uploaded) / Recall from review; reviewer edit panel; audit trail (3 most recent events, expandable to full history); comments.
- **Create Action** (`/actions/new`, reviewer only): title, Finding, Recommendation, one or more Required Evidence requirements (add/remove rows), Owner, Reviewer, Control Area, Severity, Due Date (defaults to +14 days), Create / Cancel. Validation: required fields non-empty, at least one evidence requirement, owner ≠ reviewer. On create, the action is written to Supabase with the next sequential reference and a "created and assigned" audit event, and the user lands on its detail page.
- **Reviewer Queue** (`/review`, reviewer only): summary cards (Awaiting your review / Past due date / Approved), the "Submitted for review" list with reference, title, severity, owner, area, due date and evidence completion (e.g. "Evidence 2/2"), a "Completed after your approval" list, plus skeleton loading, empty ("No actions are waiting for your review.") and error ("We couldn't load the actions awaiting review. Try again." with retry) states.
- **Review Action** (`/review/:id`, reviewer only): read-only view of Finding, Recommendation, evidence (uploaded documents download through short-lived signed links for inspection), comments, audit trail; **Approve action** opens a confirmation step explaining that approving marks the action Completed, requires a final review comment, and offers "Approve & complete" / "Cancel"; **Return to owner** requires a mandatory comment; reviewer edit panel for action information.
- **Role-based navigation**: Reviewer queue and New action are visible/accessible only to users holding the reviewer capability.
- **Real evidence storage**: uploaded documents are stored in a private Supabase Storage bucket; validation (type, content, 25 MB) on upload; reviewer downloads via short-lived signed URLs; access derived from the action's ownership/reader rules, not the uploader.
- **Server-enforced workflow and append-only audit**: status transitions and metadata edits are checked in the database through RLS policies and a transition trigger. Audit events are written by the application client after workflow operations and are insert-only; they cannot be edited or deleted through app roles.
- **End-to-end workflow**: Create → assign → owner works the action (evidence, comments) → Send for review → Reviewer queue → Review → Approve (→ Completed) or Return to owner (→ rework, comment recorded) → owner resubmits → approval. Recall from review lets the owner pull a submitted action back before the reviewer decides.

### Out of scope (not implemented)

- Public self-sign-up or in-app user administration; accounts and the reviewer capability are managed directly in the Supabase dashboard.
- Notifications, emails (beyond the password-reset email), reminders, or escalation.
- Malware scanning of uploaded evidence (documented production consideration — see Risks).
- Multi-team management; the single team "Controls & Assurance" is a demo simplification, not a validated production assumption.
- Reopening, deleting, or archiving completed actions; editing evidence after completion; any post-completion lifecycle.
- Reporting, exports, due-date changes by owners, delegation, or reassignment workflows beyond the reviewer edit panel.
- Mobile-specific or accessibility-audited experiences (not assessed).

## Requirements

Priorities: **P0** = core workflow, **P1** = important supporting behavior, **P2** = polish. Acceptance criteria describe current observed behavior.

| # | Requirement | Priority | Acceptance criteria (current behavior) |
| --- | --- | --- | --- |
| R1 | Structured action record | P0 | Every action has title, Finding, Recommendation, ≥1 Required Evidence item, Owner, Reviewer, Control Area, Severity (Critical/High/Medium/Low), Due Date, status, and a sequential 3-digit reference (001, 002, …; new actions take max ref + 1) shown in a visually neutral grey circle that never encodes severity or status. |
| R2 | Dashboard needs-attention view | P0 | Dashboard shows Open actions, Overdue and Due soon counts for the active scope; the action list is sorted by due date and excludes completed actions; selecting a row highlights it and populates the detail panel. |
| R3 | Scope filtering | P1 | "My team" (default) shows actions owned by the team; "Mine" shows only actions owned by the signed-in user's profile. Switching scope resets selection and filters. |
| R4 | Interactive breakdowns | P1 | Control-area and severity breakdowns act as toggle filters; they combine; active filters appear as removable chips with a Clear filters button; severity counts recompute within the selected area; a dead-end combination shows an empty state with Clear filters. |
| R5 | Search | P2 | The search box filters the visible list by title, owner, or control area; a no-match state explains the search found nothing. |
| R6 | Action creation (reviewer only) | P0 | Create Action validates non-empty title/finding/recommendation, ≥1 evidence requirement, and owner ≠ reviewer; due date defaults to today +14 days; Create saves the action to the database with a "created and assigned" audit event and navigates to its detail page; Cancel discards. Only users with the reviewer capability can reach the screen. |
| R7 | Evidence management (owner only) | P0 | The owner can upload files (PDF/DOC/DOCX/XLS/XLSX/CSV/TXT/PNG/JPG/JPEG, max 25 MB) to each evidence requirement and remove individual files; uploading marks the item Uploaded; removing the last file returns it to Missing; evidence controls are hidden for non-owners and locked while the action is awaiting review or completed. Ownership of evidence rights follows the action's owner, not who uploaded a given file. |
| R8 | Send for review / recall | P0 | Send for review is enabled only when every evidence item is uploaded; it sets status to Awaiting review and records an audit event; while awaiting review the owner sees "Recall from review", which returns the action to its editable status — recall is allowed only before the reviewer's decision. |
| R9 | Reviewer queue (reviewer only) | P0 | The queue lists actions awaiting the signed-in reviewer's review with owner, area, severity, due date, and evidence completion; it shows skeleton rows while loading, "No actions are waiting for your review." when empty, and "We couldn't load the actions awaiting review. Try again." with a retry button on failure; completed-after-approval actions appear in a separate list. |
| R10 | Review decision: approve | P0 | "Approve action" never completes immediately; it opens a confirmation stating the review outcome is Approved and the status becomes Completed, requires a final review comment, and offers "Approve & complete" (disabled until a comment is entered) and "Cancel". On confirm: status → Completed, an audit event records the approval, the comment is added under the reviewer, and the user returns to the queue. |
| R11 | Review decision: return | P0 | "Return to owner" requires a mandatory comment; on confirm the action returns to its editable status, the comment is recorded under the reviewer, an audit event records the return, and the owner can rework and resubmit. |
| R12 | Evidence inspection by reviewer | P1 | In the review view, uploaded evidence documents are clickable and download through a short-lived signed URL for inspection before approving or returning; evidence name, Uploaded status, and file counts are preserved. |
| R13 | Reviewer editing of action information | P1 | The action's reviewer can edit Finding, Recommendation, Owner, Reviewer, Control Area, Severity, and Due Date on both the Action Detail and Review Action pages (any non-completed status); changes persist, each changed field is recorded in the audit trail, and the reviewer cannot modify owner-uploaded evidence. |
| R14 | Audit trail | P0 | Significant workflow and user events (creation, assignment, evidence upload/removal, send/recall, edits, comments and review outcome) append an actor + event + timestamp entry written by the application client after the operation; the trail is insert-only (no edits or deletes via app roles); the UI shows the 3 most recent with an expand toggle for the full trail.|
| R15 | Comments | P1 | Both detail and review pages support adding comments, shown with author and time; review-outcome comments (approval, return) are added automatically. |
| R16 | Role-based navigation | P0 | "Reviewer queue" and "+ New action" are visible and reachable only for users holding the reviewer capability in `user_roles`; others don't see them and are redirected to the dashboard on direct URL access. The check uses the real Supabase capability, never names/emails/labels, and tolerates page refreshes while the session loads. |
| R17 | Authentication & account menu | P0 | Email/password sign-in; forgot-password sends a reset email (built-in Supabase sender); signed-out users are redirected to the sign-in screen; the avatar menu shows the signed-in person, their role label, and Sign out; no public sign-up. |
| R18 | Date consistency | P1 | Due dates, overdue/due-soon labels, and statuses are computed relative to the current date; statuses derive from the due offset (overdue < 0 days, due soon ≤ 5 days) unless awaiting review or completed. |

## Success Metrics

The application has not been deployed to production, so no production performance metrics are currently available. The following metrics would be used to evaluate whether the product is supporting the intended workflow.

### Primary outcome

- **Action completion without clarification** — percentage of Action Owners who can understand what they need to do and what evidence they need to provide without requesting additional clarification.

### Leading indicators

- Percentage of actions submitted for review with all required evidence provided.
- Percentage of submitted actions approved without being returned for rework.
- Number of clarification requests per action.
- Time from action assignment to first submission for review.
- Time from submission to reviewer decision.
- Percentage of overdue open actions.

The primary outcome is directly tied to the validated product hypothesis around Finding, Recommendation and Required Evidence.

## Assumptions & Risks

### Current assumptions

- Action Owners can understand and act on a control action when Finding, Recommendation and Required Evidence are clearly structured.
- A single Action Owner and a single Reviewer are sufficient for the core workflow.
- Controls & Audit / Reviewers are responsible for creating actions and maintaining action metadata.
- Action Owners should control the evidence they submit, while Reviewers should control the action definition and review decision.
- The core create → execute → review → rework / complete lifecycle represents the remaining control-action tracking use case sufficiently.
- One team ("Controls & Assurance") covering all six people is acceptable for the demo; it is a simplification, not a validated production assumption.
- The built-in Supabase email sender is acceptable for password reset at demo volume (rate limits are low).
- Workflow atomicity and audit integrity: workflow mutations currently use separate client-side database operations rather than a single transaction/RPC. A failure between a status update, required comment and audit-event insert could leave incomplete history. Required approval/return comments are enforced in the UI rather than in the database.

### Current risks

- **Deployment validation risk** — the workflow has been validated as a demo; it has not yet run with real users at scale.
- **Malware scanning** — uploaded evidence is validated for extension, MIME type and content signature (magic-number check for binaries; text/format check for TXT/CSV) but is **not** scanned for malware. This is a documented production consideration; passing validation does not claim a file is safe.
- **Disguised files** — validation rejects reliably detectable disguised or mismatched types, but no claim is made that it catches every spoofing technique.
- **Email rate limits** — the built-in Supabase sender is not a production email service; reset emails are rate-limited and unstyled.
- **Legacy migration risk** — the data, evidence and historical records that may need to move from the legacy application have not yet been defined.
- **User administration** — creating users, linking profiles and granting the reviewer capability happen in the Supabase dashboard; there is no admin UI and no Super Admin role.

## Data & events

### Entities

Data lives in Supabase PostgreSQL. Row-Level Security enforces read/write access; helper functions (`has_role`, `my_profile_id`, `can_read_action`, `is_action_owner`, …) resolve the signed-in account to its linked profile once and apply owner/reviewer/team logic.

**Profile** — a person, not an account. `id`, `full_name`, `initials`, `team`, and an **optional** `auth_user_id` link to Supabase Auth. Actions reference profile ids; only linked profiles can sign in.

**User role** — `user_id` → auth.users, `role` ('reviewer'). The reviewer capability; owner rights do not live here.

**ControlAction** — `control_actions`.

| Field | Notes |
| --- | --- |
| `id` | UUID |
| `ref` | Sequential 3-digit display reference (`001`…); new actions take max ref + 1 |
| `title`, `finding`, `recommendation` | Free text; finding/recommendation required |
| `area` | One of six fixed control areas: Access management, Data security, Change management, Financial reporting, Vendor management, Finance operations |
| `owner_id`, `reviewer_id`, `created_by` | Foreign keys to **profiles** (people), not auth accounts |
| `due_date` | ISO date; overdue/due-soon derived relative to today |
| `status` | `in_progress` \| `awaiting_review` \| `completed` (database values; the UI additionally renders overdue/due-soon from the due date). Review outcome "Approved" is distinct from status "Completed". |
| `severity` | Critical / High / Medium / Low (CHECK constraint) |

**EvidenceRequirement** — `evidence_requirements`: one row per required item per action; the UI renders Uploaded/Missing from whether attached files exist.

**EvidenceFile** — `evidence_files`: `storage_path` (unique, generated `actionId/requirementId/uuid-name`), `original_name`, `mime_type`, `size_bytes`, `uploaded_by` → profiles. The binary lives in the private Storage bucket "evidence" (25 MB limit).

**ActionComment** — `action_comments`: author, text, timestamp; insert-only.

**ActionEvent** — `action_events`: actor, event text, timestamp; insert-only, written server-side on every workflow change.

### State changes (events recorded in the audit trail)

- `Action created and assigned to <owner>` (on create)
- `Uploaded evidence: <name> (<files>)` / `Removed evidence file: <file> (<name>)`
- `Sent for review to <reviewer>` / `Recalled from review`
- `Review outcome: Approved · action marked completed` / `Returned to <owner> for rework`
- `Action details updated: <changed fields>` (reviewer edits)
- `Commented on the action`

### User actions driving those changes

Sign in/out; dashboard filtering/search/scope; create; evidence upload/remove; send for review; recall; approve (with required final comment); return (with required comment); reviewer field edits; commenting.

### Storage reality

- **Database**: Supabase PostgreSQL (external project "Vantage Controls"), accessed through the generated Supabase client; all reads/writes honor RLS as the signed-in user.
- **Auth**: Supabase Auth (email/password); sessions persist client-side; password reset via the built-in email sender.
- **Files**: private Supabase Storage bucket; downloads are time-limited signed URLs; server-side policies mirror the action read rules.
- **Seed data**: six people, eight demo actions (refs 001–008, seeded with fixed dates around 2026-09-21, original owner/reviewer assignments preserved), 17 evidence requirements, eight pre-attached placeholder PDFs in the bucket, comments and the corresponding audit history.
- Nothing is held only in the browser: refreshing, switching browsers or signing out/in shows the same data.

## Open questions

- **Legacy transition**: the migration approach from the legacy application is not defined — which actions, evidence, and audit history to retain or migrate is still unknown.
- **Notification expectations**: nothing tells an owner an action was assigned or returned, or a reviewer that something awaits review. Whether and how users are notified is unclear.
- **Post-completion lifecycle**: completed actions drop off the dashboard but remain reachable; whether they can be reopened, archived, or reported on is undefined.
- **Team and control-area administration**: the single team and the six control areas are fixed. Who manages these lists in a real deployment is unclear.
- **Due-date governance**: currently only the reviewer can change a due date; whether owners can request changes, and whether overdue status should require escalation, is unresolved.
- **Multiple reviewers / delegation**: an action has exactly one reviewer; coverage for absence or delegation is not modeled.
- **Production hardening**: malware scanning, a production email service for reset emails, in-app user administration, and a decision on whether the one-team simplification should remain are all open.
- **Retention & access policy**: how long evidence and audit history are kept, and who may see the full trail beyond the current rules, is undecided.
