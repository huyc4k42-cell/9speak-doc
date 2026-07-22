# Mock Test V2 — §5 Xử lý Edge case khi thi — v0.3 (chốt kỹ thuật chi tiết B1/B2/D1 qua grill-me phiên 2)

> Ngày cập nhật: 2026-07-07 · Trạng thái: Chính thức · Đối chiếu [`useExamRecordingController.ts`](../../../../features/mock-test/hooks/useExamRecordingController.ts), [`useMockExamRecorder.ts`](../../../../features/shared/hooks/useMockExamRecorder.ts) · Thay thế toàn bộ nội dung bản v0.1 (2026-06-20, chuyển thể từ Google Doc "Mocktest V2") · v0.2 → v0.3: chốt ngưỡng kỹ thuật B1/B2, cơ chế detect D1, thứ tự ưu tiên build, số lần nhắc — qua grill-me phiên thứ 2 (2026-07-07).

## Đổi log so với v0.1

| # | Thay đổi | Lý do |
|---|---|---|
| 1 | Xác nhận qua code: **B1 (nói quá ngắn), B2 (im lặng), B3 (tạp âm), B5 (sai ngôn ngữ) là 0% xây dựng** — không có logic phát hiện độ dài/im lặng/nhiễu/ngôn ngữ ở bất kỳ đâu trong `useExamRecordingController.ts` hay pipeline chấm điểm; không có hành vi "giám khảo nhắc trong vai" nào tồn tại. Hôm nay, các tình huống này chỉ dẫn tới điểm Azure/OpenAI thấp, **không có tín hiệu nào trong app** | Đối chiếu trực tiếp `useExamRecordingController.ts` — không tìm thấy detection logic cho length/silence/noise/language |
| 2 | Giữ nguyên toàn bộ phạm vi đặc tả B1/B2/B3/B5/D1/C1 làm **spec đích** (không descope B3/B5 dù khó về kỹ thuật) — đánh dấu rõ "chưa build, cần đầu tư hạ tầng nhận diện" thay vì cắt bớt | Quyết định đã chốt qua grill-me trong phiên làm việc file #04 (2026-07-07), áp dụng lại cho file này vì file #04 khi đó đã nêu "toàn bộ #05 (Edge case B/A/D/C) hoàn toàn chưa xây" |
| 3 | Xác nhận qua code: **A1/A2 (mic bị từ chối/không có thiết bị) CHỈ được xử lý một phần** — `useMockExamRecorder.ts:187-191` bắt lỗi `getUserMedia()` khi bắt đầu ghi (set `status="error"` + thông báo chung "Không thể truy cập microphone.", re-throw lên `useExamRecordingController.ts` ~dòng 197-199 để hiện toast/error card). Xử lý này **chỉ áp dụng tại thời điểm bắt đầu ghi** — không có bất kỳ listener nào (`track.onended`/`onmute`/`stream.oninactive`) cho việc mất quyền/mất thiết bị xảy ra **trong lúc đang ghi** | Đối chiếu `useMockExamRecorder.ts` + `useExamRecordingController.ts` |
| 4 | **D1 (mất kết nối mic giữa lúc ghi) được nâng thành mức độ nghiêm trọng CAO HƠN nhóm B** — đánh dấu riêng bằng marker "🔴 Rủi ro dữ liệu (ưu tiên cao hơn nhóm B)", không xếp chung 1 hàng ưu tiên với B1-B5 | Chốt qua grill-me (2026-07-07): D1 là rủi ro toàn vẹn dữ liệu (âm thầm nộp bài hỏng với tín hiệu giả là thành công), khác về bản chất với B1-B5 (chỉ thiếu phản hồi/hướng dẫn, không giả mạo trạng thái) — xem §5.8 |
| 5 | Xác nhận qua code: **D1 chưa build và đang có nguy cơ dữ liệu thật, không chỉ là "chưa build" đơn thuần** — không có `track.onended`/`onmute` (grep xác nhận không tồn tại trong `useMockExamRecorder.ts`); `ScriptProcessorNode.onaudioprocess` (`useMockExamRecorder.ts:159-162`) tiếp tục đẩy bất kỳ buffer nào nó nhận được — kể cả im lặng/dữ liệu hỏng — vào bản ghi mà **không phát hiện track đã chết**. Hệ quả: mic rớt giữa câu trả lời → bản ghi âm thầm tiếp tục thu dữ liệu rỗng/hỏng → user dừng và nộp bình thường, không có cảnh báo → hệ thống coi là nộp bài thành công (điểm thấp bất thường xuất hiện sau, không rõ lý do) | Đối chiếu `useMockExamRecorder.ts:159-162` (callback ghi âm) + grep xác nhận không có track lifecycle listener |
| 6 | Tách rõ **C1 (lỗi mạng) thành 2 điểm lỗi khác nhau, không gộp chung**: (a) chấm điểm live theo từng câu/Part **trong lúc thi** — đã có xử lý (catch lỗi, giữ bản ghi, toast báo lỗi, đánh dấu câu failed) nhưng **không có nút "Thử lại" tức thời**; (b) finalize vocab/grammar ở giai đoạn Processing (#06) — có cơ chế retry riêng, đã mô tả ở #06/#08, không lặp lại chi tiết ở đây | Đối chiếu `useExamRecordingController.ts:284-325` (question-level) + `:348-391` (part-level) + `~240-254` (upload R2 presigned trước khi chấm) — 2 luồng có trạng thái triển khai khác nhau, gộp chung sẽ gây hiểu lầm |
| 7 | **Xác nhận qua grill-me: KHÔNG thêm nút "Thử lại" tức thời cho C1 ở tầng nộp-chấm từng câu trong lúc thi** — giữ nguyên hành vi hiện tại (toast báo lỗi + tự động cho qua câu tiếp theo) | Chốt qua grill-me (2026-07-07): đã có cơ chế retry theo Part ở Report (#08, BR-05) xử lý việc này sau; thêm 1 lớp retry tức thời giữa lúc thi sẽ phá vỡ chủ đích mô phỏng áp lực phòng thi thật (không cho dừng lại sửa giữa chừng) — xem §5.8 |
| 8 | Chuẩn hoá cấu trúc tài liệu theo khung đang áp dụng cho #01/#03/#04: thêm mục **Goal / Non-functional Requirements / User Types**, đổi mục 6 "Tiêu chí chấp nhận" từ bảng phẳng sang **định dạng Given/When/Then theo từng AC**, đổi số mục từ 1-7 sang 5.1-5.8, thêm **Đổi log** và **Câu hỏi mở — đã chốt** | Chuẩn hoá tài liệu theo yêu cầu product owner, đồng bộ với #01/#03/#04 |
| 9 | **Chốt cơ chế phát hiện D1**: kết hợp event-based (`track.onended`/`track.onmute`/`stream.oninactive`) làm tầng chính + energy polling làm tầng phụ giám sát. **Chỉ event-based mới được trigger D1** — energy polling KHÔNG dùng để trigger (tránh false positive khi user im lặng suy nghĩ). Mic "zombie" (connected nhưng im lặng) chấp nhận là edge case hiếm, không cố detect | Grill-me phiên 2 (2026-07-07) — xem §5.8 |
| 10 | **Chốt hành vi khi D1 phát hiện**: dừng ghi ngay lập tức + hiển thị card lỗi "Mất kết nối microphone". Không dùng toast, không dùng banner. Card xuất hiện đúng vị trí trong phòng thi | Grill-me phiên 2 (2026-07-07) — xem §5.8 |
| 11 | **Chốt xử lý audio khi D1**: huỷ toàn bộ phần audio đang ghi, user thu lại câu trả lời từ đầu. Không giữ partial audio | Grill-me phiên 2 (2026-07-07) — xem §5.8 |
| 12 | **Chốt lượt chấm khi D1**: không hoàn lượt tự động. D1 chặn trước khi nộp → lượt chưa bị trừ trong flow chuẩn. Edge case mic rớt đúng lúc nộp câu cuối → ghi nhận lỗi, hoàn lượt thủ công qua support | Grill-me phiên 2 (2026-07-07) — xem §5.8 |
| 13 | **Chốt thứ tự ưu tiên build nhóm B**: B1 + B2 trước (Web Audio API đang có sẵn). B3 + B5 là roadmap dài hạn, không có action sprint hiện tại | Grill-me phiên 2 (2026-07-07) — xem §5.8 |
| 14 | **Chốt ngưỡng B1**: audio duration < 3 giây → trigger B1. Chỉ bắt case bấm nhầm/nói vài từ, không bắt câu trả lời cụt có chủ đích. Phát hiện bằng Web Audio API (đang có sẵn) | Grill-me phiên 2 (2026-07-07) — xem §5.8 |
| 15 | **Chốt cách detect B2**: phát hiện post-hoc sau khi nộp, không real-time. Toàn bộ bản ghi có energy RMS dưới ngưỡng tối thiểu (không có tiếng nói nào) → trigger B2. Đơn giản, không false positive khi user im lặng ngắn giữa câu | Grill-me phiên 2 (2026-07-07) — xem §5.8 |
| 16 | **Chốt số lần nhắc B1/B2**: giám khảo nhắc lại tối đa **1 lần**. Lần 2 vẫn B1/B2 → bỏ qua câu, chuyển tiếp như bình thường | Grill-me phiên 2 (2026-07-07) — xem §5.8 |
| 17 | **B3/B5 hoàn toàn defer**: không có action sprint hiện tại. B5 đặc biệt KHÔNG dùng Azure post-hoc để detect (tránh UX lửng lơ — cảnh báo trong report sau khi đã mất lượt, không giúp được gì cho user) | Grill-me phiên 2 (2026-07-07) — xem §5.8 |

---

## Goal

**Business/Product Goal:** Giữ trải nghiệm thi chân thực khi gặp sự cố khi nói (giám khảo xử lý "trong vai" thay vì bung lỗi kỹ thuật) đồng thời bảo vệ tính toàn vẹn dữ liệu khi gặp sự cố thiết bị/mạng — đặc biệt không để hệ thống âm thầm chấp nhận một bài nộp có dữ liệu hỏng mà báo hiệu "thành công".

**User Benefits:**
- Thí sinh gặp sự cố giọng nói (nói ngắn, im lặng, nhiễu, sai ngôn ngữ): (đích) được giám khảo nhắc ngay trong vai, không bị gián đoạn cảm giác thi thật bằng thông báo lỗi hệ thống.
- Thí sinh gặp sự cố thiết bị (mic bị từ chối/mất khi bắt đầu ghi): đã được báo lỗi rõ ràng, không bị mất bài đã ghi trước đó.
- Thí sinh gặp lỗi mạng khi chấm 1 câu: bản ghi không bị mất (đã upload trước khi chấm), có toast thông báo, và có đường lùi retry theo Part ở Report — dù không được chặn lại ngay lúc thi.
- Sản phẩm: tránh được rủi ro nghiêm trọng nhất trong nhóm edge case — mic rớt kết nối giữa lúc ghi mà hệ thống không hề biết, dẫn tới báo cáo điểm thấp không rõ nguyên nhân và mất niềm tin của user vào độ tin cậy của bài thi.

---

## 5.1 Tổng quan

**Mô tả ngắn:** Quy tắc xử lý các tình huống bất thường khi người dùng đang thi, tách 2 nhóm: **lỗi giọng nói** (giám khảo xử lý trong vai — Nhóm B) và **lỗi thiết bị/mạng** (báo lỗi hệ thống — Nhóm A/D/C). Đối chiếu code thật xác nhận: **toàn bộ Nhóm B (B1/B2/B3/B5) 0% xây dựng**; Nhóm A (A1/A2) chỉ xử lý một phần (chỉ tại thời điểm bắt đầu ghi); **D1 (mất mic giữa lúc ghi) 0% xây dựng và là rủi ro toàn vẹn dữ liệu nghiêm trọng hơn nhóm B**; C1 (lỗi mạng) có 2 điểm lỗi khác trạng thái triển khai.

**Mục đích / vấn đề giải quyết:** Giữ trải nghiệm thi chân thực mà vẫn không để người dùng mắc kẹt hoặc bị đánh lừa bởi tín hiệu giả khi gặp sự cố. Phân biệt rõ "đây là một phần của cuộc thi" (giám khảo xử lý trong vai) với "đây là lỗi kỹ thuật" (báo lỗi hệ thống) — và quan trọng hơn, đảm bảo hệ thống **không bao giờ báo "thành công" cho một bài nộp có dữ liệu hỏng**.

**Phạm vi:**
- Trong phạm vi: B1 (nói quá ngắn) / B2 (im lặng) / B3 (tạp âm) / B5 (sai ngôn ngữ) — lỗi giọng nói; A1/A2 (mic bị từ chối/không có thiết bị khi bắt đầu ghi) / D1 (mất kết nối mic giữa lúc ghi) — lỗi thiết bị; C1 (lỗi mạng khi chấm live từng câu/Part trong lúc thi).
- Ngoài phạm vi: happy path phòng thi (#04); mic-check fail ở màn chuẩn bị trước khi vào thi (#03, đã có 3 trạng thái lỗi riêng biệt `blocked`/`no-device`/`hardware-error`); lỗi mạng ở giai đoạn finalize vocab/grammar tại Processing (#06) và cơ chế retry theo Part ở Report (#08, BR-05) — 2 mục này đã mô tả chi tiết ở tài liệu tương ứng, chỉ cross-reference ở đây.

**Đối tượng người dùng & bên liên quan:** Học viên; Product (kịch bản giám khảo); Dev FE (`useExamRecordingController.ts`, `useMockExamRecorder.ts`); QA (kịch bản lỗi).

---

## 5.2 Yêu cầu chức năng

> **Callout — Nhóm B (B1/B2/B3/B5) hoàn toàn CHƯA XÂY DỰNG:** Không có bất kỳ logic phát hiện độ dài câu trả lời, im lặng, tạp âm, hay ngôn ngữ ở đâu trong `useExamRecordingController.ts` hay pipeline chấm điểm. Không có hành vi "giám khảo nhắc trong vai" nào tồn tại trong code hôm nay. Hôm nay, bất kỳ tình huống nào trong B1/B2/B3/B5 xảy ra thực tế chỉ dẫn tới **điểm Azure/OpenAI thấp** — user không biết có gì bất thường cho tới khi thấy band thấp trong report.

| Mã | Yêu cầu | Đầu vào | Đầu ra | Ghi chú |
|---|---|---|---|---|
| FR-01 (đích, chưa build) | Khi câu trả lời **quá ngắn (B1)**, giám khảo phải nhắc trong vai và cho nói lại | Tín hiệu độ dài | Lời nhắc + re-record | Chưa build — không có phát hiện độ dài trong `useExamRecordingController.ts`. **Ngưỡng đã chốt: audio duration < 3 giây → trigger B1.** Phát hiện bằng Web Audio API (đang có sẵn). Tối đa **1 lần nhắc** — lần 2 vẫn B1 → bỏ qua câu, chuyển tiếp. Ưu tiên build: **B1 trong sprint gần** (Web Audio API có sẵn). |
| FR-02 (đích, chưa build) | Khi **im lặng/không có giọng (B2)**, giám khảo phải nhắc nói lại | Không có giọng | Lời nhắc + re-record | Chưa build — không có phát hiện im lặng. **Cách detect đã chốt: post-hoc sau khi nộp** (không real-time) — toàn bộ bản ghi có energy RMS dưới ngưỡng tối thiểu (không có tiếng nói nào) → trigger B2. Tối đa **1 lần nhắc** — lần 2 vẫn B2 → bỏ qua câu, chuyển tiếp. Ưu tiên build: **B2 trong sprint gần** (Web Audio API có sẵn). |
| FR-03 (đích, chưa build) | Khi **tạp âm lớn (B3)**, giám khảo phải nhắc thử lại | Nhiễu nền | Lời nhắc + re-record | Chưa build — không có phát hiện tạp âm. **Roadmap dài hạn — không có action sprint hiện tại.** Giữ nguyên trong spec (không descope) theo quyết định grill-me. |
| FR-04 (đích, chưa build) | Khi **trả lời sai ngôn ngữ (B5)**, giám khảo phải yêu cầu trả lời bằng tiếng Anh | Phát hiện ngôn ngữ | Lời nhắc + re-record | Chưa build — không có phát hiện ngôn ngữ. **Roadmap dài hạn — không có action sprint hiện tại. KHÔNG dùng Azure post-hoc để detect B5** (tránh UX lửng lơ: cảnh báo xuất hiện trong report sau khi đã mất lượt, không giúp được user). Giữ nguyên trong spec (không descope) theo quyết định grill-me. |
| FR-05 | Khi **mic bị từ chối/không có thiết bị (A1/A2) tại thời điểm bắt đầu ghi**, hệ thống báo lỗi + chặn ghi | Lỗi `getUserMedia()` khi `startRecording()` | Card/toast lỗi mic, chặn ghi | **Đã có, nhưng chỉ 1 phần** — `useMockExamRecorder.ts:187-191` set `status="error"` + thông báo chung "Không thể truy cập microphone.", re-throw lên `useExamRecordingController.ts` (~dòng 197-199). Chỉ bắt được lỗi xảy ra **đúng lúc bắt đầu ghi** |
| FR-06 (đích, chưa build) 🔴 Rủi ro dữ liệu (ưu tiên cao hơn nhóm B) | Khi **mất kết nối mic giữa lúc đang ghi (D1)**, hệ thống phải phát hiện được sự kiện track chết, dừng ghi ngay lập tức, hiển thị card lỗi "Mất kết nối microphone", huỷ toàn bộ audio đang ghi, cho thu lại từ đầu — **không được để bản ghi tiếp tục âm thầm, không dùng toast/banner** | Sự kiện mất track mic giữa lúc ghi | Card "Mất kết nối microphone" + dừng ghi ngay + huỷ audio + cho thu lại từ đầu | **Hoàn toàn chưa build.** Không có `track.onended`/`onmute`/`stream.oninactive` nào trong `useMockExamRecorder.ts` (xác nhận qua grep). `ScriptProcessorNode.onaudioprocess` (`useMockExamRecorder.ts:159-162`) tiếp tục đẩy buffer (kể cả im lặng/hỏng) vào bản ghi mà không phát hiện track đã chết → nộp bài "thành công giả". **Cơ chế phát hiện đã chốt**: event-based (`track.onended`/`track.onmute`/`stream.oninactive`) làm tầng chính — **chỉ event-based mới trigger D1**; energy polling làm tầng phụ giám sát nhưng KHÔNG trigger (tránh false positive khi user im lặng suy nghĩ). Mic "zombie" (connected nhưng im lặng) chấp nhận là edge case hiếm. **Lượt chấm**: D1 chặn trước khi nộp → lượt chưa bị trừ trong flow chuẩn, không hoàn tự động; edge case mic rớt đúng lúc nộp câu cuối → hoàn lượt thủ công qua support. |
| FR-07 | Khi **lỗi mạng lúc gửi chấm 1 câu/Part trong lúc thi (C1)**, hệ thống phải bắt lỗi, giữ bản ghi đã upload, báo lỗi qua toast, và đánh dấu câu/Part đó failed — **không chặn luồng thi, không có nút "Thử lại" tức thời** | Lỗi network khi gọi Azure/API chấm | Toast lỗi + `markQuestionFailed()` + tự động chuyển câu tiếp theo | **Đã có** — `useExamRecordingController.ts:284-325` (question-level), `:348-391` (part-level); audio đã lưu qua R2 presigned URL trước đó (~dòng 240-254). Quyết định chốt qua grill-me: **giữ nguyên**, không thêm retry tức thời — xem BR-05 |
| FR-08 | Khi lỗi giọng (Nhóm B), giám khảo phải **tạm bật phụ đề** để người dùng hiểu yêu cầu | Sự kiện nhóm B | Phụ đề tạm bật | (đích, chưa build) — phụ thuộc FR-01→04 được xây trước |

**Quy tắc nghiệp vụ (Business rules):**
- **BR-01:** **Nhóm B (lỗi giọng: B1/B2/B3/B5)** → (đích) giám khảo phản hồi *trong vai* (như thi thật), sau đó quay lại lượt trả lời **đúng câu đó**. Hôm nay: hoàn toàn chưa build, không có phản ứng nào. **Chi tiết đã chốt cho B1/B2 (ưu tiên build sớm):** B1 — ngưỡng < 3 giây (audio duration), phát hiện client-side bằng Web Audio API đang có sẵn; B2 — phát hiện post-hoc sau khi nộp, toàn bộ bản ghi có energy RMS dưới ngưỡng tối thiểu (không có tiếng nói nào). Cả B1 và B2 đều áp dụng **tối đa 1 lần nhắc** — lần đầu giám khảo nhắc và cho nói lại; lần 2 vẫn B1/B2 → bỏ qua câu, chuyển tiếp. B3/B5 là roadmap dài hạn, không có action sprint hiện tại.
- **BR-02:** **Nhóm A (thiết bị lúc bắt đầu ghi: A1/A2)** → báo lỗi hệ thống rõ ràng, không "đóng vai", chặn ghi cho tới khi khắc phục. Đã có, giới hạn ở thời điểm bắt đầu ghi (xem FR-05).
- **BR-03 🔴 (D1 — mức độ nghiêm trọng CAO HƠN Nhóm B, chốt qua grill-me):** Mất kết nối mic **giữa lúc đang ghi** là rủi ro toàn vẹn dữ liệu, không phải chỉ là "thiếu phản hồi/hướng dẫn" như Nhóm B. Hôm nay hệ thống không có cách nào phát hiện track mic đã chết giữa chừng — hệ quả là bản ghi tiếp tục thu dữ liệu rỗng/hỏng, user nộp bài bình thường, và hệ thống coi đó là một lượt nộp hợp lệ (chỉ lộ ra sau qua điểm số thấp bất thường trong report, không có lời giải thích). Đây là lý do D1 phải được đánh dấu và xử lý ưu tiên cao hơn B1-B5 trong lộ trình xây dựng. **Cơ chế phát hiện đã chốt (grill-me phiên 2, 2026-07-07)**: event-based (`track.onended`/`track.onmute`/`stream.oninactive`) làm tầng chính trigger D1; energy polling làm tầng phụ giám sát nhưng KHÔNG trigger (tránh false positive khi user im lặng suy nghĩ). **Hành vi khi phát hiện**: dừng ghi ngay + card lỗi "Mất kết nối microphone" (không toast, không banner). **Xử lý audio**: huỷ toàn bộ phần đang ghi, user thu lại từ đầu (không giữ partial audio). **Lượt chấm**: D1 chặn trước khi nộp → lượt chưa bị trừ trong flow chuẩn, không hoàn tự động; edge case mic rớt đúng lúc nộp câu cuối → hoàn lượt thủ công qua support — xem §5.8.
- **BR-04:** Lỗi mạng khi chấm live từng câu/Part trong lúc thi (C1) chỉ ảnh hưởng bước gửi chấm của câu/Part đó; bản ghi đã upload R2 trước đó không bị mất; câu bị đánh dấu failed và luồng thi **tiếp tục tự động sang câu kế tiếp, không chặn lại chờ retry**. Quyết định chốt qua grill-me (2026-07-07): **không thêm nút "Thử lại" tức thời** ở tầng này — cơ chế retry theo Part đã tồn tại ở Report (#08, BR-05: "Lỗi/Pending một Part → cho retry Part đó") xử lý việc này sau khi thi xong. Lỗi mạng ở giai đoạn finalize vocab/grammar (Processing #06) là một điểm lỗi khác, **đã có retry buttons riêng** — xem #06 và #08 (BR-05), không lặp lại chi tiết ở tài liệu này.

**Phân quyền theo vai trò:**

| Vai trò | Được làm gì |
|---|---|
| Thí sinh | Nhận thông báo và thực hiện hành động khắc phục (nói lại / thử lại thiết bị / thu lại) khi có sẵn; với C1 trong lúc thi, chỉ nhận toast báo lỗi và tiếp tục thi, không có hành động khắc phục ngay tại chỗ |

---

## 5.3 Cơ chế hoạt động

**Luồng người dùng (workflow):**
1. Trong lúc thi, hệ thống/giám khảo (đích) phát hiện tình huống bất thường.
2. Phân loại nhóm: B (lỗi giọng, đích chưa build) vs A/D (thiết bị) vs C (mạng).
3. Nhóm B (đích) → giám khảo nói (tạm bật phụ đề) → quay lại lượt trả lời câu đó. **Hôm nay: không xảy ra, hệ thống không phát hiện được các tình huống này.**
4. Nhóm A (lúc bắt đầu ghi) → card/toast lỗi hệ thống (đã có) → người dùng thử lại. Nhóm D1 (giữa lúc ghi, đích) → cần phát hiện track chết → dừng ghi → card lỗi → thu lại. **Hôm nay: D1 không được phát hiện, ghi âm tiếp tục âm thầm với dữ liệu hỏng.**
5. C1 (chấm live từng câu/Part trong lúc thi) → bắt lỗi mạng → giữ bản ghi đã upload → toast lỗi → đánh dấu câu/Part failed → tự động sang câu tiếp theo, không chờ retry tại chỗ.

**Use case chính:**
- **Bối cảnh (đích, minh hoạ mục tiêu sản phẩm — chưa build):** đang ở một câu Part 1, user trả lời quá ngắn (B1) → giám khảo "That was quite short. Could you tell me a little more?" → quay lại lượt → user nói đầy đủ → tiếp tục.
- **Kịch bản thực tế hôm nay (B1/B2/B3/B5):** user trả lời quá ngắn/im lặng/có nhiễu/sai ngôn ngữ → hệ thống không phát hiện gì → câu vẫn được chấm bình thường → điểm thấp xuất hiện trong report, không có giải thích.
- **Kịch bản thực tế hôm nay (D1, rủi ro cao nhất):** đang ghi thì mic bị rút/mất kết nối → `onaudioprocess` tiếp tục đẩy buffer (im lặng/hỏng) vào bản ghi → user dừng và nộp như bình thường, không thấy cảnh báo nào → hệ thống chấm bài với dữ liệu hỏng, coi là nộp thành công → user chỉ phát hiện qua điểm số thấp bất thường trong report, không hiểu lý do.
- **Kịch bản thực tế hôm nay (C1):** lỗi mạng khi gọi Azure chấm câu 3 Part 1 → `useExamRecordingController.ts` bắt lỗi, audio đã lưu R2 từ trước không mất → toast "Part 1 - câu 3 chấm điểm thất bại." → câu được đánh dấu failed → hệ thống tự chuyển sang câu tiếp theo, không có nút "Thử lại" xuất hiện tại chỗ → user chỉ có thể retry cả Part đó sau, ở Report (#08).

**Luồng dữ liệu:**
- (Đích, chưa build) Tín hiệu từ ghi âm/đánh giá nhanh (độ dài, có giọng, nhiễu, ngôn ngữ) → bộ phân loại → render phản hồi (giám khảo trong vai / card hệ thống). Hạ tầng này **không tồn tại** hôm nay.
- (Hôm nay, A1/A2) `getUserMedia()` reject khi `startRecording()` → `useMockExamRecorder.ts:187-191` set `status="error"` → re-throw → `useExamRecordingController.ts` (~197-199) hiện toast/error card.
- (Hôm nay, D1 — thiếu) `ScriptProcessorNode.onaudioprocess` (`useMockExamRecorder.ts:159-162`) nhận buffer từ track và đẩy vào bản ghi liên tục, không kiểm tra track còn sống hay không — đây chính là lỗ hổng.
- (Hôm nay, C1) Ghi âm → upload audio lên R2 qua presigned URL (~`useExamRecordingController.ts:240-254`) → gọi API chấm (Azure) cho câu/Part → nếu lỗi mạng, catch tại `:284-325` (question-level) / `:348-391` (part-level) → `markQuestionFailed()` + toast → tiếp tục câu kế.

**Tích hợp:** Phòng thi (#04) · Test mic (#03, đã có 3 trạng thái lỗi thiết bị riêng biệt trước khi vào thi) · Processing (#06, finalize vocab/grammar có retry riêng) · Report (#08, BR-05: retry theo Part) · (chưa có) dịch vụ phát hiện ngôn ngữ/nhiễu/im lặng/độ dài · (chưa có) track lifecycle listener cho mic.

*Sơ đồ phù hợp: decision flowchart phân loại Nhóm B (đích) vs A (đã có, giới hạn) vs D1 (đích, rủi ro cao) vs C1 (đã có, không retry tức thời) → hành vi tương ứng theo trạng thái build thật.*

---

## 5.4 Xử lý ngoại lệ và trạng thái

| Trường hợp | Hành vi mong đợi / thông báo | Trạng thái build |
|---|---|---|
| B1 — Nói quá ngắn | (Đích) Giám khảo: "That was quite short. Could you tell me a little more?" → nói lại | **Chưa build** — không có phát hiện độ dài; hôm nay chỉ ảnh hưởng điểm chấm |
| B2 — Im lặng | (Đích) "I'm sorry, I didn't quite catch that. Could you answer again?" → nói lại | **Chưa build** — không có phát hiện im lặng |
| B3 — Tạp âm | (Đích) "There seems to be some background noise. Could you try again?" → nói lại | **Chưa build** — không có phát hiện nhiễu; giữ nguyên trong spec theo quyết định grill-me (không descope) |
| B5 — Sai ngôn ngữ | (Đích) "Could you please answer in English?" → nói lại | **Chưa build** — không có phát hiện ngôn ngữ; giữ nguyên trong spec theo quyết định grill-me (không descope) |
| A1/A2 — Mic bị từ chối/không có thiết bị, **tại thời điểm bắt đầu ghi** | Card/toast "Không thể truy cập microphone." + chặn ghi | **Đã có, một phần** — `useMockExamRecorder.ts:187-191` + `useExamRecordingController.ts` ~197-199; chỉ bắt được lỗi đúng lúc `startRecording()` |
| D1 — Mất kết nối mic **giữa lúc đang ghi** | (Đích) Phát hiện track chết → dừng ghi → card "Mất kết nối micro" → chặn nộp dữ liệu hỏng → cho thu lại | 🔴 **Chưa build — rủi ro dữ liệu, ưu tiên cao hơn nhóm B.** Không có `track.onended`/`onmute`/`stream.oninactive`; `onaudioprocess` (`useMockExamRecorder.ts:159-162`) tiếp tục ghi buffer hỏng/rỗng mà không phát hiện; hệ thống coi bài nộp là thành công dù dữ liệu đã hỏng |
| C1 — Lỗi mạng khi chấm live 1 câu/Part **trong lúc thi** | Toast lỗi (vd "Part 1 - câu 3 chấm điểm thất bại.") + giữ bản ghi đã upload + đánh dấu câu/Part failed + tự động sang câu tiếp theo | **Đã có** (`useExamRecordingController.ts:284-325` question-level, `:348-391` part-level, upload R2 presigned ~240-254). **Không có nút "Thử lại" tức thời** — chốt giữ nguyên qua grill-me (BR-04); retry thật sự nằm ở Report #08 BR-05 (theo Part, sau khi thi xong) |
| C1 — Lỗi mạng ở finalize vocab/grammar (Processing, sau khi thi xong) | Có nút retry | **Đã có, mô tả chi tiết ở #06 và #08 (BR-05)** — không lặp lại ở tài liệu này; đây là điểm lỗi khác, upstream/downstream khác với C1 trong lúc thi |

**Các trạng thái giao diện:** (đích, chưa build) giám khảo nhắc kèm phụ đề tạm bật cho Nhóm B · card lỗi hệ thống (rose) cho A1/A2 (đã có) · (đích, chưa build, ưu tiên cao) card "Mất kết nối micro" cho D1 · toast lỗi + tự động chuyển câu cho C1 (đã có).

---

## 5.5 Yêu cầu giao diện (UI/UX)

- Mockup tham khảo: `mocktest-flow-mockup.html` — `examinerRetry()`, `micError()`, `procError()`, ô **"Demo edge case"** (normal/A1/A2/B1/B2/B3/B5/D1/C1). Mockup mô tả UI đích cho toàn bộ các case; **không phản ánh trạng thái build thật của code hôm nay** (mockup luôn "thành công"/mô phỏng, không phải hành vi thật).
- Nhóm B (đích, chưa build): giám khảo nói (phụ đề tạm bật).
- Nhóm A (đã có, một phần): card/toast rose + thông báo chung "Không thể truy cập microphone." — chưa phân biệt A1 (từ chối quyền) vs A2 (không có thiết bị) bằng copy/màu riêng như đã làm ở #03 (nơi có 3 trạng thái `blocked`/`no-device`/`hardware-error` phân biệt rõ).
- D1 (đích, chưa build, 🔴 ưu tiên cao): card rose + icon mic-off, copy rõ ràng kiểu "Mất kết nối microphone — vui lòng thu lại câu trả lời" — cần chặn được việc nộp bản ghi hỏng, không chỉ là hiển thị cảnh báo sau khi đã nộp.
- C1 trong lúc thi (đã có): toast lỗi ngắn gọn theo Part/câu, không có modal chặn luồng, không có nút "Thử lại" trong toast.

---

## 5.6 Tiêu chí chấp nhận

**AC-01: Nhóm B chưa có phản ứng (hiện trạng, không phải đích)**
- **Given:** User trả lời quá ngắn (B1), im lặng (B2), có tạp âm lớn (B3), hoặc trả lời sai ngôn ngữ (B5) trong lúc thi
- **When:** User nộp câu trả lời đó
- **Then:** Hệ thống không hiển thị bất kỳ tín hiệu/lời nhắc nào trong lúc thi; câu vẫn được gửi chấm bình thường và chỉ thể hiện qua điểm số thấp trong report

**AC-02 (đích, chưa build): Giám khảo xử lý Nhóm B trong vai**
- **Given:** Hạ tầng phát hiện độ dài/im lặng/nhiễu/ngôn ngữ đã được xây dựng
- **When:** Hệ thống phát hiện B1/B2/B3/B5
- **Then:** Giám khảo phản hồi trong vai (tạm bật phụ đề) AND quay lại đúng lượt trả lời của câu đó, không tính là câu đã hoàn thành

**AC-03: Mic bị từ chối/không có thiết bị khi bắt đầu ghi (A1/A2)**
- **Given:** User bấm bắt đầu ghi cho 1 câu trong lúc thi
- **When:** `getUserMedia()` reject (quyền bị từ chối hoặc không tìm thấy thiết bị)
- **Then:** Hệ thống hiển thị card/toast lỗi "Không thể truy cập microphone." AND chặn việc ghi âm cho tới khi khắc phục

**AC-04 🔴 (đích, chưa build, ưu tiên cao hơn nhóm B): Phát hiện mất kết nối mic giữa lúc ghi (D1)**
- **Given:** User đang ghi âm câu trả lời và mic bị rút/mất kết nối giữa chừng
- **When:** Track mic chết trong lúc `onaudioprocess` đang chạy — event `track.onended` / `track.onmute` / `stream.oninactive` được bắn
- **Then:** Hệ thống phát hiện sự kiện event-based (không chờ energy polling), dừng ghi ngay lập tức, hiển thị card "Mất kết nối microphone" (không toast, không banner), huỷ toàn bộ phần audio đang ghi, cho user thu lại từ đầu AND chặn việc nộp bản ghi hỏng — lượt chấm chưa bị trừ (D1 chặn trước khi nộp)

**AC-05 (hiện trạng, không phải đích): D1 không được phát hiện hôm nay**
- **Given:** User đang ghi âm và mic mất kết nối giữa chừng (hiện trạng code hôm nay)
- **When:** `onaudioprocess` (`useMockExamRecorder.ts:159-162`) tiếp tục nhận buffer sau khi track đã chết
- **Then:** Hệ thống vẫn ghi nhận bản ghi (rỗng/hỏng) và cho phép nộp như bình thường, không có bất kỳ cảnh báo nào — đây là hành vi hiện tại cần được xây lại theo AC-04

**AC-06: Lỗi mạng khi chấm live 1 câu trong lúc thi (C1) — không có retry tức thời**
- **Given:** User đã nộp câu trả lời và audio đã upload thành công lên R2
- **When:** Gọi API chấm điểm (Azure) cho câu đó gặp lỗi mạng
- **Then:** Hệ thống bắt lỗi, hiển thị toast báo lỗi (vd "Part 1 - câu N chấm điểm thất bại."), đánh dấu câu đó failed, AND tự động chuyển sang câu tiếp theo — không hiển thị nút "Thử lại" tại chỗ, không chặn luồng thi

**AC-07: Bản ghi không mất khi lỗi mạng C1**
- **Given:** Lỗi mạng xảy ra khi gọi API chấm điểm cho 1 câu/Part
- **When:** Hệ thống xử lý lỗi đó
- **Then:** Bản ghi âm đã upload lên R2 trước đó không bị mất/xoá; câu/Part đó có thể được retry sau ở Report (#08, BR-05), không phải ngay lúc thi

**AC-08: Retry C1 chỉ khả dụng ở Report, không ở phòng thi**
- **Given:** 1 hoặc nhiều câu/Part bị đánh dấu failed do lỗi mạng trong lúc thi
- **When:** User xem lại các câu/Part đó
- **Then:** Không có nút "Thử lại" nào xuất hiện trong phòng thi ngay sau khi lỗi xảy ra; cơ chế retry theo Part chỉ xuất hiện ở Report (#08, BR-05) sau khi hoàn thành bài thi

**AC-09 (đích, chưa build): B1 — nói quá ngắn, tối đa 1 lần nhắc**
- **Given:** User đang thi và đã hoàn thành ghi âm 1 câu trả lời
- **When:** Độ dài audio < 3 giây (phát hiện client-side bằng Web Audio API)
- **Then:** Giám khảo nhắc trong vai "That was quite short. Could you tell me a little more?" (phụ đề tạm bật) AND user được thu lại câu đó. Nếu lần thu lại vẫn < 3 giây → bỏ qua câu, chuyển tiếp như bình thường (không nhắc lần 3)

**AC-10 (đích, chưa build): B2 — im lặng hoàn toàn, tối đa 1 lần nhắc**
- **Given:** User đã nộp 1 câu trả lời
- **When:** Post-hoc sau khi nộp: toàn bộ bản ghi có energy RMS dưới ngưỡng tối thiểu (không phát hiện tiếng nói nào trong toàn bản ghi)
- **Then:** Giám khảo nhắc trong vai "I'm sorry, I didn't quite catch that. Could you answer again?" (phụ đề tạm bật) AND user được thu lại câu đó. Nếu lần thu lại vẫn B2 → bỏ qua câu, chuyển tiếp như bình thường (không nhắc lần 3)

**AC-11 (đích, chưa build): D1 — edge case mic rớt đúng lúc nộp câu cuối**
- **Given:** User đang nộp câu cuối cùng của bài thi và mic mất kết nối đúng lúc đó
- **When:** Sự kiện D1 xảy ra đồng thời với việc nộp bài
- **Then:** Hệ thống ghi nhận lỗi, KHÔNG tự động hoàn lượt; user liên hệ support để được hoàn lượt thủ công

---

## Non-functional Requirements

**Toàn vẹn dữ liệu (quan trọng nhất cho D1):** hệ thống không được phép coi một bản ghi có dữ liệu hỏng/rỗng (do mất kết nối mic giữa lúc ghi) là một lượt nộp hợp lệ. Đây là yêu cầu ưu tiên cao hơn các yêu cầu UX khác trong tài liệu này — xem BR-03, §5.8.

**Độ tin cậy:** không mất dữ liệu bài làm đã ghi thành công trong mọi nhánh lỗi (A1/A2/C1); audio đã upload R2 trước khi gặp lỗi mạng chấm điểm phải được giữ nguyên để phục vụ retry sau.

**Trải nghiệm:** với Nhóm B (đích), phản hồi phải giữ được cảm giác "giám khảo thật" — không phá vỡ áp lực phòng thi bằng thông báo lỗi kỹ thuật. Với C1 trong lúc thi, chủ đích giữ nguyên áp lực thi thật (không cho dừng lại sửa ngay) — xem quyết định grill-me ở §5.8.

---

## User Types

Màn/luồng này không phân biệt theo gói (free/premium) — mọi thí sinh đang thi đều gặp cùng các nhánh lỗi tiềm năng như nhau. Không có "loại user" nào được đối xử khác biệt trong xử lý edge case.

---

## 5.7 Phần bổ sung

**Giả định (Assumptions):** Có tín hiệu/đánh giá nhanh để phát hiện độ dài, im lặng, nhiễu, ngôn ngữ — **hiện chưa tồn tại**, cần đầu tư hạ tầng mới hoàn toàn nếu triển khai Nhóm B (FR-01→04, FR-08).

**Phụ thuộc (Dependencies):**
- Phòng thi (#04) — nơi các sự kiện edge case xảy ra.
- Test mic (#03) — đã xử lý lỗi thiết bị **trước khi vào thi** với 3 trạng thái phân biệt (`blocked`/`no-device`/`hardware-error`); không xử lý sự cố xảy ra **giữa lúc thi** (đó là phạm vi D1 của tài liệu này).
- Processing (#06) và Report (#08, BR-05) — nơi xử lý retry cho lỗi mạng ở giai đoạn finalize vocab/grammar và retry theo Part; khác với C1 trong lúc thi được mô tả ở tài liệu này.
- **Hạ tầng phát hiện độ dài/im lặng/nhiễu/ngôn ngữ** — cần xây mới hoàn toàn cho Nhóm B; hiện không có bất kỳ thành phần nào trong `useExamRecordingController.ts` hay pipeline chấm điểm phục vụ việc này.
- **Track lifecycle listener cho mic** (`track.onended`/`onmute`/`stream.oninactive`) — cần xây mới cho D1; hiện không tồn tại trong `useMockExamRecorder.ts`.

**Câu hỏi mở / chưa chốt:**
- [ ] Cơ chế phát hiện ngôn ngữ cho B5 (client hay server, dùng dịch vụ nào) — vẫn mở, B5 là roadmap dài hạn chưa có timeline cụ thể. Azure post-hoc đã được loại khỏi danh sách lựa chọn (xem Đổi log #17 và §5.8).

---

## 5.8 Câu hỏi mở — đã chốt (2026-07-07, grill-me)

| Câu hỏi | Đã chốt tại |
|---|---|
| D1 (mất mic giữa lúc ghi) có nên xếp cùng mức ưu tiên với B1-B5 (đều "chưa build") không? | Không — **D1 phải được đánh dấu mức độ nghiêm trọng CAO HƠN B1-B5** trong tài liệu (không xếp chung 1 hàng ưu tiên), dùng marker riêng "🔴 Rủi ro dữ liệu (ưu tiên cao hơn nhóm B)". Lý do: D1 là rủi ro toàn vẹn dữ liệu (âm thầm nộp bài hỏng với tín hiệu giả là thành công), khác về bản chất với B1-B5 (chỉ là thiếu phản hồi/hướng dẫn, không giả mạo trạng thái). Im lặng nộp dữ liệu hỏng mà báo "thành công" gây mất niềm tin nghiêm trọng hơn, và user không thể tự truy vết nguyên nhân điểm thấp — xem BR-03, FR-06, AC-04/05 |
| Có nên thêm nút "Thử lại" tức thời cho lỗi mạng C1 ở tầng nộp-chấm từng câu trong lúc thi không? | Không — **giữ nguyên hành vi hiện tại** (báo lỗi qua toast + tự động cho qua câu tiếp theo, không chặn luồng thi). Lý do: đã có cơ chế retry theo Part ở Report (#08, BR-05) xử lý việc này sau; thêm 1 lớp retry tức thời giữa lúc thi sẽ phá vỡ chủ đích mô phỏng áp lực phòng thi thật (không cho dừng lại sửa giữa chừng) — xem BR-04, FR-07, AC-06/07/08 |
| Có nên descope B3 (nhiễu)/B5 (sai ngôn ngữ) khỏi phạm vi #05 vì khó về kỹ thuật không? | Không — giữ nguyên toàn bộ phạm vi đặc tả B1/B2/B3/B5/D1/C1 làm spec đích, chỉ đánh dấu rõ "chưa build, cần đầu tư hạ tầng nhận diện" thay vì cắt bớt. Quyết định này đã chốt trong phiên grill-me của file #04 (2026-07-07) và được áp dụng lại cho file này — xem Đổi log #2 |
| Cơ chế phát hiện D1: dùng event-based hay energy polling, hay kết hợp? | **Kết hợp, nhưng chỉ event-based mới trigger D1.** Tầng chính: `track.onended`/`track.onmute`/`stream.oninactive` — đây là tín hiệu trực tiếp track mic chết. Tầng phụ: energy polling chỉ để giám sát thêm, KHÔNG được dùng để trigger D1 vì dễ false positive khi user im lặng suy nghĩ. Mic "zombie" (vật lý connected nhưng im lặng liên tục) chấp nhận là edge case hiếm, không cố detect. — Grill-me phiên 2 (2026-07-07) |
| Hành vi UI khi phát hiện D1: dùng toast/banner/card? | **Card lỗi** "Mất kết nối microphone" — không dùng toast (quá nhỏ, thoáng qua), không dùng banner (không đủ rõ). Card xuất hiện tại vị trí trong phòng thi, chặn hành động nộp. — Grill-me phiên 2 (2026-07-07) |
| Xử lý audio đang ghi khi D1: giữ partial hay huỷ toàn bộ? | **Huỷ toàn bộ** phần audio đang ghi. User thu lại câu trả lời từ đầu. Không giữ partial audio (partial audio từ sau khi mic chết là dữ liệu rỗng/hỏng — không có giá trị, giữ lại chỉ gây nhầm lẫn). — Grill-me phiên 2 (2026-07-07) |
| Lượt chấm khi D1: có hoàn tự động không? | **Không hoàn tự động.** D1 chặn trước khi nộp → lượt chưa bị trừ trong flow chuẩn. Edge case mic rớt đúng lúc nộp câu cuối (race condition hiếm) → hệ thống ghi nhận lỗi, hoàn lượt thủ công qua support. — Grill-me phiên 2 (2026-07-07) |
| Thứ tự ưu tiên build nhóm B: B1/B2/B3/B5 theo thứ tự nào? | **B1 + B2 trước** (Web Audio API đang có sẵn — đủ để detect độ dài và energy RMS, không cần thêm hạ tầng ngoài). **B3 + B5 sau** — roadmap dài hạn, không có action sprint hiện tại (cần hạ tầng nhận diện nhiễu/ngôn ngữ phức tạp hơn nhiều). — Grill-me phiên 2 (2026-07-07) |
| Ngưỡng B1 là bao nhiêu giây? | **< 3 giây** (audio duration). Chỉ bắt case bấm nhầm/nói vài từ rồi dừng, không bắt câu trả lời cụt có chủ đích. Phát hiện client-side bằng Web Audio API. — Grill-me phiên 2 (2026-07-07) |
| Cách detect B2: real-time hay post-hoc? Ngưỡng thế nào? | **Post-hoc sau khi nộp** (không real-time — tránh nhầm khi user im lặng suy nghĩ giữa chừng). Điều kiện trigger: toàn bộ bản ghi có energy RMS dưới ngưỡng tối thiểu — không phát hiện tiếng nói nào trong toàn bộ thời lượng ghi. Đơn giản, ít false positive. — Grill-me phiên 2 (2026-07-07) |
| Số lần nhắc B1/B2: cho phép nói lại mấy lần? | **Tối đa 1 lần.** Lần đầu: giám khảo nhắc trong vai + cho nói lại. Lần 2 vẫn B1 hoặc B2 → bỏ qua câu, chuyển tiếp như bình thường. Không có lần nhắc thứ 3. — Grill-me phiên 2 (2026-07-07) |
| B5: có dùng Azure post-hoc để detect sai ngôn ngữ không? | **Không.** Dùng Azure post-hoc để detect B5 tạo UX lửng lơ — cảnh báo chỉ xuất hiện trong report sau khi đã mất lượt, không giúp được user sửa gì. B5 hoàn toàn defer sang roadmap dài hạn, chưa có timeline. — Grill-me phiên 2 (2026-07-07) |

Câu hỏi mở còn lại sau grill-me phiên 2: xem §5.7 (chỉ còn câu về cơ chế phát hiện ngôn ngữ B5 khi đến thời điểm roadmap).
