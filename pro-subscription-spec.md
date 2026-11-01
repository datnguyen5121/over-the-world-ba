# Đặc tả chức năng: Đăng ký Pro (Pro Subscription)

**Ngày tạo:** 2026-05-20  
**Phiên bản:** 1.1  
**Tác giả:** Business Analyst  
**Trạng thái:** 🔄 v1.1 — Updated after senior BA review  
**Dự án liên quan:** `over-the-world-be` · `over-the-world-fe-student` · `over-the-world-fe` (Admin)

---

## 1. Tổng quan & Bối cảnh kinh doanh

### 1.1 Vấn đề cần giải quyết

Hiện tại toàn bộ tính năng của nền tảng đều miễn phí. Mô hình này không bền vững về chi phí vận hành khi người dùng tăng trưởng, đặc biệt các chi phí:

- **Azure Cognitive Services / Google TTS** — phí per-character khi phát âm
- **Gemini AI** — phí per-token cho tính năng AI Explain từ vựng
- **Neon PostgreSQL** (production) — giới hạn connection pool, storage
- **Vercel / server hosting** — bandwidth & serverless function invocations

### 1.2 Mục tiêu

| # | Mục tiêu | Đo lường |
|---|---|---|
| 1 | Tạo nguồn doanh thu bền vững | MRR (Monthly Recurring Revenue) |
| 2 | Giữ trải nghiệm cốt lõi miễn phí để không chặn acquisition | ≥ 70% tính năng cốt lõi khả dụng ở FREE tier |
| 3 | Gating các tính năng AI/premium có chi phí vận hành cao | Chi phí AI/user/tháng ≤ giá Pro |
| 4 | Cung cấp công cụ quản lý subscription cho Admin | Admin không cần can thiệp thủ công |

### 1.3 Mô hình Freemium

```
FREE ──────────────────────────────► PRO
 │                                    │
 │  Học từ vựng cơ bản (HSK1-6)       │  Tất cả tính năng FREE
 │  Flashcard SM-2                    │  + AI Giải thích từ vựng (không giới hạn)
 │  Quiz cơ bản                       │  + Đánh giá phát âm (Azure Speech)
 │  Luyện viết                        │  + Export từ vựng CSV
 │  Streak & Gamification             │  + Badge "PRO" đặc biệt
 │  Bảng xếp hạng                     │  + Ưu tiên hỗ trợ (priority support)
 │  Đọc bài viết (blog)               │
 │  AI Explain: 5 lần/ngày (giới hạn) │
```

### 1.4 Giá đề xuất

| Gói | Giá | Chu kỳ | Ghi chú |
|---|---|---|---|
| **PRO Tháng** | 49,000 VND / tháng | Hàng tháng | Hủy bất cứ lúc nào |
| **PRO Năm** | 399,000 VND / năm | Hàng năm | Tiết kiệm ~32% so với tháng |

> **Quyết định mở:** Stripe (quốc tế) vs VNPay/MoMo (nội địa Việt Nam) — xem §8 để phân tích.

---

## 2. Stakeholders & Actors

| Actor | Mô tả | Liên quan |
|---|---|---|
| **Student (FREE)** | Người dùng đang dùng free tier | Chuyển đổi lên PRO |
| **Student (PRO)** | Người dùng đã nâng cấp | Truy cập đầy đủ tính năng |
| **Admin** | Quản trị viên hệ thống | Quản lý subscriptions, xem revenue |
| **Payment Gateway** | VNPay / MoMo / Stripe | Xử lý thanh toán, gửi webhook |
| **System (Cron)** | Background task | Xử lý expired subscriptions |

---

## 3. User Stories

### 3.1 Student — Nâng cấp Pro

| ID | Story | Priority | Acceptance Criteria |
|---|---|---|---|
| US-01 | Tôi muốn xem so sánh FREE vs PRO để hiểu tôi được gì khi upgrade | Must | Trang `/pricing` hoặc modal hiện bảng so sánh đầy đủ |
| US-02 | Tôi muốn mua PRO bằng ví MoMo/VNPay vì tôi không có thẻ quốc tế | Must | Có ít nhất 1 phương thức thanh toán nội địa |
| US-03 | Tôi muốn trạng thái PRO hiển thị ngay sau khi thanh toán thành công | Must | Sau webhook confirm → UI cập nhật trong ≤ 30 giây |
| US-04 | Tôi muốn nhận email xác nhận đơn hàng | Should | Email gửi trong ≤ 5 phút sau thanh toán |
| US-05 | Tôi muốn biết PRO của tôi hết hạn khi nào | Must | Dashboard / Settings hiện ngày hết hạn |
| US-06 | Tôi muốn gia hạn PRO trước khi hết hạn | Should | Nút "Gia hạn" xuất hiện 7 ngày trước khi hết hạn |
| US-07 | Tôi muốn hủy đăng ký PRO | Must | Hủy subscription, không bị charge chu kỳ tiếp theo; vẫn dùng PRO đến hết hạn hiện tại |

### 3.2 Student — Trải nghiệm tính năng Pro

| ID | Story | Priority | Acceptance Criteria |
|---|---|---|---|
| US-08 | Tôi muốn dùng AI Explain không giới hạn khi đã PRO | Must | Không có quota limit; FREE bị chặn sau 5 lần/ngày |
| US-09 | Tôi muốn đánh giá phát âm của tôi so với chuẩn | **Should** | Sau khi record → nhận điểm accuracy và feedback chi tiết |
| US-10 | Tôi muốn export danh sách từ vựng đã học ra CSV | Should | File CSV chứa: từ, pinyin, nghĩa, trạng thái học |
| US-11 | Tôi muốn thấy badge PRO trên profile và leaderboard | Could | Avatar hiện badge "⭐ PRO" |
| US-12 | Khi tôi (FREE) cố dùng tính năng PRO, tôi muốn được hướng dẫn upgrade | Must | Hiện paywall modal với CTA, không crash hay lỗi im lặng |

### 3.3 Admin — Quản lý subscription

| ID | Story | Priority | Acceptance Criteria |
|---|---|---|---|
| US-13 | Tôi muốn xem danh sách tất cả subscriptions (active, expired, cancelled) | Must | Bảng có filter status, search user, sort theo ngày |
| US-14 | Tôi muốn tặng PRO miễn phí cho một user (gifted subscription) | Should | Admin grant PRO thủ công với ngày hết hạn tùy chỉnh |
| US-15 | Tôi muốn xem tổng revenue theo tháng/năm | Should | Dashboard revenue với chart |
| US-16 | Tôi muốn cancel subscription của một user vi phạm | Must | Admin force-cancel, user về FREE ngay lập tức |
| US-17 | Tôi muốn chỉnh giá, quota AI và feature flags của gói PRO mà không cần deploy lại | Should | Trang settings lưu config vào DB, áp dụng ngay cho mọi request tiếp theo |

---

## 4. Business Rules

| ID | Rule |
|---|---|
| BR-01 | User hủy subscription vẫn giữ PRO đến cuối chu kỳ đã thanh toán (`status = CANCELLED` nhưng `expiresAt` giữ nguyên) |
| BR-02 | Subscription hết hạn → tự động downgrade về FREE bằng cron job chạy mỗi giờ |
| BR-03 | AI Explain quota reset lúc 00:00 **Asia/Ho_Chi_Minh** hàng ngày — cột `date` trong `ai_explain_usage` phải lưu theo timezone này, không phải UTC. Server cần convert `new Date()` sang UTC+7 trước khi lấy `date` component |
| BR-04 | FREE user AI Explain quota: 5 lần/ngày (đọc từ `subscription_config`); PRO user: không giới hạn |
| BR-05 | Một user chỉ có 1 subscription `ACTIVE` hoặc `CANCELLED` (còn hiệu lực) tại một thời điểm. Nếu user đang có subscription `CANCELLED` chưa hết hạn và muốn checkout mới → chặn, thông báo: *"Gói hiện tại còn đến [ngày]. Bạn có thể gia hạn sau khi hết hạn."* |
| BR-06 | Thanh toán thành công → ghi `payment.status = PAID` trước khi activate subscription (tránh race condition với webhook duplicate) |
| BR-07 | Refund không được xử lý tự động trong v1 — Admin xử lý thủ công qua cổng gateway |
| BR-08 | Gifted subscription (Admin tặng) không tạo payment record; `subscription.source = GIFT` |
| BR-09 | Khi gia hạn **trước khi hết hạn**: `expiresAt` mới = `expiresAt` cũ + duration. Khi gia hạn **sau khi đã hết hạn** (gap): `expiresAt` mới = `now()` + duration |
| BR-10 | Subscription config (giá, quota, feature flags) đọc từ bảng `subscription_config` tại runtime — thay đổi config có hiệu lực ngay mà không cần redeploy |
| BR-11 | `SubscriptionGuard` phải double-check `user.subscriptionTier === 'PRO'` **và** `user.proExpiresAt > now()` để phòng trường hợp cron fail chưa kịp expire |

---

## 5. Database Schema

### 5.1 Thay đổi bảng `user` (over-the-world-be)

Thêm 2 cột vào `user.entity.ts`:

```typescript
// Thêm vào User entity
@Column({
  type: 'enum',
  enum: ['FREE', 'PRO'],
  default: 'FREE',
})
subscriptionTier: 'FREE' | 'PRO';

@Column({ type: 'timestamp', nullable: true })
proExpiresAt: Date | null;
```

> **Lý do denormalize:** Query guard nhanh hơn — không cần JOIN subscription table mỗi request. Subscription table là source of truth, cột này là cache được sync qua cron/webhook.

### 5.2 Bảng `subscription` (mới)

| Cột | Kiểu | Ghi chú |
|---|---|---|
| `id` | UUID (PK) | |
| `userId` | UUID (FK → user) | Index |
| `plan` | enum: `MONTHLY`, `YEARLY` | |
| `status` | enum: `ACTIVE`, `CANCELLED`, `EXPIRED` | Lifecycle state — GIFTED không phải trạng thái, xem `source` |
| `source` | enum: `PAYMENT`, `GIFT` | Nguồn gốc: thanh toán hay Admin tặng |
| `startedAt` | timestamp | |
| `expiresAt` | timestamp | |
| `cancelledAt` | timestamp nullable | Khi user hủy |
| `grantedByAdminId` | UUID nullable (FK → user) | Nếu admin tặng |
| `paymentId` | UUID nullable (FK → payment) | Null nếu gifted |
| `createdAt` | timestamp | |
| `updatedAt` | timestamp | |

### 5.3 Bảng `payment` (mới)

| Cột | Kiểu | Ghi chú |
|---|---|---|
| `id` | UUID (PK) | |
| `userId` | UUID (FK → user) | |
| `subscriptionId` | UUID nullable (FK → subscription) | Set sau khi tạo subscription |
| `gateway` | enum: `MOMO`, `VNPAY`, `STRIPE` | |
| `gatewayOrderId` | varchar(255) UNIQUE | Order ID từ gateway |
| `gatewayTransactionId` | varchar(255) nullable | Transaction ID sau khi confirm |
| `amount` | int | VND (hoặc cents nếu Stripe) |
| `currency` | varchar(10) | `VND` hoặc `USD` |
| `plan` | enum: `MONTHLY`, `YEARLY` | |
| `status` | enum: `PENDING`, `PAID`, `FAILED`, `REFUNDED` | |
| `webhookPayload` | jsonb nullable | Raw payload từ gateway (audit) |
| `failedReason` | varchar(500) nullable | Lý do thất bại từ gateway — dùng cho support |
| `paidAt` | timestamp nullable | |
| `createdAt` | timestamp | |

### 5.4 Bảng `ai_explain_usage` (mới — quota tracking)

| Cột | Kiểu | Ghi chú |
|---|---|---|
| `id` | UUID (PK) | |
| `userId` | UUID (FK → user) | |
| `date` | date | Ngày dùng — **phải convert sang Asia/Ho_Chi_Minh trước khi lưu** (xem BR-03) |
| `count` | int | Số lần đã dùng hôm nay |
| PK option | `(userId, date)` composite | Có thể dùng thay UUID để tối ưu index |
| Unique constraint | `(userId, date)` | Đảm bảo 1 row/user/ngày |

### 5.5 Bảng `subscription_config` (mới — runtime configuration)

| Cột | Kiểu | Ghi chú |
|---|---|---|
| `key` | varchar(100) (PK) | Tên config — unique |
| `value` | text | Giá trị (số nguyên, boolean, hoặc JSON string) |
| `description` | varchar(255) nullable | Mô tả nội bộ cho Admin |
| `updatedAt` | timestamp | |
| `updatedByAdminId` | UUID nullable (FK → user) | Admin nào thay đổi lần cuối |

**Config keys khởi tạo (seed data):**

| Key | Default | Mô tả |
|---|---|---|
| `price_monthly_vnd` | `49000` | Giá gói tháng (VND) |
| `price_yearly_vnd` | `399000` | Giá gói năm (VND) |
| `ai_explain_free_daily_quota` | `5` | Số lần AI Explain FREE/ngày |
| `feature_pronunciation_enabled` | `true` | Bật/tắt tính năng đánh giá phát âm |
| `feature_csv_export_enabled` | `true` | Bật/tắt export CSV |
| `pricing_page_bullets` | `[...]` JSON | Bullet points hiển thị trên trang `/pricing` |

---

## 6. Backend API (`over-the-world-be`)

### 6.1 Module mới: `subscription`

**File structure:**
```
src/subscription/
  subscription.module.ts
  subscription.service.ts
  subscription.controller.ts
  subscription.cron.ts          ← Expire cron job
  dto/
    create-checkout.dto.ts
    admin-grant.dto.ts
    query-subscription.dto.ts
    update-config.dto.ts
  entities/
    subscription.entity.ts
    payment.entity.ts
    subscription-config.entity.ts
    ai-explain-usage.entity.ts
```

### 6.2 Endpoints — Student

| Method | Path | Auth | Mô tả |
|---|---|---|---|
| `POST` | `/subscription/checkout` | JWT (Student) | Tạo checkout session / payment URL |
| `GET` | `/subscription/my` | JWT (Student) | Xem subscription hiện tại của mình |
| `POST` | `/subscription/cancel` | JWT (Student) | Hủy auto-renew (CANCELLED nhưng active đến hết hạn) |
| `GET` | `/subscription/payments` | JWT (Student) | Lịch sử thanh toán của mình (paginated, max 20/page) |
| `POST` | `/subscription/webhook/momo` | Public (HMAC verify) | Nhận IPN từ MoMo |
| `POST` | `/subscription/webhook/vnpay` | Public (HMAC verify) | Nhận IPN từ VNPay |
| `POST` | `/subscription/webhook/stripe` | Public (Stripe-Signature) | Nhận webhook từ Stripe |

**`POST /subscription/checkout` — Request Body:**
```json
{
  "plan": "MONTHLY" | "YEARLY",
  "gateway": "MOMO" | "VNPAY" | "STRIPE",
  "redirectUrl": "https://app.tiengtrungminmin.com/settings?tab=subscription"
}
```

**Response:**
```json
{
  "paymentId": "uuid",
  "checkoutUrl": "https://payment.momo.vn/...",
  "expiresAt": "2026-05-20T10:15:00Z"
}
```

### 6.3 Endpoints — Admin

| Method | Path | Auth | Mô tả |
|---|---|---|---|
| `GET` | `/admin/subscription` | JWT (Admin) | Danh sách subscriptions (paginated, filterable) |
| `GET` | `/admin/subscription/revenue` | JWT (Admin) | Thống kê revenue tháng/năm |
| `POST` | `/admin/subscription/grant` | JWT (Admin) | Tặng PRO thủ công cho user |
| `DELETE` | `/admin/subscription/:id` | JWT (Admin) | Force-cancel subscription của user |
| `GET` | `/admin/subscription/settings` | JWT (Admin) | Lấy toàn bộ subscription config hiện tại |
| `PUT` | `/admin/subscription/settings` | JWT (Admin) | Cập nhật một hoặc nhiều config key; ghi audit log |

### 6.4 Module cập nhật: `vocabulary` (AI Explain quota)

Khi endpoint `GET /vocabulary/:id/explain` được gọi:

1. Lấy `subscriptionTier` của user từ JWT/DB
2. Nếu `FREE`: kiểm tra `ai_explain_usage` hôm nay
   - `count >= 5` → throw `403 ForbiddenException` với `code: "AI_QUOTA_EXCEEDED"`
   - Ngược lại: tăng count, tiếp tục gọi Gemini
3. Nếu `PRO`: bỏ qua quota check, gọi Gemini trực tiếp

### 6.5 Guard mới: `SubscriptionGuard`

```typescript
// Dùng decorator để gate endpoint theo tier
@UseGuards(SubscriptionGuard)
@RequireSubscription('PRO')   // custom decorator
@Post('/pronunciation/assess')
async assessPronunciation(...) {}
```

Guard inject `UsersService` để check **cả hai**: `user.subscriptionTier === 'PRO'` **và** `user.proExpiresAt > new Date()`. Bắt buộc check đồng thời để phòng trường hợp cron fail chưa kịp expire (BR-11).

### 6.6 Cron Job — Expire Subscriptions

Chạy mỗi giờ (hoặc mỗi 30 phút):

```typescript
@Cron('0 * * * *')
async expireSubscriptions() {
  // 1. Tìm subscriptions có expiresAt < now() và status IN ('ACTIVE', 'CANCELLED')
  //    → CANCELLED vẫn có thể còn trong hạn, cần expire khi đến ngày
  // 2. Set status = EXPIRED
  // 3. Set user.subscriptionTier = 'FREE', user.proExpiresAt = null
  // 4. Batch update để tránh N+1 queries
}
```

### 6.7 Module mới (hoặc mở rộng `tts`): `pronunciation`

> Hiện tại `tts.controller.ts` trống hoàn toàn → cần implement.

| Method | Path | Auth | Tier | Mô tả |
|---|---|---|---|---|
| `POST` | `/pronunciation/assess` | JWT | PRO only | Nhận audio blob + vocabularyId → trả về score |

**Request:** `multipart/form-data` — `audio: File`, `vocabularyId: string`  
**Response:**
```json
{
  "overallScore": 85,
  "accuracyScore": 88,
  "fluencyScore": 82,
  "toneScore": 79,
  "feedback": "Thanh điệu thứ 3 chưa đủ thấp. Thử lại...",
  "phonemes": [...]
}
```

---

## 7. Frontend — Student (`over-the-world-fe-student`)

### 7.1 Routes mới

| Route | Mô tả |
|---|---|
| `/pricing` | Trang so sánh FREE vs PRO, CTA mua |
| `/settings?tab=subscription` | Xem trạng thái subscription, lịch sử thanh toán |
| `/payment/success` | Redirect sau thanh toán thành công |
| `/payment/failed` | Redirect sau thanh toán thất bại |

### 7.2 Components mới

**`<ProBadge />`**
- Hiển thị badge "⭐ PRO" cạnh avatar khi `subscriptionTier === 'PRO'`
- Xuất hiện ở: Header avatar, Profile page, Leaderboard

**`<PaywallModal />`**
- Trigger khi FREE user cố dùng PRO feature
- Props: `feature: string` (để customize message)
- Nội dung: mô tả tính năng, lợi ích PRO, nút "Nâng cấp ngay"

**`<AiExplainButton />`** (cập nhật component hiện có nếu có)
- FREE: hiện quota badge `3/5 lần hôm nay`
- Khi hết quota: disable + show `<PaywallModal />`
- PRO: không hiện quota, hiện icon ✨

**`<PronunciationAssessor />`** (mới)
- Record audio từ microphone
- Gửi lên `/pronunciation/assess`
- Hiển thị score breakdown (Overall / Accuracy / Fluency / Tone)
- Gating: Chỉ available cho PRO

**`<SubscriptionCard />`** (trong Settings)
- Hiện tier hiện tại, ngày hết hạn
- Nút "Nâng cấp" (FREE) hoặc "Quản lý / Hủy" (PRO)
- Lịch sử thanh toán (dạng bảng nhỏ)

### 7.3 Cập nhật services

**`useGetMyProfileService`** — response cần thêm:
```typescript
interface MyProfile {
  // ... existing fields
  subscriptionTier: 'FREE' | 'PRO';
  proExpiresAt: string | null;
  aiExplainUsedToday: number; // từ ai_explain_usage
}
```

**`useSubscriptionService`** (mới):
- `useCreateCheckout(plan, gateway)` — POST checkout
- `useMySubscription()` — GET /subscription/my
- `useCancelSubscription()` — POST /subscription/cancel

### 7.4 Luồng thanh toán (FE perspective)

```
User click "Nâng cấp PRO"
  → Chọn gói (Tháng / Năm)
  → Chọn cổng thanh toán (MoMo / VNPay)
  → POST /subscription/checkout
  → Redirect đến checkoutUrl (MoMo/VNPay)
  → [User hoàn tất thanh toán trên cổng]
  → Redirect về /payment/success?orderId=...
  → FE query GET /subscription/my (polling 3s, max 5 lần)
  → Nếu active → hiện success state, refresh profile
  → Nếu timeout → hiện "Đang xử lý, vui lòng chờ" + link support
```

> **Note:** Không nên trust redirect params để xác nhận thanh toán thành công — chỉ trust data từ API `/subscription/my` sau khi webhook đã xử lý.

### 7.5 Cập nhật `useAuth` / `AuthContext`

Thêm `subscriptionTier` vào context để component toàn app có thể read mà không cần re-fetch:

```typescript
interface AuthContextType {
  // ... existing
  isPro: boolean;  // subscriptionTier === 'PRO' && proExpiresAt > now
}
```

---

## 8. Frontend — Admin (`over-the-world-fe`)

### 8.1 Routes mới trong Admin panel

| Route | Mô tả |
|---|---|
| `/admin/subscriptions` | Danh sách subscriptions (table + filter) |
| `/admin/subscriptions/revenue` | Dashboard revenue |
| `/admin/subscriptions/settings` | Cấu hình gói PRO: giá, quota, feature flags |

### 8.2 Features

**Subscription List Table:**
- Columns: User (avatar + name + email), Plan, Status, Start, Expires, Gateway, Amount, Actions
- Filter: Status (Active / Expired / Cancelled / Gifted), Plan, Date range
- Actions: View detail, Grant PRO, Force cancel

**Revenue Dashboard:**
- MRR, ARR (tính từ active subscriptions)
- Chart doanh thu theo tháng (6 tháng gần nhất)
- Breakdown theo gateway (MoMo / VNPay / Stripe)
- Số lượng Pro users active hiện tại

**Subscription Settings Page (`/admin/subscriptions/settings`):**
- **Pricing:** Input fields cho giá MONTHLY và YEARLY (VND); validate là số nguyên dương
- **AI Quota:** Input số lần AI Explain FREE/ngày; áp dụng ngay cho request tiếp theo
- **Feature Flags:** Toggle switches cho từng tính năng PRO (Pronunciation, CSV Export...)
- **Pricing Page Content:** Textarea JSON để chỉnh bullet points hiển thị trên `/pricing` của FE-Student
- Mỗi row config hiện: *"Last updated by [admin] at [time]"*
- Nút **"Preview /pricing"** mở tab mới để xem trước trang pricing

**Grant PRO Modal:**
```
User:     [search input — tìm theo email / username]
Duration: [1 tháng / 3 tháng / 6 tháng / 1 năm / custom date picker]
Reason:   [text — ghi chú nội bộ, không hiện với user]
          [Confirm Grant]
```

---

## 9. Payment Gateway Integration

### 9.1 So sánh cổng thanh toán

| Tiêu chí | MoMo | VNPay | Stripe |
|---|---|---|---|
| Target người dùng VN | ✅ Rất phổ biến | ✅ Phổ biến | ❌ Cần thẻ QT |
| Tích hợp kỹ thuật | Trung bình | Trung bình | Dễ (SDK tốt) |
| Phí giao dịch | ~1-2% | ~1-2% | 2.9% + $0.30 |
| Webhook độ tin cậy | Tốt | Tốt | Rất tốt |
| Sandbox | ✅ | ✅ | ✅ |
| Subscription native | ❌ (manual renew) | ❌ (manual renew) | ✅ (recurring) |

**Khuyến nghị v1:** Tích hợp **MoMo** là primary (thị trường VN), **Stripe** là secondary (user quốc tế). Subscription tự động gia hạn v1 dùng cơ chế tự tạo (cron + notify user gia hạn thủ công), thay vì dùng recurring API vì MoMo không support.

### 9.2 Webhook Security

Tất cả webhook endpoint **phải** verify signature trước khi xử lý:

```typescript
// MoMo: HMAC-SHA256 verify
// VNPay: SHA512 hmac verify  
// Stripe: stripe.webhooks.constructEvent(body, sig, secret)
```

Webhook payload raw (chưa parse) phải được lưu vào `payment.webhookPayload` để audit.

**Idempotency:** Webhook có thể gửi nhiều lần. Guard bằng `gatewayOrderId` unique constraint — nếu đã `PAID` thì skip, không activate 2 lần.

---

## 10. Non-Functional Requirements

| # | Yêu cầu | Metric |
|---|---|---|
| NFR-01 | Webhook xử lý trong vòng 5 giây | p95 < 5s |
| NFR-02 | Sau webhook PAID → user thấy PRO trong ≤ 30 giây | FE polling max 10 lần |
| NFR-03 | AI quota check không làm chậm response | Thêm < 20ms (in-memory cache với Redis nếu có, hoặc DB indexed query) |
| NFR-04 | Subscription expiry chạy đúng giờ | Cron job monitoring, alert nếu fail |
| NFR-05 | Không lộ payment details trong FE (chỉ hiện masked info) | Security audit |
| NFR-06 | HTTPS bắt buộc cho tất cả payment endpoints | TLS enforced |

---

## 11. Edge Cases & Xử lý lỗi

| Tình huống | Xử lý |
|---|---|
| Webhook arrive trước khi FE redirect về | FE polling sẽ bắt được sau vài giây |
| User đóng cửa sổ giữa chừng khi thanh toán | Payment `PENDING` → cron cleanup sau 1 giờ; user có thể tạo checkout mới |
| Người dùng trả tiền 2 lần (double submit) | `gatewayOrderId` unique → lần 2 bị block bởi idempotency guard |
| PRO expires trong khi đang dùng tính năng | Session không bị kick; lần request tiếp theo guard check lại |
| Cron job bị fail | Subscription vẫn hiện là ACTIVE cho đến khi cron chạy lại; không critical nhưng cần alert |
| Admin tặng PRO cho user đang PRO | Cộng dồn vào `expiresAt` hiện tại (theo BR-09) |
| Gemini API key hết quota | Trả lỗi thân thiện `"AI tạm thời không khả dụng"`, không expose lỗi kỹ thuật |
| User xóa tài khoản khi đang PRO | Subscription record giữ lại (soft reference), không refund tự động |

---

## 12. Acceptance Criteria — Tổng hợp

### Done Definition cho BE:

- [ ] Migration: thêm `subscriptionTier`, `proExpiresAt` vào `user`
- [ ] Tạo bảng `subscription`, `payment`, `ai_explain_usage`
- [ ] `SubscriptionGuard` hoạt động đúng với mọi tier
- [ ] `POST /subscription/checkout` trả về URL hợp lệ cho từng gateway
- [ ] Webhook endpoint verify HMAC trước khi xử lý
- [ ] Webhook idempotency: gọi 2 lần không activate 2 lần
- [ ] Cron job expire subscription chạy và update đúng
- [ ] AI Explain quota FREE = 5/ngày, PRO = unlimited
- [ ] Admin endpoints hoạt động: list, grant, force-cancel, revenue
- [ ] Unit test cho: quota logic, webhook handler, cron expire
- [ ] `GET/PUT /admin/subscription/settings` hoạt động; mỗi PUT ghi audit log (admin + thời gian)
- [ ] `GET /subscription/payments` trả đúng lịch sử của user, paginated
- [ ] `SubscriptionGuard` block khi `proExpiresAt` đã qua dù `subscriptionTier` vẫn là PRO (BR-11)

### Done Definition cho FE-Student:

- [ ] Trang `/pricing` hiện bảng so sánh FREE vs PRO
- [ ] Luồng checkout → redirect → polling → hiện PRO status hoạt động end-to-end
- [ ] `<PaywallModal />` trigger đúng khi FREE user dùng PRO feature
- [ ] `<AiExplainButton />` hiện quota badge đúng (FREE) hoặc không giới hạn (PRO)
- [ ] Settings tab subscription hiện đúng thông tin, nút cancel hoạt động
- [ ] `/payment/success` và `/payment/failed` page có nội dung phù hợp
- [ ] `<SubscriptionCard />` hiện lịch sử thanh toán từ `GET /subscription/payments`
- [ ] Khi FREE user có subscription CANCELLED chưa hết hạn → checkout bị chặn + hiện thông báo đúng (BR-05)
- [ ] `<ProBadge />` xuất hiện ở Header, Profile, Leaderboard khi PRO

### Done Definition cho FE-Admin:

- [ ] `/admin/subscriptions` list đúng, filter hoạt động
- [ ] Grant PRO modal hoạt động, record xuất hiện trong list
- [ ] Force-cancel hoạt động, user về FREE ngay
- [ ] Revenue dashboard hiện MRR, chart, breakdown gateway
- [ ] `/admin/subscriptions/settings` load config đúng, save thành công, hiện audit trail
- [ ] Feature flag: tắt Pronunciation → `SubscriptionGuard` block ngay cả PRO user
- [ ] Thay đổi giá trong settings → checkout BE tính đúng giá mới

---

## 13. Out of Scope (v1)

- Auto-renew subscription (recurring charge) — v2
- Refund tự động — xử lý thủ công bởi Admin
- Coupon / discount code
- Trial period (dùng thử 7 ngày miễn phí)
- Team/Group subscription (nhiều user trong 1 tài khoản tổ chức)
- Invoice/VAT generation
- Apple Pay / Google Pay
- Affiliate / referral program

---

## 14. Dependencies & Risks

| Dependency | Risk | Mitigation |
|---|---|---|
| Đăng ký merchant MoMo/VNPay | Cần giấy phép kinh doanh, mất 1-2 tuần | Bắt đầu process đăng ký sớm; dùng Stripe sandbox trong khi chờ |
| Gemini API key quota | Key hiện tại `limit: 0` | Tạo key mới từ project mới trong Google AI Studio |
| Azure Speech API | Cần Azure subscription | Đăng ký free tier $200 credit; estimate usage trước |
| Cron job reliability | Cron fail = user giữ PRO sai | Thêm monitoring + alerting; có manual trigger endpoint cho Admin |

---

## 15. Lịch triển khai đề xuất

| Sprint | Nội dung |
|---|---|
| Sprint 1 (BE) | DB migration, Subscription entity, Payment entity, AI quota tracking |
| Sprint 1 (FE) | `<PaywallModal />`, update `useAuth` với `isPro`, quota badge trên AI Explain |
| Sprint 2 (BE) | Checkout endpoint, Webhook handler (MoMo), Cron expire |
| Sprint 2 (FE) | Trang `/pricing`, luồng checkout, `/payment/success|failed` |
| Sprint 3 (BE) | Admin endpoints (list, grant, cancel, revenue), Pronunciation API (Azure) |
| Sprint 3 (FE) | Settings subscription tab, Admin panel subscriptions |
| Sprint 4 | E2E testing, security audit webhook, go-live |
