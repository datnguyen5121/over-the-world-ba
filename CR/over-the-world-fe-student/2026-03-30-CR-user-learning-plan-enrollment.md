# Change Request: UserLearningPlan — Student Enrollment in Learning Plans

**CR ID:** CR-2026-03-30-user-learning-plan-enrollment  
**Date:** 2026-03-30  
**Status:** 📋 Drafted — Pending Next Sprint  
**Target:** `over-the-world-be` + `over-the-world-fe-student`  
**Related BA requirement:** F-09b (BA_REQUIREMENTS.md v1.4)

---

## 1. Background

In sprint 2026-03-30, `LearningPlan` was redesigned from a user-bound record to a reusable **admin template** (with `durationDays` replacing `userId`/`startDate`/`endDate`).

This CR defines the **next step**: allowing students to **enroll** in a template, creating a personal `UserLearningPlan` record with concrete dates and status tracking.

---

## 2. Problem Statement

Currently, students have no way to associate themselves with a `LearningPlan`. The `LearningPlan` entity is a template only. To complete the study-plan workflow, students need:

1. A mechanism to pick a plan template and enroll in it.
2. A personal enrollment record storing their `startDate`, `endDate`, and `status`.
3. A UI in the student portal to view and manage their enrolled plans.

---

## 3. Proposed Solution

### 3.1 New Entity: `UserLearningPlan`

```typescript
@Entity()
export class UserLearningPlan {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @ManyToOne(() => User, (user) => user.learningPlans, { onDelete: 'CASCADE' })
  @JoinColumn({ name: 'userId' })
  user: User;

  @ManyToOne(() => LearningPlan, { onDelete: 'CASCADE' })
  @JoinColumn({ name: 'learningPlanId' })
  learningPlan: LearningPlan;

  @Column({ type: 'date' })
  startDate: string;          // ISO 8601 — e.g. "2026-04-01"

  @Column({ type: 'date' })
  endDate: string;            // startDate + durationDays

  @Column({
    type: 'enum',
    enum: ['active', 'completed', 'abandoned'],
    default: 'active',
  })
  status: 'active' | 'completed' | 'abandoned';

  @CreateDateColumn({ type: 'timestamp' })
  createdAt: Date;

  @UpdateDateColumn({ type: 'timestamp' })
  updatedAt: Date;
}
```

**Computed field:** `endDate = startDate + learningPlan.durationDays - 1 day`

### 3.2 API Endpoints

| Method | Endpoint | Role | Description |
|--------|----------|------|-------------|
| `POST` | `/user-learning-plans` | STUDENT | Enroll in a plan template |
| `GET` | `/user-learning-plans/me` | STUDENT | Get all my enrolled plans |
| `GET` | `/user-learning-plans/:id` | STUDENT | Get single enrollment detail |
| `PATCH` | `/user-learning-plans/:id/status` | STUDENT | Update status (complete / abandon) |
| `DELETE` | `/user-learning-plans/:id` | STUDENT | Unenroll |

### 3.3 POST `/user-learning-plans` — Enroll

**Request body:**
```json
{
  "learningPlanId": "uuid",
  "startDate": "2026-04-01"
}
```

**Server logic:**
1. Validate `learningPlanId` exists in `LearningPlan`.
2. Compute `endDate = startDate + durationDays - 1`.
3. Check user does not already have an `active` enrollment for the same plan.
4. Create and return `UserLearningPlan` record.

**Response:** `201 Created`
```json
{
  "id": "uuid",
  "learningPlan": { "id": "...", "type": "...", "durationDays": 7 },
  "startDate": "2026-04-01",
  "endDate": "2026-04-07",
  "status": "active",
  "createdAt": "..."
}
```

### 3.4 Business Rules

| Rule ID | Description |
|---------|-------------|
| BR-60 | `learningPlanId` must reference an existing `LearningPlan` template. Return `404` if not found. |
| BR-61 | `startDate` must be today or a future date. Past start dates are rejected with `400 Bad Request`. |
| BR-62 | `endDate` is calculated server-side: `endDate = startDate + durationDays - 1 day`. Clients must not send `endDate`. |
| BR-63 | A student may not enroll in the same plan twice while an `active` enrollment exists. Return `409 Conflict`. |
| BR-64 | `status` transitions: `active → completed`, `active → abandoned`. `completed` and `abandoned` are terminal states. |
| BR-65 | Deleting a `UserLearningPlan` record does **not** affect the `LearningPlan` template. |

---

## 4. Student Portal UI Changes

### 4.1 New Pages / Components (`over-the-world-fe-student`)

| Component | Path | Description |
|-----------|------|-------------|
| Learning Plans catalog | `/learning-plans` | Browse available templates; each card shows type, description, duration badge (e.g. "7 days"). Enroll button opens date picker. |
| My Plans page | `/my-plans` | List of student's enrolled plans with status badges (Active 🟢 / Completed ✅ / Abandoned ⛔). |
| Plan detail | `/my-plans/[id]` | Shows plan name, dates, progress %, linked vocabulary sessions. |
| Enroll modal | (inline on catalog) | Date picker for `startDate` → computed `endDate` preview → Confirm button. |

### 4.2 Enroll Flow

```
Catalog page
  → Student clicks "Enroll"
  → Modal opens: Date picker (startDate, default = today)
  → Preview: "End date: YYYY-MM-DD (7 days)"
  → Confirm → POST /user-learning-plans
  → Toast: "Enrolled successfully!"
  → Redirect to /my-plans
```

---

## 5. Database Migration

New migration required: `CreateUserLearningPlanTable`

```sql
CREATE TABLE user_learning_plan (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  "userId" UUID NOT NULL REFERENCES "user"(id) ON DELETE CASCADE,
  "learningPlanId" UUID NOT NULL REFERENCES learning_plan(id) ON DELETE CASCADE,
  "startDate" DATE NOT NULL,
  "endDate" DATE NOT NULL,
  status VARCHAR(20) NOT NULL DEFAULT 'active',
  "createdAt" TIMESTAMP DEFAULT NOW(),
  "updatedAt" TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_user_learning_plan_user ON user_learning_plan("userId");
CREATE UNIQUE INDEX idx_user_learning_plan_active 
  ON user_learning_plan("userId", "learningPlanId") 
  WHERE status = 'active';
```

The unique partial index on `(userId, learningPlanId) WHERE status = 'active'` enforces BR-63 at the database level.

---

## 6. Impact Analysis

| Area | Change Required | Effort |
|------|----------------|--------|
| BE: New entity `UserLearningPlan` | Yes | Small |
| BE: New module & service | Yes | Medium |
| BE: New controller (5 endpoints) | Yes | Medium |
| BE: Migration | Yes | Small |
| BE: `User` entity — add `OneToMany userLearningPlans` | Yes | Trivial |
| FE-Student: New pages (catalog, my-plans, detail) | Yes | Large |
| FE-Student: Enroll modal + date picker | Yes | Medium |
| FE-Student: API service hooks | Yes | Small |

---

## 7. Out of Scope (This CR)

- Progress tracking within an enrolled plan (separate CR)
- Admin ability to view all student enrollments
- Push notifications when a plan is about to end
- Spaced repetition scheduling tied to plan duration

---

## 8. Test Checklist

- [ ] Enroll with a valid `learningPlanId` and today's date → `201` with correct `endDate`
- [ ] Enroll with a past `startDate` → `400 Bad Request`
- [ ] Enroll in the same active plan twice → `409 Conflict`
- [ ] Enroll with invalid `learningPlanId` → `404 Not Found`
- [ ] `GET /user-learning-plans/me` returns only the current user's plans
- [ ] `PATCH status` to `completed` → status updates; subsequent `PATCH` to `active` is rejected
- [ ] `DELETE` removes enrollment but keeps `LearningPlan` template intact
- [ ] Student portal: Enroll modal computes `endDate` preview correctly
- [ ] Student portal: My Plans page shows correct status badges
- [ ] Migration creates partial unique index; duplicate active enrollments blocked at DB level

---

*Drafted: 2026-03-30*  
*Author: BA Agent*  
*Next action: Schedule for next sprint planning session*
