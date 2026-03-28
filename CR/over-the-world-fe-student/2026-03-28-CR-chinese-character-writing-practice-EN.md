# CR: Chinese Character Writing Practice (Luyện viết chữ Hán)

**Date:** 2026-03-28  
**Module:** Writing Practice  
**Type:** Change Request — New Feature  
**Scope:** Backend (NestJS) + Frontend (over-the-world-fe-student)  
**Library:** [HanziWriter](https://hanziwriter.org/) (MIT License, ~30KB gzipped)  

---

## 1. Overview

Add a new **Chinese Character Writing Practice** module that allows students to practice writing Hán characters stroke-by-stroke. The feature uses the open-source **HanziWriter** library for animated stroke guides and freehand writing recognition, with stroke data loaded from CDN (jsdelivr). A separate SM-2 spaced repetition schedule tracks writing mastery independently from flashcard (reading) mastery.

### Why Separate from Flashcard Review?

Reading recognition and handwriting are distinct skills. A student may recognize a character in flashcards (mastered) but cannot reproduce it from memory. The `WritingProgress` entity maintains its own SM-2 fields so writing review intervals are independent of reading review intervals.

---

## 2. User Stories

| # | As a... | I want to... | So that... |
|---|---------|-------------|------------|
| US-01 | Student | See animated stroke order for any character | I learn the correct stroke sequence |
| US-02 | Student | Practice writing characters with guided stroke hints | I receive real-time feedback on each stroke |
| US-03 | Student | Freely practice writing any character on a canvas | I can drill characters without guidance |
| US-04 | Student | Review characters due for writing practice (SR) | I reinforce writing retention using spaced repetition |
| US-05 | Student | See my writing accuracy score after each attempt | I know how well I performed |
| US-06 | Student | View overall writing progress and statistics | I can track my improvement over time |
| US-07 | Student | Practice writing characters from a specific lesson | I can align writing practice with lesson content |
| US-08 | Student | Access writing practice from the main navigation | I can easily find and start practicing |

---

## 3. Acceptance Criteria

| # | Criterion |
|---|-----------|
| 1 | A **"Luyện viết"** link appears in the main navigation header for authenticated users |
| 2 | The `/writing` page displays 4 mode cards: Stroke Order, Free Practice, Lesson Writing, Writing Review |
| 3 | **Stroke Order:** User selects a character → animated stroke-by-stroke demonstration plays → user can replay or try writing |
| 4 | **Free Practice:** User enters any Chinese character → draws on canvas → gets accuracy score (0–100%) |
| 5 | **Lesson Writing:** User selects a lesson → practices writing each vocabulary character from that lesson |
| 6 | **Writing Review (SR):** Shows characters where `nextWritingReviewDate <= now` → user writes each → SM-2 updates |
| 7 | After each writing attempt, the system displays: accuracy %, correct strokes / total strokes, visual feedback |
| 8 | Writing progress is persisted to the backend via `POST /writing/submit` |
| 9 | The writing review queue is independent of the flashcard review queue |
| 10 | Stroke data is loaded from jsdelivr CDN (no local stroke data files bundled) |
| 11 | The canvas is touch-friendly for mobile/tablet use |
| 12 | A writing stats widget is added to the dashboard showing: characters practiced, due for review count |

---

## 4. Screens

### 4.1 Writing Main Page (`/writing`)

- 4 mode cards laid out in a responsive grid (2×2 on desktop, 1-column on mobile)
- Each card: icon, title, description, CTA button
- Cards: **Hướng dẫn nét bút** (Stroke Order), **Luyện viết tự do** (Free Practice), **Luyện viết theo bài** (Lesson Writing), **Ôn tập viết** (Writing Review)
- Writing Review card shows due count badge (e.g., "5 ký tự cần ôn")

### 4.2 Stroke Order Page (`/writing/stroke-order`)

- Character search input at top
- Character selector: browse from user's vocabulary list or type any character
- HanziWriter animation panel (large, centered):
  - **Animate** button: plays full stroke order animation
  - **Quiz** button: switches to guided writing mode (user draws each stroke)
- Character info sidebar: pinyin, meaning, stroke count, radical

### 4.3 Free Practice Page (`/writing/free-practice`)

- Character input field (user types the character they want to practice)
- Large HanziWriter canvas in quiz mode (no outline hints — blank canvas)
- After completion: accuracy score, stroke breakdown, retry button
- Option: toggle "Show outline" for guided mode vs. blank canvas

### 4.4 Writing Review Page (`/writing/review`)

- Queue of characters due for writing review (`nextWritingReviewDate <= now`)
- Shows: character prompt (pinyin + meaning, character hidden)
- Student writes on HanziWriter canvas
- After submission:
  - Accuracy score displayed
  - SM-2 quality rating mapped from accuracy: 95%→5, 80%→4, 60%→3, 40%→2, 20%→1, <20%→0
  - Next review date calculated and persisted
  - Auto-advance to next character
- Session summary at end: total reviewed, average accuracy, next scheduled date
- Empty state: "Bạn đã ôn xong! 🎉" with next scheduled date

### 4.5 Lesson Writing Page (`/writing/lesson/[lessonId]`)

- User selects a lesson from lesson list
- Displays vocabulary characters from that lesson in sequence
- Each character: HanziWriter guided quiz mode
- Progress bar showing current position in lesson
- Summary at end: characters practiced, average accuracy

---

## 5. Backend — New Entity

### 5.1 `WritingProgress` Entity

| Column | Type | Default | Notes |
|--------|------|---------|-------|
| `id` | uuid (PK) | auto-generated | Primary key |
| `userId` | uuid (FK → User) | — | `ON DELETE CASCADE` |
| `vocabularyId` | uuid (FK → Vocabulary) | — | `ON DELETE CASCADE` |
| `character` | varchar(10) | — | The specific character being practiced |
| `bestScore` | float | 0 | Best accuracy score (0.0 – 1.0) |
| `attemptCount` | int | 0 | Total number of writing attempts |
| `lastPracticed` | timestamp | null | Last practice timestamp |
| `status` | varchar(20) | 'new' | 'new' \| 'learning' \| 'mastered' |
| `writingEaseFactor` | float | 2.5 | SM-2 ease factor for writing |
| `writingInterval` | int | 0 | SM-2 interval (days) |
| `writingRepetitions` | int | 0 | SM-2 repetition count |
| `nextWritingReviewDate` | timestamp | null | Next scheduled writing review |
| `createdAt` | timestamp | auto | Record creation |
| `updatedAt` | timestamp | auto | Last update |

**Constraints:**
- Unique: `(userId, vocabularyId)` — one writing progress record per user per vocabulary
- Foreign keys: `userId → User.id`, `vocabularyId → Vocabulary.id`

---

## 6. Backend — API Endpoints

### 6.1 `GET /writing/due`

Returns characters due for writing review for the authenticated user.

**Auth:** `JwtAuthGuard`  
**Query Params:** `limit` (optional, default 20)

**Response (200):**
```json
{
  "data": [
    {
      "id": "uuid",
      "vocabularyId": "uuid",
      "character": "你",
      "pinyin": "nǐ",
      "meaning": "Xin chào / You",
      "bestScore": 0.75,
      "attemptCount": 3,
      "nextWritingReviewDate": "2026-03-28T00:00:00Z"
    }
  ],
  "total": 5
}
```

### 6.2 `POST /writing/submit`

Submits a writing attempt and updates SM-2 fields.

**Auth:** `JwtAuthGuard`  
**Request Body:**
```json
{
  "vocabularyId": "uuid",
  "character": "你",
  "accuracy": 0.85,
  "totalStrokes": 7,
  "correctStrokes": 6,
  "mistakes": 1,
  "timeSpentMs": 12000
}
```

**Response (201):**
```json
{
  "id": "uuid",
  "bestScore": 0.85,
  "attemptCount": 4,
  "status": "learning",
  "nextWritingReviewDate": "2026-03-30T00:00:00Z",
  "writingEaseFactor": 2.6,
  "writingInterval": 2,
  "writingRepetitions": 1,
  "qualityRating": 4
}
```

**SM-2 Quality Mapping:**
| Accuracy | Quality |
|----------|---------|
| ≥ 95% | 5 |
| ≥ 80% | 4 |
| ≥ 60% | 3 |
| ≥ 40% | 2 |
| ≥ 20% | 1 |
| < 20% | 0 |

### 6.3 `GET /writing/progress`

Returns aggregate writing statistics for the authenticated user.

**Auth:** `JwtAuthGuard`

**Response (200):**
```json
{
  "totalCharacters": 45,
  "masteredCount": 12,
  "learningCount": 28,
  "newCount": 5,
  "dueForReviewCount": 8,
  "averageAccuracy": 0.72,
  "totalAttempts": 156
}
```

### 6.4 `GET /writing/progress/:vocabularyId`

Returns writing progress for a specific vocabulary item.

**Auth:** `JwtAuthGuard`

**Response (200):**
```json
{
  "id": "uuid",
  "vocabularyId": "uuid",
  "character": "你",
  "bestScore": 0.85,
  "attemptCount": 4,
  "lastPracticed": "2026-03-28T10:30:00Z",
  "status": "learning",
  "writingEaseFactor": 2.6,
  "writingInterval": 2,
  "writingRepetitions": 1,
  "nextWritingReviewDate": "2026-03-30T00:00:00Z"
}
```

**Response (404):** If no progress exists for this vocabulary item → returns empty with defaults.

### 6.5 `GET /writing/history`

Returns paginated writing attempt history.

**Auth:** `JwtAuthGuard`  
**Query Params:** `page` (default 1), `pageSizes` (default 20)

**Response (200):**
```json
{
  "data": [
    {
      "id": "uuid",
      "character": "你",
      "bestScore": 0.85,
      "attemptCount": 4,
      "status": "learning",
      "lastPracticed": "2026-03-28T10:30:00Z"
    }
  ],
  "total": 45,
  "page": 1,
  "pageSizes": 20
}
```

---

## 7. Frontend — New Files

### 7.1 Types (in `src/types/index.ts`)

```typescript
export interface WritingProgress {
  id: string;
  vocabularyId: string;
  character: string;
  bestScore: number;
  attemptCount: number;
  lastPracticed?: string;
  status: VocabularyStatus;
  writingEaseFactor: number;
  writingInterval: number;
  writingRepetitions: number;
  nextWritingReviewDate?: string;
}

export interface WritingDueItem {
  id: string;
  vocabularyId: string;
  character: string;
  pinyin: string;
  meaning: string;
  bestScore: number;
  attemptCount: number;
  nextWritingReviewDate?: string;
}

export interface WritingSubmitPayload {
  vocabularyId: string;
  character: string;
  accuracy: number;
  totalStrokes: number;
  correctStrokes: number;
  mistakes: number;
  timeSpentMs: number;
}

export interface WritingSubmitResponse {
  id: string;
  bestScore: number;
  attemptCount: number;
  status: VocabularyStatus;
  nextWritingReviewDate?: string;
  writingEaseFactor: number;
  writingInterval: number;
  writingRepetitions: number;
  qualityRating: number;
}

export interface WritingStats {
  totalCharacters: number;
  masteredCount: number;
  learningCount: number;
  newCount: number;
  dueForReviewCount: number;
  averageAccuracy: number;
  totalAttempts: number;
}
```

### 7.2 New Files List

| # | File | Description |
|---|------|-------------|
| 1 | `src/types/index.ts` (update) | Add `WritingProgress`, `WritingDueItem`, `WritingSubmitPayload`, `WritingSubmitResponse`, `WritingStats` |
| 2 | `src/apiEndpoints/index.ts` (update) | Add 5 writing endpoints |
| 3 | `src/services/writing/useGetWritingDueService.ts` | Query hook — fetch due characters |
| 4 | `src/services/writing/useGetWritingProgressService.ts` | Query hook — fetch aggregate stats |
| 5 | `src/services/writing/useGetWritingHistoryService.ts` | Query hook — fetch paginated history |
| 6 | `src/services/writing/useSubmitWritingService.ts` | Mutation hook — submit writing attempt |
| 7 | `src/components/writing/HanziCanvas.tsx` | HanziWriter wrapper — animation, quiz, freehand modes |
| 8 | `src/components/writing/StrokeResult.tsx` | Accuracy display after writing attempt |
| 9 | `src/components/writing/CharacterInfo.tsx` | Sidebar: pinyin, meaning, stroke count |
| 10 | `src/components/writing/CharacterQueue.tsx` | Queue indicator for review/lesson sessions |
| 11 | `src/routes/writing/index.tsx` | Main writing page — 4 mode cards |
| 12 | `src/routes/writing/stroke-order.tsx` | Stroke order demonstration page |
| 13 | `src/routes/writing/free-practice.tsx` | Free practice canvas page |
| 14 | `src/routes/writing/review.tsx` | Writing SR review page |
| 15 | `src/routes/writing/lesson.$lessonId.tsx` | Lesson-based writing page |
| 16 | `src/components/Header.tsx` (update) | Add "Luyện viết" nav link |
| 17 | `src/routes/index.tsx` (update) | Add writing stats dashboard widget |

---

## 8. HanziWriter Integration

### 8.1 Library Details

- **NPM:** `hanzi-writer` (MIT license)
- **Size:** ~30KB gzipped (library only)
- **Stroke Data:** Loaded per-character from jsdelivr CDN (~2–8KB per character)
- **CDN URL:** `https://cdn.jsdelivr.net/npm/hanzi-writer-data@2.0/{character}.json`
- **Cache:** jsdelivr sets 1-year cache headers; browser caches automatically

### 8.2 Usage Modes

| Mode | HanziWriter Method | Use Case |
|------|-------------------|----------|
| **Animate** | `writer.animateCharacter()` | Stroke order demonstration |
| **Quiz** | `writer.quiz({ ... })` | Guided writing with stroke hints |
| **Freehand** | Quiz mode with `showHintAfterMisses: false` | Free practice without hints |

### 8.3 HanziCanvas Component API

```typescript
interface HanziCanvasProps {
  character: string;
  mode: 'animate' | 'quiz' | 'freehand';
  width?: number;           // default: 300
  height?: number;          // default: 300
  showOutline?: boolean;    // default: true
  onComplete?: (result: { totalMistakes: number; totalStrokes: number; }) => void;
  onCorrectStroke?: () => void;
  onMistake?: () => void;
}
```

---

## 9. SM-2 Algorithm for Writing

The same SM-2 algorithm used for flashcard review is applied to writing practice, with the quality rating derived from writing accuracy:

```
accuracy = correctStrokes / totalStrokes
quality = mapAccuracyToQuality(accuracy)

if quality >= 3:
  if repetitions == 0: interval = 1
  elif repetitions == 1: interval = 6
  else: interval = round(interval * easeFactor)
  repetitions += 1
else:
  repetitions = 0
  interval = 1

easeFactor = max(1.3, easeFactor + (0.1 - (5 - quality) * (0.08 + (5 - quality) * 0.02)))
nextWritingReviewDate = now + interval days
```

**Status Transitions:**
- `new` → `learning`: after first writing attempt
- `learning` → `mastered`: when `writingRepetitions >= 5` AND `bestScore >= 0.8`
- `mastered` → `learning`: if quality < 3 on a review (regression)

---

## 10. Implementation Phases

| Phase | Scope | Est. |
|-------|-------|------|
| **Phase 1** | BE: `WritingProgress` entity + migration + 5 API endpoints | 4h |
| **Phase 2** | FE: `HanziCanvas` component + Stroke Order page | 4h |
| **Phase 3** | FE: Free Practice page + Writing Review (SR) page | 4h |
| **Phase 4** | FE: Lesson Writing page + Main Writing page + Navigation | 3h |
| **Phase 5** | FE: Dashboard widget + Polish + Build verification | 2h |

**Total Estimate:** ~17h

---

## 11. Security & Permissions

| Concern | Mitigation |
|---------|-----------|
| **Auth** | All endpoints require `JwtAuthGuard` — no public access |
| **Ownership** | Service filters by `userId` from JWT token — users can only access their own progress |
| **Input Validation** | DTOs use `class-validator`: `@IsUUID()`, `@IsNumber()`, `@Min(0)`, `@Max(1)` for accuracy |
| **CDN Data** | Stroke data from jsdelivr is read-only JSON; no user input is sent to CDN |
| **Rate Limiting** | Standard API rate limits apply to `POST /writing/submit` |

---

## 12. Non-Functional Requirements

| Requirement | Target |
|-------------|--------|
| HanziWriter canvas renders at 60fps | Tested on mid-range mobile |
| Stroke data loads in < 500ms (CDN cached after first load) | jsdelivr edge caching |
| Writing submit API responds in < 200ms | Simple upsert + SM-2 calculation |
| Mobile touch support | HanziWriter has built-in touch/pointer events |
| Responsive canvas | Scales to container width, min 250px, max 400px |
| Offline awareness | Show "offline" banner if CDN unreachable; disable submit |

---

## 13. Dependencies

| Dependency | Version | License | Purpose |
|------------|---------|---------|---------|
| `hanzi-writer` | ^3.5 | MIT | Stroke animation + writing quiz engine |
| jsdelivr CDN | — | Public | Stroke data JSON files (~9000 characters) |

**No other new dependencies required.** The backend uses existing TypeORM, class-validator, NestJS patterns. The frontend uses existing TanStack Query, Tailwind, shadcn/ui.

---

## 14. Out of Scope

- Gamification XP integration for writing (may be added in a future CR)
- Audio pronunciation during writing practice
- AI-powered handwriting recognition (uses HanziWriter's built-in stroke matching)
- Writing practice for non-CJK characters
- Offline writing practice (requires CDN for stroke data)
- Multi-character compound word writing (single character per attempt)
