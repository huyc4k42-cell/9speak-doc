# Mock Test V2 — §7 Report — Tổng quan & Accordion tiêu chí — v0.2 (đối chiếu code thật + chốt qua grill-me)

> Ngày cập nhật: 2026-07-07 · Trạng thái: Chính thức · Đối chiếu [`ReportScoreAccordion.tsx`](../../../../features/mock-test/components/reports/ReportScoreAccordion.tsx), [`evaluate-mock-exam-overall-breakdown.ts`](../../../../features/mock-test/server/evaluate-mock-exam-overall-breakdown.ts), [`mock-exam-report.ts`](../../../../features/mock-test/lib/mock-exam-report.ts), [`useOverallBreakdownSynthesis.ts`](../../../../features/mock-test/hooks/useOverallBreakdownSynthesis.ts), [`cefr-mapping.ts`](../../../../features/shared/lib/cefr-mapping.ts) · Thay thế toàn bộ nội dung bản v0.1 (2026-06-24, chuyển thể + gộp từ Google Doc "Mocktest V2").

## Đổi log so với v0.1

| # | Thay đổi | Lý do |
|---|---|---|
| 1 | **Chốt nguồn 3 sub-score mỗi tiêu chí: AI sinh trực tiếp, không phải tính toán/derive phía client.** LLM trả về mảng đúng 3 số (`subScores: number[]`, ràng buộc bằng JSON schema) cho mỗi tiêu chí, được kẹp về thang IELTS (0-9, bước 0.5) ở server; client chỉ render nguyên giá trị, zip với nhãn cố định | Đối chiếu `evaluate-mock-exam-overall-breakdown.ts:38-53, 79-91` (schema), `:126-180` (hàm chạy), prompt tại `openai-prompts.json:51-52`; client render tại `ReportScoreAccordion.tsx:109-119` — đóng câu hỏi mở cũ "Nguồn 3 sub-score" |
| 2 | **Chốt nguồn descriptor: AI sinh động theo từng user, không phải bảng tra cứu tĩnh theo band do content team soạn sẵn.** LLM nhận `partsJson` (band + feedbackSummary thật của từng Part) làm ngữ cảnh, sinh free-text tiếng Việt "bám hiệu suất thực tế", ≤~55 từ; không có enum/lookup theo band | Đối chiếu `evaluate-mock-exam-overall-breakdown.ts:143-149`, prompt `openai-prompts.json:51` ("bám hiệu suất thực tế") — đóng câu hỏi mở cũ "Bộ descriptor tiếng Việt chuẩn theo band" |
| 3 | **Chốt mapping band→CEFR chính thức**, đã có sẵn trong code (không phải TBD) | `cefr-mapping.ts:16-38`, hàm `cefrFromOverallBand` — đóng câu hỏi mở cũ "Mapping band→CEFR chính thức" |
| 4 | Xác nhận qua code: **tiêu chí đầu tiên (Trôi chảy & Mạch lạc) luôn mặc định mở**, không phải "tiêu chí đầu" chung chung — thứ tự tiêu chí cố định theo mảng hiển thị, `fluencyAndCoherence` luôn ở vị trí 0 | `ReportScoreAccordion.tsx:36` (`useState(model.criteria[0]?.key ?? null)`), thứ tự tại `mock-exam-criteria-display.ts:17-42` |
| 5 | Xác nhận copy chính xác cho chế độ Part lẻ: header **"Band ước lượng" + " · Part X"** khi có số Part, chip cảnh báo amber đúng nguyên văn **"Chỉ luyện 1 phần — không phải band tổng chính thức"** | `ReportScoreAccordion.tsx:45-46, 54` |
| 6 | Xác nhận qua code: **trạng thái skeleton là theo từng field độc lập** (sub-score và descriptor mỗi tiêu chí chờ AI synthesis riêng, không đồng bộ với nhau) — band tổng/band tiêu chí luôn có ngay (tính đồng bộ từ các Part đã lock), không chờ AI | `ReportScoreAccordion.tsx:13-33` (`SubScoreSkeleton`, `DescriptorSkeleton`), `:109/122` (nhánh subScores), `:125/134` (nhánh descriptor); band tổng/band tiêu chí tại `mock-exam-report.ts:131-153` (`buildOverallCriteria`, tính synchronous) |
| 7 | Xác nhận **có cơ chế thử lại ngầm khi synthesis thất bại** (không phải mất vĩnh viễn), nhưng đây KHÔNG phải nút "Thử lại" hiển thị trên UI Tổng quan — cơ chế khác với retry per-tab thật sự ở #08. Việc có nút retry hiển thị riêng cho lỗi này hay không **chưa xác nhận được qua code**, giữ là câu hỏi mở thật sự (xem §7.7) | `useOverallBreakdownSynthesis.ts:75-78, 121-131` — `attemptedRef` xoá sessionId khi lỗi, cho phép thử lại ở lần mount kế tiếp (vd user rời rồi mở lại report) |
| 8 | **Quyết định (grill-me, 2026-07-07): giữ "Báo cáo mẫu" trong tài liệu như spec đích, đánh dấu rõ "chưa build/chưa xác nhận có trong roadmap"**, thay vì xoá hẳn — dù research không tìm thấy bằng chứng tính năng này tồn tại trong code (`grep` toàn repo không có kết quả). Đồng thời **sửa lại mô tả "Tiếp tục bài dở"**: hành vi thật là resume đúng theo `examMode` đã lưu khi bắt đầu (full hoặc Part lẻ), KHÔNG ép về full mode như suy đoán trước đó | Đối chiếu `useMockExamProgress.ts` (đọc `perExamStatus` từ localStorage) — xem §7.7 lý do đầy đủ |
| 9 | Chuẩn hoá cấu trúc tài liệu theo khung đang áp dụng cho #01/#03/#04/#05/#06: thêm mục **Goal / Non-functional Requirements / User Types**, đổi "Tiêu chí chấp nhận" từ bảng phẳng sang **định dạng Given/When/Then theo từng AC**, đổi số mục từ 1-7 sang 7.1-7.8, thêm **Đổi log** và **Câu hỏi mở — đã chốt** | Chuẩn hoá tài liệu theo yêu cầu product owner, đồng bộ với #01/#03/#04/#05/#06 |

---

## Goal

**Business/Product Goal:** Biến report từ "một con số" thành một trải nghiệm mang tính giáo dục — người học không chỉ thấy band tổng mà hiểu rõ *vì sao* đạt band đó ở từng tiêu chí, thông qua sub-score và descriptor được AI sinh riêng theo hiệu suất thực tế của họ (không phải nội dung tĩnh giống nhau cho mọi user cùng band).

**User Benefits:**
- Thí sinh: thấy ngay band tổng + CEFR mà không phải chờ AI (band tính đồng bộ từ các Part đã lock).
- Thí sinh: khi mở từng tiêu chí, thấy 3 sub-score và một đoạn mô tả band **bám sát bài làm thật của chính mình** — không phải một đoạn văn mẫu giống hệt cho mọi người cùng band 7.0.
- Thí sinh luyện Part lẻ: được cảnh báo rõ ràng đây chỉ là band ước lượng, tránh hiểu nhầm thành band tổng chính thức (IELTS không có khái niệm overall band cho 1 Part).
- Thí sinh gặp lỗi synthesis: không mất kết quả — band tổng/band tiêu chí vẫn hiển thị đầy đủ, chỉ riêng sub-score/descriptor tạm thời không có, và hệ thống tự thử lại ở lần mở report kế tiếp.

---

## 7.1 Tổng quan

**Mô tả ngắn:** Phần đầu của báo cáo: hiển thị **điểm tổng + CEFR** (chế độ Full) hoặc **band ước lượng + cảnh báo** (chế độ Part lẻ), cùng **accordion 4 tiêu chí** với 3 sub-score và descriptor tiếng Việt sinh bởi AI theo hiệu suất thực tế của từng user.

**Mục đích / vấn đề giải quyết:** Cho người học thấy ngay kết quả tổng và hiểu vì sao đạt band đó theo từng tiêu chí — mang tính giáo dục (giải thích rubric, cá nhân hoá theo bài làm thật) thay vì chỉ một con số hay một đoạn mô tả chung chung giống mọi người cùng band.

**Phạm vi:**
- Trong phạm vi: header điểm band/9.0 + CEFR (hoặc band ước lượng + cảnh báo cho Part lẻ); accordion 4 tiêu chí; 3 sub-score mỗi tiêu chí (AI sinh); descriptor mỗi tiêu chí (AI sinh); trạng thái mở/đóng; skeleton per-field khi đang chờ AI synthesis.
- Ngoài phạm vi: phản hồi chi tiết theo Part/câu & 4 panel phân tích (#08); retry per-tab thật sự (#08); trạng thái report đặc biệt (đang chấm/khoá — đề cập ngắn, chi tiết #06/#08).

**Đối tượng người dùng & bên liên quan:** Học viên; Product; Dev FE (`ReportScoreAccordion.tsx`) & BE (`evaluate-mock-exam-overall-breakdown.ts`); QA.

---

## 7.2 Yêu cầu chức năng

| Mã | Yêu cầu | Đầu vào | Đầu ra | Ghi chú |
|---|---|---|---|---|
| FR-01 | Hiển thị **điểm tổng** dạng band/9.0 + **CEFR** (chế độ Full) | `overallBand` | Header điểm | Band tính đồng bộ từ các Part đã lock, luôn có ngay — `mock-exam-report.ts:131-153` (`buildOverallCriteria`) |
| FR-02 | Map band→CEFR theo bảng chính thức | `overallBand` | `cefrLevel` | `cefrFromOverallBand`: ≤4.0→A2, 4.5-5.0→B1, 5.5-6.5→B2, 7.0-8.0→C1, ≥8.5→C2 — `cefr-mapping.ts:16-38` |
| FR-03 | Hiển thị **accordion 4 tiêu chí** với band mỗi tiêu chí | `criteria` | 4 hàng accordion | Band tiêu chí cũng tính đồng bộ, có ngay không cần chờ AI — `ReportScoreAccordion.tsx:88-96` |
| FR-04 | Khi mở một tiêu chí, hiển thị **3 sub-score AI sinh riêng cho tiêu chí đó** | `criterion.subScores` (mảng 3 số từ AI) | Danh sách 3 sub-score + nhãn cố định | AI trả `subScores: number[]` (đúng 3 phần tử, ràng buộc JSON schema), zip với nhãn cố định `subLabels` — `evaluate-mock-exam-overall-breakdown.ts:44-53, 126-180`; client render tại `ReportScoreAccordion.tsx:109-119` |
| FR-05 | Khi mở một tiêu chí, hiển thị **band descriptor tiếng Việt AI sinh riêng theo hiệu suất thực tế** | `criterion.descriptor` (free text từ AI) | Đoạn mô tả | AI nhận `partsJson` (band + feedbackSummary từng Part) làm ngữ cảnh, sinh descriptor ≤~55 từ "bám hiệu suất thực tế" — KHÔNG phải tra bảng band→text tĩnh — `evaluate-mock-exam-overall-breakdown.ts:143-149`, prompt `openai-prompts.json:51` |
| FR-06 | Cho **mở/đóng** từng tiêu chí; **mặc định tiêu chí đầu tiên (Trôi chảy & Mạch lạc) luôn tự mở** | Thao tác click | Trạng thái mở/đóng | `ReportScoreAccordion.tsx:36`: `useState(model.criteria[0]?.key ?? null)`; `criteria[0]` luôn là `fluencyAndCoherence` theo thứ tự cố định — `mock-exam-criteria-display.ts:17-42` |
| FR-07 | Khi report ở **chế độ luyện Part lẻ**, header đổi thành **"Band ước lượng" + " · Part X"** (khi có số Part) + chip cảnh báo amber **"Chỉ luyện 1 phần — không phải band tổng chính thức"** (thay cho band/9.0 + CEFR của full) | `model.isSinglePart`, `model.partNumber` | Header theo chế độ | Copy đúng nguyên văn — `ReportScoreAccordion.tsx:44-59` |
| FR-08 | Khi sub-score/descriptor của 1 tiêu chí chưa sẵn sàng (AI synthesis chưa xong), hiển thị **skeleton riêng cho từng field** (không chặn hiển thị band tiêu chí hay các tiêu chí khác) | `criterion.subScores`/`criterion.descriptor` = chưa có | `SubScoreSkeleton`/`DescriptorSkeleton` | `ReportScoreAccordion.tsx:13-33`, điều kiện render tại `:109/122` (subScores) và `:125/134` (descriptor) |
| FR-09 | Khi synthesis thất bại hẳn (không phải đang chờ), descriptor hiển thị thông báo tạm thay vì skeleton vô thời hạn | `model.breakdownStatus === "failed"` | "Chưa tải được nhận xét chi tiết cho tiêu chí này." | `ReportScoreAccordion.tsx:128-132` |

**Quy tắc nghiệp vụ (Business rules):**
- **BR-01:** 4 tiêu chí IELTS Speaking theo thứ tự cố định: Trôi chảy & Mạch lạc, Từ vựng, Ngữ pháp, Phát âm — `mock-exam-criteria-display.ts:17-42`.
- **BR-02:** Mỗi tiêu chí có đúng 3 sub-score cố định về nhãn (vd Trôi chảy & Mạch lạc → Độ trôi chảy / Liên kết ý / Phát triển chủ đề), nhưng **giá trị số của 3 sub-score do AI sinh riêng mỗi lần**, không phải công thức chia đều/derive tuyến tính từ band tiêu chí. AI được ràng buộc: trung bình 3 sub-score phải nằm trong 0.25 so với band tiêu chí đã lock, và không sub-score nào lệch band quá 1.0 — đây là ràng buộc nội dung prompt, không phải constraint code cứng ở tầng schema (schema chỉ ép đúng 3 số, kiểu number, không ép khoảng giá trị) — `evaluate-mock-exam-overall-breakdown.ts:38-53`, prompt `openai-prompts.json:51`.
- **BR-03:** Descriptor là **văn bản AI sinh động, cá nhân hoá theo bài làm thật** (grounded bằng `feedbackSummary` từng Part), không phải nội dung tĩnh tra theo mức band — mỗi user cùng band vẫn có thể nhận descriptor khác nhau về câu chữ tuỳ vào điểm mạnh/yếu cụ thể trong bài của họ.
- **BR-04:** Toàn bộ nhãn tiếng Việt; band/điểm là số IELTS (0–9, bước 0.5, kẹp qua `clampBand`/`roundBandScore`).
- **BR-05:** IELTS **không có overall band cho 1 part**; chế độ Part lẻ hiển thị **band ước lượng** (`model.overallBand` khi `isSinglePart=true`, suy từ 4 tiêu chí của part đó) kèm chip cảnh báo amber, không phải band tổng chính thức. Report đầy đủ (Full mock test) luôn hiển thị header band/9.0 + CEFR chuẩn.
- **BR-06 (mới):** Sub-score và descriptor của mỗi tiêu chí chờ **1 lần gọi AI synthesis toàn bài** (`useOverallBreakdownSynthesis`, gọi 1 lần cho cả 4 tiêu chí, không phải 4 lần riêng), kích hoạt ngay khi tất cả Part đã có band lock (không cần chờ vocab/grammar review xong) — idempotent theo `sessionId`, không gọi lại nếu đã `status="completed"`.
- **BR-07 (mới):** Nếu synthesis thất bại, hệ thống xoá sessionId khỏi tập "đã thử" nội bộ, cho phép tự thử lại vào lần mount kế tiếp (vd user rời report rồi quay lại) — đây là retry ngầm "khi mở lại", không phải nút "Thử lại" hiển thị trên UI Tổng quan (xem câu hỏi mở §7.7).

**Phân quyền theo vai trò:**

| Vai trò | Được làm gì |
|---|---|
| Chủ bài thi | Xem điểm tổng + chi tiết 4 tiêu chí (sub-score + descriptor cá nhân hoá theo bài làm của chính mình) |

---

## 7.3 Cơ chế hoạt động

**Luồng người dùng (workflow):**
1. Vào report (sau Processing #06, hoặc từ lịch sử; xem thêm mục "Báo cáo mẫu" ở §7.7 — chưa xác nhận có trong roadmap).
2. Thấy header điểm ngay lập tức (band tổng/CEFR hoặc band ước lượng — tính đồng bộ từ các Part, không cần chờ AI) + accordion 4 tiêu chí (tiêu chí đầu — Trôi chảy & Mạch lạc — mở sẵn).
3. Nếu AI synthesis (sub-score + descriptor) chưa xong, mỗi tiêu chí hiện skeleton riêng cho phần đang chờ; band tiêu chí vẫn hiển thị bình thường.
4. Bấm từng tiêu chí để xem 3 sub-score + descriptor khi đã sẵn sàng.

**Use case chính:**
- **Bối cảnh:** thí sinh muốn hiểu band tổng 7.5 đến từ đâu.
- **Kịch bản chính:** mở "Trôi chảy & Mạch lạc 8.0" → thấy 3 sub-score AI sinh (vd Độ trôi chảy 8.0, Liên kết ý 7.5, Phát triển chủ đề 8.5) + descriptor mô tả cụ thể điểm mạnh/yếu bám theo bài làm thật của người này → hiểu vì sao đạt band đó, không chỉ đọc một câu chung chung.
- **Kịch bản thay thế:** AI synthesis chưa xong khi user vừa vào report → sub-score/descriptor hiện skeleton; user bấm mở tiêu chí trong lúc đang chờ vẫn thấy skeleton chuyển sang dữ liệu thật khi AI trả xong (không cần refresh trang).
- **Kịch bản lỗi:** synthesis thất bại → descriptor hiện thông báo tạm "Chưa tải được nhận xét chi tiết cho tiêu chí này."; band tổng/band tiêu chí không bị ảnh hưởng; hệ thống tự thử lại synthesis ở lần mở report kế tiếp.

**Luồng dữ liệu:** Session (các Part đã lock band + feedbackSummary) → `buildOverallCriteria` tính band tiêu chí đồng bộ (`mock-exam-report.ts:131-153`) → hiển thị header/accordion ngay → song song, `useOverallBreakdownSynthesis` gọi `finalizeMockExamOverallBreakdown` 1 lần (khi đủ 4 Part có band) → server (`evaluate-mock-exam-overall-breakdown.ts`) gọi LLM với `overallBand`, `cefrLevel`, `criteria`, `subDimensions` (nhãn cố định), `partsJson` (band + feedbackSummary từng Part) → LLM trả JSON schema-constrained `{ subScores: number[3], descriptor: string }` cho mỗi tiêu chí → kẹp `subScores` về thang IELTS server-side (`clampBand`) → gắn nhãn cố định theo thứ tự → cập nhật session → client render sub-score/descriptor khi có, xoá skeleton.

**Tích hợp:** Session/report store (`useMockExamReportSession`) · `useOverallBreakdownSynthesis` (client hook điều phối gọi AI) · `evaluate-mock-exam-overall-breakdown.ts` (server, gọi LLM) · `cefr-mapping.ts` (map band→CEFR) · `mock-exam-criteria-display.ts` (thứ tự tiêu chí + nhãn sub-score cố định) · Analytics (`FeedbackScreenViewedEvent`).

*Sơ đồ phù hợp: không bắt buộc (UI tĩnh + accordion + 1 lần gọi AI nền).*

---

## 7.4 Xử lý ngoại lệ và trạng thái

| Trường hợp | Hành vi mong đợi / thông báo |
|---|---|
| Band tổng/band tiêu chí chưa có (case hiếm — thường không xảy ra vì Report chỉ mở khi tất cả Part đã `completed` + có `evaluation`, xem #06 BR-01) | Header hiển thị "--" (`formatBand` trả "--" khi `!hasAnyScores`) |
| Sub-score/descriptor của 1 tiêu chí chưa có (đang chờ AI synthesis) | Hiển thị `SubScoreSkeleton`/`DescriptorSkeleton` riêng cho field đó, không chặn phần còn lại của accordion — `ReportScoreAccordion.tsx:109-136` |
| AI synthesis thất bại hẳn (`breakdownStatus === "failed"`) | Descriptor hiển thị "Chưa tải được nhận xét chi tiết cho tiêu chí này." thay vì skeleton vô thời hạn; sub-score vẫn hiện skeleton nếu chưa có; hệ thống tự thử lại ở lần mở report kế tiếp (không phải nút bấm — xem §7.7 câu hỏi mở) |
| Không có session | Màn "Chưa có dữ liệu báo cáo" (chi tiết #08) |
| Chế độ Part lẻ | Header đổi hẳn sang "Band ước lượng · Part X" + chip cảnh báo amber, accordion 4 tiêu chí của Part đó giữ nguyên cơ chế sub-score/descriptor như Full |

**Các trạng thái giao diện:** header có ngay (band tổng/tiêu chí) → accordion mở sẵn tiêu chí đầu → mỗi tiêu chí độc lập giữa 2 field (sub-score/descriptor): skeleton hoặc dữ liệu thật hoặc (riêng descriptor) thông báo lỗi tạm.

---

## 7.5 Yêu cầu giao diện (UI/UX)

- Mockup: `mocktest-flow-mockup.html` — `scoreCard()`, `critRow()`. Code thật: `ReportScoreAccordion.tsx` (trong `MockExamReport.tsx` + `components/reports/*`, shell sáng có sidebar).
- Header chế độ Full: band/9.0 đậm (`text-3xl font-extrabold text-indigo-700`) + vạch ngăn dọc + CEFR. Accordion: chấm màu tròn theo `criterion.dot` + tên + band (`text-emerald-600`) + chevron; mở ra "Giám khảo đánh giá điều gì" + 3 sub-score (dot nhỏ + nhãn + điểm) + descriptor (đoạn text nhỏ, viền trên) — `ReportScoreAccordion.tsx:40-136`.
- **Theo chế độ Part lẻ:** `model.isSinglePart` → header "Band ước lượng" (chữ nhỏ, uppercase, `text-indigo-500`) + " · Part X" nếu có số Part + band lớn `text-indigo-700` + chip amber "Chỉ luyện 1 phần — không phải band tổng chính thức" (`bg-amber-50 text-amber-700`) — `ReportScoreAccordion.tsx:44-59`.
- Skeleton: thanh xám bo tròn, animate-pulse, 3 dòng cho sub-score / 2 dòng cho descriptor — `ReportScoreAccordion.tsx:13-33`.

---

## 7.6 Tiêu chí chấp nhận

**AC-01: Header Full hiển thị đúng band + CEFR**
- **Given:** Report ở chế độ Full (không phải Part lẻ), đã có `overallBand`
- **When:** User mở report
- **Then:** Header hiển thị `overallBand`/9.0 AND CEFR đúng theo mapping (`cefrFromOverallBand`)

**AC-02: Accordion mở sẵn tiêu chí đầu**
- **Given:** User mở report, accordion 4 tiêu chí đã render
- **When:** Report vừa load xong
- **Then:** Tiêu chí "Trôi chảy & Mạch lạc" (luôn là `criteria[0]`) tự động ở trạng thái mở AND 3 tiêu chí còn lại ở trạng thái đóng

**AC-03: Mở tiêu chí hiển thị 3 sub-score AI sinh + descriptor**
- **Given:** User bấm mở 1 tiêu chí, AI synthesis đã hoàn tất cho tiêu chí đó
- **When:** Tiêu chí mở ra
- **Then:** Hiển thị đúng 3 sub-score (giá trị do AI sinh, không phải tính lại phía client) với nhãn cố định tương ứng AND hiển thị descriptor tiếng Việt do AI sinh, nội dung phản ánh hiệu suất thực tế (không phải văn bản tĩnh giống nhau cho mọi user cùng band)

**AC-04: Bấm lại tiêu chí đang mở thì đóng**
- **Given:** 1 tiêu chí đang ở trạng thái mở
- **When:** User bấm lại đúng tiêu chí đó
- **Then:** Tiêu chí đóng lại (không có tiêu chí nào khác tự mở thay thế)

**AC-05: Skeleton per-field khi đang chờ AI synthesis**
- **Given:** User mở 1 tiêu chí trong lúc AI synthesis toàn bài chưa hoàn tất
- **When:** Tiêu chí đó chưa có `subScores` và/hoặc chưa có `descriptor`
- **Then:** Field thiếu hiển thị skeleton riêng (`SubScoreSkeleton`/`DescriptorSkeleton`) AND field đã có dữ liệu (nếu có) hiển thị bình thường, không bị khoá theo field kia

**AC-06: Chế độ Part lẻ hiển thị đúng header + cảnh báo**
- **Given:** Report đang ở chế độ luyện Part lẻ (`isSinglePart=true`)
- **When:** User mở report
- **Then:** Header hiển thị "Band ước lượng" + " · Part X" (nếu có số Part) thay cho band/9.0 + CEFR AND hiển thị chip cảnh báo đúng nguyên văn "Chỉ luyện 1 phần — không phải band tổng chính thức" AND accordion 4 tiêu chí vẫn hoạt động bình thường cho Part đó

**AC-07: Synthesis thất bại không chặn band đã có, hiện thông báo tạm cho descriptor**
- **Given:** AI synthesis toàn bài thất bại (`breakdownStatus="failed"`)
- **When:** User mở 1 tiêu chí chưa có descriptor
- **Then:** Band tổng và band tiêu chí vẫn hiển thị bình thường (không bị ảnh hưởng) AND descriptor hiển thị "Chưa tải được nhận xét chi tiết cho tiêu chí này." thay vì skeleton treo vô thời hạn

**AC-08: Synthesis tự thử lại ở lần mở report kế tiếp sau khi lỗi**
- **Given:** AI synthesis đã thất bại ở lần mở report trước đó, chưa có `subScores`/`descriptor` cho tiêu chí nào
- **When:** User rời report rồi mở lại (component mount lại) cho cùng session
- **Then:** Hệ thống tự động gọi lại synthesis (không cần thao tác thủ công của user) AND nếu thành công lần này, sub-score/descriptor hiển thị đầy đủ và không gọi lại nữa cho session đó

---

## Non-functional Requirements

**Performance:** mở report (đã chấm band xong, tức đã qua Processing #06) hiển thị header + band tiêu chí ≤ 1s p50 / ≤ 2.5s p95 — vì các giá trị này tính đồng bộ, không phụ thuộc AI. Sub-score/descriptor phụ thuộc thời gian gọi AI synthesis (1 lần cho cả 4 tiêu chí), hiển thị skeleton trong lúc chờ, không có SLA riêng đã công bố cho bước này.

**Khả năng phục hồi:** synthesis thất bại không làm mất band đã chấm; hệ thống tự thử lại ở lần mount kế tiếp (`useOverallBreakdownSynthesis.ts:126`), idempotent theo `sessionId` (không gọi lại khi đã `status="completed"`).

**Accessibility:**
- Nút mở/đóng tiêu chí có `aria-expanded` đúng trạng thái (`ReportScoreAccordion.tsx:85`).
- Skeleton đánh dấu `aria-hidden` để không gây nhiễu screen reader (`ReportScoreAccordion.tsx:14, 28`).
- Không dùng màu làm tín hiệu phân biệt duy nhất cho chip cảnh báo Part lẻ — luôn kèm text tường minh.

---

## User Types

| User Type | Definition | Feature Behavior |
|---|---|---|
| Thí sinh — report Full | Vừa hoàn tất hoặc xem lại 1 bài Full mock test | Thấy header band/9.0 + CEFR; accordion 4 tiêu chí với sub-score/descriptor cá nhân hoá theo bài làm thật |
| Thí sinh — report Part lẻ | Vừa hoàn tất hoặc xem lại 1 bài luyện Part riêng lẻ (Part 1/2/3) | Thấy header "Band ước lượng · Part X" + chip cảnh báo; accordion 4 tiêu chí hoạt động như Full nhưng band là ước lượng, không phải band tổng chính thức |
| Thí sinh — đang chờ AI synthesis | Vừa vào report ngay sau khi các Part vừa lock band, synthesis toàn bài chưa chạy xong | Thấy band tổng/band tiêu chí ngay; sub-score/descriptor hiện skeleton cho tới khi AI trả xong |
| Thí sinh — synthesis lỗi | AI synthesis toàn bài thất bại ở lần gọi trước | Band tổng/band tiêu chí không bị ảnh hưởng; descriptor hiện thông báo tạm; hệ thống tự thử lại ngầm ở lần mở report kế tiếp (chưa xác nhận có nút "Thử lại" hiển thị riêng cho case này hay không — xem §7.7) |

---

## 7.7 Phần bổ sung

**Giả định (Assumptions):** Pipeline chấm (#06) đã lock band + `feedbackSummary` cho từng Part trước khi `useOverallBreakdownSynthesis` được kích hoạt; cấu hình AI server-side (`resolveAiScoringConfig`) sẵn sàng khi gọi synthesis.

**Phụ thuộc (Dependencies):** Pipeline chấm (#06, cung cấp band + feedbackSummary từng Part) · Report chi tiết (#08, chứa retry per-tab thật sự, khác với retry ngầm của mục này) · `cefr-mapping.ts` · `mock-exam-criteria-display.ts`.

**Mục tiêu tương lai — "Báo cáo mẫu" (đích, chưa build — chưa xác nhận có trong roadmap):** Tài liệu v0.1 có nhắc tới một luồng vào report qua "nút Report mẫu" (report demo/xem trước không cần thi thật). Research đối chiếu code (tìm kiếm toàn repo) **không tìm thấy bằng chứng tính năng này tồn tại** — không có route, component, hay flag nào liên quan tới "báo cáo mẫu"/"report mẫu"/"sample report". Quyết định (grill-me, 2026-07-07): **giữ lại mục này trong tài liệu như một spec đích, đánh dấu rõ ràng "chưa build/chưa xác nhận có trong roadmap"**, thay vì xoá hẳn khỏi tài liệu. Lý do: đây có thể vẫn là tính năng có kế hoạch làm sau mà product owner biết rõ hơn agent nghiên cứu code; xoá hẳn có rủi ro mất dấu một ý định sản phẩm thật, trong khi giữ lại kèm đánh dấu trạng thái "chưa xác nhận" là lựa chọn an toàn hơn. Product owner cần xác nhận riêng liệu tính năng này có còn nằm trong roadmap hay không.

**"Tiếp tục bài dở" — hành vi thật đã xác nhận (khác với suy đoán cũ):** Route vào report sau khi resume 1 bài đang làm dở **không ép về chế độ Full**. `useMockExamProgress` đọc `perExamStatus` theo `catalogId` từ localStorage; khi resume, session được tải lại đúng theo `examMode` (full hoặc Part lẻ) mà bài đó **đã được bắt đầu từ đầu** — nếu user bắt đầu ở chế độ Part lẻ, resume vẫn tiếp tục ở Part lẻ, không tự chuyển sang Full.

**Câu hỏi mở / chưa chốt (không đoán, để lại thật sự mở):**
- [ ] Có tồn tại nút/UI "Thử lại" hiển thị riêng cho user khi AI synthesis (sub-score/descriptor) thất bại trên section Tổng quan hay không — research chỉ xác nhận có cơ chế tự thử lại ngầm ở lần mở report kế tiếp (`useOverallBreakdownSynthesis.ts:126`), KHÔNG xác nhận được có/không có nút bấm chủ động cho riêng trường hợp này (khác với retry per-tab đã xác nhận có ở #08).
- [ ] "Báo cáo mẫu" có thực sự nằm trong roadmap sản phẩm hay không — cần product owner xác nhận trực tiếp, ngoài phạm vi research code.

---

## 7.8 Câu hỏi mở — đã chốt (2026-07-07, grill-me + đối chiếu code)

| Câu hỏi | Đã chốt tại |
|---|---|
| Nguồn 3 sub-score (model trả riêng hay derive từ tiêu chí tổng?) | **AI trả riêng, không derive.** LLM sinh đúng 3 số/tiêu chí qua JSON schema-constrained call, kẹp về thang IELTS server-side; client chỉ render — xem Đổi log #1, FR-04, BR-02, `evaluate-mock-exam-overall-breakdown.ts:44-53, 126-180` |
| Bộ descriptor tiếng Việt chuẩn theo band (nguồn nội dung) | **Không phải bộ nội dung tĩnh theo band — AI sinh động, cá nhân hoá theo hiệu suất thực tế của từng user** (grounded bằng `feedbackSummary` từng Part) — xem Đổi log #2, FR-05, BR-03, `evaluate-mock-exam-overall-breakdown.ts:143-149` |
| Mapping band→CEFR chính thức | **Đã có sẵn trong code, không phải TBD:** ≤4.0→A2, 4.5-5.0→B1, 5.5-6.5→B2, 7.0-8.0→C1, ≥8.5→C2 — xem Đổi log #3, FR-02, `cefr-mapping.ts:16-38` |
| "Báo cáo mẫu" có nên giữ trong tài liệu khi không tìm thấy bằng chứng trong code không, hay xoá hẳn? | **Giữ lại như spec đích, đánh dấu rõ "chưa build/chưa xác nhận có trong roadmap"** thay vì xoá hẳn — vì đây có thể vẫn là ý định sản phẩm thật mà product owner biết rõ hơn agent nghiên cứu code; xoá hẳn có rủi ro mất dấu, giữ lại có đánh dấu trạng thái rõ ràng là lựa chọn an toàn hơn — xem Đổi log #8, §7.7 |

Còn lại 2 câu hỏi mở thật sự chưa chốt cho §7 (nút retry hiển thị cho lỗi synthesis; "Báo cáo mẫu" có trong roadmap không) — xem §7.7, cần xác nhận thêm ngoài phạm vi research code lần này.
