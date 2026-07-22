# Module Practice — Tiêu chí chấp nhận (Acceptance Criteria)

> **Mục đích:** Checklist nghiệm thu chi tiết cho từng tính năng module Practice (bản tối ưu), phục vụ QA & dev tự kiểm.
> **Phạm vi:** tính năng trong [`practice-flow-optimized.html`](./practice-flow-optimized.html). Đi kèm: [`PRACTICE_MODULE_FEATURES.md`](./PRACTICE_MODULE_FEATURES.md).
> **Cách dùng:** mỗi mục là 1 điều kiện **độc lập kiểm chứng được** (pass/fail). `[ ]` = chưa kiểm · đánh dấu khi đạt.
> **Ngày:** 2026-06-16.

---

## 1. Màn Chọn nội dung (Index)

### 1.1. IELTS Forecast
- [ ] Banner hiển thị **số câu hỏi cho từng Part** (1/2/3) và **ngày cập nhật**.
- [ ] Số liệu khớp với Part đang xem.

### 1.2. Chọn Part
- [ ] Có đủ 3 tab Part 1 / Part 2 / Part 3.
- [ ] Tab đang chọn được làm nổi (màu khác các tab còn lại).
- [ ] Đổi tab → danh sách chủ đề đổi sang đúng Part.
- [ ] Đổi tab → tự chọn chủ đề đầu tiên & cập nhật khu chi tiết.

### 1.3. Danh sách chủ đề
- [ ] Mỗi chủ đề hiển thị: tên, số câu, thời lượng (và danh mục với Part 2/3).
- [ ] Bấm chủ đề → chủ đề được highlight + khu chi tiết cập nhật.
- [ ] Chỉ một chủ đề ở trạng thái "đang chọn" tại một thời điểm.

### 1.4. Chi tiết chủ đề
- [ ] Hiển thị tiêu đề, danh mục, thời lượng, số câu.
- [ ] Hiển thị danh sách câu hỏi; **Part 2 hiển thị cue card nhiều dòng** đúng định dạng.
- [ ] Bấm 1 câu hỏi → vào phòng luyện **đúng câu đó**.
- [ ] Bấm "Bắt đầu luyện tập" → vào phòng luyện từ **câu đầu tiên**.
- [ ] Hộp "Mẹo cho phần này" hiển thị danh sách mẹo.

---

## 2. Phòng luyện — khung & điều hướng

### 2.1. Header
- [ ] Hiển thị nút "Về danh sách", nhãn Part, tên chủ đề, bộ đếm **"Câu X / Y"**.
- [ ] Bấm "Về danh sách" → quay lại màn Index.
- [ ] Bộ đếm cập nhật đúng khi chuyển câu.

### 2.2. Điều hướng câu (Trước / Tiếp)
- [ ] Nút "Câu trước" **vô hiệu** ở câu đầu.
- [ ] Nút "Câu tiếp" **vô hiệu** ở câu cuối.
- [ ] Chuyển câu → reset trạng thái ghi âm về ban đầu.
- [ ] Chuyển câu → xóa khu kết quả/lượt của câu trước (không lẫn sang câu mới).

### 2.3. Câu hỏi & nghe câu hỏi
- [ ] Hiển thị đúng nội dung câu hỏi hiện tại.
- [ ] Part 2 hiển thị dạng cue card (nhiều dòng).
- [ ] Nút 🔊 "nghe câu hỏi" có phản hồi khi bấm.

---

## 3. Ghi âm

### 3.1. Trạng thái nút ghi
- [ ] Trạng thái **chờ ghi**: nút xanh, nhãn "Bắt đầu ghi âm".
- [ ] Trạng thái **đang ghi**: nút đỏ, có sóng âm + **đồng hồ đếm giờ**, nhãn "Nhấn để dừng".
- [ ] Trạng thái **đang chấm**: nút xám/không bấm được, nhãn "Đang chấm điểm…".
- [ ] Sau khi có kết quả: nhãn đổi thành "Ghi âm lại".

### 3.2. Giới hạn thời lượng
- [ ] Khi sắp chạm giới hạn tối đa → hiển thị **cảnh báo đếm ngược**.
- [ ] Chạm giới hạn → **tự dừng** và **vẫn chấm bình thường**.
- [ ] Lượt tự dừng được gắn nhãn "Đã tự dừng ở giới hạn".

### 3.3. Ghi quá ngắn
- [ ] Dừng khi đoạn ghi dưới ngưỡng tối thiểu → hiện card "Đoạn ghi quá ngắn (Xs)" + nút "Ghi âm lại" (không chấm).

---

## 4. Khu hội thoại (timeline)

- [ ] Khi chưa có lượt → hiển thị gợi ý "Bấm ghi âm để bắt đầu".
- [ ] Mỗi lượt hiển thị nhãn "Lần X" + khu kết quả chấm điểm.
- [ ] **Không** hiển thị transcript lặp lại ở bong bóng riêng (với lượt đã chấm).
- [ ] Khi tạo lượt mới có kết quả → **các lượt cũ tự thu gọn**, chỉ lượt mới nhất mở rộng.
- [ ] Bấm "Mở rộng" trên lượt cũ → mở lại đầy đủ lượt đó.

---

## 5. Cột AI hỗ trợ

- [ ] Bấm "Câu trả lời mẫu" → thêm khối câu mẫu vào cột phải.
- [ ] Bấm "Từ vựng chủ đề" → thêm danh sách từ vựng kèm cấp độ CEFR.
- [ ] Bấm "Ghi chú" → thêm ô nhập ghi chú tự do.
- [ ] Phân tích tiêu chí và cải thiện câu cùng hiển thị trong cột này và cuộn chung.

---

## 6. Kết quả chấm điểm

### 6.1. Điểm tổng & 4 tiêu chí
- [ ] Hiển thị band tổng + 4 thẻ (Trôi chảy & Mạch lạc, Từ vựng, Phát âm, Ngữ pháp) khi có kết quả.
- [ ] Mỗi thẻ tô đúng tông màu tiêu chí.
- [ ] Rê chuột vào thẻ → tooltip "Xem phân tích".
- [ ] Bấm thẻ → mở phân tích tiêu chí tương ứng ở cột AI hỗ trợ.

### 6.2. Câu trả lời đã sửa lỗi
- [ ] Hiển thị lỗi **gạch bỏ (đỏ)** + **bản sửa (xanh)** inline.
- [ ] Bấm vào chỗ sửa → popup hiển thị loại lỗi + `lỗi → bản sửa` + giải thích.
- [ ] Bấm ra ngoài popup → popup đóng.
- [ ] Khi chưa có dữ liệu phân tích → hiển thị skeleton "AI đang phân tích & sửa lỗi…".

---

## 7. Phân tích chi tiết — chung
- [ ] Mỗi card phân tích có nút **đóng (✕)** và nút **⛶ Bản đầy đủ**.
- [ ] Mở nhiều tiêu chí → các card cùng tồn tại trong cột AI hỗ trợ.
- [ ] Bản đầy đủ mở dạng panel rộng (2 cột).
- [ ] Phân tích **không** lặp lại lỗi đã hiển thị ở "Câu trả lời đã sửa lỗi".

### 7.1. Phát âm
- [ ] Rail: hiển thị phân bố Tốt/Khá/Yếu, top 3 từ cần sửa (kèm %), âm yếu nhất.
- [ ] Panel: transcript tô nền theo 3 band (Tốt/Khá/Cần cải thiện) + chú giải.
- [ ] Panel: **danh sách lỗi sắp xếp nặng → nhẹ**, gộp số lần lặp, kèm % điểm.
- [ ] Mỗi lỗi có **2 nút nghe** (âm chuẩn + đoạn bạn nói) và câu ngữ cảnh.
- [ ] Có phần **nhóm theo âm** (mỗi nhóm: mẹo + các từ dính).
- [ ] Có phần **trọng âm & ngữ điệu** (từ sai trọng âm + nhận xét).
- [ ] Bấm "Luyện" / bấm từ → mở drill luyện âm.

### 7.2. Trôi chảy & Mạch lạc
- [ ] Rail: badge Mạch lạc + nhận xét; badge Trôi chảy; tốc độ (wpm + nhãn); tóm tắt ngắt nghỉ.
- [ ] Panel: **thước đo tốc độ** hiển thị vùng Tốt (120–150) và kim chỉ đúng vị trí wpm.
- [ ] Panel: transcript đánh dấu **ngắt nghỉ 3 mức** (tốt/vừa/dài) + filler.
- [ ] Panel: phần Mạch lạc có badge + nhận xét + liên từ đã dùng + gợi ý.

### 7.3. Từ vựng
- [ ] Rail: badge cấp độ + phân bố CEFR (A1→C2) + top gợi ý nâng cấp.
- [ ] Panel: transcript **tô màu theo CEFR** + chú giải A1–C2.
- [ ] Panel: phân bố CEFR đầy đủ + **card gợi ý paraphrase** (từ cơ bản → lựa chọn nâng cấp).

### 7.4. Ngữ pháp
- [ ] Rail: độ chính xác (số câu không lỗi), độ đa dạng câu (đơn/phức), lỗi thường gặp.
- [ ] Panel: transcript **theo từng câu** với vạch màu (không lỗi/có lỗi) + nhãn Câu đơn/Câu phức.
- [ ] Panel: **cấu trúc nâng cao** với trạng thái (đúng/chưa đúng/nên thử).
- [ ] Panel: **lỗi theo loại** (kèm số lượng) + **gợi ý cấu trúc** có ví dụ.

### 7.5. Luyện âm từng từ (drill)
- [ ] Hiển thị từ + IPA + nghĩa + nút nghe.
- [ ] Hiển thị so sánh **MẪU vs BẠN** theo phoneme (đúng/sai phân biệt màu).
- [ ] Có hướng dẫn khẩu hình cho âm sai + nút "Ghi âm ngay".
- [ ] Khi mở từ panel rộng → drill hiển thị **đè lên trên** panel rộng (không bị che).

---

## 8. Cải thiện câu (coach)

- [ ] Khu kết quả có **3 nút mục tiêu**: Nâng từ vựng / Đa dạng ngữ pháp / Phát triển ý sâu.
- [ ] Bấm 1 mục tiêu → có trạng thái "đang tạo" rồi hiện **card coach** ở cột AI hỗ trợ.
- [ ] Card coach: câu nâng cấp có **chỗ thay đổi tô màu**; bấm chỗ tô màu → giải thích **vì sao + kỹ thuật**.
- [ ] Card có nhãn nhóm đã nâng (Từ vựng/Ngữ pháp/Liên kết/Collocation).
- [ ] Bản đầy đủ: hiển thị diff theo tiêu chí + **nghe mẫu** + **dịch nghĩa**.
- [ ] Bản đầy đủ: phần **"Bạn đã nâng cấp gì"** liệt kê theo tiêu chí.
- [ ] Bản đầy đủ: phần **"Cụm đáng học"**, mỗi cụm có nút **Lưu** → hiện toast xác nhận đã lưu.
- [ ] Có nút **"Thử nói lại theo gợi ý"** (đóng coach + nhắc ghi âm lại).
- [ ] Không hiển thị số band cụ thể kiểu "phù hợp với band 7.0" (chỉ "hướng tới band kế tiếp").

---

## 9. So sánh qua các lượt

- [ ] Lượt 1 **không** hiển thị delta.
- [ ] Từ lượt 2: band tổng có chip delta (▲ tăng / ▼ giảm / – không đổi) đúng dấu & màu.
- [ ] Từ lượt 2: mỗi thẻ tiêu chí có delta nhỏ tương ứng.
- [ ] Dòng động viên: tăng → tích cực; **giảm → vẫn hiện ▼ kèm lời động viên**; bằng → nêu tiêu chí ↑/↓.
- [ ] **Mini trend** band tổng hiển thị khi có ≥ 2 lượt.
- [ ] Mini trend đặt **cố định phía trên khu chat**, vẫn thấy khi cuộn danh sách lượt.
- [ ] Mini trend cập nhật khi có lượt mới.

---

## 10. Edge case khi nói

- [ ] **A1** (mic từ chối): hiện card lỗi + "Hướng dẫn cấp quyền" + "Thử lại"; **không** có bong bóng transcript.
- [ ] **A2** (không có mic): hiện card lỗi + "Thử lại".
- [ ] **B1** (quá ngắn): card "quá ngắn (Xs)" + "Ghi âm lại".
- [ ] **B2** (im lặng): card "Không nghe thấy giọng nói" + "Ghi âm lại".
- [ ] **B3** (nhiễu): card "Âm thanh bị nhiễu" + "Ghi âm lại".
- [ ] **B4** (quá dài): tự dừng + **vẫn chấm** + nhãn "Đã tự dừng ở giới hạn".
- [ ] **B5** (sai ngôn ngữ): card "chưa nói tiếng Anh" + "Ghi âm lại".
- [ ] **C1** (lệch đề): card "Off-Topic…" nêu chủ đề bị lệch; **vẫn hiển thị transcript**; chỉ có nút ghi lại.
- [ ] B/C (đã ghi) hiển thị "Bản ghi Xs"; A1/A2 không hiển thị bản ghi.

---

## 11. Giao diện gọn (compact)
- [ ] Các khu (kết quả, rail, card lỗi, panel rộng, drill) hiển thị gọn, không tràn/cắt chữ.
- [ ] Panel rộng & popup không bị che/đè sai thứ tự (drill luôn trên panel rộng).

---

## 12. Khác biệt theo Part
- [ ] Part 1 & Part 3 áp dụng đầy đủ các mục trên.
- [ ] Part 2 hiển thị cue card đúng định dạng; các chức năng kết quả/phân tích/cải thiện/edge case hoạt động tương tự.

---

## 13. Đo lường (analytics) — nghiệm thu gắn sự kiện
- [ ] Có track: xem Index, chọn Part, chọn chủ đề, bắt đầu luyện.
- [ ] Có track: bắt đầu/dừng ghi (kèm thời lượng), ghi lại, edge case hiển thị (theo mã).
- [ ] Có track: xem điểm, mở phân tích từng tiêu chí, mở Bản đầy đủ.
- [ ] Có track: nghe âm chuẩn/nghe lại, mở luyện âm.
- [ ] Có track: bấm mục tiêu cải thiện, lưu cụm đáng học, "Thử nói lại".
- [ ] Có track: số lượt/câu, thay đổi band giữa các lượt.
- [ ] Mỗi sự kiện có thuộc tính tối thiểu phù hợp (part, chủ đề, câu hỏi, mã lỗi/tiêu chí khi liên quan).

---

## 14. Thời gian trả kết quả & cảm nhận chờ (performance) ⏱️

> **Lý do:** chờ lâu là nguyên nhân bỏ cuộc hàng đầu. Mục tiêu: phản hồi tức thì + **hiển thị kết quả từng phần (progressive reveal)** để không bắt user chờ trắng màn hình.
> *Các mốc thời gian dưới là **đề xuất** — cần đo thực tế và chốt theo hạ tầng. Đo bằng p50 (trung vị) & p95.*

### 14.1. Phản hồi tức thì khi thao tác
- [ ] Bấm nút ghi/dừng → **đổi trạng thái ngay (< 100 ms)**, không độ trễ cảm nhận được.
- [ ] Sau khi dừng ghi → **trạng thái "đang chấm" + chỉ báo xuất hiện < 300 ms** (không có khoảng "đứng hình").
- [ ] Mọi thao tác mở (panel/drill/card phân tích) phản hồi **< 300 ms** vì dùng dữ liệu đã có sẵn (không gọi mạng).

### 14.2. Thời gian tới kết quả (kể từ khi dừng ghi)
- [ ] **Kết quả đầu tiên** (band tổng + 4 tiêu chí + phát âm) trả về: **p50 ≤ 12 s · p95 ≤ 20 s**.
- [ ] **Phần phân tích sâu** (câu đã sửa + từ vựng + ngữ pháp) trả về sau đó: **+ ≤ 10 s** (p95 **tổng ≤ 30 s**).
- [ ] **Progressive reveal**: band + phát âm hiển thị **TRƯỚC**, KHÔNG chờ từ vựng/ngữ pháp xong mới hiện.
- [ ] **Cải thiện câu (coach)** trả gợi ý: **p50 ≤ 8 s · p95 ≤ 15 s**.

### 14.3. Hành động lazy-load
- [ ] Nghe **âm chuẩn** / **nghe lại đoạn** bắt đầu phát **≤ 1.5 s**.
- [ ] Nghe **mẫu câu nâng cấp** bắt đầu phát **≤ 2 s**.
- [ ] Mở panel rộng / drill **không gọi thêm mạng** → mở tức thì (**< 300 ms**).

### 14.4. Giảm cảm giác chờ
- [ ] Trong suốt thời gian chờ luôn có **chỉ báo tiến trình/đang xử lý** — **không bao giờ để màn hình trắng**.
- [ ] Khu vực chờ dùng **skeleton** đúng hình dạng nội dung sắp hiện (không nhảy layout khi có data).
- [ ] Có **thông điệp trấn an** trong lúc chờ (vd "AI đang phân tích…") thay vì spinner trơ.
- [ ] User có thể **rời đi và quay lại** mà không mất tiến độ/kết quả lượt đang chấm.

### 14.5. Timeout & lỗi chậm
- [ ] Nếu **kết quả đầu** vượt **~45 s** → hiển thị lỗi + nút "Thử lại" (KHÔNG treo vô hạn).
- [ ] Nếu **phân tích sâu** vượt **~60 s** → hiển thị lỗi từng phần + cho **thử lại riêng phần đó**.
- [ ] **Lỗi/chậm một phần** (từ vựng HOẶC ngữ pháp HOẶC phát âm) **không chặn** các phần đã có; retry từng phần độc lập.
- [ ] Khi quá tải/hết hạn dịch vụ → thông báo rõ ràng, không hiện kết quả rỗng/sai.

### 14.6. Đo lường để giám sát (gắn với §13)
- [ ] Track **thời gian từ lúc dừng ghi → kết quả đầu** và **→ kết quả đầy đủ** (để theo dõi p50/p95 thực tế).
- [ ] Track **tỷ lệ bỏ giữa chừng khi đang chấm** (user rời trang trước khi có kết quả) — chỉ số cảnh báo sớm.
