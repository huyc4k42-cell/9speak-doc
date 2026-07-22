# Yêu cầu CMS — Dashboard & Quản lý User

> **Mục tiêu:** Cung cấp cho team một trang CMS để (1) nắm hiện trạng sản phẩm qua các chỉ số then chốt, và (2) tra cứu thông tin user. Tài liệu này mô tả **yêu cầu sản phẩm** (cái cần hiển thị và vì sao), phần triển khai kỹ thuật do tech lead quyết định.
>
> **Phạm vi v1:** chủ yếu **xem & tra cứu**, kèm **một thao tác quản trị: nâng cấp / đổi gói cho user** (xem [§2.7](#27-nâng-cấp--đổi-gói-cho-user-admin-action)). Các thao tác sửa khác (trừ lượt thủ công, ban, refund…) chưa nằm trong v1.
> **Đối tượng dùng:** team nội bộ (business, product, vận hành, CSKH).
> **Bản mẫu giao diện (mockup):** [`cms-mockup.html`](./cms-mockup.html) — mở bằng trình duyệt để xem layout & tương tác.
> **Ngày tạo:** 2026-06-11 · **Cập nhật:** 2026-06-12 (đồng bộ với bản mockup).

---

## 0. Quy ước

- **Ưu tiên:** `P0` = bắt buộc cho v1 · `P1` = nên có · `P2` = bổ sung sau.
- **Khung thời gian (time range):** mọi chỉ số/biểu đồ có tính thời gian đều áp dụng bộ chọn chung ở mục [4](#4-bộ-lọc--khoảng-thời-gian-chung).
- **Quy ước biểu đồ cột thời gian:** **mỗi cột = 1 ngày**, số cột = số ngày trong khung thời gian đang chọn (xem mục [4](#4-bộ-lọc--khoảng-thời-gian-chung)).
- "Lượt" = số lần chấm điểm (mock test / practice) mà user được dùng.

---

## 1. Phần A — Dashboard (tổng hợp key metrics)

Trang tổng quan gồm: **thanh chọn khung thời gian** ở trên cùng → **các thẻ số (KPI cards)** → **biểu đồ** theo từng nhóm. KPI cards hiển thị giá trị tổng trong kỳ + so sánh kỳ trước (▲▼ %). Biểu đồ thể hiện **biến thiên theo ngày** trong khung thời gian.

Dashboard chia làm **3 nhóm**: Kinh doanh · Tăng trưởng · Kết quả học tập.

### 1.1. Nhóm Kinh doanh / Doanh thu

| # | Chỉ số | Định nghĩa | Ưu tiên | Hiển thị (theo mockup) |
|---|--------|-----------|---------|----------------|
| B1 | Số user trả phí (đang hoạt động) | User có gói premium còn hạn (chưa hết hạn) | P0 | KPI card |
| B2 | User mua mới trong kỳ | Số user lần đầu kích hoạt gói trả phí trong kỳ | P0 | KPI card |
| B3 | Tỷ lệ chuyển đổi Free → Paid | % user từng free đã nâng cấp lên trả phí | P0 | KPI card (%) |
| B6 | Sắp hết hạn | Số user có gói hết hạn trong ≤ 7 ngày tới | P0 | KPI card (cảnh báo) |
| B7 | Đã hết hạn (churn) | Số user gói đã hết hạn trong kỳ và chưa gia hạn | P1 | KPI card |
| B8 | Doanh thu | Tổng doanh thu trong kỳ (nếu hệ thống lưu được — xem [mục 6](#6-câu-hỏi-mở--cần-tech-lead-xác-nhận)) | P1 | KPI card + **biểu đồ cột/ngày** |
| B9 | ARPU | Doanh thu trung bình / user trả phí (phụ thuộc B8) | P2 | KPI card |
| B4 | Phân bố theo gói | Số user theo từng gói premium đang sở hữu | P1 | Bar/pie chart |

**Biểu đồ nhóm này (theo mockup):**
- 📊 **Doanh thu** — cột/ngày (đơn vị triệu ₫).

### 1.2. Nhóm Tăng trưởng / Engagement

| # | Chỉ số | Định nghĩa | Ưu tiên | Hiển thị (theo mockup) |
|---|--------|-----------|---------|----------------|
| G1 | Tổng số user | Tổng tài khoản đã đăng ký (tích lũy) | P0 | KPI card |
| G2 | User mới trong kỳ | Số tài khoản tạo mới theo ngày tham gia | P0 | KPI card + **cột/ngày** + đường |
| G3 | User hoạt động (DAU / MAU) | Số user có ≥ 1 phiên luyện trong ngày / tháng | P0 | KPI card |
| G4 | Tổng số phiên luyện trong kỳ | Số phiên practice + mock test, tách riêng 2 loại | P0 | KPI card + **cột chồng/ngày** |
| G6 | Tỷ lệ hoàn thành onboarding | % user hoàn tất onboarding sau khi đăng ký | P1 | **Funnel** (Đăng ký → Mục tiêu → Mic-check → Phiên đầu) |
| G5 | Phiên trung bình / user hoạt động | G4 ÷ số user hoạt động | P1 | KPI card |
| G7 | Retention (D1 / D7 / D30) | % user quay lại sau 1 / 7 / 30 ngày kể từ đăng ký | P1 | Bảng cohort hoặc line chart |
| G8 | Tỷ lệ user còn lượt vs hết lượt | Phân bố user theo còn/hết lượt mock & practice | P2 | Bar chart |
| G9 | Lượt **Mock đã dùng** TB / user | Trung bình số lượt mock test mỗi user đã **tiêu thụ** trong kỳ | P1 | KPI card |
| G10 | Lượt **Practice đã dùng** TB / user | Trung bình số lượt practice mỗi user đã **tiêu thụ** trong kỳ | P1 | KPI card |

> ⚠️ **G9, G10 — dữ liệu chưa có sẵn trực tiếp.** Hồ sơ user chỉ lưu lượt **còn lại** (`remaining*`), **không** lưu lượt đã dùng. Hai chỉ số này phải **suy ra từ số phiên (session) hoặc charge ledger** → cần backend tổng hợp. Xem câu hỏi mở [§7](#7-câu-hỏi-mở--cần-tech-lead-xác-nhận).

**Biểu đồ nhóm này (theo mockup):**
- 📈 **User mới & Phiên luyện** — biểu đồ đường, 2 đường (user mới + tổng phiên), theo ngày.
- 📊 **Phiên luyện** — cột chồng/ngày, tách **Practice** vs **Mock test**.
- 📊 **User mới** — cột/ngày.
- 🪜 **Funnel onboarding** — 4 bước, kèm % hoàn tất.

### 1.3. Nhóm Kết quả học tập

| # | Chỉ số | Định nghĩa | Ưu tiên | Hiển thị (theo mockup) |
|---|--------|-----------|---------|----------------|
| L1 | Phân bố band hiện tại | Số user theo từng mức band hiện tại (current band) | P0 | **Histogram** (cột theo mức band) |
| L5 | Band trung bình theo 4 tiêu chí | Điểm TB Fluency / Lexical / Grammar / Pronunciation trên toàn bộ bài chấm | P1 | **Thanh ngang** 4 tiêu chí |
| L3 | Khoảng cách Target − Current | Chênh lệch trung bình giữa band mục tiêu và hiện tại | P1 | KPI/label kèm biểu đồ L5 |
| L2 | Phân bố band mục tiêu | Số user theo từng mức band mục tiêu (target band) | P1 | Histogram |
| L4 | Cải thiện band theo thời gian | Thay đổi band trung bình của nhóm user theo thời gian | P1 | Line chart |
| L8 | Tổng số bài đã chấm | Tổng số bài AI đã chấm điểm trong kỳ | P1 | KPI card |
| L6 | Phần thi được luyện nhiều nhất | Tỷ trọng phiên theo Part 1 / 2 / 3 và mock vs practice | P2 | Bar chart |
| L7 | Chủ đề luyện nhiều nhất | Top chủ đề (topic/category) được luyện nhiều nhất | P2 | Bảng top N |

**Biểu đồ nhóm này (theo mockup):**
- 📊 **Phân bố band hiện tại** — histogram, cột theo mức band (3.5 → 7.5+).
- 📊 **Band trung bình theo 4 tiêu chí** — 4 thanh ngang + chỉ số khoảng cách Target − Current.

> *Lưu ý:* các biểu đồ ở nhóm Kết quả học tập là **ảnh chụp hiện trạng theo kỳ** (không phải biến thiên theo ngày như nhóm Kinh doanh/Tăng trưởng).

---

## 2. Phần B — Danh sách & tra cứu User (read-only)

### 2.1. Bảng danh sách user

Mỗi dòng là một user. Các cột (theo mockup):

| Cột | Mô tả | Ưu tiên |
|-----|-------|---------|
| User (Avatar + Tên + Email) | Gộp 1 cột: ảnh đại diện, tên, email | P0 |
| Loại | member / admin (dạng nhãn màu) | P0 |
| Band | Band hiện tại → band mục tiêu (vd `5.5 → 7.0`) | P0 |
| Gói | Gói premium đang sở hữu (hoặc "Free") | P0 |
| Trạng thái | Đang dùng / Sắp hết hạn / Đã hết hạn / Free (nhãn màu) | P0 |
| Hết hạn | Ngày gói hết hiệu lực | P1 |
| Lượt Mock | Lượt mock còn lại, tách trả phí + miễn phí (vd `8 +2f`) | P0 |
| Lượt Practice | Lượt practice còn lại, tách trả phí + miễn phí | P0 |
| Phiên | Tổng số phiên mock + practice user đã làm | P1 |
| Tham gia | Ngày tạo tài khoản | P0 |

> Cột **Hoạt động gần nhất** và **Trạng thái onboarding** hiển thị trong trang chi tiết (drawer) thay vì trên bảng để gọn.

### 2.2. Tìm kiếm

- `P0` Một ô tìm kiếm chung — gõ tìm theo **tên hoặc email** (khớp một phần).

### 2.3. Bộ lọc (dropdown trên thanh công cụ)

- `P0` **Loại tài khoản:** member / admin.
- `P0` **Trạng thái hạn:** Đang dùng / Sắp hết hạn / Đã hết hạn / Free.
- `P0` **Gói:** Free / Standard / Premium / Pro.
- `P1` Band hiện tại (khoảng từ–đến).
- `P1` Ngày tham gia (khoảng thời gian).
- `P1` Còn lượt / hết lượt (mock, practice).

### 2.4. Sắp xếp

- `P0` Mới nhất / Cũ nhất (theo ngày tham gia).
- `P0` Band cao → thấp.
- `P1` Sắp hết hạn, Lượt còn lại, Hoạt động gần nhất.

### 2.5. Phân trang & xuất dữ liệu

- `P0` Phân trang danh sách (kèm tổng số kết quả — vd "Hiển thị 20 / 19.142 user").
- `P1` Nút **Xuất CSV** danh sách đang lọc.

### 2.6. Trang chi tiết user — dạng drawer trượt phải (read-only)

Khi bấm vào 1 dòng, mở **panel chi tiết trượt từ phải** với các nhóm thông tin:

- `P0` **Hồ sơ:** tên, email, avatar, loại tài khoản, band hiện tại, band mục tiêu, ngày tham gia, **hoạt động gần nhất**.
- `P0` **Gói & lượt:** gói hiện tại, trạng thái, ngày hết hạn, lượt mock & practice còn lại (trả phí + miễn phí).
- `P1` **Onboarding:** mục tiêu học, trạng thái hoàn tất onboarding.
- `P1` **Lịch sử phiên luyện gần đây:** danh sách phiên mock/practice + band đạt được + thời gian.
- `P2` **Lịch sử sử dụng lượt** (các lần bị trừ lượt chấm điểm) và **lịch sử mua gói / thanh toán**.

### 2.7. Nâng cấp / đổi gói cho user (admin action)

> Thao tác quản trị duy nhất trong v1. Cho phép admin **thay đổi loại gói** của user (nâng cấp, hạ cấp, hoặc đổi sang Free).

- `P0` Trong drawer chi tiết user, ở khối **Gói & lượt**, có nút **"Nâng cấp / Đổi gói"**.
- `P0` Bấm nút → mở **form (modal)** gồm:
  - **Gói mới:** chọn trong danh sách gói (Free / Standard / Premium / Pro…).
  - **Ngày hết hạn:** chọn ngày gói hết hiệu lực (ẩn khi chọn gói Free).
  - **Ghi chú:** ô nhập tự do (tùy chọn) — vd lý do đổi gói.
- `P0` Nút **Xác nhận** → cập nhật gói của user; **Hủy** → đóng form, không đổi.
- `P0` Sau khi xác nhận:
  - Gói, trạng thái hạn (Đang dùng / Sắp hết hạn / Đã hết hạn / Free), ngày hết hạn của user được **cập nhật ngay** trên bảng và drawer.
  - **Số lượt mock/practice** được đặt theo gói mới (đổi sang Free → về 0).
  - Hiển thị **thông báo (toast)** xác nhận đã đổi gói thành công.
- `P1` Chỉ **tài khoản admin** mới thấy & dùng được nút này.
- `P1` Ghi **nhật ký thay đổi (audit log):** ai đổi, thời điểm, từ gói nào → gói nào, ghi chú — phục vụ tra soát.

---

## 3. Bố cục thực tế (theo mockup)

```
┌──────────────┬──────────────────────────────────────────────────────┐
│  9Speak CMS  │  Dashboard            Xin chào, Admin  (avatar)       │
│              ├──────────────────────────────────────────────────────┤
│ ▸ Dashboard  │ [Hôm nay][7 ngày*][30 ngày][Tháng này][Tùy chọn]  ☑ So sánh kỳ trước │
│ ▸ Users      │                                                      │
│              │ ── KINH DOANH ──────────────────────────             │
│ (Gói & DT)   │ [Trả phí][Mua mới][Conversion][Sắp hết hạn]          │
│ (Cài đặt)    │ [Churn][Doanh thu][ARPU]                             │
│              │ 📊 Doanh thu/ngày                                    │
│              │                                                      │
│              │ ── TĂNG TRƯỞNG ─────────────────────────             │
│              │ [Tổng user][User mới][DAU/MAU][Phiên]                │
│              │ 📈 User mới & Phiên (đường)   🪜 Funnel onboarding    │
│              │ 📊 Phiên/ngày (chồng)         📊 User mới/ngày        │
│              │                                                      │
│              │ ── KẾT QUẢ HỌC TẬP ─────────────────────             │
│              │ 📊 Phân bố band          📊 4 tiêu chí               │
└──────────────┴──────────────────────────────────────────────────────┘
```

---

## 4. Bộ lọc & khoảng thời gian chung

- `P0` **Bộ chọn khung thời gian** áp dụng cho toàn bộ biểu đồ thời gian của Dashboard:
  **Hôm nay / 7 ngày / 30 ngày / Tháng này / Tùy chọn (custom range)**.
- `P0` **Quy tắc hiển thị biểu đồ cột:** **mỗi cột = 1 ngày**; số cột thay đổi theo khung chọn:

  | Khung chọn | Số cột (ngày) |
  |---|---|
  | Hôm nay | 1 |
  | 7 ngày | 7 |
  | 30 ngày | 30 |
  | Tháng này | Số ngày đã qua trong tháng (vd hôm nay 12 → 12 cột) |
  | Tùy chọn | Theo số ngày trong khoảng người dùng chọn (mặc định mockup: 14) |

- `P0` **Xử lý hiển thị khi nhiều cột:** nhãn trục ngày dạng `d/m`, hiển thị **thưa** (≈ tối đa 12 mốc) khi nhiều ngày; nhãn số trên đầu cột chỉ hiện khi ít cột (≤ 10), còn lại **xem chi tiết bằng hover (tooltip: ngày + giá trị)**.
- `P1` Nút **so sánh với kỳ trước** (hiển thị % thay đổi trên KPI card).
- `P2` Lọc theo phân khúc: loại tài khoản, gói, band range.

---

## 5. Tương tác & chi tiết UI (theo mockup)

- `P0` Bấm chip khung thời gian → toàn bộ biểu đồ thời gian **render lại theo số ngày** tương ứng.
- `P0` **Hover vào cột / điểm** trên biểu đồ → tooltip hiện ngày + giá trị.
- `P0` Bấm 1 dòng user → mở **drawer chi tiết** (đóng bằng nút × hoặc bấm nền mờ).
- `P0` Bộ lọc + tìm kiếm + sắp xếp ở trang Users hoạt động **đồng thời** (lọc gộp), kèm cập nhật số đếm kết quả.
- Nhãn trạng thái dùng **màu phân biệt:** Đang dùng (xanh) · Sắp hết hạn (cam) · Đã hết hạn (đỏ) · Free (xám).

---

## 6. Ngoài phạm vi v1 (Out of scope)

- Chỉnh sửa user **khác ngoài đổi gói** (cấp/trừ lượt thủ công, ban/khóa, refund) — cân nhắc cho v2. *(Nâng cấp/đổi gói đã đưa vào v1 — xem §2.7.)*
- Nhóm chỉ số **Vận hành / AI** (tỷ lệ chấm fail, chi phí AI, charge kẹt) — chưa đưa vào v1.
- Phân quyền chi tiết theo vai trò trong CMS — v1 chỉ cần quyền xem.
- Cảnh báo / thông báo tự động (email khi churn tăng, v.v.).

---

## 7. Câu hỏi mở — cần tech lead xác nhận

> Một số chỉ số phụ thuộc vào việc hệ thống hiện có lưu đủ dữ liệu hay không. Nhờ tech lead xác nhận tính khả thi & ưu tiên:

1. **Doanh thu (B8, B9):** hệ thống có lưu **số tiền giao dịch** thực tế không, hay chỉ lưu loại gói? Nếu chỉ có loại gói, doanh thu cần map cứng giá gói hay lấy từ cổng thanh toán?
2. **DAU/MAU & Retention (G3, G7):** định nghĩa "hoạt động" = có phiên luyện, hay tính cả đăng nhập? Dữ liệu hoạt động lấy từ phiên luyện hay từ hệ thống analytics?
3. **Conversion Free → Paid (B3):** có xác định được mốc thời điểm user chuyển từ free sang paid không?
4. **Biểu đồ theo ngày:** dữ liệu phiên/doanh thu có **timestamp theo ngày** để dựng cột/ngày chính xác không? (cần cho mọi biểu đồ thời gian)
5. **Cải thiện band theo thời gian (L4):** band của user được cập nhật theo thời gian (lịch sử) hay chỉ lưu giá trị mới nhất?
6. **Hoạt động gần nhất (drawer):** có trường "last active" riêng, hay tạm dùng thời điểm cập nhật hồ sơ / phiên gần nhất?
7. **Top chủ đề / phần thi (L6, L7):** dữ liệu phiên có gắn rõ chủ đề (topic/category) và phần thi (part) để thống kê không?
8. **Phân trang & tìm kiếm user:** số lượng user hiện tại khoảng bao nhiêu (ảnh hưởng cách phân trang/tìm kiếm phía backend)?
9. **Khung "Tùy chọn":** cho người dùng nhập khoảng ngày tự do (date range picker) hay giới hạn tối đa (vd ≤ 90 ngày để tránh quá nhiều cột)?
10. **Đổi gói (admin — §2.7):** thao tác đổi gói gọi API nào của NineSpeak backend? Khi đổi gói có **tự động cấp/đặt lại số lượt** theo gói mới không (theo quy tắc nào)? Cần **giới hạn quyền chỉ admin** và **ghi audit log** ở mức nào?
11. **Lượt đã dùng TB/user (G9, G10):** tính từ **số session** mock/practice của user, hay từ **charge ledger** (`scoring_charges`)? Một session có luôn tương ứng đúng **1 lượt** không (vd mock test nhiều part tính 1 lượt)? Lượt miễn phí có gộp chung khi tính "đã dùng" không?

---

## 8. Tóm tắt ưu tiên cho v1 (P0)

**Dashboard — KPI cards:** B1, B2, B3, B6 · G1, G2, G3, G4 · L1.
**Dashboard — biểu đồ:** Doanh thu/ngày, User mới & Phiên (đường), Phiên/ngày (cột chồng), User mới/ngày, Phân bố band, 4 tiêu chí.
**Khung thời gian:** bộ chọn Hôm nay/7/30 ngày/Tháng này/Tùy chọn + quy tắc mỗi cột = 1 ngày.
**Users:** bảng với các cột P0 + tìm theo tên/email + lọc theo loại tài khoản/trạng thái hạn/gói + sắp xếp (mới nhất, band) + drawer chi tiết (hồ sơ + gói & lượt) + **nâng cấp/đổi gói** (admin action — §2.7).
