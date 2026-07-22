# Mock Test V2 — §1 Chọn đề & Gợi ý — v2.0 (layout mới)

> Ngày: 2026-07-07 · Trạng thái: Chính thức · **Thay thế toàn bộ §1 trong bản PRD Mock Test V2 hợp nhất (2026-07-01)**
> Đồng thời bổ sung 1 mục nhỏ vào §7 (Report tổng quan) — xem cuối file.
> Nguồn quyết định: 3 phiên grill-me (cấu trúc tổng thể → Placement Test → Card content model) + phiên grill-me thứ 2 về Part lẻ (2026-07-07).

## Đổi log so với v1.0 (2026-07-01)

| # | Thay đổi | Lý do |
|---|---|---|
| 1 | Thẻ trạng thái (Hero cũ) và section **"Đề gợi ý cho bạn"** (lưới 3 thẻ) là 2 khối độc lập, cùng tồn tại | Layout mới tách rõ 2 vai trò: thẻ trạng thái xử lý case ưu tiên, lưới gợi ý là kênh khám phá thêm |
| 2 | **Bỏ hẳn "Kiểm tra trình độ" như 1 tính năng riêng** — chỉ là copy khác của state cold-start (attempts=0), bấm vào là Full Mock Test bình thường | Đơn giản hoá, không cần pipeline/report/quota riêng |
| 3 | **Bỏ state "Chưa đặt mục tiêu"** khỏi màn Chọn đề — targetBand được hỏi sau khi có kết quả bài thi Full đầu tiên (trong Report) | Không chặn user mới trước khi họ biết trình độ |
| 4 | Thẻ "Đang làm dở" đơn giản hoá: chỉ hiện Part đang dở, bỏ % + breakdown + thời gian còn | Theo yêu cầu tối giản của product owner |
| 5 | Tab bar dời xuống **dưới** "Đề gợi ý cho bạn"; lưới gợi ý trộn lẫn mọi scope (Full+Part1-3+Custom) | Cho phép gợi ý đa dạng hơn, không bị giới hạn theo tab đang chọn |
| 6 | Thuật toán "Đề gợi ý cho bạn": random trong đề **chưa làm**, bỏ band-matching | Đơn giản hoá — band-matching chỉ giữ lại cho thẻ trạng thái |
| 7 | Filter bar mới: Tất cả/Forecast/Đã làm/Chưa làm (pill, exclusive) + dropdown chọn kỳ | Thay cho ý tưởng filter đa nguồn đề (Cambridge/IELTS Trainer) — không vào scope lần này |
| 8 | Card đề có thêm: icon riêng theo loại (Full/Part1/2/3), tag Hot/New/Forecast/Premium | Định nghĩa mới theo yêu cầu — xem §1.3 |
| 9 | Khoá Premium: chỉ banner chung + tag trên card + popup paywall khi bấm — bỏ nút "Mở khoá" riêng trên card | Đơn giản hoá luồng, giảm 1 loại CTA |
| 10 | **Sửa lại: free user hết lượt chấm KHÔNG bị chặn vào thi** — vẫn thi bình thường ở mọi thẻ/card, chỉ khoá khi vào Report | "Lượt" = lượt chấm điểm AI, không phải lượt được phép thi. Bản trước ghi sai là chặn CTA "bắt đầu" |
| 11 | Thêm 3 mục **Goal / Non-functional Requirements / User Types** theo khung `/create-prd` | Chuẩn hoá tài liệu theo yêu cầu product owner |
| 12 | **Bỏ hẳn BR-01b (badge cảnh báo "hết lượt chấm")** — thẻ trạng thái khi hết lượt chấm hiển thị **y hệt bình thường**, không có bất kỳ tín hiệu/badge khác biệt nào | Product owner chốt lại: "khi hết lượt chấm không cần thể hiện gì" — đảo ngược quyết định BR-01b trước đó |
| 13 | **Chốt schema session Part lẻ** — dùng chung schema session hiện tại, chỉ thay `visibleParts` thành 1 phần tử (vd `["part2"]`); ledger tính 1 lượt chấm cho 1 Part visible; bỏ hardcode `["part1","part2","part3"]` tại `mock-exam-definition-from-catalog.ts:32` | Phiên grill-me thứ 2 về Part lẻ (2026-07-07) — không tạo schema mới |
| 14 | **Chốt hành vi CTA "Bắt đầu Part X"** — sau khi build, nút này tạo session Part lẻ (1 Part) và vào phòng thi; hiện tại đang hiển thị nhưng dẫn vào full 3 Part (broken promise → cần fix) | Phiên grill-me thứ 2 về Part lẻ (2026-07-07) |
| 15 | **Thứ tự ưu tiên build: D1 → Part lẻ → B1/B2** — CTA "Bắt đầu Part X" được ưu tiên build sau D1 do đang broken promise | Phiên grill-me thứ 2 về Part lẻ (2026-07-07) |

---

## Goal

### Business / Product Goal
- Tăng tỷ lệ user bắt đầu ≥1 bài Mock Test trong phiên vào màn Chọn đề (giảm ma sát quyết định).
- Tăng discovery đề thông qua "Đề gợi ý cho bạn" (random, trộn mọi scope) — giúp user khám phá đề ngoài thói quen chọn lặp lại.
- Tăng nhận diện đề Premium qua tag trực quan trên card, hỗ trợ chuyển đổi free → premium.

### User Benefits
- User mới (attempts=0): được mời rõ ràng vào 1 bài Full test đầu tiên qua "Kiểm tra trình độ", không phải tự mò catalog.
- User có bài dở: được nhắc tiếp tục ngay, không mất tiến độ, không bị phân tán bởi thông tin thừa (% hoàn thành, breakdown).
- User cũ: dễ khám phá đề mới nhờ gợi ý ngẫu nhiên + tag New/Hot, giảm việc luôn chọn lại đề quen thuộc.
- User free hết lượt chấm: không bị gián đoạn trải nghiệm luyện tập — vẫn thi bình thường, chỉ cần nâng cấp khi muốn xem report chi tiết.

---

## 1.1 Tổng quan

**Mô tả ngắn:** Màn chọn đề Thi thử gồm 2 khối gợi ý độc lập (thẻ trạng thái đơn ưu tiên theo tình huống user; lưới "Đề gợi ý cho bạn" khám phá ngẫu nhiên) và một lưới "Tất cả đề" đầy đủ có filter theo scope/nguồn/trạng thái/kỳ.

**Mục đích:** Giảm ma sát chọn đề cho mọi nhóm user — user mới (chưa biết trình độ) được mời vào thẳng 1 bài Full test; user có bài dở được nhắc tiếp tục; user rảnh muốn khám phá có lưới gợi ý ngẫu nhiên; user muốn tự chọn có lưới đầy đủ + filter.

**Phạm vi:** Trong phạm vi — thẻ trạng thái, lưới "Đề gợi ý cho bạn", tab bar, filter bar, lưới "Tất cả đề", card content model (tag + icon), banner Premium. Ngoài phạm vi — nội dung Câu hỏi tự chọn (#02), Mic check (#03), Phòng thi (#04), Report (#07/#08), thanh toán (chỉ điều hướng popup paywall).

---

## 1.2 Cấu trúc trang (sơ đồ khối)

```
┌───────────────────────────────────────────┐
│ 1. Thẻ trạng thái (đơn, theo thang ưu tiên)│
├───────────────────────────────────────────┤
│ 2. Banner Premium chung (chỉ free user)    │
├───────────────────────────────────────────┤
│ 3. "Đề gợi ý cho bạn" — 3 thẻ random,      │
│    trộn mọi scope, LUÔN hiển thị           │
├───────────────────────────────────────────┤
│ 4. Tab bar: Thi đầy đủ|Part1|Part2|Part3|  │
│    Đề thi custom  (5 tab cố định)          │
├───────────────────────────────────────────┤
│ 5. Filter bar: Tất cả|Forecast|Đã làm|     │
│    Chưa làm  +  dropdown chọn kỳ           │
├───────────────────────────────────────────┤
│ 6. "Tất cả đề" — lưới lọc theo tab+filter  │
└───────────────────────────────────────────┘
```

---

## 1.3 Card đề — Content Model

Áp dụng cho mọi card trong khối 3 ("Đề gợi ý cho bạn") và khối 6 ("Tất cả đề").

### Bố cục 1 card

```
┌─────────────────────────────┐
│ [Hot] [New]                 │  ← đỉnh card
│ 🄵  [Forecast] 02 · Full mock test │  ← icon loại đề + badge Forecast + meta rút gọn
│ Society                     │  ← title (đậm)
│ Chưa làm                    │  ← trạng thái làm bài
│                             │
│ [Premium]      [▶ Bắt đầu] │  ← góc dưới trái + CTA, ngang hàng
└─────────────────────────────┘
```

### Bảng thành phần

| Thành phần | Nội dung | Quy tắc |
|---|---|---|
| Icon loại đề | Badge chữ/số 1 màu trung tính (xám/navy nhạt) ở góc icon tài liệu: "F" (Full) / "1" / "2" / "3" | **Bắt buộc** vì "Đề gợi ý cho bạn" trộn lẫn mọi scope. Cố tình dùng 1 màu trung tính duy nhất cho cả 4 biến thể để không cạnh tranh thị giác với 4 tag Hot/New/Forecast/Premium (tránh bị hiểu nhầm là tag thứ 5) |
| Tag Hot | Badge, đỉnh card | Xem tiêu chí §1.3.2 |
| Tag New | Badge, đỉnh card, cạnh Hot | Xem tiêu chí §1.3.2 |
| Tag Forecast | Badge, ngay trước title | Thay thế chữ "Forecast" trong dòng meta cũ |
| Dòng meta | `[Số đề] · [Loại đề]` — vd "02 · Full mock test" | Rút gọn từ "Forecast 02 · Full mock test" vì đã có badge Forecast riêng |
| Title | Tên chủ đề (vd "Society") | = `catalog.title`, content team đặt tay khi soạn đề. Card Part 1/2/3 riêng **dùng chung title** với bản Full tương ứng |
| Trạng thái làm bài | "Chưa làm" / "Đang làm dở · Part X" / "Đã làm · [band] ([delta])" | vd "Đã làm · 7.0 (+0.5)" |
| Thời lượng | — | **Không hiển thị** trên card lưới (chỉ có ở thẻ trạng thái đầu trang) |
| Tag Premium | Badge, góc dưới trái, ngang hàng CTA | Không giới hạn hiển thị đồng thời với Hot/New/Forecast |
| CTA | "Bắt đầu" (luôn vậy, kể cả đề Premium) | Free user bấm vào đề Premium → popup paywall, **không** có nút "Mở khoá" riêng trên card |

### 1.3.2 Tiêu chí gắn tag

| Tag | Tiêu chí | Ghi chú |
|---|---|---|
| **New** | Đề thuộc kỳ/quý phát hành hiện tại — quý xác định **tự động theo ngày hệ thống** (dương lịch, Q1/Q2/Q3/Q4), không cần admin cấu hình thủ công | Dùng field kỳ phát hành đã thêm cho dropdown filter |
| **Hot** | Tự động — **top 3** đề có lượt "Bắt đầu" nhiều nhất trong 30 ngày gần nhất | Đã chốt N=3, phù hợp quy mô catalog hiện tại |
| **Forecast** | Gắn cho mọi đề thuộc nguồn Forecast | Hiện tại toàn bộ catalog Full/Part đều là nguồn Forecast |
| **Premium** | Theo field khoá gói hiện có trên đề | Không đổi so với logic cũ |

**Không giới hạn số tag hiển thị đồng thời** — 1 card có thể có cả 4 tag cùng lúc (Hot+New ở đỉnh, Forecast trước title, Premium góc dưới trái).

### 1.3.3 Card "Đề thi custom" (ngoại lệ)
Áp dụng icon Part mới (P1/P2/P3) để đồng bộ hình ảnh, nhưng **không** áp dụng tag New/Hot/Forecast/Premium (câu hỏi tự chọn không thuộc catalog, các tiêu chí trên không tính được). Format còn lại giữ nguyên như đã định nghĩa ở #02 (badge Part + tiêu đề + nội dung rút gọn).

---

## 1.4 Yêu cầu chức năng

| Mã | Yêu cầu | Đầu vào | Đầu ra |
|---|---|---|---|
| FR-01 | Hiển thị đúng 1 thẻ trạng thái theo thang ưu tiên mới (BR-01) | Trạng thái user | 1 thẻ trạng thái |
| FR-02 | Ưu tiên "Tiếp tục bài đang làm dở" nếu có bài dở, chỉ hiện Part đang dở | Cờ bài dở + Part hiện tại | Thẻ "Đang làm dở · Part X" |
| FR-03 | Khi free user hết lượt chấm, thẻ trạng thái vẫn hiển thị và hoạt động y hệt bình thường — **không** thêm badge/tín hiệu cảnh báo nào, không đổi CTA, không chặn thi | user=free, quota=out | Thẻ trạng thái không đổi so với khi còn lượt |
| FR-04 | Khi attempts=0, hiển thị thẻ "Kiểm tra trình độ để chọn đề phù hợp"; bấm CTA vào thẳng Full Mock Test (Mic check #03 → Phòng thi #04 examMode='full') dùng đề Forecast/khởi động | attempts=0 | Thẻ + điều hướng Full test |
| FR-05 | Khi attempts≥1 và không có bài dở, hiển thị thẻ "Sẵn sàng luyện tiếp" + "Trình độ hiện tại: Band X" (band gần nhất từ lịch sử) + CTA vào Full mock test band-matched. Engine chọn đề **chỉ cần band gần nhất**, không bắt buộc cần targetBand; nếu chưa có targetBand thì ẩn dòng copy "theo mục tiêu Band X", các phần còn lại của thẻ không đổi | band gần nhất (bắt buộc), targetBand (tuỳ chọn, chỉ ảnh hưởng copy) | Thẻ + điều hướng |
| FR-06 | Hiển thị banner Premium chung cho free user, dưới thẻ trạng thái | user=free | Banner |
| FR-07 | Hiển thị section "Đề gợi ý cho bạn": lưới 3 thẻ, random trong đề **chưa làm**, trộn mọi scope | Danh sách đề chưa làm (mọi scope) | 3 thẻ |
| FR-08 | Section "Đề gợi ý cho bạn" luôn hiển thị, kể cả khi có bài dở hoặc hết lượt | — | Không bị ẩn bởi BR-01 |
| FR-09 | Hiển thị tab bar dưới "Đề gợi ý cho bạn": Thi đầy đủ / Part 1 / Part 2 / Part 3 / Đề thi custom — luôn đủ 5 tab | — | Tab bar cố định |
| FR-10 | Hiển thị filter bar: pill "Tất cả" (exclusive) + "Forecast" + "Đã làm" + "Chưa làm", cộng 1 dropdown chọn kỳ phát hành | Lựa chọn filter | Lưới lọc tương ứng |
| FR-11 | Lưới "Tất cả đề" lọc theo tab + filter đang chọn | tab, filter | Lưới đề tương ứng |
| FR-12 | Mỗi card đề hiển thị đủ thành phần theo Card Content Model (§1.3) | Dữ liệu đề | Card đầy đủ |
| FR-13 | Card Part 1/2/3 riêng dùng chung title với bản Full tương ứng | catalog.title của đề gốc | Title đồng nhất |
| FR-14 | Card đề Premium giữ nguyên CTA "Bắt đầu"; free user bấm vào → popup paywall | user=free, đề=premium | Popup paywall |
| FR-15 | Card "Đề thi custom" áp dụng icon Part mới, không áp dụng tag catalog (New/Hot/Forecast/Premium) | — | Card custom đồng bộ icon |

---

## 1.5 Quy tắc nghiệp vụ (Business rules)

- **BR-01 (thang ưu tiên mới):** (1) bài đang làm dở → (2) attempts=0 → "Kiểm tra trình độ" → (3) attempts≥1, không bài dở → "Sẵn sàng luyện tiếp". **State "Chưa đặt mục tiêu" và state "Hết lượt — Nâng cấp" không còn tồn tại như 1 thẻ trạng thái riêng ở màn này.** Free user hết lượt chấm vẫn thi bình thường (không bị chặn ở màn Chọn đề lẫn Phòng thi) — chỉ bị khoá khi vào Report (đã có sẵn state "Hết lượt → chờ chấm thủ công" ở #08 trong bản v1.0).
- ~~**BR-01b (badge hết lượt)**~~ — **ĐÃ BỎ (2026-07-06).** Quyết định trước đó (thêm badge cảnh báo amber "Hết lượt chấm — vẫn thi được, xem report cần nâng cấp" vào CTA của thẻ trạng thái) đã bị **đảo ngược**. Chốt mới: khi free user hết lượt chấm, thẻ trạng thái hiển thị **y hệt bình thường** (dù là "Kiểm tra trình độ", "Sẵn sàng luyện tiếp", hay "Tiếp tục bài đang làm dở") — **không có bất kỳ badge, dòng chữ, hay tín hiệu thị giác nào khác biệt** ở màn Chọn đề. Việc "hết lượt chấm" chỉ thực sự lộ diện khi user vào tới Report (theo state "Hết lượt → chờ chấm thủ công" ở #08, ngoài phạm vi màn này).
- **BR-02 (giữ nguyên từ v1.0):** Engine C-lai cho thẻ trạng thái: attempts=0 → Forecast/đề khởi động; attempts≥1 → đề band-matched theo band gần nhất + tiêu chí yếu nhất.
- **BR-03:** "Kiểm tra trình độ" không phải tính năng/pipeline riêng — chỉ là copy cho state cold-start. Luồng sau khi bấm là Full Mock Test tiêu chuẩn (`examMode='full'`), tốn 1 lượt quota như một lượt thi Full bình thường.
- **BR-04:** targetBand không còn là điều kiện chặn ở màn Chọn đề. Được capture sau khi có kết quả bài thi Full đầu tiên, trong Report (#07 — xem addendum cuối file). Cho tới khi targetBand được đặt, engine ở FR-05 dùng band gần nhất từ lịch sử làm cơ sở duy nhất (không cần targetBand để chọn đề band-matched).
- **BR-05:** Section "Đề gợi ý cho bạn" độc lập hoàn toàn với thẻ trạng thái — luôn hiển thị, không bị chi phối bởi BR-01.
- **BR-06:** Thuật toán "Đề gợi ý cho bạn" — random đều trong tập đề (mọi scope) mà user chưa làm; không tính band, không tính tiêu chí yếu. Nếu không đủ 3 đề chưa làm (user đã làm gần hết catalog), **lấy bù bằng đề đã làm**, ưu tiên đề đã làm lâu nhất, để luôn đủ 3 thẻ và không lệch layout.
- **BR-06b (card "Đang làm dở" trong lưới, mới):** Trong lưới "Tất cả đề", đề đang làm dở hiển thị y như trạng thái "Chưa làm" bình thường — **không** có badge/copy đặc biệt trong lưới (thông tin "Đang làm dở" chỉ xuất hiện ở thẻ trạng thái đầu trang, tránh trùng lặp).
- **BR-07 (tag New):** Đề thuộc kỳ/quý phát hành = kỳ hiện tại của hệ thống, xác định **tự động theo ngày hệ thống** (dương lịch), không cần admin cấu hình mốc chuyển kỳ.
- **BR-08 (tag Hot):** Top 3 đề có lượt "Bắt đầu" nhiều nhất trong 30 ngày gần nhất.
- **BR-09 (tag Forecast):** Gắn cho mọi đề thuộc nguồn Forecast.
- **BR-10 (tag Premium):** Theo field khoá gói hiện có.
- **BR-11:** Không giới hạn số tag hiển thị đồng thời trên 1 card; vị trí cố định theo §1.3.
- **BR-12:** Title card = `catalog.title` do content team đặt tay; card Part riêng dùng chung title với bản Full.
- **BR-13 (cập nhật 2026-07-07):** `examMode` do scope xác định; resume đúng `examMode` đã lưu. **Session Part lẻ dùng chung schema session hiện tại** — chỉ khác ở `visibleParts`: Full exam có `visibleParts: ["part1","part2","part3"]`, Part lẻ có `visibleParts: ["partX"]` (1 phần tử). Ledger tính **1 lượt chấm cho 1 Part visible** (Part lẻ = 1 lượt, không phải 3 lượt như Full). Không tạo schema session mới — bỏ hardcode `["part1","part2","part3"]` tại `mock-exam-definition-from-catalog.ts:32` để cho phép truyền vào mảng tuỳ ý. CTA "Bắt đầu Part X" khi build xong sẽ tạo session với `visibleParts: ["partX"]` và redirect vào phòng thi với `examMode = "partX"`, không vào full. **Trạng thái hiện tại (chưa build Part lẻ):** CTA "Bắt đầu Part X" đang hiển thị nhưng dẫn vào full 3 Part — broken promise, sẽ được fix khi build Part lẻ (sau D1).

**Phân quyền theo vai trò:** giữ nguyên như v1.0 (User free: xem gợi ý, vào thi khi còn lượt, chạm đề Premium → paywall; User premium: không giới hạn, không thấy banner).

---

## 1.6 Xử lý ngoại lệ và trạng thái

| Trường hợp | Hành vi mong đợi |
|---|---|
| Đang tải catalog | Skeleton cho cả thẻ trạng thái, lưới gợi ý, lưới "Tất cả đề" |
| Lỗi tải catalog | Màn lỗi + "Thử lại" |
| "Tất cả đề" rỗng sau khi lọc | Empty state "Không có đề nào khớp bộ lọc" |
| "Đề gợi ý cho bạn" không đủ 3 đề chưa làm | Lấy bù bằng đề đã làm lâu nhất (BR-06), luôn giữ đủ 3 thẻ |
| User free hết lượt chấm | Vẫn vào thi bình thường ở mọi thẻ/card (không chặn); thẻ trạng thái không hiển thị gì khác biệt (không badge, không tín hiệu); khoá thực sự xảy ra ở màn Report (#07/#08), không phải ở đây |
| Free user bấm đề Premium | Popup paywall (FR-14) |
| Chưa đăng nhập | Điều hướng đăng nhập (ngoài phạm vi) |

---

## 1.7 Yêu cầu giao diện (UI/UX)
Card bo góc, nền tối/sáng theo theme hiện có của app. Tag Hot/New dùng badge nhỏ màu nổi bật (cam/đỏ) ở đỉnh card. Tag Forecast dùng badge màu trung tính (xanh dương nhạt) ngay trước title. Tag Premium dùng badge màu tím, góc dưới trái ngang hàng nút CTA. Icon loại đề (Full/Part1/2/3) là badge chữ/số 1 màu trung tính (xám/navy nhạt) ở góc icon tài liệu mặc định — cùng 1 màu cho cả 4 biến thể, chỉ đổi ký tự ("F"/"1"/"2"/"3").

---

## 1.8 Tiêu chí chấp nhận

**AC-01: Ưu tiên bài đang làm dở**
- **Given:** User có 1 bài Mock Test đang làm dở (chưa hoàn thành)
- **When:** User vào màn Chọn đề
- **Then:** Thẻ trạng thái hiển thị "Tiếp tục bài đang làm dở · Part X" AND không hiện % hoàn thành hay breakdown Part khác

**AC-02: Hết lượt chấm không đổi thẻ trạng thái**
- **Given:** User là free và đã hết lượt chấm điểm AI
- **When:** User vào màn Chọn đề
- **Then:** Thẻ trạng thái hiển thị đúng theo thang ưu tiên BR-01 (Kiểm tra trình độ / Sẵn sàng luyện tiếp / Đang làm dở), không kèm bất kỳ badge/tín hiệu cảnh báo nào AND CTA vẫn cho vào thi bình thường, không chặn

**AC-03: Cold-start (attempts=0)**
- **Given:** User chưa từng hoàn thành bài Full nào (attempts=0)
- **When:** User vào màn Chọn đề rồi bấm CTA trên thẻ trạng thái
- **Then:** Thẻ hiển thị "Kiểm tra trình độ để chọn đề phù hợp" AND CTA điều hướng thẳng vào Mic check rồi Phòng thi (`examMode='full'`) dùng đề Forecast/khởi động

**AC-04: Sẵn sàng luyện tiếp (attempts≥1, không có bài dở)**
- **Given:** User có attempts≥1 và không có bài đang làm dở
- **When:** User vào màn Chọn đề
- **Then:** Thẻ trạng thái hiển thị "Sẵn sàng luyện tiếp" + "Trình độ hiện tại: Band X" AND CTA vào Full mock test band-matched

**AC-05: "Đề gợi ý cho bạn" luôn hiển thị**
- **Given:** Bất kỳ trạng thái user nào (kể cả có bài dở hoặc hết lượt chấm)
- **When:** User vào màn Chọn đề
- **Then:** Section "Đề gợi ý cho bạn" luôn hiển thị 3 thẻ ngẫu nhiên trong đề chưa làm (mọi scope), không bị ẩn bởi trạng thái ưu tiên khác

**AC-06: Tab bar đủ 5 tab**
- **Given:** User ở màn Chọn đề, bất kỳ trạng thái nào
- **When:** Màn Chọn đề render
- **Then:** Tab bar hiển thị đủ 5 tab cố định (Thi đầy đủ/Part 1/Part 2/Part 3/Đề thi custom)

**AC-07: Filter pill loại trừ lẫn nhau**
- **Given:** User đang ở filter bar của lưới "Tất cả đề"
- **When:** User chọn pill "Tất cả" HOẶC chọn 1 pill khác (Forecast/Đã làm/Chưa làm)
- **Then:** Nếu chọn "Tất cả" thì mọi pill khác tự bỏ chọn; nếu chọn pill khác thì "Tất cả" tự bỏ chọn (exclusive)

**AC-08: Dropdown kỳ lọc đúng**
- **Given:** User đang xem lưới "Tất cả đề"
- **When:** User đổi lựa chọn ở dropdown kỳ phát hành
- **Then:** Lưới "Tất cả đề" lọc lại đúng theo kỳ vừa chọn

**AC-09: Icon + title đồng nhất Part/Full**
- **Given:** Card đề thuộc scope Part 1/2/3 (dẫn xuất từ đề Full)
- **When:** Card render trong lưới
- **Then:** Card hiển thị icon riêng biệt theo loại (badge "1"/"2"/"3") AND dùng chung title với bản Full tương ứng

**AC-10: Hiển thị đủ tag đồng thời**
- **Given:** Card đề thoả đồng thời nhiều tiêu chí tag (Hot/New/Forecast/Premium)
- **When:** Card render
- **Then:** Card hiển thị đủ các tag tương ứng, đúng vị trí quy định (không giới hạn số tag hiển thị đồng thời)

**AC-11: Card "Đã làm" hiển thị band + delta**
- **Given:** User đã làm đề đó ít nhất 1 lần
- **When:** Card render trạng thái làm bài
- **Then:** Card hiển thị "Đã làm · [band] ([delta])" đúng dữ liệu lịch sử

**AC-12: Card không hiện thời lượng**
- **Given:** Bất kỳ card đề nào trong lưới (khối "Đề gợi ý cho bạn" hoặc khối "Tất cả đề")
- **When:** Card render
- **Then:** Card không hiển thị thời lượng ước tính (thông tin này chỉ có ở thẻ trạng thái đầu trang)

**AC-13: Free user bấm đề Premium**
- **Given:** User là free, card đề có gắn tag Premium
- **When:** User bấm CTA "Bắt đầu" trên card đó
- **Then:** Hệ thống hiện popup paywall AND không điều hướng vào phòng thi

**AC-14: Card custom không gắn tag catalog**
- **Given:** Card thuộc tab "Đề thi custom"
- **When:** Card render
- **Then:** Card hiển thị icon Part tương ứng (P1/P2/P3) AND không hiển thị bất kỳ tag New/Hot/Forecast/Premium nào

**AC-15: Lấy bù khi thiếu đề gợi ý**
- **Given:** Số đề chưa làm của user (mọi scope) ít hơn 3
- **When:** Hệ thống dựng section "Đề gợi ý cho bạn"
- **Then:** Hệ thống lấy bù bằng đề đã làm lâu nhất, đảm bảo luôn hiển thị đủ 3 thẻ

**AC-16: Bài dở không có badge riêng trong lưới**
- **Given:** User có 1 đề đang làm dở
- **When:** Đề đó xuất hiện trong lưới "Tất cả đề" (không phải ở thẻ trạng thái)
- **Then:** Card hiển thị trạng thái "Chưa làm" như bình thường AND không có badge/copy đặc biệt nào cho biết đang làm dở

**AC-17: Chưa có targetBand vẫn band-matched**
- **Given:** User chưa từng đặt targetBand
- **When:** Thẻ trạng thái "Sẵn sàng luyện tiếp" được render (attempts≥1, không có bài dở)
- **Then:** Engine vẫn chọn đề band-matched theo band gần nhất từ lịch sử AND chỉ ẩn dòng copy "theo mục tiêu Band X" (phần còn lại của thẻ không đổi)

---

## Non-functional Requirements

**Performance:** card render <1s sau khi có catalog; chuyển tab/filter phản hồi <300ms (ưu tiên lọc trên dữ liệu client-side đã tải, không gọi lại API catalog mỗi lần đổi filter).

**Compatibility:** theo chuẩn chung hiện có của app (Chrome/Safari/Firefox bản mới, iOS/Android bản đang hỗ trợ) — không có yêu cầu riêng cho màn này.

**Accessibility:**
- Tag màu (Hot/New/Forecast/Premium) phải đạt tối thiểu WCAG 2.2 AA về độ tương phản.
- Mỗi tag phải đi kèm text, không dùng màu làm tín hiệu phân biệt duy nhất (user khiếm thị màu vẫn đọc được).
- Icon loại đề (Full/Part1/2/3) phải có alt-text/aria-label mô tả đúng loại đề tương ứng.

---

## User Types

> App yêu cầu đăng nhập mới truy cập được webapp, nên **không có user type "Visitor"** ở màn này.

| User Type | Definition | Feature Behavior |
|---|---|---|
| Free, attempts=0 | Free user chưa có lượt thi nào (chưa từng hoàn thành 1 bài Full) | Thấy thẻ trạng thái "Kiểm tra trình độ để chọn đề phù hợp"; thấy banner Premium; thấy đủ "Đề gợi ý cho bạn" + "Tất cả đề" |
| Free, attempts≥1, còn lượt chấm | Free user đã có lịch sử thi, còn lượt chấm | Thấy thẻ "Sẵn sàng luyện tiếp" + band gần nhất; không có badge cảnh báo |
| Free, đang làm dở | Free user có 1 bài chưa hoàn thành (bất kể còn/hết lượt chấm) | Thấy thẻ "Tiếp tục bài đang làm dở · Part X" — ưu tiên cao nhất (BR-01 #1) |
| Free, hết lượt chấm | Free user đã dùng hết lượt chấm điểm AI | **Vẫn vào thi bình thường** ở mọi thẻ/card, không bị chặn ở màn Chọn đề hay Phòng thi; thẻ trạng thái hiện đúng theo attempts (Kiểm tra trình độ / Sẵn sàng luyện tiếp / Đang làm dở) **không có gì khác biệt so với lúc còn lượt** (không badge, không tín hiệu); **chỉ bị khoá khi vào Report** (#07/#08) |
| Premium | Đã nâng cấp gói | Không giới hạn lượt chấm; không thấy banner Premium; card đề Premium không hiện popup paywall khi bấm |

---

## 1.9 Phần bổ sung

**Giả định:** Catalog có field kỳ phát hành (quarter) và nguồn đề (hiện chỉ có Forecast); có dữ liệu lượt "Bắt đầu" theo đề trong 30 ngày để tính Hot.

**Phụ thuộc:** Catalog API (cần bổ sung field kỳ phát hành) · Lịch sử thi/profile · Cơ chế quota · #02 (Custom Question) · #03/#04 (Mic/Phòng thi, dùng chung, không đổi) · #07 (Report — bổ sung banner chào mừng lần đầu, xem addendum).

## 1.10 Câu hỏi mở — đã chốt (2026-07-03, phiên grill-me thứ 4; cập nhật 2026-07-07, phiên grill-me thứ 2 về Part lẻ)

Toàn bộ câu hỏi mở đã được chốt và phản ánh trực tiếp vào các mục tương ứng trong file này:

| Câu hỏi | Đã chốt tại | Phiên |
|---|---|---|
| Hình dạng 4 icon Full/Part1/2/3 | Badge chữ/số 1 màu trung tính — §1.3 Bảng thành phần, §1.7 | 2026-07-03 |
| N cụ thể cho tag Hot | Top 3 — §1.3.2, BR-08 | 2026-07-03 |
| Card "Đang làm dở" trong lưới "Tất cả đề" | Không đặc biệt, hiện như "Chưa làm" — BR-06b, AC-16 | 2026-07-03 |
| Cách xác định "kỳ hiện tại" cho tag New | Tự động theo ngày hệ thống (dương lịch) — BR-07 | 2026-07-03 |
| UI capture targetBand sau bài Full đầu tiên | Banner inline ghép vào banner chào mừng, có nút "Bỏ qua" — xem Addendum §7 bên dưới | 2026-07-03 |
| Fallback khi thiếu đề gợi ý chưa làm | Lấy bù bằng đề đã làm lâu nhất — BR-06, AC-15 | 2026-07-03 |
| Engine có bắt buộc cần targetBand không | Không — chỉ cần band gần nhất, targetBand chỉ ảnh hưởng copy — FR-05, AC-17 | 2026-07-03 |
| Schema session Part lẻ có cần tạo mới không | Không — dùng chung schema hiện tại, chỉ thay `visibleParts` thành 1 phần tử; bỏ hardcode tại `mock-exam-definition-from-catalog.ts:32` — BR-13 | 2026-07-07 |
| Lượt chấm của Part lẻ là bao nhiêu | 1 lượt chấm cho 1 Part visible (không phải 3 lượt như Full) — BR-13 | 2026-07-07 |
| Hành vi CTA "Bắt đầu Part X" sau khi build | Tạo session `visibleParts: ["partX"]`, redirect phòng thi `examMode="partX"`, không vào full — BR-13 | 2026-07-07 |
| Thứ tự ưu tiên build Part lẻ | D1 → Part lẻ → B1/B2; Part lẻ được ưu tiên sau D1 vì CTA đang broken promise — Đổi log #14, #15 | 2026-07-07 |

Không còn câu hỏi mở nào cho §1 tại thời điểm này.

---

## Addendum — bổ sung vào §7 Report — Tổng quan & Accordion tiêu chí

**FR-07 (mới):** Khi đây là lần hoàn thành Full Mock Test đầu tiên của user (attempts trước đó = 0), Report phải hiển thị 1 banner chào mừng đặc biệt, khác với Report các lần sau. Banner này đồng thời là nơi capture targetBand lần đầu: hiển thị band vừa đạt được + 1 input/dropdown chọn band mục tiêu + nút "Bỏ qua". Banner **không chặn** việc xem report — user thấy điểm ngay, banner chỉ nằm ở đầu trang.

**FR-08 (mới):** Nếu user bấm "Bỏ qua" ở banner capture targetBand, hệ thống không lưu targetBand; hành vi engine ở #01 FR-05 áp dụng đúng theo BR-04/FR-05 (chỉ cần band gần nhất, không bắt buộc targetBand).

**AC-07 (mới): Banner chào mừng lần đầu**
- **Given:** User vừa hoàn thành Full Mock Test đầu tiên (attempts trước đó = 0)
- **When:** Report render
- **Then:** Hiển thị banner "🎉 Bài Full đầu tiên hoàn thành! Band hiện tại: [band vừa đạt] · Đặt mục tiêu: [input] [Bỏ qua]" AND Report các lần thi sau không hiển thị banner này

**AC-08 (mới): Lưu/bỏ qua targetBand không nhắc lại**
- **Given:** User đang xem banner capture targetBand lần đầu
- **When:** User nhập targetBand vào banner HOẶC bấm "Bỏ qua"
- **Then:** Nếu nhập, hệ thống lưu targetBand; nếu bấm "Bỏ qua", targetBand không được lưu — cả 2 trường hợp đều không hiển thị lại banner capture ở các lần thi sau (không nhắc lại nhiều lần)

**Câu hỏi mở mới cho #07:** Nội dung copy chính xác của banner (TBD, cần content); nếu user bấm "Bỏ qua" thì có cơ hội đặt targetBand ở nơi khác sau này không (vd trong Settings/Profile) — cần xác nhận có tồn tại đường dẫn thay thế hay không.
