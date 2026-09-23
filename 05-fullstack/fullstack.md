# Full-Stack: Data, Access Rules, Edge Cases, Deploy

> Module 5 · Full-Stack. Add data schemas, access rules, and edge cases; stress-test and deploy.

## Deployed link

https://vantage-controls.lovable.app
_____

## Data schema

| Entity | Key fields | Notes |
|---|---|---|
| `profiles` | `id`, `auth_user_id` (optional), `full_name`, `initials`, `team` | Person record; not every person has an auth account. Only linked profiles can sign in. |
| `user_roles` | `user_id` → `auth.users`, `role` | Stores the `reviewer` capability; each user can read only their own rows. |
| `control_actions` | `id`, `ref` (unique, e.g. 001), `title`, `area`, `severity`, `finding`, `recommendation`, `owner_id`, `reviewer_id`, `created_by`, `due_date`, `status` | Status enum: `in_progress`, `awaiting_review`, `completed`. Ownership is a per-action relationship, not a global role. |
| `evidence_requirements` | `id`, `action_id`, `name`, `position` | Reviewer-defined evidence checklist items per action. |
| `evidence_files` | `id`, `requirement_id`, `storage_path` (unique), `original_name`, `mime_type`, `size_bytes`, `uploaded_by` | Metadata for uploaded documents; the file itself lives in Storage. |
| `action_comments` | `id`, `action_id`, `author_id`, `body`, `created_at` | Persistent comments; insert-only (no edit/delete). |
| `action_events` | `id`, `action_id`, `actor_id`, `actor_label`, `description`, `created_at` | Immutable audit trail; insert-only (no edit/delete). |
| Storage bucket `evidence` | Private; paths `action-id/requirement-id/uuid-filename` | 25 MB limit; access governed by storage policies tied to action access. |

## Access rules

All authorization is enforced by Supabase RLS in PostgreSQL. Every query runs as the signed-in user, and the UI checks only control visibility. A signed-in user can read an action if they hold the reviewer capability, are the assigned owner or reviewer, or are on the same team as the action's owner (single team "Controls & Assurance" in the current seed). Any signed-in user who can read an action can also comment on it. The database verifies that the comment's `author_id` matches the signed-in user's own profile, and comments remain insert-only with no edit or delete access. Assigned Action Owners can add and remove evidence files while the action is editable (not after completion), send it for review, and recall it while awaiting review. Reviewers can additionally create actions, edit action details (finding, recommendation, owner, reviewer, control area, severity, due date), manage evidence requirements, and return or approve actions. Signed-out users can access only the sign-in and password-reset screens. No data is fetched and no results are cached without a resolved session. Profile records are readable by their owner and by members of the same team. Role records are readable only by their own user, with reviewer capability resolved through security-definer database functions that bypass these read rules.

## Edge cases hardened

| Case | Before | After |
| --- | --- | --- |
| Empty / first-run state | Dashboard and reviewer queue rendered empty results with no explanation; a signed-out fetch could remain cached after sign-in. | Dashboard and reviewer queue show explicit empty states such as "No matching actions" or "No actions are waiting for your review." Data loads after sign-in and clears on sign-out. |
| Bad / malicious input | Unsupported evidence files such as HTML were rejected, but the UI failed silently with no explanation. | Uploads are validated for type, extension, MIME match, size and file signature, with clear messages for unsupported, oversized or invalid files. |
| Failure / offline | Failed operations could fail silently or appear to hang; loss of connectivity was not detected. | Connectivity is detected app-wide with an offline banner and offline-aware navigation. The last valid user identity is preserved during connectivity failures and refreshed automatically when connectivity returns. Network, session, permission and server/database failures are handled separately with appropriate messages. |

## Stress test results


### Stress Test 1 — Kill Switch / Offline Recovery

**Test:**  
Switched the browser network to Offline using DevTools while viewing an Action Detail page, then restored the connection to test whether the application recovered correctly without a reload.

**Initial result:**  
The application correctly detected the offline state and blocked internal navigation. However, two recovery issues were identified:

1. A failed profile and permission refresh while offline could replace the valid in-memory profile ID with an empty identity. After connectivity was restored, the Supabase session remained valid but some write operations could fail.
2. Operation errors were not classified correctly. An authorization failure could therefore be displayed as a connectivity problem.

**Fix:**  
The application now preserves the last valid identity when a profile or permission refresh fails and automatically revalidates the session and refreshes the user's profile and permissions when connectivity returns.

Write operations are blocked while identity is unresolved, and failures are classified as connectivity, session, permission or general operation errors instead of treating every failure as an offline error.

Internal navigation also remains inside Vantage Controls while offline and displays the existing connectivity message rather than falling through to the browser's native offline page.

**Retest:**  
With the Action Detail page open:

1. Switched DevTools from No throttling to Offline.
2. Confirmed the offline banner and blocked navigation.
3. Switched browser tabs while offline to trigger the identity refresh scenario.
4. Restored Network to No throttling.
5. Added a comment on the same Action Detail page without refreshing or navigating away.
6. Confirmed the application recovered and the operation completed normally.

**Result:** PASS

**Known limitation:**  
Full page reloads, manually entered URLs, new tabs and browser back/forward navigation while offline are not supported because offline caching is intentionally not implemented.

---

### Stress Test 2 — Spam Click

**Test:**  
Entered the comment "Testing Spam clicking" and rapidly clicked Send repeatedly before the request completed.

**Expected:**  
Only one request should be processed. Repeated clicks should not create duplicate comments.

**Observed:**  
After the first click, the Send button changed to "Sending" and could not trigger additional submissions while the request was in progress. Only one comment was created.

**Result:** PASS

**Protection observed:**  
Mutation controls use an in-progress state to prevent duplicate submissions.

_____
