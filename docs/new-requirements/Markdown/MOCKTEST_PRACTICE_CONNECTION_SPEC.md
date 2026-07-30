# MockTest ↔ Practice Connection Flow — Spec chính thức V1

> **Trạng thái:** Sẵn sàng triển khai — tất cả quyết định đã chốt (Discovery → Brainstorm → Design Critique → Conflict Resolution → Grill-me)  
> **Ngày:** 2026-07-25  
> **Input:** [`MOCKTEST_PRACTICE_CONNECTION_HANDOFF.md`](./MOCKTEST_PRACTICE_CONNECTION_HANDOFF.md) — toàn bộ decisions đã chốt, conflict resolutions, business rules  
> **Loại tài liệu:** Product requirement (hành vi & UI). Phần triển khai kỹ thuật do dev quyết định.

---

## Goal

**Business/Product Goal:** Đóng vòng lặp **Diagnose → Practice → Validate** — hiện tại 83.8% users không cross modules trong 7 ngày, average gap Practice→MockTest ~14.5h. Feature này xây các touchpoints kết nối để user thi xong biết phải luyện gì, luyện xong nhớ quay lại thi kiểm tra tiến bộ.

**User Benefits:**
- Thí sinh sau MockTest: thấy ngay criterion yếu nhất và biết bước tiếp theo (không phải tự đoán).
- Thí sinh vào Practice từ MockTest: thấy context tại sao mình đang luyện criterion này.
- Tất cả user: Performance page `/hieu-suat` chủ động gợi ý action phù hợp theo trạng thái hiện tại.
- User còn 0 lượt: upsell in-context ngay tại MockTest report, không bị redirect sang màn khác rồi bị chặn.

**Constraint kiến trúc bắt buộc:** MockTest và Practice KHÔNG được import lẫn nhau (ESLint enforced). Toàn bộ cross-module = URL navigation only: `?source=mock-report&criterion=[criterion]&sessionId=[id]`.

---

## Non-Goals (V1)

| Non-goal | Lý do |
|----------|-------|
| Deep-link đến criterion-filtered view trong Practice | BFF `/api/app/speaking/practice/catalog` chưa có `criterionTags` trên topics — blocker kỹ thuật |
| Dual-CTA Pronunciation trong MockTest report | Phụ thuộc criterion filter, deferred cùng với blocker trên |
| Pre-exam warm-up suggestion ở màn intro MockTest | Timing sai (peak anxiety), complexity cao, benefit không chắc chắn — DROPPED không queue V2 |
| Nudge dựa trên session count (BR-04 cũ) | Không có data backing, selection bias trong correlation data — DROPPED không queue V2 |
| Causation claim giữa Practice sessions và MockTest score | Không có causal backing — Before/After chỉ hiển thị delta thuần túy |

---

## Phạm vi V1 — 4 components

| Component | Touchpoint | Mô tả |
|-----------|-----------|--------|
| **A** | MockTest Report | Zone "Bước tiếp theo": Before/After delta card + Practice CTA + upsell khi 0 lượt |
| **B** | Practice | Context Banner khi đến từ MockTest report via URL params |
| **C** | Performance Page `/hieu-suat` | Action Card 4 states (A/C/D/B) theo time-based logic |
| **D** | MockTest Report — tab Phát âm | Mini-drill differentiation label |

---

## Business Rules tổng hợp

| BR | Touchpoint | Mô tả |
|----|-----------|--------|
| BR-01 | MockTest Report → Practice CTA | Sau report hoàn chỉnh, hiển thị CTA cho criterion yếu nhất |
| BR-02 | Practice CTA content | Label + sub-text lượt còn lại / Unlimited / Upsell khi 0 lượt |
| BR-03 | CTA deep-link | V1: Practice index `?source=mock-report&criterion=[criterion]&sessionId=[id]` |
| BR-05 | Quota display | Hiển thị lượt còn lại trong sub-text CTA |
| BR-06 | Performance page action card | 4 states A/C/D/B theo time-based logic, cooldown 7 ngày |
| BR-07 | Context banner trong Practice | Hiển thị khi có URL params từ MockTest report |

**Fallback order criterion yếu nhất** (dùng nhất quán ở mọi touchpoint):  
`Vocabulary → Grammar → Fluency → Pronunciation`

---

---

## Component A — MockTest Report: Zone "Bước tiếp theo"

### A.1 Tổng quan

**Mô tả ngắn:** Zone đặt sau accordion 4 tiêu chí (cuối section Tổng quan), trước Part selector. Gồm 2 phần: (1) Before/After delta card khi xem session mới nhất, và (2) Practice CTA dẫn đến luyện criterion yếu nhất.

**Mục đích:** Biến MockTest report từ điểm kết thúc thành điểm khởi đầu hành động tiếp theo — user không chỉ thấy band mà biết rõ phải làm gì tiếp theo.

**Điều kiện hiển thị toàn bộ zone:**
- Report phải **hoàn chỉnh**: band + vocab/grammar đầy đủ cho cả 4 tiêu chí.
- Zone **ẩn hoàn toàn** khi report partial (đang chấm, hoặc thiếu tiêu chí nào đó).

---

### A.2 Yêu cầu chức năng

| Mã | Yêu cầu | Ghi chú |
|----|---------|---------|
| FR-A01 | Xác định **criterion yếu nhất** theo band thấp nhất trong 4 tiêu chí | Khi tie: fallback cố định `Vocabulary → Grammar → Fluency → Pronunciation` |
| FR-A02 | Hiển thị **Before/After delta card** khi `isLatestSession = true` | Ẩn hoàn toàn khi user browse history |
| FR-A03 | Nội dung delta card: `"Lần trước: [X.X] → Lần này: [X.X] (+/−[Y])"` | Khi score giảm: thêm framing (xem §A.4) |
| FR-A04 | Hiển thị **Practice CTA** với label `"Luyện [Criterion] tại Practice →"` | Luôn đúng **1 CTA duy nhất** trong V1 |
| FR-A05 | Sub-text CTA: `"Band [X.X] · [N] lượt còn lại"` | Premium user: `"Band [X.X] · Unlimited"` |
| FR-A06 | Khi user còn **0 lượt**: CTA chuyển sang **upsell in-context** | Không navigate sang Practice rồi bị chặn |
| FR-A07 | Upsell CTA label: `"Nạp lượt để luyện [Criterion] →"` | Click mở billing flow ngay tại MockTest report |
| FR-A08 | Before/After card + Practice CTA **render đồng thời** khi user scroll đến cuối Tổng quan | Không timer, không event tracking |

**Naming criterion trong UI:**

| Criterion key | Tên hiển thị trong CTA |
|--------------|----------------------|
| `vocabulary` | Từ vựng |
| `grammar` | Ngữ pháp |
| `fluency` | Độ trôi chảy |
| `pronunciation` | Phát âm |

**Business rules:**
- **BR-A01:** Zone chỉ render khi `report.isComplete === true` (band + vocab/grammar đủ 4 tiêu chí).
- **BR-A02:** Màu accent cho zone và CTA dùng màu hệ thống hiện tại — **không dùng amber** (amber giữ nguyên cho warning/caveat). Icon cross-module: ↻ hoặc label `"Bước tiếp theo →"` nhất quán.
- **BR-A03:** `isLatestSession` check dựa trên so sánh `sessionId` với sessionId mới nhất trong history của user cho cùng examMode (full/part).
- **BR-A04:** Sub-text lượt còn lại đọc từ `remainingPracticeUses` trong NineSpeak profile (preloaded tại root app).

---

### A.3 Cơ chế hoạt động

**Luồng chính:**
1. Report hoàn chỉnh → zone render tự động.
2. `isLatestSession = true` → Before/After delta card xuất hiện trước CTA.
3. Criterion yếu nhất xác định → CTA hiển thị đúng criterion đó.
4. User click CTA (còn lượt) → navigate sang `/practice?source=mock-report&criterion=[criterion]&sessionId=[id]`.
5. User click CTA (0 lượt) → mở billing flow in-context.

**Luồng khi browse history:**
- `isLatestSession = false` → Before/After card ẩn, chỉ hiển thị Practice CTA.

**Dữ liệu cần:**
- `report.criteria[].band` — để xác định criterion yếu nhất
- `report.isComplete` — gate hiển thị zone
- `session.isLatestSession` — gate Before/After card
- `previousSession.overallBand` — cho delta card (chỉ cần khi `isLatestSession = true`)
- `profile.remainingPracticeUses` — cho sub-text CTA

---

### A.4 Nội dung & copy

**Before/After delta card:**
```
Lần trước: [X.X] → Lần này: [X.X] (+[Y])   ← score tăng
Lần trước: [X.X] → Lần này: [X.X] (−[Y])   ← score giảm
```

Khi score giảm (`delta < 0`), thêm framing dưới card:
```
"Band có thể dao động giữa các lần thi — tiếp tục luyện để ổn định kết quả."
```

**Practice CTA (còn lượt):**
```
[Label]    "Luyện Từ vựng tại Practice →"
[Sub-text] "Band 6.0 · 3 lượt còn lại"        ← user thường
[Sub-text] "Band 6.0 · Unlimited"              ← premium user
```

**Practice CTA (0 lượt — upsell):**
```
[Label]    "Nạp lượt để luyện Từ vựng →"
```

---

### A.5 Xử lý ngoại lệ

| Trường hợp | Hành vi |
|-----------|---------|
| Report partial (đang chấm hoặc thiếu tiêu chí) | Zone ẩn hoàn toàn |
| Không có session trước (lần thi đầu tiên) | Before/After card ẩn (dù `isLatestSession = true`, không có `previousSession` để so sánh) |
| Tie giữa 2+ tiêu chí cùng band thấp nhất | Dùng fallback order: Vocabulary → Grammar → Fluency → Pronunciation |
| `remainingPracticeUses` chưa load | Sub-text hiển thị `"Band [X.X]"` (không hiển thị lượt cho đến khi data có) |

---

### A.6 Tiêu chí chấp nhận

**AC-A01: Zone ẩn khi report partial**
- **Given:** Report chưa hoàn chỉnh (đang chấm hoặc thiếu tiêu chí)
- **When:** User mở MockTest report
- **Then:** Zone "Bước tiếp theo" không hiển thị

**AC-A02: Zone render sau accordion 4 tiêu chí**
- **Given:** Report hoàn chỉnh (`isComplete = true`)
- **When:** User scroll đến cuối section Tổng quan
- **Then:** Zone "Bước tiếp theo" render sau accordion tiêu chí AND trước Part selector

**AC-A03: Criterion yếu nhất xác định đúng**
- **Given:** Report với 4 tiêu chí có band khác nhau
- **When:** Zone render
- **Then:** CTA hiển thị đúng tên criterion có band thấp nhất; khi tie, dùng đúng fallback order Vocabulary → Grammar → Fluency → Pronunciation

**AC-A04: Before/After card chỉ xuất hiện với session mới nhất**
- **Given:** Report hoàn chỉnh
- **When:** User xem session mới nhất (`isLatestSession = true`)
- **Then:** Before/After card hiển thị với delta so với session trước AND Practice CTA hiển thị phía dưới
- **When:** User browse session cũ (`isLatestSession = false`)
- **Then:** Before/After card không hiển thị; chỉ Practice CTA hiển thị

**AC-A05: Delta card hiển thị đúng và framing khi giảm**
- **Given:** User đang xem session mới nhất, có session trước để so sánh
- **When:** Delta card render
- **Then:** Hiển thị `"Lần trước: [X.X] → Lần này: [X.X] (+/−[Y])"` với giá trị chính xác; khi `delta < 0`, framing `"Band có thể dao động..."` xuất hiện phía dưới card

**AC-A06: Practice CTA đúng label và sub-text**
- **Given:** Report hoàn chỉnh, user còn lượt Practice (`remainingPracticeUses > 0`)
- **When:** Zone render
- **Then:** Label `"Luyện [Criterion] tại Practice →"` với tên criterion đúng; sub-text `"Band [X.X] · [N] lượt còn lại"` (user thường) hoặc `"Band [X.X] · Unlimited"` (premium)

**AC-A07: CTA navigate đúng URL khi click**
- **Given:** User còn lượt, click Practice CTA
- **When:** Click
- **Then:** Navigate sang `/practice?source=mock-report&criterion=[criterion]&sessionId=[id]` với đúng giá trị criterion và sessionId

**AC-A08: Upsell khi 0 lượt**
- **Given:** `remainingPracticeUses = 0`
- **When:** Zone render
- **Then:** CTA label đổi thành `"Nạp lượt để luyện [Criterion] →"`; click mở billing flow in-context (không navigate sang Practice)

**AC-A09: Luôn đúng 1 CTA**
- **Given:** Bất kỳ trạng thái nào của report
- **When:** Zone render
- **Then:** Luôn hiển thị đúng 1 CTA duy nhất (không phải 2 CTA dù criterion yếu nhất là Pronunciation)

---

---

## Component B — Practice: Context Banner

### B.1 Tổng quan

**Mô tả ngắn:** Banner hiển thị ở đầu Practice khi user đến từ MockTest report (URL params present), cho user biết context tại sao họ đang luyện criterion này, và cung cấp link quay lại báo cáo.

**Constraint kiến trúc:** Banner đọc URL params khi mount, clear params ngay bằng `router.replace` — không giữ params trong URL sau lần render đầu.

---

### B.2 Yêu cầu chức năng

| Mã | Yêu cầu | Ghi chú |
|----|---------|---------|
| FR-B01 | Hiển thị banner khi URL có `source=mock-report` | Đọc params khi mount |
| FR-B02 | Nội dung banner: `"Đang luyện theo gợi ý từ phiên thi [ngày] · Xem lại báo cáo ›"` | `sessionId` dùng để lookup ngày thi, link quay lại report |
| FR-B03 | Sau khi banner render lần đầu: **xóa URL params** bằng `router.replace` | Không tạo history entry mới |
| FR-B04 | Banner **biến mất** khi user navigate sang câu hỏi khác | Không persist khi chuyển câu |
| FR-B05 | Banner **không re-appear** khi dùng browser back | Params đã clear, không reload |
| FR-B06 | Click `"Xem lại báo cáo ›"`: navigate sang MockTest report của `sessionId` đó | Deep-link sang MockTest report |

**Business rules:**
- **BR-B01:** `sessionId` trong URL params là của chính user đang đăng nhập — không cần validate chéo vì MockTest và Practice cùng account. Shared link (ai đó share URL) sẽ không có `sessionId` match với account khác → banner không xuất hiện (graceful fallback).
- **BR-B02:** Nội dung banner không expose thông tin nhạy cảm của user khác — banner chỉ xuất hiện khi `sessionId` thuộc về user hiện tại.

---

### B.3 Cơ chế hoạt động

**Lifecycle:**
1. User click CTA từ MockTest report → navigate `/practice?source=mock-report&criterion=vocabulary&sessionId=xxx`.
2. Practice mount → đọc URL params → lưu vào local state (`bannerData = { criterion, sessionId, date }`).
3. `router.replace('/practice')` — xóa params khỏi URL (không tạo history entry mới).
4. Banner render với `bannerData`.
5. User navigate sang câu hỏi khác → banner biến mất (unmount hoặc reset local state).
6. User dùng browser back → URL là `/practice` (params đã clear) → không re-trigger banner.

---

### B.4 Xử lý ngoại lệ

| Trường hợp | Hành vi |
|-----------|---------|
| `sessionId` không tồn tại hoặc không thuộc user hiện tại | Banner không xuất hiện (skip gracefully, không error) |
| `source=mock-report` có nhưng thiếu `sessionId` | Banner không xuất hiện |
| Shared link với `sessionId` của người khác | Banner không xuất hiện (lookup fail → skip) |

---

### B.5 Tiêu chí chấp nhận

**AC-B01: Banner xuất hiện khi đến từ MockTest**
- **Given:** User click Practice CTA trong MockTest report
- **When:** Practice page mount
- **Then:** Banner `"Đang luyện theo gợi ý từ phiên thi [ngày] · Xem lại báo cáo ›"` xuất hiện ở đầu trang

**AC-B02: URL params bị clear sau render đầu**
- **Given:** Practice page vừa mount với URL params `?source=mock-report&...`
- **When:** Sau lần render đầu
- **Then:** URL đổi thành `/practice` (không còn params) AND banner vẫn hiển thị (đọc từ local state)

**AC-B03: Không tạo history entry mới khi clear params**
- **Given:** Params vừa được clear bằng `router.replace`
- **When:** User nhấn browser back
- **Then:** Không quay về `/practice?source=mock-report...` (vì replace không tạo entry mới)

**AC-B04: Banner biến mất khi navigate câu khác**
- **Given:** Banner đang hiển thị
- **When:** User chọn câu hỏi khác trong Practice
- **Then:** Banner không còn hiển thị

**AC-B05: Banner không re-appear sau browser back**
- **Given:** User đã vào Practice từ MockTest, banner đã hiển thị, params đã clear
- **When:** User dùng browser back (quay về MockTest report) rồi forward lại Practice
- **Then:** Banner không xuất hiện lần thứ hai

**AC-B06: Link "Xem lại báo cáo" navigate đúng**
- **Given:** Banner đang hiển thị với sessionId hợp lệ
- **When:** User click `"Xem lại báo cáo ›"`
- **Then:** Navigate sang MockTest report của đúng session đó

**AC-B07: Banner không xuất hiện với sessionId người khác**
- **Given:** URL chứa `sessionId` không thuộc user hiện tại
- **When:** Practice page mount
- **Then:** Banner không xuất hiện (không error, không crash)

---

---

## Component C — Performance Page `/hieu-suat`: Action Card

### C.1 Tổng quan

**Mô tả ngắn:** Một action card trên Performance page, hiển thị gợi ý action phù hợp nhất theo trạng thái MockTest hiện tại của user. Luôn hiển thị đúng 1 card (hoặc không hiển thị gì nếu không match state nào).

**Mục đích:** Performance page hiện là passive display — card này biến nó thành điểm khởi đầu hành động dựa trên data.

**Data source:** `history-client` từ `features/shared/` (đã có, Performance page đang dùng).

---

### C.2 Logic 4 states

| State | Priority | Điều kiện | Destination |
|-------|----------|-----------|-------------|
| **A** | 1 (cao nhất) | Chưa có MockTest nào | MockTest |
| **C** | 2 | MockTest cuối: 7–30 ngày trước | MockTest |
| **D** | 3 | MockTest cuối: >30 ngày trước | MockTest |
| **B** | 4 | MockTest cuối: trong 7 ngày qua (cool-down) | Practice |
| **Default** | — | Không match state nào | Không hiển thị card |

**Hiển thị đúng 1 card theo priority A > C > D > B.**

**Cool-down = 7 ngày** (không phải 14 ngày — chốt sau conflict resolution để align với State B).

**State C: time-based only** — không có điều kiện session count (DROPPED vì không có data backing).

---

### C.3 Copy từng state

**State A — Chưa có MockTest:**
```
"Thi thử để biết band thật của bạn →"
```

**State B — Cool-down (MockTest trong 7 ngày qua):**
```
"Tiếp tục luyện [Criterion yếu nhất] →"
```
*(Không nudge "thi lại" — user vừa thi, đang trong cool-down)*

**State C — MockTest từ 7–30 ngày trước:**
```
"Đã [X] ngày kể từ lần thi — thử lại để cập nhật band →"
```

**State D — MockTest từ >30 ngày trước:**
```
"Đã lâu không kiểm tra — thi thử để cập nhật band →"
```

---

### C.4 Yêu cầu chức năng

| Mã | Yêu cầu | Ghi chú |
|----|---------|---------|
| FR-C01 | Xác định state theo thời gian MockTest cuối cùng của user | Đọc từ history-client (shared) |
| FR-C02 | Hiển thị đúng **1 card** theo priority A > C > D > B | Không hiển thị 2 card cùng lúc |
| FR-C03 | State B: criterion yếu nhất dùng cùng fallback order với Component A | Vocabulary → Grammar → Fluency → Pronunciation |
| FR-C04 | State B click: navigate sang Practice (không phải MockTest) | `?source=performance-page` (optional, tracking) |
| FR-C05 | State A/C/D click: navigate sang MockTest | |
| FR-C06 | Không hiển thị card khi không match state nào (edge case) | Không render element placeholder |

**Business rules:**
- **BR-C01:** State B KHÔNG hiển thị "thi lại" — tránh mâu thuẫn với user vừa mới thi.
- **BR-C02:** Cooldown 7 ngày đồng bộ với state B logic (từ ngày thi, không phải ngày xem report).
- **BR-C03:** Tính số ngày: `Math.floor((now - lastMockTestDate) / (1000 * 60 * 60 * 24))`.
- **BR-C04:** "Criterion yếu nhất" cho State B đọc từ report của MockTest gần nhất.

---

### C.5 Xử lý ngoại lệ

| Trường hợp | Hành vi |
|-----------|---------|
| History-client chưa load | Card không hiển thị (không hiện skeleton cho card này) |
| MockTest gần nhất ở đúng ranh giới 7 ngày | Dùng `>=7 ngày → State C` (inclusive) |
| Không có MockTest nào trong history | State A |
| Không có Practice report để xác định criterion yếu nhất (State B) | Hiển thị `"Tiếp tục luyện →"` không có criterion cụ thể — hoặc dùng "Vocabulary" làm default |

---

### C.6 Tiêu chí chấp nhận

**AC-C01: State A — chưa có MockTest**
- **Given:** User chưa có MockTest nào trong history
- **When:** Performance page load
- **Then:** Card State A hiển thị với copy `"Thi thử để biết band thật của bạn →"`; click navigate sang MockTest

**AC-C02: State B — cool-down (trong 7 ngày)**
- **Given:** MockTest gần nhất < 7 ngày trước
- **When:** Performance page load
- **Then:** Card State B hiển thị với copy `"Tiếp tục luyện [Criterion] →"` (không có text "thi lại"); click navigate sang Practice

**AC-C03: State C — 7–30 ngày**
- **Given:** MockTest gần nhất 7–30 ngày trước
- **When:** Performance page load
- **Then:** Card State C hiển thị với copy `"Đã [X] ngày kể từ lần thi — thử lại để cập nhật band →"` với số ngày chính xác; click navigate sang MockTest

**AC-C04: State D — >30 ngày**
- **Given:** MockTest gần nhất >30 ngày trước
- **When:** Performance page load
- **Then:** Card State D hiển thị với copy `"Đã lâu không kiểm tra — thi thử để cập nhật band →"`; click navigate sang MockTest

**AC-C05: Priority đúng khi overlap**
- **Given:** User có MockTest 5 ngày trước (match State B)
- **When:** Performance page load
- **Then:** Hiển thị State B (không phải State C/D), vì priority B > C/D về thời gian gần đây

**AC-C06: Không hiển thị 2 card cùng lúc**
- **Given:** Bất kỳ trạng thái nào
- **When:** Performance page load
- **Then:** Luôn hiển thị đúng 1 card (hoặc không card nào nếu default)

**AC-C07: State B dùng criterion yếu nhất với đúng fallback**
- **Given:** MockTest gần nhất có tie giữa Vocabulary và Grammar (cùng band thấp nhất)
- **When:** State B card render
- **Then:** Hiển thị `"Tiếp tục luyện Từ vựng →"` (Vocabulary trước Grammar theo fallback order)

---

---

## Component D — Mini-drill: Differentiation Label

### D.1 Tổng quan

**Mô tả ngắn:** Thêm label phụ cho nút `"Luyện ›"` trong tab Phát âm của MockTest report, để phân biệt rõ với Practice CTA ở Component A.

**Vấn đề giải quyết:** Cả hai đều là "luyện" — user không rõ hai nút này khác gì nhau.

**Giải pháp:** Label phụ + tách biệt vật lý (in-tab vs. cuối Tổng quan).

---

### D.2 Yêu cầu chức năng

| Mã | Yêu cầu |
|----|---------|
| FR-D01 | Nút `"Luyện ›"` trong tab Phát âm thêm label phụ `"(Miễn phí · 1 từ)"` |
| FR-D02 | Không thay đổi hành vi/destination của nút `"Luyện ›"` (giữ nguyên logic mini-drill) |
| FR-D03 | Không thêm label cho nút `"Luyện ›"` trong các tab khác (chỉ tab Phát âm) |

---

### D.3 Tiêu chí chấp nhận

**AC-D01: Label phụ hiển thị đúng chỗ**
- **Given:** User mở tab Phát âm trong MockTest report
- **When:** Tab render
- **Then:** Nút `"Luyện ›"` hiển thị kèm label phụ `"(Miễn phí · 1 từ)"`

**AC-D02: Hành vi nút không thay đổi**
- **Given:** Label phụ đã thêm
- **When:** User click nút `"Luyện ›"`
- **Then:** Hành vi giống hệt trước khi thêm label (mini-drill flow giữ nguyên)

**AC-D03: Các tab khác không bị ảnh hưởng**
- **Given:** User mở tab Từ vựng hoặc Ngữ pháp trong MockTest report
- **When:** Tab render
- **Then:** Nút `"Luyện ›"` (nếu có) không có label phụ `"(Miễn phí · 1 từ)"`

---

---

## Conflict Resolution Summary

| Conflict | Vấn đề | Giải pháp chốt |
|----------|--------|---------------|
| Mini-drill vs. Practice CTA | Cả hai đều là "luyện", user không rõ khác gì | Mini-drill: thêm label `"(Miễn phí · 1 từ)"`; Practice CTA: ở zone riêng cuối Tổng quan. Tách biệt vật lý. |
| Performance page cool-down vs. MockTest report | Cool-down 14 ngày → mâu thuẫn khi user thi lại sau 10 ngày | Cool-down = **7 ngày** (align với State B). State B không nudge "thi lại". |
| Pre-exam warm-up | Màn intro MockTest = peak anxiety, timing sai | **DROPPED** hoàn toàn. Alternative tùy chọn: text tĩnh "Hít thở sâu, tự tin bắt đầu." trong intro nếu design cần. |

---

## V2 Backlog

| Item | Blocker | Scope |
|------|---------|-------|
| Deep-link criterion-filtered view trong Practice | BFF `/api/app/speaking/practice/catalog` cần có `criterionTags` trên topics | FR-B03 upgrade |
| Dual-CTA Pronunciation trong MockTest report | Phụ thuộc criterion filter V2 | Component A |
| Mixpanel instrumentation: `attempt_number`, `score_delta` | Cần instrument `mock_scoring_completed` event | Data layer |

---

## Success Metrics

**Leading indicators (tuần đầu sau launch):**

| Metric | Mô tả | Target |
|--------|-------|--------|
| CTA click-through rate | % users xem report hoàn chỉnh click Practice CTA | ≥20% (baseline: ~0%) |
| Cross-module rate | % users có cả MockTest lẫn Practice trong 7 ngày | Tăng từ 16.2% → ≥25% |
| Upsell engagement | % users 0 lượt click upsell CTA | ≥10% |

**Lagging indicators (1 tháng sau launch):**

| Metric | Mô tả | Target |
|--------|-------|--------|
| Time gap Practice → MockTest | Average gap giảm | Từ ~14.5h → <12h |
| MockTest repeat rate | % users thi lại trong 30 ngày | Tăng ≥5 percentage points |
| Performance page engagement | % users click Action Card | ≥30% của users vào `/hieu-suat` |

---

## Files liên quan

| File | Mục đích |
|------|---------|
| [`docs/new-requirements/Markdown/MOCKTEST_PRACTICE_CONNECTION_HANDOFF.md`](./MOCKTEST_PRACTICE_CONNECTION_HANDOFF.md) | Input decisions — handoff từ sessions discovery/grill-me |
| [`docs/modules/mock-test/screens/07-report-tong-quan.md`](../../modules/mock-test/screens/07-report-tong-quan.md) | Tổng quan section spec — Component A đặt sau accordion ở đây |
| [`docs/modules/mock-test/screens/08-report-phan-hoi-chi-tiet.md`](../../modules/mock-test/screens/08-report-phan-hoi-chi-tiet.md) | Part selector, criterion tabs, mini-drill — Component D ở đây |
| [`docs/modules/practice/PRD.md`](../../modules/practice/PRD.md) | Practice module scope — Component B |
| [`docs/modules/performance/PRD.md`](../../modules/performance/PRD.md) | Performance page PRD — Component C |
| [`docs/new-requirements/Markdown/PRACTICE_OPTIMIZE_FULL_SPEC.md`](./PRACTICE_OPTIMIZE_FULL_SPEC.md) | Practice optimize spec (layout, criterion cards, coach answer) |

---

## Open Questions

| # | Câu hỏi | Owner | Blocking? |
|---|---------|-------|-----------|
| OQ-01 | `remainingPracticeUses = 0` nhưng user đang mid-session trong Practice — upsell in-context có bị trigger mid-flow không? | Engineering | No (edge case) |
| OQ-02 | Ngày thi trong banner Practice (`"phiên thi [ngày]"`) — format ngày: "25/7" hay "25 tháng 7" hay "hôm qua"? | Design | No |
| OQ-03 | State B criterion yếu nhất: nếu MockTest gần nhất không có đủ 4 tiêu chí (partial report), lấy criterion từ đâu? | Engineering | No (dùng Vocabulary default) |
| OQ-04 | Performance page Action Card: vị trí trong layout — trên hay dưới chart hiệu suất tổng hợp? | Design | Yes (trước khi bắt đầu build) |
