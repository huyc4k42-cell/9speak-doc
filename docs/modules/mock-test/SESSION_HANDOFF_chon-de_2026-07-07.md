# Session Handoff — Mock Test V2 §1 "Chọn đề & Gợi ý"

> Ngày đóng phiên: 2026-07-07 · Mục đích: bàn giao cho phiên làm việc mới, không mất ngữ cảnh.

---

## ⚠️ Đọc mục này trước tiên

Trong lúc phiên này diễn ra, **project đã bị tái cấu trúc bởi 1 luồng làm việc khác** (không phải conversation này):

- `features/speaking/` → tách thành `features/{mock-test,practice,billing,onboarding-dashboard,shared}/`.
- Xuất hiện cấu trúc tài liệu chính thức mới: `docs/modules/<module>/{PRD.md,TECH.md}` + `docs/modules/mock-test/screens/*.md` + `docs/modules/mock-test/gap.md`.
- `AGENTS.md` gốc giờ **bắt buộc đọc** `docs/modules/<module>/{PRD.md,TECH.md}` trước khi sửa bất kỳ phần nghiệp vụ nào (xem mục "Trước khi phản hồi — ĐỌC TÀI LIỆU" trong `9speak-web/AGENTS.md`).

**Hệ quả trực tiếp tới việc của phiên này:**

1. File tôi viết trong phiên `docs/PRD_MockTest_ChonDe_v2_2026-07-03.md` **đã được migrate** sang vị trí chính thức: `docs/modules/mock-test/screens/01-chon-de-goi-y.md`, trạng thái đã đổi từ "Nháp" → **"Chính thức"**.
2. File chính thức đó **đã được 1 phiên khác tiếp tục cập nhật** (phiên grill-me thứ 2 về "Part lẻ", 2026-07-07) — thêm quyết định về schema session Part lẻ, hành vi CTA "Bắt đầu Part X", và thứ tự ưu tiên build (D1 → Part lẻ → B1/B2). Xem Đổi log #13–15 và mục 1.10 trong file chính thức.
3. → **File cũ `docs/PRD_MockTest_ChonDe_v2_2026-07-03.md` (nếu còn tồn tại ở root `docs/`) đã lỗi thời, cần xoá hoặc gắn rõ "SUPERSEDED, xem docs/modules/mock-test/screens/01-chon-de-goi-y.md" để tránh 2 người đọc 2 bản khác nhau.**
4. Có nhánh code thật đang chạy: `feat/mocktest-exam-v2`, và `docs/modules/mock-test/gap.md` đang track gap cho phần Report (không phải phần Chọn đề tôi làm) — CTA "Bắt đầu Part X" hiện **vẫn dẫn vào full 3 Part** (broken promise, đã ghi nhận là việc cần làm ưu tiên cao).

**Việc đầu tiên phiên mới nên làm:** đọc `docs/modules/mock-test/screens/01-chon-de-goi-y.md` (bản mới nhất, không phải bản trong conversation này) + `docs/modules/mock-test/TECH.md` + `docs/modules/mock-test/gap.md` để nắm đúng trạng thái hiện tại, rồi mới quyết định có cần đồng bộ lại Figma/HTML demo theo các quyết định Part-lẻ mới hay không.

---

## Phiên này đã làm gì (tóm tắt theo trình tự)

### 1. Đọc & phân tích PRD gốc
Đọc file `.docx` PRD Mock Test V2 do user cung cấp, tập trung phân tích ưu/nhược điểm phần Mock Test V2 (8 tài liệu con: Chọn đề, Câu hỏi tự chọn, Mic, Phòng thi, Edge case, Processing, Report tổng quan, Report chi tiết). Dùng `/grill-me` nhiều vòng để chốt quyết định trước khi viết.

### 2. Viết PRD hợp nhất v1.0
Gộp v0.1+v0.2 của 8 file gốc thành 1 bản, tích hợp `examMode` (full/part1/2/3/custom) làm khái niệm xuyên suốt, vá 3 lỗ hổng (Custom Question thiếu trong examMode, quota Part-mode, resume). File: `Mock_Test_V2_PRD_Optimized_2026-07-01.md` (⚠️ **chỉ lưu trong scratchpad phiên, không có trong project** — nếu cần nội dung §2–§8/§ báo cáo học thuật, phải tái tạo lại hoặc tìm trong lịch sử conversation).

### 3. Redesign §1 theo mockup mới (layout 2 khối, card content model)
User gửi screenshot layout mới (dark/light mode) khác hẳn Hero 1-thẻ cũ. Chạy nhiều vòng `/grill-me` chốt: 2 khối độc lập (thẻ trạng thái + "Đề gợi ý cho bạn"), bỏ "Kiểm tra trình độ" như tính năng riêng, bỏ state "Chưa đặt mục tiêu", đơn giản hoá "Đang làm dở", tab bar dời xuống dưới, thuật toán gợi ý đổi sang random, card content model đầy đủ (icon F/1/2/3, tag Hot/New/Forecast/Premium). Viết thành `docs/PRD_MockTest_ChonDe_v2_2026-07-03.md` (nay đã migrate, xem cảnh báo ở đầu file).

### 4. Vòng grill-me thứ 3 — Card content model chi tiết
Chốt: icon loại đề = badge chữ/số 1 màu trung tính, tag Hot = top 3 theo lượt bắt đầu/30 ngày, tag New = tự động theo kỳ hệ thống, fallback khi thiếu đề gợi ý, engine không bắt buộc cần targetBand, v.v. (7 câu hỏi, đã chốt hết).

### 5. Đối chiếu thiết kế Figma với PRD → phát hiện & sửa lệch
Kết nối Figma (`https://www.figma.com/design/ZoBpOtGnfV8Pf8G9QDJrDq`), phát hiện file có **4 thế hệ thiết kế cũ chồng chéo** (~50 frame). Dọn dẹp còn 15 frame canonical, port thêm case "hết lượt chấm" từ thế hệ mới nhất, đồng bộ màu theo `components/ui/index.tsx` thật trong code (Button `bg-blue-600`, Badge variants). Sau đó tự đối chiếu lại với PRD, phát hiện: (a) copy "Dựa theo mục tiêu Band X" mâu thuẫn thuật toán random đã chốt, (b) thẻ trạng thái thiếu tách bạch "Sẵn sàng luyện tiếp" vs band, (c) card Custom Question thiếu nội dung rút gọn — đã sửa cả 3 trong Figma.

**→ Kết quả:** [Figma file](https://www.figma.com/design/ZoBpOtGnfV8Pf8G9QDJrDq/Untitled?node-id=37-7), frame chính `main-happy-path` (37:7) + các case biến thể + `case-het-luot-*`.

### 6. Đảo ngược quyết định badge "hết lượt chấm"
User chốt lại: khi hết lượt chấm **không hiển thị gì khác biệt** ở màn Chọn đề (đảo ngược BR-01b trước đó). Đã cập nhật PRD (đổi log #12) và sau đó cả HTML demo.

### 7. Xây HTML demo tĩnh
Tạo `ui-demo/mock-test-chon-de.html` (chạy qua config `ui-demo` có sẵn trong `.claude/launch.json`, port 4173) tái hiện đúng Figma đã chốt — không lấy phong cách ngoài web theo yêu cầu của user. Có demo bar để chuyển 3 state + trạng thái data (loading/error/empty).

### 8. Audit + fix vòng 1 (accessibility + design + PRD compliance)
Chạy `/accessibility-review` + `/design-critique`: phát hiện 3 lỗi Critical (tab bar/pill không dùng được bàn phím, modal thiếu focus trap/Escape/ARIA, contrast subtitle dưới ngưỡng), thiếu 3 state loading/error/empty, mobile layout vỡ khi status-card không stack. Đã sửa toàn bộ + thêm popup paywall dạng bảng 4 gói (theo ảnh tham khảo user gửi) + modernize banner Premium (đưa lên trên thẻ trạng thái) + redesign tag (bỏ nút "Mở khoá" riêng, không đổi PRD-locked content model).

### 9. Audit vòng 2 — tự phát hiện lỗi do chính mình gây ra
Sau khi thêm gradient cho tag Hot/New để "đẹp hơn", tự kiểm tra lại và phát hiện gradient làm tụt contrast dưới 4.5:1 ở đầu sáng (Hot: 3.76:1, New: 2.49:1) — quay lại solid color, đồng thời fix layout: title 2 dòng phá vỡ căn chỉnh giữa các card cùng hàng (đã test bằng cách đổi title dài trực tiếp trong browser), thêm `line-clamp` + `min-height`, thêm divider trước footer, đồng bộ data mẫu (chip Forecast) trên toàn bộ card.

### 10. Hero card "hook" — priority action
User gửi ảnh tham khảo (card có icon huy hiệu, hoạ tiết trang trí, CTA pill rộng). Thử `/canvas-design` và `/algorithmic-art` — cả 2 đều tạo **tác phẩm/công cụ độc lập**, không nhúng được vào UI đang chạy → tự thiết kế bằng SVG inline theo đúng tinh thần ảnh. Redesign thẻ trạng thái thành hero card: icon huy hiệu, hoạ tiết SVG mờ góc trên-phải, gradient chéo, CTA pill nổi bật, cho cả 3 state. Lần đầu bị giới hạn `max-width:640px` — user yêu cầu sửa full-width khớp layout, đã fix (bố cục ngang: nội dung trái + CTA gọn phải trên desktop, stack dọc trên mobile).

---

## Artifacts hiện có (đường dẫn chính xác)

| Loại | Đường dẫn | Trạng thái |
|---|---|---|
| PRD chính thức | `9speak-web/docs/modules/mock-test/screens/01-chon-de-goi-y.md` | ✅ Chính thức, đã cập nhật thêm bởi phiên khác (Part lẻ) |
| PRD cũ (phiên này viết) | `9speak-web/docs/PRD_MockTest_ChonDe_v2_2026-07-03.md` | ⚠️ Lỗi thời — cần xoá hoặc đánh dấu superseded |
| Figma | `https://www.figma.com/design/ZoBpOtGnfV8Pf8G9QDJrDq/Untitled?node-id=37-7` | Khớp PRD tại thời điểm 2026-07-03/04, **chưa cập nhật theo quyết định Part-lẻ mới (2026-07-07)** |
| HTML demo | `9speak-web/../ui-demo/mock-test-chon-de.html` (chạy `preview_start` config `ui-demo`, port 4173) | Khớp Figma, đã qua 2 vòng audit a11y+layout, **chưa cập nhật theo quyết định Part-lẻ mới** |
| Memory đã lưu | `feedback_design_must_match_prd.md` (memory system) | Nguyên tắc: luôn đối chiếu design với PRD trước khi báo hoàn thành |

---

## Việc còn dang dở / cần làm ở phiên mới

1. **Ưu tiên cao nhất:** đọc PRD chính thức mới nhất + `TECH.md` + `gap.md` trước khi làm gì tiếp — đừng dựa vào file cũ trong conversation này.
2. **Xoá/archive** `docs/PRD_MockTest_ChonDe_v2_2026-07-03.md` (bản trùng lặp, lỗi thời) sau khi xác nhận với user.
3. **Đồng bộ Figma + HTML demo theo quyết định Part-lẻ mới** (schema session, CTA "Bắt đầu Part X" phải thực sự dẫn vào session 1-Part thay vì full) — hiện Figma/HTML demo được build trước khi các quyết định này tồn tại, có thể đã lệch.
4. Các câu hỏi mở còn treo ở `docs/modules/mock-test/screens/07-report-tong-quan.md` addendum: copy chính xác banner chào mừng lần đầu (TBD content), đường dẫn thay thế đặt targetBand nếu bấm "Bỏ qua".
5. Xác nhận với PM: nội dung/giá trong popup pricing 4 gói trong HTML demo lấy nguyên văn từ ảnh user gửi — cần xác nhận đây là giá thật hay chỉ tham khảo giao diện.
