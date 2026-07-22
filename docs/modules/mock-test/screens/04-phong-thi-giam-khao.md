# Mock Test V2 — §4 Phòng thi — Giám khảo dẫn dắt — v0.4 (chốt thêm Part lẻ + cơ chế D1 qua grill-me phiên 2)

> Ngày cập nhật: 2026-07-07 · Trạng thái: Chính thức · Đối chiếu [`mock-exam-definition-from-catalog.ts`](../../../../features/mock-test/server/mock-exam-definition-from-catalog.ts), [`mock-exam-scripts.ts`](../../../../features/mock-test/lib/mock-exam-scripts.ts), [`MockExamFull.tsx`](../../../../features/mock-test/pages/exams/MockExamFull.tsx), [`MockExamSessionProvider.tsx`](../../../../features/mock-test/providers/MockExamSessionProvider.tsx), [`question-store.ts`](../../../../features/shared/server/question-store.ts), [`mocktest-charge-ledger.ts`](../../../../features/mock-test/server/mocktest-charge-ledger.ts), [`mock-exam-route-handlers.ts`](../../../../features/mock-test/server/mock-exam-route-handlers.ts) · Thay thế toàn bộ nội dung bản v0.2 (Google Doc "Mocktest V2" bản gốc v0.1 2026-06-20 + bổ sung v0.2 2026-06-24).
> **v0.4:** cập nhật 2026-07-07 — phiên grill-me thứ 2 — chốt cơ chế phát hiện D1, schema Part lẻ, redirect/history/script Part lẻ, thứ tự ưu tiên build.

## Đổi log so với v0.2

| # | Thay đổi | Lý do |
|---|---|---|
| 1 | **Giữ nguyên toàn bộ phạm vi đặc tả edge-case giọng nói** (#05: B1 nói ngắn, B2 im lặng, B3 nhiễu, B5 sai ngôn ngữ, D1 mất mic, C1 lỗi mạng) — **không** descope B3/B5 dù khó về kỹ thuật; đánh dấu rõ "spec đích, chưa build" thay vì cắt bớt | Chốt qua grill-me (2026-07-07): đây là PRD định hướng dev, việc descope nên là quyết định ở sprint planning/kế hoạch triển khai, không cắt ngay trong PRD chỉ vì độ khó kỹ thuật cao |
| 2 | Xác nhận qua code: **toàn bộ #05 (B/A/D/C) hoàn toàn chưa xây** — không có hạ tầng phát hiện im lặng/nhiễu/ngôn ngữ, giám khảo không phản ứng gì trong các tình huống này hôm nay (khác đánh giá cũ "gần xong") | Đối chiếu `useExamRecordingController.ts` + pipeline chấm điểm — không tìm thấy bất kỳ logic nào; lỗi hiện chỉ lộ diện qua điểm thấp hoặc trạng thái "failed" chung chung |
| 3 | **Resume sau reload giữa bài: tự động khôi phục lặng lẽ đúng Part/câu, KHÔNG hiện màn/hộp xác nhận "Tiếp tục từ Câu X?"** | Chốt qua grill-me: reload giữa bài là tình huống tai nạn (rớt mạng/refresh nhầm), không phải hành động chủ đích — thêm bước xác nhận chỉ gây khó chịu |
| 4 | **Thêm hộp thoại xác nhận 2 bước trước khi "Hủy bài thi"** (thay vì huỷ ngay 1 chạm như code hiện tại — `handleCancelExam` trong `MockExamFull.tsx`) | Chốt qua grill-me: huỷ là hành động phá huỷ tiến độ không thể hoàn tác (có thể đã ghi âm xong 2/3 Part), rủi ro thao tác nhầm quá cao nếu không xác nhận |
| 5 | Ghi nhận đúng hành vi code: **"Hủy bài thi" hiện tại CHỈ reset client-side** (`resetSession()` → `setSession(null)`), **không có API xoá/huỷ nào ở backend** — bản ghi session/kết quả từng Part đã chấm trên NineSpeak/Firestore vẫn tồn tại vĩnh viễn, chỉ thành "mồ côi" (không resume/truy cập lại được), **không** bị xoá thật; nhãn nút "Mất dữ liệu" cũ hơi gây hiểu lầm | Xác minh qua `app-bff-client.ts` — chỉ có `upsertExamSession`, không có endpoint DELETE/cancel/archive |
| 6 | **Chặn "Hủy bài thi" một khi tất cả Part đã `scoreDone`** (lượt chấm đã bị trừ theo ledger) — chỉ còn "Thoát" (giữ session, resume được, xem report sau) khả dụng | Chốt qua grill-me: một khi đã tốn lượt chấm, không có lý do sản phẩm nào để cho phép user tự làm mất quyền xem kết quả đã trả phí; cách rẻ nhất để tránh mất lượt oan mà không cần xây API xoá/refund |
| 7 | Bổ sung fact: **Part 1 có trần tổng thời lượng 5 phút, Part 3 có trần 7 phút** (`PART_DURATION_SECONDS`) — tài liệu cũ chỉ ghi "nhiều câu", chưa có ngân sách thời gian tổng | Đối chiếu `MockExamFull.tsx:174-178` |
| 8 | Thêm mục **Goal / User Types**, tách **Non-functional Requirements** thành mục riêng, thêm **Đổi log** và **Câu hỏi mở — đã chốt** theo khung chuẩn đang áp dụng cho #01/#03 | Chuẩn hoá tài liệu theo yêu cầu product owner |
| 9 | **Chốt cơ chế phát hiện D1 (mất mic):** event-based (`track.onended`/`track.onmute`/`stream.oninactive`) là trigger chính; energy polling là tầng phụ nhưng **không** trigger — chỉ để validate. Hành vi: dừng ghi + hiện card lỗi → user thu lại từ đầu (không giữ partial audio). Chi tiết kỹ thuật đầy đủ ở #05 | Chốt qua grill-me (2026-07-07, phiên 2) |
| 10 | **Chốt schema Part lẻ:** dùng chung schema session hiện tại — Part lẻ là 1 session với `visibleParts` chỉ có 1 phần tử (vd `["part2"]`). Chỉ cần bỏ hardcode `["part1","part2","part3"]` tại `mock-exam-definition-from-catalog.ts:32`. Ledger tự tính 1 lượt chấm cho 1 Part visible (không fork schema) | Chốt qua grill-me (2026-07-07, phiên 2) |
| 11 | **Chốt redirect sau khi xong Part lẻ:** redirect thẳng về report của Part đó (giống luồng full). Report chỉ hiển thị 1 Part. Không có màn trung gian | Chốt qua grill-me (2026-07-07, phiên 2) |
| 12 | **Chốt history Part lẻ:** ghi chung vào History với full exam. Card hiển thị label rõ loại: "Luyện Part 2" thay vì "Mock Test Full" | Chốt qua grill-me (2026-07-07, phiên 2) |
| 13 | **Chốt script giám khảo Part lẻ:** dùng script Part tương ứng từ `mock-exam-scripts.ts`, thêm lời mở/đóng ngắn riêng cho chế độ Part lẻ (vd "Let's focus on Part 2 today." và "That's the end of your Part 2 practice."). **Không fork** toàn bộ file kịch bản | Chốt qua grill-me (2026-07-07, phiên 2) |
| 14 | **Chốt thứ tự ưu tiên build:** D1 → Part lẻ → B1/B2. Part lẻ ưu tiên hơn B1/B2 vì CTA "Bắt đầu Part X" đang hiển thị nhưng dẫn vào full exam (broken promise với người dùng) | Chốt qua grill-me (2026-07-07, phiên 2) |

---

## Goal

**Business/Product Goal:** Tái hiện áp lực phòng thi IELTS Speaking thật (giám khảo dẫn dắt liên tục, không có đáp án mẫu, ghi âm-nộp từng lượt) để người học làm quen tâm thế thi thật, đồng thời **bảo vệ tiến độ và lượt chấm đã trả phí của người dùng** khỏi mất mát do tai nạn (reload giữa bài, bấm nhầm nút thoát/huỷ).

**User Benefits:**
- Thí sinh: trải nghiệm thi mô phỏng chân thực, có cảm xúc, giúp giảm bỡ ngỡ khi thi thật.
- Thí sinh gặp sự cố ngoài ý muốn (rớt mạng, refresh nhầm tab): không mất tiến độ — hệ thống tự khôi phục đúng vị trí, không cần thao tác gì thêm.
- Thí sinh cân nhắc thoát giữa bài: được cảnh báo rõ ràng qua 2 bước trước khi huỷ thật, tránh mất tiến độ chỉ vì bấm nhầm.
- Thí sinh đã tốn lượt chấm (đã hoàn thành đủ Part): được bảo vệ khỏi tự ý làm mất quyền xem report đã trả phí — hệ thống chủ động chặn hành động huỷ ở giai đoạn này.

---

## 4.1 Tổng quan

**Mô tả ngắn:** Trải nghiệm thi mô phỏng do **giám khảo ảo dẫn dắt**, người dùng nghe câu hỏi, ghi âm và nộp từng lượt; chấm sau khi xong. Đặc tả đích hỗ trợ **4 chế độ**: thi đầy đủ 3 Part hoặc luyện riêng Part 1 / Part 2 / Part 3 — **hiện tại code chỉ chạy chế độ full 3 Part** (xem callout §4.2/§4.4).

**Mục đích / vấn đề giải quyết:** Tạo cảm giác thi thật, chân thực và có cảm xúc (không phải "đọc câu hỏi rồi đọc đáp án"), giúp người học làm quen áp lực phòng thi IELTS Speaking; đồng thời không để tai nạn kỹ thuật (reload, thoát nhầm) làm mất tiến độ hay lượt chấm đã trả phí.

**Phạm vi:**
- Trong phạm vi: giao diện thi tối (focus), avatar giám khảo + sóng âm, phụ đề bật/tắt, stepper, 3 Part (Part 1 hỏi–đáp, Part 2 cue card + chuẩn bị + nói, Part 3 thảo luận), ghi âm/nộp từng lượt, giám khảo đóng bài → chuyển Processing, resume tự động sau reload, thoát/huỷ giữa bài (2 hành động phân biệt + hộp xác nhận huỷ), chạy theo chế độ (full/Part lẻ — đích).
- Ngoài phạm vi: chi tiết hạ tầng nhận diện edge case khi nói (#05 — mô tả hành vi đích ở đây nhưng triển khai kỹ thuật thuộc phạm vi khác), chấm điểm/processing (#06), report (#07–08).

**Đối tượng người dùng & bên liên quan:** Học viên; Product (kịch bản giám khảo); Dev FE (ghi âm, timer, state máy thi, persistence); QA.

---

## 4.2 Yêu cầu chức năng

> **Callout — Part-lẻ (`examMode`) chưa triển khai:** `mock-exam-definition-from-catalog.ts:32` luôn set `visibleParts: ["part1","part2","part3"]` — không có nhánh rẽ theo scope ở bất kỳ đâu trong code. Phòng thi hôm nay **luôn chạy đủ luồng full 3 Part**, bất kể scope Part lẻ được chọn ở màn Chọn đề (#01). FR-11/12, BR-05/06, AC-07→09 dưới đây là **spec đích, chưa build**.
>
> **Schema đã chốt (2026-07-07, phiên 2):** Part lẻ **không cần schema mới** — dùng chung session hiện tại, chỉ thay đổi `visibleParts` thành 1 phần tử (vd `["part2"]`). Thay đổi code cần thiết là bỏ hardcode `["part1","part2","part3"]` tại `mock-exam-definition-from-catalog.ts:32` và thay bằng logic đọc scope từ chế độ đã chọn. Ledger tự tính 1 lượt chấm cho 1 Part visible mà không cần thay đổi gì thêm.

| Mã | Yêu cầu | Đầu vào | Đầu ra | Ghi chú |
|---|---|---|---|---|
| FR-01 | Hiển thị avatar giám khảo và **phát lời thoại** theo Part/câu | Kịch bản đề (`mock-exam-scripts.ts`) | Lời thoại + avatar | Kịch bản hiện là **tĩnh, chỉ 1 biến thể** (`MockExamLeadIns`, `MockExamCommonScripts`), chưa có nhánh theo mode |
| FR-02 | Khi giám khảo nói, hiển thị **sóng âm vòng tròn** quanh avatar | Trạng thái "đang nói" | Hiệu ứng ping | Không dùng thanh sóng cạnh avatar |
| FR-03 | Cho **bật/tắt phụ đề**, mặc định **TẮT** | Toggle | Hiện/ẩn lời thoại | Cue card luôn hiện |
| FR-04 | Hiển thị **stepper** tiến độ Part/câu | Tiến độ | Thanh tiến độ | |
| FR-05 | Part 1: lần lượt hỏi → cho ghi âm → nộp → chuyển câu; **tổng thời lượng Part 1 có trần 5 phút** | Đề Part 1 | Lượt hỏi–đáp | `PART_DURATION_SECONDS.part1 = 5*60` (`MockExamFull.tsx:174-178`); nếu gặp B/A/D/C (im lặng, nhiễu, sai ngôn ngữ, mất mic, lỗi mạng) — **chưa có xử lý trong app**, xem callout §4.4 |
| FR-06 | Part 2: hiển thị cue card + **1 phút chuẩn bị** (timer) + **2 phút nói** | Đề Part 2 | Overlay chuẩn bị + ghi âm | Xác nhận đúng qua `question-store.ts` (`prepSeconds: 60`) + `PART_DURATION_SECONDS.part2 = 2*60` |
| FR-07 | Part 3: hỏi sâu nhiều câu (thảo luận); **tổng thời lượng Part 3 có trần 7 phút** | Đề Part 3 | Lượt hỏi–đáp | `PART_DURATION_SECONDS.part3 = 7*60` (`MockExamFull.tsx:174-178`) — tài liệu cũ chưa từng ghi ngân sách thời gian tổng cho Part 1/3, chỉ ghi "nhiều câu" |
| FR-08 | Cho **ghi âm/dừng/nộp** từng lượt, có trạng thái xử lý ngắn | Thao tác ghi | Bản ghi + chuyển lượt | Nếu mic mất kết nối giữa lúc ghi (D1) — **chưa có xử lý trong app**, xem callout §4.4 |
| FR-09 | Hết Part 3, giám khảo phải **đóng bài** rồi chuyển Processing | Hoàn tất Part 3 | Lời kết + chuyển màn | |
| FR-10 | Khi reload trang giữa bài, hệ thống **tự động khôi phục lặng lẽ đúng Part/câu đang dở**, không hiển thị màn/hộp xác nhận "Tiếp tục từ Câu X?" | Session đã lưu (`usePersistExamSession.ts`, `mock-exam-persistence.ts`, đồng bộ NineSpeak) | Vào thẳng đúng vị trí đang dở, không có bước xác nhận trung gian | Quyết định chốt qua grill-me (2026-07-07) |
| FR-11 | Cho **Thoát** giữa bài (`handleExitMock`): chỉ ghi analytics, **giữ nguyên session** để resume sau | Bấm "Thoát" | Rời phòng thi, session còn nguyên | Không xoá gì, không cảnh báo mất dữ liệu (vì không mất) |
| FR-12 | Cho **Hủy bài thi** giữa bài, nhưng phải qua **hộp thoại xác nhận 2 bước** ("Bạn có chắc muốn huỷ? Toàn bộ câu trả lời sẽ mất") trước khi huỷ thật | Bấm "Hủy bài thi" | Hộp xác nhận → nếu đồng ý mới gọi `resetSession()` | Chốt qua grill-me — thay cho hành vi huỷ 1 chạm hiện tại (`handleCancelExam`) |
| FR-13 | **Ẩn/disable nút "Hủy bài thi"** một khi tất cả Part hiển thị (visible parts) đã `scoreDone` — chỉ còn "Thoát" khả dụng | Trạng thái ledger (`isComplete` từ `mocktest-charge-ledger.ts:144`) | Nút "Hủy bài thi" biến mất/khoá; "Thoát" vẫn hoạt động | Chốt qua grill-me — tránh mất lượt chấm đã trừ mà không xem được report |
| FR-14 (đích, chưa build) | Phòng thi phải **chỉ chạy phần của chế độ đã chọn** và **kết thúc ngay** sau phần đó (Part lẻ không đi tiếp Part khác) | `examMode` | Luồng đúng chế độ | Chưa build — xem callout đầu mục. **Schema chốt:** bỏ hardcode `visibleParts` tại `mock-exam-definition-from-catalog.ts:32`, dùng `visibleParts` 1 phần tử cho Part lẻ |
| FR-15 (đích, chưa build) | Giám khảo (lời mở/đóng bài), **stepper/stage** và **timer** phải tùy biến theo chế độ | `examMode` | UI theo chế độ | Chưa build — kịch bản giám khảo hiện tại (`mock-exam-scripts.ts:1-89`) là **tĩnh, đơn chế độ**. **Script chốt:** dùng script Part tương ứng từ file hiện tại, thêm lời mở/đóng ngắn riêng cho chế độ Part lẻ — **không fork** toàn bộ file (xem bảng luồng bên dưới) |

**Luồng & giám khảo theo chế độ (FR-14/15 — đích, chưa build):**

| Chế độ | Luồng | Giám khảo mở bài | Đóng bài | Stepper / timer |
|---|---|---|---|---|
| Full | P1 → cue card → P3 | "…full speaking test" | "…end of the speaking test" | 3 chip Part 1·2·3 |
| Part 1 | Chỉ hỏi–đáp Part 1 | "Let's focus on Part 1 today." | "That's the end of your Part 1 practice." | 1 chip "Luyện Part 1" · 5:00 |
| Part 2 | Vào thẳng cue card (1' chuẩn bị + 2' nói) | "Let's focus on Part 2 today." | "That's the end of your Part 2 practice." | 1 chip "Luyện Part 2" · 2:00 |
| Part 3 | Chỉ thảo luận Part 3 | "Let's focus on Part 3 today." | "That's the end of your Part 3 practice." | 1 chip "Luyện Part 3" · 5:00 |

**Quy tắc nghiệp vụ (Business rules):**
- **BR-01:** Máy trạng thái bài thi: pre → part1 → (prep2) → part2 → part3 → completed.
- **BR-02:** Mock Test chấm **sau khi xong cả bài** (không hiện điểm giữa chừng).
- **BR-03:** Phụ đề mặc định tắt; khi cần xử lý lỗi giọng (đích, xem #05), giám khảo có thể tạm bật phụ đề để người dùng hiểu — **hành vi này chưa tồn tại trong code hôm nay**.
- **BR-04:** Part 2 có pha chuẩn bị 1 phút trước khi nói 2 phút (`prepSeconds: 60` + `PART_DURATION_SECONDS.part2 = 120`); Part 1 có trần tổng 5 phút, Part 3 có trần tổng 7 phút.
- **BR-05 (đích, chưa build):** **Chế độ thi (`examMode`)** do scope ở màn Chọn đề quyết định (#01); `examParts` = các phần của chế độ; phòng thi bắt đầu ở phần đầu và **kết thúc khi hết phần cuối** của chế độ. Thực tế hôm nay: `visibleParts` luôn cố định `["part1","part2","part3"]` (`mock-exam-definition-from-catalog.ts:32`), không đọc theo scope.
- **BR-06 (đích, chưa build):** Stepper hiển thị các phần của chế độ (full = 3 chip; Part lẻ = 1 chip "Luyện Part X"); thanh tiến độ tính theo số phần của chế độ.
- **BR-07 (resume, chốt qua grill-me):** Khi reload giữa bài, hệ thống dùng dữ liệu đã persist (`usePersistExamSession.ts` + `mock-exam-persistence.ts` + đồng bộ NineSpeak) để **tự động khôi phục lặng lẽ** đúng Part/câu đang dở — không có bước xác nhận nào chắn giữa việc load lại trang và việc quay lại đúng vị trí.
- **BR-08 (huỷ vs thoát, chốt qua grill-me):** "Thoát" (`handleExitMock`) và "Hủy bài thi" (`handleCancelExam` → `resetSession()`) là **2 hành động khác nhau về bản chất**: Thoát chỉ ghi analytics và giữ nguyên session (resume được); Hủy xoá `session` ở phía client (`setSession(null)` trong `MockExamSessionProvider.tsx:185-187`) và **không** gọi bất kỳ API backend nào (không có endpoint DELETE/cancel/archive trong `app-bff-client.ts`, chỉ có `upsertExamSession`). Do đó "Hủy bài thi" **không xoá dữ liệu thật ở server** — bản ghi session/kết quả từng Part đã `upsertExamSession` trên NineSpeak/Firestore vẫn tồn tại vĩnh viễn, chỉ trở thành "mồ côi" (không ai truy cập/resume lại được nữa qua UI). Nhãn nút hiện tại "Hủy bài thi (Mất dữ liệu)" nên hiểu là "mất quyền truy cập cục bộ", không phải xoá dữ liệu server.
- **BR-09 (chặn huỷ sau khi đã trừ lượt, chốt qua grill-me):** Ledger tính `isComplete = partsToCheck.every(part => Boolean(parts[part]?.scoreDone))` (`mocktest-charge-ledger.ts:144`) và trừ lượt qua `chargeIfReady()` (`mock-exam-route-handlers.ts:334-350`) ngay khi tất cả Part hiển thị đã `scoreDone`. Nếu user huỷ **trước** khi `isComplete`, chưa có lượt nào bị trừ (an toàn). Nếu user huỷ **sau** khi `isComplete` (lượt đã bị trừ, bài gần như xong chỉ chờ vocab/grammar finalize), huỷ client-side sẽ làm mất quyền xem report của 1 lượt đã trả phí — do đó **nút "Hủy bài thi" phải bị ẩn/disable trong trạng thái này**, chỉ còn "Thoát" khả dụng.
- **BR-10 (xác nhận huỷ, chốt qua grill-me):** "Hủy bài thi" bắt buộc phải qua hộp thoại xác nhận 2 bước trước khi thực sự gọi `resetSession()` — không cho phép huỷ ngay 1 chạm như hành vi hiện tại.
- **BR-11 (schema Part lẻ, chốt 2026-07-07 phiên 2):** Part lẻ dùng chung schema session hiện tại — `visibleParts` chỉ có 1 phần tử (vd `["part2"]`). Thay đổi duy nhất: bỏ hardcode `["part1","part2","part3"]` tại `mock-exam-definition-from-catalog.ts:32`. Ledger tự tính **1 lượt chấm** cho 1 Part visible (không cần thay đổi logic ledger).
- **BR-12 (redirect Part lẻ, chốt 2026-07-07 phiên 2):** Khi thi Part lẻ xong, hệ thống redirect thẳng về report của Part đó — giống luồng full, không có màn trung gian. Report chỉ hiển thị 1 Part.
- **BR-13 (history Part lẻ, chốt 2026-07-07 phiên 2):** Kết quả thi Part lẻ ghi chung vào History với full exam. Card lịch sử hiển thị label rõ loại: "Luyện Part X" (vd "Luyện Part 2") thay vì "Mock Test Full".

**Phân quyền theo vai trò:**

| Vai trò | Được làm gì |
|---|---|
| Thí sinh | Nghe, ghi âm, nộp, bật/tắt phụ đề, thoát (giữ session), huỷ (qua xác nhận 2 bước, chỉ khi chưa `scoreDone` toàn bộ) |

---

## 4.3 Cơ chế hoạt động

**Luồng người dùng (workflow):**
1. Vào phòng thi sau khi test mic OK (#03).
2. Giám khảo chào + hỏi Part 1 → user ghi âm → nộp → câu kế (tối đa 5 phút tổng cho Part 1).
3. Chuyển Part 2: hiện cue card → 1 phút chuẩn bị → 2 phút nói.
4. Part 3: thảo luận nhiều câu (tối đa 7 phút tổng cho Part 3).
5. Giám khảo đóng bài → Processing.

**Use case chính:**
- **Bối cảnh:** thí sinh thi đủ 3 Part (chế độ full — chế độ duy nhất chạy được hôm nay).
- **Kịch bản chính (happy path):** trả lời trôi chảy từng câu → hết bài → chấm → report.
- **Kịch bản thay thế / ngoại lệ:** lỗi giọng/thiết bị/mạng (đích, xem #05 và callout §4.4 — **chưa build**); reload giữa bài → tự khôi phục lặng lẽ (FR-10/BR-07); thoát giữa bài → session giữ nguyên (FR-11); huỷ giữa bài → xác nhận 2 bước rồi mới huỷ client-side, có thể bị chặn nếu đã `scoreDone` toàn bộ (FR-12/13).

**Luồng dữ liệu:** Đề (catalog) → kịch bản giám khảo (tĩnh, `mock-exam-scripts.ts`) + câu hỏi → user ghi âm từng lượt → bản ghi (audioUrl) + metadata theo Part/câu → gom vào session → `upsertExamSession` lưu tiến độ liên tục (phục vụ resume) → hết Part 3 → chuyển Processing để chấm → khi tất cả Part `scoreDone`, ledger tính `isComplete` và `chargeIfReady()` trừ 1 lượt chấm.

**Tích hợp:** Ghi âm (mic) · Lưu/khôi phục session (`usePersistExamSession.ts`, `mock-exam-persistence.ts`, `upsertExamSession`, đồng bộ NineSpeak) · Ledger lượt chấm (`mocktest-charge-ledger.ts`) · Processing (#06) · Analytics (`mock_started`, `part_completed`).

*Sơ đồ phù hợp: state diagram máy thi (BR-01) + sequence giám khảo↔thí sinh mỗi lượt + state diagram Thoát/Huỷ (BR-08/09/10).*

---

## 4.4 Xử lý ngoại lệ và trạng thái

> **Callout — Edge case giọng nói (#05) hoàn toàn CHƯA XÂY DỰNG:** Toàn bộ nhóm B/A/D/C (B1 nói quá ngắn, B2 im lặng, B3 nhiễu ồn, B5 sai ngôn ngữ, D1 mất kết nối mic, C1 lỗi mạng) là **spec đích, chưa có bất kỳ hạ tầng nào trong code hôm nay** — không có phát hiện im lặng, không có phát hiện ngôn ngữ, không có logic giám khảo phản ứng "trong vai" ở `useExamRecordingController.ts` hay bất kỳ đâu trong pipeline chấm điểm. Đây là khoảng trống lớn hơn đánh giá trước đây ("gần xong") — thực tế là **0%**. Quyết định chốt qua grill-me (2026-07-07): **giữ nguyên toàn bộ phạm vi đặc tả** bên dưới làm spec đích (không descope B3/B5 dù khó về kỹ thuật), chỉ đánh dấu rõ "chưa build, cần đầu tư hạ tầng nhận diện" — việc descope là quyết định của giai đoạn lập kế hoạch triển khai, không phải của PRD.
>
> **Hệ quả cho FR-01 → FR-10:** hôm nay, bất kỳ tình huống nào trong B1/B2/B3/B5/D1/C1 xảy ra thực tế trong lúc thi chỉ dẫn tới **điểm số thấp từ Azure/OpenAI** hoặc trạng thái "failed" chung chung — **không có tín hiệu nào trong app** cho user biết hệ thống đã nhận ra vấn đề, và giám khảo không có phản ứng "trong vai" nào tương ứng.

| Trường hợp | Hành vi mong đợi / thông báo | Trạng thái build |
|---|---|---|
| B1 — Nói quá ngắn | (Đích) Giám khảo nhắc trong vai, mời nói lại | **Chưa build** — không có phát hiện độ dài câu trả lời; hôm nay chỉ ảnh hưởng điểm chấm |
| B2 — Im lặng | (Đích) Giám khảo phát hiện im lặng, nhắc lại câu hỏi | **Chưa build** — không có phát hiện im lặng trong `useExamRecordingController.ts` |
| B3 — Nhiễu ồn | (Đích) Giám khảo cảnh báo môi trường ồn, gợi ý tìm nơi yên tĩnh | **Chưa build** — không có phát hiện tạp âm; giữ nguyên trong spec theo quyết định grill-me (không descope) |
| B5 — Sai ngôn ngữ | (Đích) Giám khảo nhắc trả lời bằng tiếng Anh | **Chưa build** — không có phát hiện ngôn ngữ; giữ nguyên trong spec theo quyết định grill-me (không descope) |
| D1 — Mất kết nối mic giữa lúc ghi | (Đích) Card lỗi hệ thống + cho thu lại | **Chưa build** — lỗi hiện chỉ dẫn tới trạng thái "failed" chung, không có card riêng trong phòng thi |
| C1 — Lỗi mạng | (Đích) Thông báo lỗi mạng + retry | **Chưa build** — chưa xác nhận có xử lý riêng trong phòng thi (khác cơ chế resume sau reload) |
| Reload trang giữa bài | **Tự động khôi phục lặng lẽ** đúng Part/câu đang dở, không hỏi xác nhận | **Đã có** — dữ liệu persist qua `usePersistExamSession.ts` + `mock-exam-persistence.ts` + đồng bộ NineSpeak; hành vi "lặng lẽ, không hỏi" chốt qua grill-me (FR-10/BR-07) |
| Bấm "Thoát" giữa bài | Rời phòng thi, session giữ nguyên, có thể resume sau, không cảnh báo mất dữ liệu | **Đã có** (`handleExitMock`) |
| Bấm "Hủy bài thi" giữa bài, **chưa** `scoreDone` toàn bộ | Hộp xác nhận 2 bước ("Bạn có chắc muốn huỷ? Toàn bộ câu trả lời sẽ mất") → đồng ý mới `resetSession()` (client-side only, dữ liệu server thành mồ côi, không bị xoá thật) | Hộp xác nhận: **chưa build** (chốt mới qua grill-me, FR-12/BR-10); `resetSession()` **đã có** nhưng hiện chạy ngay không qua xác nhận |
| Bấm "Hủy bài thi" giữa bài, **đã** `scoreDone` toàn bộ (lượt đã bị trừ) | Nút "Hủy bài thi" **ẩn/disable**, chỉ còn "Thoát" | **Chưa build** (chốt mới qua grill-me, FR-13/BR-09) |
| Part 2 nói quá 2 phút | (Cần chốt) giám khảo chủ động dừng — chưa làm | Chưa build — vẫn là câu hỏi mở (xem §4.7) |

**Các trạng thái giao diện:** giám khảo đang nói / chờ user / đang ghi / đang xử lý ngắn / chuyển Part / (đích, chưa build) giám khảo phản ứng lỗi giọng trong vai.

---

## 4.5 Yêu cầu giao diện (UI/UX)

- Mockup: `mocktest-flow-mockup.html` — scrLive, `examinerAvatar()`, `askQ()`, `part2Prep()`, `userStart()`/`userStop()`.
- Giao diện **tối** (focus, không sidebar). Avatar giám khảo trung tâm; 2 vòng ping khi nói. Toggle "Phụ đề". Nút ghi âm (xanh idle → maroon stop). Cue card sáng kiểu serif. Overlay timer 1 phút (Part 2).
- **Theo chế độ (đích, chưa build):** `beginExam()` bắt đầu đúng phần đầu của `examParts`; `nextPartOrEnd()` kết thúc sớm sau phần cuối của chế độ; `endExam()` chọn lời đóng bài theo chế độ; `renderChrome()` lọc stage pills theo chế độ.
- **Hộp thoại xác nhận Hủy bài thi (mới, chốt qua grill-me):** dạng modal 2 bước — bước 1 hỏi "Bạn có chắc muốn huỷ bài thi? Toàn bộ câu trả lời sẽ mất." kèm 2 nút rõ ràng ("Huỷ bài thi" / "Quay lại làm bài"); chỉ khi xác nhận ở bước 1 mới thực sự gọi `resetSession()`. Modal này **không xuất hiện** nếu tất cả Part đã `scoreDone` — trong trường hợp đó nút "Hủy bài thi" không hiển thị/bị disable ngay từ đầu.
- **Resume sau reload:** không có UI trung gian nào (không skeleton "đang khôi phục", không hộp xác nhận) — trải nghiệm phải giống như user chưa từng rời trang, chỉ có độ trễ tải dữ liệu tự nhiên.

---

## 4.6 Tiêu chí chấp nhận

**AC-01: Hiệu ứng khi giám khảo nói**
- **Given:** Giám khảo đang phát lời thoại (TTS)
- **When:** Trạng thái "đang nói" được kích hoạt
- **Then:** Hiển thị sóng âm vòng tròn quanh avatar AND (nếu phụ đề đang bật) hiện lời thoại dạng text

**AC-02: Phụ đề mặc định tắt**
- **Given:** User vào phòng thi lần đầu trong phiên
- **When:** Màn phòng thi render
- **Then:** Phụ đề mặc định tắt; bật toggle "Phụ đề" thì hiện lời thoại ngay

**AC-03: Cue card + timer Part 2**
- **Given:** User đang ở Part 2
- **When:** Phòng thi chuyển vào Part 2
- **Then:** Hiển thị cue card AND đếm ngược 1 phút chuẩn bị, sau đó tự chuyển sang đếm 2 phút nói

**AC-04: Chuyển lượt sau khi nộp**
- **Given:** User đang trả lời một câu hỏi
- **When:** User nộp bản ghi âm của lượt đó
- **Then:** Hệ thống chuyển sang câu/Part kế tiếp đúng thứ tự

**AC-05: Kết thúc bài thi**
- **Given:** User đã hoàn thành Part 3
- **When:** Giám khảo đọc xong lời đóng bài
- **Then:** Hệ thống chuyển sang màn Processing

**AC-06: Vượt trần thời gian Part 1/3**
- **Given:** User đang ở Part 1 hoặc Part 3 và chưa trả lời hết câu hỏi của Part
- **When:** Thời gian chạy đủ trần của Part đó (5 phút cho Part 1, 7 phút cho Part 3 — `PART_DURATION_SECONDS`)
- **Then:** Hệ thống tự chuyển sang Part tiếp theo, bất kể còn câu hỏi chưa hỏi

**AC-07 (đích, chưa build): Chỉ chạy 1 Part ở chế độ Part lẻ**
- **Given:** User chọn chế độ Part lẻ (Part 1/2/3) ở màn Chọn đề
- **When:** Phòng thi chạy hết phần của Part đó
- **Then:** Hệ thống kết thúc bài thi ngay, không chuyển sang Part khác

**AC-08 (đích, chưa build): Vào thẳng cue card ở chế độ Part 2**
- **Given:** User chọn chế độ Part 2
- **When:** User vào phòng thi
- **Then:** Hệ thống vào thẳng cue card, không chạy Part 1 trước

**AC-09 (đích, chưa build): Giám khảo/stepper/timer tùy biến theo chế độ**
- **Given:** User đang ở chế độ Part X (không phải full)
- **When:** Phòng thi render lời mở bài, stepper và timer
- **Then:** Cả 3 thành phần hiển thị đúng theo Part X; chế độ full vẫn giữ nguyên 3 chip Part

**AC-10: Tự động khôi phục sau reload**
- **Given:** User đang thi dở (đã lưu tiến độ qua `usePersistExamSession`)
- **When:** User reload trang giữa bài
- **Then:** Hệ thống tự động khôi phục lặng lẽ đúng Part/câu đang dở AND không hiển thị bất kỳ màn/hộp xác nhận nào trước khi vào lại

**AC-11: Thoát giữa bài (giữ session)**
- **Given:** User đang thi dở
- **When:** User bấm "Thoát"
- **Then:** Hệ thống rời phòng thi AND giữ nguyên session (resume được sau) AND không hiển thị cảnh báo mất dữ liệu

**AC-12: Xác nhận trước khi Hủy bài thi (chưa scoreDone toàn bộ)**
- **Given:** Không phải tất cả Part hiển thị đã `scoreDone`
- **When:** User bấm "Hủy bài thi"
- **Then:** Hệ thống hiện hộp xác nhận 2 bước trước; chỉ khi user xác nhận mới thực sự gọi `resetSession()`, không huỷ ngay 1 chạm

**AC-13: Chặn Hủy bài thi sau khi đã trừ lượt**
- **Given:** Tất cả Part hiển thị đã `scoreDone` (lượt chấm đã bị trừ theo ledger)
- **When:** User xem màn phòng thi/thoát
- **Then:** Nút "Hủy bài thi" bị ẩn hoặc disable; chỉ còn "Thoát" khả dụng

**AC-14: Dữ liệu server không bị xoá khi Hủy**
- **Given:** User đã xác nhận "Hủy bài thi" thành công
- **When:** Hệ thống thực thi `resetSession()`
- **Then:** Chỉ session ở phía client bị xoá; dữ liệu đã `upsertExamSession` trên server vẫn tồn tại (không bị xoá, chỉ không còn resume/truy cập lại được qua UI)

**AC-15 (đích, chưa build): Redirect thẳng về report sau khi xong Part lẻ**
- **Given:** User đang thi Part lẻ (vd Part 2) và đã hoàn thành phần đó
- **When:** Giám khảo đọc xong lời đóng bài của Part lẻ
- **Then:** Hệ thống redirect thẳng về report của Part đó; report chỉ hiển thị 1 Part; không có màn trung gian nào xuất hiện

**AC-16 (đích, chưa build): History hiển thị label "Luyện Part X"**
- **Given:** User đã hoàn thành 1 session Part lẻ (vd Part 2)
- **When:** User vào màn History
- **Then:** Card lịch sử hiển thị "Luyện Part 2" (không phải "Mock Test Full") để phân biệt rõ loại luyện tập

**AC-17 (đích, chưa build): Part lẻ tiêu 1 lượt chấm**
- **Given:** User đã hoàn thành session Part lẻ (1 Part visible)
- **When:** Ledger tính `isComplete` và gọi `chargeIfReady()`
- **Then:** Hệ thống trừ đúng **1 lượt chấm** (không phải 3) vì `visibleParts` chỉ có 1 phần tử

---

## Non-functional Requirements

**Hiệu năng:** phản hồi sau khi nộp 1 lượt ≤ 1.5s (hiện trạng thái xử lý); chuyển Part mượt; khôi phục sau reload không được có độ trễ cảm nhận được ngoài thời gian tải dữ liệu tự nhiên (không thêm bước chờ nhân tạo).

**Trải nghiệm:** nhịp dẫn dắt tự nhiên, không khựng giữa các lượt; hộp xác nhận huỷ phải rõ ràng, không mập mờ giữa "Huỷ" và "Thoát" (2 hành động có hệ quả rất khác nhau — xem BR-08).

**Toàn vẹn dữ liệu:** mọi thao tác ghi âm/nộp phải được `upsertExamSession` liên tục trong lúc thi để đảm bảo resume chính xác; hành động "Hủy bài thi" không được xoá dữ liệu đã ghi ở server (chỉ được phép mất quyền truy cập cục bộ, theo BR-08).

---

## User Types

Màn này không phân biệt theo gói (free/premium) — mọi thí sinh đã vào tới phòng thi đều theo cùng 1 luồng thi. Điểm khác biệt hành vi duy nhất là **trạng thái ledger đã `scoreDone` toàn bộ hay chưa** (ảnh hưởng nút "Hủy bài thi" có khả dụng hay không — xem FR-13/BR-09), không phải một "loại user" theo nghĩa sản phẩm (free vs premium không ảnh hưởng gì tới màn này).

---

## 4.7 Phần bổ sung

**Giả định (Assumptions):** Bản ghi từng lượt được lưu để chấm sau (per-part / per-question); dữ liệu session persist đủ chi tiết (Part/câu hiện tại) để phục vụ resume chính xác.

**Phụ thuộc (Dependencies):**
- Test mic (#03) · Lưu/khôi phục session (`usePersistExamSession.ts`, `mock-exam-persistence.ts`) · Pipeline chấm (#06) · Ledger lượt chấm (`mocktest-charge-ledger.ts`, `mock-exam-route-handlers.ts`).
- **Việc mở rộng kịch bản giám khảo theo mode** (`mock-exam-scripts.ts`) — cần thêm lời mở/đóng ngắn riêng cho từng Part lẻ (vd "Let's focus on Part X today." / "That's the end of your Part X practice.") vào file hiện tại; **không fork** toàn bộ file (chốt 2026-07-07 phiên 2, xem đổi log #13 và FR-15).
- **Hạ tầng phát hiện im lặng/tạp âm/ngôn ngữ** — cần đầu tư mới hoàn toàn nếu triển khai #05 (B1-B5/D1/C1); hiện không có bất kỳ thành phần nào trong `useExamRecordingController.ts` hay pipeline chấm điểm phục vụ việc này.
- **Hộp thoại xác nhận huỷ 2 bước** (FR-12) và **logic chặn huỷ khi đã `scoreDone`** (FR-13) — cần xây UI + đọc trạng thái ledger phía client trước khi cho phép hiển thị nút "Hủy bài thi".

**Câu hỏi mở / chưa chốt:**
- [ ] Part 2 nói quá 2 phút → giám khảo có chủ động dừng hay không, và dừng bằng cách nào (chưa làm, chưa chốt).
- [ ] Lời thoại giám khảo: tĩnh theo kịch bản hay TTS động? (chưa chốt — hiện tại là tĩnh, chưa rõ có kế hoạch đổi sang động không).
- [ ] Có cho nghe lại câu hỏi giám khảo (replay) trong lúc thi không? (chưa chốt).
- [ ] Thứ tự ưu tiên triển khai hạ tầng edge-case (#05) — bắt đầu từ B nào trước (B1/B2 dễ hơn kỹ thuật so với B3/B5)? (chưa chốt — nằm ngoài phạm vi PRD, thuộc sprint planning).

---

## 4.8 Câu hỏi mở — đã chốt (2026-07-07, grill-me)

| Câu hỏi | Đã chốt tại |
|---|---|
| Có nên descope B3 (nhiễu)/B5 (sai ngôn ngữ) khỏi #05 vì khó kỹ thuật không? | Không — giữ nguyên toàn bộ phạm vi đặc tả, chỉ đánh dấu "chưa build, cần đầu tư hạ tầng nhận diện"; descope là quyết định của sprint planning, không phải của PRD — §4.4 callout, Đổi log #1 |
| Resume sau reload giữa bài nên hiện xác nhận "Tiếp tục từ Câu X?" hay tự động? | Tự động khôi phục lặng lẽ, không hộp xác nhận — reload là tai nạn, không phải hành động chủ đích — FR-10/BR-07, AC-10 |
| Có cần xác nhận trước khi "Hủy bài thi" không (hiện tại huỷ ngay 1 chạm)? | Có — bắt buộc hộp xác nhận 2 bước trước khi huỷ thật, vì đây là hành động phá huỷ tiến độ không thể hoàn tác — FR-12/BR-10, AC-12 |
| "Hủy bài thi" hiện tại có thật sự xoá dữ liệu trên server không? | Không — chỉ reset client-side (`setSession(null)`), không có API xoá/huỷ nào ở backend; dữ liệu đã lưu trên NineSpeak/Firestore vẫn tồn tại vĩnh viễn nhưng thành "mồ côi" (không truy cập/resume lại được) — BR-08, AC-14 |
| Có nên chặn "Hủy bài thi" sau khi lượt chấm đã bị trừ (tất cả Part `scoreDone`) không? | Có — ẩn/disable nút "Hủy bài thi" trong trạng thái này, chỉ còn "Thoát" khả dụng, để tránh mất lượt chấm đã trả phí mà không xem được report — FR-13/BR-09, AC-13 |
| Cơ chế phát hiện D1 (mất mic) nên dùng polling hay event? | Event-based (`track.onended` / `track.onmute` / `stream.oninactive`) là trigger chính vì phản ứng tức thì không tốn tài nguyên; energy polling là tầng phụ để validate nhưng không trigger độc lập. Hành vi khi phát hiện: dừng ghi + hiện card lỗi → user thu lại từ đầu (không giữ partial audio). Chi tiết kỹ thuật đầy đủ ở #05 — Đổi log #9 |
| Part lẻ có cần schema session mới hay dùng chung? | Dùng chung schema hiện tại — Part lẻ là 1 session với `visibleParts` chỉ có 1 phần tử (vd `["part2"]`). Chỉ cần bỏ hardcode `["part1","part2","part3"]` tại `mock-exam-definition-from-catalog.ts:32`. Ledger tự tính 1 lượt chấm cho 1 Part visible mà không cần thay đổi thêm — BR-11, AC-17, Đổi log #10 |
| Part lẻ khi xong redirect về đâu? | Redirect thẳng về report của Part đó (giống luồng full). Report chỉ hiển thị 1 Part. Không có màn trung gian — BR-12, AC-15, Đổi log #11 |
| Part lẻ có ghi vào History chung không? Hiển thị label gì? | Ghi chung vào History với full exam. Card hiển thị label "Luyện Part X" (vd "Luyện Part 2") thay vì "Mock Test Full" để người dùng phân biệt loại luyện tập — BR-13, AC-16, Đổi log #12 |
| Script giám khảo Part lẻ có cần fork file riêng không? | Không fork toàn bộ file — dùng script Part tương ứng từ `mock-exam-scripts.ts` hiện tại, thêm lời mở/đóng ngắn riêng cho chế độ Part lẻ: mở "Let's focus on Part X today.", đóng "That's the end of your Part X practice." — FR-15, Đổi log #13 |
| Thứ tự ưu tiên build cho các tính năng còn thiếu? | D1 (mất mic) trước, sau đó Part lẻ, rồi mới B1/B2. Part lẻ ưu tiên hơn B1/B2 vì CTA "Bắt đầu Part X" đang hiển thị nhưng dẫn vào full exam — broken promise với người dùng — Đổi log #14 |

Không còn câu hỏi mở nào đã được yêu cầu chốt cho §4 tại thời điểm này (các câu hỏi còn lại ở §4.7 vẫn mở, chưa thuộc phạm vi phiên grill-me này).
