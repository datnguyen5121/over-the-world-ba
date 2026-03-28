# CR: Luyện viết chữ Hán (Chinese Character Writing Practice)

**Ngày:** 28/03/2026  
**Module:** Luyện viết  
**Loại:** Yêu cầu thay đổi — Tính năng mới  
**Phạm vi:** Backend (NestJS) + Frontend (over-the-world-fe-student)  
**Thư viện:** [HanziWriter](https://hanziwriter.org/) (Giấy phép MIT, ~30KB gzipped)  

---

## 1. Tổng quan

Thêm module **Luyện viết chữ Hán** mới cho phép học viên luyện viết từng nét chữ Hán. Tính năng sử dụng thư viện mã nguồn mở **HanziWriter** để hiển thị hướng dẫn nét bút động và nhận diện nét viết tay, với dữ liệu nét bút được tải từ CDN (jsdelivr). Một lịch trình SM-2 (lặp lại ngắt quãng) riêng biệt theo dõi mức độ thành thạo viết, độc lập với mức độ thành thạo đọc (flashcard).

### Tại sao tách biệt với Ôn tập Flashcard?

Khả năng nhận diện chữ (đọc) và khả năng viết tay là hai kỹ năng khác nhau. Một học viên có thể nhận ra chữ trong flashcard (đã thành thạo đọc) nhưng không thể viết lại từ trí nhớ. Entity `WritingProgress` duy trì các trường SM-2 riêng để khoảng cách ôn tập viết không phụ thuộc vào khoảng cách ôn tập đọc.

---

## 2. User Stories (Câu chuyện người dùng)

| # | Với tư cách... | Tôi muốn... | Để... |
|---|----------------|-------------|-------|
| US-01 | Học viên | Xem hướng dẫn thứ tự nét bút động cho bất kỳ chữ nào | Tôi học được trình tự nét bút đúng |
| US-02 | Học viên | Luyện viết chữ với gợi ý nét bút hướng dẫn | Tôi nhận được phản hồi theo thời gian thực cho từng nét |
| US-03 | Học viên | Luyện viết tự do bất kỳ chữ nào trên canvas | Tôi có thể rèn luyện chữ mà không cần hướng dẫn |
| US-04 | Học viên | Ôn tập các chữ đến hạn luyện viết (SR) | Tôi củng cố khả năng ghi nhớ viết bằng lặp lại ngắt quãng |
| US-05 | Học viên | Xem điểm chính xác sau mỗi lần viết | Tôi biết mình viết tốt đến đâu |
| US-06 | Học viên | Xem tiến độ và thống kê luyện viết tổng thể | Tôi có thể theo dõi sự tiến bộ của mình |
| US-07 | Học viên | Luyện viết các chữ từ một bài học cụ thể | Tôi có thể gắn luyện viết với nội dung bài học |
| US-08 | Học viên | Truy cập luyện viết từ thanh điều hướng chính | Tôi có thể dễ dàng tìm và bắt đầu luyện tập |

---

## 3. Tiêu chí chấp nhận

| # | Tiêu chí |
|---|----------|
| 1 | Liên kết **"Luyện viết"** xuất hiện trong thanh điều hướng header cho người dùng đã đăng nhập |
| 2 | Trang `/writing` hiển thị 4 thẻ chế độ: Hướng dẫn nét bút, Luyện viết tự do, Luyện viết theo bài, Ôn tập viết |
| 3 | **Hướng dẫn nét bút:** Người dùng chọn chữ → hoạt ảnh nét bút từng nét phát → có thể phát lại hoặc thử viết |
| 4 | **Luyện viết tự do:** Người dùng nhập chữ Hán → vẽ trên canvas → nhận điểm chính xác (0–100%) |
| 5 | **Luyện viết theo bài:** Người dùng chọn bài học → luyện viết từng chữ từ vựng trong bài |
| 6 | **Ôn tập viết (SR):** Hiển thị các chữ có `nextWritingReviewDate <= now` → người dùng viết → cập nhật SM-2 |
| 7 | Sau mỗi lần viết, hệ thống hiển thị: % chính xác, nét đúng / tổng nét, phản hồi trực quan |
| 8 | Tiến độ viết được lưu vào backend qua `POST /writing/submit` |
| 9 | Hàng đợi ôn tập viết độc lập với hàng đợi ôn tập flashcard |
| 10 | Dữ liệu nét bút tải từ CDN jsdelivr (không đóng gói file dữ liệu nét bút cục bộ) |
| 11 | Canvas hỗ trợ cảm ứng trên điện thoại/máy tính bảng |
| 12 | Widget thống kê viết được thêm vào dashboard: số chữ đã luyện, số chữ cần ôn tập |

---

## 4. Các màn hình

### 4.1 Trang chính Luyện viết (`/writing`)

- 4 thẻ chế độ bố cục dạng lưới responsive (2×2 trên máy tính, 1 cột trên di động)
- Mỗi thẻ: biểu tượng, tiêu đề, mô tả, nút CTA
- Các thẻ: **Hướng dẫn nét bút**, **Luyện viết tự do**, **Luyện viết theo bài**, **Ôn tập viết**
- Thẻ Ôn tập viết hiển thị badge số lượng cần ôn (ví dụ: "5 ký tự cần ôn")

### 4.2 Trang Hướng dẫn nét bút (`/writing/stroke-order`)

- Ô nhập tìm kiếm chữ phía trên
- Bộ chọn chữ: duyệt từ danh sách từ vựng của người dùng hoặc nhập bất kỳ chữ nào
- Bảng hoạt ảnh HanziWriter (lớn, căn giữa):
  - Nút **Xem hoạt ảnh**: phát hoạt ảnh thứ tự nét bút đầy đủ
  - Nút **Thử viết**: chuyển sang chế độ viết có hướng dẫn (người dùng vẽ từng nét)
- Thanh bên thông tin chữ: phiên âm, nghĩa, số nét, bộ thủ

### 4.3 Trang Luyện viết tự do (`/writing/free-practice`)

- Ô nhập chữ (người dùng gõ chữ muốn luyện)
- Canvas HanziWriter lớn ở chế độ quiz (không gợi ý viền — canvas trống)
- Sau khi hoàn thành: điểm chính xác, phân tích nét, nút thử lại
- Tùy chọn: bật/tắt "Hiện viền" cho chế độ có hướng dẫn so với canvas trống

### 4.4 Trang Ôn tập viết (`/writing/review`)

- Hàng đợi các chữ đến hạn ôn tập viết (`nextWritingReviewDate <= now`)
- Hiển thị: gợi ý chữ (phiên âm + nghĩa, chữ bị ẩn)
- Học viên viết trên canvas HanziWriter
- Sau khi gửi:
  - Hiển thị điểm chính xác
  - Xếp hạng chất lượng SM-2 ánh xạ từ độ chính xác: 95%→5, 80%→4, 60%→3, 40%→2, 20%→1, <20%→0
  - Tính toán và lưu ngày ôn tập tiếp theo
  - Tự động chuyển sang chữ tiếp theo
- Tóm tắt phiên ở cuối: tổng số đã ôn, độ chính xác trung bình, ngày ôn tiếp theo
- Trạng thái trống: "Bạn đã ôn xong! 🎉" kèm ngày ôn tập tiếp theo

### 4.5 Trang Luyện viết theo bài (`/writing/lesson/[lessonId]`)

- Người dùng chọn bài học từ danh sách bài
- Hiển thị các chữ từ vựng trong bài theo trình tự
- Mỗi chữ: chế độ quiz HanziWriter có hướng dẫn
- Thanh tiến trình hiển thị vị trí hiện tại trong bài
- Tóm tắt cuối: số chữ đã luyện, độ chính xác trung bình

---

## 5. Backend — Entity mới

### 5.1 Entity `WritingProgress`

| Cột | Kiểu | Mặc định | Ghi chú |
|-----|------|----------|---------|
| `id` | uuid (PK) | tự tạo | Khóa chính |
| `userId` | uuid (FK → User) | — | `ON DELETE CASCADE` |
| `vocabularyId` | uuid (FK → Vocabulary) | — | `ON DELETE CASCADE` |
| `character` | varchar(10) | — | Chữ cụ thể đang luyện viết |
| `bestScore` | float | 0 | Điểm chính xác cao nhất (0.0 – 1.0) |
| `attemptCount` | int | 0 | Tổng số lần viết |
| `lastPracticed` | timestamp | null | Thời điểm luyện lần cuối |
| `status` | varchar(20) | 'new' | 'new' \| 'learning' \| 'mastered' |
| `writingEaseFactor` | float | 2.5 | Hệ số dễ SM-2 cho viết |
| `writingInterval` | int | 0 | Khoảng cách SM-2 (ngày) |
| `writingRepetitions` | int | 0 | Số lần lặp SM-2 |
| `nextWritingReviewDate` | timestamp | null | Ngày ôn tập viết tiếp theo |
| `createdAt` | timestamp | tự động | Ngày tạo bản ghi |
| `updatedAt` | timestamp | tự động | Ngày cập nhật cuối |

**Ràng buộc:**
- Duy nhất: `(userId, vocabularyId)` — mỗi người dùng chỉ có một bản ghi tiến độ viết cho mỗi từ vựng
- Khóa ngoại: `userId → User.id`, `vocabularyId → Vocabulary.id`

---

## 6. Backend — API Endpoints

### 6.1 `GET /writing/due`

Trả về các chữ đến hạn ôn tập viết cho người dùng đã xác thực.

**Xác thực:** `JwtAuthGuard`  
**Tham số Query:** `limit` (tùy chọn, mặc định 20)

**Phản hồi (200):**
```json
{
  "data": [
    {
      "id": "uuid",
      "vocabularyId": "uuid",
      "character": "你",
      "pinyin": "nǐ",
      "meaning": "Bạn / You",
      "bestScore": 0.75,
      "attemptCount": 3,
      "nextWritingReviewDate": "2026-03-28T00:00:00Z"
    }
  ],
  "total": 5
}
```

### 6.2 `POST /writing/submit`

Gửi kết quả luyện viết và cập nhật các trường SM-2.

**Xác thực:** `JwtAuthGuard`  
**Body yêu cầu:**
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

**Phản hồi (201):**
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

**Ánh xạ Chất lượng SM-2:**
| Độ chính xác | Chất lượng |
|-------------|-----------|
| ≥ 95% | 5 |
| ≥ 80% | 4 |
| ≥ 60% | 3 |
| ≥ 40% | 2 |
| ≥ 20% | 1 |
| < 20% | 0 |

### 6.3 `GET /writing/progress`

Trả về thống kê tổng hợp luyện viết cho người dùng đã xác thực.

**Xác thực:** `JwtAuthGuard`

**Phản hồi (200):**
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

Trả về tiến độ viết cho một từ vựng cụ thể.

**Xác thực:** `JwtAuthGuard`

**Phản hồi (200):**
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

**Phản hồi (404):** Nếu chưa có tiến độ cho từ vựng này → trả về rỗng với giá trị mặc định.

### 6.5 `GET /writing/history`

Trả về lịch sử luyện viết có phân trang.

**Xác thực:** `JwtAuthGuard`  
**Tham số Query:** `page` (mặc định 1), `pageSizes` (mặc định 20)

**Phản hồi (200):**
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

## 7. Frontend — Các file mới

### 7.1 Kiểu dữ liệu (trong `src/types/index.ts`)

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

### 7.2 Danh sách file mới

| # | File | Mô tả |
|---|------|--------|
| 1 | `src/types/index.ts` (cập nhật) | Thêm `WritingProgress`, `WritingDueItem`, `WritingSubmitPayload`, `WritingSubmitResponse`, `WritingStats` |
| 2 | `src/apiEndpoints/index.ts` (cập nhật) | Thêm 5 endpoint viết |
| 3 | `src/services/writing/useGetWritingDueService.ts` | Hook query — lấy danh sách chữ cần ôn |
| 4 | `src/services/writing/useGetWritingProgressService.ts` | Hook query — lấy thống kê tổng hợp |
| 5 | `src/services/writing/useGetWritingHistoryService.ts` | Hook query — lấy lịch sử có phân trang |
| 6 | `src/services/writing/useSubmitWritingService.ts` | Hook mutation — gửi kết quả viết |
| 7 | `src/components/writing/HanziCanvas.tsx` | Wrapper HanziWriter — chế độ hoạt ảnh, quiz, tự do |
| 8 | `src/components/writing/StrokeResult.tsx` | Hiển thị kết quả chính xác sau lần viết |
| 9 | `src/components/writing/CharacterInfo.tsx` | Thanh bên: phiên âm, nghĩa, số nét |
| 10 | `src/components/writing/CharacterQueue.tsx` | Chỉ báo hàng đợi cho phiên ôn tập/bài học |
| 11 | `src/routes/writing/index.tsx` | Trang chính luyện viết — 4 thẻ chế độ |
| 12 | `src/routes/writing/stroke-order.tsx` | Trang hướng dẫn thứ tự nét bút |
| 13 | `src/routes/writing/free-practice.tsx` | Trang luyện viết tự do |
| 14 | `src/routes/writing/review.tsx` | Trang ôn tập viết SR |
| 15 | `src/routes/writing/lesson.$lessonId.tsx` | Trang luyện viết theo bài |
| 16 | `src/components/Header.tsx` (cập nhật) | Thêm liên kết "Luyện viết" trong nav |
| 17 | `src/routes/index.tsx` (cập nhật) | Thêm widget thống kê viết trên dashboard |

---

## 8. Tích hợp HanziWriter

### 8.1 Thông tin thư viện

- **NPM:** `hanzi-writer` (giấy phép MIT)
- **Kích thước:** ~30KB gzipped (chỉ thư viện)
- **Dữ liệu nét bút:** Tải theo từng chữ từ CDN jsdelivr (~2–8KB mỗi chữ)
- **URL CDN:** `https://cdn.jsdelivr.net/npm/hanzi-writer-data@2.0/{character}.json`
- **Cache:** jsdelivr đặt cache header 1 năm; trình duyệt tự động cache

### 8.2 Các chế độ sử dụng

| Chế độ | Phương thức HanziWriter | Trường hợp sử dụng |
|--------|------------------------|---------------------|
| **Hoạt ảnh** | `writer.animateCharacter()` | Minh họa thứ tự nét bút |
| **Quiz** | `writer.quiz({ ... })` | Viết có hướng dẫn gợi ý nét |
| **Tự do** | Quiz mode với `showHintAfterMisses: false` | Luyện tự do không gợi ý |

### 8.3 API Component HanziCanvas

```typescript
interface HanziCanvasProps {
  character: string;
  mode: 'animate' | 'quiz' | 'freehand';
  width?: number;           // mặc định: 300
  height?: number;          // mặc định: 300
  showOutline?: boolean;    // mặc định: true
  onComplete?: (result: { totalMistakes: number; totalStrokes: number; }) => void;
  onCorrectStroke?: () => void;
  onMistake?: () => void;
}
```

---

## 9. Thuật toán SM-2 cho Luyện viết

Cùng thuật toán SM-2 được sử dụng cho ôn tập flashcard, áp dụng cho luyện viết, với xếp hạng chất lượng được tính từ độ chính xác viết:

```
accuracy = correctStrokes / totalStrokes (nét đúng / tổng nét)
quality = mapAccuracyToQuality(accuracy) (ánh xạ thành chất lượng)

nếu quality >= 3:
  nếu repetitions == 0: interval = 1
  nếu repetitions == 1: interval = 6
  ngược lại: interval = round(interval * easeFactor)
  repetitions += 1
ngược lại:
  repetitions = 0
  interval = 1

easeFactor = max(1.3, easeFactor + (0.1 - (5 - quality) * (0.08 + (5 - quality) * 0.02)))
nextWritingReviewDate = now + interval ngày
```

**Chuyển đổi trạng thái:**
- `new` → `learning`: sau lần viết đầu tiên
- `learning` → `mastered`: khi `writingRepetitions >= 5` VÀ `bestScore >= 0.8`
- `mastered` → `learning`: nếu quality < 3 trong lần ôn tập (thoái lui)

---

## 10. Các giai đoạn triển khai

| Giai đoạn | Phạm vi | Ước lượng |
|-----------|---------|-----------|
| **Giai đoạn 1** | BE: Entity `WritingProgress` + migration + 5 API endpoints | 4h |
| **Giai đoạn 2** | FE: Component `HanziCanvas` + trang Hướng dẫn nét bút | 4h |
| **Giai đoạn 3** | FE: Trang Luyện viết tự do + trang Ôn tập viết (SR) | 4h |
| **Giai đoạn 4** | FE: Trang Luyện viết theo bài + Trang chính + Điều hướng | 3h |
| **Giai đoạn 5** | FE: Widget dashboard + Hoàn thiện + Kiểm tra build | 2h |

**Tổng ước lượng:** ~17h

---

## 11. Bảo mật & Phân quyền

| Vấn đề | Biện pháp |
|--------|-----------|
| **Xác thực** | Tất cả endpoint yêu cầu `JwtAuthGuard` — không truy cập công khai |
| **Quyền sở hữu** | Service lọc theo `userId` từ JWT token — người dùng chỉ truy cập tiến độ của mình |
| **Xác thực đầu vào** | DTO sử dụng `class-validator`: `@IsUUID()`, `@IsNumber()`, `@Min(0)`, `@Max(1)` cho accuracy |
| **Dữ liệu CDN** | Dữ liệu nét bút từ jsdelivr là JSON chỉ đọc; không gửi dữ liệu người dùng đến CDN |
| **Giới hạn tần suất** | Giới hạn tần suất API tiêu chuẩn áp dụng cho `POST /writing/submit` |

---

## 12. Yêu cầu phi chức năng

| Yêu cầu | Mục tiêu |
|----------|----------|
| Canvas HanziWriter render ở 60fps | Kiểm tra trên thiết bị di động trung bình |
| Dữ liệu nét bút tải trong < 500ms (CDN cache sau lần đầu) | Edge caching của jsdelivr |
| API gửi kết quả viết phản hồi trong < 200ms | Upsert đơn giản + tính toán SM-2 |
| Hỗ trợ cảm ứng di động | HanziWriter tích hợp sẵn sự kiện touch/pointer |
| Canvas responsive | Co giãn theo chiều rộng container, tối thiểu 250px, tối đa 400px |
| Nhận biết offline | Hiện banner "ngoại tuyến" nếu không kết nối được CDN; vô hiệu hóa gửi |

---

## 13. Phụ thuộc

| Phụ thuộc | Phiên bản | Giấy phép | Mục đích |
|-----------|-----------|-----------|----------|
| `hanzi-writer` | ^3.5 | MIT | Engine hoạt ảnh nét bút + quiz viết |
| jsdelivr CDN | — | Công khai | File JSON dữ liệu nét bút (~9000 chữ) |

**Không cần thêm phụ thuộc mới nào khác.** Backend sử dụng các pattern TypeORM, class-validator, NestJS hiện có. Frontend sử dụng TanStack Query, Tailwind, shadcn/ui hiện có.

---

## 14. Ngoài phạm vi

- Tích hợp XP gamification cho luyện viết (có thể thêm trong CR tương lai)
- Phát âm khi luyện viết
- Nhận diện chữ viết tay bằng AI (sử dụng nhận diện nét tích hợp của HanziWriter)
- Luyện viết cho ký tự ngoài CJK
- Luyện viết offline (cần CDN cho dữ liệu nét bút)
- Viết từ ghép nhiều chữ (một chữ mỗi lần)
