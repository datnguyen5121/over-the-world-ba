# Sơ đồ luồng học tập — Over The World Student App

> Phiên bản: 1.0 · Ngày: 2026-04-06
> Mục đích: Tài liệu hóa toàn bộ luồng logic học từ vựng, ôn tập, luyện tập và hệ thống tracking/gamification.

---

## 1. Tổng quan kiến trúc

```
┌─────────────────────────────────────────────────────────────────┐
│                        STUDENT APP (Next.js 15)                 │
│                                                                 │
│  Dashboard ──→ Chọn Category ──→ 3 chế độ học                   │
│      │                            ├─ 📖 Browse (học từ mới)     │
│      │                            ├─ 🔄 Due Review (SM-2)       │
│      │                            └─ 🧪 Quiz (luyện tập)        │
│      │                                                          │
│      ├──→ /review (ôn tập tổng hợp)                             │
│      ├──→ /practice (quiz tất cả category)                      │
│      ├──→ /my-list (quản lý danh sách cá nhân)                  │
│      └──→ /my-plans (kế hoạch học tập)                          │
│                                                                 │
│  ┌────────────────────────────────────────────────────────┐     │
│  │ Data Layer (TanStack Query)                             │     │
│  │  Mutations: recordActivity, awardXp, addToList,         │     │
│  │            submitReview, submitQuizAnswer, completeQuiz  │     │
│  │  Queries:   userVocabList, dueFlashcards, streak,       │     │
│  │            gamification, activityCalendar, categories    │     │
│  └────────────────────────────────────────────────────────┘     │
│                          │                                      │
│                          ▼                                      │
│  ┌────────────────────────────────────────────────────────┐     │
│  │ Backend (NestJS 11 + TypeORM + PostgreSQL)              │     │
│  │  /user-vocabularies   — SM-2 data, status, favorites    │     │
│  │  /activity/record     — daily counters (increment)      │     │
│  │  /gamification/award-xp — XP + level + badges           │     │
│  │  /quizzes/*           — quiz session + results          │     │
│  │  /user-learning-plans — enrollment tracking             │     │
│  └────────────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. Tình hình hiện tại

### ✅ Hoạt động đúng

| Tính năng | Mô tả |
|-----------|-------|
| SM-2 Flashcard Review | Mỗi thẻ được PATCH `/user-vocabularies/{id}` với đầy đủ SM-2 data |
| Quiz Answer Submission | Mỗi câu trả lời được POST `/quizzes/{sessionId}/answer` |
| Quiz XP Award | Quiz completion gọi `awardXp()` đúng cách |
| Quiz Activity Recording | Quiz completion gọi `recordActivity({ quizQuestionsAnswered })` |
| Add to List | Thêm từ vào DS gọi `POST /user-vocabularies` → invalidate cache |
| Streak Display | Sidebar hiển thị streak từ `GET /activity/streak` |
| Dashboard Stats | Hiển thị total/new/learning/mastered/dueForReview |

### ❌ Các vấn đề hiện tại (7 items)

| # | Vấn đề | Mức độ | File liên quan |
|---|--------|--------|----------------|
| 1 | BrowseFlashcards ghi `newWordsAdded: vocabs.length` (50) thay vì số thực tế thêm | 🔴 Critical | `BrowseFlashcards.tsx` |
| 2 | Flashcard Review (ôn tập) không tặng XP | 🔴 Critical | `DueReviewSession.tsx`, `review/page.tsx` |
| 3 | Quiz đúng không cập nhật `UserVocabulary.status` | 🟠 High | `quiz.service.ts` (BE) |
| 4 | `xpEarned` gửi cả trong `recordActivity` VÀ `awardXp` → duplicate | 🟡 Medium | `practice/[sessionId]/page.tsx` |
| 5 | Category Progress không đổi nếu browse xong nhưng không thêm từ | 🟠 High | `learn-vocab/page.tsx`, `CategoryCard.tsx` |
| 6 | Browse 0 từ vẫn gọi `recordActivity` (ghi sai) | 🟡 Medium | `BrowseFlashcards.tsx` |
| 7 | Không có tài liệu flow → khó maintain | 🟡 Medium | — |

---

## 3. Luồng 1: Học từ vựng mới (Browse)

### 3.1 Entry point

```
Dashboard → Click CategoryCard → /learn-vocab/[categoryId]
```

### 3.2 Quyết định chế độ hiển thị

```
┌──────────────────────────────────────────────────────────┐
│ learn-vocab/[categoryId]/page.tsx                        │
│                                                          │
│ Fetch: useGetUserVocabularyListService (pageSizes=500)   │
│ Fetch: useGetCategoryListService                         │
│ Filter: words that belong to this categoryId             │
│                                                          │
│ ┌─ hasProgress = inCategory.length > 0 ?                 │
│ │                                                        │
│ ├─ NO  → PageState = "intro" → CategoryIntroView         │
│ │        → Nút "Bắt đầu học" → setPageState("browse")   │
│ │                                                        │
│ ├─ YES + dueCards.length > 0                             │
│ │        → PageState = "due-review" → DueReviewSession   │
│ │                                                        │
│ └─ YES + no due cards                                    │
│          → PageState = "browse" → BrowseFlashcards       │
└──────────────────────────────────────────────────────────┘
```

> ⚠️ **VẤN ĐỀ #5**: `hasProgress` chỉ dựa vào `inCategory.length > 0`. Nếu user browse hết flashcard nhưng KHÔNG thêm từ nào → lần sau quay lại vẫn hiện "intro" thay vì "tiếp tục học".
>
> **FIX**: Thêm localStorage tracking khi nhấn "Bắt đầu học".

### 3.3 Browse Flashcards flow

```
BrowseFlashcards.tsx
    │
    │  Fetch: useGetVocabularyListService (page=1, pageSizes=50, categoryId)
    │
    ├─ Hiển thị thẻ flip 3D (front: 字+pinyin, back: nghĩa+ví dụ)
    │
    ├─ Action: "Thêm vào DS" (nếu chưa có trong list)
    │   └─ POST /user-vocabularies { userId, vocabularyId }
    │   └─ Invalidate: ["userVocabularyList"]
    │
    ├─ Action: "Nghe phát âm" → Web Speech API (speakChineseText)
    │
    ├─ Navigation: ← → keys hoặc buttons, Space/Enter để lật thẻ
    │
    └─ Thẻ cuối → Nút "Hoàn thành 🎉"
        ├─ Gọi: recordActivity({ newWordsAdded: addedCount })
        ├─ Gọi: awardXp({ amount: addedCount * 2, source: "WORD_ADDED" })
        ├─ Chỉ gọi nếu addedCount > 0
        └─ router.push("/dashboard")
```

### 3.4 API calls (Browse)

| Thời điểm | API | Data | Side Effects |
|-----------|-----|------|--------------|
| Click "Thêm vào DS" | `POST /user-vocabularies` | `{ userId, vocabularyId }` | Invalidate `["userVocabularyList"]` |
| Click "Hoàn thành" | `POST /activity/record` | `{ newWordsAdded: addedCount }` | Invalidate `["streak", "activityCalendar", "activityToday", "myGamification"]` |
| Click "Hoàn thành" | `POST /gamification/award-xp` | `{ amount: addedCount*2, source: "WORD_ADDED" }` | Invalidate `["myGamification", "leaderboard"]` |

---

## 4. Luồng 2: Ôn tập Flashcard (SM-2 Review)

### 4.1 Hai entry point

```
Entry A: Dashboard → CategoryCard → /learn-vocab/[categoryId]
         (nếu hasProgress=true && dueCards.length > 0 → DueReviewSession)

Entry B: Dashboard → QuickActions "Ôn tập ngay" → /review
         (ôn TẤT CẢ category, dùng useGetDueFlashcardsService)
```

### 4.2 Review session flow

```
DueReviewSession.tsx / review/page.tsx
    │
    │  Cards: UserVocabulary[] có nextReviewDate ≤ now
    │
    ├─ Hiển thị thẻ flip 3D (front: 字+pinyin, back: nghĩa)
    │
    ├─ Lật thẻ → Hiện 6 nút đánh giá (SM-2 quality 0–5)
    │   ├─ 0: Quên hoàn toàn  ┐
    │   ├─ 1: Sai nhiều        ├─ quality < 3 → FAIL
    │   ├─ 2: Sai một phần    ┘  → reset interval=1, reps=0
    │   ├─ 3: Khó nhớ         ┐  → requeue ở cuối session
    │   ├─ 4: Nhớ tốt          ├─ quality ≥ 3 → PASS
    │   └─ 5: Hoàn hảo        ┘  → advance interval
    │
    ├─ Mỗi thẻ → PATCH /user-vocabularies/{id}
    │   {
    │     easeFactor, interval, repetitions,
    │     nextReviewDate, status, lastReviewed
    │   }
    │   Invalidate: ["dueFlashcards", "userVocabularyList"]
    │
    ├─ Thẻ fail (quality < 3) → requeue ở cuối hàng đợi
    │
    └─ Hết thẻ → Summary screen
        ├─ Hiển thị: correct/total, accuracy %, trophy
        ├─ recordActivity({ flashcardsReviewed: reviewed.length })
        ├─ awardXp({ amount: correctCount * 3, source: "FLASHCARD" })
        │   (chỉ gọi nếu correctCount > 0)
        └─ Buttons: "Ôn lại" | "Tiếp tục học" / "Về trang chủ"
```

### 4.3 SM-2 Algorithm (src/lib/sm2.ts)

```
calculateSM2(quality, easeFactor, interval, repetitions)

IF quality ≥ 3 (pass):
  reps == 0 → interval = 1 ngày
  reps == 1 → interval = 6 ngày
  reps >= 2 → interval = round(interval × easeFactor)
  repetitions++

IF quality < 3 (fail):
  interval = 1 ngày
  repetitions = 0

Luôn cập nhật:
  easeFactor = EF + (0.1 - (5 - q) × (0.08 + (5 - q) × 0.02))
  Min easeFactor = 1.3

nextReviewDate = now + interval ngày
```

### 4.4 Status progression từ review

```
quality ≥ 4 → status = "mastered"
quality == 3 → status = "learning"
quality < 3  → giữ nguyên status hiện tại
```

### 4.5 API calls (Review)

| Thời điểm | API | Data | Side Effects |
|-----------|-----|------|--------------|
| Đánh giá mỗi thẻ | `PATCH /user-vocabularies/{id}` | SM-2 fields + status | Invalidate `["dueFlashcards", "userVocabularyList"]` |
| Summary hiện | `POST /activity/record` | `{ flashcardsReviewed: count }` | Invalidate `["streak", "activityCalendar", "activityToday", "myGamification"]` |
| Summary hiện | `POST /gamification/award-xp` | `{ amount: correctCount*3, source: "FLASHCARD" }` | Invalidate `["myGamification", "leaderboard"]` |

---

## 5. Luồng 3: Luyện tập Quiz

### 5.1 Flow tổng thể

```
Dashboard → QuickActions "Luyện tập" → /practice
    │
    │  Chọn Quiz Mode:
    │  ┌─────────────────────────────────────────────┐
    │  │ MULTIPLE_CHOICE  │ Trắc nghiệm    │ 10 câu │
    │  │ FILL_IN_BLANK    │ Điền từ         │ 10 câu │
    │  │ PINYIN_MATCH     │ Ghép pinyin     │ 5 câu  │
    │  └─────────────────────────────────────────────┘
    │
    │  Optional: chọn category filter
    │  Validation: ≥ 4 từ (≥ 5 cho PINYIN_MATCH)
    │
    ├─ POST /quizzes/generate { quizType, categoryId?, limit }
    │  → Returns: { sessionId, questions[], totalQuestions }
    │  → sessionStorage.setItem(`quiz-${sessionId}`, data)
    │
    └─ Navigate: /practice/[sessionId]
```

### 5.2 Quiz execution

```
/practice/[sessionId]/page.tsx
    │
    │  Load quiz data from sessionStorage
    │
    ├─ MULTIPLE_CHOICE:
    │   Hiện 字 + pinyin → 4 options (A/B/C/D)
    │   Click → feedback ✓/✗ → lock
    │
    ├─ FILL_IN_BLANK:
    │   Hiện nghĩa → input text
    │   Normalize: lowercase + bỏ dấu → so sánh
    │   Hỗ trợ nhiều đáp án (acceptedAnswers)
    │   Button 💡 cho hint
    │
    ├─ PINYIN_MATCH:
    │   2 cột: 字 vs pinyin (xáo trộn)
    │   Click word → click pinyin → ghép
    │   Sai → shake đỏ, đúng → fade out
    │   Submit tất cả khi ghép xong
    │
    ├─ Mỗi câu trả lời:
    │   POST /quizzes/{sessionId}/answer
    │   { vocabularyId, question, correctAnswer, userAnswer, isCorrect }
    │   Backend: ghi QuizResult + increment session counters
    │
    └─ Hết câu → POST /quizzes/{sessionId}/complete
        → Returns: QuizSummary { totalQuestions, correctAnswers, accuracy, xpEarned }
        │
        ├─ recordActivity({ quizQuestionsAnswered: totalQuestions })
        │
        ├─ awardXp({ amount: xpEarned, source: "QUIZ", referenceId: sessionId })
        │
        ├─ sessionStorage.removeItem(`quiz-${sessionId}`)
        │
        └─ QuizResultScreen: trophy, score, time, XP, per-question details
```

### 5.3 API calls (Quiz)

| Thời điểm | API | Data | Side Effects |
|-----------|-----|------|--------------|
| Chọn mode + start | `POST /quizzes/generate` | `{ quizType, categoryId?, limit }` | Tạo QuizSession |
| Mỗi câu trả lời | `POST /quizzes/{sid}/answer` | `{ vocabularyId, question, correctAnswer, userAnswer, isCorrect }` | Tạo QuizResult, increment session counters |
| Hết câu hỏi | `POST /quizzes/{sid}/complete` | — | Mark session COMPLETED |
| After complete | `POST /activity/record` | `{ quizQuestionsAnswered: total }` | Invalidate streak, calendar, today, gamification |
| After complete | `POST /gamification/award-xp` | `{ amount: xpEarned, source: "QUIZ", referenceId: sessionId }` | Invalidate gamification, leaderboard |

---

## 6. Hệ thống tracking & gamification

### 6.1 DailyActivity (POST /activity/record)

```
RecordActivityDto (tất cả optional, incremental):
{
  flashcardsReviewed?: number,   ← Review session
  quizQuestionsAnswered?: number, ← Quiz completion
  newWordsAdded?: number,         ← Browse flashcards
  xpEarned?: number               ← deprecated từ FE
}

Backend behavior:
- Ngày đầu tiên: CREATE row mới
- Cùng ngày: INCREMENT counters (không replace)
- Unique constraint: (userId, date)
```

### 6.2 Gamification (POST /gamification/award-xp)

```
AwardXpDto:
{
  amount: number (≥ 1),
  source: "FLASHCARD" | "QUIZ" | "WORD_ADDED" | "WORD_MASTERED"
          | "STREAK" | "ACHIEVEMENT" | "DAILY_BONUS",
  referenceId?: string (uuid, optional)
}

Backend behavior:
- Tạo XPTransaction (immutable audit log)
- Cập nhật UserXP (totalXp, weeklyXp, monthlyXp)
- Tính level mới (thresholds: 0/100/300/600/1000/1500/2500/4000/6000/10000)
- Auto-check badge conditions → award nếu đủ điều kiện (+50 XP bonus)
```

### 6.3 XP Rate Plan

| Hành động | XP | Source | Khi nào |
|-----------|---:|--------|---------|
| Thêm từ vào list | 2 / từ | WORD_ADDED | Browse "Hoàn thành" |
| Ôn flashcard đúng (quality ≥ 3) | 3 / thẻ | FLASHCARD | Review summary |
| Quiz trả lời đúng | 5 / câu | QUIZ | Quiz complete |
| Badge bonus | 50 | ACHIEVEMENT | Auto (server) |

### 6.4 Streak

```
GET /activity/streak → { currentStreak, longestStreak, lastActivityDate }

Logic:
- Streak = số ngày liên tiếp có DailyActivity record
- Tính từ hôm nay/hôm qua ngược về
- Nếu lastActivity không phải hôm nay/hôm qua → streak = 0
```

---

## 7. Luồng dữ liệu tổng hợp

```
                   ┌──────────────┐
                   │  User Action  │
                   └──────┬───────┘
                          │
          ┌───────────────┼───────────────┐
          ▼               ▼               ▼
    ┌──────────┐   ┌──────────┐   ┌──────────┐
    │ Browse   │   │ Review   │   │  Quiz    │
    │ (add)    │   │ (SM-2)   │   │ (answer) │
    └────┬─────┘   └────┬─────┘   └────┬─────┘
         │              │              │
    ┌────▼─────┐   ┌────▼─────┐   ┌────▼─────┐
    │ POST     │   │ PATCH    │   │ POST     │
    │ /user-   │   │ /user-   │   │ /quizzes/│
    │ vocabs   │   │ vocabs/  │   │ answer   │
    │          │   │ {id}     │   │          │
    └────┬─────┘   └────┬─────┘   └────┬─────┘
         │              │              │
         │         (on summary)   (on complete)
         │              │              │
    ┌────▼──────────────▼──────────────▼─────┐
    │          POST /activity/record          │
    │  { newWordsAdded | flashcardsReviewed   │
    │    | quizQuestionsAnswered }             │
    └────────────────┬───────────────────────┘
                     │
    ┌────────────────▼───────────────────────┐
    │        POST /gamification/award-xp      │
    │  { amount, source, referenceId? }       │
    └────────────────┬───────────────────────┘
                     │
              ┌──────▼──────┐
              │  Dashboard   │
              │  updated via │
              │  query cache │
              │  invalidation│
              └─────────────┘
```

---

## 8. Cache invalidation map

```
Mutation                    → Invalidated Query Keys
────────────────────────────────────────────────────────
addToMyList                 → ["userVocabularyList"]
submitFlashcardReview       → ["dueFlashcards", "userVocabularyList"]
recordActivity              → ["streak", "activityCalendar", "activityToday", "myGamification"]
awardXp                     → ["myGamification", "leaderboard"]
completeQuiz                → (FE handles via recordActivity + awardXp)
enrollLearningPlan          → ["myEnrolledPlans"]
updateEnrollmentStatus      → ["myEnrolledPlans", "enrolledPlanDetail"]
unenrollLearningPlan        → ["myEnrolledPlans"]
```

---

## 9. Category Progress & "Tiếp tục học"

### 9.1 Hiện trạng

```
CategoryCard hiển thị progress dựa trên:
  learned = số từ user có trong list thuộc category
  total = wordCount của category

  learned > 0 → hiện progress bar + "X/Y đã học"
  learned == 0 → hiện "Chưa bắt đầu"

learn-vocab page quyết định state dựa trên:
  hasProgress = inCategory.length > 0

  hasProgress=false → hiện CategoryIntroView ("Bắt đầu học")
  hasProgress=true  → hiện BrowseFlashcards hoặc DueReviewSession
```

### 9.2 Giải pháp: localStorage tracking

```
Khi user nhấn "Bắt đầu học":
  → localStorage.setItem(`category-started-${categoryId}`, "true")

learn-vocab page:
  hasProgress = inCategory.length > 0
               || localStorage.getItem(`category-started-${categoryId}`) === "true"

CategoryCard:
  learned > 0   → progress bar (giữ nguyên)
  started=true  → "Tiếp tục học"
  default       → "Chưa bắt đầu"
```

---

## 10. Các fix cần thực hiện

### Frontend (over-the-world-fe-student)

| ID | File | Fix |
|----|------|-----|
| F1 | `BrowseFlashcards.tsx` | Thêm `addedCount` state, track đúng số từ đã thêm |
| F2 | `BrowseFlashcards.tsx` | `recordActivity({ newWordsAdded: addedCount })` thay vì `vocabs.length` |
| F3 | `BrowseFlashcards.tsx` | Thêm `awardXp({ amount: addedCount*2, source: "WORD_ADDED" })` |
| F4 | `DueReviewSession.tsx` | Thêm `awardXp({ amount: correctCount*3, source: "FLASHCARD" })` khi summary |
| F5 | `review/page.tsx` | Tương tự F4 |
| F6 | `practice/[sessionId]/page.tsx` | Bỏ `xpEarned` khỏi `recordActivity()` (2 chỗ) |
| F7 | `learn-vocab/[categoryId]/page.tsx` | Track "category started" localStorage |
| F8 | `CategoryCard.tsx` | Thêm `started` prop, hiện "Tiếp tục học" |
| F9 | `dashboard/page.tsx` | Tính startedCategories từ localStorage, pass prop |

### Backend (over-the-world-be)

| ID | File | Fix |
|----|------|-----|
| B1 | `quiz/quiz.service.ts` | submitAnswer: cập nhật UserVocabulary `"new"→"learning"` khi đúng |

---

## 11. Future considerations

- **Offline support**: Queue recordActivity/awardXp calls khi offline, retry khi online
- **Server-side activity recording**: Quiz complete endpoint tự gọi recordActivity + awardXp server-side (giảm dependency vào FE)
- **Vocabulary mastery từ quiz**: Nếu user trả lời đúng từ X liên tục 3 lần → "mastered"
- **Category completion badge**: Khi đã thêm 100% từ trong category → badge "Hoàn thành HSK1"
- **Learning plan progress**: Link quiz/review activity vào progress entity
- **Anti-cheat**: Validate quiz answers server-side thay vì trust client
- **Session duration tracking**: Record thời gian học mỗi session
- **Per-category activity**: Ghi categoryId vào DailyActivity để phân tích chi tiết
