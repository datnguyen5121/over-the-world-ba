# Báo cáo Phân tích Tác động Đa hệ thống
## CR: Luyện viết chữ Hán — Đánh giá ảnh hưởng đến `over-the-world-be` và `over-the-world-fe`

**Ngày:** 28/03/2026  
**Người lập:** BA Team  
**CR Gốc:** [2026-03-28-CR-writing-practice-EN.md](./2026-03-28-CR-chinese-character-writing-practice-EN.md)  
**Phạm vi phân tích:** `over-the-world-be` (NestJS Backend) + `over-the-world-fe` (Admin Dashboard)  
**Trạng thái:** Draft — Chờ xác nhận quyết định phạm vi (xem Mục 4)

---

## 1. Tóm tắt điều hành

CR "Luyện viết chữ Hán" được thiết kế chủ yếu cho ứng dụng học viên (`over-the-world-fe-student`). Báo cáo này đánh giá mức độ ảnh hưởng lan ra hai hệ thống còn lại:

| Hệ thống | Có ảnh hưởng? | Mức độ | Ghi chú |
|----------|--------------|--------|---------|
| `over-the-world-be` (Backend) | ✅ **Có — Bắt buộc** | Cao | Phải tạo mới entity, service, controller, migration |
| `over-the-world-fe` (Admin Dashboard) | ⚠️ **Có — Tùy chọn** | Thấp | Không có thay đổi bắt buộc trong phạm vi CR này |

**Kết luận nhanh:** Backend cần thay đổi đáng kể nhưng đã được tính vào phạm vi CR gốc. Admin Dashboard **không cần thay đổi** để tính năng hoạt động, nhưng có 3 cơ hội cải tiến đáng cân nhắc cho các CR tương lai.

---

## 2. Phân tích ảnh hưởng — `over-the-world-be`

### 2.1 Các thay đổi BẮT BUỘC (P0 — Feature không hoạt động nếu thiếu)

| # | File | Loại thay đổi | Mô tả |
|---|------|--------------|--------|
| 1 | `src/writing/entities/writing-progress.entity.ts` | **TẠO MỚI** | Entity `WritingProgress` với SM-2 fields độc lập |
| 2 | `src/writing/dto/submit-writing.dto.ts` | **TẠO MỚI** | DTO xác thực input `POST /writing/submit` |
| 3 | `src/writing/writing.service.ts` | **TẠO MỚI** | Service xử lý 5 endpoints + tính toán SM-2 |
| 4 | `src/writing/writing.controller.ts` | **TẠO MỚI** | Controller định nghĩa 5 routes (JWT guard) |
| 5 | `src/writing/writing.module.ts` | **TẠO MỚI** | Module wiring toàn bộ writing feature |
| 6 | `src/migrations/1775000000000-AddWritingProgressTable.ts` | **TẠO MỚI** | Migration tạo bảng `writing_progress` trong PostgreSQL |
| 7 | `src/app.module.ts` | **SỬA ĐỔI** | Thêm `WritingModule` vào `imports[]` |
| 8 | `src/db/data-source.ts` | **SỬA ĐỔI** | Thêm `WritingProgress` vào `entities[]` (cần cho TypeORM CLI migration) |

> **Nhận xét BA:** Tất cả 8 thay đổi trên đã được đề cập trong CR gốc. Không có surprise. Đây là pattern chuẩn của dự án (entity → DTO → service → controller → module → migration → đăng ký).

---

### 2.2 Các thay đổi KHÔNG CẦN (Đã phân tích — BA xác nhận giữ nguyên)

| File | Lý do không thay đổi |
|------|---------------------|
| `src/users/entities/user.entity.ts` | TypeORM cho phép quan hệ một chiều. `WritingProgress` có `@ManyToOne(() => User)` là đủ. Không cần `@OneToMany(() => WritingProgress)` ngược lại trong `User` — tránh phình entity. |
| `src/notification/notification.service.ts` | Hệ thống notification hiện tại chỉ nhận trigger từ admin (broadcast), không có cơ chế event-driven tự động. Luyện viết không kích hoạt notification. |
| `src/audit-log/` | Audit log là **opt-in** qua decorator `@AuditLogAction`. GET requests không bao giờ được log. `POST /writing/submit` có thể thêm decorator về sau nếu cần, nhưng không bắt buộc trong Sprint đầu. |

---

### 2.3 Quyết định phạm vi CẦN XÁC NHẬN — Backend deferred items

Dưới đây là 2 vấn đề **không bắt buộc** nhưng ảnh hưởng đến chất lượng dữ liệu nếu bỏ qua. Cần BA/Tech Lead xác nhận có đưa vào CR này không hay để CR tiếp theo.

---

#### ⚠️ Quyết định 1: Luyện viết có tính vào hoạt động hàng ngày (`DailyActivity`) không?

**Hiện trạng:** Entity `DailyActivity` hiện có các cột:
- `flashcardsReviewed` — số flashcard đã ôn
- `quizQuestionsAnswered` — số câu hỏi quiz
- `newWordsAdded` — số từ mới thêm
- `xpEarned` — XP kiếm trong ngày

**Vấn đề:** Nếu học viên luyện viết 10 chữ trong ngày, `DailyActivity` **không ghi nhận** hoạt động này. Hệ quả:
- Streak calendar không phản ánh ngày học viên chỉ luyện viết (không ôn flashcard, không làm quiz)
- Dashboard "Today's Activity" thiếu thống kê luyện viết
- Báo cáo hoạt động của admin không đầy đủ

**Giải pháp nếu muốn tích hợp:**
1. Thêm cột `writingPracticesCompleted INT DEFAULT 0` vào bảng `daily_activity` — migration `1776000000000-AddWritingColumnToDailyActivity.ts`
2. Sửa `DailyActivityService` để increment counter khi `POST /writing/submit` thành công
3. Cập nhật WritingService để gọi DailyActivityService sau mỗi lần submit

**Tác động nếu bỏ qua:** Luyện viết "vô hình" với hệ thống streak và daily tracking hiện tại.

| Lựa chọn | Mô tả | Ưu | Nhược |
|----------|-------|-----|-------|
| **A — Tích hợp ngay** | Thêm cột + cập nhật service trong CR này | Streak chính xác ngay từ đầu | Tăng scope, thêm migration |
| **B — Defer sang CR tiếp theo** | Ra mắt tính năng trước, tích hợp DailyActivity sau | Giảm scope Sprint 1 | Streak sẽ không đếm ngày luyện viết cho đến khi thêm |

> **Đề xuất BA:** Chọn **Phương án A** nếu sprint có đủ thời gian. Dữ liệu streak bị thiếu từ đầu khó reconcile về sau.

---

#### ⚠️ Quyết định 2: Có ghi audit log cho `POST /writing/submit` không?

**Hiện trạng:** Audit log ghi lại các hành động quản trị (tạo/sửa/xóa từ vựng, tài khoản, v.v.). Không có cơ chế bắt buộc — hoàn toàn opt-in.

**Vấn đề:** `POST /writing/submit` là action học viên thực hiện nhiều lần mỗi ngày (hàng chục lần). Nếu ghi audit log:
- **Ưu:** Traceability đầy đủ — có thể điều tra nếu có gian lận điểm số
- **Nhược:** Bảng `audit_log` tăng trưởng rất nhanh (mỗi học viên × N lần/ngày) → hiệu năng query audit log bị ảnh hưởng về lâu dài

**Đề xuất BA:** **Không thêm audit log cho writing submit** trong giai đoạn đầu. Đây là student-generated activity data, không phải admin/teacher action. Nếu cần anti-cheat, nên thiết kế cơ chế riêng (rate limiting, server-side validation) thay vì dùng audit log. ChỈ enable nếu có yêu cầu compliance cụ thể.

---

## 3. Phân tích ảnh hưởng — `over-the-world-fe` (Admin Dashboard)

### 3.1 Kiểm tra hiện trạng

Admin Dashboard hiện tại (phân tích thực tế từ codebase) chỉ hiển thị:

| Thống kê | Nguồn dữ liệu | Có liên quan đến Writing? |
|---------|--------------|--------------------------|
| Tổng số người dùng | `users` table | Không |
| Tổng từ vựng | `vocabulary` table | Không |
| Tổng danh mục | `category` table | Không |
| Tổng kế hoạch học | `learning_plan` table | Không |
| Người dùng theo vai trò | `users.role` | Không |
| Đăng ký mới theo ngày | `users.createdAt` | Không |
| 5 từ vựng gần nhất | `vocabulary.createdAt` | Không |
| 5 người dùng gần nhất | `users.createdAt` | Không |

**Kết luận:** Admin Dashboard hiện tại **hoàn toàn không bị ảnh hưởng** — nó chỉ hiển thị nội dung và tài khoản, không liên quan đến hoạt động học của sinh viên.

---

### 3.2 Trang chi tiết tài khoản (`/admin/account-management/:id`)

Trang này hiện chỉ hiển thị: username, email, role, createdAt, updatedAt — cùng các hành động Edit / Delete / Reset Password.

**Không có** bất kỳ thống kê học tập nào (quiz history, flashcard mastery, writing progress). Việc thêm writing stats vào đây sẽ là một tính năng mới gắn với CR khác, không phải yêu cầu của CR này.

---

### 3.3 Sidebar điều hướng Admin

```
Dashboard | Account Mgmt | Category Mgmt | Word Mgmt | 
Learning Plan Mgmt | Audit Logs | Storage Mgmt | Notifications
```

Không có mục nào liên quan đến "Tiến độ học viên" hay "Báo cáo luyện viết". **Không cần thay đổi** để CR này hoạt động.

---

### 3.4 Kết luận Admin Frontend

**Không có thay đổi bắt buộc nào** trong `over-the-world-fe` cho CR này. Admin không cần xem, quản lý hay xuất dữ liệu luyện viết để tính năng hoạt động ở phía học viên.

---

## 4. Các cơ hội mở rộng trong tương lai (không phải scope của CR này)

Sau khi tính năng luyện viết đi vào hoạt động và có dữ liệu thực, BA **khuyến nghị** xem xét 3 CR sau trong các sprint tiếp theo:

### CR Tương lai #1: Dashboard Admin — Thống kê luyện viết
**Mô tả:** Thêm 1 tile/card thống kê trên trang Admin Dashboard:
- Tổng số ký tự đã được học viên luyện viết
- Số học viên đang sử dụng tính năng luyện viết
- Tỷ lệ chính xác trung bình toàn hệ thống

**Tác động backend:** Cần mở rộng `AdminStatsService` để query bảng `writing_progress`. Low complexity.  
**Tác động admin FE:** Thêm 1 card component trên trang dashboard. Low complexity.

---

### CR Tương lai #2: Quản lý tiến độ học viên — Admin xem per-user
**Mô tả:** Trên trang chi tiết tài khoản (`/admin/account-management/:id`), thêm tab "Tiến độ học tập" hiển thị:
- Số flashcard đã ôn / tổng
- Điểm quiz trung bình
- Số chữ đã luyện viết / tỷ lệ thành thạo
- Streak học tập

**Tác động backend:** Cần endpoint mới `GET /admin/users/:id/progress` tổng hợp từ nhiều bảng. Medium complexity.  
**Tác động admin FE:** Tab mới trong account detail page. Multiple query hooks. Medium complexity.

---

### CR Tương lai #3: Xuất báo cáo luyện viết (Export)
**Mô tả:** Cho phép admin xuất CSV/Excel toàn bộ writing progress của tất cả học viên, phục vụ báo cáo định kỳ.

**Tác động backend:** Mở rộng `AdminExportModule` (`src/admin/export/`) thêm endpoint `GET /admin/export/writing-progress`. Low-medium complexity.  
**Tác động admin FE:** Thêm nút Export trên trang mới hoặc trang dashboard. Low complexity.

---

## 5. Ma trận quyết định tổng hợp

### 5.1 Backend — Phân loại thay đổi

| Thay đổi | Bắt buộc? | Thuộc CR này? | Thuộc CR nào? |
|---------|----------|--------------|--------------|
| Tạo `WritingProgress` entity + module | ✅ Bắt buộc | ✅ CR này | — |
| Migration `writing_progress` table | ✅ Bắt buộc | ✅ CR này | — |
| Đăng ký trong `app.module.ts` + `data-source.ts` | ✅ Bắt buộc | ✅ CR này | — |
| Thêm `writingPracticesCompleted` vào `DailyActivity` | ⚠️ Cần quyết định | ⁉️ Cân nhắc | CR này hoặc CR tiếp theo |
| Audit log cho writing submit | ❌ Không cần | ❌ Không đưa vào | N/A |
| Bidirectional relation trong `User` entity | ❌ Không cần | ❌ Không đưa vào | N/A |
| Writing stats trong `AdminStatsService` | ✅ Nên có | ❌ CR tiếp theo | CR Tương lai #1 |
| Per-user writing progress endpoint cho admin | ✅ Nên có | ❌ CR tiếp theo | CR Tương lai #2 |

### 5.2 Admin Frontend — Phân loại thay đổi

| Thay đổi | Bắt buộc? | Thuộc CR này? | Thuộc CR nào? |
|---------|----------|--------------|--------------|
| Sidebar nav: thêm mục "Writing Progress" | ❌ Không cần | ❌ | CR Tương lai #2 |
| Dashboard: Thêm writing stats card | ❌ Không cần | ❌ | CR Tương lai #1 |
| Account detail: Tab tiến độ luyện viết | ❌ Không cần | ❌ | CR Tương lai #2 |
| API endpoints cho admin writing data | ❌ Không cần | ❌ | CR Tương lai #1/#2 |
| Export writing progress CSV | ❌ Không cần | ❌ | CR Tương lai #3 |

---

## 6. Checklist thay đổi thực tế cho CR này

### ✅ Backend (`over-the-world-be`) — phải làm

```
[ ] Tạo src/writing/entities/writing-progress.entity.ts
[ ] Tạo src/writing/dto/submit-writing.dto.ts
[ ] Tạo src/writing/writing.service.ts
[ ] Tạo src/writing/writing.controller.ts
[ ] Tạo src/writing/writing.module.ts
[ ] Tạo src/migrations/1775000000000-AddWritingProgressTable.ts
[ ] Sửa src/app.module.ts → thêm WritingModule vào imports[]
[ ] Sửa src/db/data-source.ts → thêm WritingProgress vào entities[]
[ ] (Quyết định #1) Nếu đưa DailyActivity vào: thêm cột + migration 1776000000000
[ ] npm run build → verify 0 TypeScript errors
[ ] npm run migration:run → verify table created
```

### ✅ Admin Frontend (`over-the-world-fe`) — không cần thay đổi

```
[—] Không có thay đổi bắt buộc
[—] Không có file cần tạo hoặc sửa
[—] Bảo lưu tất cả thay đổi admin FE sang CR tương lai (xem Mục 4)
```

### ✅ Student Frontend (`over-the-world-fe-student`) — theo CR gốc

```
[Xem CR gốc: 2026-03-28-CR-chinese-character-writing-practice-VI.md]
```

---

## 7. Rủi ro và phụ thuộc

| Rủi ro | Mức độ | Biện pháp |
|--------|--------|-----------|
| **DailyActivity không được tích hợp** → streak của học viên luyện viết không được ghi nhận | Trung bình | Quyết định sớm ở Quyết định #1 (Mục 2.3) |
| **Migration timestamp trùng lặp** | Thấp | Đã xác nhận timestamp kế tiếp là `1775000000000` (cao hơn migration cuối `1774000000000-AddLessonAndGamification`) |
| **HanziWriter CDN không truy cập được** | Thấp | Đã có kế hoạch offline banner ở CR gốc |
| **Admin không theo dõi được writing abuse** | Thấp | Không có gamification XP trong CR này → rủi ro farming thấp. Rate limiting ở API level là đủ |

---

## 8. Phụ lục: Sơ đồ mô-đun sau khi triển khai CR

```
over-the-world-be/src/
├── app.module.ts                 ← THÊM: WritingModule
├── db/data-source.ts             ← THÊM: WritingProgress entity
├── migrations/
│   ├── 1774000000000-...         ← Cuối cùng hiện tại
│   └── 1775000000000-AddWritingProgressTable.ts  ← MỚI
├── writing/                      ← THƯ MỤC MỚI HOÀN TOÀN
│   ├── entities/
│   │   └── writing-progress.entity.ts
│   ├── dto/
│   │   └── submit-writing.dto.ts
│   ├── writing.service.ts
│   ├── writing.controller.ts
│   └── writing.module.ts
│
│   [Không đổi vì CR này:]
├── admin/stats/                  ← KHÔNG THAY ĐỔI
├── audit-log/                    ← KHÔNG THAY ĐỔI
├── users/entities/user.entity.ts ← KHÔNG THAY ĐỔI
├── notification/                 ← KHÔNG THAY ĐỔI
└── daily-activity/               ← KHÔNG ĐỔI (trừ khi quyết định #1 = A)
```

---

*Báo cáo này được lập bởi BA Team dựa trên phân tích thực tế mã nguồn của 3 repository tính đến ngày 28/03/2026.*
