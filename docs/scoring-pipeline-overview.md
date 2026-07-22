# Luồng API chấm điểm Mock Test & Practice — Tổng quan cho team Sản phẩm

> Tài liệu này mô tả luồng gọi API mới (sau khi hoàn tất Phase 1–6) cho cả **Mock Test** và **Practice**, đồng thời giải thích từng **prompt** đang được dùng để chấm điểm. Mục tiêu: giúp team sản phẩm hiểu rõ ai gọi cái gì, khi nào, vì sao, và mỗi prompt đang yêu cầu mô hình trả về điều gì.

Cập nhật: 2026-05-19 (Sprint 1-4: parallel vocab/grammar Practice + Mock test progressive finalize + fail-fast calibration + Firestore charge ledger) · Branch: `feature/good-bad-pause-marker`

---

## 1. Bối cảnh nhanh

Trước đây pipeline chấm điểm gọi OpenAI nhiều vòng riêng lẻ, mỗi vòng rescore lại điểm — gây trôi điểm, mâu thuẫn dữ liệu, và khi một bước fail thì client treo / loading vô hạn vì không có recovery path.

Pipeline mới được thiết kế theo nguyên tắc:

1. **Chấm điểm một lần, lock điểm cho các bước sau.** Chỉ có Quick Score quyết định band; Vocab Review và Grammar Review nhận điểm này như input "locked" và **không được rescore**.
2. **Pronunciation band được calibrate bằng ChatGPT (Phase D, fail-fast Sprint 3 M3).** Azure cung cấp aggregate scores (accuracy/fluency/prosody/completeness) + word-level evidence. Một LLM call riêng (`evaluation.pronunciationCalibration`) chuyển toàn bộ tín hiệu Azure thành band IELTS 0–9 trước khi lock cho 3 bước sau. **Nếu LLM call fail → propagate error, client banner + retry** (không còn fallback Azure cứng vì độ chính xác kém — Sprint 3 M3).
3. **Mock Test progressive reveal (Sprint 2 V3A), Practice parallel vocab/grammar (Sprint 1 V1).**
   - **Mock Test Part 1/Part 3 segmented:** finalize tách thành **3 endpoint** `/finalizations/{score,vocab,grammar}` — `/score` chạy calibration + quickScore trước, sau đó `/vocab` + `/grammar` parallel. User thấy band tổng + tab Phát âm ngay sau /score (~15-20s) thay vì chờ 30-50s atomic.
   - **Mock Test Part 2 (whole-part)** giữ atomic 4 bước (1 lần ghi âm dài, không tách được).
   - **Practice:** `/assessments` (calibration + quickScore atomic) → `/vocab` + `/grammar` **parallel** (`Promise.allSettled`). Tab Từ vựng + Ngữ pháp populate độc lập, không chờ tuần tự.
4. **Mỗi bước có status & retry riêng (cả Practice + Mock Test).** Nếu `/vocab` hoặc `/grammar` fail, người dùng vẫn thấy điểm tổng + tab phát âm, và có nút **Thử lại** riêng cho tab Từ vựng hoặc Ngữ pháp. Sequence counter ở client chống race khi user spam retry.
5. **Token deduction theo bài hoàn chỉnh (Sprint 4 M5).** 1 lượt Practice trừ sau khi 4 bước AI (calibration + quickScore + vocab + grammar) đều thành công. 1 lượt Mock Test trừ sau khi 3 parts × 4 bước = đầy đủ. Firestore charge ledger `scoring_charges` lưu state, 2-phase commit lite đảm bảo idempotent + race-safe.

---

## 2. Các scope prompt và endpoint dùng chung

Cả Mock Test và Practice **dùng chung 4 scope OpenAI** đặt trong [`features/shared/server/config/openai-prompts.json`](../features/shared/server/config/openai-prompts.json):

- **`evaluation.pronunciationCalibration`** — function: [`runPronunciationCalibration`](../features/shared/server/evaluation-pipeline.ts) — **Phase D, fail-fast Sprint 3 M3**
  - Nhận Azure aggregate scores (accuracy/fluency/prosody/completeness/pronunciation) + transcript stats + top 10 weak words.
  - Output: `band` (0–9 half-step) + `rationale` (VN ≤2 câu).
  - Gọi **trước** quickScore. Band kết quả được lock cho 3 bước sau. **Fail → propagate error** (Sprint 3 M3), client banner + retry — không còn fallback Azure cứng vì độ chính xác kém.
- **`evaluation.quickScore`** — function: [`runQuickScore`](../features/shared/server/evaluation-pipeline.ts)
  - Chấm 3 tiêu chí Fluency, Lexical, Grammar + tính `overallBand` + `cefrLevel` + `feedbackSummary` + `improvements`.
  - Pronunciation đã được calibration step lock sẵn, prompt cấm rescore.
- **`evaluation.vocabReview`** — function: [`runVocabReview`](../features/shared/server/evaluation-pipeline.ts)
  - Bóc lỗi & gợi ý nâng cấp từ vựng (≤10 lỗi, ≤4 gợi ý) + phân phối CEFR.
  - Không rescore band.
- **`evaluation.grammarReview`** — function: [`runGrammarReview`](../features/shared/server/evaluation-pipeline.ts)
  - Bóc lỗi ngữ pháp cơ bản + lỗi cấu trúc phức tạp + nhận xét chung (đúng template 4 câu).
  - Không rescore band.

Riêng Practice còn 1 scope nữa, **mock test không có**:

- **`practice.answerRewrite`** — function: [`generatePracticeAnswerRewrite`](../features/practice/server/evaluate-practice.ts)
  - Viết lại câu trả lời theo `mode = improve | expand` cho học viên đối chiếu mẫu.

Toàn bộ pronunciation evidence đến từ **Azure Speech Pronunciation Assessment**. **Phase D**: thay vì quy đổi cứng qua [`calculateIeltsPronunciationBand`](../features/shared/lib/pronunciation-band.ts), tín hiệu Azure được gửi cho ChatGPT qua scope `evaluation.pronunciationCalibration` để LLM đóng vai trò examiner và trả về band IELTS. Band kết quả được "lock" trước khi gọi 3 scope còn lại, và `evaluation.quickScore` bị cấm rescore phát âm. **Sprint 3 M3**: bỏ fallback `calculateIeltsPronunciationBand` để đảm bảo độ chính xác; LLM fail = client retry. File `pronunciation-band.ts` được giữ làm reference (chỉ log `azureReferenceBand` cho debug).

**Sprint 2 M1 — Azure aggregates per-question:** Mock Test segmented (Part 1/Part 3) per-question `/assessments` lưu lại 5 Azure aggregate scores trong `MockExamQuestionAssessmentPayload.azureAggregates`. Khi finalize, helper `aggregatePerQuestionAzureAggregates` tính weighted average theo word count → calibration nhận input THẬT (không proxy). Backward compat: assessment cũ chưa có aggregates → fallback proxy.

---

## 3. Mock Test — Luồng API

Mock Test có 2 nhóm route, tuỳ Part:

- **Part 1 / Part 3 (segmented):** mỗi câu hỏi gọi `/assessments` riêng để chấm phát âm Azure cho câu đó. Cuối Part gọi **3 endpoint progressive** `/finalizations/{score,vocab,grammar}` (Sprint 2 V3A) để user thấy band tổng nhanh.
- **Part 2 (whole-part):** không segmented. Một bài nói dài → một lần `/assessments` duy nhất chạy cả Azure + 4 bước OpenAI atomic (calibration + 3 evaluation) trong cùng request.

### 3.1. `POST /api/app/speaking/mock-exams/parts/[part]/assessments`

Handler: [`handleMockExamPartAssessmentsPost`](../features/mock-test/server/mock-exam-route-handlers.ts) → gọi 1 trong 2 evaluator:

- `evaluateMockExamSegmentedQuestion` — chỉ Azure Speech, không OpenAI (Part 1 / Part 3, từng câu). **Sprint 2 M1**: lưu thêm `azureAggregates` (5 score 0-100) vào response.
- `evaluateMockExamOpenAiAzurePart` — Azure + 4 bước OpenAI atomic (Part 2 whole-part).

Input chính: `audio` (File), `part`, `questionId | questionIndex | questionPrompt | questionTitle`, optional `promptOverrides` + `aiScoringOverrides` (admin prompt lab), **`mockExamSessionId`** (Sprint 4 M5 — cho charge ledger).

### 3.2. Progressive finalize Part 1/Part 3 — 3 endpoint (Sprint 2 V3A)

Thay vì 1 endpoint `/finalizations` atomic, giờ tách thành 3 endpoint chạy progressive:

| Endpoint | Mục đích | Time |
|---|---|---|
| `POST /finalizations/score` | Calibration + Quick Score atomic | ~15-20s |
| `POST /finalizations/vocab` | Vocab Review (cần `lockedScore` từ /score) | ~10-15s |
| `POST /finalizations/grammar` | Grammar Review (cần `lockedScore` từ /score) | ~10-15s |

Handler: [`handleMockExamPartFinalizationScorePost`](../features/mock-test/server/mock-exam-route-handlers.ts) / `...VocabPost` / `...GrammarPost` → gọi 3 evaluator riêng [`evaluateMockExamSegmentedPart{Score,Vocab,Grammar}`](../features/mock-test/server/evaluate-mock-exam-segmented.ts).

Sequence trên client:
1. `/score` chạy đầu tiên → user thấy overall band + tab Phát âm ngay.
2. Client gom `lockedScore` từ response → fire `/vocab` + `/grammar` **parallel** (`Promise.allSettled`).
3. Tab Từ vựng + Ngữ pháp populate độc lập khi mỗi response về.

**Endpoint cũ `/finalizations` atomic vẫn giữ** (backward compat 30 ngày cho client cũ chưa update).

### 3.3. Pipeline 4 bước (vẫn giữ trong từng evaluator)

Pipeline AI internal vẫn là 4 bước, nhưng sách chia ra qua 3 HTTP endpoint cho Part 1/3:

```
                ┌──────────────────────────────────────┐
                │ Azure Speech evidence (per-question  │
                │ assessments đã có sẵn — Sprint 2 M1) │
                │ → transcript + 5 aggregate scores    │
                │   (accuracy/fluency/prosody/etc.)    │
                └──────────────────────────────────────┘
                              │
                              ▼
    ╔══════════════════════ /finalizations/score ══════════════════════╗
    ║                                                                  ║
    ║  ┌──────────────────────────────────────┐                        ║
    ║  │ runPronunciationCalibration()        │ ← Phase D + M3         ║
    ║  │ scope: pronunciationCalibration      │                        ║
    ║  │ → band (0–9) + rationale VN          │                        ║
    ║  │ • Fail → throw (Sprint 3 M3)         │                        ║
    ║  └──────────────────────────────────────┘                        ║
    ║                │  (pronunciation band lock)                      ║
    ║                ▼                                                 ║
    ║  ┌──────────────────────────────────────┐                        ║
    ║  │ runQuickScore() → lockedScore        │                        ║
    ║  │ → overallBand, criteria, cefrLevel,  │                        ║
    ║  │   summary, improvements              │                        ║
    ║  └──────────────────────────────────────┘                        ║
    ║                                                                  ║
    ║  Response: partial MockExamPartEvaluationPayload                 ║
    ║  (vocab/grammar arrays = [], chỉ pronunciation annotations)      ║
    ║                                                                  ║
    ╚═══════════════════════════════════════════════════════════════════╝
                              │
                  (Client store lockedScore + fire 2 endpoints parallel)
                              │
                ┌─────────────┴─────────────┐
                ▼                           ▼
    ╔ /finalizations/vocab ╗     ╔ /finalizations/grammar ╗
    ║                      ║     ║                        ║
    ║ runVocabReview()     ║     ║ runGrammarReview()     ║
    ║ → lexicalProfile     ║     ║ → grammarProfile       ║
    ║   vocabAnnotations   ║     ║   grammarAnnotations   ║
    ║                      ║     ║                        ║
    ╚══════════════════════╝     ╚════════════════════════╝
                │                           │
                └──────► merge ◄────────────┘
                         vào UI state per-tab
```

**Per-step status (Sprint 2 V3A):** mỗi `MockExamPartEvaluation` có thêm field `partVocabReviewStatus` + `partGrammarReviewStatus`:
- `"pending"` — đang chờ response từ /vocab hoặc /grammar.
- `"completed"` — response đã về.
- `"failed"` — fail; banner đỏ + nút **Thử lại Từ vựng/Ngữ pháp** (Sprint 4+ retry).

**Atomic boundaries:**
- Mỗi endpoint atomic riêng. `/score` fail → markPartFailed → user retry cả part. `/vocab` hoặc `/grammar` fail độc lập → banner đỏ per-tab + retry button (KHÔNG ảnh hưởng band tổng).
- Part 2 (whole-part) vẫn atomic 4 bước trong 1 endpoint `/assessments`.

### 3.4. Client phía Mock Test (Sprint 2 V3A)

- `evaluateMockExamAudio` ([`features/mock-test/lib/mock-exam-client.ts`](../features/mock-test/lib/mock-exam-client.ts)) gọi `/assessments` per-question (Part 1/3 Azure-only) hoặc `/assessments` atomic (Part 2).
- `finalizeMockExamSegmentedPartScore` / `...Vocab` / `...Grammar` — 3 client function gọi 3 endpoint progressive.
- `finalizeMockExamSegmentedPart` (legacy atomic) giữ để backward compat 30 ngày.
- [`useSegmentedPartFinalization`](../features/mock-test/hooks/useSegmentedPartFinalization.ts) hook auto-detect khi part finalize ready → chạy `/score` → setPartEvaluation với vocab/grammar pending → `Promise.allSettled([/vocab, /grammar])` → merge slice vào state.
- Hook expose `retryPartVocabFinalization(part)` + `retryPartGrammarFinalization(part)` cho retry per-tab (Sprint 4+). Sequence counter chống race khi user spam click.

---

## 4. Practice — Luồng API

Practice tách 3 endpoint riêng để có **progressive reveal**: học viên thấy điểm tổng + transcript + tab Phát âm ngay sau `/assessments`, sau đó `/vocab` + `/grammar` chạy **parallel** (Sprint 1 V1) — tab Từ vựng và Ngữ pháp populate ĐỘC LẬP, không chờ nhau. Mỗi tab có retry riêng nếu fail.

### 4.1. `POST /api/app/speaking/practice/assessments`

Handler: [`handlePracticeAssessmentsPost`](../features/practice/server/practice-route-handlers.ts) → gọi `evaluatePracticeAudio` ([`features/practice/server/evaluate-practice.ts`](../features/practice/server/evaluate-practice.ts)).

Bên trong:

1. Azure Speech Pronunciation Assessment trên file audio → transcript + word-level pronunciation + aggregate scores + pause stats.
2. **`runPronunciationCalibration`** (scope `evaluation.pronunciationCalibration`) — LLM chuyển Azure aggregates + weak words thành IELTS Pronunciation band 0–9 + rationale VN. **Sprint 3 M3: fail-fast** — LLM throw → propagate ra route handler → client banner + retry.
3. `runQuickScore` (scope `evaluation.quickScore`) — sinh điểm tổng + 3 tiêu chí + `cefrLevel` + `feedbackSummary` + `improvements`. Nhận band pronunciation đã lock từ bước 2.

Output: payload `PracticeEvaluationResult` đầy đủ điểm số + transcript + pronunciation. Hai trường `corrections` và `correctionsStatus` ban đầu là `null` / `"idle"` — client sẽ gọi tiếp `/vocab` rồi `/grammar` để populate.

### 4.2. `POST /api/app/speaking/practice/vocab`

Handler: [`handlePracticeVocabPost`](../features/practice/server/practice-route-handlers.ts) → gọi `runVocabReview` với `lockedScore` được rút từ response của `/assessments`.

Trả về `evaluation` đã được merge thêm:
- `corrections.lexical` (tối đa 10 lỗi).
- `vocabCefrDistribution` (6 mức A1..C2, tổng 100%).
- `annotations` (highlight lexical trên transcript, đã giữ lại annotations grammar/pronunciation nếu đã có).
- `correctionsStatus: "completed"`.

### 4.3. `POST /api/app/speaking/practice/grammar`

Handler: [`handlePracticeGrammarPost`](../features/practice/server/practice-route-handlers.ts) → gọi `runGrammarReview` với cùng `lockedScore`.

Trả về `evaluation` đã được merge thêm:
- `corrections.grammar` (lỗi cơ bản + cấu trúc phức tạp).
- `grammarErrorSummary`, `grammarComplexStructureUsage`, `grammarGeneralFeedback` (4 câu VN cố định).
- `annotations` (highlight grammar, preserved lexical/pronunciation).
- `correctionsStatus: "completed"`.

### 4.4. `POST /api/app/speaking/practice/rewrites`

Handler: [`handlePracticeRewritesPost`](../features/practice/server/practice-route-handlers.ts) → gọi `generatePracticeAnswerRewrite` (scope `practice.answerRewrite`).

Input: `mode = "improve" | "expand"`, `targetBand`, `evaluation` (transcript + `lockedScore` + corrections để AI biết tránh lỗi gì khi viết lại).
Output: `answer` (English rewrite) + `summary` (tiếng Việt giải thích vì sao).

### 4.5. Client orchestration (Sprint 1 V1+V2 — parallel)

[`useSpeakingPractice`](../features/practice/hooks/useSpeakingPractice.ts) thực hiện luồng:

```
   recording → upload
        │
        ▼
   POST /practice/assessments     ← atomic: calibration + quickScore
        │ (recorderState: "scored", show transcript + điểm tổng + tab Phát âm)
        ▼
   setEvaluation({ vocabReviewStatus: "pending",
                   grammarReviewStatus: "pending" })
        │
        ▼
   Promise.allSettled([
       POST /practice/vocab,        ← parallel (Sprint 1 V1)
       POST /practice/grammar,      ← parallel
   ])
        │
        ├─ vocab thành công → applyVocabSuccess (merge lexical slice, status=completed)
        ├─ vocab thất bại   → applyVocabFailure (status=failed, vocabReviewError set)
        ├─ grammar thành công → applyGrammarSuccess (merge grammar slice, status=completed)
        └─ grammar thất bại  → applyGrammarFailure (status=failed, grammarReviewError set)
        │
        ▼
   Khi cả 2 đều completed → set detailStatus="completed" + fire onEvaluation isFinal=true
```

**Sprint 1 V2**: vocab fail KHÔNG block grammar (cũ: sequential, vocab fail = bỏ qua grammar). Giờ độc lập, fail riêng.

**Sprint 1 L2** (double-click guard): `isRunningEvaluationRef` ref-based mutex chặn double-fire `/assessments` trong cùng tick trước React re-render. Release ở `finally` để vocab/grammar background không bị block.

**Sprint 2 M2** (race chống retry stale): mỗi `retryVocabReview` / `retryGrammarReview` bump sequence counter. Response cũ stale → drop khi seq mismatch.

UI mapping (tham khảo [`PracticeEvaluationPanel.tsx`](../features/practice/components/practice/PracticeEvaluationPanel.tsx)):

- `vocabReviewStatus === "pending"` → tab Từ vựng hiện `CorrectionsSkeleton`.
- `vocabReviewStatus === "failed"` → tab Từ vựng hiện banner đỏ + nút **Chấm lại Từ vựng**.
- `vocabReviewStatus === "completed"` → render dữ liệu thật.
- Tương tự cho Ngữ pháp.
- Tab Phát âm không phụ thuộc 2 status trên (dữ liệu Azure đã có từ `/assessments`).

**Sprint 1 L3 — rewrite failed state:** `rewriteStatus` thêm `"failed"`. Hook `requestAnswerRewrite` catch lỗi → set state failed + `rewriteError.improve/expand` thay vì throw. UI `GeneratedResponses` hiện `RewriteFailedBanner` với retry button reuse `onImprove`/`onExpand` callback.

**Sprint 1 L4 — audio upload toast:** `primePracticeAttemptPersistence` catch upload error → toast warning "Không lưu được bản ghi âm. Kết quả chấm điểm vẫn được lưu" + alpha-log `practice-audio-upload-failed`.

Nút **"Xem chấm điểm chi tiết cho câu"** (Phase A): short-circuit khi `vocabReviewStatus === "completed"` && `grammarReviewStatus === "completed"` → chỉ flip `detailStatus: "completed"` (không re-fire OpenAI). Alpha-log event `practice-detail-skip-completed`.

### 4.6. Tương thích history cũ

History trước Phase 6 **không** có `vocabReviewStatus` / `grammarReviewStatus`. UI dùng cờ `legacyCorrectionsFallback`: khi cả 2 status đều `undefined`, rơi về `correctionsStatus` để giữ hành vi cũ (skeleton chung). Tất cả field Phase 4 (`vocabCefrDistribution`, `grammarErrorSummary`, `grammarComplexStructureUsage`, `grammarGeneralFeedback`) đều optional + chỉ persist, không có UI consume trực tiếp.

---

## 5. Logging & cách đọc log để debug

Mọi handler đều có:

- `requestId` (UUID v4) gắn vào toàn bộ log của request đó — search theo `requestId` để xem trọn chuỗi sự kiện.
- `appendAlphaLog(createAlphaLogRecord(...))` ghi vào alpha log buffer (xem ở `/settings/logs`).
- `runWithAlphaLogContext` bọc request để mọi log con tự kế thừa `sessionId` + `userId`.

Mỗi route phát các event chuẩn:

- **`*-started`** — info — phát ra trước khi gọi pipeline.
- **`*-succeeded`** — info — phát ra sau khi pipeline trả về thành công.
- **`*-ai-failed`** — error — OpenAI structured-output throw `OpenAiStructuredOutputError`.
- **`*-route-failed`** — error — mọi lỗi khác (auth, rate limit, network, parse).

Per-phase log (Phase 6C + Phase D + Sprint 2-4) cho phép biết **đúng bước nào fail** trong pipeline 4 bước:

**Mock Test:**
- **`mock-exam-pronunciation-calibration-started/succeeded/failed`** — bước calibration. Sprint 3 M3: failed = propagate, **không** silent fallback.
- **`mock-exam-quick-score-started/succeeded/failed`** — quickScore.
- **`mock-exam-vocab-review-started/succeeded/failed`** — vocabReview.
- **`mock-exam-grammar-review-started/succeeded/failed`** — grammarReview.
- **`mock-exam-finalize-{score,vocab,grammar}-{request,response,route-failed}`** — Sprint 2 V3A: 3 endpoint progressive finalize riêng cho Part 1/Part 3.

**Practice:**
- **`practice-pronunciation-calibration-started/succeeded/failed`** — calibration (Phase D, trong `/assessments`).
- **`practice-quick-score-started/succeeded/failed`** — quickScore (trong `/assessments`).
- **`practice-vocab-review-started/succeeded/ai-failed/route-failed`** — endpoint `/practice/vocab`.
- **`practice-grammar-review-started/succeeded/ai-failed/route-failed`** — endpoint `/practice/grammar`.
- **`practice-detail-skip-completed`** — Phase A: nút "Xem chi tiết" short-circuit khi vocab+grammar đã hoàn tất.
- **`practice-audio-upload-failed`** — Sprint 1 L4: upload audio sang Cloud Storage fail.

**Charge ledger (Sprint 4 M5):**
- **`scoring-charge-locked`** — Phase 1 txn lock state="charging", chuẩn bị gọi NineSpeak deduct.
- **`scoring-charge-completed`** — Phase 3 commit state="charged".
- **`scoring-charge-failed`** — Phase 2 NineSpeak deduct fail; doc kẹt state="charge-failed" cho admin reconcile.
- **`scoring-charge-stuck-cleanup`** — admin manual trigger: marked N stuck charging docs as failed (server crash giữa Phase 2-3).
- **`scoring-charge-stuck-cleanup-failed`** — exception trong manual cleanup.

Khi user báo "tab Từ vựng đỏ", filter alpha log theo user → `requestId` → tìm `practice-vocab-review-*-failed` để biết error message + `aiFailure` (status code, finishReason, payloadSnippet) của lần gọi OpenAI.

---

## 6. Giải thích các Prompt đang dùng

Prompt file: [`features/shared/server/config/openai-prompts.json`](../features/shared/server/config/openai-prompts.json). Mỗi scope có `system` (prompt hệ thống) + `userTemplate` (template chứa các placeholder dạng `{{biến}}` được render lúc gọi).

Admin có thể override prompt qua **Prompt Lab** (`/settings/prompt-lab`). Khi override hợp lệ, override được gửi kèm vào request và thay thế prompt mặc định cho riêng request đó (không persist).

### 6.1. `evaluation.pronunciationCalibration` (dùng chung Mock Test + Practice) — Phase D

**Vai trò:** chuyển Azure aggregate scores + word-level evidence → IELTS Pronunciation band (0–9 half-step) + rationale VN ngắn. **Chỉ chấm pronunciation** — prompt cấm rescore Fluency, Lexical, Grammar.

**Anchors band** trong system prompt (4 / 4.5 / 5 / 5.5 / 6 / 6.5 / 7 / 8) dựa trên ngưỡng cụ thể của các Azure scores:

- *AccuracyScore* (segmental accuracy 0–100).
- *ProsodyScore* (stress + intonation 0–100).
- *FluencyScore* (connected speech 0–100).
- *CompletenessScore* (whether learner attempted whole utterance 0–100).
- *PronunciationScore* (Azure overall summary 0–100).
- *Weak word percent* + *top 10 weak words*.

Quy tắc: không over-rely vào AccuracyScore (strong accuracy nhưng flat prosody = band 6 ceiling). Round conservatively khi transcript < 20 words hoặc completenessScore < 50. Khi 2 anchor cạnh tranh, ưu tiên band thấp hơn trừ khi prosody rõ ràng mạnh.

**Đầu vào (userTemplate):**

```jsonc
{
  "part": "Practice Part 1",
  "topicTitle": "Hometown",
  "prompt": "Where is your hometown?",
  "transcript": "...",
  "azureAggregates": {
    "accuracyScore": 87,
    "fluencyScore": 82,
    "prosodyScore": 78,
    "completenessScore": 95,
    "pronunciationScore": 84
  },
  "transcriptStats": {
    "wordCount": 75,
    "durationSeconds": 28,
    "speechRateWpm": 102,
    "goodPauseCount": 3,
    "badPauseCount": 2,
    "fillerCount": 1
  },
  "wordScoreDistribution": {
    "goodWordPercent": 78,
    "fairWordPercent": 17,
    "poorWordPercent": 5,
    "weakWordCount": 4
  },
  "topWeakWords": [
    { "word": "specifically", "accuracyScore": 51 },
    { "word": "develop", "accuracyScore": 42 }
  ]
}
```

**Đầu ra:**

```jsonc
{
  "band": 6.5,
  "rationale": "Phát âm rõ ràng đa số từ thông dụng. Âm cuối /θ/ và /ð/ vẫn yếu cùng ngữ điệu phẳng kéo band xuống nửa nấc."
}
```

**Fail-fast (Sprint 3 M3):** Khi LLM throw (OpenAiStructuredOutputError, timeout, validation fail), server **propagate lỗi** ra route handler → return 5xx → client banner + retry. KHÔNG còn fallback Azure formula như giai đoạn đầu Phase D vì band Azure cứng không đủ chính xác (PR phản hồi user). File `calculateIeltsPronunciationBand` giữ làm reference (log `azureReferenceBand` trong start event).

### 6.2. `evaluation.quickScore` (dùng chung Mock Test + Practice)

**Vai trò:** chấm 3 tiêu chí Fluency / Lexical / Grammar trên thang 0–9 IELTS (cho phép nửa band). **Pronunciation đã bị lock từ Azure** — prompt cấm rescore.

**Anchors band rõ ràng** trong system prompt cho từng tiêu chí (5/6/7/8):

- *Fluency & Coherence*: ưu tiên đánh giá tốc độ ổn định, số bad pauses (tìm từ), filler "uh/um", linking words, mức self-correction phá flow.
- *Lexical Resource*: dải CEFR thống trị (A2-B1 → 5, B1-B2 → 6, B2-C1 → 7, C1-C2 → 8), collocations, từ chuyên đề, mật độ lỗi.
- *Grammatical Range & Accuracy*: tỉ lệ câu phức so với câu đơn, các cấu trúc đã thử (conditional, inversion, cleft), tỉ lệ lỗi cơ bản.

**Đầu vào (userTemplate):**

```jsonc
{
  "part": "...",                // "Phần 1: Phỏng vấn", "Practice Part 2"...
  "topicTitle": "...",          // có thể null
  "prompt": "...",              // đề bài / câu hỏi
  "transcript": "...",          // Azure transcript
  "metrics": {
    "durationSeconds": ...,
    "speechRateWpm": ...,
    "goodPauseCount": ...,      // pause kết câu — tốt
    "badPauseCount": ...,       // pause giữa câu để tìm từ — xấu
    "fillerCount": ...
  },
  "lockedPronunciationBand": 6.5  // đã calibrate sang IELTS
}
```

**Đầu ra:**

```jsonc
{
  "overallBand": 6.5,                  // = trung bình 4 tiêu chí, round IELTS
  "criteria": {
    "fluencyAndCoherence": 6.5,
    "lexicalResource": 6.0,
    "grammaticalRangeAndAccuracy": 6.5,
    "pronunciation": 0                  // sẽ được server inject từ Azure
  },
  "cefrLevel": "B2",                   // map overallBand → CEFR
  "feedbackSummary": "...",            // Tiếng Việt, tự nhiên
  "improvements": ["...", "...", ...]  // ≤5 mục, tiếng Việt
}
```

Sau khi nhận về, server còn **recompute** `overallBand` từ 4 criteria (inject lại pronunciation từ Azure) để đảm bảo không bị drift. Lock này là source-of-truth cho 2 bước sau.

### 6.3. `evaluation.vocabReview` (dùng chung Mock Test + Practice)

**Vai trò:** bóc lỗi từ vựng + gợi ý nâng cấp. **Không rescore.** Tên field giữ tiếng Anh theo schema, nhưng nội dung học viên thấy là tiếng Việt.

**Constraints quan trọng:**

- `maxErrors` = 10 và `maxSuggestions` = 4 (từ [`config/vocab-limits.ts`](../features/shared/server/config/vocab-limits.ts)). Prompt yêu cầu **ưu tiên mục có giá trị học cao nhất** khi vượt giới hạn.
- `lexicalErrors[]`: mỗi item có `errorSubtype` thuộc 5 enum cố định + `errorExplanation` 1-2 câu VN giải thích vì sao sai và vì sao bản sửa tốt hơn:
  - `wrong_meaning` — từ tồn tại nhưng dùng sai nghĩa ("I gained a headache" → "got").
  - `wrong_collocation` — đúng nghĩa, sai kết hợp ("do a mistake" → "make a mistake").
  - `wrong_register` — sai sắc thái formal/informal.
  - `literal_translation` — dịch word-by-word ("open the TV" → "turn on the TV").
  - `wrong_word_form` — sai dạng từ ("develop country" → "developed country").
- `lexicalSuggestions[]`: từ học viên **dùng đúng** nhưng nên nâng cấp. 4 rule kích hoạt một gợi ý:
  - Từ ≤ B1 trong khi học viên đã thể hiện ≥ B2 ở chỗ khác.
  - Từ high-frequency/overused (good, bad, nice, big, very, important, thing, get, do, make).
  - Lặp ≥ 3 lần cho cùng ý.
  - Từ mô tả mơ hồ trong khi có thể dùng từ chuyên đề.
  Mỗi suggestion có 3–4 `replacementOptions` với `candidate`, `meaning` (VN), `cefrLevel` (cao hơn original), `contextNote` (VN, khi nào dùng). Original A2 → ưu tiên candidate ≥ B2; original B1 → ít nhất một candidate ≥ B2.
- `cefrDistribution`: 6 phần trăm A1..C2 phản ánh **độ khó từ vựng của transcript trong bối cảnh** (không phải pronunciation). Tổng đúng 100.
- Transcript trống / quá ngắn (< ~20 từ) / không phải tiếng Anh → arrays rỗng + distribution hợp lý vẫn tổng 100.

**Đầu vào:**

```jsonc
{
  "part": "...",
  "topicTitle": "...",
  "prompt": "...",
  "transcript": "...",
  "lockedScore": {
    "overallBand": 6.5,
    "cefrLevel": "B2",
    "criteria": { ... }       // đầy đủ 4 tiêu chí đã lock
  },
  "limits": {
    "maxErrors": 10,
    "maxSuggestions": 4
  }
}
```

### 6.4. `evaluation.grammarReview` (dùng chung Mock Test + Practice)

**Vai trò:** bóc lỗi ngữ pháp + đánh giá nỗ lực dùng cấu trúc phức tạp + đưa ra feedback chung theo template cố định. **Không rescore.**

**Cấu trúc output:**

- `grammarErrors[]`: lỗi ngữ pháp cơ bản. Subtype enum:
  `verb_tense | subject_verb_agreement | singular_plural | article | preposition | word_order | other`.
  Yêu cầu **không rewrite cả câu** trừ khi cả câu mới là phần sai. Mỗi item: `text` (đoạn ngắn sai), `correctedText` (bản sửa ngắn), `subtype`, `explanation` (1 câu VN).
- `complexStructureErrors[]`: lỗi khi học viên cố dùng cấu trúc nâng cao. Subtype enum:
  `inversion | relative_clause | conditional_1 | conditional_2 | conditional_3 | mixed_conditional | subjunctive | modal_passive | cleft_sentence | reduced_relative | participle_phrase | other`.
  Thêm `structureType` (tên VN dễ đọc, ví dụ "Câu điều kiện loại 2").
- `errorSummary`: `{ mostCommonSubtype, count }` của `grammarErrors[]`. Rỗng → `mostCommonSubtype: null, count: 0`.
- `complexStructureUsage`: 3 mảng string mô tả thí nghiệm cấu trúc nâng cao:
  - `attempted`: tất cả tên cấu trúc đã thử (đúng hay sai cũng tính).
  - `successful`: subset dùng đúng.
  - `failed`: subset dùng sai (overlap với `complexStructureErrors[]`).
- `generalFeedback`: **đoạn 4 câu tiếng Việt cố định theo template 4 slot**, ≤ 90 từ:
  1. Đánh giá tổng quan độ chính xác (yếu / cơ bản ổn / khá / vững).
  2. Điểm yếu nổi bật theo `errorSummary.mostCommonSubtype` + count (nếu không có lỗi thì thừa nhận).
  3. Độ đa dạng câu, dựa vào `complexStructureUsage` (số attempted/successful/failed + tên cấu trúc sai).
  4. Một action item ≤ 15 từ gắn với câu 2 hoặc 3.
  Cấm padding, cấm khen chung chung, cấm double negative, cấm hedging mơ hồ ("khá là", "có vẻ", "tương đối").
- Transcript trống / quá ngắn / không phải tiếng Anh → arrays rỗng + feedback thừa nhận thiếu evidence.

**Đầu vào:** giống `vocabReview` (transcript + lockedScore), không có `limits`.

### 6.5. `practice.answerRewrite` (chỉ Practice)

**Vai trò:** viết lại câu trả lời của học viên cho 2 chế độ.

**Hai mode:**

- `improve` — sửa các lỗi quan trọng nhất về từ vựng, ngữ pháp, cohesion, từ nhạy phát âm. **Không** kéo dài bài.
- `expand` — viết dài hơn, phong phú hơn, dùng câu phức và từ vựng chuyên đề phù hợp `targetBand`.

**Quy ước output:** `answer` bằng tiếng Anh, `summary` bằng tiếng Việt (giải thích vì sao viết lại như vậy / điểm chính cần học).

**Đầu vào:**

```jsonc
{
  "mode": "improve" | "expand",
  "targetBand": 7.0,                  // guidance, không bắt buộc
  "part": "...",
  "topicTitle": "...",
  "prompt": "...",
  "originalAnswer": "...",            // transcript
  "lockedScore": { "overallBand": 6.5, "criteria": { ... } },
  "corrections": { ... }              // shape legacy đầy đủ để AI tránh lặp lỗi
}
```

---

## 7. Tra cứu nhanh các file quan trọng

- **Prompts JSON** — [`features/shared/server/config/openai-prompts.json`](../features/shared/server/config/openai-prompts.json)
- **Vocab limits (maxErrors, maxSuggestions)** — [`features/shared/server/config/vocab-limits.ts`](../features/shared/server/config/vocab-limits.ts)
- **Pipeline 4 bước (`runPronunciationCalibration`, `runQuickScore`, `runVocabReview`, `runGrammarReview` + schemas)** — [`features/shared/server/evaluation-pipeline.ts`](../features/shared/server/evaluation-pipeline.ts)
- **Pronunciation band reference formula** — [`features/shared/lib/pronunciation-band.ts`](../features/shared/lib/pronunciation-band.ts) (chỉ dùng log, không fallback)
- **Practice evaluator (Azure + calibration + quickScore)** — [`features/practice/server/evaluate-practice.ts`](../features/practice/server/evaluate-practice.ts)
- **Mock Test evaluators (`evaluateMockExamSegmentedPart{Score,Vocab,Grammar}` + atomic legacy)** — [`features/mock-test/server/evaluate-mock-exam-segmented.ts`](../features/mock-test/server/evaluate-mock-exam-segmented.ts)
- **Practice route handlers (4 endpoints)** — [`features/practice/server/practice-route-handlers.ts`](../features/practice/server/practice-route-handlers.ts)
- **Mock Test route handlers (assessments + 3 finalize phases + atomic legacy)** — [`features/mock-test/server/mock-exam-route-handlers.ts`](../features/mock-test/server/mock-exam-route-handlers.ts)
- **Practice client (fetch + alpha log)** — [`features/practice/lib/practice-client.ts`](../features/practice/lib/practice-client.ts)
- **Mock Test client (`finalizeMockExamSegmentedPart{Score,Vocab,Grammar}` + atomic legacy)** — [`features/mock-test/lib/mock-exam-client.ts`](../features/mock-test/lib/mock-exam-client.ts)
- **Practice orchestration hook (parallel vocab/grammar)** — [`features/practice/hooks/useSpeakingPractice.ts`](../features/practice/hooks/useSpeakingPractice.ts)
- **Mock Test progressive finalize hook (auto + retry)** — [`features/mock-test/hooks/useSegmentedPartFinalization.ts`](../features/mock-test/hooks/useSegmentedPartFinalization.ts)
- **Practice UI panel + tab status** — [`features/practice/components/practice/PracticeEvaluationPanel.tsx`](../features/practice/components/practice/PracticeEvaluationPanel.tsx)
- **Mock Test report UI partial state + retry banner** — [`features/mock-test/components/reports/MockExamReportNavigation.tsx`](../features/mock-test/components/reports/MockExamReportNavigation.tsx)
- **Mock Test report page (wire retry callbacks)** — [`features/mock-test/pages/reports/MockExamReport.tsx`](../features/mock-test/pages/reports/MockExamReport.tsx)
- **beforeunload guard (Practice + Mock Test)** — [`features/mock-test/hooks/useMockExamBeforeUnloadGuard.ts`](../features/mock-test/hooks/useMockExamBeforeUnloadGuard.ts) + inline trong `useSpeakingPractice`
- **Firestore charge ledger (Sprint 4 M5)** — [`features/mock-test/server/mocktest-charge-ledger.ts`](../features/mock-test/server/mocktest-charge-ledger.ts) (Mock Test) + [`features/practice/server/practice-charge-ledger.ts`](../features/practice/server/practice-charge-ledger.ts) (Practice)
- **Firebase admin client (Auth + Firestore)** — [`lib/firebase/admin.ts`](../lib/firebase/admin.ts)
- **Admin manual cleanup stuck charges** — [`app/api/internal/cron/cleanup-stuck-charges/route.ts`](../app/api/internal/cron/cleanup-stuck-charges/route.ts) (POST với `Authorization: Bearer $CRON_SECRET`)
- **Alpha logger** — [`features/shared/server/alpha-logger.ts`](../features/shared/server/alpha-logger.ts) + UI ở `/settings/logs`
- **Prompt Lab override** — [`features/shared/lib/prompt-overrides.ts`](../features/shared/lib/prompt-overrides.ts) + UI ở `/settings/prompt-lab`
- **AI Scoring Settings (engine OpenAI/DeepSeek + model + token limits)** — [`features/onboarding-dashboard/pages/AiScoringSettingsPage.tsx`](../features/onboarding-dashboard/pages/AiScoringSettingsPage.tsx) + UI ở `/ai-settings`

---

## 8. Tóm tắt khác biệt Mock Test vs Practice

- **Số endpoint OpenAI**
  - Mock Test Part 1/Part 3 segmented (Sprint 2 V3A): **3 endpoint progressive** `/finalizations/{score,vocab,grammar}`. Per-question `/assessments` chỉ Azure-only.
  - Mock Test Part 2 (whole-part): 1 endpoint `/assessments` chứa 4 bước atomic.
  - Practice: 3 endpoint — `/assessments` (calibration + quickScore atomic) → `/vocab` + `/grammar` **parallel** (Sprint 1 V1).
- **Gói atomic?**
  - Mock Test Part 1/3: `/score` atomic 2 bước; `/vocab` và `/grammar` atomic riêng. Mỗi endpoint fail độc lập.
  - Mock Test Part 2: atomic 4 bước, fail bất kỳ = throw cả request.
  - Practice: calibration + quickScore atomic trong `/assessments`; vocab + grammar parallel + fail độc lập.
- **Có nút retry per-tab?**
  - Mock Test: **có** (Sprint 4+ retry button — clone pattern Practice). Per-tab Vocab + Ngữ pháp ở report page.
  - Practice: có (tab Từ vựng / Ngữ pháp riêng) từ Phase 6A.
- **Có scope `answerRewrite`?**
  - Mock Test: không.
  - Practice: có. Sprint 1 L3: failed state + retry banner cho rewrite.
- **Có `cefrLevel` user-facing?**
  - Mock Test: có (trong report).
  - Practice: có (trong panel).
- **Pronunciation source**
  - Mock Test: ChatGPT calibration (Phase D), fail-fast (Sprint 3 M3).
  - Practice: ChatGPT calibration (Phase D), fail-fast (Sprint 3 M3).
- **Prompt overrides (Prompt Lab)**
  - Mock Test: áp dụng được cho cả 4 scope `evaluation.*` (`pronunciationCalibration`, `quickScore`, `vocabReview`, `grammarReview`).
  - Practice: áp dụng được cho cả 5 scope (`evaluation.*` + `practice.answerRewrite`).
- **Charge ledger (Sprint 4 M5)**
  - Mock Test: 1 lượt trừ khi 3 parts × 3 step = 9 step xong (Part 2 atomic mark 3 step cùng lúc). Doc ID `{userId}__mocktest__{sessionId}`.
  - Practice: 1 lượt trừ khi 3 step (scoreDone + vocabDone + grammarDone) xong. Doc ID `{userId}__practice__{requestId}`. Improve/expand rewrite KHÔNG tính.
- **beforeunload guard (Sprint 3 V4)**
  - Mock Test: hook `useMockExamBeforeUnloadGuard` scan session.parts.* trong "processing" hoặc vocab/grammar "pending".
  - Practice: inline useEffect trong `useSpeakingPractice` check status + vocab/grammar/detail status.

---

Nếu cần thêm chi tiết về một scope cụ thể (ví dụ muốn xem schema validator JSON đầy đủ, hay muốn xem thêm anchor band cho Pronunciation calibration của Azure), liên hệ team kỹ thuật để cập nhật tài liệu này.

---

## 9. Prompt thực tế & sample request gửi đi OpenAI

Phần này dán nguyên văn các prompt đang chạy production và một request thực tế gửi đi `/chat/completions` cho từng scope.

### 9.1. Tổng quan wire-format gửi đi OpenAI

Mọi scope đều đi qua cùng một wrapper [`requestStructuredOpenAiJson`](../features/shared/server/openai-structured-output.ts). Cấu hình mặc định (từ env, có thể override qua **Prompt Lab > AI Scoring**):

- **Provider URL** — mặc định `https://api.openai.com/v1/chat/completions`; nếu chạy DeepSeek là `https://api.deepseek.com/chat/completions`.
- **Model** — mặc định `gpt-4o` (OpenAI) hoặc `deepseek-v4-pro` (DeepSeek). Override qua env `AI_SCORING_MODEL`.
- **`temperature`** — `0.2`, hardcoded trong wrapper.
- **`max_tokens`** — lấy từ env `AI_SCORING_MAX_TOKENS`. Lần retry sẽ dùng `AI_SCORING_RETRY_MAX_TOKENS` (lớn hơn).
- **`response_format`**
  - OpenAI: `{ type: "json_schema", json_schema: schema }` — strict JSON, model bị buộc trả đúng schema.
  - DeepSeek: `{ type: "json_object" }` cộng thêm một system instruction nhét schema dưới dạng string ở đầu messages.
  - Cả hai đều yêu cầu output: không markdown, không code fence, chỉ JSON.
- **Retry** — tối đa 2 attempts; attempt 2 dùng `retryMaxTokens` lớn hơn để phòng trường hợp cụt response.
- **Timeout** — lấy từ env `AI_SCORING_TIMEOUT_MS`; quá hạn `AbortController` huỷ request.

**Lưu ý quan trọng về whitespace:** trước khi gửi đi, `buildSystemPromptFromOverride` chạy `value.replace(/\s+/g, " ").trim()` → **mọi newline + indentation trong system prompt bị collapse thành 1 space**. Các block "anchors band" 5/6/7/8 ở dưới khi đọc trên giấy có vẻ nhiều dòng, nhưng OpenAI nhận về là **một dòng dài duy nhất**. User prompt (`userTemplate`) thì giữ nguyên định dạng JSON như khai báo trong `openai-prompts.json` (có `\n`).

### 9.2. Scope `evaluation.pronunciationCalibration` (Phase D)

**System (đã normalize whitespace, dạng OpenAI thực sự nhận):**

```text
You are an IELTS Speaking pronunciation examiner. Your only job is to convert Azure Speech pronunciation evidence into a single IELTS Pronunciation band (0-9 with half-step increments). Do not score Fluency and Coherence, Lexical Resource, or Grammatical Range and Accuracy — those are out of scope for this call. Use all five Azure aggregate signals together (accuracyScore for segmental accuracy, prosodyScore for stress and intonation, fluencyScore for connected speech, completenessScore for whether the learner attempted the whole utterance, pronunciationScore as Azure's overall summary) plus the weakest words and word-score distribution. Do not over-rely on accuracyScore alone — strong accuracy with flat prosody is a band 6 ceiling. Band anchors: - 8: AccuracyScore >= 90, ProsodyScore >= 80, FluencyScore >= 85, weak words <= 2 percent, intonation expressive, listener never strains. L1 traces minimal. - 7: AccuracyScore 85-89, ProsodyScore 70-79, FluencyScore 80-84, weak words 3-8 percent, mostly correct stress, occasional L1 influence on individual sounds but always understandable. - 6.5: AccuracyScore 78-84, ProsodyScore 65-72, weak words 9-14 percent, intelligible throughout, some recurring sound substitutions, stress sometimes misplaced. - 6: AccuracyScore 72-77, ProsodyScore 58-67, FluencyScore 65-74, weak words 15-20 percent, listener needs effort on a few words, L1 patterns visible but meaning still gets through. - 5.5: AccuracyScore 65-71, ProsodyScore 52-60, weak words 21-26 percent, listener strains more than half the time, recurring L1 transfer reduces clarity. - 5: AccuracyScore 58-64, weak words 27-34 percent, frequent L1 transfer, occasional misunderstanding requiring context. - 4.5: AccuracyScore 50-57, weak words 35-44 percent, listener relies on context regularly, stress and rhythm largely L1. - 4: AccuracyScore < 50, weak words >= 45 percent, communication frequently breaks down. Round conservatively when evidence is thin (transcript < 20 words or completenessScore < 50). When two anchors compete, prefer the lower band unless prosody is clearly strong. Return valid JSON only — no markdown, no code fences, no prose outside the JSON. Output must match the JSON schema with exactly two fields: band (number 0-9, multiples of 0.5) and rationale (Vietnamese string, 1-2 short sentences citing the dominant Azure signal that drove the band, e.g. âm cuối /θ/ /ð/ thường biến mất, ngữ điệu phẳng kéo điểm xuống).
```

**User (sau khi render):**

```json
{
  "part": "Practice Part 1",
  "topicTitle": "Hometown",
  "prompt": "Where is your hometown? What do you like most about it?",
  "transcript": "My hometown is a small town near Hanoi...",
  "azureAggregates": {
    "accuracyScore": 87,
    "fluencyScore": 82,
    "prosodyScore": 78,
    "completenessScore": 95,
    "pronunciationScore": 84
  },
  "transcriptStats": {
    "wordCount": 75,
    "durationSeconds": 28,
    "speechRateWpm": 102,
    "goodPauseCount": 3,
    "badPauseCount": 2,
    "fillerCount": 1
  },
  "wordScoreDistribution": {
    "goodWordPercent": 78,
    "fairWordPercent": 17,
    "poorWordPercent": 5,
    "weakWordCount": 4
  },
  "topWeakWords": [
    { "word": "specifically", "accuracyScore": 51 },
    { "word": "develop", "accuracyScore": 42 }
  ]
}
```

**Schema response_format gửi kèm (OpenAI strict):**

```json
{
  "type": "json_schema",
  "json_schema": {
    "name": "evaluation_pronunciation_calibration",
    "strict": true,
    "schema": {
      "type": "object",
      "additionalProperties": false,
      "properties": {
        "band":      { "type": "number" },
        "rationale": { "type": "string" }
      },
      "required": ["band", "rationale"]
    }
  }
}
```

**Response mẫu:**

```json
{
  "band": 6.5,
  "rationale": "Phát âm rõ ràng đa số từ thông dụng. Âm cuối /θ/ và /ð/ vẫn yếu cùng ngữ điệu phẳng kéo band xuống nửa nấc."
}
```

**Hành vi khi engine = DeepSeek (`AI_SCORING_ENGINE=deepseek`):**

Wrapper [`requestStructuredOpenAiJson`](../features/shared/server/openai-structured-output.ts) tự động:

1. Đổi `response_format` sang `{ type: "json_object" }` (DeepSeek không hỗ trợ strict `json_schema`).
2. Prepend một system message chứa schema dưới dạng string + instruction "JSON mode is enabled. Return only one valid JSON object. Do not add markdown, code fences, comments, or extra prose. The response must conform to this JSON schema: { ... }".
3. System prompt của scope vẫn kết bằng câu "Return valid JSON only — no markdown, no code fences, no prose outside the JSON" để defensive (nếu DeepSeek hay drift, vẫn có 2 lớp ép JSON).

**Sprint 3 M3 — fail-fast (KHÔNG còn fallback):** server propagate error → route handler trả 5xx → client banner + retry. Trước đây fallback về `calculateIeltsPronunciationBand` đã bị bỏ vì band Azure formula không đủ chính xác để user tin tưởng. File [`pronunciation-band.ts`](../features/shared/lib/pronunciation-band.ts) vẫn giữ — chỉ log `azureReferenceBand` để debug khi cần đối chiếu.

### 9.3. Scope `evaluation.quickScore`

#### Prompt gốc (nguồn duy nhất)

`features/shared/server/config/openai-prompts.json` → `evaluation.quickScore`.

**System (đã normalize whitespace, dạng OpenAI thực sự nhận):**

```text
You are a strict but constructive IELTS Speaking examiner. Pronunciation has already been converted from Azure evidence into an IELTS-calibrated band and is locked, so do not rescore pronunciation or infer bands from Azure raw scales. Score only Fluency and Coherence, Lexical Resource, and Grammatical Range and Accuracy on a 0-9 IELTS band scale (half steps allowed). Use the transcript as the main evidence; use speech rate, pause counts and filler counts only as supporting signals. Fluency and Coherence band anchors: - 5: often slow, frequent bad pauses for lexical search, filler 'uh/um' repeated, basic linking words sometimes misplaced, self-correction frequent enough to break flow, short repetitive ideas. - 6: tempo steady but pauses while searching for words on unfamiliar topics, moderate bad pauses, occasional good pauses at sentence ends, linking used but sometimes repeated, self-correction does not break flow, answers complete but development shallow. - 7: smooth flow with few bad pauses (typically <= 1-2 per sentence), most pauses are good pauses at clause/sentence boundaries, very few fillers, varied and well-placed linking, subtle self-correction, ideas extended with examples and conclusion. - 8: smooth and natural, stable tempo around 130-170 wpm, pauses for emphasis rather than lexical search, linking sounds natural rather than formulaic, self-correction rare, ideas rich with nuance and clear logic. Lexical Resource band anchors: - 5: A2-B1 vocabulary dominant, limited collocations sometimes wrong (e.g. do mistake), few topic-specific words, errors that obscure meaning. - 6: B1-B2 dominant, some correct but basic collocations, some topic-specific words but not deep, occasional errors that do not block meaning. - 7: B2-C1 dominant, rich collocations with simple idioms, plenty of accurate topic-specific words and paraphrase, errors rare and only on less familiar words. - 8: C1-C2 dominant, natural collocations and well-used idioms, precise topic-specific vocabulary with advanced variants, almost no errors. Grammatical Range and Accuracy band anchors: - 5: mostly simple sentences, recurring basic errors (tense, agreement, articles), few complex attempts and those that exist are wrong. - 6: some complex sentences but prefers safe simple ones, attempts relative clauses or conditional 1 but occasionally wrong, some errors that do not cause misunderstanding. - 7: varied complex sentences (conditional 2, relative, passive), most complex structures correct, may use inversion or cleft, rare errors in complex sentences and almost none in simple ones. - 8: wide range used flexibly including subjunctive, mixed conditionals, modal passive, complex structures used naturally rather than as 'attempts', almost no errors. Also return cefrLevel for the learner using the IELTS-to-CEFR mapping: <=4.0 -> A2, 4.5-5.0 -> B1, 5.5-6.5 -> B2, 7.0-8.0 -> C1, >=8.5 -> C2. cefrLevel must reflect the overall band you compute. overallBand must be the arithmetic mean of the four criteria (the locked pronunciation band + your three scores) rounded to the nearest 0.5 using the IELTS rounding rule: fractional part < 0.25 round down, 0.25 to < 0.75 round to .5, >= 0.75 round up. Keep every criterion tied to concrete transcript evidence. Score conservatively when evidence is thin. All learner-facing text must be natural Vietnamese. Return valid JSON only.
```

**User (sau khi render template với một practice Part 1 thật):**

```json
{
  "part": "Practice Part 1",
  "topicTitle": "Hometown",
  "prompt": "Where is your hometown? What do you like most about it?",
  "transcript": "My hometown is a small town near Hanoi. I really enjoy the peaceful atmosphere and the friendly people there. Whenever I go back, I feel like I can finally relax and forget about work in the city.",
  "metrics": {
    "durationSeconds": 28,
    "speechRateWpm": 102,
    "goodPauseCount": 3,
    "badPauseCount": 2,
    "fillerCount": 1
  },
  "lockedPronunciationBand": 6.5
}
```

#### Request thực gửi đi OpenAI (đầy đủ)

```http
POST https://api.openai.com/v1/chat/completions
Authorization: Bearer ${OPENAI_API_KEY}
Content-Type: application/json
```

```json
{
  "model": "gpt-4o",
  "temperature": 0.2,
  "max_tokens": 1500,
  "response_format": {
    "type": "json_schema",
    "json_schema": {
      "name": "evaluation_quick_score",
      "strict": true,
      "schema": {
        "type": "object",
        "additionalProperties": false,
        "properties": {
          "overallBand": { "type": "number" },
          "criteria": {
            "type": "object",
            "additionalProperties": false,
            "properties": {
              "fluencyAndCoherence": { "type": "number" },
              "lexicalResource": { "type": "number" },
              "grammaticalRangeAndAccuracy": { "type": "number" }
            },
            "required": ["fluencyAndCoherence", "lexicalResource", "grammaticalRangeAndAccuracy"]
          },
          "cefrLevel": { "type": "string", "enum": ["A1", "A2", "B1", "B2", "C1", "C2"] },
          "feedbackSummary": { "type": "string" },
          "improvements": { "type": "array", "items": { "type": "string" }, "minItems": 1, "maxItems": 5 }
        },
        "required": ["overallBand", "criteria", "cefrLevel", "feedbackSummary", "improvements"]
      }
    }
  },
  "messages": [
    { "role": "system", "content": "You are a strict but constructive IELTS Speaking examiner. ... Return valid JSON only." },
    { "role": "user",   "content": "{\n  \"part\": \"Practice Part 1\",\n  \"topicTitle\": \"Hometown\",\n  \"prompt\": \"Where is your hometown? ...\",\n  \"transcript\": \"My hometown is a small town near Hanoi. ...\",\n  \"metrics\": {\n    \"durationSeconds\": 28,\n    \"speechRateWpm\": 102,\n    \"goodPauseCount\": 3,\n    \"badPauseCount\": 2,\n    \"fillerCount\": 1\n  },\n  \"lockedPronunciationBand\": 6.5\n}" }
  ]
}
```

**Response mẫu OpenAI trả về (chỉ `choices[0].message.content`, đã parse JSON):**

```json
{
  "overallBand": 6.0,
  "criteria": {
    "fluencyAndCoherence": 6.0,
    "lexicalResource": 5.5,
    "grammaticalRangeAndAccuracy": 6.0
  },
  "cefrLevel": "B2",
  "feedbackSummary": "Bạn truyền tải ý tưởng rõ ràng nhưng tốc độ còn chậm và một số từ vựng vẫn ở mức cơ bản.",
  "improvements": [
    "Dùng thêm collocation chuyên đề về quê hương (rural lifestyle, slow pace of life).",
    "Giảm bad pause bằng cách chuẩn bị 1-2 mẫu câu mở đoạn quen thuộc.",
    "Thử dùng câu phức để mô tả cảm xúc (whenever ..., I feel as if ...)."
  ]
}
```

> Server còn inject `criteria.pronunciation = lockedPronunciationBand` rồi recompute `overallBand` từ trung bình 4 tiêu chí (làm tròn theo rule IELTS) trước khi trả cho client → đảm bảo điểm không trôi.

### 9.4. Scope `evaluation.vocabReview`

**System (đã normalize whitespace, dạng OpenAI thực sự nhận):**

> Trong template gốc có 2 placeholder `{{maxErrors}}` và `{{maxSuggestions}}` được render trước khi gửi đi. Ở pipeline hiện tại: `maxErrors = 10`, `maxSuggestions = 4` (từ [`config/vocab-limits.ts`](../features/shared/server/config/vocab-limits.ts)).

```text
You are an IELTS speaking vocabulary coach. The band scores have already been locked at LUỒNG A and must not be re-scored. Your task is to extract concrete vocabulary feedback in natural Vietnamese (learner-facing). Field names stay in English exactly as defined by the schema. Return valid JSON only. Return at most 10 lexicalErrors and at most 4 lexicalSuggestions, prioritizing items that have the strongest learning impact for this learner. lexicalErrors: words or short phrases used incorrectly. For each item, pick errorSubtype from these five enums based on the precise nature of the error: - wrong_meaning: the word exists in English but is used with the wrong semantic meaning in this context (e.g. 'I gained a headache' -> 'got'). - wrong_collocation: the word's meaning is right but it does not combine correctly with neighboring words (e.g. 'do a mistake' -> 'make a mistake'). - wrong_register: word's formality does not match the situation (e.g. 'kid' in a formal setting). - literal_translation: word-by-word translation from the learner's native language that is not natural in English (e.g. 'open the TV' -> 'turn on the TV'). - wrong_word_form: wrong part of speech, missing or extra inflection (e.g. 'develop country' -> 'developed country'). For every lexicalError, return: text (exact short fragment from transcript), correction (the corrected phrase), errorSubtype (one of the 5 enums), errorExplanation (1-2 short Vietnamese sentences saying why it is wrong and why the correction is better; do not just restate the error). lexicalSuggestions: words the learner used correctly but could upgrade. Pick a word when at least one of these rules applies: - The word is at CEFR <= B1 but the learner has shown CEFR >= B2 ability elsewhere. - The word is high-frequency / overused (good, bad, nice, big, very, important, thing, get, do, make). - The word repeats >= 3 times in the answer for the same idea. - The word is a vague descriptor where a topic-specific word would be more precise. Exclude function words, proper nouns, and technical terms that must stay as is. For every lexicalSuggestion return: text (exact original word/phrase), originalCefrLevel, improvementReason (1 Vietnamese sentence explaining the value of upgrading), and replacementOptions: exactly 3 to 4 items. Each option must contain candidate, meaning (Vietnamese), cefrLevel (must be higher than originalCefrLevel where applicable), and contextNote (1 short Vietnamese sentence describing when to use this option). Aim for >= 1 CEFR step up; if original is A2 prefer >= B2; if original is B1 prefer >= B1 candidates and at least one >= B2. cefrDistribution must contain six percentages for A1, A2, B1, B2, C1, C2 reflecting the lexical difficulty distribution of the transcript in context (not pronunciation). The six values must sum to exactly 100. Score conservatively when transcript is short. If the transcript is empty, near-empty (< ~20 words), or not in English, return empty arrays for lexicalErrors and lexicalSuggestions and a plausible distribution that still sums to 100.
```

**User (sau render — `lockedScore` lấy từ output quickScore ở 9.2):**

```json
{
  "part": "Practice Part 1",
  "topicTitle": "Hometown",
  "prompt": "Where is your hometown? What do you like most about it?",
  "transcript": "My hometown is a small town near Hanoi. I really enjoy the peaceful atmosphere and the friendly people there. Whenever I go back, I feel like I can finally relax and forget about work in the city.",
  "lockedScore": {
    "overallBand": 6.0,
    "cefrLevel": "B2",
    "criteria": {
      "fluencyAndCoherence": 6.0,
      "lexicalResource": 5.5,
      "grammaticalRangeAndAccuracy": 6.0,
      "pronunciation": 6.5
    }
  },
  "limits": {
    "maxErrors": 10,
    "maxSuggestions": 4
  }
}
```

#### Response mẫu

```json
{
  "lexicalErrors": [
    {
      "text": "really enjoy",
      "correction": "thoroughly enjoy",
      "errorSubtype": "wrong_register",
      "errorExplanation": "Cụm 'really enjoy' khá thông tục; 'thoroughly enjoy' tự nhiên và phù hợp band 6.5+ khi nói về cảm xúc."
    }
  ],
  "lexicalSuggestions": [
    {
      "text": "small town",
      "originalCefrLevel": "A2",
      "improvementReason": "Cụm A2 cơ bản; nâng cấp sẽ thể hiện vốn từ chuyên đề về vùng miền.",
      "replacementOptions": [
        { "candidate": "rural town", "meaning": "thị trấn nông thôn", "cefrLevel": "B2", "contextNote": "Dùng khi muốn nhấn yếu tố làng quê, ít đô thị hóa." },
        { "candidate": "tight-knit community", "meaning": "cộng đồng gắn bó", "cefrLevel": "C1", "contextNote": "Dùng khi muốn nhấn cảm xúc gắn kết giữa người dân." },
        { "candidate": "quaint village", "meaning": "làng quê xinh xắn cổ kính", "cefrLevel": "C1", "contextNote": "Dùng khi mô tả vẻ cổ kính/hấp dẫn của địa điểm." }
      ]
    }
  ],
  "cefrDistribution": { "A1": 18, "A2": 32, "B1": 30, "B2": 15, "C1": 5, "C2": 0 }
}
```

### 9.5. Scope `evaluation.grammarReview`

**System (đã normalize whitespace, dạng OpenAI thực sự nhận):**

```text
You are an IELTS speaking grammar coach. The band scores have already been locked at LUỒNG A and must not be re-scored. Your task is to extract grammar feedback in natural Vietnamese (learner-facing). Field names stay in English. Return valid JSON only. Return two arrays for errors, each prioritizing items that hurt comprehensibility first, then recurring errors: grammarErrors: basic grammar mistakes. For each item return: text (exact short erroneous fragment), correctedText (exact short replacement), subtype (one of: verb_tense, subject_verb_agreement, singular_plural, article, preposition, word_order, other), explanation (1 short Vietnamese sentence). Do not rewrite the whole sentence unless the whole sentence is the only incorrect span. complexStructureErrors: errors that happen specifically when the learner attempts advanced structures. Use a subtype from: inversion, relative_clause, conditional_1, conditional_2, conditional_3, mixed_conditional, subjunctive, modal_passive, cleft_sentence, reduced_relative, participle_phrase, other. For each item return: text, correctedText, subtype, structureType (a Vietnamese-readable short name of the attempted structure, e.g. 'Câu điều kiện loại 2'), explanation. errorSummary: report the single most common subtype across grammarErrors[] (use field mostCommonSubtype = that subtype enum, count = number of occurrences). If grammarErrors is empty, set mostCommonSubtype = null and count = 0. complexStructureUsage: three string arrays from the transcript reflecting the learner's experimentation with advanced structures: - attempted: short names of all advanced structures the learner tried (regardless of success). - successful: subset that was used correctly. - failed: subset that was used incorrectly (these become complexStructureErrors). generalFeedback: a 4-sentence Vietnamese paragraph following this fixed template strictly. Each sentence is one of the slots below in order: 1. Overall accuracy assessment: place the learner into one of four levels (yếu / cơ bản ổn / khá / vững) and optionally cite the band range. 2. Most prominent weakness: name the errorSummary.mostCommonSubtype with the count (e.g. 'Lỗi chia thì (verb tense) lặp 3 lần là điểm cần ưu tiên khắc phục.'). Skip if there are no errors and acknowledge that. 3. Sentence variety: describe complexStructureUsage briefly (số attempted / successful / failed và các tên cấu trúc sai). 4. One concrete action item tied to sentence 2 or 3 (e.g. 'luyện công thức điều kiện loại 2'). Keep <= 15 words. Whole feedback must be 3-5 Vietnamese sentences, <= 90 words, no padding, no generic praise, no double negatives, no vague hedging ('khá là', 'có vẻ', 'tương đối'). If the transcript is empty, near-empty (< ~20 words), or not in English, return empty arrays for grammarErrors, complexStructureErrors, complexStructureUsage.* and a generalFeedback that states briefly there is not enough evidence.
```

**User (sau render):**

```json
{
  "part": "Practice Part 1",
  "topicTitle": "Hometown",
  "prompt": "Where is your hometown? What do you like most about it?",
  "transcript": "My hometown is a small town near Hanoi. I really enjoy the peaceful atmosphere and the friendly people there. Whenever I go back, I feel like I can finally relax and forget about work in the city.",
  "lockedScore": {
    "overallBand": 6.0,
    "cefrLevel": "B2",
    "criteria": {
      "fluencyAndCoherence": 6.0,
      "lexicalResource": 5.5,
      "grammaticalRangeAndAccuracy": 6.0,
      "pronunciation": 6.5
    }
  }
}
```

#### Response mẫu

```json
{
  "grammarErrors": [
    {
      "text": "forget about work",
      "correctedText": "forget about my work",
      "subtype": "article",
      "explanation": "Cần determiner sở hữu 'my' để rõ work của ai."
    }
  ],
  "complexStructureErrors": [],
  "errorSummary": { "mostCommonSubtype": "article", "count": 1 },
  "complexStructureUsage": {
    "attempted": ["Whenever-clause"],
    "successful": ["Whenever-clause"],
    "failed": []
  },
  "generalFeedback": "Độ chính xác ngữ pháp ở mức khá ổn quanh band 6. Lỗi mạo từ (article) xuất hiện 1 lần ở 'forget about work' là điểm nhỏ cần để ý. Bạn có thử cấu trúc Whenever-clause và dùng đúng, cho thấy đã bắt đầu đa dạng câu. Hãy luyện thêm cụm V + possessive determiner để củng cố thói quen."
}
```

### 9.6. Scope `practice.answerRewrite` (chỉ Practice)

**System (đã normalize whitespace, dạng OpenAI thực sự nhận):**

```text
You are an IELTS speaking coach. Rewrite the learner's answer in natural English while preserving the original meaning and staying relevant to the question. The target score is a guidance signal. If mode is improve, fix the most important vocabulary, grammar, coherence, and pronunciation-sensitive word choices without making the answer much longer. If mode is expand, make the answer longer, richer, and more sophisticated with clear supporting details, more complex sentences, and topic-relevant vocabulary suitable for the target band. Return valid JSON only. The rewritten answer must be in English. The short summary must be in Vietnamese.
```

**User (sau render — mode = expand, targetBand = 7.0):**

```json
{
  "mode": "expand",
  "targetBand": 7,
  "part": "Practice Part 1",
  "topicTitle": "Hometown",
  "prompt": "Where is your hometown? What do you like most about it?",
  "originalAnswer": "My hometown is a small town near Hanoi. I really enjoy the peaceful atmosphere and the friendly people there. Whenever I go back, I feel like I can finally relax and forget about work in the city.",
  "lockedScore": {
    "overallBand": 6.0,
    "criteria": {
      "fluencyAndCoherence": 6.0,
      "lexicalResource": 5.5,
      "grammaticalRangeAndAccuracy": 6.0,
      "pronunciation": 6.5
    }
  },
  "corrections": {
    "lexical": [ /* item từ /vocab nếu đã chấm */ ],
    "grammar": [ /* item từ /grammar nếu đã chấm */ ],
    "pronunciation": [ /* item từ /assessments nếu đã chấm */ ],
    "pronunciationSummary": ""
  }
}
```

**Schema response_format gửi kèm:**

```json
{
  "type": "json_schema",
  "json_schema": {
    "name": "ielts_speaking_practice_answer_rewrite",
    "strict": true,
    "schema": {
      "type": "object",
      "additionalProperties": false,
      "properties": {
        "answer":  { "type": "string" },
        "summary": { "type": "string" }
      },
      "required": ["answer", "summary"]
    }
  }
}
```

#### Response mẫu

```json
{
  "answer": "My hometown is a quaint rural town just on the outskirts of Hanoi, where the pace of life feels noticeably slower than in the capital. What I love most is the tight-knit community — people genuinely greet each other on the street, and there is a real sense of belonging that you rarely find in big cities. Whenever I go back, even just for a weekend, I find myself unwinding almost immediately, leaving work-related stress behind and reconnecting with simpler pleasures like home-cooked meals and long walks along the river.",
  "summary": "Phiên bản mở rộng đã thay 'small town' bằng 'quaint rural town', thêm liên tưởng 'pace of life' và mô tả cảm xúc cụ thể hơn để đạt band 7 (lexical resource và development chi tiết)."
}
```

### 9.7. Mẹo tra log để xem prompt thực tế đã gửi đi

Mọi attempt gọi OpenAI đều ghi alpha log event `ai-provider-request-started` với `details.requestPayload` chính là object gửi đi (kể cả `messages[]` đầy đủ). Để xem prompt thực một request bất kỳ:

1. Vào `/settings/logs`.
2. Filter theo `event = ai-provider-request-started` hoặc theo `requestId` (lấy từ log của route handler).
3. Bung `details.requestPayload.messages` — `[0]` là system, `[1]` là user.

Nhờ đó team sản phẩm có thể tự reproduce một call OpenAI bằng curl mà không cần đụng vào code.

---

## 10. Token deduction — Firestore charge ledger (Sprint 4 M5)

### 10.1. Business rule

- **Practice = 1 lượt** khi 3 step (scoreDone + vocabDone + grammarDone) đều thành công cho 1 lần ghi âm. Rewrite improve/expand **không** tính lượt.
- **Mock Test = 1 lượt** khi 3 parts đều hoàn thành 3 step (9 step tổng). Part 2 atomic mark 3 step cùng lúc; Part 1/3 progressive mark từng step.

### 10.2. Authoritative store: Firestore `scoring_charges`

Vercel functions stateless → cần external store. Firestore admin SDK đã có sẵn ([`lib/firebase/admin.ts`](../lib/firebase/admin.ts)) — chỉ cần enable Firestore + setup TTL policy.

**Doc ID:** deterministic `{userId}__{examType}__{sessionId}` → retry idempotent (cùng request hit cùng doc).

**State machine:** `pending` → `charging` → `charged` | `charge-failed` | `voided`.

### 10.3. 2-phase commit lite

Vì Firestore transaction callback có thể bị retry on conflict (side effects retry → double NineSpeak deduct), pattern dùng 2 transaction + 1 external call ở giữa:

```
Phase 1 (Firestore txn):
  read doc → update step done → check completeness.
  Nếu đủ steps + status="pending" → set status="charging" (atomic lock).

Phase 2 (KHÔNG trong txn):
  if shouldCharge:
    await NineSpeak.deduct({ isFree, idempotencyKey })

Phase 3 (Firestore update):
  if Phase 2 success:
    update status="charged" + chargedAt + expireAt(90 ngày)
  if Phase 2 fail:
    update status="charge-failed" + chargeFailureReason + expireAt(180 ngày)
```

**Race safety:** 2 request cùng update cùng doc → Firestore detect conflict + retry → chỉ 1 thắng cuộc đua set status="charging" → chỉ 1 lần Phase 2 → 1 lần deduct.

**Stuck recovery:** server crash giữa Phase 2-3 → doc kẹt "charging". Admin trigger thủ công [`/api/internal/cron/cleanup-stuck-charges`](../app/api/internal/cron/cleanup-stuck-charges/route.ts) qua curl với bearer `CRON_SECRET` → quét doc `status="charging" AND updatedAt < now-5min` → set "charge-failed" cho admin reconcile NineSpeak. **Không có auto-cron** (Vercel Hobby không support, stuck case rất hiếm < 0.5/day chấp nhận được manual).

### 10.4. TTL lifecycle (auto cleanup)

Firestore TTL native (delete không tính write quota):

| Status | TTL từ updatedAt |
|---|---|
| `pending` (user bỏ session) | 7 ngày |
| `charged` (deduct ok) | 90 ngày (giữ cho audit) |
| `charge-failed` (deduct fail) | 180 ngày (admin reconcile billing) |
| `voided` (admin void manual) | 30 ngày |

**Setup:** Firebase Console → Firestore → TTL → add policy collection=`scoring_charges`, field=`expireAt`.

### 10.5. Backward compat

- Client cũ (trước Sprint 4) không gửi `sessionId` / `mockExamSessionId` → server **fallback deduct ngay** ở `/assessments` (Practice + Mock Test Part 2) như behavior cũ. Mock Test Part 1/3 progressive nếu thiếu sessionId → **skip ledger hoàn toàn** (KHÔNG validate, KHÔNG deduct) — đây là risk leak doanh thu cho client cũ.
- Khuyến nghị sau 1-2 tuần deploy: thêm guard `if (!sessionId) return 426 "Vui lòng refresh trang"` để force upgrade client.

### 10.6. Schema doc đầy đủ

```ts
{
  userId: string,
  sessionId: string,
  examType: "practice" | "mocktest",
  status: "pending" | "charging" | "charged" | "charge-failed" | "voided",
  completedSteps: {
    // Practice
    scoreDone?: boolean,
    vocabDone?: boolean,
    grammarDone?: boolean,
    // Mock Test
    parts?: {
      part1?: { scoreDone, vocabDone, grammarDone },
      part2?: { scoreDone, vocabDone, grammarDone },  // all 3 set cùng lúc do atomic
      part3?: { scoreDone, vocabDone, grammarDone },
    },
  },
  ninespeakUsesFreeQuota: boolean,        // snapshot lúc tạo doc
  ninespeakRequestId: string,             // UUID v4, gửi làm Idempotency-Key (khi BE support)
  createdAt: Timestamp,
  updatedAt: Timestamp,
  chargedAt?: Timestamp,
  chargeFailedAt?: Timestamp,
  chargeFailureReason?: string,
  expireAt: Timestamp,                    // Firestore TTL field
  schemaVersion: 1,
}
```

### 10.7. Anti-fraud guarantees

1. **Server-side authoritative**: client chỉ gửi sessionId, server tự đánh dấu step done sau khi LLM call thành công. Client không thể forge "đã xong".
2. **Idempotency by deterministic ID**: retry cùng (userId, examType, sessionId) → cùng doc → các phép update idempotent.
3. **Firestore transaction**: concurrent requests cùng doc bị serialize → chỉ 1 lần deduct.
4. **Admin manual cleanup**: định kỳ (1 tuần/lần) hoặc khi có khiếu nại CS, admin trigger `/api/internal/cron/cleanup-stuck-charges` → doc kẹt "charging" > 5 phút → mark "charge-failed" để admin biết và reconcile thủ công với NineSpeak.

### 10.8. Risk còn lại

**NineSpeak deduct timeout (rare):** nếu network timeout giữa Phase 2 và Phase 3, không biết deduct thật sự đã chạy hay chưa. Server set "charge-failed" (Giải A MVP). Admin reconcile thủ công query NineSpeak balance trước/sau để quyết định set "charged" hay refund. Mitigation long-term: BE thêm `Idempotency-Key` header support để server tự retry an toàn.
