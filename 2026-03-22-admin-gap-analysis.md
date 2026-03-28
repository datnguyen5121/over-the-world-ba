# Admin Panel Gap Analysis
# Over The World — Business Analyst Review

> **Document Type:** Gap Analysis — Admin Panel Functions  
> **Date:** March 22, 2026  
> **Author:** Business Analyst  
> **Audience:** Development Team, Project Management  
> **Status:** Draft v1.0

---

## Table of Contents

1. [Analysis Scope](#1-analysis-scope)
2. [Currently Implemented Admin Functions](#2-currently-implemented-admin-functions)
3. [Gap Analysis — Missing or Incomplete Functions](#3-gap-analysis--missing-or-incomplete-functions)
4. [Priority Summary Table](#4-priority-summary-table)
5. [Recommended Implementation Roadmap](#5-recommended-implementation-roadmap)

---

## 1. Analysis Scope

This document reviews the **Admin side of the Over The World platform** to identify:
- Which core admin functions are already present and operational.
- Which functions are missing, incomplete, or only partially scaffolded.
- The business value and priority of each gap.

**Admin panel URL base:** `/admin/*` (React SPA)  
**Admin API base:** `http://localhost:3333` (NestJS REST API)  
**Admin roles in scope:** `ADMIN`, `TEACHER` (where applicable)

---

## 2. Currently Implemented Admin Functions

The following functions are confirmed present and active in both the frontend (admin panel) and backend (REST API).

| # | Function | FE Route | BE Endpoint(s) | Role Access | Status |
|---|---|---|---|---|---|
| 1 | **User Account Management** | `/admin/account-management` | `GET/POST/PATCH/DELETE /users` | ADMIN | ✅ Complete |
| 2 | **User Account Detail** | `/admin/account-management/:id` | `GET /users/:id` | ADMIN | ✅ Complete |
| 3 | **Bulk Delete Users** | `/admin/account-management` | `DELETE /users/multiple` | ADMIN | ✅ Complete |
| 4 | **Category Management** | `/admin/category-management` | `GET/POST/PATCH/DELETE /category` | ADMIN, TEACHER | ✅ Complete |
| 5 | **Bulk Delete Categories** | `/admin/category-management` | `DELETE /category/multiple` | ADMIN, TEACHER | ✅ Complete |
| 6 | **Vocabulary (Word) Management** | `/admin/word-management` | `GET/POST/PATCH/DELETE /vocabularies` | ADMIN, TEACHER | ✅ Complete |
| 7 | **Vocabulary Image Upload** | `/admin/word-management` | `POST /supabase/upload` | ADMIN, TEACHER | ✅ Complete |
| 8 | **Vocabulary Category Tagging** | `/admin/word-management` | `POST /vocabularies` (with categoryIds[]) | ADMIN, TEACHER | ✅ Complete |
| 9 | **Bulk Delete Vocabulary** | `/admin/word-management` | `DELETE /vocabularies/multiple` | ADMIN | ✅ Complete |
| 10 | **User Avatar Upload** | `/admin/account-management` | `POST /supabase/upload` | ADMIN | ✅ Complete |
| 11 | **Paginated & Searchable Lists** | All admin pages | All `GET` list endpoints | ADMIN, TEACHER | ✅ Complete |
| 12 | **JWT Authentication** | `/auth/login` | `POST /auth/login`, `/auth/refresh`, `/auth/logout` | All | ✅ Complete |
| 13 | **Role Guard on Write Operations** | — | `RolesGuard` on category/vocab write | ADMIN, TEACHER | ✅ Complete |

### Observations on Current Implementation

- **Learning Plan Management** has a full backend (CRUD + search + bulk delete), but **no frontend admin route exists** for it. It is completely absent from the admin panel UI.
- **Example Sentence Management** has an entity and `ExampleModule` registered, but has **no controller, service, or frontend UI**.
- **Progress & Task** are database scaffolds only — no API or UI.
- Several user and vocabulary **write endpoints have no authentication guard**, which is a security risk (noted in BA_REQUIREMENTS.md SEC-10).

---

## 3. Gap Analysis — Missing or Incomplete Functions

---

### GAP-01: Admin Dashboard / Analytics Overview

- **Function Name:** Admin Dashboard
- **Description:** A landing page for the admin panel displaying key platform metrics at a glance: total users by role, total vocabulary words, total categories, total learning plans, recent registrations, most-studied words, and active user count. May include charts (line graph for registrations over time, pie chart for role distribution).
- **Business Value:** Without a dashboard, admins must navigate to each management page individually to understand the state of the platform. A dashboard provides instant situational awareness, helps identify growth trends, and allows PMs to track platform adoption without querying the database directly.
- **Priority:** **High**
- **Suggested Inputs:** Date range filter (last 7 days / 30 days / custom)
- **Suggested Outputs:**
  - Total counts: users, vocabulary, categories, learning plans
  - User role breakdown (pie/donut chart)
  - New registrations over time (line chart)
  - Recently added vocabulary (last 5 words)
  - Recently registered users (last 5 users)
- **Backend Work Required:** New `GET /admin/stats` endpoint aggregating counts and time-series data. No new DB tables needed — queries only.
- **Frontend Work Required:** New `/admin/dashboard` route + chart library integration (recommend `recharts` or `@mui/x-charts`).

---

### GAP-02: Role & Permission Management

- **Function Name:** User Role Assignment
- **Description:** Admins must be able to **change a user's role** (e.g., promote a `STUDENT` to `TEACHER`, or demote an account) directly from the admin panel. Currently, role is set at account creation but cannot be changed from the UI — only via direct API call or database edit.
- **Business Value:** As the platform grows, teachers need to be onboarded and their permissions elevated without requiring a developer to intervene. This is a day-one operational need for any real deployment.
- **Priority:** **High**
- **Current Gap:** The `PATCH /users/:id` endpoint accepts `role` in the body, so the backend already supports this. The frontend `AppAccountDialog` update form needs to expose the `role` field as an editable dropdown/select.
- **Business Rules:**
  - Only `ADMIN` role can change another user's role.
  - An admin cannot change their own role to a lower-privileged role (prevents accidental lockout).
  - Role changes should be logged (see GAP-04 Audit Logs).
- **Backend Work Required:** None (already supported). Add guard to ensure only `ADMIN` can update role field.
- **Frontend Work Required:** Add `role` select field to the update account form (`updateAccountFormSchema` + `AppAccountDialog`).

---

### GAP-03: Learning Plan Management (Admin UI)

- **Function Name:** Learning Plan Management
- **Description:** A full admin UI for managing learning plans — list, create, update, delete, and assign plans to users. The backend is **fully implemented** (CRUD, search, pagination, bulk delete), but the admin panel has **no corresponding frontend route** at all.
- **Business Value:** Learning plans are a core domain entity that defines structured study paths for students. Without admin UI, teachers and admins cannot create or manage study curricula, making the learning plan domain unused from an operational standpoint.
- **Priority:** **High**
- **Current Gap:** `over-the-world-fe/src/routes/admin/` has no `learning-plan-management/` route. No mutation services, no form schema, no dialog components exist for learning plans.
- **Affected Backend Endpoints (already implemented):**
  - `GET /learning-plans` — paginated list with search
  - `GET /learning-plans/:id` — detail
  - `GET /learning-plans/user/:userId` — by user
  - `POST /learning-plans` — create
  - `PATCH /learning-plans/:id` — update
  - `DELETE /learning-plans/:id` and `DELETE /learning-plans/multiple` — delete
- **Frontend Work Required:**
  - New route: `/admin/learning-plan-management/`
  - `AppLearningPlanDialog` component (create/edit form)
  - Mutation services: `useCreateLearningPlanService`, `useUpdateLearningPlanService`, `useDeleteLearningPlanService`, `useDeleteLearningPlanMultipleService`
  - Query services: `useGetLearningPlanListService`
  - Yup schema: `createLearningPlanFormSchema`
  - Add navbar link in `AppAdminNavbar`

---

### GAP-04: Audit Log / Activity History

- **Function Name:** Admin Audit Log
- **Description:** A read-only log of all significant admin actions: who did what, on which record, and when. Examples: "Admin john_doe deleted user student_123", "Teacher anna created vocabulary 你好", "Admin john_doe changed user's role from STUDENT to TEACHER".
- **Business Value:** Audit logs are a **compliance and accountability requirement** for any multi-admin system. Without them, there is no way to trace which admin deleted a record, investigate disputes, or meet data governance obligations. This becomes critical once the platform has real users and data.
- **Priority:** **High**
- **Business Rules:**
  - Log entries must be **immutable** — no update or delete operations on audit logs.
  - Logs must capture: `actorId` (who), `action` (CREATE/UPDATE/DELETE), `entityType` (User/Vocabulary/Category/LearningPlan), `entityId`, `payload` (snapshot of changed fields), `timestamp`, `ip`.
  - Only `ADMIN` role can view the audit log.
  - Logs must be retained for a minimum of 90 days.
- **Backend Work Required:**
  - New `AuditLog` entity and table.
  - NestJS interceptor or service method to write a log entry on every state-changing operation.
  - New `GET /admin/audit-logs` endpoint (paginated, filterable by actor, entity type, date range).
- **Frontend Work Required:**
  - New `/admin/audit-logs` route with a read-only, filterable `AppTableDataGrid`.

---

### GAP-05: Example Sentence Management

- **Function Name:** Example Sentence Management
- **Description:** Admins and teachers can add, edit, and delete example sentences linked to vocabulary words. The `Example` entity and `ExampleModule` are registered in the backend, but there is **no controller, no service, and no API endpoints** — and no frontend UI.
- **Business Value:** Example sentences are critical for language learning effectiveness. Learners understand how a word is used in context, not just its isolated meaning. Without a management interface, example sentences cannot be populated, making vocabulary entries less educationally valuable.
- **Priority:** **High**
- **Current Gap:** `ExampleModule` is registered in `app.module.ts` but `example.controller.ts` and `example.service.ts` only have scaffolded stubs with no real methods.
- **Suggested Endpoints:**
  - `GET /examples?vocabularyId=` — list examples for a word
  - `POST /examples` — create example sentence
  - `PATCH /examples/:id` — update sentence
  - `DELETE /examples/:id` — delete sentence
- **Frontend Work Required:**
  - Example sentences section embedded in the vocabulary detail/edit dialog (`AppWordDialog`) as a sub-management table.
  - Not necessarily a standalone route — can be a tabbed section within word management.

---

### GAP-06: Password Reset / Account Recovery ✅ IMPLEMENTED (2026-03-22)

- **Function Name:** Password Reset (Admin-Triggered & Self-Service)
- **Description:** Two sub-functions: (1) An admin can force-reset a user's password from the admin panel. (2) A user can request a password reset via email (forgot password flow) on the login page.
- **Business Value:** Password reset is a fundamental user management feature. Without it, a user who forgets their password has no recovery path — they must contact a developer to manually update the database. This creates both a scaling problem and a poor user experience.
- **Priority:** **High**
- **Business Rules:**
  - Admin-triggered reset: Admin enters a new password for the user; system hashes and saves it. All existing refresh tokens for that user must be revoked.
  - Self-service reset: User submits email → system sends a time-limited reset link (valid 60 minutes) → user clicks link → sets new password. Requires email sending capability (SMTP via nodemailer).
  - Reset tokens must be single-use and expire.
- **Backend Work Required:**
  - Admin reset: `PATCH /users/:id/reset-password` (admin only).
  - Self-service: `POST /auth/forgot-password`, `POST /auth/reset-password` endpoints + email integration.
  - `PasswordResetToken` entity (token hash, userId, expiresAt, used flag).
- **Frontend Work Required:**
  - Admin panel: "Reset Password" action button in account detail.
  - Login page: "Forgot password?" link + email submission form + new-password form.
- **Implementation Details:**
  - `PasswordResetToken` entity: `src/auth/entities/password-reset-token.entity.ts` (SHA-256 hashed token, 60-min TTL, single-use)
  - `MailService`/`MailModule`: `src/mail/` (nodemailer SMTP, reads `SMTP_HOST/PORT/USER/PASS/FROM` + `FRONTEND_URL` env vars)
  - `AdminResetPasswordDto`: `src/users/dto/admin-reset-password.dto.ts`
  - `ForgotPasswordDto` / `ResetPasswordDto`: `src/auth/entities/`
  - FE routes: `/auth/forgot-password`, `/auth/reset-password?token=`
  - FE component: `AppResetPasswordDialog` (admin panel account detail)
  - FE services: `useAdminResetPasswordService`, `useForgotPasswordService`, `useResetPasswordService`
  - Edit + Delete buttons on account detail page also fixed (were `console.log` stubs)

---

### GAP-07: System Configuration / Settings

- **Function Name:** Admin System Settings
- **Description:** A settings page where admins can configure platform-level options without code changes: default pagination page size, allowed image file types and max size, platform display name, maintenance mode toggle, feature flags (e.g., enable/disable student registration).
- **Business Value:** Hardcoded configuration values (currently embedded in code and `.env`) cannot be changed at runtime without a redeployment. A settings panel gives non-developer admins control over operational parameters and reduces developer involvement for routine configuration changes.
- **Priority:** **Medium**
- **Business Rules:**
  - Only `ADMIN` role can view and modify system settings.
  - Settings must be persisted in a database table (not just environment variables) to survive server restarts.
  - A change log of settings updates should be recorded (links to GAP-04 Audit Log).
- **Backend Work Required:**
  - New `SystemSetting` entity (key, value, description, updatedBy, updatedAt).
  - `GET /admin/settings` and `PATCH /admin/settings` endpoints.
- **Frontend Work Required:**
  - New `/admin/settings` route with a form-based settings editor.

---

### GAP-08: Content Moderation / Vocabulary Approval Workflow

- **Function Name:** Vocabulary Approval Workflow
- **Description:** A submission workflow where `TEACHER` role users submit vocabulary entries for review, and `ADMIN` users approve or reject them before they go live in the catalog. Currently, any teacher can publish vocabulary immediately with no review step.
- **Business Value:** On a shared platform with multiple teachers, unreviewed content risks quality issues — incorrect pinyin, misleading meanings, inappropriate images. An approval gate ensures content quality and gives admins editorial control over what students see.
- **Priority:** **Medium**
- **Business Rules:**
  - New `status` field on `Vocabulary`: `DRAFT`, `PENDING_REVIEW`, `APPROVED`, `REJECTED`.
  - Teachers submit words in `DRAFT` status; they submit for review → `PENDING_REVIEW`.
  - Only admins can `APPROVE` or `REJECT`.
  - Only `APPROVED` vocabulary appears in the student-facing catalog.
  - Rejection must include an optional reason/comment.
- **Backend Work Required:**
  - Add `status` enum column to `Vocabulary` entity + migration.
  - `PATCH /vocabularies/:id/approve` and `PATCH /vocabularies/:id/reject` endpoints (admin only).
  - Filter student list endpoint to only return `APPROVED` entries.
- **Frontend Work Required:**
  - New `/admin/moderation` route listing `PENDING_REVIEW` vocabulary.
  - Approve / Reject action buttons with reject-reason modal.

---

### GAP-09: OAuth / Social Login Configuration

- **Function Name:** OAuth Provider Management
- **Description:** Admin UI to enable/disable and configure social login providers (Google, Facebook). Currently the OAuth strategies are **defined in code** (`google.strategy.ts`, `facebook.strategy.ts`) but not wired to any controller routes and not surfaced in the admin panel.
- **Business Value:** OAuth login significantly lowers the registration friction for students. Having it blocked behind code deployment means it cannot be activated for a real launch without developer involvement. An admin toggle provides operational flexibility.
- **Priority:** **Medium**
- **Business Rules:**
  - Google and Facebook OAuth callback routes must be implemented in `AuthController` before this UI makes sense.
  - OAuth strategies must be added to `AuthModule` providers.
  - Admin should be able to enable/disable each provider without code changes.
- **Backend Work Required:**
  - Wire `GoogleStrategy` and `FacebookStrategy` into `AuthModule`.
  - Add `GET /auth/google`, `GET /auth/google/callback`, `GET /auth/facebook`, `GET /auth/facebook/callback` routes.
  - Optionally: `SystemSetting` entries for `oauth.google.enabled`, `oauth.facebook.enabled` (links to GAP-07).
- **Frontend Work Required:**
  - OAuth enable/disable toggles in the system settings page (GAP-07).
  - Social login buttons on `/auth/login`.

---

### GAP-10: Active Session Management

- **Function Name:** User Session Management
- **Description:** A view where admins can see all active (non-revoked, non-expired) refresh token records for any user, with device info (IP and user agent), and individually revoke sessions. This allows admins to forcibly log out a user from all or specific devices.
- **Business Value:** Essential for security incident response — if an account is suspected compromised, the admin needs to terminate all sessions immediately. Currently, the only way to do this is to call `revokeAllForUser()` indirectly through the logout endpoint, which requires knowing the user's own credentials.
- **Priority:** **Medium**
- **Business Rules:**
  - Only `ADMIN` can view or revoke other users' sessions.
  - Session list must show: device IP, user agent, created date, expires date.
  - "Revoke all sessions" action must set `revoked = true` on all non-expired tokens for the user.
- **Backend Work Required:**
  - `GET /admin/users/:id/sessions` — list active refresh tokens for a user.
  - `DELETE /admin/users/:id/sessions` — revoke all sessions.
  - `DELETE /admin/users/:id/sessions/:jti` — revoke a specific session.
- **Frontend Work Required:**
  - Sessions tab/section in the account detail page (`/admin/account-management/:id`).

---

### GAP-11: Reporting & Data Export

- **Function Name:** Data Export / Reports
- **Description:** Admins can export platform data to CSV or Excel format: user lists, vocabulary catalogs, learning plan summaries, user vocabulary progress snapshots. May also include printable summary reports (PDF) for PM presentations.
- **Business Value:** PMs and stakeholders need periodic data snapshots for reporting to management, identifying trends, or off-platform analysis. Database access should not be required for routine data exports.
- **Priority:** **Medium**
- **Business Rules:**
  - Export must respect the current search/filter state (export what you see).
  - Exports are generated asynchronously for large datasets (>1000 rows) and delivered as a downloadable file.
  - Only `ADMIN` can export user data (contains PII).
  - Vocabulary exports can be available to `TEACHER` role.
- **Backend Work Required:**
  - `GET /admin/export/users?format=csv`, `GET /admin/export/vocabularies?format=csv` endpoints.
  - Use a library like `json2csv` or `exceljs` server-side.
- **Frontend Work Required:**
  - "Export CSV" button on each admin management page.

---

### GAP-12: Notification / Announcement Management

- **Function Name:** Platform Announcements
- **Description:** Admins can compose and publish announcements or notifications to all users, users of a specific role (e.g., all students), or individual users. Examples: "New vocabulary pack added", "Maintenance scheduled for Sunday 2am".
- **Business Value:** Without a notification system, admins have no channel to communicate with users from within the platform. This forces out-of-band communication (email, social media), which is inefficient and easily missed.
- **Priority:** **Low**
- **Business Rules:**
  - Notifications may be: in-app (banner/toast on login), email, or both.
  - Admins can schedule notifications for a future time.
  - Notifications have a `read` status per user.
- **Backend Work Required:**
  - New `Notification` entity (title, body, targetRole, targetUserId, scheduledAt, sentAt).
  - Email integration (SMTP or Supabase Email).
  - `GET /notifications` for users; `POST /admin/notifications` for admins.
- **Frontend Work Required:**
  - Notification bell in `AppAdminHeader` (currently shows a badge icon with no data).
  - `/admin/notifications` management page.
  - User-facing notification panel (future student UI).

---

### GAP-13: Storage / Media Management

- **Function Name:** Supabase Storage Management
- **Description:** A view that lists all files stored in Supabase Storage buckets (`vocab-images`, `user-avatar-image`), shows storage usage, identifies orphaned files (uploaded but not referenced in the database), and allows bulk deletion of orphaned assets.
- **Business Value:** Over time, especially given the current rollback pattern (upload first, then DB write — rollback only on failure), there is a risk of orphaned files accumulating in Supabase Storage if any edge cases in the rollback logic fail silently. A storage audit tool prevents unnecessary storage costs and keeps the system clean.
- **Priority:** **Low**
- **Business Rules:**
  - Orphan detection: compare file list from Supabase with `imageUrl` / `avatar` values in the database. Files not referenced by any record are orphans.
  - Delete operations on storage files must also check the DB — do not delete files still referenced.
  - Only `ADMIN` can access storage management.
- **Backend Work Required:**
  - `GET /admin/storage/stats` — bucket usage summary.
  - `GET /admin/storage/orphans` — list of unreferenced files.
  - `DELETE /admin/storage/orphans` — bulk remove orphaned files.
- **Frontend Work Required:**
  - `/admin/storage` route with storage usage card and orphan list.

---

### GAP-14: Admin Profile Management

- **Function Name:** Admin Self-Profile Management
- **Description:** Any authenticated user (admin or teacher) can view and update their own profile information (username, email, avatar) and change their own password from within the admin panel. Currently `/users/profile` returns the JWT payload but there is no edit UI for the logged-in admin.
- **Business Value:** Basic self-service profile management is a standard expectation of any authenticated system. Without it, admins must ask other admins to update their own account — a poor UX and an operational inefficiency.
- **Priority:** **Low**
- **Business Rules:**
  - Users can only edit their own profile via this route (not other users' profiles).
  - Password change requires providing the current password for confirmation.
  - Avatar change triggers the same Supabase upload + DB rollback pattern used elsewhere.
- **Backend Work Required:**
  - `PATCH /users/me` (self-update, authenticated by JWT — no admin role required).
  - `PATCH /users/me/password` (current password verification + re-hash).
- **Frontend Work Required:**
  - `/admin/profile` route or a profile drawer/modal accessible from the user avatar menu in `AppAdminHeader` (currently the menu shows "Profile" and "Settings" items that lead nowhere).

---

## 4. Priority Summary Table

| Gap ID | Function Name | Priority | Backend Effort | Frontend Effort | Dependency |
|---|---|---|---|---|---|
| GAP-01 | Admin Dashboard / Analytics | **High** | Medium | Medium | None |
| GAP-02 | Role & Permission Management | **High** | Low | Low | None |
| GAP-03 | Learning Plan Management (UI) | **High** | None (BE done) | Medium | None |
| GAP-04 | Audit Log / Activity History | **High** | High | Medium | None |
| GAP-05 | Example Sentence Management | **High** | Medium | Medium | GAP-03 (if embedded) |
| GAP-06 | Password Reset / Account Recovery | **High** | Medium | Medium | Email service |
| GAP-07 | System Configuration / Settings | **Medium** | Medium | Low | None |
| GAP-08 | Content Moderation / Approval Workflow | **Medium** | Medium | Medium | None |
| GAP-09 | OAuth / Social Login Configuration | **Medium** | Medium | Low | GAP-07 |
| GAP-10 | Active Session Management | **Medium** | Low | Low | None |
| GAP-11 | Reporting & Data Export | **Medium** | Medium | Low | None |
| GAP-12 | Notification / Announcement Management | **Low** | High | High | Email service |
| GAP-13 | Storage / Media Management | **Low** | Medium | Medium | None |
| GAP-14 | Admin Profile Management | **Low** | Low | Low | None |

---

## 5. Recommended Implementation Roadmap

Based on business value, effort, and dependencies, the following phased approach is recommended:

### Phase 1 — Critical Gaps (Address Before First Real User Onboarding)

| Order | Gap | Rationale |
|---|---|---|
| 1 | **GAP-02** Role Assignment | Low effort, already has BE support. Needed immediately for teacher onboarding. |
| 2 | **GAP-03** Learning Plan Admin UI | BE is fully done. Add the FE routes and hooks — medium effort, high return. |
| 3 | **GAP-05** Example Sentence Management | Core content quality feature. Language learning is ineffective without examples. |
| 4 | **GAP-06** Password Reset | Users will forget passwords. No recovery path = user churn. |
| 5 | **GAP-01** Admin Dashboard | First thing admins look for. Establish situational awareness. |

### Phase 2 — Operational Improvements (Next Sprint Cycle)

| Order | Gap | Rationale |
|---|---|---|
| 6 | **GAP-04** Audit Logs | Required for accountability once multiple admins exist. |
| 7 | **GAP-10** Session Management | Security incident response capability. |
| 8 | **GAP-11** Data Export | PM reporting without DB access. |
| 9 | **GAP-08** Content Moderation | Needed when multiple teachers create content. |

### Phase 3 — Platform Maturity (Post-Launch Enhancements)

| Order | Gap | Rationale |
|---|---|---|
| 10 | **GAP-09** OAuth Configuration | Improve student registration rates. |
| 11 | **GAP-07** System Settings | Operational flexibility for non-technical admins. |
| 12 | **GAP-14** Admin Profile | Quality-of-life for admins. |
| 13 | **GAP-13** Storage Management | Cost control and hygiene — less urgent until storage grows. |
| 14 | **GAP-12** Notifications | Nice-to-have; high effort relative to current phase value. |

---

## Appendix: Security Gaps Noted During Review

The following security issues were identified during this analysis that are separate from functional gaps but should be addressed alongside Phase 1 work:

| # | Issue | Current State | Recommendation |
|---|---|---|---|
| S-01 | No guard on `POST /users`, `PATCH /users/:id` | These endpoints are completely open — anyone can create or update any user record without authentication. | Add `JwtAuthGuard` + `RolesGuard(ADMIN)` to both. |
| S-02 | No guard on `POST /vocabularies`, `PATCH /vocabularies/:id`, `DELETE /vocabularies/:id` | Open to unauthenticated calls. | Add `JwtAuthGuard` + `RolesGuard(ADMIN, TEACHER)`. |
| S-03 | No guard on `POST/GET/PATCH/DELETE /user-vocabularies/*` | Open to unauthenticated calls. | Add `JwtAuthGuard`. |
| S-04 | No guard on `POST /supabase/upload` | Anyone can upload files to Supabase storage buckets. | Add `JwtAuthGuard`. |
| S-05 | Logout does not clear FE localStorage | Tokens remain in browser after logout, exploitable if browser is shared. | Implement token clear in `AppAdminHeader` logout handler. |
| S-06 | No refresh token rotation in FE Axios | Access token expiry causes silent failures — no retry logic. | Implement 401 interceptor in `jwtApiClient`. |

---

*End of Admin Gap Analysis — Over The World Platform*  
*Document: `2026-03-22-admin-gap-analysis.md`*  
*Next review date: After Phase 1 completion*
