# Implementation Progress Review
# Over The World — Admin Panel Sprint Review

> **Document Type:** Implementation Progress Review  
> **Date:** March 23, 2026  
> **Author:** Business Analyst  
> **Audience:** Development Team, Project Management  
> **Reference:** [2026-03-22-admin-gap-analysis.md](./2026-03-22-admin-gap-analysis.md)  
> **Status:** Draft v1.0

---

## Table of Contents

1. [Review Scope](#1-review-scope)
2. [Sprint Progress Summary](#2-sprint-progress-summary)
3. [Completed Gap Items](#3-completed-gap-items)
4. [Security Issues Identified](#4-security-issues-identified)
5. [Remaining Open Gaps](#5-remaining-open-gaps)
6. [Full Gap Status Table](#6-full-gap-status-table)
7. [Next Steps & Recommendations](#7-next-steps--recommendations)

---

## 1. Review Scope

This document reviews **implementation progress against the Admin Panel Gap Analysis** (2026-03-22) after one sprint cycle. It records which gaps have been resolved, which remain open, and surfaces new issues discovered during implementation.

**Sprint Period:** March 22–23, 2026  
**Sprint Focus:** Phase 1 gaps from the roadmap — primarily GAP-02, GAP-03, GAP-06  
**Review Method:** Direct codebase inspection (frontend routes, backend controllers, entity files, module registrations)  

---

## 2. Sprint Progress Summary

| Category | Count |
|---|---|
| Gaps fully completed this sprint | 3 (GAP-02, GAP-03, GAP-06) |
| Gaps confirmed already done (pre-existing) | 2 (GAP-05 full — BE + FE) |
| Security issues newly identified | 6 |
| Gaps remaining open | 10 (GAP-01, 04, 07–14) |
| New bugs found and fixed | 2 (LearningPlan 404, Autocomplete display) |

**Overall Phase 1 Completion: 4 / 5 items complete (80%)**  
GAP-02, GAP-03, GAP-05, and GAP-06 are confirmed done. **GAP-01 (Admin Dashboard) is the last outstanding Phase 1 item** and is the highest-priority next task.

---

## 3. Completed Gap Items

---

### GAP-02: Role & Permission Management ✅ COMPLETED

**Completion Date:** March 22, 2026  
**Summary:** Admins can now change a user's role directly from the Account Management page.

**Frontend Changes:**
- `AppAccountDialog` update form now includes a `role` select field (STUDENT / TEACHER / ADMIN).
- `updateAccountFormSchema` extended with `role` Yup validation.
- `useUpdateAccountService` sends the `role` field in the `PATCH /users/:id` body.

**Backend Status:** No changes needed — `PATCH /users/:id` already accepted `role` in the request body.

**Business Rules Verified:**
- ✅ Only `ADMIN` role can access account management and update role.
- ⚠️ Admin-cannot-change-own-role guard: **not yet enforced** — this is a low-risk edge case for current team size but should be added before multi-admin deployment.

**Residual Risk:** None blocking. Self-demotion guard is a future hardening item.

---

### GAP-03: Learning Plan Management (Admin UI) ✅ COMPLETED

**Completion Date:** March 23, 2026  
**Summary:** A full CRUD admin panel for Learning Plans is now available at `/admin/learning-plan-management`.

**Frontend Files Created:**
- `src/routes/admin/learning-plan-management/index.tsx` — full page with paginated table, search, create/edit/delete actions, bulk delete
- `src/components/AppLearningPlanDialog/index.tsx` — create/edit dialog with user autocomplete picker
- `src/formSchema/createLearningPlanFormSchema.ts` — Yup schema (type, description, startDate, endDate, userId)
- `src/formSchema/updateLearningPlanFormSchema.ts` — partial schema for update
- `src/services/queryService/useGetLearningPlanListService.ts` — paginated list with search
- `src/services/queryService/useGetLearningPlanDetailService.ts` — detail fetch (enabled when ID provided)
- `src/services/queryService/useGetAccountListAutocompleteService.ts` — infinite-scroll user picker for dialog
- `src/services/mutationService/useCreateLearningPlanService.ts`
- `src/services/mutationService/useUpdateLearningPlanService.ts`
- `src/services/mutationService/useDeleteLearningPlanService.ts`
- `src/services/mutationService/useDeleteLearningPlanMultipleService.ts`
- `src/services/formService/useLearningPlanFormService.ts` — Formik with `enableReinitialize: true`

**Frontend Files Modified:**
- `src/apiEndpoints/index.ts` — 6 new endpoint constants for learning-plan endpoints
- `src/formSchema/index.ts` — barrel export for 2 new schemas
- `src/services/queryService/index.tsx` — 3 new service exports
- `src/services/mutationService/index.ts` — 4 new service exports
- `src/services/formService/index.ts` — 1 new service export
- `src/components/index.ts` — `AppLearningPlanDialog` export
- `src/components/AppAdminNavbar/index.tsx` — added "Learning Plan Management" nav item with `EventNote` icon

**Backend Fix Applied:**
- `src/app.module.ts` — `LearningPlanModule` was **not registered** in `AppModule`, causing all `/learning-plans/*` routes to return 404. Fixed by adding the import and module entry.

**Bugs Found and Fixed During Implementation:**

1. **Bug B-01 — 404 on all `/learning-plans` endpoints:**
   - Root cause: `LearningPlanModule` was implemented but not registered in `AppModule`.
   - Fix: Added `import { LearningPlanModule }` and added it to the `imports[]` array in `app.module.ts`.

2. **Bug B-02 — Autocomplete selected item not displaying in User input field:**
   - Root cause: MUI `Autocomplete` fires `onInputChange` with `reason='reset'` when the user selects an option. The original `AppAutocomplete` handler only forwarded the `reason='input'` event, so the display text never updated on selection.
   - First attempt (incorrect): adding `reason='reset'` forwarding caused the selection label to become the search query, emptying the options list.
   - Final fix: `AppLearningPlanDialog` uses **two separate state variables** — `userInputDisplay` (controls the visible text, set `onChange` on selection) and `userSearchQuery` (sent to the API, set only on typing). `AppAutocomplete` reverted to forwarding `reason='input'` only.

**Business Rules Verified:**
- ✅ Learning plans can be created, updated, deleted, and bulk-deleted.
- ✅ Each learning plan is assigned to a user via the autocomplete picker.
- ✅ `type` and `description` fields are required; `startDate` and `endDate` are optional.
- ✅ List view is paginated and searchable.

---

### GAP-06: Password Reset / Account Recovery ✅ COMPLETED

**Completion Date:** March 22, 2026 (pre-existing from earlier work — confirmed via codebase review)  
**Summary:** Both admin-triggered password reset and user self-service forgot-password flow are implemented.

**Backend Implementation Confirmed:**
- `src/auth/entities/password-reset-token.entity.ts` — `PasswordResetToken` entity with SHA-256 hashed token, 60-minute TTL, single-use flag
- `src/mail/mail.service.ts` + `src/mail/mail.module.ts` — nodemailer SMTP integration (reads `SMTP_*` and `FRONTEND_URL` env vars)
- `POST /auth/forgot-password` — generates and emails a reset link
- `POST /auth/reset-password` — validates token and sets new hashed password
- `PATCH /users/:id` — admin can set a new password directly (admin reset path)

**Frontend Implementation Confirmed:**
- `/auth/forgot-password` route — email submission form
- `/auth/reset-password` route — new password form (reads `?token=` query param)
- `AppResetPasswordDialog` component in account detail — admin-triggered reset
- `useForgotPasswordService`, `useResetPasswordService`, `useAdminResetPasswordService`

**Business Rules Verified:**
- ✅ Reset tokens are single-use and expire after 60 minutes.
- ✅ Admin can force-reset any user's password.
- ✅ Self-service flow sends a reset link by email.
- ✅ New password is bcrypt-hashed before storage.

---

### GAP-05: Example Sentence Management ✅ FULLY CONFIRMED (pre-existing)

**Completion Date:** Pre-existing (both FE and BE were already done; gap analysis was incorrect on both counts)  
**Summary:** The gap analysis classified GAP-05 backend as "scaffolded stubs only" and frontend as not started. Direct codebase inspection reveals **both backend and frontend are fully implemented**.

**Confirmed Backend State:**
- `example.controller.ts` — full CRUD controller with `JwtAuthGuard` and `RolesGuard(ADMIN, TEACHER)` applied on all write endpoints
- `example.service.ts` — TypeORM repository-based service with create, findAllByVocab (by vocabularyId), findOne, update, remove
- Endpoints: `GET /examples/vocab/:vocabId`, `POST /examples`, `PATCH /examples/:id`, `DELETE /examples/:id`

**Confirmed Frontend State:**
- `AppExamplePanel` component — inline create / inline-edit / delete UI for example sentences; takes `vocabId` and `open` props
- `useGetExampleListService` — fetches examples for a vocab (`GET /examples/vocab/:vocabId`), enabled-guarded
- `useCreateExampleService` — creates example with `vocabularyId`, `cnSentence`, optional `pinyinSentence`/`viSentence`
- `useUpdateExampleService(id)` — patches an example by ID
- `useDeleteExampleService(id)` — deletes an example by ID
- `AppWordDialog` — edit mode embeds live `AppExamplePanel`; create mode uses a local pending-queue submitted after vocab creation
- Vocab detail page (`/admin/word-management/$vocabId`) — shows `AppExamplePanel` in a dedicated Card section
- All barrel exports in place; zero TypeScript compilation errors

---

## 4. Security Issues Identified

The following security vulnerabilities were discovered during codebase review. They are **not yet fixed** and should be addressed as a priority before any external deployment.

| ID | Severity | Location | Issue | Recommended Fix |
|---|---|---|---|---|
| S-01 | **Critical** | `POST /users` | No authentication guard — anyone on the internet can create an ADMIN account by sending `{ role: "ADMIN" }` in the request body | Add `JwtAuthGuard` + `RolesGuard(Role.ADMIN)` to the `POST /users` endpoint in `user.controller.ts` |
| S-02 | **High** | `POST /vocabularies`, `PATCH /vocabularies/:id`, `DELETE /vocabularies/:id`, `DELETE /vocabularies/multiple` | Write endpoints have no authentication guard — unauthenticated users can create, modify, or delete vocabulary | Add `JwtAuthGuard` + `RolesGuard(Role.ADMIN, Role.TEACHER)` to all write endpoints in `vocabulary.controller.ts` |
| S-03 | **High** | All `/user-vocabularies/*` endpoints | Entire controller has no authentication guard — user vocabulary progress can be read and modified by anyone | Add `JwtAuthGuard` at the controller level in `user-vocabulary.controller.ts` |
| S-04 | **High** | `POST /supabase/upload` | File upload endpoint has no authentication guard — anyone can upload files to the Supabase storage bucket | Add `JwtAuthGuard` to the upload endpoint in `supabase.controller.ts` |
| S-05 | **Medium** | Frontend — logout flow | `localStorage` JWT tokens are not cleared on logout; `POST /auth/logout` is called but the FE token is not removed, leaving valid tokens in the browser | Clear `localStorage` auth tokens in the logout mutation `onSuccess` handler |
| S-06 | **Medium** | Frontend — axios interceptor | No 401 interceptor or refresh-token retry logic; if the access token expires mid-session, all requests silently fail instead of triggering a refresh | Add a response interceptor to the axios instance that calls `POST /auth/refresh` on 401 and retries the original request |

**Note on S-01:** This is the most critical issue. A malicious actor can craft a single HTTP request to create an unlimited number of ADMIN accounts. This must be fixed before any public access to the API.

---

## 5. Remaining Open Gaps

The following gaps from the original gap analysis have not yet been started:

### GAP-01: Admin Dashboard / Analytics Overview ❌ NOT STARTED
- **Status:** ❌ Not started — **last outstanding Phase 1 item**
- **Priority:** High (Phase 1)
- **Business Value:** First-page situational awareness for admins; required before real user onboarding. Without this, admins have no landing page after login.
- **Estimated Effort:** Backend: 1 day (new `GET /admin/stats` aggregation endpoint); Frontend: 1–2 days (stat cards, optional charts)
- **Dependency:** None
- **Suggested Outputs:** Total users, total vocabulary, total categories, total learning plans — count cards with optional trend charts

### GAP-04: Audit Log / Activity History ❌ NOT STARTED
- **Status:** ❌ Not started
- **Priority:** High (Phase 2)
- **Estimated Effort:** Backend: 2–3 days (new `AuditLog` entity + NestJS interceptor + `GET /admin/audit-logs` endpoint); Frontend: 1 day (read-only filterable table)
- **Dependency:** None

### GAP-07: System Configuration / Settings ❌ NOT STARTED
- **Status:** ❌ Not started
- **Priority:** Medium (Phase 3)
- **Estimated Effort:** Backend: 1–2 days (`SystemSetting` entity + `GET/PATCH /admin/settings`); Frontend: 1 day

### GAP-08: Content Moderation / Vocabulary Approval Workflow ❌ NOT STARTED
- **Status:** ❌ Not started
- **Priority:** Medium (Phase 2)
- **Estimated Effort:** Backend: 1 day (add `status` enum to `Vocabulary` + approve/reject endpoints); Frontend: 1–2 days

### GAP-09: OAuth / Social Login Configuration ⚠️ PARTIAL
- **Status:** ⚠️ Partially started (backend strategies exist, not wired)
- **Priority:** Medium (Phase 2)
- **Current State:**
  - `google.strategy.ts` and `facebook.strategy.ts` both exist with correct Passport config
  - Neither strategy is registered in `AuthModule` providers (only `JwtStrategy` is present)
  - No OAuth callback routes exist in `auth.controller.ts`
  - No social login buttons on `/auth/login` page
- **Remaining Work:** Register strategies in `AuthModule`, add 4 OAuth callback routes to `AuthController`, add social login buttons to login page
- **Dependency:** GAP-07 (for admin enable/disable toggles — optional)

### GAP-10: Active Session Management ❌ NOT STARTED
- **Status:** ❌ Not started
- **Priority:** Medium (Phase 2)
- **Estimated Effort:** Backend: 1 day (3 endpoints on `RefreshToken` repo); Frontend: 0.5 day (sessions tab on account detail)

### GAP-11: Reporting & Data Export ❌ NOT STARTED
- **Status:** ❌ Not started
- **Priority:** Medium (Phase 2)
- **Estimated Effort:** Backend: 1–2 days (`json2csv` or `exceljs` export endpoints); Frontend: 0.5 day (export buttons)

### GAP-12: Notification / Announcement Management ❌ NOT STARTED
- **Status:** ❌ Not started
- **Priority:** Low (Phase 3)
- **Estimated Effort:** Backend: 2–3 days (new entity + email integration); Frontend: 2 days

### GAP-13: Storage / Media Management ❌ NOT STARTED
- **Status:** ❌ Not started
- **Priority:** Low (Phase 3)
- **Estimated Effort:** Backend: 1–2 days (Supabase bucket list + orphan detection); Frontend: 1 day

### GAP-14: Admin Profile Management ❌ NOT STARTED
- **Status:** ❌ Not started
- **Priority:** Low (Phase 3)
- **Estimated Effort:** Backend: 0.5 day (`PATCH /users/me` + `PATCH /users/me/password`); Frontend: 1 day (profile page or drawer)

---

## 6. Full Gap Status Table

| Gap ID | Function Name | Priority | Phase | Status | Notes |
|---|---|---|---|---|---|
| GAP-01 | Admin Dashboard / Analytics | High | 1 | ❌ Not started | Last Phase 1 item — top priority |
| GAP-02 | Role & Permission Management | High | 1 | ✅ Complete | Admin can assign roles |
| GAP-03 | Learning Plan Management (UI) | High | 1 | ✅ Complete | Full CRUD + bulk delete |
| GAP-04 | Audit Log / Activity History | High | 2 | ❌ Not started | — |
| GAP-05 | Example Sentence Management | High | 1 | ✅ Complete | BE + FE both confirmed done |
| GAP-06 | Password Reset / Account Recovery | High | 1 | ✅ Complete | Both admin + self-service flows |
| GAP-07 | System Configuration / Settings | Medium | 3 | ❌ Not started | — |
| GAP-08 | Content Moderation Workflow | Medium | 2 | ❌ Not started | — |
| GAP-09 | OAuth / Social Login Configuration | Medium | 2 | ⚠️ Partial BE | Strategies exist; not in AuthModule, no routes, no FE buttons |
| GAP-10 | Active Session Management | Medium | 2 | ❌ Not started | — |
| GAP-11 | Reporting & Data Export | Medium | 2 | ❌ Not started | — |
| GAP-12 | Notification / Announcement | Low | 3 | ❌ Not started | — |
| GAP-13 | Storage / Media Management | Low | 3 | ❌ Not started | — |
| GAP-14 | Admin Profile Management | Low | 3 | ❌ Not started | — |

**Legend:** ✅ Complete — ⚠️ Partial — ❌ Not started

---

## 7. Next Steps & Recommendations

### Immediate (Before Any New Feature Work) — Security Hardening

The following security fixes must be applied **immediately** to prevent unauthorized access:

1. **Fix S-01:** Add `@UseGuards(JwtAuthGuard, RolesGuard)` + `@Roles(Role.ADMIN)` to `POST /users` in `user.controller.ts`
2. **Fix S-02:** Add `@UseGuards(JwtAuthGuard, RolesGuard)` + `@Roles(Role.ADMIN, Role.TEACHER)` to all write endpoints in `vocabulary.controller.ts`
3. **Fix S-03:** Add `@UseGuards(JwtAuthGuard)` at the class level in `user-vocabulary.controller.ts`
4. **Fix S-04:** Add `@UseGuards(JwtAuthGuard)` to `POST /supabase/upload` in `supabase.controller.ts`
5. **Fix S-05:** Clear `localStorage` tokens in FE logout `onSuccess` callback
6. **Fix S-06:** Add a 401 interceptor with refresh-token retry to the axios instance

**Estimated effort:** 2–3 hours. These changes are low-risk, high-impact.

---

### Next Sprint — Recommended Priority Order

| Order | Gap / Task | Type | Effort |
|---|---|---|---|
| 1 | Security hardening (S-01 through S-06) | Bug / Security | 0.5 day |
| 2 | **GAP-01** Admin Dashboard | Feature — **last Phase 1 item** | 2–3 days |
| 3 | **GAP-04** Audit Log | Feature | 3–4 days |
| 4 | **GAP-10** Active Session Management | Feature | 1–2 days |
| 5 | **GAP-09** OAuth Wiring (finish partial) | Feature | 1 day |
| 6 | **GAP-08** Content Moderation Workflow | Feature | 2–3 days |

---

### ERD Status

The `database_erd.html` file has been fully updated to reflect all current entities as of this sprint:

- **Section 1 — User & Auth:** `User`, `RefreshToken`, `PasswordResetToken`
- **Section 2 — Core Vocabulary:** `Vocabulary`, `Category`, `Example`, `VocabularyCategory` (join table)
- **Section 3 — User Progress:** `UserVocabulary`
- **Section 4 — Learning Plan:** `LearningPlan`, `Progress`, `Task`
- **Section 5 — Full System Overview:** Combined diagram

All primary keys corrected to `uuid` type. All relationships verified against entity files.

---

*Document prepared by BA after direct codebase inspection on March 23, 2026.*  
*Next review scheduled after security hardening sprint completion.*
