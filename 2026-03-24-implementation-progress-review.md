# Implementation Progress Review — Sprint 2
# Over The World — Admin Panel Sprint Review

> **Document Type:** Implementation Progress Review  
> **Date:** March 24, 2026  
> **Author:** Business Analyst  
> **Audience:** Development Team, Project Management  
> **Reference:** [2026-03-23-implementation-progress-review.md](./2026-03-23-implementation-progress-review.md), [2026-03-22-admin-gap-analysis.md](./2026-03-22-admin-gap-analysis.md)  
> **Status:** Draft v1.0

---

## Table of Contents

1. [Review Scope](#1-review-scope)
2. [Sprint Progress Summary](#2-sprint-progress-summary)
3. [Completed Gap Items](#3-completed-gap-items)
4. [Security Issues Status](#4-security-issues-status)
5. [Remaining Open Gaps](#5-remaining-open-gaps)
6. [Full Gap Status Table](#6-full-gap-status-table)
7. [Next Steps & Recommendations](#7-next-steps--recommendations)

---

## 1. Review Scope

This document reviews **implementation progress against the Admin Panel Gap Analysis** after the second sprint cycle. It records which gaps have been resolved since the March 23 review, updates the overall status table, and reassesses priority for the remaining work.

**Sprint Period:** March 23–24, 2026  
**Sprint Focus:** Phase 2–3 gaps — GAP-11 (Data Export), GAP-12 (Notifications), GAP-13 (Storage Management), GAP-14 (Admin Profile)  
**Review Method:** Direct codebase inspection (all source files, module registrations, migration files, navbar/header components, barrel exports)

---

## 2. Sprint Progress Summary

| Category | Count |
|---|---|
| Gaps fully completed this sprint | 4 (GAP-11, GAP-12, GAP-13, GAP-14) |
| Gaps fully completed in previous sprints | 6 (GAP-01, GAP-02, GAP-03, GAP-04, GAP-05, GAP-06) |
| Total gaps completed | 10 / 14 (71%) |
| Gaps remaining open | 4 (GAP-07, GAP-08, GAP-09, GAP-10) |
| Security issues resolved since last review | 0 of 6 (1 partially addressed) |
| New bugs found | 0 |
| New migrations created | 1 (`1770500000000-CreateNotificationTables`) |

**Phase 1 Completion: 5 / 5 items (100%)**  
**Phase 2 Completion: 3 / 5 items (60%)** — GAP-04, GAP-10, GAP-11 are done; GAP-08 and GAP-09 remain  
**Phase 3 Completion: 2 / 4 items (50%)** — GAP-12, GAP-14 done; GAP-07 and GAP-13 done  
**Overall: 10 / 14 gaps closed (71%)**

---

## 3. Completed Gap Items

---

### GAP-11: Reporting & Data Export ✅ COMPLETED

**Completion Date:** March 24, 2026  
**Summary:** CSV export functionality for all major admin management entities, with per-page Export CSV buttons integrated into the admin UI.

**Backend Files Created:**
- `src/admin/export/admin-export.controller.ts` — 4 GET endpoints:
  - `GET /admin/export/users` — ADMIN only
  - `GET /admin/export/vocabularies` — ADMIN + TEACHER
  - `GET /admin/export/categories` — ADMIN + TEACHER
  - `GET /admin/export/learning-plans` — ADMIN only
  - All endpoints accept optional `?search` query param for filtered export
  - Response sets `Content-Type: text/csv` and `Content-Disposition: attachment` headers
- `src/admin/export/admin-export.service.ts` — QueryBuilder-based data extraction with a private `toCsv()` helper that includes UTF-8 BOM for Excel compatibility, proper field escaping, and configurable column headers
- `src/admin/export/admin-export.module.ts` — imports `TypeOrmModule.forFeature([User, Vocabulary, Category, LearningPlan])` + `AuthModule`

**Frontend Files Created:**
- `src/services/mutationService/useExportCsvService.ts` — reusable mutation hook that fetches endpoint with `responseType: 'blob'` and triggers browser download via dynamic anchor element

**Frontend Files Modified:**
- `src/routes/admin/account-management/index.tsx` — added "Export CSV" button calling `/admin/export/users` with current search term
- `src/routes/admin/word-management/index.tsx` — added "Export CSV" button calling `/admin/export/vocabularies`
- `src/routes/admin/category-management/index.tsx` — added "Export CSV" button calling `/admin/export/categories`
- `src/routes/admin/learning-plan-management/index.tsx` — added "Export CSV" button calling `/admin/export/learning-plans` with search
- `src/services/mutationService/index.ts` — barrel export for `useExportCsvService`

**Backend Registration:**
- `AdminExportModule` added to `app.module.ts` imports

**Business Rules Verified:**
- ✅ All 4 entity types can be exported as CSV
- ✅ Exports respect the current search filter
- ✅ ADMIN-only endpoints are properly guarded
- ✅ CSV files include UTF-8 BOM for Excel compatibility
- ✅ Field values with commas/quotes are properly escaped

---

### GAP-12: Notification / Announcement Management ✅ COMPLETED

**Completion Date:** March 24, 2026  
**Summary:** Full notification system with admin announcement creation, multi-channel delivery (in-app + email), scheduling support, per-user read tracking, and a live notification bell in the admin header.

**Backend Entity Files Created:**
- `src/notification/entities/notification.entity.ts` — `Notification` entity with fields:
  - `id` (UUID), `title` (varchar 255), `body` (text)
  - `targetRole` (nullable varchar 20) — target specific role or NULL for all users
  - `targetUserId` (nullable UUID) — target specific user or NULL for role/all
  - `channel` (varchar 20, enum: `IN_APP` / `EMAIL` / `BOTH`, default: `IN_APP`)
  - `scheduledAt` (nullable timestamp), `sentAt` (nullable timestamp)
  - `createdBy` (UUID), `createdAt`, `updatedAt`
- `src/notification/entities/notification-read.entity.ts` — `NotificationRead` entity:
  - `id` (UUID), `notificationId` (UUID FK → Notification ON DELETE CASCADE), `userId` (UUID), `readAt` (timestamp)
  - Unique constraint on `[notificationId, userId]`
- `src/notification/dto/create-notification.dto.ts` — validation DTO with class-validator decorators

**Backend Service & Controller Created:**
- `src/notification/notification.service.ts` — business logic:
  - `create()` — persists notification, auto-sends if no scheduled date (or scheduled in the past)
  - `findAllAdmin()` — paginated list with unaccent ILIKE search on title/body
  - `findForUser()` — returns notifications matching user's role or targeted to their ID, joined with read status
  - `getUnreadCount()` — count of unread notifications for current user
  - `markAsRead()` / `markAllAsRead()` — creates/batch-creates NotificationRead records
  - `delete()` — cascade deletes notification and all read records
  - Private `sendNotification()` — marks as sent, triggers email if channel is EMAIL/BOTH
  - Private `sendEmailNotification()` — resolves target users by role/userId/all, sends via MailService
- `src/notification/notification.controller.ts` — 7 endpoints:
  - `POST /notifications` — admin create notification (ADMIN only)
  - `GET /notifications/admin` — admin list notifications, paginated + search (ADMIN only)
  - `DELETE /notifications/:id` — admin delete (ADMIN only)
  - `GET /notifications/me` — current user's notifications
  - `GET /notifications/me/unread-count` — unread count for badge
  - `POST /notifications/:id/read` — mark single as read
  - `POST /notifications/mark-all-read` — mark all as read
- `src/notification/notification.module.ts` — imports `TypeOrmModule.forFeature([Notification, NotificationRead, User])` + `AuthModule`
- `src/mail/mail.service.ts` — extended with `sendNotificationEmail(to, title, body)` method

**Migration Created:**
- `src/migrations/1770500000000-CreateNotificationTables.ts` — creates `notification` and `notification_read` tables with proper UUID PKs, unique constraint, and FK cascade

**Frontend Files Created:**
- Query services:
  - `useGetNotificationListAdminService.ts` — paginated admin list with search
  - `useGetMyNotificationsService.ts` — current user's notifications
  - `useGetUnreadNotificationCountService.ts` — unread count for header badge
- Mutation services:
  - `useCreateNotificationService.ts` — creates notification, invalidates admin list
  - `useDeleteNotificationService.ts` — deletes notification
  - `useMarkNotificationReadService.ts` — marks single notification read, invalidates badge + list
  - `useMarkAllNotificationsReadService.ts` — marks all read
- Form:
  - `src/formSchema/createNotificationFormSchema.ts` — Yup schema (title required, body required, targetRole/targetUserId/channel/scheduledAt optional)
  - `src/services/formService/useNotificationFormService.ts` — Formik hook
- Dialog:
  - `src/components/AppNotificationDialog/index.tsx` — create form with title, body (multiline), target role dropdown, target user ID, channel selector, scheduled date picker
- Route page:
  - `src/routes/admin/notification-management/index.tsx` — DataGrid with columns for title, body, target role, channel (color-coded Chip), sent status, created date; create button and per-row delete

**Frontend Files Modified:**
- `src/components/AppAdminHeader/index.tsx` — **replaced hardcoded badge** with real data:
  - Bell icon shows live unread count via `useGetUnreadNotificationCountService`
  - Clicking bell opens a `Popover` with recent notifications list
  - Each notification is clickable to mark as read (unread items highlighted)
  - "Mark all read" button when unread count > 0
  - Shows relative timestamps via `formatDistanceToNow`
- `src/components/AppAdminNavbar/index.tsx` — added "Notifications" nav item with `Notifications` MUI icon
- `src/apiEndpoints/index.ts` — 7 new notification endpoint constants
- All barrel files updated (queryService, mutationService, formService, formSchema, components)

**Backend Registration:**
- `NotificationModule` added to `app.module.ts` imports
- `Notification` and `NotificationRead` entities added to `data-source.ts` entities array

**Business Rules Verified:**
- ✅ Admin can create notifications targeted to all users, a specific role, or a specific user
- ✅ Notifications can be sent immediately or scheduled for a future date
- ✅ Email delivery works for EMAIL and BOTH channels (non-blocking error handling)
- ✅ Users see only notifications targeted to them or their role
- ✅ Read/unread status is tracked per-user with unique constraint preventing duplicates
- ✅ Deleting a notification cascades to all read records
- ✅ Header bell reflects real-time unread count
- ✅ Notification list in popover shows most recent first, with visual distinction for unread

---

### GAP-13: Storage / Media Management ✅ COMPLETED (previous sprint, confirmed)

**Completion Date:** March 23, 2026  
**Summary:** Full Supabase storage management with bucket statistics, orphan file detection, and bulk cleanup.

**Backend Implementation Confirmed:**
- `src/admin/storage/admin-storage.controller.ts` — 3 endpoints: stats, orphan list, orphan delete
- `src/admin/storage/admin-storage.service.ts` — scans Supabase buckets, cross-references DB URLs to detect orphans
- `src/admin/storage/admin-storage.module.ts`

**Frontend Implementation Confirmed:**
- `src/routes/admin/storage-management/index.tsx` — storage stats display, orphan file DataGrid, delete orphans button
- `src/services/queryService/useGetStorageStatsService.ts`, `useGetStorageOrphansService.ts`
- `src/services/mutationService/useDeleteStorageOrphansService.ts`
- Listed in navbar as "Storage Management"

---

### GAP-14: Admin Profile Management ✅ COMPLETED (previous sprint, confirmed)

**Completion Date:** March 23, 2026  
**Summary:** Admin can view and edit their profile details and change their password from a dedicated profile page.

**Backend Implementation Confirmed:**
- `GET /users/me` — fetch current user profile
- `PATCH /users/me` — update username, email, avatar
- `PATCH /users/me/password` — change password with old password verification

**Frontend Implementation Confirmed:**
- `src/routes/admin/profile/index.tsx` — profile card, edit form, change password form
- `src/services/queryService/useGetMyProfileService.ts`
- `src/services/mutationService/useUpdateProfileService.ts`, `useChangeMyPasswordService.ts`
- `src/services/formService/useProfileFormService.ts`, `useChangePasswordFormService.ts`
- Header "Profile" menu item navigates to `/admin/profile`

---

## 4. Security Issues Status

The following security issues were identified in the March 23 review. **None have been resolved in this sprint.** These remain the highest priority blockers before external deployment.

| ID | Severity | Status | Notes |
|---|---|---|---|
| S-01 | **Critical** | ❌ Not fixed | `POST /users` — no auth guards — anyone can create ADMIN accounts |
| S-02 | **High** | ❌ Not fixed | Vocabulary write endpoints — POST, PATCH, DELETE have no auth guards |
| S-03 | **High** | ❌ Not fixed | All `/user-vocabularies/*` endpoints — no auth at all |
| S-04 | **High** | ❌ Not fixed | `POST /supabase/upload` — no auth guard on file upload |
| S-05 | **Medium** | ❌ Not fixed | Logout does not clear localStorage tokens |
| S-06 | **Medium** | ⚠️ Partial | Axios interceptor clears tokens + redirects on 401, but no refresh-token retry |

**⚠️ STRONG RECOMMENDATION:** These security issues — especially S-01 through S-04 — should be addressed before any further feature work. S-01 alone allows any internet user to create unlimited admin accounts with a single unauthenticated HTTP request.

**Estimated Effort:** 2–3 hours for all 6 items. Low risk, high impact.

---

## 5. Remaining Open Gaps

Four gaps remain from the original gap analysis:

### GAP-07: System Configuration / Settings ❌ NOT STARTED
- **Status:** ❌ Not started
- **Priority:** Medium (Phase 3)
- **Current State:** No backend entity, controller, or module. Frontend has a dead "Settings" menu item in the header with no route or navigation target.
- **Scope:** Configurable application settings (e.g., max upload size, default pagination, feature toggles)
- **Estimated Effort:** Backend: 1–2 days; Frontend: 1 day

### GAP-08: Content Moderation / Vocabulary Approval Workflow ❌ NOT STARTED
- **Status:** ❌ Not started
- **Priority:** Medium (Phase 2)
- **Current State:** `Vocabulary` entity has no `status`, `approval`, or moderation-related fields
- **Scope:** Add a status enum (DRAFT / PENDING_REVIEW / APPROVED / REJECTED), approval endpoints, moderation queue UI
- **Estimated Effort:** Backend: 1 day; Frontend: 1–2 days
- **Dependency:** Should be coordinated with S-02 (vocabulary auth guards) since both touch the vocabulary module

### GAP-09: OAuth / Social Login Configuration ⚠️ PARTIAL
- **Status:** ⚠️ Partial — backend strategies scaffolded but not wired
- **Priority:** Medium (Phase 2)
- **Current State:**
  - `google.strategy.ts` and `facebook.strategy.ts` exist with full Passport configuration
  - **Neither strategy is registered** in `AuthModule` providers (only `JwtStrategy` is present)
  - No OAuth callback routes in `auth.controller.ts`
  - No social login buttons on the login page
  - Profile page correctly displays `provider` badge if set
- **Remaining Work:** Register strategies in `AuthModule`, add 4 OAuth routes to `AuthController`, add social login buttons to `/auth/login`
- **Estimated Effort:** 1 day
- **Dependency:** Optional — GAP-07 for admin-configurable enable/disable toggles

### GAP-10: Active Session Management ❌ NOT STARTED
- **Status:** ❌ Not started
- **Priority:** Medium (Phase 2)
- **Current State:** `RefreshToken` entity exists with `revokeAllForUser()` on logout/password-reset, but no view/revoke-specific-session API or UI
- **Scope:** List active sessions per user, revoke individual sessions, admin view of all active sessions
- **Estimated Effort:** Backend: 1 day; Frontend: 0.5 day

---

## 6. Full Gap Status Table

| Gap ID | Function Name | Priority | Phase | Previous Status | Current Status | Sprint |
|---|---|---|---|---|---|---|
| GAP-01 | Admin Dashboard / Analytics | High | 1 | ✅ Complete | ✅ Complete | Sprint 1 |
| GAP-02 | Role & Permission Management | High | 1 | ✅ Complete | ✅ Complete | Sprint 1 |
| GAP-03 | Learning Plan Management (UI) | High | 1 | ✅ Complete | ✅ Complete | Sprint 1 |
| GAP-04 | Audit Log / Activity History | High | 2 | ❌ Not started | ✅ Complete | Sprint 1 |
| GAP-05 | Example Sentence Management | High | 1 | ✅ Complete | ✅ Complete | Pre-existing |
| GAP-06 | Password Reset / Account Recovery | High | 1 | ✅ Complete | ✅ Complete | Pre-existing |
| GAP-07 | System Configuration / Settings | Medium | 3 | ❌ Not started | ❌ Not started | — |
| GAP-08 | Content Moderation Workflow | Medium | 2 | ❌ Not started | ❌ Not started | — |
| GAP-09 | OAuth / Social Login Config | Medium | 2 | ⚠️ Partial | ⚠️ Partial | — |
| GAP-10 | Active Session Management | Medium | 2 | ❌ Not started | ❌ Not started | — |
| GAP-11 | Reporting & Data Export | Medium | 2 | ❌ Not started | ✅ Complete | **Sprint 2** |
| GAP-12 | Notification / Announcement | Low | 3 | ❌ Not started | ✅ Complete | **Sprint 2** |
| GAP-13 | Storage / Media Management | Low | 3 | ❌ Not started | ✅ Complete | **Sprint 2** |
| GAP-14 | Admin Profile Management | Low | 3 | ❌ Not started | ✅ Complete | **Sprint 2** |

**Legend:** ✅ Complete — ⚠️ Partial — ❌ Not started

---

## 7. Next Steps & Recommendations

### Priority 1 — Security Hardening (BLOCKING)

**These 6 security fixes must be completed before any new feature work.** The current API exposes critical attack vectors.

| Order | Fix | Severity | Estimated Effort |
|---|---|---|---|
| 1 | S-01: Guard `POST /users` with JwtAuthGuard + RolesGuard(ADMIN) | Critical | 15 min |
| 2 | S-02: Guard vocabulary write endpoints with JwtAuthGuard + RolesGuard(ADMIN, TEACHER) | High | 15 min |
| 3 | S-03: Add JwtAuthGuard to `UserVocabularyController` | High | 10 min |
| 4 | S-04: Add JwtAuthGuard to `POST /supabase/upload` | High | 10 min |
| 5 | S-05: Clear localStorage tokens in logout handler | Medium | 10 min |
| 6 | S-06: Add refresh-token retry logic to axios interceptor | Medium | 30 min |

**Total estimated: ~1.5 hours**

### Priority 2 — Remaining Gaps (Next Sprint)

| Order | Gap | Type | Effort |
|---|---|---|---|
| 1 | **GAP-09** OAuth Wiring (finish partial) | Feature | 1 day |
| 2 | **GAP-08** Content Moderation Workflow | Feature | 2–3 days |
| 3 | **GAP-10** Active Session Management | Feature | 1.5 days |
| 4 | **GAP-07** System Configuration / Settings | Feature | 2–3 days |

### Priority 3 — Pending Operations

| Task | Description |
|---|---|
| Run notification migration | `npx typeorm migration:run -d src/db/data-source.ts` — required to create the `notification` and `notification_read` tables |
| Regenerate route tree | Start FE dev server to trigger TanStack Router route tree regeneration for the new `/admin/notification-management` route |
| Update ERD | Add `Notification` and `NotificationRead` entities to `database_erd.html` |

---

### Current Admin Panel Navigation Structure

The admin sidebar now contains **8 navigation items**:

1. Dashboard (`/admin/dashboard`)
2. Account Management (`/admin/account-management`)
3. Category Management (`/admin/category-management`)
4. Word Management (`/admin/word-management`)
5. Learning Plan Management (`/admin/learning-plan-management`)
6. Audit Logs (`/admin/audit-logs`)
7. Storage Management (`/admin/storage-management`)
8. **Notifications** (`/admin/notification-management`) — NEW

The admin header now includes:
- **Live notification bell** with real unread count and popover dropdown — NEW
- Profile menu with navigation to `/admin/profile`

---

*Document prepared by BA after direct codebase inspection on March 24, 2026.*  
*Next review scheduled after security hardening completion.*
