# Mock Test V2 — §8 Report — Phản hồi chi tiết theo Part/câu — v0.2 (đối chiếu code thật)

> Ngày cập nhật: 2026-07-07 · Trạng thái: Chính thức · Đối chiếu [`MockExamReportSections.tsx`](../../../../features/shared/components/reports/MockExamReportSections.tsx), [`MockExamReportNavigation.tsx`](../../../../features/shared/components/reports/MockExamReportNavigation.tsx), [`MockExamReport.tsx`](../../../../features/mock-test/pages/reports/MockExamReport.tsx), [`useMockExamReportInteractions.ts`](../../../../features/shared/hooks/useMockExamReportInteractions.ts), [`evaluation-pipeline.ts`](../../../../features/shared/server/evaluation-pipeline.ts), [`mock-exam.ts`](../../../../features/shared/types/mock-exam.ts), [`MockExamPronunciationDrillModal.tsx`](../../../../features/mock-test/components/reports/MockExamPronunciationDrillModal.tsx), [`mock-exam-drill-client.ts`](../../../../features/mock-test/lib/mock-exam-drill-client.ts) · Thay thế toàn bộ nội dung bản v0.1 (2026-06-20, chuyển thể + gộp từ Google Doc "Mocktest V2"). Đây là màn cuối cùng trong 9 màn hình cốt lõi được nâng cấp theo quy trình research + viết lại này — xem [screens/README.md](README.md).

## Đổi log so với v0.1

| # | Thay đổi | Lý do |
|---|---|---|
| 1 | **Xác nhận qua code: "Nhóm theo âm" (phoneme grouping) là output của pipeline phía server, không phải tính toán client, và là bước tổng hợp/heuristic (không phải AI phân tích lại phát âm).** Client chỉ đọc thẳng `evaluation.pronunciationProfile.topWeakSounds` — không có logic gom nhóm nào chạy ở client | `mock-exam.ts:359-369` (kiểu `MockExamPronunciationSoundInsight`: `sound`, `averageAccuracy`, `occurrenceCount`, `exampleWord`, `exampleWordScore`, `exampleIpa`); đọc tại `MockExamReportSections.tsx:2013`. Đóng câu hỏi mở cũ "Nguồn Nhóm theo âm" — **độ tin cậy**: chắc chắn không phải client-side, chắc chắn không phải LLM call giống highlight ngữ pháp/từ vựng; chưa xác nhận 100% chi tiết bước gom nhóm trong pipeline là hard-code hay có thành phần khác |
| 2 | **Xác nhận qua code: "Cấu trúc nâng cao" (`complexStructureUsage`) là AI sinh, từ LLM call review ngữ pháp** — JSON schema ép đúng hình `{ attempted: string[], successful: string[], failed: string[] }` | `evaluation-pipeline.ts:784-792` (schema); sinh bởi `runGrammarReview()` (`:880-906`) qua `requestStructuredOpenAiJson()` (OpenAI structured output). Đóng câu hỏi mở cũ "Nguồn Cấu trúc nâng cao" |
| 3 | **Xác nhận qua code: nghe lại chính xác (`onPlayPrecise`) seek + phát đúng đoạn từ đó, dùng timestamp cấp-từ của Azure** — không phải phát cả câu | `useMockExamReportInteractions.ts:34-82` (`playAudioSegment`): tính `playbackStartSeconds`/`playbackEndSeconds` tương đối theo `startSeconds` của segment audio chứa từ đó (`:58-60`), seek `audio.currentTime` tới đầu từ (`:72`), dừng lại đúng cuối từ bằng `setTimeout` (`:76-78`, tối thiểu 180ms). Biên từ lấy từ `weakWord.precisePlaybackStartSeconds`/`precisePlaybackEndSeconds` (dùng tại `MockExamReportSections.tsx:987-992`) — đây là timestamp cấp-từ do chính Azure trả, không phải ước lượng phía client. Đóng câu hỏi mở cũ "Cơ chế precise playback" |
| 4 | **Xác nhận qua code: mini-drill "Luyện ›" là modal ghi âm lại → chấm qua Azure ở route riêng `.../drills/pronunciation`, MIỄN PHÍ (không trừ lượt), và kết quả là điểm luyện tập MỘT LẦN, không ghi ngược vào điểm gốc của report.** Đây là hành vi đúng/thiết kế có chủ đích (giữ toàn vẹn điểm thi đã chốt), không phải bug | `MockExamPronunciationDrillModal.tsx:30-164` (modal); `mock-exam-drill-client.ts:6,16` (endpoint `/api/app/speaking/mock-exams/drills/pronunciation` + comment "không trừ lượt chấm"). Kết quả (`MockExamPronunciationDrillModal.tsx:120-125`) hiển thị standalone trong modal, không có đường ghi nào quay lại `weakWord` gốc của report. Đóng câu hỏi mở cũ "Luyện › dẫn tới đâu" |
| 5 | **Xác nhận qua code: Retry (Thử lại) chỉ áp dụng riêng theo từng tab (vocab/grammar), không theo cả Part.** Retry Từ vựng của 1 Part không kích hoạt lại Ngữ pháp của Part đó, và ngược lại — 2 lời gọi hoàn toàn độc lập | `MockExamReportNavigation.tsx:147-173`: mỗi tab vocab/grammar có field trạng thái riêng (`partVocabReviewStatus`/`partGrammarReviewStatus`), bọc trong `<ProgressiveFinalizeSection>` riêng với `onRetry`/`isRetrying` riêng; `MockExamReport.tsx:613-621` gọi `segmentedRetry.retryPartVocabFinalization()` và `retryPartGrammarFinalization()` như 2 hàm độc lập hoàn toàn. Tab Trôi chảy/Phát âm không có nút retry riêng — 2 tab này lấy dữ liệu từ cùng bài chấm cơ bản (đã có sẵn hoặc không, cùng lúc với band), chỉ Từ vựng/Ngữ pháp là 2 bước finalize tiến triển tách rời có thể lỗi độc lập |
| 6 | **Xác nhận qua code: chế độ Part lẻ ẩn bộ chọn Part bằng cách gỡ hẳn khỏi DOM, không phải ẩn bằng CSS.** Khi chỉ có 1 Part hiển thị, component bộ chọn Part trả về `null`, không mount | `MockExamReport.tsx:484`: `{visibleParts.length > 1 ? <MockExamReportPartTabs .../> : null}` |
| 7 | **Bổ sung bảng tổng hợp nguồn dữ liệu theo từng tiêu chí** (rule/heuristic vs AI-sinh vs pipeline-tổng hợp vs tĩnh) — làm rõ cho người đọc sản phẩm phần nào là "máy tính ra", phần nào là "AI viết", phần nào là "cố định sẵn" | Tổng hợp từ Đổi log #1-2 + code đối chiếu trực tiếp từng field — xem bảng mới trong §8.2 |
| 8 | Chuẩn hoá cấu trúc tài liệu theo khung đang áp dụng cho #01/#07: thêm mục **Goal / Non-functional Requirements / User Types**, đổi "Tiêu chí chấp nhận" từ bảng phẳng sang **định dạng Given/When/Then theo từng AC**, đổi số mục từ 1-7 sang 8.1-8.8, thêm **Đổi log** và **Câu hỏi mở — đã chốt** | Chuẩn hoá tài liệu theo yêu cầu product owner, đồng bộ với #01/#07. Không có quyết định sản phẩm mới ở vòng này — toàn bộ 4 câu hỏi mở cũ đã được research đối chiếu code trả lời dứt điểm, không cần grill-me thêm |

---

## Goal

**Business/Product Goal:** Biến điểm số thành lộ trình cải thiện cụ thể — người học không chỉ biết "band bao nhiêu" mà thấy đúng chỗ cần sửa theo từng Part/từng câu/từng tiêu chí, kèm hành động khả thi ngay tại chỗ (nghe lại đúng đoạn phát âm sai, luyện lại miễn phí, xem gợi ý paraphrase).

**User Benefits:**
- Thí sinh: soi được lỗi ở đúng từ/đúng câu thay vì chỉ đọc nhận xét chung chung — highlight theo tiêu chí (màu CEFR, mức phát âm, đơn/phức ngữ pháp).
- Thí sinh: nghe lại chính xác đoạn mình phát âm sai (không phải nghe lại cả câu rồi tự đoán) nhờ timestamp cấp-từ của Azure.
- Thí sinh: luyện lại từ khó ngay tại chỗ, miễn phí, không lo ảnh hưởng tới điểm bài thi đã chốt (mini-drill không ghi đè điểm gốc).
- Thí sinh luyện Part lẻ: không bị rối bởi bộ chọn Part 3 phần khi chỉ thi 1 phần — giao diện gọn, chỉ hiện đúng phần đã làm.
- Thí sinh gặp lỗi ở bước phân tích Từ vựng hoặc Ngữ pháp: có thể thử lại đúng phần bị lỗi mà không cần chờ/ảnh hưởng phần còn lại.

---

## 8.1 Tổng quan

**Mô tả ngắn:** Phần phản hồi chi tiết của báo cáo: chọn **Part** → xem **từng câu hỏi/câu trả lời**, với 4 tab tiêu chí đổi cả cách hiển thị câu trả lời lẫn **panel phân tích chuyên sâu** bên phải.

**Mục đích / vấn đề giải quyết:** Giúp người học soi đúng chỗ cần sửa theo từng tiêu chí và từng câu, kèm gợi ý hành động (paraphrase, luyện âm, sửa cấu trúc) — biến điểm số thành lộ trình cải thiện.

**Phạm vi:**
- Trong phạm vi: bộ chọn Part 1/2/3; cue card Part 2; danh sách câu hỏi–câu trả lời + audio từng câu; 4 tab tiêu chí; cách render câu trả lời theo tab; 4 panel phân tích (Trôi chảy/Mạch lạc, Từ vựng, Phát âm, Ngữ pháp); mini-drill luyện phát âm; retry per-tab (vocab/grammar); các trạng thái report.
- Ngoài phạm vi: header điểm + accordion tổng quan (#07); pipeline chấm điểm (technical, xem `docs/scoring-pipeline-overview.md`); các hạng mục học thuật chưa build (xem [gap.md](../gap.md)).

**Đối tượng người dùng & bên liên quan:** Học viên; Product; Dev FE (`MockExamReportSections.tsx`, `MockExamReportNavigation.tsx`); QA.

---

## 8.2 Yêu cầu chức năng

| Mã | Yêu cầu | Đầu vào | Đầu ra | Ghi chú |
|---|---|---|---|---|
| FR-01 | Cho **chọn Part 1/2/3** và đổi danh sách câu tương ứng | Chọn part | Danh sách câu của part | |
| FR-02 | Part 2 phải hiển thị **cue card** | Đề Part 2 | Khung cue card | |
| FR-03 | Hiển thị **từng câu**: Câu N + câu hỏi + câu trả lời + **player audio** | questionRuns | Khối câu hỏi–trả lời | |
| FR-04 | Có **4 tab tiêu chí**; đổi tab đổi cả render trái + panel phải | Chọn tab | UI tương ứng | |
| FR-05 | Tab **Trôi chảy**: render câu trả lời với pause 3 mức + filler; panel tốc độ nói + Mạch lạc + chips | Dữ liệu fluency | Transcript + panel | |
| FR-06 | Tab **Từ vựng**: tô màu từ theo CEFR; panel phân bố CEFR + gợi ý paraphrase | Dữ liệu lexical | Transcript + panel | |
| FR-07 | Tab **Phát âm**: highlight từ yếu; panel lỗi từng từ (✓Đúng/✗Bạn nói + nghe + Luyện) + nhóm theo âm | Dữ liệu pronunciation | Transcript + panel | |
| FR-08 | Tab **Ngữ pháp**: câu-theo-câu (nhãn đơn/phức + có lỗi/không); panel độ chính xác + cấu trúc nâng cao + lỗi theo loại + gợi ý | Dữ liệu grammar | Sentence cards + panel | |
| FR-09 | Việt hoá nhãn UI; giữ tiếng Anh nội dung nói/câu hỏi | — | UI tiếng Việt | |
| FR-10 | Hỗ trợ trạng thái report đặc biệt (đang chấm/chờ thủ công/khoá/lỗi) + retry | Trạng thái session | Banner/màn tương ứng | |
| FR-11 | Khi report ở **chế độ luyện Part lẻ**, **gỡ hẳn khỏi DOM** (không chỉ ẩn CSS) bộ chọn Part 1/2/3, chỉ hiển thị phản hồi chi tiết của **phần đã thi** | `visibleParts.length` | Chỉ 1 Part, không mount component chọn Part | `MockExamReport.tsx:484`: `{visibleParts.length > 1 ? <MockExamReportPartTabs .../> : null}` |
| FR-12 | Nghe lại chính xác đoạn phát âm sai (`onPlayPrecise`): seek + phát đúng đoạn từ đó theo timestamp cấp-từ của Azure, không phát cả câu | `weakWord.precisePlaybackStartSeconds/EndSeconds` | Audio phát đúng đoạn từ, tối thiểu 180ms | `useMockExamReportInteractions.ts:34-82`, dùng tại `MockExamReportSections.tsx:987-992` |
| FR-13 | Mini-drill "Luyện ›": mở modal ghi âm lại 1 từ → chấm qua Azure ở route riêng, **miễn phí, không trừ lượt**; kết quả hiển thị standalone, **không ghi ngược** vào điểm/report gốc | Từ cần luyện + ghi âm mới | Điểm luyện tập một lần (modal) | `MockExamPronunciationDrillModal.tsx:30-164`, endpoint `mock-exam-drill-client.ts:6,16` |
| FR-14 | Retry chỉ áp dụng **riêng theo từng tab** (vocab/grammar) trong 1 Part — không kéo theo tab kia | `partVocabReviewStatus` / `partGrammarReviewStatus` | Chỉ tab được chọn chạy lại | `MockExamReportNavigation.tsx:147-173`, `MockExamReport.tsx:613-621` (`retryPartVocabFinalization`/`retryPartGrammarFinalization` độc lập) |

**Bảng nguồn dữ liệu theo tiêu chí (xác nhận qua code — mức tin cậy khác nhau theo từng dòng):**

| Tiêu chí | Thành phần | Nguồn |
|---|---|---|
| **Trôi chảy (Fluency)** | Phân loại 3 mức ngắt (pause tier) | Rule/heuristic — `resolvePauseTier` |
| | Đếm filler ("uhm"...) | Server đếm sẵn (không phải LLM) |
| | Danh sách liên từ đã dùng (cohesive devices) | AI sinh — LLM call review ngữ pháp/trôi chảy |
| | Gợi ý hiển thị cạnh liên từ | Copy tĩnh/hardcode, không cá nhân hoá |
| **Từ vựng (Vocab)** | Màu/mức CEFR theo từng từ | Pipeline gán sẵn (bước phân loại CEFR theo từ, nằm trước khi tới màn này) |
| | Biểu đồ phân bố CEFR | Derive từ mức CEFR theo từ ở trên |
| | Gợi ý paraphrase (`replacementOptions`) | AI sinh — LLM call review từ vựng |
| **Ngữ pháp (Grammar)** | Nhãn câu đơn/phức | Heuristic **client-side** (`classifySentenceComplexity`, regex — vd phát hiện dấu hiệu mệnh đề phụ) — **không phải AI, không phải server** |
| | Highlight lỗi ngữ pháp + `complexStructureUsage` (Cấu trúc nâng cao) | AI sinh — LLM call review ngữ pháp (`runGrammarReview`, `evaluation-pipeline.ts:784-792, 880-906`) |
| **Phát âm (Pronunciation)** | Phân loại 3 mức theo từ (tốt/khá/cần cải thiện) | Rule ngưỡng đơn giản (≥75 / 50-75 / <75) áp lên điểm từ do Azure trả — không phải LLM |
| | Danh sách từ yếu + điểm phoneme | Output đánh giá của Azure |
| | "Nhóm theo âm" (`topWeakSounds`) | Tổng hợp phía **server**, gom điểm phoneme Azure theo âm — không phải LLM, không phải client (Đổi log #1) |

**Quy tắc nghiệp vụ (Business rules):**
- **BR-01:** Mỗi Part = một run state; mỗi câu = một phần tử `questionRuns` (transcript + assessment).
- **BR-02:** Cách render câu trả lời phụ thuộc tab tiêu chí (pause/CEFR/highlight/sentence-card).
- **BR-03:** Panel phân tích phản ánh đúng tiêu chí đang chọn.
- **BR-04:** Report mở được khi đủ điều kiện; phần còn chấm hiển thị skeleton (progressive reveal).
- **BR-05:** Lỗi/Pending một Part → cho retry Part đó (không buộc làm lại cả bài); trong 1 Part, retry Từ vựng và retry Ngữ pháp là 2 hành động độc lập hoàn toàn, không có liên đới (xem FR-14, Đổi log #5).
- **BR-06:** **Chế độ thi (`examMode`, #01/#04)** quyết định phạm vi report: full → đủ bộ chọn Part 1/2/3; Part lẻ → bộ chọn Part bị gỡ hẳn khỏi DOM (`null`), chỉ phần đã thi.
- **BR-07 (mới):** Mini-drill "Luyện ›" không bao giờ ghi đè hay cập nhật lại điểm/`weakWord` gốc trong report — đây là hành vi đúng/có chủ đích để giữ toàn vẹn điểm bài thi đã chốt (một report đã khoá không nên bị "thổi phồng" bởi luyện tập sau đó), không phải thiếu sót cần sửa.
- **BR-08 (mới):** "Nhóm theo âm" là kết quả tổng hợp phía server từ điểm phoneme Azure theo từng câu trả lời (gom theo âm, tính điểm trung bình + số lần xuất hiện) — không phải một lời gọi AI phân tích lại phát âm, và không tính toán ở client.

**Phân quyền theo vai trò:**

| Vai trò | Được làm gì |
|---|---|
| Chủ bài thi | Xem chi tiết theo Part/câu/tiêu chí; nghe lại audio (kể cả nghe chính xác đoạn phát âm sai); luyện lại từ khó miễn phí; retry per-tab (vocab/grammar) khi lỗi |

---

## 8.3 Cơ chế hoạt động

**Luồng người dùng (workflow):**
1. Cuộn xuống "Phản hồi chi tiết".
2. Chọn Part (1/2/3) — hoặc bỏ qua bước này nếu đang ở chế độ Part lẻ (bộ chọn không tồn tại) → thấy danh sách câu của part.
3. Chọn tab tiêu chí → câu trả lời + panel đổi theo.
4. Nghe lại audio từng câu (nguyên câu) hoặc nghe chính xác 1 từ phát âm sai (`onPlayPrecise`, seek theo timestamp Azure); xem gợi ý cải thiện (paraphrase/luyện âm/sửa cấu trúc).
5. Ở tab Phát âm, bấm "Luyện ›" trên 1 từ yếu → mở modal ghi âm lại → chấm Azure miễn phí → thấy điểm luyện tập một lần (không đổi điểm report gốc).
6. Nếu tab Từ vựng hoặc Ngữ pháp của 1 Part lỗi → bấm "Thử lại" đúng tab đó — không ảnh hưởng tab kia.

**Use case chính:**
- **Bối cảnh:** thí sinh muốn biết Part 1 sai phát âm chỗ nào.
- **Kịch bản chính (happy path):** chọn Part 1 → tab Phát âm → thấy từ yếu highlight + lưới lỗi từng từ (✓Đúng/✗Bạn nói + nghe) + nhóm theo âm → bấm nút nghe để nghe lại đúng đoạn mình phát âm sai (không phải cả câu) → bấm "Luyện ›" → ghi âm lại → thấy điểm luyện tập ngay trong modal.
- **Kịch bản thay thế / ngoại lệ:** một Part còn đang chấm → panel hiện skeleton; tab Từ vựng của 1 Part lỗi trong khi tab Ngữ pháp của Part đó vẫn ổn → chỉ tab Từ vựng hiện nút retry, tab Ngữ pháp không bị ảnh hưởng.

**Luồng dữ liệu:** Session đã chấm → mỗi Part: `questionRuns[].assessment` (transcript, transcriptSentences), criteria, `speechRateWpm`, `lexicalProfile`, `grammarProfile.highlights` + `complexStructureUsage` (AI sinh), `pronunciationProfile.weakWords` + `topWeakSounds` (server tổng hợp từ Azure), annotations → render theo tab. Nghe chính xác 1 từ: `weakWord.precisePlaybackStartSeconds/EndSeconds` (timestamp Azure) → `playAudioSegment` tính offset tương đối theo segment audio chứa từ → seek + auto-stop.

**Tích hợp:** Report store · pipeline segmented (vocab/grammar/word-cefr finalize nền, `evaluation-pipeline.ts`) · audio playback (`useMockExamReportInteractions.ts`) · route mini-drill (`/api/app/speaking/mock-exams/drills/pronunciation`) · Analytics (`FeedbackPartViewedEvent`, `FeedbackCriteriaViewedEvent`, `ClickListenPronuncErrorEvent`).

*Sơ đồ phù hợp: component/data-binding diagram (tiêu chí → render trái + panel phải).*

---

## 8.4 Xử lý ngoại lệ và trạng thái

| Trường hợp | Hành vi mong đợi / thông báo |
|---|---|
| 1 Part còn đang chấm | Banner "Báo cáo tự làm mới" + skeleton phần đó; tự cập nhật |
| Hết lượt → chờ chấm thủ công | Banner amber + "Chấm lại toàn bộ bài thi" |
| Chưa đủ 3 Part / dừng giữa chừng | Màn "Chưa thể xem báo cáo" + về Dashboard |
| Không có session | Màn "Chưa có dữ liệu báo cáo" |
| 1 Part lỗi (failed) — riêng tab Từ vựng hoặc Ngữ pháp | Trong tab đó: thông báo lỗi + nút retry **chỉ cho tab đang lỗi** (không kéo theo tab còn lại của cùng Part) |
| Mở từ lịch sử chưa chấm đủ | Modal gợi ý "Chấm lại toàn bộ" |
| Mini-drill lỗi (không truy cập được mic / chấm lỗi) | Modal hiện thông báo lỗi tại chỗ, cho ghi âm lại; không ảnh hưởng report gốc |

**Các trạng thái giao diện:** loading (skeleton panel) / đủ dữ liệu / đang chấm nền / chờ thủ công / khoá / rỗng / lỗi.

---

## 8.5 Yêu cầu giao diện (UI/UX)

- Mockup: `mocktest-flow-mockup.html` — `feedbackCard()`, `answersCol()`, `answerBody()`, `flPanel()`/`vocabPanel()`/`pronPanel()`/`gramPanel()`. Code thật: `MockExamReportSections.tsx` + `MockExamReportNavigation.tsx`.
- Bộ chọn Part dạng pill; cue card Part 2; mỗi câu là một khối (Câu N + câu hỏi + trả lời + mini player). 4 tab tiêu chí có chấm màu (`MockExamReportNavigation.tsx:180-185`: Trôi chảy tím, Từ vựng xanh ngọc, Phát âm cam, Ngữ pháp đỏ); panel phải đổi theo tiêu chí.
- **Theo chế độ:** chế độ Part lẻ (`visibleParts.length === 1`) gỡ hẳn component bộ chọn Part khỏi DOM (`MockExamReport.tsx:484`), đặt Part cố định theo phần đã thi.
- **Panel Phát âm:** nút nghe "Bạn nói" gọi `onPlayPrecise` seek đúng đoạn từ (không phát cả câu) — `MockExamReportSections.tsx:987-992`.

---

## 8.6 Tiêu chí chấp nhận

**AC-01: Đổi Part cập nhật đúng danh sách câu**
- **Given:** User đang xem report ở chế độ full (có bộ chọn Part)
- **When:** User chọn Part khác
- **Then:** Danh sách câu hỏi/câu trả lời đổi đúng theo Part vừa chọn

**AC-02: Part 2 hiển thị cue card**
- **Given:** User đang xem Part 2 trong report
- **When:** Part 2 render
- **Then:** Hiển thị khung cue card đúng đề Part 2 đó

**AC-03: Mỗi câu hiển thị đủ thành phần**
- **Given:** User xem 1 câu trong danh sách
- **When:** Câu đó render
- **Then:** Hiển thị Câu N + câu hỏi + câu trả lời + player audio riêng cho câu đó

**AC-04: Đổi tab tiêu chí đổi cả render trái và panel phải**
- **Given:** User đang xem 1 tab tiêu chí bất kỳ
- **When:** User chọn tab tiêu chí khác
- **Then:** Cả cách render câu trả lời (trái) và panel phân tích (phải) đổi đúng theo tiêu chí mới chọn

**AC-05: Tab Phát âm hiển thị lỗi từng từ đầy đủ**
- **Given:** User ở tab Phát âm, có ít nhất 1 từ yếu
- **When:** Panel Phát âm render
- **Then:** Liệt kê lỗi từng từ với ✓Đúng/✗Bạn nói + nút nghe + nút "Luyện" AND nhóm theo âm hiển thị riêng (không gắn dấu ✗)

**AC-06: Tab Ngữ pháp hiển thị câu-theo-câu**
- **Given:** User ở tab Ngữ pháp
- **When:** Panel Ngữ pháp render
- **Then:** Mỗi câu trả lời hiển thị dạng sentence-card với nhãn đơn/phức (client heuristic) + có lỗi/không lỗi

**AC-07: Part đang chấm hiển thị skeleton và tự cập nhật**
- **Given:** 1 Part còn đang chấm (chưa xong finalize)
- **When:** User xem report ở Part đó
- **Then:** Panel hiển thị skeleton AND tự động cập nhật thành dữ liệu thật khi chấm xong, không cần tải lại trang

**AC-08: Nhãn UI tiếng Việt, nội dung nói giữ tiếng Anh**
- **Given:** Bất kỳ màn hình nào trong Phản hồi chi tiết
- **When:** Trang render
- **Then:** Toàn bộ nhãn UI là tiếng Việt AND nội dung transcript/câu hỏi giữ nguyên tiếng Anh

**AC-09: Chế độ Part lẻ gỡ hẳn bộ chọn Part khỏi DOM**
- **Given:** Report đang ở chế độ luyện Part lẻ (`visibleParts.length === 1`)
- **When:** Trang Phản hồi chi tiết render
- **Then:** Component bộ chọn Part không được mount (`null`, không chỉ ẩn bằng CSS) AND chỉ hiển thị phản hồi chi tiết của phần đã thi; chế độ full (`visibleParts.length > 1`) vẫn hiển thị đủ bộ chọn Part 1/2/3

**AC-10: Nghe chính xác đoạn phát âm sai không phát cả câu**
- **Given:** User bấm nút nghe trên 1 từ yếu ở tab Phát âm
- **When:** Audio phát
- **Then:** Audio seek đúng tới điểm bắt đầu của từ đó (theo timestamp cấp-từ của Azure) AND tự dừng ngay sau khi kết thúc từ đó (tối thiểu 180ms), không phát tiếp phần còn lại của câu

**AC-11: Mini-drill không ghi ngược vào report gốc**
- **Given:** User bấm "Luyện ›" trên 1 từ yếu, ghi âm lại và nhận điểm luyện tập trong modal
- **When:** User đóng modal và quay lại report
- **Then:** Điểm/`weakWord` gốc của report không thay đổi so với trước khi luyện AND không có lượt chấm nào bị trừ cho hành động luyện tập này

**AC-12: Retry vocab và retry grammar độc lập hoàn toàn**
- **Given:** Trong cùng 1 Part, tab Từ vựng đang ở trạng thái lỗi, tab Ngữ pháp đang ở trạng thái bình thường (hoặc ngược lại)
- **When:** User bấm "Thử lại" trên tab đang lỗi
- **Then:** Chỉ tab vừa bấm retry chạy lại finalize AND tab còn lại của cùng Part không bị kích hoạt lại, không đổi trạng thái

---

## Non-functional Requirements

**Performance:** progressive reveal (mở sớm + skeleton per-field cho vocab/grammar); đổi tab/Part phản hồi tức thì (không gọi lại API); nghe chính xác đoạn phát âm phản hồi ngay khi bấm (audio đã tải sẵn theo segment).

**Khả năng phục hồi:** retry theo Part/sub-step (vocab/grammar) khi lỗi, độc lập giữa 2 sub-step trong cùng Part; mini-drill lỗi không ảnh hưởng dữ liệu report gốc.

**Accessibility:** nhãn UI tiếng Việt rõ ràng cho mọi trạng thái (skeleton/lỗi/rỗng); nút nghe/luyện có nhãn text đi kèm icon, không chỉ dựa vào icon.

---

## User Types

| User Type | Definition | Feature Behavior |
|---|---|---|
| Thí sinh — report Full | Đang xem report của 1 bài Full mock test | Thấy đủ bộ chọn Part 1/2/3; 4 tab tiêu chí hoạt động đầy đủ cho từng Part |
| Thí sinh — report Part lẻ | Đang xem report của 1 bài luyện Part riêng lẻ | Không thấy bộ chọn Part (gỡ hẳn khỏi DOM); chỉ xem phản hồi chi tiết của phần đã thi |
| Thí sinh — đang chờ vocab/grammar finalize | 1 Part vừa có band nhưng Từ vựng/Ngữ pháp chưa phân tích xong | Thấy skeleton riêng cho tab đang chờ; tab Trôi chảy/Phát âm (không phụ thuộc finalize riêng) hiển thị bình thường |
| Thí sinh — vocab hoặc grammar lỗi | 1 Part có tab Từ vựng hoặc Ngữ pháp finalize thất bại | Thấy thông báo lỗi + nút "Thử lại" riêng cho đúng tab đó; tab còn lại không bị ảnh hưởng |
| Thí sinh — đang luyện mini-drill | Vừa bấm "Luyện ›" trên 1 từ yếu | Thấy modal ghi âm → điểm luyện tập miễn phí, một lần; không ảnh hưởng report gốc dù luyện nhiều lần |

---

## 8.7 Phần bổ sung

**Giả định (Assumptions):** Pipeline trả đủ dữ liệu per-question (transcript, annotations, profiles) cho từng Part trước khi report này mở được.

**Phụ thuộc (Dependencies):** Pipeline chấm (#06) · Report tổng quan (#07) · audio playback · route mini-drill `/api/app/speaking/mock-exams/drills/pronunciation`.

**📋 Đối chiếu với tài liệu học thuật (đã làm / chưa làm):** Xem [gap.md](../gap.md) cho danh sách chi tiết các hạng mục học thuật **chưa build** — không lặp lại ở đây, chỉ tham chiếu: badge độ sâu Mạch lạc + nhận xét chiều sâu, vị trí filler trong transcript, gắn nhãn thành ngữ/collocation, gợi ý cấu trúc "💡 Nên thử" + câu mẫu nâng cấp, bảng coaching khẩu hình/lưỡi theo phoneme. Các mục này **không phải** câu hỏi mở của tài liệu này — là hạng mục cần mở thêm pipeline/content, được track riêng ở gap.md.

---

## 8.8 Câu hỏi mở — đã chốt (2026-07-07, đối chiếu code)

Toàn bộ 4 câu hỏi mở trước đó (v0.1) đã được research đối chiếu code trả lời dứt điểm — không còn câu hỏi mở nào cho §8 tại thời điểm này:

| Câu hỏi (v0.1) | Đã chốt tại |
|---|---|
| Dữ liệu panel dùng chung minh hoạ hay bind riêng theo Part/câu từ pipeline? | **Bind thật, riêng theo từng Part/câu** — mọi panel đọc trực tiếp từ `evaluation` của Part đang chọn (`questionRuns`, `pronunciationProfile`, `lexicalProfile`, `grammarProfile`), không dùng dữ liệu minh hoạ chung — xem §8.2, §8.3 |
| Nguồn "Mạch lạc" (liên từ + gợi ý), "Nhóm theo âm", "Cấu trúc nâng cao" — model trả hay rule-based? | **Hỗn hợp, đã tách rõ theo từng thành phần:** liên từ đã dùng = AI sinh (LLM review); gợi ý cạnh liên từ = copy tĩnh; "Nhóm theo âm" = tổng hợp phía **server** từ điểm phoneme Azure (không phải LLM, không phải client) — Đổi log #1, BR-08; "Cấu trúc nâng cao" (`complexStructureUsage`) = **AI sinh**, LLM call `runGrammarReview` với schema ép cứng — Đổi log #2 |
| Cơ chế nghe lại audio chính xác từng từ (precise playback) cho lỗi phát âm | **Xác nhận: seek + phát đúng đoạn từ đó theo timestamp cấp-từ của Azure** (`precisePlaybackStartSeconds/EndSeconds`), tự dừng sau khi hết từ (tối thiểu 180ms) — không phát cả câu — Đổi log #3, FR-12 |
| Nút "Luyện ›" dẫn tới đâu (mini-drill phát âm?) | **Xác nhận: modal ghi âm lại → chấm Azure qua route riêng, miễn phí (không trừ lượt), kết quả standalone không ghi ngược vào report gốc** — đây là thiết kế đúng (giữ toàn vẹn điểm đã chốt), không phải thiếu sót — Đổi log #4, FR-13, BR-07 |

**Phát hiện thêm ngoài 4 câu hỏi mở gốc (từ cùng đợt research):** phạm vi retry per-tab (vocab/grammar độc lập hoàn toàn, không theo cả Part — Đổi log #5, FR-14, BR-05) và cách ẩn bộ chọn Part ở chế độ Part lẻ (gỡ hẳn khỏi DOM, không phải CSS — Đổi log #6, FR-11).

Không còn câu hỏi mở nào cho §8 tại thời điểm này. Các hạng mục học thuật chưa build (khác với câu hỏi mở) tiếp tục theo dõi ở [gap.md](../gap.md).
