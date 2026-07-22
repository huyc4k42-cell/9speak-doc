# Mock Test — #02 Câu hỏi tự chọn (Custom Question)

> Ngày cập nhật: 2026-06-20 · Trạng thái: Chính thức · Nguồn: chuyển thể từ Google Doc "Mocktest V2" (v0.1, không có bản V2).

> **Đồng bộ với [01-chon-de-goi-y.md](01-chon-de-goi-y.md) (v2.0):** tab "Đề thi custom" là 1 trong 5 tab cố định ở màn Chọn đề (§1.3.3 của file 01) — dùng icon Part P1/P2/P3 để đồng bộ hình ảnh nhưng **không** áp dụng tag New/Hot/Forecast/Premium. Nội dung/luồng thêm-xoá-luyện câu hỏi tự chọn mô tả đầy đủ trong file này không đổi.

---

## 1. Tổng quan

**Mô tả ngắn:** Cho phép người dùng tự thêm câu hỏi Speaking (tìm online hoặc tự tạo) để luyện thi với chính đề mình quan tâm.

**Mục đích / vấn đề giải quyết:** Người học muốn luyện câu hỏi cụ thể (đề trường, đề dự đoán riêng, chủ đề yêu thích) ngoài bộ đề có sẵn. Tính năng tăng tính linh hoạt và giữ chân người dùng nâng cao.

**Phạm vi:**
- Trong phạm vi: tab "Câu hỏi tự chọn", empty state, modal thêm câu hỏi (phần thi + nội dung + tiêu đề), danh sách câu đã thêm, xoá câu, bắt đầu trả lời.
- Ngoài phạm vi: chấm điểm câu tự chọn (dùng chung pipeline phòng thi/report), lưu trữ dài hạn/đồng bộ nhiều thiết bị (cần chốt — xem câu hỏi mở).

**Đối tượng người dùng & bên liên quan:** Học viên nâng cao; Product; Dev FE (state câu hỏi); QA.

---

## 2. Yêu cầu chức năng

| Mã | Yêu cầu | Đầu vào | Đầu ra | Ghi chú |
|---|---|---|---|---|
| FR-01 | Hiển thị tab "Câu hỏi tự chọn" trong bộ lọc scope | Chọn tab | Khu vực custom | |
| FR-02 | Khi chưa có câu nào, hiển thị **empty state** + nút "+" | Danh sách rỗng | Empty state | |
| FR-03 | Cho người dùng **thêm câu hỏi** qua modal: Phần thi, Nội dung câu hỏi, Tiêu đề | Form modal | Câu hỏi mới trong danh sách | Part bắt buộc |
| FR-04 | **Validate** form khi thêm (bắt buộc chọn Phần thi) | Form | Báo lỗi nếu thiếu | |
| FR-05 | Hiển thị danh sách câu đã thêm dạng thẻ (phần thi, tiêu đề, nội dung rút gọn) | Danh sách | Lưới thẻ | |
| FR-06 | Cho **xoá** một câu hỏi | Thao tác xoá | Danh sách cập nhật | |
| FR-07 | Cho **Bắt đầu trả lời** từ một câu tự chọn | Chọn thẻ | Vào phòng thi với câu đó | |

**Quy tắc nghiệp vụ (Business rules):**
- **BR-01:** Phần thi là bắt buộc; gợi ý Part 2 nên ở dạng cue card (Describe… / You should say…).
- **BR-02:** Tiêu đề là tuỳ chọn; nếu trống hiển thị "(Chưa đặt tiêu đề)".
- **BR-03:** Nội dung câu hỏi rút gọn tối đa ~3 dòng ở thẻ (line-clamp).

**Phân quyền theo vai trò:**

| Vai trò | Được làm gì |
|---|---|
| User free | (Cần chốt) có/không được dùng câu hỏi tự chọn; nếu có thì vẫn theo quota lượt chấm |
| User premium | Thêm/xoá/luyện câu hỏi tự chọn |

---

## 3. Cơ chế hoạt động

**Luồng người dùng (workflow):**
1. Chọn tab "Câu hỏi tự chọn".
2. Nếu rỗng → thấy empty state → bấm "+".
3. Điền modal (Phần thi + Nội dung + Tiêu đề) → Lưu.
4. Câu hỏi xuất hiện trong danh sách.
5. Bấm "Bắt đầu trả lời" → vào phòng thi với câu đó; hoặc "Xoá".

**Use case chính:**
- **Bối cảnh:** user có đề trường muốn luyện.
- **Kịch bản chính:** thêm 1 câu Part 2 cue card → bắt đầu trả lời → ghi âm → nhận report.
- **Kịch bản thay thế / ngoại lệ:** không chọn Phần thi → báo lỗi; xoá nhầm → (cần chốt undo?).

**Luồng dữ liệu:** Câu hỏi do user nhập → lưu state (hiện tại: client-side trong mockup) → đưa vào phòng thi như một đề → chấm qua pipeline chung.

**Tích hợp:**
- Phòng thi (#04) nhận câu hỏi tự chọn như một đề.
- (Cần chốt) Lưu trữ BE để giữ câu hỏi qua phiên/thiết bị.

*Sơ đồ phù hợp: state diagram rỗng ↔ có danh sách ↔ modal mở.*

---

## 4. Xử lý ngoại lệ và trạng thái

| Trường hợp | Hành vi mong đợi / thông báo |
|---|---|
| Thiếu Phần thi khi lưu | Báo lỗi "Hãy chọn phần thi", chặn lưu |
| Nội dung quá ngắn/rỗng | (Cần chốt) cảnh báo / chặn |
| Danh sách rỗng | Empty state + nút "+" |
| Xoá câu hỏi | Cập nhật danh sách ngay (cần chốt confirm) |
| Mất dữ liệu khi reload | Phụ thuộc cơ chế lưu trữ (câu hỏi mở) |

**Các trạng thái giao diện:** rỗng (empty + CTA) / có danh sách (lưới thẻ) / modal mở (form).

---

## 5. Yêu cầu giao diện (UI/UX)

- Mockup: `mocktest-flow-mockup.html` — `renderCustom()`, modal `#cmModal`.
- Empty state: minh hoạ + tiêu đề "Đang chờ bạn thêm câu hỏi đầu tiên" + nút "+".
- Modal: chọn Phần thi (dropdown), textarea nội dung (placeholder ví dụ cue card), input tiêu đề.
- Thẻ câu: badge Part + tiêu đề + nội dung rút gọn + nút "Bắt đầu trả lời" + nút xoá.

---

## 6. Tiêu chí chấp nhận

| Mã | Tiêu chí (Given – When – Then) |
|---|---|
| AC-01 | Khi danh sách rỗng, thì hiển thị empty state + nút "+". |
| AC-02 | Khi bấm "+", thì mở modal thêm câu hỏi. |
| AC-03 | Khi lưu mà chưa chọn Phần thi, thì báo lỗi và không thêm. |
| AC-04 | Khi lưu hợp lệ, thì câu hỏi xuất hiện trong danh sách với đúng Part/tiêu đề/nội dung. |
| AC-05 | Khi bấm xoá, thì câu hỏi biến mất khỏi danh sách. |
| AC-06 | Khi bấm "Bắt đầu trả lời", thì vào phòng thi với câu đã chọn. |

---

## 7. Phần bổ sung

**Giả định (Assumptions):** Pipeline chấm xử lý được câu hỏi tự do (không thuộc catalog).

**Phụ thuộc (Dependencies):** Phòng thi (#04) · (tuỳ chọn) lưu trữ BE.

**Yêu cầu phi chức năng liên quan:**
- Hiệu năng: thêm/xoá phản hồi tức thì.
- Bảo mật: lọc nội dung nhập (tránh XSS khi render).

**Câu hỏi mở / chưa chốt:**
- [ ] Lưu câu hỏi tự chọn ở đâu (client tạm hay BE bền vững, đồng bộ thiết bị)?
- [ ] User free có được dùng không, giới hạn số câu?
- [ ] Có cần xác nhận trước khi xoá / undo?
- [ ] Có cho upload bản ghi âm sẵn (thay vì thu trực tiếp) không?
