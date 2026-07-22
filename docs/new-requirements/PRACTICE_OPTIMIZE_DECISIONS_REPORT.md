# Practice Optimize — Báo cáo quyết định (Claude tự chốt) & phát hiện khi triển khai

> Ghi lại các quyết định Claude tự chốt + phát hiện kỹ thuật quan trọng trong lúc triển khai, để bạn duyệt/override. Cập nhật dần theo từng phase. Tham chiếu kế hoạch: [PRACTICE_OPTIMIZE_IMPLEMENTATION_PLAN.md](./PRACTICE_OPTIMIZE_IMPLEMENTATION_PLAN.md).

---

# ⭐ TỔNG KẾT & RÀ SOÁT ĐỐI CHIẾU TÀI LIỆU (2026-06-23)

> Nhánh `feat/practice-p0-results-ui` (stack trên `feat/practice-shared-foundation`), **chưa merge `dev`**. Build xanh toàn bộ. **Mock Test: 0 file đụng; scoring dùng chung: 0 diff vs `dev`.** Nguồn đối chiếu: `PRACTICE_OPTIMIZE_FULL_SPEC.md`, `PRACTICE_ACCEPTANCE_CRITERIA.md`, `9Speak - PRD-2.docx`, gap analysis A–K.

## A. Tóm tắt tính năng & luồng mới (theo khu vực)

**1. Khu kết quả chấm điểm (§2, P0)** — Bố cục mới: điểm tổng + "AI Chấm điểm" → **4 thẻ tiêu chí là nút bấm** → mở phân tích ở cột AI hỗ trợ → "Câu trả lời đã sửa lỗi" → 3 nút cải thiện. **Bỏ 3 tab cũ + nút/modal "Xem chi tiết"** (cả Part 1/2/3). Lượt cũ tự thu gọn.

**2. "Câu trả lời đã sửa lỗi" — track-changes (§3, P0)** — Transcript với lỗi từ vựng/ngữ pháp **gạch đỏ + bản sửa xanh inline**, bấm → popup giải thích (loại lỗi + lý do). Skeleton khi đang phân tích. **Luồng:** corrections (vocab+grammar) → `buildCorrectionPreviewEvaluation` map `wordIds` → `PracticeCorrectedAnswer` render + tái dùng `AnnotationPopup`. Lỗi "thiếu từ" được neo vào cụm tồn tại của `correctedText` (L3).

**3. So sánh qua các lượt (§5, P1)** — Chip delta band tổng (▲/▼/–) + delta 4 thẻ + dòng động viên theo xu hướng + **mini-trend** band tổng (sticky trên khu chat) + **auto-collapse** lượt cũ. Tính so lượt **đã chấm** liền trước (loại pending).

**4. Phân tích 4 tiêu chí ở cột AI (§6, P1)** — Bấm thẻ điểm → card phân tích trong rail (cộng dồn, đóng/mở lại) + nút **"⛶ Bản đầy đủ"** mở panel rộng 2 cột. Nội dung: Phát âm (phân bố Tốt/Khá/Yếu + top từ + âm yếu + transcript + 🔊 nghe âm chuẩn), Trôi chảy (gauge tốc độ + ngắt nghỉ 3 mức + liên từ), Từ vựng (phân bố CEFR + paraphrase + transcript tô màu CEFR), Ngữ pháp (độ chính xác + đơn/phức + cấu trúc nâng cao + lỗi theo loại). **Luồng:** panel `onSelectCriterion` → timeline `onOpenCriterion(attempt,key)` → page state `criterionAction` → rail render `PracticeCriterionCard`.

**5. Coach cải thiện câu (§7, P1)** — **3 nút mục tiêu** (🔵 Nâng từ vựng · 🟠 Đa dạng ngữ pháp · 🟢 Phát triển ý) → câu nâng cấp với **chỗ thay đổi tô màu** (hover = vì sao + kỹ thuật) + "Bạn đã nâng cấp gì" + "Cụm đáng học" + **dịch nghĩa** + **🔊 nghe mẫu** + **🎙 Thử nói lại theo gợi ý**. Bỏ số band cụ thể.

**6. Edge case khi nói (§8)** — Card chuyên biệt: **A1 mic bị chặn / A2 không có mic / B1 ghi quá ngắn** (phân loại từ `DOMException.name` + ngưỡng 3s).

**7. Hạ tầng** — **Timeout client** (§14.5, AbortController per-endpoint, tránh treo vô hạn) · **Analytics** mở phân tích/bản đầy đủ · **Compact UI** (§9).

## B. Quyết định kỹ thuật chính

- **Prompt:** Chỉ sửa **`practice.answerRewrite`** (prompt RIÊNG Practice, Mock Test không dùng): thêm `changes[]` (diff: text/replacement/category/reason), `phrasesToLearn[]`, `translation`; route 3 mục tiêu vocab/grammar/depth. **`evaluation.grammarReview` (dùng chung Mock Test) KHÔNG đổi** — đã revert (xem mục "Đính chính scope").
- **Cách gọi:** `fetchWithTimeout` (AbortController) bọc 4 endpoint Practice (assessments 45s, vocab/grammar 60s, rewrite 30s, word-cefr 30s). Rewrite gọi **đúng mục tiêu user bấm** (1 call/mục tiêu, on-demand). word-cefr giữ fire-and-forget.
- **Enum rewrite:** `PracticeRewriteMode += vocab|grammar|depth` (giữ improve/expand cho history); `rewriteStatus`→`Partial<Record>` + **computed-key `[mode]`** (bỏ ternary improve/expand → mode-agnostic).
- **Kiến trúc cross-column:** card phân tích mở chéo panel(trái)→rail(phải) qua state `criterionAction` ở page; rail dùng `RailBlock` kind `"criterion"`.
- **Không đụng scoring chung:** §6.2 pause 3 mức tính **từ `pauseDurationMs` có sẵn** (không thêm field shared); §6.4 (đơn/phức, câu không lỗi) + C4 (liên từ) bằng **heuristic client-side** (`practice-text-heuristics.ts`); cấu trúc nâng cao lấy từ `grammarComplexStructureUsage` (LLM data sẵn có). L3 neo lỗi thiếu từ ở **tầng build annotation của Practice**.
- **TTS:** "nghe mẫu/âm chuẩn" dùng Web Speech API trình duyệt (`useSyncExternalStore` SSR-safe), feature-detect ẩn nếu không hỗ trợ.

## C. Bảng rà soát đối chiếu spec

| Mục spec | Trạng thái | Ghi chú |
|---|---|---|
| §2 Bố cục mới + bỏ tab/xem chi tiết (P0) | ✅ Đủ | Cả 3 Part |
| §3 Track-changes + popup + skeleton (P0) | ✅ Đủ | +L3 lỗi thiếu từ |
| §5 Delta + động viên + mini-trend + auto-collapse (P1) | ✅ Đủ | |
| §6.1 Phát âm | ✅ Đủ* | Phân bố + top từ + âm yếu + transcript ✅. **2 nút nghe đối chiếu** (Mẫu TTS + Bạn = cắt đoạn audio user, ở bản đầy đủ) ✅. **Nhóm lỗi theo âm** (gom theo worst-phoneme) ✅. *"trọng âm/ngữ điệu" bỏ — Azure không cấp (gap B2) |
| §6.2 Trôi chảy | ✅ Đủ | gauge + pause 3 mức + liên từ |
| §6.3 Từ vựng | ✅ Đủ | CEFR + paraphrase |
| §6.4 Ngữ pháp | ✅ Đủ | accuracy + đơn/phức (heuristic) + cấu trúc nâng cao + lỗi theo loại ✅. **Vạch màu từng câu inline** (so khớp đoạn lỗi `corrections.grammar.text` → câu ⚠️/✅) ✅ |
| §7 Coach (3 mục tiêu + diff + cụm + nghe mẫu + thử nói lại) (P1) | ✅ Đủ* | *Lưu flashcard = toast "đang phát triển" (chờ backend) |
| §8 Edge: A1/A2/B1 (P0) | ✅ Đủ | |
| §8 Edge: **B2 im lặng** (P1) | ✅ Đủ | transcript rỗng → server code `no_speech` (422) → edge card |
| §8 Edge: **B4 quá dài** (P1) | ✅ Đủ | auto-stop theo ngưỡng/Part + toast + vẫn chấm (countdown UI = follow-up nhỏ) |
| §8 Edge: **C1 lệch đề** (P1) | ✅ Cảnh báo chung | heuristic độ trùng từ → banner "có vẻ lệch đề", vẫn hiện kết quả (không nêu tên chủ đề — đúng gap C1) |
| §8 Edge: B3 nhiễu / B5 sai ngôn ngữ (P1) | 🚫 BỎ | **Chốt bỏ** (user 2026-06-23): không có tín hiệu detect tin cậy (mức nhiễu audio / language-detect) ở tầng Practice. Không làm. |
| §9 Compact (P1) | ✅ Pass bảo thủ | Tinh chỉnh thêm khi xem app |
| §10 Khác biệt Part 1/2/3 | ✅ Đủ | Part 2 cue card giữ nguyên |
| §13 Analytics | ✅ Đủ | open analysis/full + event cũ ✅. Thêm: `play_pronunciation_audio` (reference/self), `start_word_drill`, `save_vocab_phrase`, `practice_result_delivered` (band-change + time-to-result) ✅ |
| §14 Performance | ✅ Đủ | §14.5 timeout ✅; progressive reveal ✅; **§14.6**: `practice_result_delivered.time_to_result_ms` đã gắn (dashboard tính p50/p95) ✅ |
| §6.1 Drill luyện âm từng từ (Đợt 3/P2) | ✅ Đủ | Nút "Luyện" mỗi top-weak-word → popup ghi âm 1 từ → Azure chấm → điểm + phoneme MẪU/BẠN tô màu. Không tốn lượt. |

## D. Còn lại — phân loại theo lý do chưa làm

- **Chờ backend:** Lưu flashcard (E2) — hiện toast "đang phát triển".
- **Cần verify LLM staging:** prompt `practice.answerRewrite` (3 mục tiêu + diff): model trả `replacement` verbatim để tô màu khớp + route đúng mục tiêu + không cụt JSON.
- **Cần tín hiệu pipeline / quyết định sản phẩm (Phase 6):** edge B2/B3/B5/C1 (im lặng/nhiễu/sai ngôn ngữ/lệch đề) + B4 max-duration; ngưỡng phát hiện (gap F).
- **Đợt 3/P2:** drill luyện âm từng từ (§6.1) ✅ · per-segment audio (tái dùng `playAudioSegment` cho nút "Bạn") ✅.
- **Tinh chỉnh khi xem app (đã làm 2026-06-23):** §6.1 nhóm theo âm + 2 nút nghe ✅ · §6.4 vạch màu từng câu ✅ · §13/§14.6 analytics ✅. Còn lại: Compact §9 (pass bảo thủ, tinh chỉnh mật độ khi xem app thật).

**Kết luận rà soát (cập nhật 2026-06-23 — sau tinh chỉnh §6.1/§6.4/§13/§14.6):** **Đợt 1 (P0) — ĐỦ.** **Đợt 2 (P1) — ĐỦ** ở các hạng mục khả thi: §8 có **A1/A2/B1/B2/B4/C1** (B3/B5 **đã chốt BỎ** — không detect được); §6.1/§6.4 nay **ĐỦ** (nhóm theo âm + 2 nút nghe + vạch màu từng câu); §13/§14.6 analytics **ĐÃ GẮN** (play audio/drill/save-phrase + time-to-result + band-change). **Đợt 3 (P2) — drill §6.1 ĐÃ LÀM.** Còn lại: phần đụng LLM (prompt `practice.answerRewrite`) cần verify staging; **flashcard chờ API backend** (nút hiện toast + đã ghi nhận event `save_vocab_phrase`); §9 compact tinh chỉnh khi xem app thật.

## E. Quyết định ngưỡng & cách làm edge case (2026-06-23, Claude tự chốt)

> Làm thêm §8 edge case **Practice-only, không đụng scoring chung** (mock-test = 0 file, 3 file scoring = 0 diff). Tự chốt ngưỡng:

- **B4 — giới hạn thời lượng (auto-stop):** ngưỡng **Part 1 = 90s · Part 2 = 120s · Part 3 = 90s** (theo gợi ý §11, **tunable** qua prop `maxRecordingSeconds`). Cách làm: `useSpeakingPractice` theo dõi `recordingDurationSeconds`; chạm ngưỡng → tự gọi `stopRecording` (vẫn chấm bình thường) + **toast** "Đã tự dừng ở giới hạn Xs". (Đếm ngược trực quan = follow-up nhỏ; không đụng recorder shared.)
- **B2 — im lặng/không có giọng:** tín hiệu = **transcript rỗng** từ Azure → `evaluate-practice` throw → route `/assessments` trả **HTTP 422 + code `no_speech`** → client `isPracticeNoSpeechError` → hook set `edgeCase: "no_speech"` → card "Không nghe thấy giọng nói". **Không ra điểm → ledger không trừ** (đúng H-2).
- **C1 — lệch đề:** heuristic `isLikelyOffTopic` (client) = độ trùng **từ nội dung** (bỏ stopword) giữa transcript và `question.text`. Gắn cờ khi: câu hỏi ≥3 từ nội dung **và** transcript ≥15 từ **và** overlap **< 12%**. Hành vi: **banner cảnh báo chung** "có vẻ lệch đề" trên kết quả (KHÔNG nêu tên chủ đề, KHÔNG chặn — vẫn hiện điểm/transcript để tránh phạt oan câu sáng tạo; đúng đề xuất gap C1 đợt đầu).
- **B3 nhiễu / B5 sai ngôn ngữ — BỎ (user chốt 2026-06-23):** không có tín hiệu detect tin cậy (mức nhiễu audio / language-detection) ở tầng Practice; không làm.
- **Cách gọi/cấu trúc:** tất cả ở tầng Practice (`evaluate-practice`/`practice-route-handlers`/`useSpeakingPractice`/`PracticeEdgeCaseCard`/`practice-text-heuristics`). Không thêm field/scope vào scoring dùng chung.

## F. Drill luyện âm từng từ (§6.1, P2) — 2026-06-23

- **Luồng:** nút "Luyện" mỗi top-weak-word (card Phát âm) → popup `PracticeWordDrill` (đè z-70) → nghe mẫu (TTS) + MẪU phonemes → "Ghi âm ngay" (recorder riêng) → `POST /api/app/speaking/practice/word-drill` (multipart audio) → `assessPracticeWordDrill` gọi **Azure unscripted có sẵn**, lấy từ đầu tiên → trả `{recognizedWord, pronunciationScore, referenceIpa, phonemes}` → BẠN: điểm + phoneme tô màu (xanh ≥75 / vàng ≥50 / đỏ <50).
- **Quyết định:** chỉ **điểm phát âm + phân tích phoneme** (đúng yêu cầu — không đánh giá cụ thể hơn). Dùng Azure unscripted (không referenceText) cho đơn giản; `recognizedWord` là từ Azure nhận ra (ước lượng). **KHÔNG ledger/quota** (drill miễn phí — H-1), **KHÔNG LLM**. Endpoint + handler thuần Practice; gọi `evaluateSpeechAssessment` (shared) **read-only** — không sửa shared, mock-test không ảnh hưởng.

---

## Phase 0 — Nền tảng shared (nhánh `feat/practice-shared-foundation`, 2026-06-22)

### Đã làm
- Thêm type `MockExamPauseDurationLevel` + field optional `pauseDurationLevel` trên `MockExamTranscriptWord` (`shared/types/mock-exam.ts`).
- Thêm helper thuần `classifyPauseDurationLevel(ms)` + ngưỡng tunable (`shared/lib/transcript-pause-classifier.ts`).
- Populate field trong Practice (`evaluate-practice.ts` → `buildTranscriptSentences`).
- Build xanh: `tsc --noEmit` = 0, `eslint` (3 file) = 0, `npm run build` = 0.

### Quyết định chốt

**C1-1 — Ngưỡng pause 3 mức (theo PRD #11, áp dụng):**
- `short` (🟢 tốt): 200ms ≤ t ≤ 500ms · `medium` (🟡 vừa): 500ms < t ≤ 1000ms · `long` (🔴 dài): t > 1000ms · `none`: t < 200ms (dưới ngưỡng renderable, không vẽ marker).
- Là 🔢 **tunable** (hằng số `PAUSE_LEVEL_SHORT_MAX_MS=500`, `PAUSE_LEVEL_MEDIUM_MAX_MS=1000`); calibrate lại theo dữ liệu thật sau.

**Quyết định kiến trúc (QUAN TRỌNG — lệch so với plan ban đầu):**
- Plan ban đầu định "thêm mức `medium` vào `PauseClassification` (`none|good|bad`)". **Spike phát hiện điều này sẽ làm hỏng chấm điểm:** `pauseClassification` (good/bad) phân loại theo **ngữ nghĩa vị trí** (good = ranh giới mệnh đề, bad = bí từ) và **feed thẳng vào `goodPauseCount`/`badPauseCount`** — input chấm band của **cả Practice lẫn Mock Test**. PRD #11 lại muốn 3 mức theo **trường độ** — khác trục hoàn toàn.
- → **Đã tách thành 2 trục độc lập:** giữ nguyên `pauseClassification` (scoring) + thêm mới `pauseDurationLevel` (chỉ hiển thị). Kết quả: **không đổi 1 byte output của Mock Test**, không perturb band. Đây là cách additive an toàn nhất.

### Phát hiện cần xử lý ở phase sau

**L3 (XÁC NHẬN THẬT) — Track-changes lỗi "thiếu từ" bị DROP:**
- `buildAnnotationsFromEvaluation` → `findWordMatchesForText` (`evaluation-pipeline.ts:915`) map `wordIds` bằng **so khớp text**. Lỗi kiểu thiếu mạo từ/giới từ (vd `err.text="a"` không tồn tại trong transcript) → `wordIds` rỗng → bị **filter bỏ** ở line ~1021 → annotation **biến mất**, không render được track-changes.
- **Ảnh hưởng:** §3 "Câu trả lời đã sửa lỗi" (P0) sẽ **không hiển thị lỗi chèn (thiếu từ)**.
- **Hướng xử lý (Phase 1/4):** thêm cơ chế **anchor cho lỗi chèn** — hoặc (a) yêu cầu prompt grammar trả `text` kèm 1 từ ngữ cảnh đang tồn tại (vd "to school" → "to the school", `text="to school"` match được), hoặc (b) thêm field anchor (wordId của từ trước/sau điểm chèn) + render marker chèn giữa 2 từ. Quyết định cụ thể khi vào Phase 1.

**L2 — Không có đo token output:** `requestStructuredOpenAiJson` chỉ phát hiện cụt qua `finish_reason==="length"` / `looksLikeTruncatedJson`, **không log `usage.completion_tokens`**. Default `AI_SCORING_MAX_TOKENS=12000`, retry=16000. Khi mở rộng schema grammar/quickScore (Phase 4) cần **thêm log token thực tế** để biết có sát trần không trước khi nhồi thêm field.

**Test runner:** repo **không có** jest/vitest/`.test.ts`. Helper `classifyPauseDurationLevel` thuần/deterministic, verify qua tsc + build thay vì unit test. Nếu muốn có test, cần dựng test harness (đề xuất tách việc riêng, không gộp vào Phase 0).

### Trạng thái merge
- Đụng `shared/` (dùng chung Mock Test) → **GIỮ TRÊN NHÁNH, chưa merge `dev`**. Chờ owner duyệt. Mock Test đã verify không đổi hành vi (field optional, không nơi nào của mock-test populate/đọc `pauseDurationLevel`).
- **Mô hình nhánh:** các phase **xếp chồng** — `feat/practice-p0-results-ui` (Phase 1) nhánh từ `feat/practice-shared-foundation` (Phase 0), để tài liệu (plan + report) tích lũy liền mạch và compose. Vẫn chưa merge `dev`.

---

## Phase 1 — Đợt 1 / P0 (nhánh `feat/practice-p0-results-ui`, stack trên Phase 0, đang làm)

### Slice 1 — Edge case cards A1/A2/B1 ✅ (build xanh)
- Component mới `PracticeEdgeCaseCard.tsx`: mã `mic_denied` (A1) · `no_device` (A2) · `too_short` (B1). A1 có nút "Hướng dẫn cấp quyền" (các bước inline) + "Thử lại"; A2/B1 nút ghi lại.
- `useSpeakingPractice`: phân loại lỗi recorder TẠI NGUỒN qua `DOMException.name` (`classifyRecorderStartError`): NotAllowedError/SecurityError→A1; NotFound/NotReadable/Overconstrained→A2. Too-short (<3s)→B1 kèm số giây thực. Expose `edgeCase`; clear ở reset/startRecording/runEvaluation.
- `PracticeQuestionShell` + 3 page (Part1/2/3): render card thay banner đỏ chung khi có `edgeCase`. **Không đụng `shared/`** (đọc `error.name` ở tầng practice).
- **F-1 (một phần):** ngưỡng "quá ngắn" giữ **3s** (`MIN_RECORDING_DURATION_SECONDS`). Tinh chỉnh theo Part + "thời gian nói thực" gộp Phase 6.
- B2/B3/B5/C1 **chưa làm** (cần tín hiệu thật từ pipeline → Phase 6).

### Slice 2 — §3 "Câu trả lời của bạn — đã sửa lỗi" (track-changes) ✅ (build xanh)
- Component mới `PracticeCorrectedAnswer.tsx`: render transcript với lỗi **từ vựng (critical) + ngữ pháp** gạch bỏ đỏ + bản sửa xanh inline; bấm → popup giải thích (tái dùng `AnnotationPopup` của panel qua `onSelectAnnotation`). Skeleton khi cả vocab+grammar pending; RetryNote khi 1 loại failed; câu chúc khi không lỗi.
- Tái dùng dữ liệu annotation đã có (`buildCorrectionPreviewEvaluation` → wordIds). **Không đụng `shared/`.**
- Chèn **thêm** (additive) phía trên khối "Phân tích bài nói" trong `PracticeEvaluationPanel`.

**Quyết định sequencing (lệch plan, có chủ đích):** §3 chỉ hiển thị lỗi **vocab critical + grammar** (đúng tinh thần "nơi DUY NHẤT hiển thị lỗi"); bỏ qua lexical advanced/repetitive (gợi ý — thuộc card Từ vựng Phase 3) và phát âm (card Phát âm Phase 3).

**Đính chính §2 (bỏ 3 tab + 4 thẻ điểm bấm được) → DỜI sang Phase 3:** thay vì xóa tabs ở Phase 1 rồi dựng lại card phân tích ở Phase 3, gộp 2 việc vào Phase 3 (relocate nội dung tab → card cột AI trong 1 lần) để tránh xóa-rồi-dựng-lại + tránh trạng thái regress. Phase 1 vì vậy **giữ tạm 3 tab** song song với track-changes (overlap tạm trên nhánh, sẽ gỡ ở Phase 3). §2.7 (bỏ bong bóng transcript "Lần X" lặp) làm tiếp ở slice 3.

### L3 (lỗi "thiếu từ") — hiện trạng trong slice 2
- Track-changes hiện **chưa render lỗi chèn/thiếu từ** (annotation rỗng wordIds đã bị bỏ ở tầng build annotation). Sẽ xử lý khi sửa prompt grammar (Phase 4): yêu cầu `text` kèm 1 từ ngữ cảnh tồn tại để map được, hoặc thêm anchor. Ghi nhận, chưa chặn P0 (lỗi thay-thế/sai từ — phần lớn — hiển thị tốt).

### Slice 3 — §2.7 bỏ transcript lặp ở bong bóng "Lần X" ✅ (build xanh)
- `PracticeAttemptTimeline.tsx`: lượt **đã chấm** không còn in lại transcript ở bong bóng (transcript đã có trong khu kết quả "Câu trả lời đã sửa lỗi"). Giữ nhãn "Lần X" + audio "nghe lại"; lượt pending vẫn hiện thông báo trạng thái.

### Tổng kết Phase 1 (P0 — đã làm trong đợt này)
- ✅ §8 edge cards A1/A2/B1 · ✅ §3 track-changes "Câu trả lời đã sửa lỗi" · ✅ §2.7 bỏ transcript lặp.
- ⏭️ **DỜI sang Phase 3** (gộp với relocate phân tích → card cột AI): §2 bỏ 3 tab + biến 4 thẻ điểm thành nút mở phân tích + bỏ nút/modal "Xem chi tiết".
- ⏭️ **Phase 4:** L3 render lỗi "thiếu từ" (sửa prompt grammar).
- Tất cả trên nhánh stack `feat/practice-p0-results-ui` (trên Phase 0), **chưa merge `dev`**.

---

## Phase 2 — Đợt 2 / P1: So sánh lượt (nhánh `feat/practice-p0-results-ui`, stack) ✅ (build xanh)

> Thuần `features/practice/` (timeline + panel), **không đụng `shared/`**. Tiếp tục trên cùng nhánh Phase 1.

- **Delta band tổng (§5):** chip `▲/▼/–` cạnh band lớn, so với **lượt ĐÃ CHẤM liền trước** (pending bị loại khỏi tính delta/trend). Lượt 1 không có chip.
- **Delta 4 thẻ tiêu chí:** mỗi `ScoreToneCard` hiện `▲/▼/–` + giá trị nhỏ.
- **Dòng động viên:** tăng → emerald tích cực; giảm → amber động viên (không gắt); bằng → liệt kê tiêu chí ↑/↓.
- **Mini-trend (`PracticeBandTrend`):** sparkline SVG band tổng qua các lượt + tổng delta; **sticky top** trong khu chat (luôn thấy khi cuộn); hiện khi ≥ 2 lượt đã chấm. Trục y co [3..9] cho dễ thấy chênh lệch.
- **Auto-collapse (§2.7/G):** chỉ lượt **mới nhất (đã chấm)** mở rộng; lượt cũ tự thu gọn qua prop `defaultExpanded` + `useEffect` (đổi khi có lượt mới). Thao tác mở/thu gọn thủ công của user vẫn giữ tới khi `defaultExpanded` đổi tiếp. Lượt thu gọn vẫn hiện điểm + delta + động viên.
- Component mới: `PracticeBandTrend.tsx`. Panel thêm props `scoreDelta` + `defaultExpanded` + export type `PracticeScoreDelta`.
- **Giới hạn (L9):** delta/trend tính trên `attempts` trong phiên (PracticeSessionStore). Xuyên phiên (mở lại sau refresh / từ Lịch sử nhiều lượt) phụ thuộc history rehydrate — chưa xác minh đầy đủ; nếu chỉ có 1 lượt khi mở lại thì không có trend (đúng kỳ vọng).

---

## Phase 3 — Đợt 2 / P1: Phân tích 4 tiêu chí ở cột AI (nhánh `feat/practice-p0-results-ui`, stack)

> Phase lớn nhất → chia nhiều slice. **Không đụng `shared/`** (tái dùng component report của shared).

### Slice 1 — Cross-column: bấm thẻ điểm → card phân tích ở cột AI (Part 1) ✅ (build xanh)
- **Thẻ điểm bấm được:** `ScoreToneCard` thành nút khi panel nhận `onSelectCriterion` (tooltip "Xem phân tích · [label]") + dòng gợi ý. Chỉ bật khi có handler → Part 2/3 (chưa nối) vẫn là thẻ tĩnh.
- **Plumbing chéo cột:** `PracticeEvaluationPanel.onSelectCriterion` → `PracticeAttemptTimeline.onOpenCriterion(attempt, key)` → page `criterionAction {requestId, criterionKey}` → `PracticeAssistantRail.criterionAction`. Rail thêm `RailBlock` kind `"criterion"` (cộng dồn như rewrite), dedupe theo requestId+key, đóng được (X) → bấm lại mở lại.
- **Component mới `PracticeCriterionCard`:** header (label + điểm + đóng) + body: lexical/grammar/pronunciation → tái dùng `TranscriptViewer` + `AnnotationLegend` + `AnnotationPopup` riêng (đúng nội dung tab hiện tại); fluency → tóm tắt wpm + ngắt nghỉ tốt/dài + filler (gauge + transcript 3 mức là slice sau). `buildCorrectionPreviewEvaluation` được export để rail tái dùng.
- **Sequencing:** **giữ tabs** ở cột trái (additive) — chưa gỡ. Gỡ tabs + nút/modal "Xem chi tiết" sẽ làm ở slice cuối Phase 3 khi 4 card đã đủ "expert".

### Slice 2 — Phát âm (§6.1 rail) + Trôi chảy (§6.2 gauge + pause 3 mức) ✅ (build xanh)
- **`PracticePronunciationInsight`:** phân bố Tốt/Khá/Yếu (thanh + %) + **top 3 từ cần sửa** (kèm % + IPA) + **âm yếu nhất** (sound + từ ví dụ + %). Dữ liệu từ `feedbackContext` + `mispronouncedWords` + `pronunciationSoundInsights` (đã có sẵn). Vẫn giữ TranscriptViewer phía dưới.
- **`PracticeFluencyInsight`:** **gauge tốc độ** (60–200, vùng tốt 120–150, kim chỉ wpm — ngưỡng PRD #11) + tóm tắt **ngắt nghỉ 3 mức** (đếm short/medium/long từ `pauseDurationLevel` Phase 0) + filler + **transcript đánh dấu ngắt nghỉ 3 mức** (chấm màu giữa từ). Bài cũ thiếu `pauseDurationLevel` → hiện dòng "chưa có dữ liệu".
- Gắn vào `PracticeCriterionCard` (thay tóm tắt fluency tạm ở slice 1).
- **Còn để bản đầy đủ (slice 4):** transcript phát âm tô màu theo band + 2 nút nghe đối chiếu + nhóm theo âm; phần Mạch lạc (liên từ) của fluency.

### Slice 3 — Từ vựng (§6.3) ✅ (build xanh)
- **`PracticeVocabInsight`:** badge cấp độ (CEFR overall) + **phân bố CEFR A1→C2** (thanh + %, màu xám→đỏ) từ `vocabCefrDistribution` + **card gợi ý paraphrase** (từ cơ bản → 3 lựa chọn nâng cấp kèm CEFR chip + nghĩa/contextNote) từ `corrections.lexical` category advanced/repetitive.
- Transcript tô màu CEFR + highlight lỗi từ vựng do `TranscriptViewer` (lexical) đảm nhiệm bên dưới (đã có `cefrColorClass` theo `word.cefrLevel` từ `/word-cefr`).
- **Ngữ pháp (§6.4):** card hiện vẫn là `TranscriptViewer` (grammar). Phần "đầy đủ" (câu đơn/phức, câu không lỗi, cấu trúc nâng cao ✅/⚠/💡, lỗi theo loại) **chờ Phase 4** mở rộng prompt grammar (đụng `shared/`).

### Slice 4 — Nối Part 2 + gỡ 3 tab (chốt §2 cho Part 1/2) ✅ (build xanh)
> User chốt: ưu tiên nối Part 2 + gỡ tab; gỡ tab "khi card đã đủ" (chỉ ở Part đã nối card, tránh Part 3 mất phân tích).
- **Part 2 nối card phân tích:** thêm state `criterionAction` + `handleOpenCriterion` + reset khi đổi câu; timeline nhận `onOpenCriterion`; rail nhận `criterionAttempts={attempts}` + `criterionAction`. Rewrite của Part 2 **vẫn ở panel trái** (không kéo vào rail) nhờ prop mới `criterionAttempts` tách khỏi `rewriteAttempts` (Part 2 không truyền `rewriteAttempts` → rail không sinh card rewrite).
- **Gỡ 3 tab + nút "Xem chi tiết":** `PracticeEvaluationPanel` ẩn khối "Phân tích bài nói" (3 tab) **và** nút `detailAction` khi có `onSelectCriterion` (tức Part đã nối card = Part 1/2). **Part 3 chưa nối → vẫn giữ tabs** (không mất phân tích). Track-changes "Câu trả lời đã sửa lỗi" + retry vocab/grammar (qua RetryNote) vẫn còn.
- **Hệ quả:** §2 (bỏ tab + 4 thẻ điểm = nút mở phân tích) **HOÀN TẤT cho Part 1 & Part 2**.

### Slice 5 — Card Ngữ pháp §6.4 (từ dữ liệu có sẵn) ✅ (build xanh)
- **`PracticeGrammarInsight`:** độ chính xác (từ `accurateSentencePercent` nếu có, else số lỗi) + **lỗi theo loại** (gom `corrections.grammar` theo subtype → nhãn VN) + **cấu trúc nâng cao** (✅ `successful` / ⚠ `failed` / 💡 "nên thử" = bộ tĩnh theo PRD #11 trừ đi `attempted`) + nhận xét chung (`grammarGeneralFeedback`). **Không đụng shared** — toàn bộ từ field đã có.
- Gắn vào `PracticeCriterionCard` cho `grammaticalRangeAndAccuracy`. → 4/4 card tiêu chí đã có nội dung "expert" từ dữ liệu hiện có.
- **Phần thật-sự cần Phase 4 (LLM/shared, để sau):** phân loại **câu đơn/phức theo từng câu**, **coherence + liên từ đã dùng/gợi ý** (C4), **fix L3** (render lỗi "thiếu từ" trong track-changes). Các phần này cần mở rộng prompt grammar (đụng `shared/`, ảnh hưởng mock-test) + **test LLM thật** — không verify được ở môi trường hiện tại nên chưa làm "mù".

### Slice 6 — "⛶ Bản đầy đủ" (panel rộng 2 cột) ✅ (build xanh)
- **`PracticeCriterionWidePanel`:** overlay modal (z-60, backdrop, đóng khi click ngoài/X) 2 cột: **transcript (trái)** + **phân tích (phải)** cho lexical/grammar/pronunciation; fluency hiển thị insight 1 cột. Tái dùng TranscriptViewer + 4 insight component + AnnotationPopup riêng.
- `PracticeCriterionCard`: thêm nút **"⛶ Bản đầy đủ"** ở header → mở overlay. Áp dụng cho cả 4 tiêu chí (Part 1/2).

### Slice 7 — Timeout client (§14.5/L10, Phase 6 làm sớm) ✅ (build xanh)
- `practice-client.ts`: helper `fetchWithTimeout` (AbortController hủy fetch khi quá hạn) áp cho 4 endpoint: **assessments 45s** (kết quả đầu), **vocab/grammar 60s** (phân tích sâu), **rewrite 30s**, **word-cefr 30s** (background, đã fail-silent). Quá hạn → `PracticeRequestError` code `client_timeout` → vào banner "thử lại" / tab failed sẵn có (không treo vô hạn). Response trễ đã có guard stale ở hook. Không đụng `shared/`.

### Slice 8 — Analytics §13 cho tương tác mới ✅ (build xanh)
- Thêm 2 event (additive shared, `analytics-events/practice.ts`): `OpenCriterionAnalysisEvent` (bấm thẻ điểm → mở card) + `OpenFullAnalysisEvent` (mở "⛶ Bản đầy đủ"); cùng enum `criteria_type` ("fluency"/"lexical"/"grammar"/"pronunc") như tab cũ.
- Helper `criterionAnalyticsType(key)` ở practice-evaluation lib.
- Fire: `handleOpenCriterion` (Part 1 & 2) → OpenCriterion; nút "Bản đầy đủ" trong card → OpenFull. **PII-safe** (chỉ session/question/criteria_type, qua wrapper analytics chuẩn).
- Các edge case A1/B1 + score viewed đã có tracking sẵn (giữ nguyên). Event cho coach/drill/save-phrase chờ khi build các feature đó (Phase 5/7).

### Slice 9 — Compact UI §9 (pass bảo thủ) ✅ (build xanh)
- Khu kết quả (`PracticeEvaluationPanel`): outer `p-5 sm:p-6 space-y-5` → `p-4 sm:p-5 space-y-4`; score box `rounded-[28px] p-5` → `rounded-2xl p-4`; band `44px` → `34px`; icon/heading nhỏ lại; `mt-5` → `mt-4`; mọi section `rounded-[28px]` → `rounded-2xl`. `PracticeCorrectedAnswer` heading `text-lg` → `text-base`.
- **Bảo thủ có chủ đích:** chỉ siết các phần rõ ràng oversized; các component mới vốn đã compact (text-sm/[13px]). **Cần xem app thật để tinh chỉnh tiếp** (mật độ theo demo) — đây là pass an toàn, không phá layout.

> **HẾT nhóm 🟢 (frontend verify được).** Còn lại đều cần input ngoài: 🟡 Phase 4/5 (test LLM), 🟠 recorder B4 (shared), Part 3 (refactor), Compact tinh chỉnh (xem app).

### Slice 10 — Nối Part 3 ✅ (build xanh)
- **Đính chính:** Part 3 thực ra **dùng đúng kiến trúc Part 1/2** (`PracticeQuestionShell` + `PracticeAttemptTimeline` + `PracticeAssistantRail`) — giả định "kiến trúc khác" (PracticeConsole/SupportRail) ở plan là **sai** (đó là mô tả PRD, không phải page thực tế). Nối = lặp lại Part 2.
- `PracticePart3`: thêm state `criterionAction` + `handleOpenCriterion` (+ analytics OpenCriterion) + reset khi đổi câu; timeline `onOpenCriterion`; rail `criterionAttempts` + `criterionAction`.
- → **3/3 Part** đã có: thẻ điểm bấm được → card phân tích cột AI + tự ẩn 3 tab + "⛶ Bản đầy đủ". §2/§6 hoàn tất cho cả Part 1, 2, 3.

### Còn lại của Phase 3 (deferred)
- (không còn — Part 3 đã nối)

---

## Phase 4 — Grammar/Coherence + fix L3 (đụng `shared/`, cần test LLM staging)

> User chốt "viết trước, test LLM sau". Đụng prompt grammar dùng chung Mock Test → **giữ nhánh, build xanh cả 2, chờ verify model + duyệt**.

### Slice 1 — Fix L3 (track-changes lỗi "thiếu từ") ✅ (build xanh, JSON valid)
- Chỉ sửa **prompt** `evaluation.grammarReview` (`openai-prompts.json`): yêu cầu `text` là đoạn **xuất hiện verbatim trong transcript** (copy lời người học), và **lỗi thiếu từ** phải dùng cụm bao quanh gap (vd thiếu "the" → text "go to school", correctedText "go to the school") thay vì đặt text = từ thiếu (vốn không có trong transcript → bị `findWordMatchesForText` bỏ).
- **Hiệu quả:** lỗi chèn/thiếu từ giờ map được `wordIds` → hiển thị trong track-changes (§3) + highlight grammar. Prompt-only, additive, **không đổi schema** → áp dụng cả Practice lẫn Mock Test (tốt lên đồng đều).
- ⚠️ **Cần test LLM:** verify model thực sự copy verbatim + chọn cụm bao quanh đúng (chưa chạy model ở môi trường này).

### Slice 2 — Grammar sentenceStats + Coherence (§6.4 + C4) ✅ (build xanh) — CẦN VERIFY LLM
- **Schema `evaluation.grammarReview`** (shared) thêm 2 field **required**: `sentenceStats {total, complex, errorFree}` + `coherence {comment, connectorsUsed[], suggestedConnectors[]}`. Type `GrammarReviewResult` + parse (defensive) + prompt (hướng dẫn sinh + ca rỗng) cập nhật.
- **Contract** `shared/types/practice-evaluation.ts`: thêm `grammarSentenceStats?` + `coherence?` vào `PracticeEvaluationResult` (optional, backward compat).
- **Handler** `/grammar` map 2 field vào evaluation.
- **UI:** card Ngữ pháp hiện **độ chính xác = errorFree/total** + **độ đa dạng = complex%/đơn%** (thay ước lượng cũ); card Trôi chảy thêm khối **Mạch lạc** (comment + liên từ đã dùng + nên dùng thêm). Bài cũ thiếu → hiện dòng giải thích.
- ⚠️ **RỦI RO cần verify trước khi merge:**
  1. **Đụng Mock Test:** schema dùng chung → mock-test grammarReview giờ cũng PHẢI trả 2 field này (required) → output dài hơn → **cần đo token** (L2: chưa có log token) tránh truncation cho CẢ practice lẫn mock-test (truncate = grammar review fail cả 2 module).
  2. **Chất lượng:** model có đếm câu/đơn-phức/coherence đúng không — cần chạy staging.
  3. **Persistence:** field mới optional; cần xác minh `practice-persistence` round-trip (nếu không, history không hiện — graceful, không vỡ).

---

## Phase 5 — Coach cải thiện câu (§7), cần verify LLM

### Slice 1 — Structured coach output (diff + cụm đáng học + dịch) ✅ (build xanh) — CẦN VERIFY LLM
- **Prompt + schema `practice.answerRewrite`** (RIÊNG Practice — KHÔNG đụng mock-test) thêm: `changes[]` (diff: text/replacement/category vocab|grammar|collocation|linking|idea/reason), `phrasesToLearn[]` (cụm đáng học EN+VN), `translation` (dịch). Type + parse (defensive + normalize category) + return cập nhật.
- **Contract** `PracticeGeneratedResponse` thêm optional `changes/phrasesToLearn/translation`.
- **UI mới `PracticeCoachAnswer`**: câu nâng cấp với **chỗ thay đổi tô màu theo tiêu chí** (rê chuột = vì sao + kỹ thuật) + **"Bạn đã nâng cấp gì"** (changes theo nhóm) + **"Cụm đáng học"** + **dịch nghĩa**. Dùng cho cả improve & expand trong `GeneratedResponses`.
- **§2.5/E:** **bỏ số band cụ thể** — nút đổi thành "Gợi ý câu trả lời tốt hơn"; coach subtitle "hướng tới band kế tiếp"; xóa `rewriteTargetBandLabel`.
- ⚠️ **Cần verify LLM:** model trả `replacement` verbatim trong answer (để tô màu khớp) + reason/phrases hợp lý — chạy staging. Schema strict → 3 field required (output dài hơn, đo token).

### Slice 2 — Nghe mẫu (TTS) §7/§6.1 (K1) ✅ (build xanh, verify được — không LLM)
- **`practice-tts.ts`** (mới): `speakEnglish(text)` qua Web Speech API (SpeechSynthesis, chọn giọng en, rate chậm) + hook `useSpeechSynthesisAvailable()` dùng `useSyncExternalStore` (SSR-safe, tránh setState-in-effect của React 19).
- **Coach (`PracticeCoachAnswer`)**: nút "🔊 Nghe mẫu" đọc câu nâng cấp.
- **Phát âm (`PracticePronunciationInsight`)**: 🔊 mỗi từ trong top từ cần sửa → nghe âm chuẩn.
- Feature-detect: ẩn nút nếu trình duyệt không hỗ trợ TTS (K1).

### Chưa làm (follow-up)
- **3 nút mục tiêu riêng** (Nâng từ vựng/Đa dạng ngữ pháp/Phát triển ý) — hiện dùng improve/expand (đã có coach richness); tách mode là refactor enum (nhiều nơi).
- **Lưu flashcard** E2 (chờ backend) · **"Thử nói lại theo gợi ý"** (đóng vòng — cần nối recorder).
- **Vạch màu từng câu inline** (§6.4 panel) — per-sentence array, truncation-risky.
- **Ngữ pháp §6.4 đầy đủ** (câu đơn/phức từng câu) + transcript phát âm tô màu theo band + 2 nút nghe đối chiếu + nhóm theo âm + coherence/liên từ — cần **Phase 4** (mở rộng prompt, đụng `shared/`, cần test LLM).

---

## ĐÍNH CHÍNH SCOPE (2026-06-23) — gỡ mọi dấu chân lên scoring chung & Mock Test

> User chỉ ra đúng: scope là **Practice thôi**, không được đụng chấm điểm dùng chung. Đã **revert** các thay đổi chạm Mock Test.

**Đã revert (đưa file dùng chung về đúng `dev`):**
- **Phase 4 grammar** (đụng `evaluation.grammarReview` — dùng chung Mock Test): bỏ hết fix-L3-prompt + `sentenceStats` + `coherence` khỏi `openai-prompts.json` (grammarReview), `evaluation-pipeline.ts` (schema/type/parse), `practice-route-handlers.ts` (mapping), `shared/types/practice-evaluation.ts` (field/type), và UI tiêu thụ. → `grammarReview` của Mock Test **nguyên trạng**.
- **Phase 0 pause field** (`MockExamPauseDurationLevel` + field trên `mock-exam.ts` + helper trong `transcript-pause-classifier.ts` + populate ở `evaluate-practice.ts`): bỏ hết. → 3 file shared dùng chung về đúng `dev`.

**Giữ được mà KHÔNG đụng shared:**
- **§6.2 pause 3 mức** vẫn hoạt động: `PracticeFluencyInsight` tính mức (short/medium/long) **trực tiếp từ `pauseDurationMs`** (đã có sẵn trên transcript word từ trước) — không thêm field/scoring.

**Vẫn còn (đúng phạm vi Practice, KHÔNG đụng Mock Test):**
- `analytics-events/practice.ts` (event Practice), `types/practice-evaluation.ts` (contract Practice + types coach Phase 5), `config/openai-prompts.json` → **`practice.answerRewrite`** (prompt RIÊNG Practice — Mock Test không có), `evaluate-practice.ts` (Practice server). Đây là nội dung Practice-sở-hữu sống dưới `shared/` theo cấu trúc repo, không phải scoring dùng chung.

**Xác minh:** `git diff dev...HEAD -- features/mock-test` = 0 file; 3 file scoring dùng chung (`evaluation-pipeline.ts`, `mock-exam.ts`, `transcript-pause-classifier.ts`) + scope `grammarReview` đã về đúng `dev`.

**Hệ quả tính năng:** §6.4 (câu đơn/phức từng câu, độ chính xác chính xác) + C4 (coherence/liên từ) **rút lại** — chỉ giữ phần suy từ dữ liệu có sẵn (cấu trúc nâng cao, lỗi theo loại, accuracy ước lượng). Muốn làm đầy đủ mà giữ "Practice-only" → cần **call LLM RIÊNG cho Practice** (không đụng grammarReview chung) — chờ bạn quyết.

---

## §6.4 + C4 khôi phục bằng HEURISTIC client-side (2026-06-23) — Practice-only, không LLM

Thay vì call LLM/đụng grammarReview chung, làm bằng suy luận phía client từ dữ liệu đã có:
- **`practice-text-heuristics.ts`** (mới, client): `computeGrammarSentenceHeuristic` (đếm câu không lỗi = câu không chứa text lỗi grammar; đơn/phức = dấu hiệu mệnh đề phụ/quan hệ/nối + dấu phẩy) + `detectConnectors` (quét transcript theo bộ liên từ tĩnh → đã dùng / nên dùng thêm).
- **Card Ngữ pháp**: "Độ chính xác X/Y câu không lỗi" + "Độ đa dạng N% phức/ghép" (ước lượng tương đối — đúng tinh thần §6.4); cấu trúc nâng cao vẫn từ LLM `grammarComplexStructureUsage` (đã có).
- **Card Trôi chảy**: khối "Mạch lạc · liên từ" (đã dùng + nên dùng thêm).
- **KHÔNG**: đụng `shared` scoring, không thêm call LLM, không đổi mock-test. Build xanh, verify được.
- Đánh đổi: phân loại đơn/phức là heuristic (độ chính xác vừa phải) — hiển thị rõ "ước lượng". Muốn chuẩn hơn → call LLM RIÊNG Practice sau (không đụng chung).

---

## Follow-up (2026-06-23) — triển khai lần lượt, Practice-only

1. **L3 (track-changes lỗi "thiếu từ")** ✅ — `findGrammarAnchorMapping` (PracticeEvaluationPanel): nếu `text` không khớp transcript (lỗi chèn), neo vào cụm tồn tại dài nhất của `correctedText`. KHÔNG đụng prompt/scoring chung.
2. **"Thử nói lại theo gợi ý"** ✅ — nút trong coach (`PracticeCoachAnswer.onTryAgain`) thread qua GeneratedResponses → timeline/rail → page (chuyển tab practice + handleRecordToggle). Cả 3 Part.
3. **3 nút mục tiêu (§7)** ✅ — `PracticeRewriteMode += vocab|grammar|depth` (giữ improve/expand cho history); `rewriteStatus`→Partial + computed-key `[mode]`; prompt `practice.answerRewrite` (RIÊNG Practice) route 3 mục tiêu; 3 nút trigger ở panel + GeneratedResponses render generic + onGoal thread. **⚠️ Cần verify LLM staging** (route prompt + flow rewrite) như Phase 5. KHÔNG đụng grammarReview/mock-test.
4. **Lưu flashcard (E2)** ✅ Tầng 1 — `practice-flashcards.ts` lưu **localStorage** + toast (đạt acceptance §7 "Lưu được + toast"); nút "Lưu/Đã lưu" trên mỗi cụm đáng học. Backend chưa có → khi sẵn sàng chỉ thay read/write trong lib (giữ chữ ký), UI không đổi.

**Xác minh:** mock-test = 0 file; scoring chung (grammarReview/evaluation-pipeline/mock-exam/classifier) = 0 diff vs dev. Build xanh từng bước.

### Còn lại (cần đầu vào ngoài)
- Verify LLM staging cho prompt `practice.answerRewrite` (3 goal) + coach output (replacement verbatim).
- Backend flashcard (đồng bộ thiết bị) để nâng từ Tầng 1 localStorage.
- Vạch màu từng câu inline §6.4 (truncation-risky) · Compact tinh chỉnh (theo app).
