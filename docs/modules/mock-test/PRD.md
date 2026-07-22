# Mock Test — Tài liệu nghiệp vụ (PRD)

> Đối tượng đọc: AI agent và đầu mối phụ trách module `features/mock-test/`. Mục tiêu: hiểu đầy đủ nghiệp vụ thi thử Mock Exam + AI Simulation, nắm các ràng buộc với module khác (đặc biệt `shared` và `billing`) để phát triển/sửa lỗi an toàn.
>
> Cặp tài liệu: [TECH.md](./TECH.md) (đặc tả kỹ thuật). Bối cảnh module hóa toàn cục: [../../README-modules.md](../../README-modules.md). Context cục bộ cho AI: [../../../features/mock-test/AGENTS.md](../../../features/mock-test/AGENTS.md).

---

## 1. Tổng quan & mục tiêu module

Module `features/mock-test/` chịu trách nhiệm toàn bộ trải nghiệm **thi thử IELTS Speaking trọn bộ 3 Part** trong điều kiện gần phòng thi thật, cộng với chế độ **AI Simulation** (phòng thi ảo có AI Examiner dẫn dắt). Module điều khiển vòng đời một phiên thi từ chuẩn bị → ghi âm từng câu/từng part → finalize chấm điểm → hiển thị báo cáo chi tiết.

Điểm kỹ thuật cốt lõi của nghiệp vụ này là **chấm điểm phân đoạn (Segmented Assessment)**: Part 1 và Part 3 chấm phát âm theo từng câu (segment) rồi tổng hợp cuối part; Part 2 chấm cả part trong một lần (một bài nói dài).

Mục tiêu sản phẩm:
- Tái hiện áp lực phòng thi thật (giới hạn thời gian, tuần tự câu hỏi, không tua lại).
- Trả band tổng nhanh để giảm thời gian chờ (progressive reveal), rồi bổ sung phân tích Từ vựng/Ngữ pháp sau.
- Cho học viên báo cáo chi tiết theo 4 tiêu chí IELTS + CEFR + phân tích phát âm/từ vựng/ngữ pháp.

---

## 2. Người dùng & vai trò

| Vai trò | Mô tả | Tương tác chính |
|---|---|---|
| Học viên (end-user) | Người thi thử để đo band và luyện áp lực phòng thi | Vào phòng thi, ghi âm, xem báo cáo, nạp lượt khi hết |
| Học viên trả phí | Đã có lượt chấm / gói subscription | Thi thử và được chấm điểm AI đầy đủ |
| Học viên hết lượt | Vào được phòng thi nhưng bị chặn ở bước chấm | Thấy paywall, mua gói lẻ (top-up) hoặc subscription |
| Admin / kỹ thuật | Tinh chỉnh prompt, đối soát ledger | Prompt Lab, log `/settings/logs`, cron reconcile |

> Không có vai trò "giám thị người thật" — examiner trong AI Simulation hoàn toàn do AI/TTS mô phỏng.

---

## 3. Phạm vi tính năng chi tiết

### 3.1 Thi thử Mock Exam trọn bộ 3 Part — `MockExamFull`
Trang: [features/mock-test/pages/exams/MockExamFull.tsx](../../../features/mock-test/pages/exams/MockExamFull.tsx) (mount tại route `/mock-exam/full`).

Luồng tổng quát (đặc tả sản phẩm đầy đủ từng bước: [screens/03-chuan-bi-kiem-tra-mic.md](screens/03-chuan-bi-kiem-tra-mic.md) · [screens/04-phong-thi-giam-khao.md](screens/04-phong-thi-giam-khao.md) · [screens/05-edge-case-khi-thi.md](screens/05-edge-case-khi-thi.md) · [screens/06-processing-cham-diem.md](screens/06-processing-cham-diem.md)):
1. **Chuẩn bị**: màn intro + kiểm tra mic ([ExamPreparationScreen](../../../features/mock-test/components/ExamPreparationScreen.tsx), [ExamIntroCard](../../../features/mock-test/components/ExamIntroCard.tsx), hook [useMicTest](../../../features/mock-test/hooks/useMicTest.ts)). Kiểm tra mic dùng **sóng âm phản ứng âm thanh thật** + **thanh progress** chạy khi có tiếng nói; đầy 100% + lặng ngắn → tích "Microphone hoạt động ổn định".
2. **Ghi âm theo Part**: hệ thống đọc đề (TTS), tự động bật ghi âm, giới hạn thời gian, tự chuyển câu tiếp theo ([ExamQuestionStepper](../../../features/mock-test/components/ExamQuestionStepper.tsx), điều phối bởi [useExamRecordingController](../../../features/mock-test/hooks/useExamRecordingController.ts) + [useQuestionTtsAutoRecord](../../../features/mock-test/hooks/useQuestionTtsAutoRecord.ts)).
3. **Chấm theo phân đoạn**:
   - **Part 1 / Part 3 (segmented)**: mỗi câu hỏi được chấm phát âm Azure riêng (`/assessments`). Hết part mới finalize bằng 3 endpoint progressive `/finalizations/{score,vocab,grammar}` → user thấy band tổng + tab Phát âm trước, Từ vựng/Ngữ pháp populate sau.
   - **Part 2 (whole-part)**: một bài nói dài → một lần `/assessments` chạy atomic cả Azure + 4 bước OpenAI.
4. **Finalize toàn bài**: tổng hợp 3 part, lưu phiên qua NineSpeak, điều hướng sang báo cáo (chuyển tiếp qua màn Processing — chấm điểm).

### 3.2 AI Simulation — `AISimulation`
Trang: [features/mock-test/pages/exams/AISimulation.tsx](../../../features/mock-test/pages/exams/AISimulation.tsx) (route `/ai-simulation/examiner`).

Phòng thi ảo mô phỏng vấn đáp trực tiếp với AI Examiner: examiner "nói" câu hỏi (TTS), hiển thị stage động ([AISimulationLiveStage](../../../features/mock-test/components/AISimulationLiveStage.tsx), [AISimulationControls](../../../features/mock-test/components/AISimulationControls.tsx)), transcript trực tiếp ([LiveTranscriptSidebar](../../../features/mock-test/components/LiveTranscriptSidebar.tsx)). Dùng chung cơ chế session/ghi âm/chấm với Mock Exam Full; khác biệt chủ yếu ở lớp UI/diễn xuất examiner. Có panel trợ lý ([ExamAssistantPanel](../../../features/mock-test/components/ExamAssistantPanel.tsx), [MockExamAssistantLazyPanel](../../../features/mock-test/components/MockExamAssistantLazyPanel.tsx)) dựa trên [lib/mock-exam-assistant.ts](../../../features/mock-test/lib/mock-exam-assistant.ts).

### 3.3 Báo cáo kết quả — `MockExamReport`
Trang: [features/mock-test/pages/reports/MockExamReport.tsx](../../../features/mock-test/pages/reports/MockExamReport.tsx) (route `/report/[sessionId]`).

Hiển thị band tổng + 4 tiêu chí + CEFR + **phản hồi chi tiết theo tiêu chí** (xem dưới). Hỗ trợ **trạng thái partial** (band có trước, Từ vựng/Ngữ pháp populate sau) + **nút Thử lại per-tab**. UI dùng "report kit" từ `shared` ([components/reports/*](../../../features/shared/components/reports)). Đặc tả sản phẩm đầy đủ: [screens/07-report-tong-quan.md](screens/07-report-tong-quan.md) (header + accordion) và [screens/08-report-phan-hoi-chi-tiet.md](screens/08-report-phan-hoi-chi-tiet.md) (4 tab tiêu chí + panel).

> **"Tổng quan" v2 — accordion tiêu chí** ([`ReportScoreAccordion`](../../../features/mock-test/components/reports/ReportScoreAccordion.tsx), **Mock-only**; Practice giữ overview cũ). Header điểm (band/9.0 + **CEFR**; Part lẻ → "Band ước lượng · Part X" + chip amber) + accordion 4 tiêu chí (chấm màu + band + chevron) → "Giám khảo đánh giá điều gì" + **3 sub-score** + **descriptor tiếng Việt**. Band = điểm đã lock (trung bình part); sub-score + descriptor do **AI synthesis** (`overallBreakdown`) sinh → chưa có thì **skeleton**. Mockup: `scoreCard()`/`critRow()`.
>
> **Synthesis breakdown** (1 call AI, eager, persist, **KHÔNG ledger**): đủ 4 part lock band → [`useOverallBreakdownSynthesis`](../../../features/mock-test/hooks/useOverallBreakdownSynthesis.ts) gọi `POST .../finalizations/overall-summary` sinh sub-score + descriptor (grounding bằng band + feedbackSummary part). Eager lúc "Đang xử lý" ở `MockExamFull`; fallback synth lại ở report nếu mở từ lịch sử. Persist xuống 9Speak metadata → mở lại không gọi AI. Chi tiết: TECH §5.3.

> **"Phản hồi chi tiết" v2** (bám mockup `feedbackCard`): 1 card → chọn Part (ẩn Part lẻ) → part label → cue card Part 2 → **tabs pill** 4 tiêu chí (Trôi chảy→Từ vựng→Phát âm→Ngữ pháp) → **grid 50/50** (trái card-per-câu, phải panel tiêu chí). Mọi panel **bind data pipeline có sẵn** (không gọi AI). Rendering/contract kỹ thuật: **TECH §8.5**.
> - **Trôi chảy**: tốc độ nói (`speechRateWpm`) + Mạch lạc (`cohesiveDevices` + gợi ý tĩnh) + chips 3 mức ngắt (`resolvePauseTier`) + filler (`fluencyProfile.fillerCounts`).
> - **Từ vựng**: transcript gạch chân màu **CEFR** + panel **CEFR 6 mức A1–C2** (`CefrProfileCard` từ `word.cefrLevel`) + paraphrase.
> - **Ngữ pháp**: độ chính xác **"X/Y câu"** + đơn/phức + cấu trúc nâng cao (`complexStructureUsage`) + lỗi theo loại (`highlights[].subtype`); **sentence-card** đơn/phức + badge có/không lỗi (AC-06).
> - **Phát âm**: tô từ **3 mức** (`pronWordTone`: ≥75/50–75/<50) + **✗ từ <75**; lưới "Lỗi phát âm (nặng→nhẹ)" (×lần + % + `✓ Đúng /referenceIpa/` + `✗ Bạn nói /IPA as-said/` + ngữ cảnh + "Luyện ›") + **Nhóm theo âm** (không ✗). **Google TTS** cho "Phát âm đúng/mẫu" (**Mock-only**); IPA as-said = `buildSpokenIpaFromPhonemes` (NBestPhonemes, ẩn nếu trùng reference).
> - **Card-per-câu** (`MockExamAnswerCard`): "Câu N" + đề + câu trả lời tô theo tiêu chí + **player** (play/pause + seek + clock) + **karaoke** rAF (⚠️ offset = `segment.startSeconds` — không trừ thì chỉ câu 1 đúng). Ngắt = chấm **sau mỗi từ có pause** (không gộp cuối câu). Bật bằng `renderMode="question-cards"` ([`TranscriptViewer`](../../../features/shared/components/reports/MockExamReportSections.tsx), **additive** — Practice không đổi). Part lẻ → ẩn chọn Part (AC-09).
> - **Part 2 — cue card**: box amber phía trên tabs (`activeQuestionPrompts[0]`) — AC-02. **Mini-drill "Luyện ›"**: modal ghi âm + chấm Azure (`.../drills/pronunciation`), **MIỄN PHÍ — KHÔNG ledger**; nút qua prop optional `onPracticeWord` (Practice không truyền → ẩn). Field optional có **empty-state guard**; nhãn đơn/phức = heuristic client (`classifySentenceComplexity`).
>
> 📋 **Đối chiếu đầy đủ với tài liệu học thuật (đã làm / chưa làm + lý do):** [`gap.md`](./gap.md) — các mục cần mở pipeline/content (badge Mạch lạc, filler vị trí, idiom, "💡 Nên thử", câu mẫu cấu trúc, khẩu hình phoneme) được liệt kê ở đó.

### 3.4 Nạp lượt / Paywall trong phòng thi
Khi hết lượt chấm: hiển thị màn paywall ([MockExamQuotaPaywallScreen](../../../features/mock-test/components/exams/MockExamQuotaPaywallScreen.tsx)) và trạng thái mua ([MockExamPurchaseStatusScreen](../../../features/mock-test/components/exams/MockExamPurchaseStatusScreen.tsx)), điều phối bởi hook [useMockExamQuotaPurchaseFlow](../../../features/mock-test/hooks/useMockExamQuotaPurchaseFlow.ts). Toàn bộ năng lực thanh toán lấy qua **barrel** `@/features/billing`.

> **Nền paywall.** Cả 2 màn đặt nội dung trước nền report blur dùng chung `PurchaseReportBlurSurface` (billing barrel) — cảm giác "có kết quả nhưng bị khóa chấm" (**mockup tĩnh**, không phải report thật). Màn chọn gói phủ scrim tối mỏng (`bg-slate-950/55`) thay vì panel đục (giữ nền blur lờ mờ). Màn trạng thái mua chỉ có card gói + retry từng part (bỏ heading + nút "Quay lại chọn gói").

### 3.5 Chọn đề & Gợi ý — màn vào của Mock Test
Trang: [features/mock-test/pages/exams/MockExamSelect.tsx](../../../features/mock-test/pages/exams/MockExamSelect.tsx) (route **`/mock-exam/select`**, full-screen shell `(focus-mock)` — không sidebar). Đây là entry mới: từ Dashboard/sidebar "Thi thử" → `/mock-exam/select` → chọn đề → `/mock-exam/full`.

- **Thiết kế đích (v2.0 — chưa triển khai, xem callout "Trạng thái triển khai" bên dưới):** đặc tả đầy đủ ở **[`screens/01-chon-de-goi-y.md`](screens/01-chon-de-goi-y.md)** (FR-01→15, BR-01→13, AC-01→17) — xem thêm [chỉ mục toàn bộ 9 màn hình](screens/README.md). Layout gồm 6 khối: (1) **Thẻ trạng thái** — đúng 1 thẻ theo thang ưu tiên mới BR-01: bài đang làm dở → attempts=0 "Kiểm tra trình độ" → attempts≥1 "Sẵn sàng luyện tiếp" (đã **bỏ** 2 state chặn cũ "chưa đặt mục tiêu" và "hết lượt — nâng cấp"); (2) banner Premium chung (free user); (3) **"Đề gợi ý cho bạn"** — lưới 3 thẻ random trong đề chưa làm, trộn mọi scope, **luôn hiển thị**, độc lập hoàn toàn với thẻ trạng thái; (4) tab bar 5 tab cố định (Thi đầy đủ/Part 1/Part 2/Part 3/Đề thi custom); (5) filter bar (Tất cả/Forecast/Đã làm/Chưa làm, exclusive, + dropdown chọn kỳ); (6) lưới "Tất cả đề" lọc theo tab+filter. Card đề theo **content model** mới: icon loại đề (badge chữ "F"/"1"/"2"/"3", 1 màu trung tính) + tag Hot/New/Forecast/Premium (không giới hạn số tag hiển thị đồng thời) + title + trạng thái làm bài + CTA "Bắt đầu" (kể cả đề Premium → popup paywall khi free user bấm). Free user **hết lượt chấm không còn bị chặn** vào thi ở màn này (chỉ chặn khi vào Report); targetBand không còn là điều kiện chặn, chuyển sang capture ở Report sau bài Full đầu tiên (addendum §7 trong PRD Chọn đề V2).
- **Trạng thái từng đề (đã làm / đang làm dở / chưa làm)** — **lưu LOCAL** (`localStorage`, chưa đụng BE): badge ở lưới + thẻ trạng thái "Tiếp tục bài dở" lấy từ [`mock-exam-progress-store`](../../../features/mock-test/lib/mock-exam-progress-store.ts) qua hook [`useMockExamProgress`](../../../features/mock-test/hooks/useMockExamProgress.ts) (reactive bằng `useSyncExternalStore`). Đánh dấu **"đang làm dở"** khi bắt đầu/tiếp tục (`goToExam`); **"đã làm"** + band khi `MockExamFull` chuyển `completed` (band cập nhật khi điểm tổng có). Key theo `catalogId` (= `item.id`: examId full / `examId::part` Part lẻ) → khớp `perExamStatus[item.id]`. `useMockExamSelectSignals` nay đọc store này thay cho stub.
- **Lưới đề**: nguồn `data/mocktest-exam-v1.json` (v2, 81 đề full, curate từ part1/2/3.json — Part 1 = 5 câu, Part 2 = 1 cue card, Part 3 = 3 câu), cờ `access` (free/premium) quyết định khoá. Scope Part 1/2/3 dẫn xuất từ đề full. Mỗi đề có **lời dẫn riêng** (`leadIns`: theo chủ đề Part 1 / cue card / chủ đề Part 2 — không đề nào giống đề nào); lời bất biến ở `commonScripts`. Đọc bằng `TextToSpeechService` (Google TTS, root `lib/`) — xem TECH §8.4.

> **Trạng thái triển khai:** Code hiện tại (`MockExamSelectScreen`) vẫn chạy theo thiết kế **CŨ** — Hero **1 thẻ duy nhất** đổi nội dung theo scope đang chọn (không tách khối "Đề gợi ý cho bạn" độc lập, chưa có tab bar/filter bar/card tag model) — **chưa** áp dụng layout v2.0 mô tả ở trên. Mục trên là spec đích cho đợt phát triển tiếp theo, không phải mô tả hiện trạng đã build.
>
> Luồng **full** (Thi đầy đủ → `/mock-exam/full`, 3 part) đã xong ở tầng phòng thi, không phụ thuộc việc redesign màn chọn đề: chuẩn bị (intro + mic) + **giám khảo riêng theo đề** (avatar + tên) đọc **đúng câu hỏi của đề** (`presetDefinition` từ `questionIds`, [mock-exam-definition-from-catalog.ts](../../../features/mock-test/server/mock-exam-definition-from-catalog.ts)) + **lời dẫn/ack/chuyển/kết** qua Google TTS đúng nhịp (narration chỉ bật khi vào từ màn select; ghi âm vẫn thủ công, không đụng chấm). **Scope Part 1/2/3 chưa wiring phòng thi riêng** (CTA Part vẫn vào full) — mốc tiếp theo.
>
> **Xung đột cần dev xử lý khi triển khai v2.0:** BR-04 (v2.0) bỏ targetBand làm điều kiện chặn ở màn Chọn đề — nhưng [§4 bước 2](#4-user-journey-từng-bước-mock-exam-full) hiện vẫn gọi `useRequireTargetBand` (shared) chặn thật trước khi vào phòng thi. Cần gỡ/nới gate này khi triển khai layout mới; targetBand capture chuyển sang banner ở Report sau bài Full đầu tiên.
>
> **Narration/phụ đề · chúc mừng · công cụ test:** toggle "Phụ đề" persist; cue card Part 2 tách toggle (đọc xong mới hiện); đếm ngược prep2 gate `!isNarrating`. Xong bài → đọc `ending` → pháo hoa "Chúc mừng" (~3.6s) → "Đang xử lý" (chấm chạy nền). Admin (**VẪN trừ lượt**): "Tải audio" (file bất kỳ → tự chuẩn hoá WAV 16kHz) + "Qua câu" (audio mẫu đề `forecast-full-337`). Chi tiết: **TECH §8.4**.

---

## 4. User journey từng bước (Mock Exam Full)

1. Học viên mở **`/mock-exam/select`** → Hero gợi ý 1 đề + lưới đề; bấm "Bắt đầu" (đề full) → `/mock-exam/full` → màn intro + mô tả cấu trúc 3 part.
2. Kiểm tra mic (**thanh progress chạy khi có tiếng nói, đầy → tích "mic ổn định"**); xác nhận target band (qua [useRequireTargetBand](../../../features/shared/hooks/useRequireTargetBand.tsx) của shared).
3. Bắt đầu Part 1: AI đọc câu hỏi → tự bật ghi âm → đếm giờ → tự dừng → chuyển câu kế. Mỗi câu Part 1 gửi `/assessments` (Azure) ngầm.
4. Hết Part 1 → client tự chạy `/finalizations/score` (band + phát âm), sau đó `/finalizations/vocab` + `/finalizations/grammar` song song.
5. Part 2: chuẩn bị (note giấy nháp — [StickyNote](../../../features/mock-test/components/StickyNote.tsx)) → ghi âm một bài dài → `/assessments` atomic chấm cả part.
6. Part 3: như Part 1 (segmented).
7. Khi đủ dữ liệu thô → trừ 1 lượt chấm (ledger), lưu phiên qua NineSpeak.
8. Điều hướng sang `/report/[sessionId]` → xem band tổng ngay, các tab phân tích bổ sung dần.
9. Nếu hết lượt ở bước chấm → paywall → mua gói → tiếp tục.

---

## 5. Business rules

### 5.1 Cơ chế chấm điểm (pipeline 4 bước)
Chi tiết đầy đủ: [../../scoring-pipeline-overview.md](../../scoring-pipeline-overview.md). Tóm tắt ràng buộc nghiệp vụ:
- **Band lock**: chỉ bước Quick Score quyết định band; Vocab Review và Grammar Review nhận band đã lock và **không được rescore**.
- **Pronunciation calibration (Phase D, fail-fast)**: tín hiệu Azure (accuracy/fluency/prosody/completeness/pronunciation + weak words) được một LLM call (`evaluation.pronunciationCalibration`) chuyển thành band IELTS 0–9 **trước** Quick Score. Nếu call fail → propagate lỗi → client banner + retry (không còn fallback công thức Azure cứng).
- **Progressive reveal (Part 1/3)**: `/score` chạy calibration + quickScore → trả band tổng nhanh; `/vocab` + `/grammar` chạy sau, song song, fail độc lập per-tab.
- **Part 2 atomic**: 4 bước trong 1 request, fail bất kỳ bước = hỏng cả request part.

### 5.2 Phần 1/3: chấm theo segment — Part 2: chấm cả part
| | Part 1 | Part 2 | Part 3 |
|---|---|---|---|
| Đơn vị chấm phát âm | Từng câu (segment) | Cả part (1 bài dài) | Từng câu (segment) |
| Số lần `/assessments` | Nhiều (mỗi câu) | 1 | Nhiều (mỗi câu) |
| Finalize | 3 endpoint progressive | atomic trong `/assessments` | 3 endpoint progressive |

### 5.3 Quota / lượt chấm (ledger)
- Ledger riêng của module tại Firestore collection **`mocktest_scoring_charges`** (tách biệt với Practice `practice_scoring_charges`).
- State machine: `pending` → `charging` → `charged` | `charge-failed`.
- **Thời điểm trừ**: trừ 1 lượt khi toàn bộ các part hiển thị (visibleParts) đã có điểm thô (`scoreDone`). Các bước Vocab/Grammar chạy sau **không** trừ thêm lượt.
- Idempotent + race-safe theo doc ID phiên thi (chi tiết doc ID xem [TECH.md](./TECH.md#7-ledger-mocktest-charge-ledger)).

### 5.4 Paywall khi hết lượt
- Khi NineSpeak báo hết lượt ở bước trừ → module hiển thị paywall thay vì lỗi cứng.
- Mua gói lẻ (top-up) hoặc subscription đều qua **barrel** `@/features/billing` (`usePaymentOrderFlow`, `usePremiumPackages`, `PricingPlanCards`, `SubscriptionPaymentModals`, `PurchaseStatusBackdrop`). Module **không** import path nội bộ của Billing.

### 5.5 Flow mode Part 1/3 (feature flag)
- Env `MOCK_EXAM_PART13_FLOW_MODE` (server) + `NEXT_PUBLIC_MOCK_EXAM_PART13_FLOW_MODE` (client) chọn chế độ chấm Part 1/3, đọc qua `feature-flags` của shared. Mặc định: `segmented-openai-azure-speech`. Xem [../../env-and-modules.md](../../env-and-modules.md) §3.3.

---

## 6. Trạng thái & vòng đời phiên thi

Phiên thi (`MockExamSession`) được tạo và quản lý ở client bởi [MockExamSessionProvider](../../../features/mock-test/providers/MockExamSessionProvider.tsx) trên nền state machine ở [lib/mock-exam-session-state.ts](../../../features/mock-test/lib/mock-exam-session-state.ts).

Vòng đời per-question và per-part:
- **Question**: `pending` → `processing` → `evaluated` | `failed`.
- **Part**: `pending` → `processing`/`finalizing` → có `evaluation` (với `partVocabReviewStatus` / `partGrammarReviewStatus` = `pending`/`completed`/`failed`).
- **Persistence**: phiên được lưu/đồng bộ qua NineSpeak ([useExamSessionSync](../../../features/mock-test/hooks/useExamSessionSync.ts), [usePersistExamSession](../../../features/mock-test/hooks/usePersistExamSession.ts), [lib/mock-exam-persistence.ts](../../../features/mock-test/lib/mock-exam-persistence.ts)). Báo cáo đọc lại phiên qua [useMockExamReportSession](../../../features/mock-test/hooks/useMockExamReportSession.ts).
- **Catalog đề thi**: tải qua `/api/app/speaking/mock-exams/catalog` (handler nằm ở onboarding-dashboard BFF, xem TECH.md).

---

## 7. Ràng buộc / phụ thuộc nghiệp vụ với module khác

| Phụ thuộc | Loại | Dùng để làm gì |
|---|---|---|
| `shared` (lib/server/types/hooks/components/data/providers) | import sâu, READ-ONLY | Pipeline chấm điểm, Azure/OpenAI/Deepgram SDK, content store + đề thi, recorder/audio hooks, report kit, band/CEFR util, contract types, NineSpeak client, analytics |
| `billing` | **chỉ qua barrel** `@/features/billing` | Paywall, mua gói lẻ/subscription |
| `onboarding-dashboard` | gián tiếp (host) | Cung cấp handler catalog đề thi qua BFF; app/ là host lắp page |
| `practice` | **CẤM** import lẫn nhau (ESLint enforce) | — |

Nguyên tắc: tài nguyên dùng chung với Practice (recorder, report kit, content store, contract type) đã được đưa về `shared`; prompt/evaluator chấm điểm thì tách riêng từng module để cải tiến độc lập.

---

## 8. Edge cases

- **Mất mạng / fail khi chấm**: mỗi bước có status + retry riêng. `/score` fail → markPartFailed → retry cả part. `/vocab` hoặc `/grammar` fail → banner đỏ + nút Thử lại per-tab, không ảnh hưởng band tổng.
- **Calibration fail**: fail-fast, không fallback — user thấy banner + retry.
- **Spam retry / race**: sequence counter ở client drop response stale.
- **Rời tab giữa lúc chấm**: beforeunload guard ([useMockExamBeforeUnloadGuard](../../../features/mock-test/hooks/useMockExamBeforeUnloadGuard.ts)) chặn reload/đóng tab khi có part đang `processing` hoặc vocab/grammar `pending`.
- **Hủy/bỏ giữa chừng**: [FocusShellActionsProvider](../../../features/mock-test/providers/FocusShellActionsProvider.tsx) phân biệt "Hủy bài thi (mất dữ liệu)" với "Thoát" (giữ session để resume).
- **Câu lỗi**: [MockExamFailedQuestionGuardDialog](../../../features/mock-test/components/MockExamFailedQuestionGuardDialog.tsx) cảnh báo khi còn câu chưa chấm xong.
- **Upload audio fail**: kết quả chấm vẫn lưu; bản ghi âm có thể thiếu (toast cảnh báo).
- **Ledger kẹt** (`charge-failed` / crash giữa phase): admin reconcile qua cron (xem TECH.md).
- **History cũ**: payload báo cáo trước khi có per-tab status rơi về hành vi cũ (fallback chung).

---

## 9. Out of scope

- Logic thanh toán/giá gói (thuộc `billing`; module chỉ tiêu thụ qua barrel).
- Luyện tập từng câu Part 1/2/3 (thuộc `practice`; cấm import chéo).
- Shell/sidebar/onboarding/history/settings tổng (thuộc `onboarding-dashboard`).
- Hạ tầng chấm điểm lõi (Azure/OpenAI/Deepgram, evaluation-pipeline, question-store) — nằm ở `shared`, module **không sửa**, chỉ tiêu thụ.
- Env/secret service lõi (đọc tập trung ở `shared`/`lib`, module không tự `process.env`).
- Trang `debug/*` (preview nội bộ, không phải luồng user thật).
