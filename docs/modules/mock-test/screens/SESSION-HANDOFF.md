# Mock Test PRD — Session Handoff (2026-07-07)

> File này tổng hợp toàn bộ công việc đã hoàn thành trong phiên làm việc 2026-07-07 và trạng thái bàn giao sang phiên tiếp theo. Đây là nguồn đọc trước duy nhất cần thiết trước khi bắt đầu phiên mới.

---

## 1. Tổng quan 9 file đặc tả

### Trạng thái hiện tại

| # | File | Version | Trạng thái | Đặc biệt |
|---|---|---|---|---|
| 01 | `01-chon-de-goi-y.md` | v2.0 + patch 2026-07-07 | ✅ Chính thức | Thêm Đổi log #13/14/15 về Part lẻ schema + CTA behavior + thứ tự ưu tiên |
| 02 | `02-cau-hoi-tu-chon.md` | v0.1 | ✅ (template cũ) | Tính năng chưa vào scope — tạm hoãn theo yêu cầu |
| 03 | `03-chuan-bi-kiem-tra-mic.md` | v0.3 | ✅ Chính thức | Đối chiếu code thật + grill-me phiên 1 |
| 04 | `04-phong-thi-giam-khao.md` | **v0.4** | ✅ Chính thức | Cập nhật 2026-07-07: Part lẻ schema + D1 cơ chế + redirect/history/script + thứ tự build |
| 05 | `05-edge-case-khi-thi.md` | **v0.3** | ✅ Chính thức | Cập nhật 2026-07-07: D1 event-based trigger + B1 <3s + B2 post-hoc + B3/B5 roadmap |
| 06 | `06-processing-cham-diem.md` | v0.2 | ✅ Chính thức | Màn "Chúc mừng" 3.6s + sửa FR-06 progressive reveal |
| 07 | `07-report-tong-quan.md` | v0.2 | ✅ Chính thức | Sub-score AI-generated, CEFR mapping xác nhận |
| 08 | `08-report-phan-hoi-chi-tiet.md` | v0.2 | ✅ Chính thức | Nguồn dữ liệu từng tiêu chí: rule/AI/pipeline/tĩnh |
| 09 | `09-report-hoc-thuat.md` | v0.2 | ✅ Chính thức | Tài liệu rubric/tham chiếu (không phải màn hình), bổ sung annotation đối chiếu code |

---

## 2. Nội dung & thay đổi từng file (tóm tắt)

### File 01 — Chọn đề & Gợi ý (v2.0 + patch 2026-07-07)

**Nội dung cốt lõi:** Layout 6 khối (thẻ trạng thái → banner Premium → gợi ý 3 thẻ → tab bar → filter bar → lưới đề). Card content model với tag (Hot/New/Forecast/Premium) và icon theo scope. Thuật toán gợi ý random đề chưa làm. Filter: Tất cả/Forecast/Đã làm/Chưa làm + dropdown kỳ.

**Thay đổi chính phiên này (Đổi log #13/14/15):**
- **#13:** Chốt schema Part lẻ — dùng chung session hiện tại, `visibleParts` 1 phần tử (vd `["part2"]`), bỏ hardcode tại `mock-exam-definition-from-catalog.ts:32`
- **#14:** Chốt CTA "Bắt đầu Part X" — sau khi build, tạo session Part lẻ và vào phòng thi; hiện đang broken (dẫn vào full 3 Part)
- **#15:** Thứ tự ưu tiên build: **D1 → Part lẻ → B1/B2**
- BR-13 cập nhật đầy đủ schema, `examMode="partX"`, 1 lượt chấm/Part
- §1.10 thêm 4 entries chốt Part lẻ

---

### File 02 — Câu hỏi tự chọn (v0.1 — KHÔNG thay đổi)

Vẫn theo template 7 mục cũ. Tính năng chưa vào scope dev. **Không cần xử lý.**

---

### File 03 — Chuẩn bị & Kiểm tra Mic (v0.3 — KHÔNG thay đổi phiên này)

Đã hoàn thiện phiên trước. 3 trạng thái lỗi mic (`blocked`/`no-device`/`hardware-error`) xác nhận qua code. Tất cả câu hỏi mở đã chốt.

---

### File 04 — Phòng thi — Giám khảo (v0.3 → **v0.4**)

**Nội dung cốt lõi:** Luồng thi full 3 Part + 4 chế độ (spec đích). Avatar giám khảo, sóng âm, phụ đề, stepper, ghi âm/nộp từng lượt. Resume tự động sau reload (không xác nhận). Huỷ 2 bước. Chặn huỷ sau khi đã trừ lượt chấm.

**Thay đổi phiên này (Đổi log #9–14):**
- **#9:** Cơ chế D1 — event-based (`track.onended`/`track.onmute`/`stream.oninactive`) là trigger chính; energy polling phụ nhưng không trigger
- **#10:** Schema Part lẻ — `visibleParts` 1 phần tử, không tạo schema mới, bỏ hardcode `mock-exam-definition-from-catalog.ts:32`
- **#11:** Redirect sau Part lẻ — thẳng về report (không màn trung gian)
- **#12:** History Part lẻ — ghi chung, label "Luyện Part X"
- **#13:** Script Part lẻ — thêm lời mở/đóng ngắn, không fork file kịch bản. Script: `"Let's focus on Part X today."` / `"That's the end of your Part X practice."`
- **#14:** Thứ tự ưu tiên build: D1 → Part lẻ → B1/B2
- Thêm BR-11/12/13, AC-15/16/17, callout §4.2 cập nhật schema

---

### File 05 — Edge case khi thi (v0.2 → **v0.3**)

**Nội dung cốt lõi:** Nhóm B (B1 nói ngắn / B2 im lặng / B3 nhiễu / B5 sai ngôn ngữ) + Nhóm A (A1/A2 mic bị từ chối) + D1 (mất mic giữa lúc ghi 🔴) + C1 (lỗi mạng). Xác nhận qua code: toàn bộ Nhóm B + D1 = 0% xây dựng.

**Thay đổi phiên này (Đổi log #9–17):**
- **#9:** D1 cơ chế detect — event-based làm trigger chính; energy polling phụ, KHÔNG trigger (tránh false positive khi user im lặng suy nghĩ)
- **#10:** D1 hành vi — dừng ghi + card lỗi (không toast, không banner)
- **#11:** D1 audio — huỷ toàn bộ, user thu lại từ đầu (không giữ partial)
- **#12:** D1 lượt chấm — không hoàn tự động; edge case mic rớt lúc nộp câu cuối → support thủ công
- **#13:** Thứ tự ưu tiên Nhóm B — B1+B2 trước (Web Audio API có sẵn); B3+B5 roadmap dài hạn
- **#14:** Ngưỡng B1 — `< 3 giây` audio duration → trigger. Tối đa 1 lần nhắc
- **#15:** B2 detect — post-hoc sau khi nộp, energy RMS toàn bản ghi dưới ngưỡng → trigger. Tối đa 1 lần nhắc
- **#16:** Số lần nhắc B1/B2 — tối đa 1 lần; lần 2 bỏ qua câu, chuyển tiếp
- **#17:** B3/B5 hoàn toàn defer; B5 KHÔNG dùng Azure post-hoc detect
- §5.7 bỏ 4 câu hỏi đã chốt, giữ 1 câu B5 language detection còn open
- §5.8 thêm 9 entries chốt từ grill-me phiên 2

---

### File 06 — Processing Chấm điểm (v0.2 — KHÔNG thay đổi phiên này)

Đã hoàn thiện phiên trước. Màn "Chúc mừng" 3.6s xác nhận qua code. Processing chờ ALL parts chấm xong (`isMockExamFullyGraded`) mới redirect — quyết định chủ đích, không phải thiếu sót.

---

### File 07 — Report Tổng quan (v0.2 — KHÔNG thay đổi phiên này)

Đã hoàn thiện phiên trước. Sub-score và descriptor AI-generated (không phải bảng tra tĩnh). CEFR mapping: ≤4.0→A2, 4.5–5.0→B1, 5.5–6.5→B2, 7.0–8.0→C1, ≥8.5→C2.

---

### File 08 — Report Phản hồi chi tiết (v0.2 — KHÔNG thay đổi phiên này)

Đã hoàn thiện phiên trước. Bảng §8.2 nguồn dữ liệu: Fluency (rule pipeline), Lexical (AI), Grammar (rule + AI hybrid), Pronunciation (Azure pipeline). Gap học thuật chi tiết ở `gap.md`.

---

### File 09 — Học thuật: Rubric + Band Descriptor (v0.2 — tạo mới phiên này)

**Nội dung cốt lõi:** Rubric/tham chiếu cho 4 tiêu chí IELTS Speaking (Fluency & Coherence / Lexical Resource / Grammatical Range & Accuracy / Pronunciation). Cấu trúc (a)–(f) mỗi tiêu chí: Đo gì / Sub-tiêu chí / Kỳ vọng theo band / Dấu hiệu & ngưỡng / Nội dung hiển thị / Ví dụ. Annotation đối chiếu code ✅/⚠️/❌.

**Divergences quan trọng đã sửa so với Google Doc gốc:**
- Pronunciation "good" threshold: doc nói 75%, code dùng **70%** → đã sửa
- CEFR mapping: doc bắt đầu từ B1, code có **A2 cho ≤4.0** → đã bổ sung
- Pause classification: doc mô tả đơn giản 3-tier (0.5s/1.0s), code là **context-aware heuristic** (`GOOD_PAUSE_MAX_MS=1200`, `GOOD_SENTENCE_END_MAX_MS=2000`, `BAD_EXTREME_PAUSE_MS=2500`) → đã cập nhật

**3 câu hỏi còn treo (chưa chốt):**
- Calibration ngưỡng (70% có phù hợp với tiếng Việt L1?)
- IPA chuẩn UK hay US?
- Phoneme taxonomy cho Vietnamese L1

---

## 3. Khoảng trống triển khai đã xác nhận (không phải lỗi tài liệu)

| Vấn đề | Mức độ | File spec | Trạng thái code | Ghi chú |
|---|---|---|---|---|
| **D1 — Mất mic giữa lúc ghi** | 🔴 Rủi ro dữ liệu | 05 §FR-06 / BR-03 | 0% build | `ScriptProcessorNode.onaudioprocess` tiếp tục ghi buffer rỗng, không có `track.onended`/`onmute` |
| **Part lẻ (examMode 1 Part)** | 🟡 Broken promise | 01 BR-13 / 04 FR-14/15 | 0% build | `visibleParts` hardcode `["part1","part2","part3"]` tại `mock-exam-definition-from-catalog.ts:32` |
| **B1 — Câu trả lời quá ngắn** | 🟡 UX | 05 §FR-01 / BR-01 | 0% build | Ngưỡng <3s, Web Audio API có sẵn |
| **B2 — Im lặng toàn bộ** | 🟡 UX | 05 §FR-02 / BR-01 | 0% build | Post-hoc detect, energy RMS toàn bản ghi |
| **B3 — Tạp âm lớn** | ⚪ Roadmap | 05 §FR-03 | 0% build | Roadmap dài hạn, không có action sprint |
| **B5 — Sai ngôn ngữ** | ⚪ Roadmap | 05 §FR-04 | 0% build | Roadmap dài hạn; cơ chế detect còn open |

---

## 4. Thứ tự ưu tiên build đã chốt

```
1. D1 (useMockExamRecorder.ts)
   → Thêm: track.onended / track.onmute / stream.oninactive listeners
   → Dừng ScriptProcessorNode, huỷ audio, hiện card lỗi
   → Energy polling chỉ giám sát (không trigger)

2. Part lẻ (mock-exam-definition-from-catalog.ts:32)
   → Bỏ hardcode visibleParts: ["part1","part2","part3"]
   → Đọc scope từ examMode đã chọn → visibleParts: ["partX"]
   → Ledger tự tính 1 lượt/Part visible (không cần thêm gì)
   → Thêm lời mở/đóng script giám khảo (không fork file)
   → History label: "Luyện Part X"

3. B1 + B2 (useExamRecordingController.ts)
   → B1: check audio duration < 3s sau khi nộp, Web Audio API
   → B2: check energy RMS toàn bản ghi post-hoc
   → Cả hai: giám khảo nhắc tối đa 1 lần, lần 2 bỏ qua câu
```

---

## 5. Quyết định chốt từ grill-me phiên 2 (2026-07-07)

| # | Vấn đề | Quyết định |
|---|---|---|
| D1-1 | Cơ chế detect | Event-based (`track.onended`/`onmute`/`stream.oninactive`) là trigger; energy polling phụ nhưng KHÔNG trigger |
| D1-2 | Hành vi khi detect | Dừng ghi ngay + card lỗi (không toast/banner) |
| D1-3 | Audio partial | Huỷ toàn bộ; user thu lại từ đầu |
| D1-4 | Lượt chấm | Không hoàn tự động; mic rớt lúc nộp câu cuối → support thủ công |
| D1-5 | Energy threshold | Chỉ event-based trigger; energy polling không trigger D1 |
| B-1 | Thứ tự build | B1+B2 sprint gần; B3+B5 roadmap dài hạn |
| B-2 | Ngưỡng B1/B2 | B1: < 3 giây audio duration; B2: toàn bản ghi im lặng (post-hoc RMS) |
| B-3 | Số lần nhắc | Tối đa 1 lần/câu; lần 2 bỏ qua câu |
| B-4 | B3/B5 post-hoc | B5 KHÔNG dùng Azure post-hoc; B3 defer hoàn toàn |
| P-1 | Schema Part lẻ | Dùng chung session, `visibleParts` 1 phần tử |
| P-2 | Redirect Part lẻ | Thẳng về report |
| P-3 | History Part lẻ | Ghi chung, label "Luyện Part X" |
| P-4 | Script Part lẻ | Thêm open/close lines; không fork |
| P-5 | Thứ tự build | D1 → Part lẻ → B1/B2 |

---

## 6. Công việc còn lại (chưa bắt đầu)

### Về tài liệu (PRD)
- [ ] Cập nhật `README.md`: version 04 và 05 vẫn đang ghi v0.3/v0.2 trong bảng, cần cập nhật lên v0.4/v0.3 và mô tả
- [ ] File 09: còn 3 câu hỏi treo về calibration ngưỡng, IPA standard, phoneme taxonomy — chờ quyết định product/content

### Về code (dev)
- [ ] **D1 fix** — `useMockExamRecorder.ts`: thêm `track.onended`/`track.onmute`/`stream.oninactive`, dừng `ScriptProcessorNode`, hiện card lỗi, huỷ audio, cho thu lại
- [ ] **Part lẻ** — `mock-exam-definition-from-catalog.ts:32`: bỏ hardcode `visibleParts`, đọc từ `examMode`; thêm open/close script; cập nhật history label
- [ ] **B1** — `useExamRecordingController.ts`: check audio duration < 3s sau nộp, giám khảo nhắc, re-record
- [ ] **B2** — `useExamRecordingController.ts`: check energy RMS post-hoc, giám khảo nhắc, re-record

### Về git
- [ ] **Chưa có commit nào** từ cả 2 phiên làm việc (tất cả thay đổi đang unstaged trên nhánh `v2-practice-mock-test`) — commit khi có yêu cầu từ product owner

---

## 7. File đã tạo/chỉnh sửa trong phiên này

| File | Hành động | Nội dung |
|---|---|---|
| `screens/09-report-hoc-thuat.md` | Tạo mới | Rubric 4 tiêu chí IELTS, annotation đối chiếu code |
| `screens/05-edge-case-khi-thi.md` | v0.2 → v0.3 | D1, B1/B2 ngưỡng, B3/B5 roadmap, 9 entries §5.8 |
| `screens/04-phong-thi-giam-khao.md` | v0.3 → v0.4 | Part lẻ schema/redirect/history/script, D1 cơ chế, BR-11/12/13, AC-15/16/17 |
| `screens/01-chon-de-goi-y.md` | patch | Đổi log #13/14/15, BR-13 cập nhật, §1.10 thêm 4 entries |
| `screens/README.md` | Cập nhật | Thêm file 09, cập nhật "Việc còn lại" |

---

## 8. Ngữ cảnh kỹ thuật quan trọng (dev cần biết trước khi code)

```
useMockExamRecorder.ts:159-162
  → ScriptProcessorNode.onaudioprocess — ghi buffer liên tục, KHÔNG check track status
  → Cần thêm: track.onended / track.onmute / stream.oninactive trước khi startRecording()

mock-exam-definition-from-catalog.ts:32
  → visibleParts: ["part1","part2","part3"]  ← hardcode cần xoá
  → Thay bằng: visibleParts: [examMode] (khi Part lẻ) hoặc ["part1","part2","part3"] (khi full)

mocktest-charge-ledger.ts:144 → isComplete
  → Ledger tự đếm đúng số lượt theo visibleParts.length — không cần thay đổi gì cho Part lẻ

features/shared/lib/cefr-mapping.ts
  → ≤4.0→A2, 4.5–5.0→B1, 5.5–6.5→B2, 7.0–8.0→C1, ≥8.5→C2 (đã xác nhận)

Pronunciation "good" threshold: 70% (không phải 75% như một số comment cũ)
```

---

*File này được tạo tự động để bàn giao phiên làm việc. Cập nhật lần cuối: 2026-07-07.*
