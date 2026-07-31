# PRD: Luồng kết nối MockTest ↔ Practice

| | |
|---|---|
| **Sản phẩm** | 9Speak — IELTS Speaking |
| **Tính năng** | Kết nối MockTest → Practice (vòng lặp học) |
| **Ngày** | 2026-07-25 |
| **Trạng thái** | Sẵn sàng triển khai — V1 |

---

## Mục tiêu

Đóng vòng lặp **Thi → Luyện → Thi lại** — hiện tại phần lớn người dùng thi MockTest xong không biết phải làm gì tiếp theo và không chuyển sang luyện Practice trong vòng 7 ngày.

Feature này tạo các điểm chuyển tiếp tự nhiên:
- Thi MockTest xong → biết ngay mình cần luyện tiêu chí nào
- Vào Practice từ MockTest → thấy rõ mình đang luyện vì lý do gì
- Vào trang Hiệu suất → nhận gợi ý hành động phù hợp với tình trạng thực tế

---

## Không thuộc phạm vi V1

| Tính năng | Lý do |
|----------|-------|
| Lọc bài luyện Practice theo đúng tiêu chí từ MockTest | Backend Practice chưa hỗ trợ lọc topic theo tiêu chí — cần backend update trước |
| Hiển thị 2 gợi ý cùng lúc khi Phát âm là tiêu chí yếu nhất | Phụ thuộc tính năng lọc phía trên |
| Gợi ý luyện khởi động trước khi thi MockTest | Sai timing — đây là lúc người dùng đang tập trung chuẩn bị thi |

---

## Phạm vi V1 — 4 thay đổi

| | Vị trí | Thay đổi |
|--|--------|---------|
| **A** | MockTest Report — cuối phần Tổng quan | Thêm zone "Bước tiếp theo": so sánh điểm + gợi ý luyện Practice |
| **B** | Practice | Thêm banner nhắc ngữ cảnh khi vào từ MockTest |
| **C** | Trang Hiệu suất `/hieu-suat` | Thêm action card gợi ý theo trạng thái hiện tại |
| **D** | MockTest Report — tab Phát âm | Thêm nhãn phụ cho nút luyện từng từ |

---

## Metrics

### Chỉ số chính

| Chỉ số | Hiện tại | Mục tiêu | Thời gian đo |
|--------|----------|---------|--------------|
| Tỷ lệ cross-module (có cả MockTest + Practice trong 7 ngày) | 16.2% | ≥ 25% | 30 ngày |
| Tỷ lệ nhấn CTA từ MockTest report | ~0% | ≥ 20% | 2 tuần |
| Tỷ lệ nhấn Action Card trên trang Hiệu suất | Chưa có | ≥ 30% | 30 ngày |

### Chỉ số cần giữ nguyên (không được tụt)

- Tỷ lệ hoàn thành report MockTest
- Tỷ lệ bắt đầu session Practice

---

---

## A. MockTest Report — Zone "Bước tiếp theo"

### Mô tả

Zone này đặt **sau bảng 4 tiêu chí (Tổng quan)**, trước phần chọn Part. Gồm hai phần:

1. Card so sánh điểm với lần thi trước
2. Nút gợi ý luyện tiêu chí yếu nhất trong Practice

**Điều kiện hiển thị:** Chỉ hiển thị khi báo cáo đã hoàn chỉnh — có đủ điểm band và phân tích cho cả 4 tiêu chí. Ẩn hoàn toàn khi báo cáo đang chờ chấm hoặc thiếu tiêu chí.

---

### A.1 Card so sánh điểm

Chỉ hiển thị khi người dùng đang xem **kết quả phiên thi mới nhất**.

**Nội dung:**
```
Lần trước: 6.0 → Lần này: 6.5 (+0.5)
```
```
Lần trước: 6.5 → Lần này: 6.0 (−0.5)
```

Khi điểm giảm, thêm dòng chú thích:
```
Band có thể dao động giữa các lần thi — tiếp tục luyện để ổn định kết quả.
```

**Không hiển thị khi:**
- Đây là lần thi đầu tiên (không có lần trước để so sánh)
- Đang xem lại lịch sử thi cũ

---

### A.2 Nút gợi ý luyện Practice

**Cách chọn tiêu chí gợi ý:** Tiêu chí có band thấp nhất trong 4 tiêu chí. Khi có 2+ tiêu chí cùng thấp, ưu tiên theo thứ tự: **Từ vựng → Ngữ pháp → Độ trôi chảy → Phát âm**.

*Thứ tự ưu tiên này dùng nhất quán ở mọi nơi trong feature.*

**Khi còn lượt luyện:**
```
[Nút]       Luyện Từ vựng tại Practice →
[Chú thích] Band 6.0 · 3 lượt còn lại
```
*(Người dùng không giới hạn: "Band 6.0 · Không giới hạn")*

Nhấn nút → chuyển sang Practice, kèm ngữ cảnh tiêu chí cần luyện.

**Khi hết lượt (0 lượt):**
```
[Nút] Nạp lượt để luyện Từ vựng →
```
Nhấn nút → mở màn nạp lượt **ngay tại trang này** — không chuyển sang Practice rồi mới báo hết lượt.

**Số lượng nút:** Luôn đúng **1 nút** trong V1.

---

### A.3 Tiêu chí chấp nhận

**AC-A01: Zone ẩn khi báo cáo chưa hoàn chỉnh**
- Cho: Báo cáo đang chấm hoặc thiếu tiêu chí
- Khi: Người dùng mở MockTest report
- Thì: Zone "Bước tiếp theo" không hiển thị

**AC-A02: Zone hiển thị đúng vị trí**
- Cho: Báo cáo hoàn chỉnh
- Khi: Người dùng cuộn đến cuối phần Tổng quan
- Thì: Zone xuất hiện sau bảng 4 tiêu chí, trước phần chọn Part

**AC-A03: Chọn đúng tiêu chí yếu nhất**
- Cho: Báo cáo với 4 tiêu chí có điểm band khác nhau
- Khi: Zone hiển thị
- Thì: Nút gợi ý đúng tiêu chí có band thấp nhất

**AC-A04: Tie-breaker đúng thứ tự**
- Cho: 2+ tiêu chí cùng band thấp nhất
- Khi: Zone hiển thị
- Thì: Gợi ý tiêu chí đầu tiên theo thứ tự Từ vựng → Ngữ pháp → Trôi chảy → Phát âm

**AC-A05: Card so sánh điểm — chỉ ở session mới nhất**
- Cho: Người dùng đang xem kết quả mới nhất, đã có lần thi trước
- Khi: Zone hiển thị
- Thì: Card so sánh xuất hiện với delta chính xác

**AC-A06: Card so sánh điểm — ẩn khi xem lịch sử cũ**
- Cho: Người dùng đang xem lại lịch sử thi cũ
- Khi: Zone hiển thị
- Thì: Card so sánh không xuất hiện; nút gợi ý Practice vẫn hiển thị bình thường

**AC-A07: Card so sánh điểm — lần thi đầu tiên**
- Cho: Đây là lần thi đầu tiên (không có lần trước)
- Khi: Zone hiển thị
- Thì: Card so sánh không xuất hiện; nút gợi ý Practice vẫn hiển thị

**AC-A08: Framing khi điểm giảm**
- Cho: Điểm band lần này thấp hơn lần trước
- Khi: Card so sánh hiển thị
- Thì: Dòng chú thích "Band có thể dao động..." xuất hiện bên dưới card

**AC-A09: Nút hiển thị đúng khi còn lượt**
- Cho: Người dùng còn lượt luyện
- Khi: Zone hiển thị
- Thì: Label "Luyện [Tiêu chí] tại Practice →" với tên tiêu chí đúng; chú thích hiển thị số lượt còn lại (hoặc "Không giới hạn" với người dùng không giới hạn)

**AC-A10: Nút chuyển đổi thành upsell khi hết lượt**
- Cho: Người dùng hết lượt luyện
- Khi: Zone hiển thị
- Thì: Label đổi thành "Nạp lượt để luyện [Tiêu chí] →"; nhấn mở màn nạp lượt ngay tại trang, không chuyển sang Practice

**AC-A11: Luôn đúng 1 nút**
- Cho: Bất kỳ trạng thái nào
- Khi: Zone hiển thị
- Thì: Chỉ có đúng 1 nút gợi ý

---

---

## B. Practice — Banner ngữ cảnh

### Mô tả

Khi người dùng vào Practice từ MockTest report, một banner hiển thị ở đầu trang để nhắc lý do đang luyện tiêu chí này.

**Nội dung banner:**
```
Đang luyện theo gợi ý từ phiên thi [ngày] · Xem lại báo cáo ›
```

---

### Vòng đời banner

| Thời điểm | Hành vi |
|-----------|---------|
| Vừa vào Practice từ MockTest | Banner hiển thị |
| Chuyển sang câu hỏi khác | Banner biến mất |
| Nhấn browser back rồi vào lại | Banner không hiển thị lại |
| Ai đó copy URL chia sẻ → người khác mở | Banner không hiển thị (không lộ thông tin phiên thi của người khác) |
| Vào Practice trực tiếp (không từ MockTest) | Banner không hiển thị |

---

### Tiêu chí chấp nhận

**AC-B01: Banner xuất hiện đúng lúc**
- Cho: Người dùng nhấn nút từ MockTest report
- Khi: Trang Practice tải xong
- Thì: Banner xuất hiện ở đầu trang với đúng ngày thi

**AC-B02: Nhấn "Xem lại báo cáo" đúng đích**
- Cho: Banner đang hiển thị
- Khi: Người dùng nhấn "Xem lại báo cáo ›"
- Thì: Chuyển về đúng báo cáo MockTest của phiên thi đó

**AC-B03: Banner biến mất khi chuyển câu**
- Cho: Banner đang hiển thị
- Khi: Người dùng chọn câu hỏi khác trong Practice
- Thì: Banner không còn hiển thị

**AC-B04: Banner không hiển thị lại sau browser back**
- Cho: Người dùng đã vào Practice từ MockTest, banner đã hiển thị
- Khi: Nhấn browser back rồi vào lại Practice
- Thì: Banner không hiển thị lần thứ hai

**AC-B05: Banner không hiển thị với tài khoản khác**
- Cho: URL Practice được chia sẻ sang tài khoản khác
- Khi: Người dùng đó mở URL
- Thì: Banner không hiển thị

---

---

## C. Trang Hiệu suất — Action Card

### Mô tả

Một card gợi ý hành động trên trang `/hieu-suat`, dựa trên thời điểm người dùng thi MockTest gần nhất. Luôn hiển thị đúng **1 card** (hoặc không có card nếu không cần).

---

### Logic 4 trạng thái

Ưu tiên hiển thị từ cao xuống thấp: **A → C → D → B**

| Trạng thái | Điều kiện | Nội dung gợi ý | Nhấn vào đâu |
|-----------|-----------|---------------|-------------|
| **A** | Chưa thi MockTest lần nào | Thi thử để biết band thật của bạn → | MockTest |
| **C** | Lần thi cuối: 7–30 ngày trước | Đã [X] ngày kể từ lần thi — thử lại để cập nhật band → | MockTest |
| **D** | Lần thi cuối: hơn 30 ngày trước | Đã lâu không kiểm tra — thi thử để cập nhật band → | MockTest |
| **B** | Lần thi cuối: trong vòng 7 ngày | Tiếp tục luyện [Tiêu chí yếu nhất] → | Practice |

**Lưu ý trạng thái B:**
- Không gợi ý "thi lại" — người dùng vừa thi, cần thời gian luyện trước
- Tiêu chí yếu nhất lấy từ kết quả MockTest gần nhất, theo thứ tự ưu tiên: Từ vựng → Ngữ pháp → Trôi chảy → Phát âm

**Lưu ý trạng thái C:**
- `[X]` = số ngày tính từ ngày thi đến hôm nay
- Không có điều kiện nào khác ngoài số ngày

---

### Xử lý trường hợp đặc biệt

| Trường hợp | Hành vi |
|-----------|---------|
| Dữ liệu chưa tải xong | Card không hiển thị |
| Lần thi cuối đúng ngày thứ 7 | Tính là ≥7 ngày → trạng thái C |
| Trạng thái B nhưng không xác định được tiêu chí yếu nhất | Hiển thị "Tiếp tục luyện Từ vựng →" (mặc định) |

---

### Câu hỏi cần design xác nhận trước khi build

| | Câu hỏi |
|--|---------|
| OQ-01 | Action Card đặt ở đâu trong layout trang Hiệu suất — trên hay dưới biểu đồ tổng hợp? |

---

### Tiêu chí chấp nhận

**AC-C01: Trạng thái A — chưa thi MockTest**
- Cho: Người dùng chưa có MockTest nào
- Khi: Vào trang Hiệu suất
- Thì: Card hiển thị "Thi thử để biết band thật của bạn →"; nhấn → vào MockTest

**AC-C02: Trạng thái B — trong vòng 7 ngày**
- Cho: Lần thi gần nhất dưới 7 ngày trước
- Khi: Vào trang Hiệu suất
- Thì: Card hiển thị "Tiếp tục luyện [Tiêu chí] →"; nhấn → vào Practice; không có từ "thi lại" hay "thi thử"

**AC-C03: Trạng thái C — 7 đến 30 ngày**
- Cho: Lần thi gần nhất từ 7–30 ngày trước
- Khi: Vào trang Hiệu suất
- Thì: Card hiển thị "Đã [X] ngày kể từ lần thi — thử lại để cập nhật band →" với số ngày chính xác; nhấn → vào MockTest

**AC-C04: Trạng thái D — hơn 30 ngày**
- Cho: Lần thi gần nhất hơn 30 ngày trước
- Khi: Vào trang Hiệu suất
- Thì: Card hiển thị "Đã lâu không kiểm tra — thi thử để cập nhật band →"; nhấn → vào MockTest

**AC-C05: Đúng ngày thứ 7**
- Cho: Lần thi gần nhất đúng 7 ngày trước
- Khi: Vào trang Hiệu suất
- Thì: Hiển thị trạng thái C (không phải B)

**AC-C06: Luôn đúng 1 card**
- Cho: Bất kỳ trạng thái nào
- Khi: Vào trang Hiệu suất
- Thì: Hiển thị đúng 1 card (hoặc không có card nào)

**AC-C07: Tie-breaker đúng thứ tự (trạng thái B)**
- Cho: MockTest gần nhất có 2 tiêu chí cùng band thấp nhất (ví dụ Từ vựng và Ngữ pháp)
- Khi: Card trạng thái B hiển thị
- Thì: Gợi ý "Tiếp tục luyện Từ vựng →" (Từ vựng ưu tiên trước Ngữ pháp)

---

---

## D. MockTest Report — Nhãn phân biệt nút luyện trong tab Phát âm

### Vấn đề

Trong tab Phát âm của MockTest report có nút **"Luyện ›"** (luyện phát âm từng từ, miễn phí). Với zone mới ở phần A cũng có nút "Luyện...", người dùng dễ nhầm lẫn hai nút này với nhau.

### Giải pháp

Thêm nhãn phụ vào nút "Luyện ›" trong tab Phát âm:

```
Luyện ›   (Miễn phí · 1 từ)
```

- Chỉ thêm nhãn này ở **tab Phát âm**
- Hành vi nút **không thay đổi**

### Tiêu chí chấp nhận

**AC-D01: Nhãn phụ hiển thị đúng chỗ**
- Cho: Người dùng mở tab Phát âm trong MockTest report
- Khi: Tab hiển thị
- Thì: Nút "Luyện ›" có nhãn phụ "(Miễn phí · 1 từ)"

**AC-D02: Hành vi nút không thay đổi**
- Cho: Nhãn phụ đã được thêm
- Khi: Người dùng nhấn nút "Luyện ›"
- Thì: Hành vi giống hệt trước — mini-drill từng từ như cũ

**AC-D03: Các tab khác không bị ảnh hưởng**
- Cho: Người dùng mở tab Từ vựng hoặc Ngữ pháp
- Khi: Tab hiển thị
- Thì: Nút "Luyện ›" (nếu có) không có nhãn phụ này

---

---

## Backlog V2

| Tính năng | Điều kiện để làm |
|----------|-----------------|
| Lọc bài luyện Practice theo tiêu chí từ MockTest (deep-link trực tiếp) | Backend Practice hỗ trợ lọc topic theo tiêu chí |
| Hiển thị 2 gợi ý khi Phát âm yếu nhất (Phát âm + tiêu chí yếu thứ 2) | Phụ thuộc tính năng lọc topic phía trên |
