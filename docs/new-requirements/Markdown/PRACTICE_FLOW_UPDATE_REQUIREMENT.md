# Yêu cầu cập nhật — Practice Flow (Luyện nói)

> **Mục tiêu:** Tối ưu trải nghiệm phần Practice: kết quả chấm điểm dễ đọc hơn, gom phân tích về một chỗ, bổ sung phản hồi phát âm chi tiết, xử lý edge case khi user nói, và thể hiện tiến bộ qua các lượt.
>
> **Loại tài liệu:** Product requirement (mô tả hành vi & giao diện). Phần kỹ thuật do dev quyết định khi map vào codebase.
> **Bản mẫu giao diện (visual spec):** [`practice-flow-optimized.html`](./practice-flow-optimized.html) — mở bằng trình duyệt. File có sẵn ô **"Demo edge case"** để QA kích hoạt từng tình huống.
> **Phạm vi:** áp dụng cho **Part 1, Part 2, Part 3** (lưu ý khác biệt Part 2 ở §4).
> **Ngày tạo:** 2026-06-16.

---

## 0. Quy ước

- **Ưu tiên:** `P0` = bắt buộc (đợt 1) · `P1` = nên có (đợt 2) · `P2` = bổ sung sau.
- "Lượt" = một lần user ghi âm & được chấm cho cùng một câu hỏi.
- "Khu kết quả" = panel chấm điểm hiển thị sau mỗi lượt nói (bên khu chat).
- "Cột AI hỗ trợ" = cột bên phải (sample/từ vựng/ghi chú + nội dung phân tích).

---

## 1. Tổng quan thay đổi (Before → After)

| # | Hạng mục | Hiện tại | Sau cập nhật | Ưu tiên |
|---|---|---|---|---|
| A | Phân tích bài nói | 3 tab Từ vựng / Ngữ pháp / Phát âm | Bỏ tab → hiển thị **"Câu trả lời đã sửa lỗi"** ngay trong kết quả | P0 |
| B | Xem chi tiết | Nút "Xem chấm điểm chi tiết cho câu" (mở modal) | **Bỏ** | P0 |
| C | Phân tích theo tiêu chí | Nằm trong tab | **Bấm vào thẻ điểm** → phân tích chi tiết mở ở **cột AI hỗ trợ** | P1 |
| D | Phản hồi phát âm | Highlight cơ bản trong tab | **Transcript chấm màu + IPA + audio + luyện âm từng từ** | P1 |
| E | Gợi ý câu hay hơn | "Cải thiện câu phù hợp với band 7.0" | **"Gợi ý câu trả lời tốt hơn"** (bỏ số band) | P1 |
| F | So sánh giữa các lượt | Không có | **Delta điểm + lời động viên + biểu đồ tiến bộ** | P1 |
| G | Nhiều lượt | Tất cả mở rộng, transcript lặp lại | **Lượt mới mở, lượt cũ tự thu gọn**; bỏ bong bóng transcript thừa | P1 |
| H | Edge case khi nói | Banner lỗi chung chung | **Card lỗi chuyên biệt** theo từng tình huống | P0–P1 |
| I | Kích thước & cỡ chữ | Khá to/thoáng | **Gọn lại** (mật độ thông tin cao, vẫn dễ đọc) | P1 |

---

## 2. Yêu cầu chi tiết

### 2.1. Khu kết quả chấm điểm — bố cục mới `P0`

**Mô tả:** Sau khi user nói xong, khu kết quả hiển thị theo thứ tự từ trên xuống:
1. **Điểm tổng (overall band)** + biểu tượng "AI Chấm điểm".
2. **4 thẻ điểm thành phần**: Trôi chảy & Mạch lạc · Từ vựng · Phát âm · Ngữ pháp (mỗi thẻ một tông màu).
3. **"Câu trả lời của bạn — đã sửa lỗi"** (xem §2.2).
4. Nút **"Gợi ý câu trả lời tốt hơn"** (xem §2.5).

**Bỏ đi:**
- 3 tab Từ vựng / Ngữ pháp / Phát âm.
- Nút "Xem chấm điểm chi tiết cho câu" và modal chi tiết.

**Nghiệm thu:**
- [ ] Không còn tab và nút "xem chi tiết" trong khu kết quả.
- [ ] Điểm tổng + 4 tiêu chí hiển thị ngay khi có kết quả chấm.
- [ ] Bố cục đúng thứ tự trên.

### 2.2. "Câu trả lời đã sửa lỗi" (track-changes) `P0`

**Mô tả:** Hiển thị transcript của user với lỗi **từ vựng & ngữ pháp** được đánh dấu trực tiếp trên câu:
- Phần sai: **gạch bỏ** (màu đỏ).
- Bản sửa: hiển thị ngay cạnh (màu xanh).
- **Bấm vào chỗ sửa** → hiện **popup giải thích lỗi**: loại lỗi (Từ vựng / Ngữ pháp), `lỗi → bản sửa`, và giải thích ngắn vì sao.
- Có chú giải: "gạch bỏ = lỗi · thay thế = bản sửa · bấm vào chỗ sửa để xem giải thích".

**Trạng thái chờ:** điểm tổng + 4 tiêu chí hiện trước; phần "câu đã sửa" hiển thị skeleton + "AI đang phân tích & sửa lỗi…" cho đến khi có dữ liệu.

**Nghiệm thu:**
- [ ] Lỗi từ vựng/ngữ pháp hiển thị dạng gạch bỏ + bản sửa inline.
- [ ] Bấm vào mỗi chỗ sửa hiện đúng popup giải thích; bấm ra ngoài thì đóng.
- [ ] Khi chưa có dữ liệu phân tích: hiện skeleton, không vỡ layout.

### 2.3. Phân tích theo tiêu chí — mở ở cột AI hỗ trợ `P1`

**Mô tả:**
- **4 thẻ điểm thành phần là nút bấm**. Rê chuột hiện tooltip "Xem phân tích · [tiêu chí]".
- Bấm 1 thẻ → **chèn thêm** một card phân tích chi tiết của tiêu chí đó vào **cột AI hỗ trợ** (cuộn chung với câu mẫu / từ vựng / ghi chú). Card có nút đóng (✕).
- Có dòng gợi ý dưới khối điểm: "Bấm vào từng tiêu chí để xem phân tích chi tiết ở cột AI hỗ trợ".

**Nghiệm thu:**
- [ ] Bấm thẻ → card phân tích tương ứng xuất hiện ở cột AI hỗ trợ.
- [ ] Mở nhiều tiêu chí → các card cùng tồn tại, đóng được từng card.
- [ ] Không còn cụm nút "Xem phân tích" riêng (đã gộp vào thẻ điểm).

### 2.4. Phản hồi phát âm chi tiết `P1` / luyện âm từng từ `P2`

**A. Card phân tích Phát âm (mở từ thẻ "Phát âm"):**
- **Transcript chấm màu theo 3 mức lỗi:**
  - **Lỗi phụ** — gạch chân chấm (xám).
  - **Lỗi nhẹ** — tô màu vàng ở âm/chữ sai.
  - **Lỗi nặng** — chữ đỏ + hiển thị **phiên âm IPA** ngay cạnh.
- **Audio player**: nghe lại bản ghi của user (play + thanh tiến trình + thời gian).
- **Chú giải** 3 mức lỗi.

**B. Panel luyện âm từng từ (`P2` — bấm vào 1 từ trong transcript):**
- Hiện từ + nút nghe + IPA + nghĩa tiếng Việt.
- **So sánh "MẪU" vs "BẠN"** theo từng âm (phoneme): âm đúng (xanh) / âm sai (vàng).
- **Hướng dẫn luyện âm** cho âm sai (cách đặt lưỡi/môi, ví dụ minh hoạ) + "xem thêm".
- Nút **"Ghi âm ngay"** để luyện lại từ đó.

**Nghiệm thu:**
- [ ] Transcript phát âm phân biệt rõ 3 mức lỗi + IPA cho lỗi nặng.
- [ ] Nghe lại được bản ghi.
- [ ] (P2) Bấm 1 từ → panel luyện âm hiện đúng so sánh MẪU/BẠN + hướng dẫn.

### 2.5. "Gợi ý câu trả lời tốt hơn" (rewrite) `P1`

**Mô tả:**
- Nút trong khu kết quả: **"Gợi ý câu trả lời tốt hơn"** (KHÔNG nêu số band cụ thể).
- Bấm → tạo gợi ý, **kết quả hiển thị ở cột AI hỗ trợ** (đang tạo → hiện loading).
- Trong card kết quả có thêm nút **"Mở rộng câu"** → tạo bản dài hơn, kèm **dịch nghĩa tiếng Việt**.
- Phụ đề card dùng câu chung chung (vd "Gợi ý cách diễn đạt tốt hơn cho câu trả lời của bạn"), không nêu số band.

**Nghiệm thu:**
- [ ] Không còn chữ "phù hợp với band X.X" ở nút và card.
- [ ] Kết quả gợi ý + mở rộng hiển thị ở cột AI hỗ trợ.
- [ ] Có trạng thái đang tạo (loading).

### 2.6. So sánh điểm qua các lượt `P1`

**Mô tả:** So sánh với **lượt liền trước** (Lần N so Lần N−1). Từ lượt 2 trở đi:
- **Band tổng**: chip delta cạnh điểm — 🟢 `▲ +0.5` (tăng) / 🔴 `▼ −0.5` (giảm) / ⚪ `– 0` (không đổi).
- **Mỗi thẻ tiêu chí**: delta nhỏ (▲ / ▼ / –).
- **Dòng động viên** (ngay dưới khối điểm):
  - Tăng → tông tích cực ("Tiến bộ! Band tổng +0.5…").
  - **Giảm → vẫn hiện ▼ kèm lời động viên** ("…đừng nản, thử lại để cải thiện nhé!").
  - Không đổi → nêu tiêu chí nào tăng/giảm.
- **Biểu đồ tiến bộ (mini trend)**: đường band tổng qua các lượt + tổng chênh lệch. **Đặt cố định phía trên khu chat** (luôn hiển thị, không cuộn theo). Chỉ hiện khi có ≥ 2 lượt.

**Nghiệm thu:**
- [ ] Lượt 1 không có delta; từ lượt 2 có delta ở band tổng + 4 thẻ.
- [ ] Khi điểm giảm vẫn hiển thị trung thực + lời động viên (không ẩn).
- [ ] Mini trend luôn nhìn thấy khi cuộn chat; cập nhật theo từng lượt.

### 2.7. Nhiều lượt — tự thu gọn `P1`

**Mô tả:**
- Khi **lượt mới có kết quả**, các **lượt cũ tự động thu gọn** (chỉ còn điểm + delta + động viên); chỉ **lượt mới nhất mở rộng** đầy đủ.
- User vẫn bấm "Mở rộng" để xem lại lượt cũ.
- Bỏ bong bóng transcript "LẦN X" lặp lại (vì khu kết quả đã hiển thị câu trả lời); chỉ giữ nhãn "Lần X".

**Nghiệm thu:**
- [ ] Tạo lượt mới → lượt cũ collapse, lượt mới expand.
- [ ] Mở lại lượt cũ thủ công được.
- [ ] Không còn transcript lặp ở bong bóng riêng cho lượt đã chấm.

### 2.8. Edge case khi user nói `P0–P1`

Khi gặp sự cố, **thay khu kết quả bằng card lỗi chuyên biệt** (icon + tiêu đề + giải thích + hành động). Phân nhóm:

| Mã | Tình huống | Hành vi | Ưu tiên |
|---|---|---|---|
| A1 | Mic bị từ chối / chưa cấp quyền | Card đỏ + nút **"Hướng dẫn cấp quyền"** + "Thử lại". Chưa ghi nên không có transcript | P0 |
| A2 | Không có mic / trình duyệt không hỗ trợ | Card đỏ "Không tìm thấy micro" + "Thử lại" | P0 |
| B1 | Ghi quá ngắn | Card "Đoạn ghi quá ngắn (Xs)…" + "Ghi âm lại" | P0 |
| B2 | Im lặng / không phát hiện giọng | Card "Không nghe thấy giọng nói" + "Ghi âm lại" | P1 |
| B3 | Tạp âm/nhiễu lớn | Card "Âm thanh bị nhiễu" + "Ghi âm lại" | P1 |
| B4 | Ghi quá dài | **Tự dừng ở giới hạn** (có cảnh báo đếm ngược) + **vẫn chấm bình thường**, gắn nhãn "Đã tự dừng ở giới hạn" | P1 |
| B5 | Nói không phải tiếng Anh | Card "Có vẻ bạn chưa nói tiếng Anh" + "Ghi âm lại" | P1 |
| C1 | Lệch đề / off-topic | Card **"Off-Topic: Đọc kỹ câu hỏi nha"** nêu rõ đang nói lệch về chủ đề gì so với câu hỏi; **vẫn hiển thị transcript của user**; chỉ cho **ghi lại** | P1 |

> Giới hạn thời lượng (B4) đề xuất: Part 1 ~90s, Part 2 ~120s, Part 3 ~90s (cần chốt — xem §5). Trong demo dùng 15s cho dễ test.

**Nghiệm thu:**
- [ ] Mỗi tình huống hiển thị đúng card tương ứng (QA dùng ô "Demo edge case" để kiểm).
- [ ] A1/A2 không tạo bong bóng transcript; B/C có "Bản ghi Xs"; C1 hiện transcript.
- [ ] B4 tự dừng và vẫn ra kết quả chấm.

### 2.9. Kích thước & typography (compact) `P1`

**Mô tả:** Siết kích thước/cỡ chữ để mật độ thông tin cao hơn nhưng vẫn dễ đọc (tham chiếu trực tiếp file demo): điểm tổng nhỏ gọn hơn, body ~13px, padding/bo góc/khoảng cách giảm hợp lý.

**Nghiệm thu:**
- [ ] Khu kết quả, cột AI hỗ trợ, card lỗi, panel luyện âm gọn như demo, không bị tràn/cắt chữ.

---

## 3. Khác biệt theo Part

- **Part 1 / Part 3:** áp dụng đầy đủ như trên.
- **Part 2 (cue card):** câu hỏi dạng cue card (nhiều dòng "You should say…"), có thời gian chuẩn bị; phần kết quả/phân tích/edge case áp dụng tương tự. Lưu ý đề bài dài → bố cục câu hỏi giữ định dạng nhiều dòng.

---

## 4. Yêu cầu dữ liệu / backend (cần để chạy thật)

Các tính năng dưới đây cần tín hiệu thật từ backend; nhờ tech lead xác nhận khả thi & nguồn dữ liệu:

1. **Câu đã sửa (§2.2):** danh sách lỗi từ vựng/ngữ pháp kèm `vị trí trong câu`, `bản sửa`, `loại lỗi`, `giải thích` — để render track-changes + popup.
2. **Phát âm chi tiết (§2.4):** điểm phát âm **từng từ** (để chia 3 mức phụ/nhẹ/nặng), **phiên âm IPA**, và **phoneme-level** (MẪU vs BẠN) cho panel luyện âm; kèm **URL bản ghi** của user để phát lại.
3. **Hướng dẫn luyện âm (§2.4B):** nội dung hướng dẫn theo từng âm (vd /t/, /ɹ/, /ŋ/, /dʒ/…) — nguồn nội dung do team biên soạn hay sinh tự động?
4. **Điểm theo từng lượt (§2.6):** lưu điểm tổng + 4 tiêu chí của mỗi lượt để tính delta và vẽ mini trend.
5. **Edge case (§2.8):** tín hiệu phân biệt **im lặng / nhiễu / sai ngôn ngữ / lệch đề** thật sự (hiện hệ thống phân loại lỏng theo message lỗi); cần: phát hiện no-speech, mức nhiễu, **language detection**, và **đánh giá độ liên quan đề bài** thực chất (không chỉ theo thời lượng).
6. **Giới hạn thời lượng B4:** chốt max duration cho từng Part + cơ chế auto-stop ở recorder.
7. **Quyền mic (A1/A2):** xử lý khi user chưa cấp/đã thu hồi quyền micro lúc bấm ghi.

---

## 5. Tóm tắt ưu tiên

**Đợt 1 (P0):**
- Bỏ 3 tab + bỏ "xem chi tiết" (§2.1)
- "Câu trả lời đã sửa lỗi" + popup giải thích (§2.2)
- Edge case mic & ghi quá ngắn: A1, A2, B1 (§2.8)

**Đợt 2 (P1):**
- 4 thẻ điểm = nút mở phân tích ở cột AI hỗ trợ (§2.3)
- Phân tích phát âm: transcript màu + IPA + audio (§2.4A)
- "Gợi ý câu trả lời tốt hơn" (§2.5)
- So sánh điểm qua các lượt + động viên + mini trend (§2.6)
- Tự thu gọn lượt cũ (§2.7)
- Edge case B2, B3, B4, B5, C1 (§2.8)
- Compact UI (§2.9)

**Đợt 3 (P2):**
- Panel luyện âm từng từ (§2.4B)

---

## 6. Tham chiếu

- **Visual spec:** [`practice-flow-optimized.html`](./practice-flow-optimized.html) — có ô **"Demo edge case"** để QA kích hoạt A1/A2/B1/B2/B3/B4/B5/C1; ghi âm nhiều lượt để xem delta + mini trend.
- **Baseline (luồng hiện tại):** [`practice-flow-mockup.html`](./practice-flow-mockup.html) — để đối chiếu "before".
