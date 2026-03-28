# Sprint Planning — Over The World Student App (Frontend)

> **Document:** Sprint Task Breakdown  
> **Date:** March 25, 2026  
> **Project:** `over-the-world-fe-student` (Next.js 16 App Router)  
> **Reference:** [BA_FE_STUDENT_REQUIREMENT.md](../../../BA_FE_STUDENT_REQUIREMENT.md) v1.0  
> **Author:** BA Team  
> **Status:** Approved for Development

---

## 1. Current Implementation Baseline

### 1.1 Implementation Summary (as of 2026-03-25)

| Feature Module | BA Ref | Phase | Implementation Status | Completion |
|----------------|--------|-------|-----------------------|------------|
| F-S01: Authentication (Email/Password) | F-S01.1, F-S01.3 | Phase 1 | ✅ Done — Login, Register, Forgot/Reset Password, Token Refresh, Middleware | 90% |
| F-S01: OAuth (Google & Facebook) | F-S01.2 | Phase 4 | ❌ Not started — Needs backend GAP-S01 | 0% |
| F-S02: Teacher Posts & Lessons (SEO) | F-S02 | Phase 3 | ❌ Not started — Needs backend GAP-S02, GAP-S03 | 0% |
| F-S03: Vocabulary Browsing | F-S03 | Phase 1 | ✅ Done — Catalog, Search, Category filter, Detail, Pagination | 95% |
| F-S04: Personal Vocabulary List | F-S04 | Phase 1 | ✅ Done — Add/Remove, Favorites, Status tabs, Pagination | 85% |
| F-S05: Flashcard Review (SM-2) | F-S05 | Phase 2 | ⚠️ Partial — Basic flashcard flip exists, **NO SM-2 algorithm**, no quality rating (0–5) | 25% |
| F-S06: Quiz & Practice | F-S06 | Phase 2 | ⚠️ Partial — Multiple-choice exists, **NO fill-in-blank, NO pinyin matching**, no session persistence | 20% |
| F-S07: Learning Plan Dashboard | F-S07 | Phase 2 | ❌ Stub — No API integration, placeholder pages only | 5% |
| F-S08: Gamification (XP, Badges, Leaderboard) | F-S08 | Phase 3 | ❌ Not started — Needs backend GAP-S06 | 0% |
| F-S09: Calendar & Streak | F-S09 | Phase 2 | ❌ Not started — Needs backend GAP-S07 | 0% |
| F-S10: Notifications | F-S10 | Phase 1 | ✅ Done — Bell, Unread count, Mark read, Mark all read, Full page | 95% |
| F-S11: User Profile & Settings | F-S11 | Phase 1–2 | ⚠️ Partial — Profile view done, **Edit profile is stub**, **Settings is stub** | 35% |
| F-S12: Dashboard (Home) | F-S12 | Phase 1 | ⚠️ Partial — Stats cards done, **missing** daily goals, streak, leaderboard preview, due-for-review | 40% |

### 1.2 Infrastructure Status

| Layer | Status |
|-------|--------|
| Next.js 16 + App Router | ✅ Configured, build passes |
| Tailwind CSS v4 + shadcn/ui (14 components) | ✅ Installed |
| TanStack Query v5 | ✅ Provider + DevTools configured |
| Axios (apiClient + jwtApiClient with 401 refresh) | ✅ Complete |
| Token management (localStorage + cookie sync) | ✅ Complete |
| Middleware (auth route protection) | ✅ Complete |
| React Hook Form + Zod (4 auth schemas) | ✅ Complete |
| TypeScript types (User, Vocabulary, Category, etc.) | ✅ Complete |
| 19 routes all compiling | ✅ Verified |

### 1.3 Backend Dependency Status

| Gap ID | Description | Required By | Backend Status |
|--------|-------------|-------------|---------------|
| GAP-S01 | OAuth Google/Facebook callbacks | Sprint 5 | ❌ Not started |
| GAP-S02 | Post entity + CRUD + public listing | Sprint 4 | ❌ Not started |
| GAP-S03 | Lesson entity + CRUD + public listing | Sprint 4 | ❌ Not started |
| GAP-S04 | SM-2 fields on UserVocabulary | Sprint 2 | ❌ Not started |
| GAP-S05 | Quiz engine (QuizSession, QuizResult) | Sprint 2 | ❌ Not started |
| GAP-S06 | Gamification (UserXP, Badge, Leaderboard) | Sprint 4 | ❌ Not started |
| GAP-S07 | DailyActivity + Streak logic | Sprint 3 | ❌ Not started |
| GAP-S08 | UserSettings entity + endpoints | Sprint 2 | ❌ Not started |
| GAP-S09 | Progress/Task API implementation | Sprint 3 | ❌ Not started |

---

## 2. Sprint Overview

| Sprint | Duration | Theme | Backend Deps |
|--------|----------|-------|--------------|
| **Sprint 1** | 1 week | Phase 1 Completion — Polish existing features, close gaps | None |
| **Sprint 2** | 2 weeks | Phase 2a — SM-2 Flashcard, Quiz Engine, User Settings | GAP-S04, GAP-S05, GAP-S08 |
| **Sprint 3** | 2 weeks | Phase 2b — Streak Calendar, Learning Plans, Dashboard Upgrade | GAP-S07, GAP-S09 |
| **Sprint 4** | 2 weeks | Phase 3 — SEO Content (Posts & Lessons), Gamification | GAP-S02, GAP-S03, GAP-S06 |
| **Sprint 5** | 1 week | Phase 4 — OAuth Login, Final Polish, Performance | GAP-S01 |

**Total estimated duration:** 8 weeks

---

## 3. Sprint 1 — Phase 1 Completion & Polish (1 week)

> **Goal:** Close all remaining gaps in Phase 1 features. Ensure the core flow (register → browse → add words → basic review) works end-to-end. Improve UX polish.

### Epic 1.1: Profile Edit Page

**BA Reference:** F-S11.2 (BS-81, BS-82, BS-83)

| Task ID | Title | Type | Priority | Est. | Acceptance Criteria |
|---------|-------|------|----------|------|---------------------|
| S1-001 | Create profile edit form with Zod schema | Feature | High | 3h | Form fields: username, email. Validated with Zod (email unique check handled server-side). Submit calls `PATCH /users/me`. Success toast + redirect to `/profile`. |
| S1-002 | Implement avatar upload with crop & compress | Feature | Medium | 4h | Upload dialog with crop (react-easy-crop). Compress image client-side (max 500KB, WebP). Upload to Supabase via `POST /supabase/upload`. Preview before save. |
| S1-003 | Implement change password form | Feature | Medium | 2h | Separate section: current password + new password + confirm. Call `PATCH /users/me/password`. Validate min 6 chars. Show success/error toast. |
| S1-004 | Add delete confirmation dialog to My List | UX Polish | Medium | 1h | Per BS-33: removing a word shows a confirmation dialog before deleting. Currently deletes immediately. |

### Epic 1.2: Dashboard Enhancements

**BA Reference:** F-S12 (BS-87, BS-88, BS-89)

| Task ID | Title | Type | Priority | Est. | Acceptance Criteria |
|---------|-------|------|----------|------|---------------------|
| S1-005 | Add "Due for Review" card to dashboard | Feature | High | 2h | Show count of words where `nextReviewDate <= now` (once SM-2 fields exist, use `lastReviewed` as fallback for now). CTA button links to `/review`. Per BS-88. |
| S1-006 | Add onboarding empty state | UX Polish | Medium | 1h | If student has 0 words in list, show onboarding prompt (BS-89): "Bắt đầu bằng cách thêm từ vựng vào danh sách của bạn!" with link to `/vocabulary`. |
| S1-007 | Add recent activity timeline section | Feature | Low | 2h | Show last 5 activities. Initially derive from `UserVocabulary.createdAt` (words added) and `lastReviewed` timestamps until `DailyActivity` is available. |

### Epic 1.3: Vocabulary Browsing Polish

**BA Reference:** F-S03 (BS-25, BS-27), F-S04 (BS-29, BS-34)

| Task ID | Title | Type | Priority | Est. | Acceptance Criteria |
|---------|-------|------|----------|------|---------------------|
| S1-008 | Show status badge on already-added words in catalog | UX Polish | Medium | 1.5h | Per BS-25: If word is already in student's list, show current status (new/learning/mastered) with a checkmark instead of the "+ Add" button. |
| S1-009 | Add summary header bar to My List | UX Polish | Medium | 1h | Per BS-34: Show total words, new count, learning count, mastered count with color-coded badges at the top of the my-list page. |
| S1-010 | Improve search debounce to 300ms | Bug Fix | Low | 0.5h | Per BS-27: Current debounce is 400ms, change to 300ms as specified. |

### Epic 1.4: Notification Polish

**BA Reference:** F-S10 (BS-77, BS-80)

| Task ID | Title | Type | Priority | Est. | Acceptance Criteria |
|---------|-------|------|----------|------|---------------------|
| S1-011 | Add notification dropdown in header | Feature | Medium | 2h | Per BS-77: clicking bell opens a dropdown panel showing the 5 most recent notifications with unread highlighted. Currently only links to `/notifications` page. |
| S1-012 | Add refetch on window focus | UX Polish | Low | 0.5h | Per BS-80: refetch unread count on `window.focus` event, in addition to the existing 60s polling. |

### Sprint 1 Summary

| Metric | Value |
|--------|-------|
| Total tasks | 12 |
| Total estimated hours | ~20h |
| High priority | 3 |
| Medium priority | 7 |
| Low priority | 2 |
| Backend dependencies | None |

---

## 4. Sprint 2 — SM-2 Flashcard & Quiz Engine (2 weeks)

> **Goal:** Implement scientifically-proven spaced repetition (SM-2) flashcard review and full quiz engine with multiple question types. Requires backend GAP-S04, GAP-S05, GAP-S08.  
>  
> **⚠️ Backend Prerequisite:** GAP-S04 (SM-2 fields on UserVocabulary), GAP-S05 (QuizSession + QuizResult entities), GAP-S08 (UserSettings entity) must be completed by backend team BEFORE or IN PARALLEL with this sprint.

### Epic 2.1: SM-2 Spaced Repetition Flashcard

**BA Reference:** F-S05 (BS-35 through BS-43)  
**Backend Dependency:** GAP-S04

| Task ID | Title | Type | Priority | Est. | Acceptance Criteria |
|---------|-------|------|----------|------|---------------------|
| S2-001 | Update `UserVocabulary` type with SM-2 fields | Infra | High | 0.5h | Add `easeFactor`, `interval`, `repetitions`, `nextReviewDate` to TypeScript `UserVocabulary` interface. |
| S2-002 | Create `useGetDueFlashcardsService` query | Service | High | 1h | Fetch user-vocabularies where `nextReviewDate <= now`, ordered by `nextReviewDate ASC`, limited by daily review limit. Accepts `limit` param. |
| S2-003 | Implement SM-2 algorithm utility function | Logic | High | 2h | Pure function `calculateSM2(quality, easeFactor, interval, repetitions)` returns `{ easeFactor, interval, repetitions, nextReviewDate }`. Unit-testable. Per F-S05.2 formula. |
| S2-004 | Create `useSubmitFlashcardReviewService` mutation | Service | High | 1h | Calls `PATCH /user-vocabularies/:id` with recalculated SM-2 fields + `lastReviewed = now`. Invalidates flashcard query cache. |
| S2-005 | Redesign Review page with SM-2 quality rating | Feature | High | 6h | **Front:** Chinese word. **Back (on flip):** pinyin, meaning, example. After flip, show 6 rating buttons (0–5) with Vietnamese labels (BS-39). After rating, auto-advance to next card. Failed cards (quality < 3) requeue at session end (BS-41). |
| S2-006 | Add flashcard session summary screen | Feature | Medium | 2h | After all cards reviewed: total reviewed, correct (quality ≥ 3), incorrect (quality < 3), XP earned placeholder. "Ôn tập lại" + "Về trang chủ" buttons. Per BS-42. |
| S2-007 | Add "No cards due" empty state | UX | Medium | 1h | Per BS-37: If no words have `nextReviewDate <= now`, show "Bạn đã ôn xong cho hôm nay! 🎉" with next scheduled date shown. |
| S2-008 | Add keyboard shortcuts for flashcard | A11Y | Low | 1.5h | Per USE-S06: Space = flip card, 0–5 = rate quality, Enter/→ = next card, ← = previous card. Show shortcut hints on desktop. |
| S2-009 | Add 3D flip animation with Framer Motion | UX | Medium | 2h | Per USE-S05: Flashcard flip uses 3D CSS transform (rotateY). Smooth 60fps. Respects `prefers-reduced-motion`. |

### Epic 2.2: Full Quiz Engine

**BA Reference:** F-S06 (BS-44 through BS-50)  
**Backend Dependency:** GAP-S05

| Task ID | Title | Type | Priority | Est. | Acceptance Criteria |
|---------|-------|------|----------|------|---------------------|
| S2-010 | Create quiz-related TypeScript types | Infra | High | 0.5h | `QuizSession`, `QuizResult`, `QuizType` enum (`MULTIPLE_CHOICE`, `FILL_IN_BLANK`, `PINYIN_MATCH`). |
| S2-011 | Create quiz service hooks | Service | High | 2h | `useGenerateQuizService` (POST /quizzes/generate), `useSubmitQuizAnswerService` (POST /quizzes/:id/answer), `useGetQuizSummaryService` (GET /quizzes/:id/summary). |
| S2-012 | Redesign practice mode selection page | Feature | High | 2h | Update `/practice` page to show 3 verified modes (Multiple Choice, Fill-in-Blank, Pinyin Matching). Each card shows description + minimum word requirement warning (BS-47: "Hãy thêm ít nhất 4 từ"). Optional category filter before starting. |
| S2-013 | Implement Multiple Choice quiz component | Feature | High | 4h | Show Chinese word → 4 Vietnamese options. Timer visible (not enforced). Immediate feedback (correct/incorrect + correct answer highlighted). Current: basic version exists, upgrade with proper API integration, distractor logic (BS-45), XP display. |
| S2-014 | Implement Fill-in-the-Blank quiz component | Feature | High | 4h | Show Vietnamese meaning → text input for Chinese word or pinyin. Case-insensitive + accent-normalized comparison (BS-46). "Hint" button shows first character. Immediate feedback with correct answer. |
| S2-015 | Implement Pinyin Matching quiz component | Feature | High | 5h | Show 5 Chinese words + 5 shuffled pinyin readings. Tap-to-match interaction (select word → select pinyin). Matched pairs highlight in green. +5 XP per correct pair. |
| S2-016 | Create quiz session result screen | Feature | Medium | 2h | Summary: score, accuracy %, XP earned, time taken. "Làm lại" (Retry), "Chế độ khác" (Other mode), "Về trang chủ" (Home) buttons. |
| S2-017 | Add minimum word count validation | Guard | Medium | 1h | Per BS-47: Before starting quiz, check if student has ≥ 4 words. If not, show message and link to vocabulary catalog. |

### Epic 2.3: User Settings

**BA Reference:** F-S11.3 (BS-84, BS-85, BS-86)  
**Backend Dependency:** GAP-S08

| Task ID | Title | Type | Priority | Est. | Acceptance Criteria |
|---------|-------|------|----------|------|---------------------|
| S2-018 | Create UserSettings type + API endpoint constants | Infra | High | 0.5h | TypeScript `UserSettings` interface. Add `GET /user-settings/me`, `PATCH /user-settings/me` to API_ENDPOINTS. |
| S2-019 | Create `useGetUserSettingsService` query | Service | High | 0.5h | Fetch settings for current user. Fallback to defaults if 404. |
| S2-020 | Create `useUpdateUserSettingsService` mutation | Service | High | 0.5h | PATCH settings. Invalidate settings cache. |
| S2-021 | Implement Settings page with real form | Feature | High | 3h | Replace stub with real form: daily flashcard goal (slider/input, default 20), daily quiz goal (default 10), daily new words goal (default 5), notification toggle, email notification toggle. Auto-save with debounce (BS-86). |

### Sprint 2 Summary

| Metric | Value |
|--------|-------|
| Total tasks | 21 |
| Total estimated hours | ~42h |
| High priority | 15 |
| Medium priority | 5 |
| Low priority | 1 |
| Backend dependencies | GAP-S04, GAP-S05, GAP-S08 |

---

## 5. Sprint 3 — Streak Calendar, Learning Plans & Dashboard Upgrade (2 weeks)

> **Goal:** Build the streak/calendar visualization (GitHub-style heatmap), implement learning plan pages, and upgrade the dashboard with daily goals and streak display.  
>  
> **⚠️ Backend Prerequisite:** GAP-S07 (DailyActivity + streak), GAP-S09 (Progress/Task API) must be completed.

### Epic 3.1: Streak & Activity Calendar

**BA Reference:** F-S09 (BS-68 through BS-75)  
**Backend Dependency:** GAP-S07

| Task ID | Title | Type | Priority | Est. | Acceptance Criteria |
|---------|-------|------|----------|------|---------------------|
| S3-001 | Install react-activity-calendar or equivalent | Infra | High | 0.5h | Install heatmap calendar package. Evaluate `react-activity-calendar` or custom implementation. |
| S3-002 | Create DailyActivity type + API endpoints | Infra | High | 0.5h | TypeScript `DailyActivity` interface. Add `GET /activity/calendar?year=`, `GET /activity/streak` to API_ENDPOINTS. |
| S3-003 | Create `useGetActivityCalendarService` query | Service | High | 1h | Fetch 365 days of activity data for heatmap rendering. Cache 5 min. |
| S3-004 | Create `useGetStreakService` query | Service | High | 0.5h | Fetch `{ currentStreak, longestStreak, lastActivityDate }`. |
| S3-005 | Build StreakHeatmapCalendar component | Feature | High | 5h | GitHub-style heatmap (BS-72): 365 days, 5-level color coding (gray, light green → darkest green) by XP earned. Tooltip on hover (BS-73): date, XP, flashcards, quizzes. Responsive layout. |
| S3-006 | Build StreakCounter component | Feature | High | 2h | 🔥 fire icon + current streak number. Shown in header and dashboard. If streak = 0, show encouraging message (BS-74). |
| S3-007 | Integrate streak into Header component | UX | Medium | 1h | Add StreakCounter next to NotificationBell in Header. Show `🔥 12` format. |
| S3-008 | Create full streak calendar page/section in Profile | Feature | Medium | 2h | Full-year heatmap on profile page. Show current streak, longest streak, total active days. Per UC-S10. |

### Epic 3.2: Learning Plan Dashboard

**BA Reference:** F-S07 (BS-51 through BS-55)  
**Backend Dependency:** GAP-S09

| Task ID | Title | Type | Priority | Est. | Acceptance Criteria |
|---------|-------|------|----------|------|---------------------|
| S3-009 | Create LearningPlan query services | Service | High | 1h | `useGetMyLearningPlansService` (GET /learning-plans/user/:userId), `useGetLearningPlanDetailService` (GET /learning-plans/:id). |
| S3-010 | Implement Learning Plans list page | Feature | High | 3h | Replace stub with real page. Each plan displayed as card (BS-51): type/title, description, date range, progress bar (mastered/total). Expired plans show "Hết hạn" badge (BS-54). |
| S3-011 | Implement Learning Plan detail page | Feature | High | 3h | Replace stub. Show plan details + list of linked vocabulary words with status chips (new/learning/mastered) color-coded (BS-53). Progress percentage at top (BS-52). |
| S3-012 | Add "Add all plan words to my list" action | Feature | Medium | 2h | Button to batch-add all vocabulary words from a plan to user's personal list. Show count of words already added vs new. |

### Epic 3.3: Dashboard V2

**BA Reference:** F-S12 (BS-87, BS-88, BS-89)

| Task ID | Title | Type | Priority | Est. | Acceptance Criteria |
|---------|-------|------|----------|------|---------------------|
| S3-013 | Add daily goals progress rings | Feature | High | 3h | Per BS-85: 3 circular progress indicators — flashcards reviewed today / dailyFlashcardGoal, quiz questions / dailyQuizGoal, new words added / dailyNewWordsGoal. Pull goals from UserSettings. |
| S3-014 | Add mini streak heatmap (30 days) | Feature | Medium | 2h | Compact heatmap of last 30 days on dashboard. Click opens full calendar on profile page. |
| S3-015 | Add leaderboard preview widget | Feature | Low | 1.5h | Show current rank + top 5 students (weekly XP). "Xem bảng xếp hạng" link. Placeholder until gamification backend is ready. |
| S3-016 | Improve due-for-review card with real SM-2 data | Enhancement | High | 1h | Query `UserVocabulary` where `nextReviewDate <= now` for count. Show "N từ cần ôn tập" + "Ôn tập ngay" CTA. |

### Sprint 3 Summary

| Metric | Value |
|--------|-------|
| Total tasks | 16 |
| Total estimated hours | ~29h |
| High priority | 11 |
| Medium priority | 4 |
| Low priority | 1 |
| Backend dependencies | GAP-S07, GAP-S09 |

---

## 6. Sprint 4 — SEO Content & Gamification (2 weeks)

> **Goal:** Build public SSG/ISR pages for teacher-authored blog posts and lessons (primary SEO strategy). Implement XP, levels, badges, and leaderboard.  
>  
> **⚠️ Backend Prerequisite:** GAP-S02 (Posts), GAP-S03 (Lessons), GAP-S06 (Gamification) must be completed.

### Epic 4.1: Teacher Blog Posts (SEO)

**BA Reference:** F-S02.1, F-S02.3 (BS-09 through BS-15, BS-21, BS-22)  
**Backend Dependency:** GAP-S02

| Task ID | Title | Type | Priority | Est. | Acceptance Criteria |
|---------|-------|------|----------|------|---------------------|
| S4-001 | Create Post types + API endpoint constants | Infra | High | 0.5h | TypeScript `Post`, `PostCategory` interfaces. API endpoints: `GET /posts`, `GET /posts/:slug`. |
| S4-002 | Create `useGetPostListService` + `useGetPostDetailService` | Service | High | 1h | Post list with pagination/category filter. Single post by slug. |
| S4-003 | Build `/posts` listing page (SSG + ISR) | Feature | High | 4h | Public page (no auth). SSG with ISR (revalidate: 60s). Category filter sidebar. Post cards: title, excerpt, cover image, author, date, view count, category badge. Pagination. Route: `/posts` (English route, per project convention). |
| S4-004 | Build `/posts/[slug]` detail page (SSG + ISR) | Feature | High | 5h | `generateStaticParams` for all published post slugs. Full article with rich text rendering (DOMPurify for XSS — SEC-S04). Author info, published date, category, view count. Open Graph meta tags (SEO-03). JSON-LD Article schema (SEO-04). |
| S4-005 | Add related posts sidebar | Feature | Medium | 2h | Show 3-5 related posts (same category) on single post page. |
| S4-006 | Add "Featured Posts" section to landing page | Feature | Medium | 2h | Show latest 3 published posts on the landing page. SSG. |

### Epic 4.2: Teacher Lessons (SEO)

**BA Reference:** F-S02.2 (BS-16 through BS-20)  
**Backend Dependency:** GAP-S03

| Task ID | Title | Type | Priority | Est. | Acceptance Criteria |
|---------|-------|------|----------|------|---------------------|
| S4-007 | Create Lesson types + API endpoint constants | Infra | High | 0.5h | TypeScript `Lesson` interface with level, vocabularyIds, estimatedMinutes. API endpoints: `GET /lessons`, `GET /lessons/:slug`. |
| S4-008 | Create lesson query services | Service | High | 1h | `useGetLessonListService`, `useGetLessonDetailService`. |
| S4-009 | Build `/lessons` catalog page (SSG + ISR) | Feature | High | 3h | Public page. Level filter (Beginner/Intermediate/Advanced). Lesson cards: title, description, level badge (BS-17), estimated time (BS-20: "~10 phút"), cover image, author. Ordered by `orderIndex` (BS-19). |
| S4-010 | Build `/lessons/[slug]` detail page (SSG + ISR) | Feature | High | 5h | Full lesson content with rich text (DOMPurify). "Từ vựng trong bài" sidebar (BS-18) — lists referenced vocabulary words with "Thêm vào danh sách" action for logged-in users. Author, level badge, estimated time. JSON-LD LearningResource schema (SEO-04). |
| S4-011 | Implement `generateStaticParams` for lessons | Infra | High | 1h | Fetch all published lesson slugs at build time. ISR revalidation: 60s. |

### Epic 4.3: SEO Infrastructure

**BA Reference:** NFR 5.1 (SEO-01 through SEO-09)

| Task ID | Title | Type | Priority | Est. | Acceptance Criteria |
|---------|-------|------|----------|------|---------------------|
| S4-012 | Create dynamic sitemap.xml | Infra | High | 2h | Auto-generate sitemap with all published post and lesson URLs + `lastmod` dates. SEO-05. |
| S4-013 | Create robots.txt | Infra | High | 0.5h | Allow public pages (`/`, `/posts/*`, `/lessons/*`). Disallow authenticated pages (`/dashboard/*`, `/vocabulary/*`, `/review/*`, `/practice/*`, `/profile/*`, etc.). SEO-06. |
| S4-014 | Add Open Graph + metadata to all public pages | Infra | Medium | 1.5h | og:title, og:description, og:image, og:url for landing, post listing, lesson listing. SEO-03. |
| S4-015 | Update navigation to include Posts & Lessons | UX | Medium | 1h | Add "Bài viết" and "Bài học" links to Header (desktop) and public layout. Visible to both guests and logged-in users. |

### Epic 4.4: Gamification — XP & Levels

**BA Reference:** F-S08.1 (BS-56 through BS-59)  
**Backend Dependency:** GAP-S06

| Task ID | Title | Type | Priority | Est. | Acceptance Criteria |
|---------|-------|------|----------|------|---------------------|
| S4-016 | Create gamification types + API endpoints | Infra | High | 0.5h | TypeScript `UserXP`, `XPTransaction`, `Badge`, `UserBadge`. API endpoints: `GET /gamification/me`, `GET /gamification/leaderboard`, `GET /gamification/badges`. |
| S4-017 | Create gamification query services | Service | High | 1.5h | `useGetMyGamificationService`, `useGetLeaderboardService` (weekly/monthly/all), `useGetMyBadgesService`. |
| S4-018 | Build XP gain popup animation | Feature | Medium | 2h | "+5 XP" floating text animation (Framer Motion) near action area after flashcard review, quiz answer, etc. Per BS-59. |
| S4-019 | Build level-up celebration | Feature | Medium | 3h | Full-screen confetti animation + level badge reveal when student reaches a new level. Per BS-57. Framer Motion + canvas confetti. |
| S4-020 | Add Level badge to Header and Profile | UX | Medium | 1h | Show current level badge (e.g., "Lv.5 Trung cấp") in Header dropdown and profile page. |

### Epic 4.5: Gamification — Badges & Leaderboard

**BA Reference:** F-S08.2, F-S08.3 (BS-60 through BS-67)  
**Backend Dependency:** GAP-S06

| Task ID | Title | Type | Priority | Est. | Acceptance Criteria |
|---------|-------|------|----------|------|---------------------|
| S4-021 | Build badge earned popup | Feature | Medium | 2h | Per BS-62: "🎉 Bạn đã đạt được huy hiệu: [Name]!" modal with badge icon and description. +50 XP bonus shown. |
| S4-022 | Build badges grid on Profile page | Feature | Medium | 2h | Display all earned badges as a grid. Unearned badges shown as grayscale with "?" icon. Tooltip shows condition. |
| S4-023 | Implement Leaderboard page | Feature | High | 4h | Replace stub. Three tabs: Weekly / Monthly / All-Time (BS-64). Each row: rank, avatar, username, level badge, XP. Top 50 displayed. Sticky "My Rank" bar at bottom (BS-65). |
| S4-024 | Add leaderboard dashboard widget (with real data) | Enhancement | Medium | 1h | Replace placeholder leaderboard preview on dashboard with real data from `useGetLeaderboardService`. |

### Sprint 4 Summary

| Metric | Value |
|--------|-------|
| Total tasks | 24 |
| Total estimated hours | ~47h |
| High priority | 16 |
| Medium priority | 8 |
| Low priority | 0 |
| Backend dependencies | GAP-S02, GAP-S03, GAP-S06 |

---

## 7. Sprint 5 — OAuth Login, Final Polish & Performance (1 week)

> **Goal:** Implement social authentication (Google & Facebook), final UX polish, performance optimization, and pre-launch checklist.  
>  
> **⚠️ Backend Prerequisite:** GAP-S01 (OAuth callback routes) must be completed.

### Epic 5.1: OAuth Login

**BA Reference:** F-S01.2 (BS-04 through BS-07)  
**Backend Dependency:** GAP-S01

| Task ID | Title | Type | Priority | Est. | Acceptance Criteria |
|---------|-------|------|----------|------|---------------------|
| S5-001 | Add Google login button on Login page | Feature | Medium | 2h | "Đăng nhập bằng Google" button. Initiates OAuth redirect to `GET /auth/google`. Handle callback → store tokens → redirect. |
| S5-002 | Add Facebook login button on Login page | Feature | Medium | 2h | "Đăng nhập bằng Facebook" button. Same flow as Google. |
| S5-003 | Add OAuth buttons to Register page | Feature | Medium | 1h | "Đăng ký bằng Google" / "Đăng ký bằng Facebook" buttons. Same redirect flow. |
| S5-004 | Handle OAuth callback page | Feature | Medium | 2h | Create `/auth/callback` page to receive OAuth redirect with tokens. Parse tokens from URL params, store, and redirect to dashboard. |
| S5-005 | Add account linking info on Profile | UX | Low | 1h | Show which providers are linked (Google ✓, Facebook ✗). Future: allow linking/unlinking. Per BS-07. |

### Epic 5.2: Accessibility & Keyboard Navigation

**BA Reference:** NFR 5.3, 5.4 (USE-S06, A11Y-01 through A11Y-04)

| Task ID | Title | Type | Priority | Est. | Acceptance Criteria |
|---------|-------|------|----------|------|---------------------|
| S5-006 | Audit and add ARIA labels across interactive components | A11Y | Medium | 2h | Flashcard, quiz, navigation, notification bell — all must have `aria-label`. Per A11Y-02. |
| S5-007 | Test and fix focus management | A11Y | Medium | 1.5h | After quiz answer, flashcard flip, form submit — focus must move appropriately. Per A11Y-03. |
| S5-008 | Ensure color independence | A11Y | Medium | 1h | All status badges, calendar heatmap, progress indicators must use text labels in addition to color. Per A11Y-04. |

### Epic 5.3: Performance & SEO Verification

**BA Reference:** NFR 5.1, 5.2 (PERF-S01 through PERF-S06, SEO-08)

| Task ID | Title | Type | Priority | Est. | Acceptance Criteria |
|---------|-------|------|----------|------|---------------------|
| S5-009 | Run Lighthouse audit on all public pages | QA | High | 2h | Landing, post listing, post detail, lesson pages must score ≥ 90 (Performance). Fix any issues found. Per SEO-08, PERF-S01. |
| S5-010 | Optimize images with Next.js Image component | Perf | Medium | 1.5h | Replace any raw `<img>` tags with Next.js `<Image>`. Ensure WebP + lazy loading + responsive srcSet. Per PERF-S03. |
| S5-011 | Verify code splitting and bundle sizes | Perf | Medium | 1h | Each route should be a separate chunk. Initial landing page bundle < 150KB gzipped. Per PERF-S04. |
| S5-012 | Tune TanStack Query cache times | Perf | Low | 0.5h | Vocabulary: 5min staleTime, Leaderboard: 5min, User profile: 30s. Per PERF-S05. |

### Epic 5.4: Final Polish

| Task ID | Title | Type | Priority | Est. | Acceptance Criteria |
|---------|-------|------|----------|------|---------------------|
| S5-013 | Add offline awareness banner | UX | Low | 1h | Per USE-S07: If user loses connection, show non-blocking banner "Bạn đang ngoại tuyến." |
| S5-014 | Add loading/error states audit | UX | Medium | 2h | Per USE-S03: Verify all data sections have skeleton loaders + error fallback + empty states. Fix any gaps. |
| S5-015 | Clean up backup files and unused code | Tech Debt | Low | 0.5h | Remove `page.tsx.bak`, unused imports, dead code. |
| S5-016 | Update middleware to new "proxy" convention | Tech Debt | Medium | 1h | Next.js 16 deprecated "middleware" in favor of "proxy". Migrate to fix build warning. |

### Sprint 5 Summary

| Metric | Value |
|--------|-------|
| Total tasks | 16 |
| Total estimated hours | ~22h |
| High priority | 1 |
| Medium priority | 11 |
| Low priority | 4 |
| Backend dependencies | GAP-S01 |

---

## 8. Cross-Sprint Concerns

### 8.1 Testing Strategy

| Level | Scope | Tool | Sprint |
|-------|-------|------|--------|
| Unit Tests | SM-2 algorithm, quiz logic, utility functions | Jest / Vitest | Sprint 2 |
| Component Tests | Flashcard, Quiz components, Form validation | React Testing Library | Sprint 2–3 |
| E2E Tests | Auth flow, complete flashcard session, quiz completion | Playwright / Cypress | Sprint 4–5 |
| Visual Regression | Key pages (dashboard, vocabulary, flashcard) | Chromatic or Percy | Sprint 5 |

### 8.2 Security Checklist

| Item | BA Ref | Sprint |
|------|--------|--------|
| DOMPurify for teacher-authored rich text content | SEC-S04 | Sprint 4 |
| Verify all PATCH/DELETE endpoints validate ownership | SEC-S03 (backend) | Sprint 1 (flag to BE team) |
| Confirm rate limiting on quiz/XP endpoints | SEC-S06 (backend) | Sprint 2 (flag to BE team) |
| Validate no XSS in user-generated content (usernames, etc.) | SEC-S04 | Sprint 5 |

### 8.3 Backend Dependency Timeline

```
Week 1  │ Sprint 1 (FE) ─── no backend deps
        │
Week 2  │ Sprint 2 start ─── ⚠️ GAP-S04, GAP-S05, GAP-S08 must be ready
Week 3  │ Sprint 2 end
        │
Week 4  │ Sprint 3 start ─── ⚠️ GAP-S07, GAP-S09 must be ready
Week 5  │ Sprint 3 end
        │
Week 6  │ Sprint 4 start ─── ⚠️ GAP-S02, GAP-S03, GAP-S06 must be ready
Week 7  │ Sprint 4 end
        │
Week 8  │ Sprint 5 start ─── ⚠️ GAP-S01 must be ready
        │ Sprint 5 end ─── 🚀 LAUNCH READY
```

> **Recommendation:** Backend team should begin GAP-S04 and GAP-S08 in Week 1 (parallel with FE Sprint 1) so they are ready for Week 2.

---

## 9. Task Summary by Priority

| Priority | Count | Hours |
|----------|-------|-------|
| High | 46 | ~100h |
| Medium | 35 | ~49h |
| Low | 8 | ~7h |
| **Total** | **89** | **~160h** |

---

## 10. Risk Register

| Risk ID | Risk | Impact | Probability | Mitigation |
|---------|------|--------|-------------|------------|
| R-01 | Backend GAP delays block Sprint 2+ | High | Medium | FE team builds with mock data / MSW. Backend team starts GAPs in Week 1. |
| R-02 | SM-2 algorithm edge cases cause incorrect intervals | Medium | Low | Comprehensive unit tests with known input/output pairs. Refer to SM-2 reference implementation. |
| R-03 | SSG/ISR pages have stale content | Low | Medium | Set revalidation to 60s. Add manual revalidation webhook from admin panel on content publish. |
| R-04 | Performance regression from heavy animations | Medium | Low | Use `prefers-reduced-motion`. Test on low-end devices. Lazy-load confetti/animation libs. |
| R-05 | XP farming / quiz abuse | High | Medium | Rate limiting (SEC-S06) on backend. Frontend cannot prevent server-side — flag to backend team. |
| R-06 | Next.js 16 middleware deprecation breaks auth guard | Medium | High | Sprint 5 includes S5-016 to migrate to "proxy" convention. Plan for early if breaking. |

---

## 11. Definition of Done (DoD)

A task is considered DONE when ALL of the following are true:

1. ✅ Code implemented matching all acceptance criteria listed in the task
2. ✅ TypeScript compiles with zero errors (`npm run build` passes)
3. ✅ Responsive design verified: mobile (375px), tablet (768px), desktop (1280px)
4. ✅ Loading states, error states, and empty states are handled
5. ✅ Toast notifications shown for all user mutations
6. ✅ Vietnamese labels/messages used for all UI text
7. ✅ Code reviewed and approved by at least 1 team member
8. ✅ No console errors or warnings in development mode
9. ✅ Accessibility: keyboard navigable, no contrast violations

---

## 12. Appendix: Task Index (Quick Reference)

### Sprint 1 (S1): 12 tasks
`S1-001` Profile edit form · `S1-002` Avatar upload · `S1-003` Change password · `S1-004` Delete confirmation · `S1-005` Due-for-review card · `S1-006` Onboarding empty state · `S1-007` Recent activity · `S1-008` Status badge on catalog · `S1-009` My List summary header · `S1-010` Debounce 300ms · `S1-011` Notification dropdown · `S1-012` Refetch on focus

### Sprint 2 (S2): 21 tasks
`S2-001`–`S2-009` SM-2 Flashcard (9 tasks) · `S2-010`–`S2-017` Quiz Engine (8 tasks) · `S2-018`–`S2-021` User Settings (4 tasks)

### Sprint 3 (S3): 16 tasks
`S3-001`–`S3-008` Streak & Calendar (8 tasks) · `S3-009`–`S3-012` Learning Plans (4 tasks) · `S3-013`–`S3-016` Dashboard V2 (4 tasks)

### Sprint 4 (S4): 24 tasks
`S4-001`–`S4-006` Blog Posts (6 tasks) · `S4-007`–`S4-011` Lessons (5 tasks) · `S4-012`–`S4-015` SEO Infra (4 tasks) · `S4-016`–`S4-020` XP & Levels (5 tasks) · `S4-021`–`S4-024` Badges & Leaderboard (4 tasks)

### Sprint 5 (S5): 16 tasks
`S5-001`–`S5-005` OAuth (5 tasks) · `S5-006`–`S5-008` Accessibility (3 tasks) · `S5-009`–`S5-012` Performance (4 tasks) · `S5-013`–`S5-016` Final Polish (4 tasks)
