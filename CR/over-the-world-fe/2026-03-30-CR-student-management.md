# CR: Student Management — Admin & Teacher CRUD + Category Assignment

Date: 2026-03-30
Scope: `over-the-world-fe` (FE admin) + `over-the-world-be` (BE)

---

## 1) Context

Hiện tại Admin Frontend (`over-the-world-fe`) có **Account Management** chỉ dành cho ADMIN (redirect nếu là TEACHER). Không có màn hình riêng để quản lý học sinh (STUDENT role). Cũng chưa có khái niệm "gán học sinh vào category" — `Category` hiện chỉ có quan hệ với `Vocabulary`, không với `User`.

**Nhu cầu:**
- Admin và Teacher cần có nơi tập trung để xem danh sách học sinh, thêm/sửa/xóa học sinh.
- Admin và Teacher cần gán học sinh vào category (ví dụ: lớp A1, lớp B2) để quản lý nhóm học, sau này dùng để giao bài/kế hoạch theo nhóm.

---

## 2) Scope Changes

### 2.1 Backend (`over-the-world-be`)

**a) User–Category Many-to-Many join table**
- Thêm `@ManyToMany` giữa `User` và `Category` via join table `user_categories`.
- Field trên `User`: `categories: Category[]`
- Field trên `Category`: `students: User[]`

**b) Migration**
- `CreateUserCategoriesTable` — tạo join table `user_categories(userId, categoryId)` với FK cascade.

**c) UserService — thêm methods**
| Method | Mô tả |
|---|---|
| `getStudents(page, pageSizes, search)` | Lấy danh sách user có role = STUDENT, load relations `categories` |
| `assignCategories(userId, categoryIds)` | Gán danh sách category cho student (replace toàn bộ) |
| `removeCategory(userId, categoryId)` | Xóa 1 category khỏi student |
| `getStudentCategories(userId)` | Lấy danh sách categories của 1 student |

**d) UserController — thêm endpoints**
| Method | Path | Roles | Mô tả |
|---|---|---|---|
| `GET` | `/users/students` | ADMIN, TEACHER | Danh sách students (paginated, search) |
| `GET` | `/users/:id/categories` | ADMIN, TEACHER | Lấy categories của student |
| `POST` | `/users/:id/categories` | ADMIN, TEACHER | Gán categories cho student |
| `DELETE` | `/users/:id/categories/:categoryId` | ADMIN, TEACHER | Xóa 1 category khỏi student |

> Endpoint `GET /users/students` phải đặt trước `GET /users/:id` để tránh conflict route.

---

### 2.2 Frontend (`over-the-world-fe`)

**a) API Endpoints mới (src/apiEndpoints/index.ts)**
```ts
getStudentList: "/users/students",
getStudentCategories: "/users",       // + /:id/categories
postAssignStudentCategories: "/users", // + /:id/categories
deleteStudentCategory: "/users",       // + /:id/categories/:categoryId
```

**b) Service hooks mới**
- `useGetStudentListService.ts` — GET /users/students (paginated, search)
- `useGetStudentCategoriesService.ts` — GET /users/:id/categories
- `useAssignStudentCategoriesService.ts` — POST /users/:id/categories
- `useRemoveStudentCategoryService.ts` — DELETE /users/:id/categories/:categoryId

**c) Routes**
- `/admin/student-management/` — Danh sách học sinh
- `/admin/student-management/$studentId/` — Chi tiết & quản lý category của học sinh

Cả hai route accessible với cả ADMIN và TEACHER (không redirect).

**d) Nav item mới (AppAdminNavbar)**
```ts
{
  key: "student-management",
  icon: <School />,
  to: "/admin/student-management",
  text: "Student Management",
  // adminOnly: false — cả teacher đều thấy
}
```

**e) UI Components**
- `StudentManagementPage` — DataGrid với columns: Name, Email, Role, Categories (chips), Created At + actions Edit/Delete/View.
- `StudentDetailPage` — Hiển thị thông tin student + danh sách categories đã gán + button Assign/Remove.
- `AppAssignCategoryDialog` — Dialog multi-select category (dùng lại `useGetCategoryListAutocompleteService`) để gán cho student.

---

## 3) Business Rules

| ID | Rule |
|---|---|
| BR-70 | Chỉ ADMIN có thể xóa student account. |
| BR-71 | ADMIN và TEACHER đều có thể xem danh sách student và gán/xóa categories. |
| BR-72 | Mỗi student có thể thuộc nhiều categories. |
| BR-73 | Gán category (POST /users/:id/categories) là **merge** — không overwrite categories cũ, chỉ thêm các categories mới (backend xử lý deduplicate). |
| BR-74 | Xóa category (DELETE /users/:id/categories/:categoryId) xóa đúng 1 liên kết. |
| BR-75 | Student không tự quản lý category assignment — đây là quyền admin/teacher. |

---

## 4) Data Flow

```
[FE Admin] Student Management Page
  → GET /users/students?page=1&pageSizes=10&search=...
  → Hiển thị danh sách với category chips

[FE Admin] Student Detail Page
  → GET /users/:id (dùng lại endpoint hiện có)
  → GET /users/:id/categories
  → Hiển thị info + categories

[FE Admin] Assign Category Dialog
  → GET /category (autocomplete list)
  → POST /users/:id/categories { categoryIds: [...] }
  → Invalidate cache /users/:id/categories

[FE Admin] Remove Category
  → DELETE /users/:id/categories/:categoryId
  → Invalidate cache /users/:id/categories
```

---

## 5) Impact Analysis

| Layer | File/Module | Loại thay đổi |
|---|---|---|
| BE | `user.entity.ts` | Thêm `@ManyToMany Category` |
| BE | `category.entity.ts` | Thêm `@ManyToMany User` (inverse) |
| BE | `user.service.ts` | Thêm 4 methods |
| BE | `user.controller.ts` | Thêm 4 endpoints |
| BE | `user.module.ts` | Import `CategoryModule` hoặc thêm `Category` vào forFeature |
| BE | migrations | `CreateUserCategoriesTable` |
| FE | `apiEndpoints/index.ts` | Thêm 4 endpoints |
| FE | `services/queryService/` | 2 hooks mới |
| FE | `services/mutationService/` | 2 hooks mới |
| FE | `routes/admin/student-management/` | 2 routes mới |
| FE | `components/AppAdminNavbar/` | Thêm nav item |
| FE | `components/AppAssignCategoryDialog/` | Component mới |

**Không ảnh hưởng:**
- Category CRUD hiện có (chỉ thêm inverse relation, không break query hiện tại).
- Account Management hiện có (vẫn dùng GET /users chung, student-management chỉ dùng GET /users/students).
- Student app (`over-the-world-fe-student`) — không chạm.

---

## 6) Test Checklist

### Backend
1. `GET /users/students` trả về đúng danh sách STUDENT, có phân trang và search.
2. `POST /users/:id/categories` với body `{ categoryIds: ["id1","id2"] }` → student có 2 categories.
3. `POST /users/:id/categories` lại với body `{ categoryIds: ["id3"] }` → student có 3 categories (merge, không overwrite).
4. `DELETE /users/:id/categories/:categoryId` → student còn 2 categories.
5. `GET /users/:id/categories` → trả đúng danh sách categories hiện tại.
6. TEACHER gọi các endpoint trên → 200 OK.
7. STUDENT gọi các endpoint trên → 403.

### Frontend
1. Admin đăng nhập → thấy "Student Management" trong navbar.
2. Teacher đăng nhập → thấy "Student Management" trong navbar.
3. Danh sách học sinh load đúng, search works.
4. Click row → vào Student Detail → thấy categories.
5. Nhấn "Assign Category" → dialog hiện, multi-select → save → categories cập nhật.
6. Nhấn remove chip category → category bị xóa.
7. Admin có nút Delete student; Teacher **không** có nút Delete.

---

## 7) Migration Note

Sau khi deploy, chạy migration để tạo bảng `user_categories`. Không ảnh hưởng data hiện có.
