# Module Practice — Tài liệu tính năng (bản tối ưu)

> **Mục đích:** Tài liệu tổng hợp nội bộ (dùng chung PM/Dev/QA) mô tả **toàn bộ tính năng của module Practice** theo bản tối ưu.
> **Phạm vi:** các tính năng có trong bản mẫu [`practice-flow-optimized.html`](./practice-flow-optimized.html).
> **Văn phong:** thuần product (mô tả tính năng & hành vi) + mục **Đo lường** (§11). Không bao gồm chi tiết kỹ thuật/tích hợp.
> **Phạm vi Part:** Part 1, Part 2, Part 3 (khác biệt Part 2 cue card ghi trong từng mục).
> **Ngày:** 2026-06-16.

---

## 0. Quy ước
- Mỗi tính năng gồm: **Mục đích · Hành vi · Nghiệm thu**.
- "Lượt" = một lần user ghi âm & được chấm cho cùng câu hỏi.
- "Cột AI hỗ trợ" = cột phải trong phòng luyện.
- "Panel rộng" = cửa sổ đầy đủ mở từ nút **⛶ Bản đầy đủ**.
- *Ghi chú QA:* bản demo có ô **"Demo edge case"** để kích hoạt thử các tình huống lỗi — đây là công cụ test, không phải tính năng sản phẩm.

---

## 1. Bản đồ tính năng

1. **Chọn nội dung luyện** (màn Index): IELTS Forecast · chọn Part · danh sách chủ đề · chi tiết chủ đề (tips + câu hỏi).
2. **Phòng luyện**: câu hỏi + nghe câu hỏi · ghi âm · điều hướng câu · khu hội thoại · cột AI hỗ trợ.
3. **Kết quả chấm điểm**: band tổng + 4 tiêu chí · dòng động viên · câu trả lời đã sửa lỗi.
4. **Phân tích chi tiết 4 tiêu chí** (Phát âm/Trôi chảy & Mạch lạc/Từ vựng/Ngữ pháp) + luyện âm từng từ.
5. **Cải thiện câu** (coach).
6. **So sánh qua các lượt** (delta + tiến bộ).
7. **Edge case khi nói**.
8. **Giao diện gọn**.

---

## 2. Màn Chọn nội dung luyện (Index)

### 2.1. IELTS Forecast
- **Mục đích:** tạo cảm giác "đề dự báo" & cho thấy quy mô ngân hàng câu hỏi.
- **Hành vi:** banner đầu cột trái hiển thị ngày cập nhật + số câu hỏi theo từng Part (Part 1/2/3).
- **Nghiệm thu:** [ ] Hiện đúng số câu mỗi Part · [ ] Có nhãn ngày cập nhật.

### 2.2. Chọn Part
- **Mục đích:** chọn phần luyện (Part 1 phỏng vấn / Part 2 cue card / Part 3 thảo luận).
- **Hành vi:** tab Part 1/2/3; đổi tab → đổi danh sách chủ đề tương ứng; tab đang chọn được làm nổi.
- **Nghiệm thu:** [ ] Đổi tab đổi đúng danh sách · [ ] Trạng thái active rõ ràng.

### 2.3. Danh sách chủ đề
- **Mục đích:** chọn chủ đề để luyện.
- **Hành vi:** mỗi chủ đề hiển thị tên, danh mục (Part 2/3), số câu, thời lượng ước tính; chọn → highlight + mở chi tiết bên phải.
- **Nghiệm thu:** [ ] Chọn chủ đề → chi tiết cập nhật · [ ] Chủ đề đang chọn được đánh dấu.

### 2.4. Chi tiết chủ đề
- **Mục đích:** xem trước câu hỏi & mẹo trước khi luyện.
- **Hành vi:**
  - Tiêu đề + danh mục + thời lượng + số câu.
  - **Danh sách câu hỏi** (Part 2 hiển thị cue card nhiều dòng); bấm 1 câu → vào phòng luyện đúng câu đó.
  - Nút **"Bắt đầu luyện tập"** → vào phòng luyện từ câu đầu.
  - Hộp **"Mẹo cho phần này"**.
- **Nghiệm thu:** [ ] Bấm câu hỏi/"Bắt đầu" vào đúng phòng luyện · [ ] Part 2 giữ định dạng cue card · [ ] Có hộp mẹo.

---

## 3. Phòng luyện

### 3.1. Header & điều hướng câu
- **Mục đích:** định hướng & chuyển câu.
- **Hành vi:** header hiển thị **Về danh sách · Part · Chủ đề · "Câu X / Y"**; nút **Câu trước / Câu tiếp** (vô hiệu ở đầu/cuối). Chuyển câu → reset trạng thái ghi âm & khu kết quả của câu đó.
- **Nghiệm thu:** [ ] Đếm câu đúng · [ ] Prev/Next vô hiệu đúng biên · [ ] Chuyển câu reset đúng.

### 3.2. Câu hỏi & nghe câu hỏi
- **Mục đích:** đọc & nghe đề.
- **Hành vi:** hiển thị câu hỏi (Part 2 dạng cue card nhiều dòng); nút 🔊 **nghe câu hỏi**.
- **Nghiệm thu:** [ ] Hiển thị đúng câu hỏi · [ ] Nút nghe hoạt động · [ ] Part 2 hiển thị cue card.

### 3.3. Ghi âm
- **Mục đích:** thu câu trả lời.
- **Hành vi:**
  - Nút ghi tròn lớn với **trạng thái**: chờ ghi → đang ghi (đỏ, có sóng âm + **đếm giờ**) → đang chấm (xám) → đã chấm.
  - **Giới hạn thời lượng tối đa**: gần giới hạn cảnh báo đếm ngược; chạm giới hạn **tự dừng + vẫn chấm** (gắn nhãn "Đã tự dừng ở giới hạn").
  - Nhãn dưới nút đổi theo trạng thái (Bắt đầu ghi / Nhấn để dừng / Đang chấm / Ghi âm lại).
- **Nghiệm thu:** [ ] Đủ 4 trạng thái + timer · [ ] Tự dừng khi quá dài và vẫn chấm.

### 3.4. Khu hội thoại (timeline) & tự thu gọn
- **Mục đích:** xem các lượt thử dạng hội thoại.
- **Hành vi:**
  - Trống → gợi ý "Bấm ghi âm để bắt đầu".
  - Mỗi lượt: nhãn **"Lần X"** + khu kết quả chấm điểm (không lặp transcript ở bong bóng riêng).
  - **Lượt mới có kết quả → các lượt cũ tự thu gọn**; chỉ lượt mới nhất mở rộng. Có nút **Thu gọn/Mở rộng** từng lượt.
- **Nghiệm thu:** [ ] Trạng thái trống đúng · [ ] Lượt mới mở, lượt cũ thu gọn · [ ] Mở lại lượt cũ được.

### 3.5. Cột AI hỗ trợ
- **Mục đích:** hỗ trợ trong lúc luyện.
- **Hành vi:** 3 nút thêm nội dung vào cột phải:
  - **Câu trả lời mẫu** · **Từ vựng chủ đề** (kèm cấp độ CEFR) · **Ghi chú** (ô nhập tự do).
  - Đây cũng là nơi hiển thị **phân tích tiêu chí** (§5) và **cải thiện câu** (§6) khi user mở.
- **Nghiệm thu:** [ ] 3 nút thêm đúng nội dung · [ ] Các nội dung cùng cuộn trong cột phải.

---

## 4. Kết quả chấm điểm

### 4.1. Điểm tổng & 4 tiêu chí
- **Mục đích:** kết quả band tổng + 4 tiêu chí.
- **Hành vi:** "AI Chấm điểm" + **band tổng** lớn; **4 thẻ**: Trôi chảy & Mạch lạc · Từ vựng · Phát âm · Ngữ pháp (mỗi thẻ tông màu). **4 thẻ là nút bấm** → mở phân tích chi tiết (§5); rê chuột có tooltip "Xem phân tích".
- **Nghiệm thu:** [ ] Band + 4 tiêu chí hiển thị khi có kết quả · [ ] Bấm thẻ mở phân tích đúng.

### 4.2. Dòng động viên & delta
- (Xem §7 — so sánh qua các lượt.)

### 4.3. Câu trả lời đã sửa lỗi
- **Mục đích:** chỉ rõ & sửa lỗi từ vựng/ngữ pháp ngay trên câu.
- **Hành vi:** transcript với lỗi **gạch bỏ (đỏ)** + **bản sửa (xanh)** inline; **bấm vào chỗ sửa** → popup giải thích (loại lỗi + lý do). Lúc chờ → skeleton "AI đang phân tích & sửa lỗi…".
- **Nghiệm thu:** [ ] Hiển thị track-changes · [ ] Bấm ra popup, bấm ngoài đóng · [ ] Có skeleton khi chờ.

---

## 5. Phân tích chi tiết 4 tiêu chí

**Chung:** bấm thẻ điểm → card **gọn ở cột AI hỗ trợ**; trong card có **⛶ Bản đầy đủ** mở **panel rộng 2 cột**. Mở nhiều tiêu chí cùng lúc; đóng từng card. Không lặp lại lỗi của §4.3.

### 5.1. Phát âm
- **Rail (gọn):** phân bố **Tốt/Khá/Yếu** · **top 3 từ cần sửa nhất** (kèm %) · **âm yếu nhất**.
- **Panel rộng (4 lớp):**
  1. **Transcript theo độ chuẩn** (Tốt không nền · Khá vàng · Cần cải thiện đỏ) + nghe lại + chú giải; bấm từ → luyện âm.
  2. **Danh sách lỗi (nặng→nhẹ):** từ + số lần lặp + % · **✓ Đúng /IPA/ 🔊** và **✗ Bạn nói /IPA/ 🔊** (2 nút nghe đối chiếu) · câu ngữ cảnh · nút **Luyện**.
  3. **Nhóm theo âm** (sửa gốc rễ): gom lỗi theo âm + mẹo + các từ dính.
  4. **Trọng âm & Ngữ điệu:** từ sai trọng âm + nhận xét nhịp & ngữ điệu.
- **Nghiệm thu:** [ ] 3 band transcript · [ ] Danh sách lỗi sắp xếp + gộp lặp + 2 nút nghe · [ ] Có nhóm âm & trọng âm/ngữ điệu.

### 5.2. Trôi chảy & Mạch lạc
- **Rail (gọn):** badge Mạch lạc + nhận xét · badge Trôi chảy · **tốc độ nói** (wpm + nhãn) · tóm tắt ngắt nghỉ.
- **Panel rộng (2 cột):** trái — transcript **chấm ngắt nghỉ 3 mức** (🟢 tốt/🟡 vừa/🔴 dài) + filler + nghe lại; phải — **thước đo tốc độ** (vùng Tốt 120–150, kim chỉ) + **Mạch lạc** (badge + nhận xét + liên từ đã dùng + gợi ý).
- **Nghiệm thu:** [ ] Gauge tốc độ · [ ] Transcript ngắt nghỉ 3 mức · [ ] Tách Trôi chảy & Mạch lạc.

### 5.3. Từ vựng
- **Rail (gọn):** badge cấp độ · **phân bố CEFR** (A1→C2) · top gợi ý nâng cấp.
- **Panel rộng (2 cột):** trái — transcript **tô màu theo CEFR** + chú giải; phải — **phân bố CEFR đầy đủ** + **card gợi ý paraphrase** (từ cơ bản → lựa chọn nâng cấp).
- **Nghiệm thu:** [ ] Phân bố CEFR · [ ] Transcript tô màu cấp độ · [ ] Gợi ý paraphrase.

### 5.4. Ngữ pháp
- **Rail (gọn):** **độ chính xác** (số câu không lỗi) · **độ đa dạng câu** (đơn/phức) · lỗi thường gặp · 1 gợi ý.
- **Panel rộng (2 cột):** trái — transcript **theo từng câu** (🟢 không lỗi/🟡 có lỗi + nhãn đơn/phức); phải — độ chính xác + độ đa dạng + **cấu trúc nâng cao** (✅ đúng/⚠️ chưa đúng/💡 nên thử) + **lỗi theo loại** + **gợi ý cấu trúc** kèm ví dụ.
- **Nghiệm thu:** [ ] Câu theo độ chính xác + nhãn đơn/phức · [ ] Cấu trúc nâng cao có trạng thái · [ ] Lỗi theo loại + gợi ý.

### 5.5. Luyện âm từng từ (drill)
- **Mục đích:** luyện 1 âm/từ cụ thể.
- **Hành vi:** mở từ mọi nút "Luyện"/từ trong transcript phát âm — từ + nghe + IPA + nghĩa + **MẪU vs BẠN** (phoneme) + hướng dẫn khẩu hình + **Ghi âm ngay**. Popup **đè lên** panel rộng.
- **Nghiệm thu:** [ ] Hiện đúng so sánh MẪU/BẠN + hướng dẫn · [ ] Hiển thị đè trên panel rộng.

---

## 6. Cải thiện câu (coach)

- **Mục đích:** dạy kỹ thuật nâng câu (không chỉ phát câu hoàn hảo) + bắt luyện lại.
- **Hành vi:**
  - **3 nút mục tiêu** trong khu kết quả: **Nâng từ vựng · Đa dạng ngữ pháp · Phát triển ý sâu**.
  - Kết quả ở cột AI hỗ trợ dạng **card coach**: câu nâng cấp với **chỗ thay đổi tô màu** (bấm → vì sao + kỹ thuật) · nhãn nhóm đã nâng · **⛶ Bản đầy đủ** · **🎙 Thử nói lại**.
  - **Panel rộng:** câu nâng cấp (diff theo tiêu chí) + **nghe mẫu** + **dịch nghĩa**; **"Bạn đã nâng cấp gì"** (theo tiêu chí); **"Cụm đáng học"** (mỗi cụm có nút **Lưu** vào flashcard); nút **"Thử nói lại theo gợi ý"**.
- **Nghiệm thu:** [ ] 3 mục tiêu cho câu nâng cấp + diff bấm xem kỹ thuật · [ ] "Đã nâng gì" + "Cụm đáng học" lưu được (toast) · [ ] Có nghe mẫu + thử nói lại.

---

## 7. So sánh qua các lượt

- **Mục đích:** thấy tiến bộ giữa các lượt.
- **Hành vi (so với lượt liền trước, từ lượt 2):**
  - **Band tổng:** chip 🟢 ▲ / 🔴 ▼ / ⚪ –.
  - **Mỗi thẻ tiêu chí:** delta nhỏ (▲/▼/–).
  - **Dòng động viên:** tăng → tích cực; **giảm → vẫn hiện ▼ + động viên**; bằng → nêu tiêu chí ↑/↓.
  - **Mini trend** (band tổng qua các lượt): đặt **cố định phía trên khu chat**, luôn thấy; hiện khi ≥ 2 lượt.
- **Nghiệm thu:** [ ] Lượt 1 không delta, từ lượt 2 có · [ ] Giảm vẫn hiển thị + động viên · [ ] Mini trend cố định, cập nhật theo lượt.

---

## 8. Edge case khi user nói

Khi gặp sự cố → **card lỗi chuyên biệt** thay khu kết quả.

| Mã | Tình huống | Hành vi |
|---|---|---|
| A1 | Mic bị từ chối | Card + "Hướng dẫn cấp quyền" + "Thử lại"; không có transcript |
| A2 | Không có mic / không hỗ trợ | Card + "Thử lại" |
| B1 | Ghi quá ngắn | Card "quá ngắn (Xs)" + "Ghi âm lại" |
| B2 | Im lặng / không có giọng | Card + "Ghi âm lại" |
| B3 | Tạp âm/nhiễu | Card + "Ghi âm lại" |
| B4 | Quá dài | Tự dừng + **vẫn chấm**, nhãn "Đã tự dừng ở giới hạn" |
| B5 | Sai ngôn ngữ | Card "chưa nói tiếng Anh" + "Ghi âm lại" |
| C1 | Lệch đề | Card "Off-Topic…" nêu lệch về chủ đề gì; **vẫn hiện transcript**; chỉ cho ghi lại |

- **Nghiệm thu:** [ ] Mỗi tình huống hiện đúng card · [ ] A1/A2 không transcript; C1 vẫn transcript; B4 tự dừng & vẫn chấm.

---

## 9. Giao diện gọn (compact)
- **Mục đích:** mật độ thông tin cao, vẫn dễ đọc.
- **Hành vi:** cỡ chữ/padding/bo góc/khoảng cách được siết hợp lý ở mọi khu (kết quả, rail, card lỗi, panel rộng, drill).
- **Nghiệm thu:** [ ] Không tràn/cắt chữ; bố cục gọn như bản mẫu.

---

## 10. Khác biệt theo Part
- **Part 1 / Part 3:** áp dụng đầy đủ.
- **Part 2 (cue card):** câu hỏi nhiều dòng + thời gian chuẩn bị; kết quả/phân tích/cải thiện/edge case áp dụng tương tự.

---

## 11. Đo lường (analytics)

Sự kiện/chỉ số nên track (mức product) — phục vụ đánh giá hành vi & hiệu quả học:

**Khám phá & bắt đầu:** xem màn Index · chọn Part · chọn chủ đề · xem chi tiết câu · bấm "Bắt đầu luyện tập".
**Ghi âm:** bắt đầu ghi · dừng ghi (kèm thời lượng) · ghi lại · lỗi edge case hiển thị (theo mã A/B/C).
**Kết quả:** xem điểm (band + tiêu chí) · mở phân tích từng tiêu chí (Phát âm/Trôi chảy/Từ vựng/Ngữ pháp) · mở "Bản đầy đủ".
**Phát âm:** nghe âm chuẩn · nghe lại đoạn · mở luyện âm từng từ · ghi âm luyện lại.
**Cải thiện câu:** bấm mục tiêu (vocab/grammar/idea) · xem giải thích kỹ thuật · lưu cụm đáng học · bấm "Thử nói lại".
**Tiến bộ:** số lượt mỗi câu · thay đổi band giữa các lượt (delta) · tỷ lệ cải thiện sau "Thử nói lại".
**Hỗ trợ:** mở câu mẫu · mở từ vựng chủ đề · dùng ghi chú.

- **Nghiệm thu:** [ ] Mỗi nhóm sự kiện trên được gắn track với thuộc tính tối thiểu (part, chủ đề, câu hỏi, mã lỗi/tiêu chí khi liên quan).

---

## 12. Tham chiếu
- **Bản mẫu (đầy đủ tính năng):** [`practice-flow-optimized.html`](./practice-flow-optimized.html).
- **Yêu cầu cập nhật & dữ liệu/backend (kèm kỹ thuật):** [`PRACTICE_OPTIMIZE_FULL_SPEC.md`](./PRACTICE_OPTIMIZE_FULL_SPEC.md).
