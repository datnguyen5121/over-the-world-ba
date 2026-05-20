# Đặc tả chức năng: Quản lý Bài viết (Posts)

**Ngày cập nhật:** 2026-05-03  
**Trạng thái:** BE ✅ Hoàn thiện | FE-Student ✅ Hoàn thiện | FE-Admin ❌ Chưa làm

---

## 1. Tổng quan

Chức năng Posts cho phép Admin/Teacher tạo và quản lý bài viết (blog), học sinh có thể đọc công khai không cần đăng nhập. Mục tiêu:

- Cung cấp nội dung học tập (ngữ pháp, văn hóa, mẹo học tiếng Trung)
- Tăng retention người dùng qua content thường xuyên
- SEO organic traffic qua route `/posts` public
- Teacher có thể tự publish tài liệu cho học sinh

### 1.1 User Stories

| Actor | Story | Acceptance Criteria |
|---|---|---|
| Teacher | Tôi muốn tạo bài viết dạng DRAFT để soạn trước, publish sau | Form lưu được DRAFT, không hiện với học sinh |
| Teacher | Tôi muốn publish bài ngay khi tạo | Status = PUBLISHED → hiện trên `/posts` trong vòng 60s |
| Teacher | Tôi muốn chỉnh sửa bài đã publish | Edit form load đúng nội dung, lưu thành công |
| Teacher | Tôi muốn assign bài vào đúng chủ đề | Có thể chọn category khi tạo/edit |
| Admin | Tôi muốn quản lý toàn bộ bài (kể cả DRAFT của teacher khác) | List all endpoint không bị giới hạn theo author |
| Student | Tôi muốn tìm bài theo chủ đề | Filter category hoạt động đúng |
| Student | Tôi muốn đọc bài liên quan | Related posts hiện sau nội dung chính |

### 1.2 Quy trình publish (không có review gate)

Hiện tại Teacher publish trực tiếp, không qua admin review. Đây là quyết định chủ ý phù hợp với quy mô nhỏ. Nếu sau này cần workflow duyệt bài → bổ sung status `PENDING_REVIEW`.

### 1.3 Phân quyền tóm tắt

| Action | STUDENT | TEACHER | ADMIN |
|---|---|---|---|
| Đọc bài PUBLISHED | ✅ | ✅ | ✅ |
| Tạo/Edit bài | ❌ | ✅ (của mình) | ✅ (tất cả) |
| Xóa bài | ❌ | ✅ (của mình) | ✅ (tất cả) |
| Tạo/Edit category | ❌ | ✅ | ✅ |
| Xóa category | ❌ | ❌ | ✅ |

---

## 2. Database Schema

### Bảng `post`

| Cột | Kiểu | Ghi chú |
|---|---|---|
| `id` | UUID (PK) | Auto-generate |
| `title` | varchar(255) | Tiêu đề |
| `slug` | varchar(255) UNIQUE | Auto-generate từ title, có xử lý tiếng Việt |
| `excerpt` | text | Tóm tắt (tối đa 500 ký tự) |
| `content` | text | Nội dung HTML đầy đủ |
| `coverImage` | varchar nullable | URL ảnh bìa |
| `status` | enum: `DRAFT`, `PUBLISHED` | Mặc định: `DRAFT` |
| `publishedAt` | timestamp nullable | Tự set khi status → PUBLISHED lần đầu |
| `tags` | simple-array nullable | Mảng string lưu dạng CSV |
| `viewCount` | int | Tăng tự động mỗi lần xem, mặc định 0 |
| `authorId` | UUID (FK → users) | |
| `categoryId` | UUID (FK → post_category) nullable | |
| `createdAt` | timestamp | |
| `updatedAt` | timestamp | |

### Bảng `post_category`

| Cột | Kiểu | Ghi chú |
|---|---|---|
| `id` | UUID (PK) | |
| `name` | varchar(100) | Tên danh mục |
| `slug` | varchar(255) UNIQUE | Auto-generate từ name |
| `description` | varchar(255) nullable | |
| `createdAt` | timestamp | |
| `updatedAt` | timestamp | |

---

## 3. Backend API

**Base URL:** `/posts` và `/post-categories`

### 3.1 Public Routes (không cần auth)

| Method | Endpoint | Mô tả |
|---|---|---|
| GET | `/posts` | Danh sách bài PUBLISHED, có filter/search/paginate |
| GET | `/posts/featured` | 3 bài mới nhất (dùng cho homepage) |
| GET | `/posts/slugs` | Tất cả slugs (dùng cho SSG) |
| GET | `/posts/slug/:slug` | Chi tiết bài theo slug |
| GET | `/posts/:id/related` | Bài viết liên quan (cùng category) |
| PATCH | `/posts/:id/view` | Tăng viewCount +1 |
| GET | `/post-categories` | Danh sách tất cả categories |

### 3.2 Admin/Teacher Routes (cần JWT + Role: ADMIN | TEACHER)

| Method | Endpoint | Mô tả |
|---|---|---|
| POST | `/posts` | Tạo bài viết mới |
| GET | `/posts/admin/:id` | Xem chi tiết bài (kể cả DRAFT) |
| PATCH | `/posts/:id` | Cập nhật bài viết |
| DELETE | `/posts/:id` | Xóa bài viết |

### 3.3 Category Routes (cần JWT + Role: ADMIN only)

| Method | Endpoint | Mô tả |
|---|---|---|
| POST | `/post-categories` | Tạo category mới |
| PATCH | `/post-categories/:id` | Cập nhật category |
| DELETE | `/post-categories/:id` | Xóa category |

### 3.4 Query Parameters — `GET /posts`

| Param | Kiểu | Mô tả |
|---|---|---|
| `page` | number | Trang hiện tại (mặc định: 1) |
| `pageSizes` | number | Số bài/trang (mặc định: 10) |
| `search` | string | Tìm kiếm theo title hoặc excerpt (ILIKE) |
| `categoryId` | UUID | Lọc theo danh mục |

### 3.5 Request Body — `POST /posts` / `PATCH /posts/:id`

```json
{
  "title": "Mẹo học từ vựng tiếng Trung hiệu quả",
  "excerpt": "Tóm tắt ngắn gọn về bài viết...",
  "content": "<p>Nội dung HTML đầy đủ...</p>",
  "coverImage": "https://...",
  "status": "DRAFT" | "PUBLISHED",
  "tags": ["hsk", "vocabulary"],
  "categoryId": "uuid-của-category"
}
```

### 3.6 Response — Bài viết đơn

```json
{
  "id": "uuid",
  "title": "...",
  "slug": "meo-hoc-tu-vung-tieng-trung-hieu-qua",
  "excerpt": "...",
  "content": "<p>...</p>",
  "coverImage": "https://...",
  "status": "PUBLISHED",
  "publishedAt": "2026-05-03T08:00:00.000Z",
  "tags": ["hsk"],
  "viewCount": 42,
  "authorId": "uuid",
  "categoryId": "uuid",
  "author": { "id": "uuid", "username": "teacher01" },
  "category": { "id": "uuid", "name": "Ngữ pháp", "slug": "ngu-phap" },
  "createdAt": "...",
  "updatedAt": "..."
}
```

### 3.7 Business Logic quan trọng

- **Auto-slug:** Title → lowercase → normalize NFD → remove diacritics → replace `đ→d` → slug. Nếu trùng slug → thêm `-1`, `-2`...
- **publishedAt:** Tự set khi `status = PUBLISHED` lần đầu tiên, không overwrite khi update lại
- **Re-slug:** Khi update title → tự generate slug mới (unique). ⚠️ **Breaking SEO** — URL cũ bị 404. Workaround hiện tại: không đổi slug nếu bài đã PUBLISHED (TODO: thêm logic check)
- **sanitizeAuthor:** Password bị loại bỏ khỏi response author
- **Tags:** Lưu dạng `simple-array` (PostgreSQL CSV). Chỉ dùng để hiển thị, **không có filter theo tag**. Tech debt: nếu cần filter sau → phải migrate sang relation table riêng.
- **Featured:** Lấy 3 bài `publishedAt DESC` — không có editorial control. Nếu cần ghim bài → thêm field `isFeatured boolean`.

### 3.8 Security Requirements

- **🔴 XSS Prevention:** Content lưu dạng HTML raw. BE PHẢI sanitize HTML trước khi save dùng `sanitize-html` (strip script tags, event handlers, javascript: URLs). FE-Student render bằng `dangerouslySetInnerHTML` — chỉ safe khi BE đã sanitize.
- **🟡 viewCount inflate:** `PATCH /posts/:id/view` không có auth và không rate limit → bot/crawler có thể inflate. Hiện tại chấp nhận được ở scale nhỏ. Nếu cần chính xác → thêm IP-based throttle hoặc chỉ count khi user đăng nhập.
- **🟡 `/posts/slugs` scale:** Trả toàn bộ slugs trong 1 request cho SSG. Assumption: số lượng bài < 1000. Nếu vượt → cần paginate hoặc incremental static regeneration.



## 4. Frontend Student (over-the-world-fe-student)

### 4.1 Routes

| Route | Kiểu | Mô tả |
|---|---|---|
| `/posts` | Public (SSR + revalidate 60s) | Danh sách bài viết |
| `/posts/[slug]` | Public (SSR + revalidate 60s) | Chi tiết bài viết |

### 4.2 Trang `/posts` (PostListClient)

**Tính năng:**
- Grid 3 cột (desktop) / 2 cột (tablet) / 1 cột (mobile)
- Filter theo category (button group)
- Search theo title/excerpt (form submit)
- Pagination (prev/next)
- Skeleton loading state
- Empty state khi không có bài

**PostCard hiển thị:**
- Ảnh bìa (aspect-video, hover scale)
- Badge category
- Title (line-clamp-2)
- Excerpt (line-clamp-2)
- Author username, ngày publish, viewCount
- Tags (tối đa 3 tags đầu)

### 4.3 Trang `/posts/[slug]` (PostDetailClient)

**Tính năng:**
- `generateStaticParams` → SSG với revalidate 60s
- `generateMetadata` → dynamic Open Graph + JSON-LD structured data
- `notFound()` khi slug không tồn tại
- Tự động increment viewCount khi mount (PATCH `/posts/:id/view`)

**Nội dung hiển thị:**
- Category badge (click → filter list)
- Title (h1)
- Meta: author, ngày publish, viewCount
- Tags với icon
- Excerpt (intro)
- Separator
- Cover image (nếu có)
- Content render bằng `dangerouslySetInnerHTML` (HTML)
  - Class: `prose prose-lg dark:prose-invert`
- Related posts grid (2 cột) cùng category

**SEO:**
- OpenGraph: title, description, type=article, publishedTime, authors, images, tags
- JSON-LD: Schema.org Article

### 4.4 API Endpoints (fe-student)

```typescript
getPostList: "/posts",
getPostFeatured: "/posts/featured",
getPostSlugs: "/posts/slugs",
getPostBySlug: "/posts/slug",      // + /:slug
getPostRelated: "/posts",           // + /:id/related
patchPostView: "/posts",            // + /:id/view
getPostCategories: "/post-categories",
```

### 4.5 Types

```typescript
export type PostStatus = "DRAFT" | "PUBLISHED";

export interface PostCategory {
  id: string;
  name: string;
  slug: string;
  description?: string;
}

export interface Post {
  id: string;
  title: string;
  slug: string;
  excerpt: string;
  content: string;
  coverImage?: string | null;
  status: PostStatus;
  publishedAt?: string | null;
  tags?: string[] | null;
  viewCount: number;
  authorId: string;
  categoryId?: string | null;
  author?: { id: string; username: string } | null;
  category?: PostCategory | null;
  createdAt: string;
  updatedAt: string;
}
```

---

## 5. Frontend Admin (over-the-world-fe) — ❌ CHƯA LÀM

### 5.1 Các màn hình cần xây dựng

| Route | Mô tả | Độ ưu tiên |
|---|---|---|
| `/admin/post-management` | Danh sách bài viết (table) | Cao |
| `/admin/post-management/create` | Tạo bài mới (form + rich text) | Cao |
| `/admin/post-management/$postId/edit` | Chỉnh sửa bài | Cao |
| `/admin/post-management/categories` | Quản lý post categories | Trung bình |

### 5.2 Màn hình danh sách bài viết

**Columns:**
- Title
- Category
- Status (badge DRAFT / PUBLISHED)
- Author
- Ngày publish / ngày tạo
- ViewCount
- Actions: Edit, Delete, Preview (mở `/posts/[slug]`)

**Filters:**
- Search theo title
- Filter status: Tất cả / DRAFT / PUBLISHED
- Filter category

### 5.3 Form tạo/chỉnh sửa bài viết

**Fields:**
| Field | Control | Validation |
|---|---|---|
| Title | Input | Required, max 255 |
| Excerpt | Textarea | Required, max 500 |
| Category | Select (load từ `/post-categories`) | Optional |
| Status | Select: DRAFT / PUBLISHED | Required |
| Cover Image | Input URL | Optional |
| Content | **TipTap rich text editor** | Required |

**Rich text editor:** Dùng **TipTap** (output HTML, tương thích với render hiện tại của fe-student)
- Extensions cần: Bold, Italic, Heading (H2, H3), BulletList, OrderedList, Link, Image, CodeBlock, Blockquote, HorizontalRule

**Lưu ý:** Tags **không implement** trong form — không có filter theo tag, value mặc định là `[]`.

**Image upload:** Chỉ nhận URL (không upload file). Teacher cần host ảnh trên Supabase Storage hoặc URL ngoài. Upload trực tiếp là v2.

### 5.4 Màn hình quản lý Post Categories

**Fields:** Name, Description
**Phân quyền:** ADMIN + TEACHER có thể tạo/sửa. Chỉ ADMIN xóa.
**Actions:** Thêm, Sửa, Xóa (xóa category → bài viết có categoryId đó set NULL, không xóa bài)

### 5.5 API Endpoints cần thêm vào `API_ENDPOINTS` (fe admin)

```typescript
// Posts Admin
getPostAdminList: "/posts/admin",
getPostAdminDetail: "/posts/admin",   // + /:id
postCreatePost: "/posts",
patchUpdatePost: "/posts",            // + /:id
deletePost: "/posts",                 // + /:id

// Post Categories (admin)
getPostCategoryList: "/post-categories",
postCreatePostCategory: "/post-categories",
patchUpdatePostCategory: "/post-categories",  // + /:id
deletePostCategory: "/post-categories",       // + /:id
```

### 5.6 Backend gaps cần bổ sung

| Gap | Mô tả | Priority |
|---|---|---|
| `GET /posts/admin` | List all posts (DRAFT + PUBLISHED) có filter status/search/category/paginate. Yêu cầu JWT + ADMIN\|TEACHER | 🔴 Cao |
| `sanitize-html` middleware | Sanitize content trước khi save — chống XSS | 🔴 Cao |
| Category CRUD cho TEACHER | Hiện tại chỉ ADMIN tạo/sửa category — TEACHER cần tạo category mới để assign khi viết bài | 🟡 Trung bình |

---

## 6. Gap & Việc cần làm

### BE
- [ ] Thêm `GET /posts/admin` — list all posts kể cả DRAFT (có pagination, filter status/search/category)
- [ ] Cài `sanitize-html` + sanitize content trước khi save (XSS prevention)
- [ ] Cho phép TEACHER tạo/edit post category (hiện chỉ ADMIN)

### FE-Admin
- [ ] Thêm API endpoints vào `API_ENDPOINTS`
- [ ] Tạo services: useGetPostAdminListService, useGetPostAdminDetailService, useCreatePostService, useUpdatePostService, useDeletePostService, useGetPostCategoryListService, useCreatePostCategoryService, useUpdatePostCategoryService, useDeletePostCategoryService
- [ ] Tạo route `/admin/post-management` — bảng danh sách (filter status, search, category)
- [ ] Tạo route `/admin/post-management/create` — form + TipTap editor
- [ ] Tạo route `/admin/post-management/$postId/edit` — form edit (preload data)
- [ ] Tạo route `/admin/post-management/categories` — CRUD post categories
- [ ] Thêm "Post Management" vào sidebar (AppAdminNavbar)

### FE-Student
- [ ] Thêm link "Bài viết" vào navbar/sidebar (hiện tại route `/posts` tồn tại nhưng không có nav link)

```
Admin/Teacher (FE-Admin)
    → POST /posts (status=DRAFT)
    → Soạn nội dung, preview
    → PATCH /posts/:id (status=PUBLISHED)
    → publishedAt auto-set

CDN/Cache (revalidate 60s)
    → SSR next.js fe-student

Student (FE-Student /posts)
    → GET /posts (public, paginate)
    → Click bài → /posts/:slug
    → PATCH /posts/:id/view (increment)
    → Xem related posts
```

---

## 8. Dependency

- `hanzi-writer` — không liên quan
- `@tiptap/*` — cần cài cho FE-Admin (chưa có)
- `dangerouslySetInnerHTML` — FE-Student đã dùng để render HTML content
- `@tailwindcss/typography` — FE-Student đã cài (dùng class `prose`)
