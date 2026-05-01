# CR: Web Speech API Pronunciation for Student App (FE-Student)

Date: 2026-03-30
Scope: `over-the-world-fe-student`

## 1) Context
- Nhu cầu: Student cần bấm nghe phát âm ngay tại các màn hình học từ vựng.
- Dữ liệu hiện có: vocabulary có `word` và `pinyin`, không có audio file riêng.
- Mục tiêu: Triển khai nhanh, không thêm backend/audio storage.

## 2) Decision
- Chọn Web Speech API để phát âm theo tương tác click.
- Dùng chung helper nội bộ để tái sử dụng và đồng nhất hành vi.

## 3) Implementation Summary
- Tạo helper:
  - `src/lib/speech.ts`
  - Normalize pinyin số thanh -> dấu thanh.
  - `speakChineseText({ word, pinyin })` với `zh-CN`, `rate = 0.9`.
- Tích hợp nút phát âm tại các vị trí học chính:
  1. `src/components/vocabulary/VocabularyCard.tsx`
  - Nút loa cạnh `pinyin`.
  2. `src/app/(protected)/vocabulary/[id]/page.tsx`
  - Nút loa ở block hiển thị `pinyin` của từ chi tiết.
  3. `src/app/(protected)/my-list/page.tsx`
  - Nút loa tại từng item danh sách của tôi.

## 4) Behavior
- Ưu tiên đọc `word` (Hanzi) để tự nhiên hơn.
- Nếu thiếu `word`, fallback đọc `pinyin` đã normalize.
- Mỗi lần phát âm mới sẽ cancel utterance trước đó để tránh chồng tiếng.

## 5) Risks
- Phụ thuộc voice pack của thiết bị (Windows/macOS/mobile) nên chất lượng có thể khác nhau.
- Một số thiết bị thiếu giọng Trung, vẫn có thể đọc nhưng không tối ưu.

## 6) QA Checklist
1. Trang vocabulary list: bấm icon loa ở card, nghe phát âm.
2. Trang vocabulary detail: bấm icon loa cạnh pinyin, nghe phát âm.
3. Trang my-list: bấm icon loa trên từng item, không làm điều hướng sai.
4. Test từ có pinyin dạng số thanh (`ni3 hao3`) để xác nhận normalize hoạt động.
5. Kiểm tra không ảnh hưởng các hành vi khác (thêm vào DS, favorite, xóa).

## 7) Next Iteration (optional)
- Toast nhẹ nếu browser không hỗ trợ Web Speech API.
- Thêm setting tốc độ đọc cho user.
