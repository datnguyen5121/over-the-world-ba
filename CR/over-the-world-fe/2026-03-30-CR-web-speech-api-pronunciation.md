# CR: Web Speech API Pronunciation for Word Management (FE)

Date: 2026-03-30
Scope: `over-the-world-fe`

## 1) Context
- Nhu cầu: Từ màn hình Word Management, user cần phát âm nhanh theo dữ liệu đang có.
- Dữ liệu hiện có: mỗi vocabulary luôn có `word` và `pinyin`; không có `audioUrl` riêng.
- Ràng buộc: Không muốn mở rộng schema backend chỉ để phát âm.

## 2) Options Considered
1. Forvo/API bên thứ 3
- Ưu: Có nguồn phát âm cộng đồng.
- Nhược: Chi phí licensing/API, phụ thuộc external service, khó kiểm soát độ ổn định.

2. eSpeak-NG
- Ưu: Có thể self-host, output ổn định giữa thiết bị.
- Nhược: Chất giọng tiếng Trung kém tự nhiên hơn, tích hợp vận hành nặng hơn cho web app.

3. Web Speech API (browser)
- Ưu: Không thêm chi phí hạ tầng, tích hợp nhanh, phù hợp use case đọc nhanh trong UI.
- Nhược: Chất lượng phụ thuộc voice pack của máy người dùng.

## 3) Decision
- Chọn Web Speech API cho phase hiện tại.
- Không lưu link phát âm vào `partOfSpeech`; giữ đúng ngữ nghĩa dữ liệu.

## 4) Implementation Summary
- Tạo helper speech ở FE:
  - `src/utils/speech.ts`
  - Có normalize pinyin từ dạng số thanh (`ni3 hao3`) sang dấu thanh (`nǐ hǎo`).
  - Hàm `speakChineseText({ word, pinyin })`:
    - Ưu tiên đọc `word` (Hanzi) nếu có.
    - Fallback đọc `pinyin` đã normalize.
    - Cấu hình `lang = zh-CN`, `rate = 0.9`.
- Export helper qua `src/utils/index.ts`.
- Tích hợp vào Word Management:
  - `src/routes/admin/word-management/index.tsx`
  - Cột `Part of Speech` hiển thị thêm icon loa để phát âm theo row.

## 5) UX Notes
- Click icon loa không trigger row navigation/select.
- Nếu trình duyệt không hỗ trợ speech synthesis thì không phát âm (safe fail).

## 6) Risks
- Một số máy không có voice `zh-*` => chất lượng giảm hoặc fallback voice.
- Browser policy có thể giới hạn autoplay ngoài tương tác user; hiện tại trigger từ click nên phù hợp.

## 7) Test Checklist
1. Mở Admin > Word Management.
2. Ở mỗi row, click icon loa trong cột Part of Speech.
3. Xác nhận có phát âm (ưu tiên đọc chữ Hán).
4. Test với pinyin có số thanh (`ma1 ma5`) xem đọc tự nhiên hơn sau normalize.
5. Kiểm tra không ảnh hưởng thao tác edit/delete/view của row.

## 8) Follow-up (optional)
- Thêm tooltip hướng dẫn khi thiết bị không có giọng Trung.
- Có thể thêm toggle "đọc Hanzi" vs "đọc Pinyin" nếu BA yêu cầu.
