# Practice Optimize — Tài liệu tổng hợp (bản mới nhất)

> **Mục tiêu:** Mô tả đầy đủ module Practice sau tối ưu: kết quả chấm điểm dễ đọc, phân tích **chi tiết theo 4 tiêu chí**, "huấn luyện viên" cải thiện câu, so sánh tiến bộ qua các lượt, và xử lý edge case khi user nói.
>
> **Loại tài liệu:** Product requirement (mô tả hành vi & UI). Phần triển khai kỹ thuật do dev quyết định.
> **Visual spec:** [`practice-flow-optimized.html`](./practice-flow-optimized.html) — mở bằng trình duyệt; có ô **"Demo edge case"** cho QA.
> **Baseline (before):** [`practice-flow-mockup.html`](./practice-flow-mockup.html) để đối chiếu luồng cũ.
> **Quan hệ với doc cũ:** Tài liệu này **bao trùm & cập nhật** [`PRACTICE_FLOW_UPDATE_REQUIREMENT.md`](./PRACTICE_FLOW_UPDATE_REQUIREMENT.md) (bản trước chưa có phân tích chi tiết 4 tiêu chí & coach cải thiện câu).
> **Phạm vi:** Part 1, Part 2, Part 3 (lưu ý Part 2 cue card — §10).
> **Ngày:** 2026-06-16.

---

## 0. Quy ước
- **Ưu tiên:** `P0` bắt buộc (đợt 1) · `P1` nên có (đợt 2) · `P2` bổ sung sau.
- "Lượt" = một lần user ghi âm & được chấm cho cùng câu hỏi.
- "Khu kết quả" = panel chấm điểm dưới mỗi lượt (cột trái, khu chat).
- "Cột AI hỗ trợ" = cột phải (mẫu/từ vựng/ghi chú + nội dung phân tích & cải thiện).
- **Pattern hiển thị phân tích & cải thiện:** **rail gọn (key chính)** + nút **"⛶ Bản đầy đủ"** mở **panel rộng** (đầy đủ). Áp dụng đồng nhất cho 4 tiêu chí và cải thiện câu.

---

## 1. Tổng quan thay đổi (Before → After)

| # | Hạng mục | Hiện tại | Sau tối ưu | Ưu tiên |
|---|---|---|---|---|
| A | Phân tích bài nói | 3 tab Từ vựng/Ngữ pháp/Phát âm | Bỏ tab → **"Câu trả lời đã sửa lỗi"** + **4 thẻ điểm = nút mở phân tích** | P0 |
| B | Xem chi tiết | Nút "Xem chấm điểm chi tiết" (modal) | **Bỏ** | P0 |
| C | Phân tích theo tiêu chí | Trong tab | **Card chi tiết riêng từng tiêu chí** ở cột AI hỗ trợ + panel rộng | P1 |
| D | Phát âm | Highlight cơ bản | **Danh sách lỗi + nghe đối chiếu + nhóm theo âm + trọng âm/ngữ điệu + luyện từng từ** | P1 |
| E | Cải thiện câu | "Câu hay hơn" + dịch | **Coach: diff theo tiêu chí + đã nâng gì + cụm đáng học (lưu) + nghe mẫu + thử nói lại** | P1 |
| F | So sánh giữa các lượt | Không | **Delta + động viên + mini trend cố định** | P1 |
| G | Nhiều lượt | Đều mở, transcript lặp | **Lượt mới mở, lượt cũ tự thu gọn**; bỏ bong bóng transcript thừa | P1 |
| H | Edge case khi nói | Banner chung | **Card lỗi chuyên biệt** theo tình huống | P0–P1 |
| I | Kích thước/cỡ chữ | To/thoáng | **Gọn (mật độ cao, vẫn dễ đọc)** | P1 |

---

## 2. Khu kết quả chấm điểm — bố cục mới `P0`

Sau mỗi lượt nói, khu kết quả hiển thị từ trên xuống:
1. **Điểm tổng (overall band)** + "AI Chấm điểm" + **delta so với lượt trước** (§5).
2. **4 thẻ điểm thành phần** (Trôi chảy & Mạch lạc · Từ vựng · Phát âm · Ngữ pháp), mỗi thẻ một tông màu, **là nút bấm** mở phân tích (§6).
3. **Dòng động viên** (§5).
4. **"Câu trả lời của bạn — đã sửa lỗi"** (§3).
5. **Cải thiện câu trả lời** — 3 nút mục tiêu (§7).
6. Nút **Thu gọn/Mở rộng** lượt.

**Bỏ:** 3 tab; nút "Xem chấm điểm chi tiết" + modal chi tiết cũ.

**Nghiệm thu:**
- [ ] Không còn tab và nút "xem chi tiết".
- [ ] Điểm tổng + 4 tiêu chí hiển thị ngay khi có kết quả.
- [ ] Bấm thẻ điểm → mở phân tích tiêu chí tương ứng (§6).

---

## 3. "Câu trả lời đã sửa lỗi" (track-changes) `P0`

Transcript của user với lỗi **từ vựng & ngữ pháp** đánh dấu trực tiếp:
- Phần sai: **gạch bỏ** (đỏ); bản sửa: hiển thị cạnh (xanh).
- **Bấm vào chỗ sửa** → popup giải thích: loại lỗi (Từ vựng/Ngữ pháp), `lỗi → bản sửa`, lý do.
- Có chú giải; lúc chờ phân tích hiện skeleton "AI đang phân tích & sửa lỗi…".

> Đây là nơi DUY NHẤT hiển thị **lỗi** từ vựng/ngữ pháp. Các card phân tích tiêu chí (§6) **không lặp lại lỗi**, mà bổ trợ (range, cấp độ, cấu trúc…).

**Nghiệm thu:**
- [ ] Lỗi hiển thị inline; bấm ra popup đúng; bấm ngoài đóng.
- [ ] Có skeleton khi chưa có dữ liệu.

---

## 4. (Đã gộp vào §2) — giữ số mục để khớp tham chiếu nội bộ

---

## 5. So sánh điểm qua các lượt `P1`

So với **lượt liền trước**. Từ lượt 2:
- **Band tổng:** chip 🟢 `▲ +0.5` / 🔴 `▼ −0.5` / ⚪ `– 0`.
- **Mỗi thẻ tiêu chí:** delta nhỏ (▲/▼/–).
- **Dòng động viên:** tăng → tích cực; **giảm → vẫn hiện ▼ + lời động viên**; bằng → nêu tiêu chí ↑/↓.
- **Mini trend (band tổng qua các lượt):** **đặt cố định phía trên khu chat** (luôn thấy, không cuộn theo); hiện khi ≥ 2 lượt.
- **Lượt mới có kết quả → lượt cũ tự thu gọn**; chỉ lượt mới nhất mở rộng. Bỏ bong bóng transcript "LẦN X" lặp; giữ nhãn "Lần X".

**Nghiệm thu:**
- [ ] Lượt 1 không delta; từ lượt 2 có delta band + 4 thẻ.
- [ ] Điểm giảm vẫn hiển thị + động viên.
- [ ] Mini trend luôn thấy khi cuộn; lượt cũ auto thu gọn.

---

## 6. Phân tích chi tiết theo 4 tiêu chí `P1`

**Pattern chung:** bấm thẻ điểm → mở **card gọn ở cột AI hỗ trợ** (key chính); trong card có nút **"⛶ Bản đầy đủ"** mở **panel rộng 2 cột** (transcript bên trái + phân tích bên phải). Có thể mở nhiều tiêu chí cùng lúc (chèn thêm vào rail), đóng từng card.

> Nguyên tắc: phân tích **bổ trợ, không lặp lỗi** đã có ở §3.

### 6.1. Phát âm — bản "expert" (4 lớp)
**Rail (gọn):** band + phân bố **Tốt/Khá/Yếu** (%) + **top 3 từ cần sửa nhất** (kèm %) + **âm yếu nhất** (1 pattern).

**Panel rộng (2 cột):**
1. **Transcript theo độ chuẩn:** từ tô nền theo band (Tốt ≥75% không nền · Khá 50–75% vàng · Cần cải thiện <50% đỏ) + **nghe lại** + chú giải. Bấm từ → luyện âm.
2. **Danh sách lỗi (sắp xếp nặng→nhẹ):** mỗi lỗi = từ + **số lần lặp** + **% điểm**, **✓ Đúng /IPA/ 🔊** và **✗ Bạn nói /IPA/ 🔊** (2 nút nghe đối chiếu), câu ngữ cảnh, nút **Luyện**.
3. **Nhóm theo âm (sửa gốc rễ):** gom lỗi theo phoneme (vd /θ/ /ð/, /æ/, âm cuối /t,d/) — mỗi nhóm: mẹo khẩu hình + các từ dính (bấm để luyện).
4. **Trọng âm & Ngữ điệu (prosody):** từ sai trọng âm (vd `COM·fort·a·ble` vs `com·FOR·ta·ble`) + nhận xét nhịp & ngữ điệu.

**Luyện âm từng từ (drill):** mở từ mọi nút "Luyện"/từ trong transcript — hiện từ + nghe + IPA + nghĩa + **MẪU vs BẠN** (phoneme) + hướng dẫn khẩu hình + **Ghi âm ngay**. (Popup này hiển thị **đè lên** panel rộng.)

**Nghiệm thu:**
- [ ] Transcript phân biệt 3 band; danh sách lỗi sắp xếp + gộp lần lặp + 2 nút nghe.
- [ ] Có nhóm theo âm và phần trọng âm/ngữ điệu.
- [ ] Bấm "Luyện" mở drill đè trên panel rộng.

### 6.2. Trôi chảy & Mạch lạc
**Rail (gọn):** badge Mạch lạc + 1 dòng nhận xét; badge Trôi chảy; **tốc độ nói** (wpm + nhãn Chậm/Tốt/Nhanh); **tóm tắt ngắt nghỉ** (🟢 tốt · 🟡 vừa · 🔴 dài · filler).

**Panel rộng (2 cột):**
- **Trái (Trôi chảy):** transcript **chấm ngắt nghỉ 3 mức** (🟢 tốt/🟡 vừa/🔴 dài) + filler + **nghe lại** + chú giải.
- **Phải:** **thước đo tốc độ** (gauge 60–200 wpm, vùng Tốt 120–150, kim chỉ vị trí) + **Mạch lạc** (badge + nhận xét + **liên từ đã dùng** + gợi ý đa dạng).

**Nghiệm thu:**
- [ ] Gauge tốc độ + transcript ngắt nghỉ 3 mức.
- [ ] Tách rõ 2 phần Trôi chảy & Mạch lạc.

### 6.3. Từ vựng
**Rail (gọn):** badge cấp độ + **phân bố CEFR** (A1→C2 %) + nhận xét + **top gợi ý nâng cấp** (vài từ → thay thế).

**Panel rộng (2 cột):**
- **Trái:** transcript **tô màu theo CEFR** (A1 xám → C2 đỏ; từ nâng cao gạch chân; rê chuột xem cấp độ) + chú giải A1–C2.
- **Phải:** **phân bố CEFR đầy đủ** + note "Dùng từ gợi ý để nâng band" + **các card gợi ý paraphrase** (từ cơ bản → 3 lựa chọn nâng cấp).

**Nghiệm thu:**
- [ ] Phân bố CEFR + transcript tô màu cấp độ.
- [ ] Card gợi ý paraphrase cho từ overused/cơ bản.

### 6.4. Ngữ pháp
**Rail (gọn):** badge + **độ chính xác** (số câu không lỗi) + **độ đa dạng câu** (đơn/phức %) + **lỗi thường gặp** (loại + số lượng) + 1 gợi ý.

**Panel rộng (2 cột):**
- **Trái:** transcript **theo từng câu** — vạch màu 🟢 không lỗi / 🟡 có lỗi + nhãn **Câu đơn/Câu phức** + chú giải.
- **Phải:** Độ chính xác + Độ đa dạng câu; **Cấu trúc nâng cao** với trạng thái (✅ Dùng đúng / ⚠️ Chưa đúng / 💡 Nên thử); **Lỗi theo loại** (thì, mạo từ, giới từ…); **Gợi ý nâng cấp cấu trúc** kèm câu ví dụ.

**Nghiệm thu:**
- [ ] Sentence-level accuracy + nhãn đơn/phức.
- [ ] Cấu trúc nâng cao có trạng thái; lỗi theo loại; gợi ý có ví dụ.

---

## 7. Cải thiện câu — "Huấn luyện viên nâng câu" `P1`

Thay tư duy "phát câu hoàn hảo" bằng **dạy kỹ thuật nâng cấp + bắt luyện lại**.

**Trigger (khu kết quả):** 3 nút mục tiêu — **Nâng từ vựng** (🔵) · **Đa dạng ngữ pháp** (🟠) · **Phát triển ý sâu** (🟢). Bấm → kết quả ở cột AI hỗ trợ.

**Card coach (rail, gọn):**
- "Câu tốt hơn · [mục tiêu]" + **hướng tới band kế tiếp** (scaffold, không nói band cố định kiểu cũ).
- Câu nâng cấp với **chỗ thay đổi tô màu**; bấm → "vì sao + kỹ thuật".
- Nhãn nhóm đã nâng (Từ vựng/Ngữ pháp/Collocation/Liên kết).
- Nút **"⛶ Bản đầy đủ"** + **"🎙 Thử nói lại"**.

**Panel rộng (full coach, 2 cột):**
- **Trái:** câu nâng cấp dạng **diff tô màu theo tiêu chí** (🔵 từ vựng · 🟠 ngữ pháp · 🟢 liên kết/ý · 🟣 collocation), bấm chỗ tô màu → vì sao + kỹ thuật; **nghe mẫu** (audio); **dịch nghĩa**.
- **Phải:** **"Bạn đã nâng cấp gì"** (liệt kê theo tiêu chí); **"Cụm đáng học"** (mỗi cụm có nút **Lưu** vào flashcard/từ vựng); nút **"🎙 Thử nói lại theo gợi ý"** (đóng vòng luyện).

**Nghiệm thu:**
- [ ] 3 mục tiêu; mỗi mục cho câu nâng cấp + diff bấm xem kỹ thuật.
- [ ] Có "đã nâng gì" theo tiêu chí + "cụm đáng học" lưu được (toast xác nhận).
- [ ] Có nghe mẫu + "thử nói lại".

---

## 8. Edge case khi user nói `P0–P1`

Khi gặp sự cố → **thay khu kết quả bằng card lỗi chuyên biệt** (icon + tiêu đề + giải thích + hành động).

| Mã | Tình huống | Hành vi | Ưu tiên |
|---|---|---|---|
| A1 | Mic bị từ chối / chưa cấp quyền | Card đỏ + **"Hướng dẫn cấp quyền"** + "Thử lại"; không có transcript | P0 |
| A2 | Không có mic / trình duyệt không hỗ trợ | Card đỏ + "Thử lại" | P0 |
| B1 | Ghi quá ngắn | Card "Đoạn ghi quá ngắn (Xs)…" + "Ghi âm lại" | P0 |
| B2 | Im lặng / không có giọng | Card "Không nghe thấy giọng nói" + "Ghi âm lại" | P1 |
| B3 | Tạp âm/nhiễu lớn | Card "Âm thanh bị nhiễu" + "Ghi âm lại" | P1 |
| B4 | Ghi quá dài | **Tự dừng ở giới hạn** (cảnh báo đếm ngược) + **vẫn chấm**; nhãn "Đã tự dừng ở giới hạn" | P1 |
| B5 | Nói không phải tiếng Anh | Card "Có vẻ bạn chưa nói tiếng Anh" + "Ghi âm lại" | P1 |
| C1 | Lệch đề / off-topic | Card **"Off-Topic: Đọc kỹ câu hỏi nha"** nêu rõ lệch về chủ đề gì; **vẫn hiện transcript**; chỉ cho **ghi lại** | P1 |

> Giới hạn thời lượng (B4) đề xuất: Part 1 ~90s · Part 2 ~120s · Part 3 ~90s (cần chốt — §11). Demo dùng 15s.

**Nghiệm thu:**
- [ ] Mỗi tình huống hiện đúng card (QA dùng ô "Demo edge case").
- [ ] A1/A2 không tạo transcript; C1 vẫn hiện transcript; B4 tự dừng & vẫn chấm.

---

## 9. Kích thước & typography (compact) `P1`
Siết kích thước/cỡ chữ để mật độ thông tin cao mà vẫn dễ đọc (tham chiếu file demo): điểm tổng gọn, body ~13px, padding/bo góc/khoảng cách giảm hợp lý.

**Nghiệm thu:** các khu (kết quả, rail, card lỗi, panel rộng, drill) gọn như demo, không tràn/cắt chữ.

---

## 10. Khác biệt theo Part
- **Part 1 / Part 3:** áp dụng đầy đủ.
- **Part 2 (cue card):** câu hỏi nhiều dòng ("You should say…"), có thời gian chuẩn bị; kết quả/phân tích/cải thiện/edge case áp dụng tương tự; giữ định dạng đề nhiều dòng.

---

## 11. Yêu cầu dữ liệu / backend

Nhờ tech lead xác nhận khả thi & nguồn dữ liệu:

1. **Câu đã sửa (§3):** danh sách lỗi từ vựng/ngữ pháp + `vị trí`, `bản sửa`, `loại lỗi`, `giải thích`.
2. **Điểm theo từng lượt (§5):** lưu band tổng + 4 tiêu chí mỗi lượt (tính delta + mini trend).
3. **Phát âm (§6.1):** điểm **% từng từ** (phân band + xếp hạng lỗi), **phoneme-level** (IPA "bạn nói" + MẪU/BẠN), **timestamp từ** (nghe lại/cắt đoạn), gom lỗi theo **phoneme**, tín hiệu **trọng âm + ngữ điệu** (prosody). Phần lớn **đã có sẵn trong Azure Pronunciation Assessment**.
4. **Trôi chảy (§6.2):** **tốc độ wpm**, **vị trí & độ dài quãng ngắt** (phân 3 mức), filler; **đánh giá coherence** + liên từ đã dùng.
5. **Từ vựng (§6.3):** **CEFR level từng từ** (tô màu + phân bố) + **từ điển paraphrase theo cấp độ**.
6. **Ngữ pháp (§6.4):** phân loại **câu đơn/phức** + **câu không lỗi**, **cấu trúc nâng cao đã thử** (đúng/sai), **thống kê lỗi theo loại**.
7. **Cải thiện câu (§7):** rewrite **kèm metadata thay đổi** (vị trí + loại + lý do/kỹ thuật) để dựng diff; "đã nâng gì" & "cụm đáng học" suy ra từ metadata (không thêm lần gọi model); **lưu flashcard** vào từ vựng user.
8. **Audio/TTS:** nghe lại bản ghi user (timestamp), nghe **âm chuẩn**/nghe mẫu câu nâng cấp — nên **cache TTS theo từ/câu** hoặc dùng SpeechSynthesis client-side.
9. **Edge case (§8):** tín hiệu thật cho im lặng/nhiễu/sai ngôn ngữ/lệch đề (không chỉ theo message lỗi); **max duration** + auto-stop; xử lý **quyền mic**.

---

## 12. Hiệu năng & ranh giới xử lý (quan trọng)

Mức độ chi tiết của **giao diện** không làm chậm output, vì độ trễ chính là **số lần gọi model** — phần phân tích chi tiết **không thêm lần gọi nào**. Ranh giới cần tuân thủ:

| Loại | Khi nào | Ví dụ |
|---|---|---|
| **Tính 1 lần lúc chấm** | Trong response chấm (tái dùng output Azure/LLM đã có) | điểm từng từ, phoneme, CEFR, ngắt nghỉ, metadata rewrite |
| **Tra tĩnh** | Tức thì, không gọi model | hướng dẫn khẩu hình theo âm, từ điển paraphrase |
| **Lazy-load** | Chỉ khi user tương tác | mở panel rộng/drill; bấm 🔊 nghe âm chuẩn/nghe lại; nghe mẫu câu nâng cấp |

**Tránh:** gọi LLM cho từng từ; sinh TTS đồng bộ cho mọi từ lúc chấm; cắt audio từng từ ở server. **Progressive reveal** (band + phát âm trả trước, vocab/grammar stream sau) giữ nguyên.

---

## 13. Ưu tiên theo đợt

**Đợt 1 (P0):** §2 bố cục mới (bỏ tab/xem chi tiết) · §3 câu đã sửa + popup · §8 A1/A2/B1.

**Đợt 2 (P1):** §5 so sánh lượt (delta/động viên/mini trend/auto-collapse) · §6 phân tích 4 tiêu chí (rail + panel rộng) · §7 coach cải thiện câu · §8 B2/B3/B4/B5/C1 · §9 compact.

**Đợt 3 (P2):** §6.1 luyện âm từng từ (drill) · các tinh chỉnh nâng cao (per-segment audio, lưu flashcard nâng cao).

---

## 14. Tham chiếu
- **Visual spec (đầy đủ tính năng):** [`practice-flow-optimized.html`](./practice-flow-optimized.html) — ô "Demo edge case" để test A1–C1; ghi nhiều lượt xem delta + mini trend; bấm từng thẻ tiêu chí xem phân tích; "⛶ Bản đầy đủ" mở panel rộng; thử 3 nút Cải thiện câu.
- **Baseline (before):** [`practice-flow-mockup.html`](./practice-flow-mockup.html).
- **Doc trước (bị bao trùm):** [`PRACTICE_FLOW_UPDATE_REQUIREMENT.md`](./PRACTICE_FLOW_UPDATE_REQUIREMENT.md).
