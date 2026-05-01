# Change Request: Reminder & Notification System — Student Daily Goal Reminder

**CR ID:** CR-2026-04-30-reminder-notification-system  
**Date:** 2026-04-30  
**Status:** 📋 Drafted — Pending Sprint Planning  
**Target:** `over-the-world-be` + `over-the-world-fe-student`  
**Priority:** Medium

---

## 1. Background

The student portal now displays a **Today's Goal Widget** (`SidebarGoalWidget`) in the sidebar showing daily progress toward flashcard, quiz, and new-word goals sourced from `UserSettings`. However, there is currently no mechanism to **remind users** when they have not yet met their daily goals.

User settings already include `notificationEnabled` and `emailNotificationEnabled` toggles which are saved to the database but **not yet consumed** by any reminder logic on the backend.

This CR defines a 3-phase roadmap to implement the reminder system — from a lightweight in-app banner (Phase 1) up to full Web Push notifications (Phase 3) — allowing the team to ship incrementally based on capacity.

---

## 2. Problem Statement

1. Users set daily goals but receive no reminder if they forget to study.
2. `UserSettings.notificationEnabled` and `emailNotificationEnabled` are stored in DB but never read by the backend when sending notifications — user preferences are ignored.
3. There is no cron-based or event-based mechanism to trigger reminders.

---

## 3. Proposed Solution — 3 Phases

### Phase 1 — In-App Evening Banner (Frontend only)

**Trigger:** User opens the app after 18:00 local time AND has not yet met at least one daily goal.  
**Output:** A dismissible warning banner rendered inside the dashboard or sidebar.  
**Dismiss persistence:** Stored in `localStorage` with a `YYYY-MM-DD` key so it does not reappear on the same day.

#### Affected files:
| File | Change |
|------|--------|
| `src/components/layout/SidebarGoalWidget.tsx` | Add time-based + goal-based condition to show inline reminder badge |
| `src/app/(protected)/dashboard/page.tsx` | Optionally render a top-of-page reminder banner |

#### Logic:
```typescript
const hour = new Date().getHours();
const isEvening = hour >= 18;
const dismissedKey = `reminder-dismissed-${today}`;
const isDismissed = localStorage.getItem(dismissedKey) === "true";
const hasUnmetGoal = flashcardsDone < flashcardGoal || quizDone < quizGoal || newWordsDone < newWordsGoal;

if (isEvening && hasUnmetGoal && !isDismissed) {
  // show banner
}
```

#### Effort estimate: 2–3 hours  
#### Backend changes: None  
#### Risk: None

---

### Phase 2 — Email Reminder via Cron Job (Backend)

**Trigger:** Scheduled cron job runs at 20:00 server time every day.  
**Output:** Sends a reminder email to users who:
- Have `emailNotificationEnabled = true`
- Have NOT yet met their daily goal (checked via `DailyActivity`)

#### Backend changes required:

**1. Fix existing bug — respect `emailNotificationEnabled`**

In `notification.service.ts → sendEmailNotification()`, add check:
```typescript
// Before sending email to user:
const userSettings = await this.userSettingsRepo.findOne({ where: { userId: user.id } });
if (!userSettings?.emailNotificationEnabled) continue;
```

**2. New cron job — `ReminderModule`**

```typescript
@Cron('0 20 * * *', { timeZone: 'Asia/Ho_Chi_Minh' })
async sendDailyReminders() {
  const today = new Date().toISOString().slice(0, 10);
  
  // Get all users with email notification enabled
  const settings = await this.userSettingsRepo.find({
    where: { emailNotificationEnabled: true },
    relations: ['user'],
  });

  for (const setting of settings) {
    const activity = await this.activityRepo.findOne({
      where: { userId: setting.userId, date: today },
    });

    const flashcardsDone = activity?.flashcardsReviewed ?? 0;
    const quizDone = activity?.quizQuestionsAnswered ?? 0;
    const newWordsDone = activity?.newWordsAdded ?? 0;

    const hasUnmetGoal =
      flashcardsDone < setting.dailyFlashcardGoal ||
      quizDone < setting.dailyQuizGoal ||
      newWordsDone < setting.dailyNewWordsGoal;

    if (hasUnmetGoal && setting.user?.email) {
      await this.mailService.sendReminderEmail(setting.user.email, {
        flashcardsDone, flashcardGoal: setting.dailyFlashcardGoal,
        quizDone, quizGoal: setting.dailyQuizGoal,
        newWordsDone, newWordsGoal: setting.dailyNewWordsGoal,
      });
    }
  }
}
```

**3. New email template — `MailService.sendReminderEmail()`**

Plain HTML email with progress summary and CTA button linking to the app.

#### New module: `src/reminder/reminder.module.ts`

Dependencies:
- `@nestjs/schedule` (install: `npm install @nestjs/schedule`)
- `DailyActivityModule`
- `UserSettingsModule`
- `MailModule`

#### Effort estimate: 1 day  
#### Backend changes: New module, bug fix in NotificationService  
#### Risk: Low — depends on SMTP config being correctly set in `.env`

---

### Phase 3 — Web Push Notification (Browser / Mobile)

**Trigger:** Same cron logic as Phase 2, but delivers a browser push notification instead of (or in addition to) email.  
**Output:** Native OS notification even when the app tab is closed.

#### Architecture:
```
Backend                        Frontend
───────                        ────────
web-push library          ←→  Service Worker (sw.js)
VAPID keypair                  navigator.serviceWorker.register()
PushSubscription entity        Notification.requestPermission()
(new DB table + migration)     PushManager.subscribe()
                               POST /push/subscribe → save to DB
```

#### New entities required:
```typescript
@Entity()
export class PushSubscription {
  @PrimaryGeneratedColumn('uuid') id: string;
  @Column({ type: 'uuid' }) userId: string;
  @Column('text') endpoint: string;
  @Column('text') p256dh: string;   // public key
  @Column('text') auth: string;     // auth secret
  @CreateDateColumn() createdAt: Date;
}
```

#### New migration required: `CreatePushSubscriptionTable`

#### New endpoints:
| Method | Path | Description |
|--------|------|-------------|
| POST | `/push/subscribe` | Save push subscription from browser |
| DELETE | `/push/unsubscribe` | Remove subscription on permission revoke |

#### Frontend flow:
1. On Settings page — "Bật thông báo trình duyệt" button
2. `Notification.requestPermission()` → if granted → `PushManager.subscribe()`
3. POST subscription to `/push/subscribe`
4. Service Worker handles `push` events and shows notification

#### Effort estimate: 4–5 days  
#### Backend changes: New entity, migration, module, VAPID config  
#### Risk: High — requires HTTPS in production, complex browser permission UX, Safari partial support

---

## 4. Current State Audit

| Item | Status | Notes |
|------|--------|-------|
| `UserSettings.notificationEnabled` stored in DB | ✅ | Saved correctly |
| `UserSettings.emailNotificationEnabled` stored in DB | ✅ | Saved correctly |
| Settings UI toggles | ✅ | Functional, auto-save |
| `emailNotificationEnabled` respected when sending | ❌ | Bug — ignored in `sendEmailNotification()` |
| `notificationEnabled` respected anywhere | ❌ | Not consumed |
| Email infrastructure (SMTP + Nodemailer) | ✅ | Working for password reset |
| `DailyActivity` data available for goal checking | ✅ | `GET /activity/today` |
| Cron infrastructure | ❌ | `@nestjs/schedule` not installed |
| Web Push infrastructure | ❌ | Not started |
| In-app reminder banner | ❌ | Not implemented |

---

## 5. Implementation Roadmap

```
Phase 1 (May 2026)        Phase 2 (June 2026)        Phase 3 (July 2026+)
──────────────────        ───────────────────        ─────────────────────
In-app banner         →   Email cron reminder    →   Web Push notification
• Frontend only           • New ReminderModule        • Service Worker
• localStorage dismiss    • Fix email pref bug        • VAPID setup
• 2–3 hours               • ~1 day                    • ~4–5 days
• Risk: None              • Risk: Low                 • Risk: High
```

**Recommendation:** Implement Phase 1 immediately (can ship same day). Schedule Phase 2 in next sprint. Phase 3 deferred until user base justifies the investment.

---

## 6. Acceptance Criteria

### Phase 1
- [ ] Banner appears in dashboard after 18:00 local time when at least one goal is unmet
- [ ] Banner is dismissible and does not reappear until the next calendar day
- [ ] Banner does not appear if all goals are already met

### Phase 2
- [ ] Cron job runs at 20:00 Vietnam time daily
- [ ] Email is only sent to users with `emailNotificationEnabled = true`
- [ ] Email is not sent if user has already met all goals for that day
- [ ] `emailNotificationEnabled = false` users receive no email (bug fix)
- [ ] Email contains current progress vs goal and a CTA link to the app

### Phase 3
- [ ] User can enable/disable browser push from Settings page
- [ ] Push notification delivered within 5 minutes of cron trigger
- [ ] Works on Chrome, Firefox, Edge (Safari optional)
- [ ] Unsubscribe on permission revoke is handled gracefully

---

## 7. Out of Scope

- SMS reminders
- Mobile native push (React Native / Expo)
- Personalized reminder time (user-configurable hour) — future enhancement
- Retry logic for failed push deliveries
