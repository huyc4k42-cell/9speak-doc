# Mock Test V2 — §6 Processing — Chấm điểm — v0.2 (đối chiếu code thật + chốt qua grill-me)

> Ngày cập nhật: 2026-07-07 · Trạng thái: Chính thức · Đối chiếu [`MockExamFull.tsx`](../../../../features/mock-test/pages/exams/MockExamFull.tsx), [`MockExamProcessingScreen.tsx`](../../../../features/mock-test/components/exams/MockExamProcessingScreen.tsx), [`mock-exam-report.ts`](../../../../features/mock-test/lib/mock-exam-report.ts) · Thay thế toàn bộ nội dung bản v0.1 (2026-06-20, chuyển thể từ Google Doc "Mocktest V2").

## Đổi log so với v0.1

| # | Thay đổi | Lý do |
|---|---|---|
| 1 | Bổ sung chính thức màn **"Chúc mừng"** (pháo hoa + TTS lời kết giám khảo) vào luồng, với thời lượng xác nhận qua code: **đúng 3.6 giây** (`CELEBRATION_HOLD_MS = 3600`, `MockExamFull.tsx:209`). Bản v0.1 chỉ ghi chú "cần bổ sung khi có dịp cập nhật" — nay đã bổ sung đầy đủ vào §6.3 | Đối chiếu `MockExamFull.tsx:207-209, 266, 435-439` |
| 2 | Xác nhận qua code: **chấm điểm chạy nền NGAY khi bài thi hoàn tất, song song với màn Chúc mừng** — không chờ hết pháo hoa mới bắt đầu chấm. `useSegmentedPartFinalization` và `useOverallBreakdownSynthesis` kích hoạt ngay khi `examState === "completed"` (`MockExamFull.tsx:623, 627-631`), trong khi màn Chúc mừng vẫn đang hiển thị (`completionStage === "celebrate"`) | Đối chiếu `MockExamFull.tsx:623, 627-631` |
| 3 | Xác nhận qua code: **thanh tiến độ là thật, không mô phỏng**, có công thức trọng số rõ ràng theo từng Part (segmented Part 1/3 chia 50% pha phát âm + 50% pha nội dung; Part 2 nhảy thẳng 0%→100%) và cơ chế "trickle" (nhích chậm giữa 2 lần poll) tôn trọng `prefers-reduced-motion` | Đối chiếu `MockExamProcessingScreen.tsx:22-60` + `mock-exam-report.ts:62-129` |
| 4 | Xác nhận qua code: điều kiện auto-redirect sang report gồm **3 điều kiện đồng thời**: `reportReady` (tất cả Part `status="completed"` VÀ có `evaluation`) AND `!hasQuotaPending` (không Part nào `status="pending"`) AND `completionStage === "processing"` (đã qua màn Chúc mừng) | Đối chiếu `MockExamFull.tsx:1108-1150`, `mock-exam-report.ts:46-60` |
| 5 | **Sửa lại FR-06 cũ (progressive reveal) — KHÔNG còn là spec đích chưa build, mà là hành vi ĐÃ CHỐT chủ đích khác với mô tả cũ.** Bản v0.1 mô tả "mở report sớm khi quick-score xong, phần còn lại hiện skeleton". Đối chiếu code: `reportReady` yêu cầu **tất cả** Part có `evaluation` đầy đủ, không chỉ điểm số (`mock-exam-report.ts:54-60`) — nghĩa là Processing luôn chờ chấm xong hoàn toàn (score+vocab+grammar) cho mọi Part rồi mới chuyển. Cơ chế skeleton/partial + retry per-tab (đã tài liệu hoá ở #07/#08) tồn tại cho trường hợp LỖI, không phải để mở report sớm một cách chủ động khi mọi thứ đang chấm bình thường | Chốt qua grill-me (2026-07-07) — xem Đổi log #5 chi tiết ở §6.8 và đối chiếu `mock-exam-report.ts:54-60` |
| 6 | Sửa lại mô tả "Lỗi mạng khi chấm (C1)" ở bảng ngoại lệ cũ: **Processing screen không có UI lỗi của riêng nó.** Khi có Part lỗi và không còn Part nào đang chấm, TOÀN BỘ màn Processing bị thay thế bằng 1 card báo lỗi + nút "Mở màn hình chấm lại" điều hướng sang Report (nơi có cơ chế retry per-tab thật sự) | Đối chiếu `MockExamFull.tsx:1477-1499` |
| 7 | Sửa lại mô tả "Hết lượt chấm" ở bảng ngoại lệ cũ: **không phải "Part chuyển pending, không mất dữ liệu" hiển thị âm thầm** — mà là Processing bị THAY THẾ HOÀN TOÀN bằng màn Paywall (`MockExamQuotaPaywallScreen`) hoặc Purchase-Status (`MockExamPurchaseStatusScreen`); user không bao giờ thấy chữ "pending" hay thanh tiến độ một khi rơi vào trạng thái này | Đối chiếu `MockExamFull.tsx:1433-1473` |
| 8 | **Quyết định (grill-me, 2026-07-07): KHÔNG yêu cầu màn Chúc mừng/pháo hoa phải tôn trọng `prefers-reduced-motion`**, dù thanh tiến độ cùng màn Processing đã làm đúng điều này. Ghi rõ đây là điểm đã cân nhắc và quyết định không sửa, không phải câu hỏi mở còn treo | Chốt qua grill-me (2026-07-07) — xem §6.7, §6.8 |
| 9 | Chuẩn hoá cấu trúc tài liệu theo khung đang áp dụng cho #01/#03/#04/#05: thêm mục **Goal / Non-functional Requirements / User Types**, đổi mục "Tiêu chí chấp nhận" từ bảng phẳng sang **định dạng Given/When/Then theo từng AC**, đổi số mục từ 1-7 sang 6.1-6.8, thêm **Đổi log** và **Câu hỏi mở — đã chốt** | Chuẩn hoá tài liệu theo yêu cầu product owner, đồng bộ với #01/#03/#04/#05 |

---

## Goal

**Business/Product Goal:** Giữ người dùng ở lại trong khoảng thời gian chấm điểm (vài chục giây) bằng cảm giác "đang có việc thật sự xảy ra" (tiến độ thật, không đứng hình) thay vì một màn chờ trống, đồng thời đảm bảo report chỉ mở ra khi dữ liệu đã đầy đủ — tránh cảm giác "nửa vời" gây mất niềm tin.

**User Benefits:**
- Thí sinh: được trấn an ngay sau khi thi xong bằng màn Chúc mừng (ăn mừng thành tích), sau đó thấy tiến độ chấm điểm thật đang chạy, không phải màn hình đứng yên không phản hồi.
- Thí sinh: không bị đưa vào report khi report còn thiếu dữ liệu (vocab/grammar chưa xong) — tránh trải nghiệm khó hiểu kiểu "sao band Grammar chưa hiện".
- Thí sinh gặp sự cố (lỗi chấm 1 Part, hoặc hết lượt chấm giữa chừng): được dẫn tới đúng màn xử lý tiếp theo (báo lỗi + chấm lại, hoặc paywall) thay vì kẹt ở màn Processing vô thời hạn.
- Sản phẩm: giảm tỉ lệ rời bỏ trong lúc chờ chấm nhờ thanh tiến độ thật (không giả), đồng thời tránh rủi ro UX "report nửa vời" bằng cách chờ chấm xong hoàn toàn mới chuyển trang.

---

## 6.1 Tổng quan

**Mô tả ngắn:** Màn chuyển tiếp giữa "kết thúc bài thi" và "xem report". Ngay sau khi thi xong, người dùng thấy màn **Chúc mừng** (pháo hoa + TTS lời kết giám khảo, đúng 3.6 giây) trong khi hệ thống đã bắt đầu chấm điểm ở nền; sau đó chuyển sang màn **Processing** với thanh tiến độ chấm thật, và **tự động** đưa người dùng sang report khi mọi Part đã chấm xong hoàn toàn.

**Mục đích / vấn đề giải quyết:** Việc chấm 3 Part tốn vài chục giây; màn Chúc mừng + Processing trấn an người dùng (ăn mừng trước, rồi thấy tiến độ thật, không đứng hình) để giảm cảm giác chờ và tỉ lệ rời bỏ. Đồng thời đảm bảo report không mở ra ở trạng thái thiếu dữ liệu.

**Phạm vi:**
- Trong phạm vi: màn Chúc mừng (pháo hoa, TTS, thời lượng), màn Processing (thanh tiến độ phân tích thật, hiệu ứng tiêu chí trôi nền), điều kiện auto-redirect sang report, hành vi khi có Part lỗi (chuyển màn báo lỗi), hành vi khi hết lượt chấm giữa chừng (chuyển màn paywall/purchase-status).
- Ngoài phạm vi: thuật toán chấm chi tiết (pipeline kỹ thuật — xem [scoring-pipeline-overview.md](../../../scoring-pipeline-overview.md)); nội dung report và cơ chế skeleton/retry per-tab thật sự (#07–08); luồng thanh toán chi tiết bên trong Paywall/Purchase-Status (chỉ cross-reference ở đây).

**Đối tượng người dùng & bên liên quan:** Học viên; Dev FE (`MockExamFull.tsx`, `MockExamProcessingScreen.tsx`) & BE (pipeline chấm); QA.

---

## 6.2 Yêu cầu chức năng

| Mã | Yêu cầu | Đầu vào | Đầu ra | Ghi chú |
|---|---|---|---|---|
| FR-01 | Khi hoàn tất bài thi, hiển thị màn **Chúc mừng** (pháo hoa + TTS lời kết giám khảo) trước khi vào Processing | `examState="completed"`, `completionStage="celebrate"` | Màn Chúc mừng | `MockExamFull.tsx:266, 420-1431`; hằng số thời lượng `CELEBRATION_HOLD_MS = 3600` tại `:209` |
| FR-02 | Màn Chúc mừng giữ đúng **3.6 giây** rồi tự chuyển `completionStage` sang `"processing"` (chờ TTS/narration kết thúc trước khi bắt đầu đếm) | `isNarrating=false` | `completionStage="processing"` | `MockExamFull.tsx:433-439` |
| FR-03 | Chấm điểm chạy nền **ngay khi bài thi hoàn tất**, song song với màn Chúc mừng — không chờ Chúc mừng kết thúc mới bắt đầu chấm | `examState="completed"` | `useSegmentedPartFinalization`, `useOverallBreakdownSynthesis` kích hoạt | `MockExamFull.tsx:623, 627-631` |
| FR-04 | Sau màn Chúc mừng, hiển thị màn Processing với **thanh tiến độ phân tích** bám theo tiến độ chấm thật, có trickle-fill khi chờ giữa 2 lần poll | Tiến độ chấm thật (`computeScoringProgress`) | % + thanh bar | `MockExamProcessingScreen.tsx:22-65`, công thức tại `mock-exam-report.ts:62-129` |
| FR-05 | Thanh tiến độ tôn trọng `prefers-reduced-motion`: khi bật, tắt trickle, bar nhảy thẳng theo mốc thật ở mỗi lần poll (120ms) | `prefers-reduced-motion: reduce` | Không animation trickle | `MockExamProcessingScreen.tsx:30-34, 40-41` |
| FR-06 (đã sửa — xem Đổi log #5) | Processing chờ **TẤT CẢ Part chấm xong hoàn toàn** (status="completed" **và** có `evaluation` đầy đủ: score+vocab+grammar) rồi mới tự chuyển sang report — **không** mở report sớm kiểu skeleton khi mọi thứ đang chấm bình thường | Trạng thái từng Part | Redirect chỉ khi đủ điều kiện | `mock-exam-report.ts:54-60` (`isMockExamFullyGraded`); đây là quyết định chủ đích, không phải thiếu sót — xem §6.8 |
| FR-07 | Khi đủ điều kiện (`reportReady` AND `!hasQuotaPending` AND đã qua Chúc mừng), **tự chuyển** sang report | `reportReady`, `hasQuotaPending`, `completionStage` | Redirect `/report/[sessionId]` | `MockExamFull.tsx:1108-1150` |
| FR-08 | Khi có ≥1 Part lỗi (`status="failed"`) và không còn Part nào đang chấm (`status="processing"`), **thay thế toàn bộ** màn Processing bằng 1 card báo lỗi + nút điều hướng sang Report | `completionProcessingParts.length === 0 && completionFailedParts.length > 0` | Card lỗi "Một số part chưa hoàn tất chấm điểm" + nút "Mở màn hình chấm lại" | `MockExamFull.tsx:1477-1499` |
| FR-09 | Khi có Part rơi vào `status="pending"` (hết lượt chấm), **thay thế toàn bộ** màn Processing bằng màn Paywall hoặc Purchase-Status | `hasQuotaPending=true` | `MockExamQuotaPaywallScreen` hoặc `MockExamPurchaseStatusScreen` | `MockExamFull.tsx:1433-1473`; user không thấy chữ "pending" hay thanh tiến độ |

**Quy tắc nghiệp vụ (Business rules):**
- **BR-01:** Chỉ redirect sang report khi đồng thời: `reportReady` = true (mọi Part `status="completed"` **và** có `evaluation`) AND `hasQuotaPending` = false (không Part nào `status="pending"`) AND `completionStage === "processing"` (đã qua màn Chúc mừng, không redirect ngay giữa lúc đang ăn mừng). — `MockExamFull.tsx:1108-1150`, `mock-exam-report.ts:46-60`.
- **BR-02:** Tiến độ hiển thị bám tiến độ chấm thật theo trọng số từng Part (`100 / visibleParts.length`); Part segmented (Part 1/3) chia 50% pha phát âm (đếm câu đã có assessment) + 50% pha nội dung (0.4 khi có `evaluation` cơ bản, +0.3 khi vocab review xong, +0.3 khi grammar review xong); Part 2 (single-phase) chỉ 0%→100% khi `status="completed"`. Khi chờ giữa 2 lần poll thì trickle nhẹ (không vượt mốc thật + 6, trần 95% cho tới khi thật sự xong). — `mock-exam-report.ts:62-129`, `MockExamProcessingScreen.tsx:36-57`.
- **BR-03:** Thanh tiến độ tôn trọng `prefers-reduced-motion` (bỏ trickle, bám thẳng mốc thật). Màn Chúc mừng/pháo hoa **không** áp dụng yêu cầu này — quyết định chủ đích, xem §6.7/§6.8.
- **BR-04:** Lỗi chấm 1 Part không buộc làm lại cả bài. Khi Part lỗi và không còn Part nào đang chấm, Processing tự thay bằng card báo lỗi dẫn sang Report — nơi có cơ chế retry per-tab thật sự cho từng Part (xem #08).
- **BR-05 (mới — chốt qua grill-me):** Processing **không** mở report sớm theo kiểu "quick-score xong là mở, phần còn lại hiện skeleton" như mô tả cũ. Report chỉ mở khi TẤT CẢ Part đã có `evaluation` đầy đủ. Cơ chế skeleton/partial-state + retry per-tab (đã tài liệu hoá ở #07/#08) dành riêng cho trường hợp một Part **lỗi thật sự**, không phải để chủ động rút ngắn thời gian chờ cảm nhận khi mọi thứ đang chấm bình thường.

**Phân quyền theo vai trò:**

| Vai trò | Được làm gì |
|---|---|
| Thí sinh | Xem màn Chúc mừng rồi Processing; không có hành động chủ động nào ở 2 màn này (không có nút "Thử lại"/"Bỏ qua" tại chỗ) — mọi xử lý tiếp theo (chấm lại, thanh toán) diễn ra ở màn kế tiếp mà hệ thống tự điều hướng tới |

---

## 6.3 Cơ chế hoạt động

**Luồng người dùng (workflow):**
1. Bài thi hoàn tất → `examState` chuyển `"completed"`, `completionStage` bắt đầu ở `"celebrate"` (`MockExamFull.tsx:266`).
2. Màn Chúc mừng hiển thị: giám khảo đọc TTS lời kết (`ending` line), pháo hoa hiện. Đồng thời — **ngay lập tức, không chờ Chúc mừng xong** — `useSegmentedPartFinalization` và `useOverallBreakdownSynthesis` đã kích hoạt chấm điểm nền (`MockExamFull.tsx:623, 627-631`).
3. Khi `isNarrating` = false (TTS đã đọc xong), đếm đúng 3600ms (`CELEBRATION_HOLD_MS`) rồi `completionStage` chuyển `"processing"` (`MockExamFull.tsx:433-439`).
4. Từ thời điểm này, hệ thống đánh giá liên tục 1 trong 3 nhánh, theo thứ tự ưu tiên trong code:
   a. Nếu `hasQuotaPending` (có Part rơi vào `pending`) → thay Processing bằng Paywall/Purchase-Status (`MockExamFull.tsx:1433-1473`).
   b. Nếu không còn Part nào đang chấm (`processing`) nhưng có Part `failed` → thay Processing bằng card báo lỗi (`MockExamFull.tsx:1477-1499`).
   c. Ngược lại → hiển thị màn Processing với thanh tiến độ thật (`MockExamFull.tsx:1502-1508`).
5. Khi `reportReady` (tất cả Part `completed` + có `evaluation`) và không `hasQuotaPending` → tự động redirect `/report/[sessionId]` (`MockExamFull.tsx:1130-1140`).

**Use case chính:**
- **Bối cảnh:** thí sinh vừa xong Part 3.
- **Kịch bản chính (happy path):** Chúc mừng (3.6s, chấm đã chạy nền từ trước đó) → Processing (thanh tiến độ thật tăng dần) → tất cả Part `completed` + có `evaluation` → tự động sang report.
- **Kịch bản thay thế / ngoại lệ:**
  - Có Part lỗi, không còn Part nào đang chấm → Processing bị thay bằng card báo lỗi + nút "Mở màn hình chấm lại" → Report (nơi retry per-tab thật sự diễn ra).
  - Có Part rơi vào pending (hết lượt chấm) → Processing bị thay bằng Paywall hoặc Purchase-Status; user không thấy tiến độ hay chữ "pending" nào.

**Luồng dữ liệu:** Session (bản ghi 3 Part) → pipeline chấm (Azure → calibrate → quick-score → vocab/grammar/word-cefr, chạy nền từ lúc `examState="completed"`) → cập nhật `status`/`evaluation` từng Part trong session → `computeScoringProgress` (`mock-exam-report.ts:115-129`) tính % hiển thị → `useTrickleProgress` (`MockExamProcessingScreen.tsx:22-60`) làm mượt hiển thị → khi `isMockExamFullyGraded` đúng và không còn pending → redirect.

**Tích hợp:** Pipeline `evaluate-mock-exam-segmented` (xem [scoring-pipeline-overview.md](../../../scoring-pipeline-overview.md)) · `useSegmentedPartFinalization` · `useOverallBreakdownSynthesis` · `MockExamProcessingScreen` · `MockExamCelebrationScreen` · `MockExamQuotaPaywallScreen` / `MockExamPurchaseStatusScreen` · Report (#07–08) · Analytics (`mock_scoring_completed`, `ClickDetailFeedbackEvent` bắn tại thời điểm redirect, `MockExamFull.tsx:1135-1139`).

*Sơ đồ phù hợp: sequence diagram `completed` → Chúc mừng (song song: chấm nền bắt đầu) → 3600ms → Processing → nhánh rẽ (paywall / lỗi / tiếp tục chấm) → redirect report; state per-part (pending/processing/completed/failed).*

---

## 6.4 Xử lý ngoại lệ và trạng thái

| Trường hợp | Hành vi mong đợi |
|---|---|
| Lỗi mạng khi chấm 1 Part (C1), dẫn tới Part đó `status="failed"` | Processing **không** tự xử lý lỗi tại chỗ. Khi không còn Part nào đang chấm và có ≥1 Part `failed`, toàn bộ màn Processing bị thay bằng card đỏ "Một số part chưa hoàn tất chấm điểm" + nút "Mở màn hình chấm lại", điều hướng sang `/report/[sessionId]` — nơi cơ chế retry per-tab thật sự tồn tại (xem #07/#08). — `MockExamFull.tsx:1477-1499` |
| 1 Part chấm thất bại, các Part khác vẫn đang chấm | Không redirect, không hiện card lỗi — Processing tiếp tục hiển thị tiến độ bình thường cho tới khi không còn Part nào ở trạng thái `processing` |
| Hết lượt chấm giữa chừng (1 Part rơi vào `status="pending"`) | Toàn bộ màn Processing bị **thay thế** bằng `MockExamQuotaPaywallScreen` (chưa chọn gói) hoặc `MockExamPurchaseStatusScreen` (đã chọn gói, chờ xác nhận) — không phải một thông báo "pending" hiển thị trong Processing; user không thấy thanh tiến độ hay chữ "pending" nào một khi rơi vào trạng thái này. — `MockExamFull.tsx:1433-1473` |
| Chờ lâu bất thường | Vẫn trickle theo cơ chế BR-02 (trần 95% cho tới khi thật sự xong); chưa có ngưỡng timeout cụ thể để chủ động gợi ý retry — xem câu hỏi mở §6.7 |

**Các trạng thái giao diện:** Chúc mừng (pháo hoa + TTS, cố định 3.6s) → đang chấm (progress bar thật) → [nhánh rẽ] card lỗi (khi có Part failed) HOẶC paywall/purchase-status (khi có Part pending) → hoàn tất (auto-redirect sang report).

---

## 6.5 Yêu cầu giao diện (UI/UX)

- Mockup: `mocktest-flow-mockup.html` — scrProc, `goProc()`, `procError()`. Code thật: `MockExamProcessingScreen.tsx`, `MockExamFull.tsx`.
- Màn Chúc mừng: `MockExamCelebrationScreen`, pháo hoa + TTS lời kết giám khảo, giữ đúng 3.6 giây (không tôn trọng `prefers-reduced-motion` — xem §6.7/§6.8).
- Màn Processing: nền tối; icon trung tâm với vòng glow nhịp nhàng; tiêu đề "Đang xử lý bài thi của bạn..." (`#00D492`); phụ đề "Báo cáo của bạn sẽ sẵn sàng trong một lúc nữa..."; thanh "TIẾN ĐỘ PHÂN TÍCH" (label + %) với `role="progressbar"` đầy đủ `aria-valuenow/min/max/label`; các tiêu chí (Fluency, Cohesion, Intonation, Pronunciation, Lexical, Grammar) trôi nhẹ nền, chỉ hiện ở màn rộng (`lg:block`, ẩn trên mobile để không che nội dung). — `MockExamProcessingScreen.tsx:8-20, 68-121`
- Card lỗi (khi có Part failed): viền/nền rose, icon `AlertCircle`, tiêu đề "Một số part chưa hoàn tất chấm điểm", nút CTA xanh emerald "Mở màn hình chấm lại". — `MockExamFull.tsx:1482-1496`

---

## 6.6 Tiêu chí chấp nhận

**AC-01: Màn Chúc mừng hiển thị đúng thời lượng**
- **Given:** User vừa hoàn tất bài thi (`examState` chuyển `"completed"`)
- **When:** Hệ thống render `completionStage="celebrate"`
- **Then:** Màn Chúc mừng (pháo hoa + TTS lời kết giám khảo) hiển thị AND giữ đúng 3600ms sau khi TTS đọc xong (`isNarrating=false`) rồi mới chuyển `completionStage` sang `"processing"`

**AC-02: Chấm điểm chạy nền song song với Chúc mừng**
- **Given:** `examState` vừa chuyển `"completed"`
- **When:** Màn Chúc mừng đang hiển thị (`completionStage="celebrate"`)
- **Then:** Pipeline chấm điểm (`useSegmentedPartFinalization`, `useOverallBreakdownSynthesis`) đã bắt đầu chạy nền ngay, không chờ Chúc mừng kết thúc

**AC-03: Thanh tiến độ Processing bám tiến độ thật**
- **Given:** User đang ở màn Processing, các Part đang được chấm
- **When:** Tiến độ chấm thật thay đổi (theo `computeScoringProgress`)
- **Then:** Thanh tiến độ hiển thị nhích theo, không đứng yên quá lâu giữa 2 lần poll (trickle lấp khoảng trống, trần mốc thật + 6 và tối đa 95% cho tới khi thật sự xong)

**AC-04: Thanh tiến độ tôn trọng reduced-motion**
- **Given:** User bật `prefers-reduced-motion: reduce`
- **When:** User ở màn Processing
- **Then:** Thanh tiến độ không trickle/animate mượt — nhảy thẳng theo mốc tiến độ thật ở mỗi lần poll (120ms)

**AC-05: Chỉ redirect khi chấm xong hoàn toàn TẤT CẢ Part**
- **Given:** User ở màn Processing, `completionStage="processing"` (đã qua Chúc mừng)
- **When:** Một số Part đã `completed` với `evaluation` đầy đủ nhưng ít nhất 1 Part CHƯA hoàn tất (chưa `completed`, hoặc `completed` nhưng thiếu `evaluation`)
- **Then:** Hệ thống KHÔNG redirect sang report; report chỉ mở khi TẤT CẢ Part visible đều `status="completed"` VÀ có `evaluation`

**AC-06: Auto-redirect khi đủ điều kiện**
- **Given:** Tất cả Part visible đã `status="completed"` và có `evaluation` (`reportReady=true`), không có Part nào `pending` (`hasQuotaPending=false`), và `completionStage="processing"`
- **When:** Điều kiện trên được thoả đồng thời
- **Then:** Hệ thống tự động điều hướng sang `/report/[sessionId]`, không cần thao tác của user

**AC-07: Không redirect trong lúc còn đang ở màn Chúc mừng**
- **Given:** Tất cả Part đã chấm xong hoàn toàn (`reportReady=true`) NHƯNG `completionStage` vẫn là `"celebrate"`
- **When:** Hệ thống kiểm tra điều kiện redirect
- **Then:** Không redirect — phải chờ `completionStage` chuyển sang `"processing"` (Chúc mừng kết thúc) trước

**AC-08: Có Part lỗi → thay bằng card báo lỗi, không phải UI lỗi trong Processing**
- **Given:** Có ít nhất 1 Part `status="failed"` và không còn Part nào `status="processing"`
- **When:** Hệ thống render màn sau Chúc mừng
- **Then:** Toàn bộ màn Processing bị thay thế bằng card báo lỗi "Một số part chưa hoàn tất chấm điểm" kèm nút "Mở màn hình chấm lại" AND bấm nút điều hướng sang `/report/[sessionId]`

**AC-09: Hết lượt chấm → thay bằng Paywall/Purchase-Status, không hiển thị "pending"**
- **Given:** Có ít nhất 1 Part rơi vào `status="pending"` (`hasQuotaPending=true`)
- **When:** Hệ thống render màn sau Chúc mừng
- **Then:** Toàn bộ màn Processing bị thay thế bằng `MockExamQuotaPaywallScreen` (nếu chưa chọn gói) hoặc `MockExamPurchaseStatusScreen` (nếu đã chọn gói, chờ xác nhận) AND user không nhìn thấy thanh tiến độ hay bất kỳ chữ "pending" nào

**AC-10: Report không mở sớm kiểu skeleton khi đang chấm bình thường**
- **Given:** Một số Part đã có điểm số cơ bản (`evaluation` tồn tại) nhưng vocab/grammar review của Part đó chưa `completed`, và không có Part nào lỗi hay pending
- **When:** Hệ thống đánh giá điều kiện redirect
- **Then:** Không mở report sớm với skeleton cho phần chưa xong — vẫn ở màn Processing cho tới khi Part đó có `evaluation` đầy đủ (score+vocab+grammar)

---

## Non-functional Requirements

**Hiệu năng:** chấm xong 1 Part p50 ≤ 18s / p95 ≤ 30s; toàn bài p50 ≤ 45s / p95 ≤ 75s.

**Khả năng phục hồi:** lỗi chấm 1 Part không buộc làm lại cả bài — retry per-tab thật sự nằm ở Report (#08), Processing chỉ điều hướng tới đó qua card lỗi.

**Accessibility:**
- Thanh tiến độ ở Processing tôn trọng `prefers-reduced-motion` (đã xác nhận qua code — `MockExamProcessingScreen.tsx:30-34, 40-41`) và có đầy đủ `role="progressbar"` + `aria-valuenow/min/max/label`.
- Màn Chúc mừng/pháo hoa **không** áp dụng yêu cầu `prefers-reduced-motion` — quyết định chủ đích (đã cân nhắc, không phải thiếu sót), vì đây là animation ngắn (3.6s) chạy đúng 1 lần, rủi ro khó chịu/chóng mặt cho user nhạy cảm vận động thấp hơn nhiều so với một thanh tiến độ animate lặp lại liên tục. Xem §6.7, §6.8.

---

## User Types

Màn/luồng này không phân biệt theo gói (free/premium) ở giao diện Processing/Chúc mừng — mọi thí sinh đều trải qua cùng luồng. Khác biệt duy nhất theo gói xảy ra **sau khi rời khỏi Processing**: free user hết lượt chấm sẽ được điều hướng sang Paywall/Purchase-Status (xem FR-09), trong khi premium user (hoặc free user còn lượt) đi thẳng qua Processing tới Report.

| User Type | Definition | Feature Behavior |
|---|---|---|
| Mọi thí sinh (free còn lượt, premium) | Không có Part nào rơi vào `pending` trong suốt quá trình chấm | Trải qua Chúc mừng → Processing → auto-redirect Report khi đủ điều kiện, không có khác biệt theo gói |
| Free user hết lượt chấm giữa chừng | Ít nhất 1 Part rơi vào `status="pending"` do hết lượt chấm | Không thấy Processing hoàn tất bình thường — bị điều hướng sang Paywall/Purchase-Status ngay khi `hasQuotaPending=true` |

---

## 6.7 Phần bổ sung

**Giả định (Assumptions):** Pipeline trả trạng thái (`status`) và `evaluation` (bao gồm `partVocabReviewStatus`/`partGrammarReviewStatus`) theo từng Part để `computeScoringProgress` tính % chính xác.

**Phụ thuộc (Dependencies):** Pipeline chấm AI (`evaluate-mock-exam-segmented`, xem [scoring-pipeline-overview.md](../../../scoring-pipeline-overview.md)) · `useSegmentedPartFinalization` · `useOverallBreakdownSynthesis` · Report (#07–08, nơi chứa cơ chế skeleton/partial-state + retry per-tab thật sự cho trường hợp lỗi) · Billing/Paywall (`MockExamQuotaPaywallScreen`, `MockExamPurchaseStatusScreen` — chi tiết luồng thanh toán ngoài phạm vi tài liệu này).

**Đã cân nhắc, quyết định KHÔNG yêu cầu sửa (không phải câu hỏi mở):**
- Màn Chúc mừng/pháo hoa không tôn trọng `prefers-reduced-motion`, dù thanh tiến độ Processing cùng luồng đã làm đúng. Đã cân nhắc và **quyết định không thêm yêu cầu này**: pháo hoa chỉ chạy 1 lần, ngắn (3.6 giây), rủi ro gây khó chịu/chóng mặt cho user nhạy cảm vận động thấp hơn nhiều so với một thanh tiến độ animate liên tục lặp lại — không cần đồng bộ yêu cầu accessibility ở mức độ này. Xem Đổi log #8, §6.8.
- FR-06 cũ ("progressive reveal" — mở report sớm với skeleton) không còn là spec đích: đã xác nhận qua code và quyết định giữ hành vi "chờ chấm xong hoàn toàn tất cả Part rồi mới chuyển report" làm thiết kế chủ đích, không phải thiếu sót cần bổ sung sau. Xem Đổi log #5, §6.8.

**Câu hỏi mở / chưa chốt:**
- [ ] Thời gian timeout cho mỗi Part trước khi gợi ý retry — chưa xác định.
- [ ] Có cho người dùng rời màn Processing (chạy nền) và nhận thông báo khi xong không? — chưa chốt.

---

## 6.8 Câu hỏi mở — đã chốt (2026-07-07, grill-me)

| Câu hỏi | Đã chốt tại |
|---|---|
| FR-06 cũ ("progressive reveal") có nên tiếp tục coi là spec đích chưa build, hay sửa lại để mô tả đúng hành vi thật? | **Sửa lại để mô tả đúng hành vi thật, không còn coi là spec đích chưa build.** Đối chiếu code xác nhận: `reportReady` (`mock-exam-report.ts:54-60`) yêu cầu TẤT CẢ Part có `evaluation` đầy đủ (không chỉ điểm số) trước khi redirect — nghĩa là Processing luôn chờ chấm xong hoàn toàn rồi mới chuyển sang report. Đây là thiết kế chủ đích: Report (#07/#08) đã có cơ chế skeleton/trạng thái partial + retry per-tab riêng dành cho trường hợp LỖI thật (một Part gặp sự cố), không phải để mặc định mở sớm khi mọi thứ đang chấm bình thường; chờ đủ rồi chuyển tránh user thấy report "nửa vời" không cần thiết. Nếu sau này có nhu cầu tối ưu thời gian chờ cảm nhận bằng cách mở sớm thật sự, đó sẽ là 1 quyết định sản phẩm mới, không nên âm thầm giữ trong spec cũ như một mục "chưa build" — xem Đổi log #5, FR-06, BR-05, AC-05/06/07/10 |
| Có nên yêu cầu màn Chúc mừng (pháo hoa) tôn trọng `prefers-reduced-motion`, giống như thanh tiến độ cùng màn Processing đã làm đúng không? | **Không.** Đã cân nhắc và quyết định KHÔNG thêm yêu cầu này. Lý do: pháo hoa chỉ chạy 1 lần, ngắn (3.6 giây), rủi ro gây khó chịu/chóng mặt cho user nhạy cảm vận động thấp hơn nhiều so với 1 progress bar animate liên tục lặp lại — không cần đồng bộ yêu cầu accessibility ở mức độ này. Đây là điểm ĐÃ CÂN NHẮC VÀ QUYẾT ĐỊNH KHÔNG YÊU CẦU SỬA, không phải một câu hỏi mở còn treo — xem Đổi log #8, §6.7 |

Không còn câu hỏi mở nào đã được yêu cầu chốt cho §6 tại thời điểm này (2 câu hỏi còn lại ở §6.7 vẫn mở, chưa thuộc phạm vi phiên grill-me này).
