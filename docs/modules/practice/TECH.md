# Practice Module — TECH (Đặc tả kỹ thuật)

> Đặc tả kỹ thuật cho **module Practice** (`features/practice/`). Đối tượng: AI agent + đầu mối phụ trách, để phát triển/sửa lỗi với nhận thức rõ ràng về ràng buộc liên-module (đặc biệt phụ thuộc `features/shared/` và `features/billing/`).
>
> Cặp tài liệu: xem [PRD.md](./PRD.md) cho nghiệp vụ. Nền tảng: [README-modules](../../README-modules.md), [AGENTS.md module](../../../features/practice/AGENTS.md), [scoring-pipeline-overview](../../scoring-pipeline-overview.md), [env-and-modules](../../env-and-modules.md).
>
> 🧭 **Thiết kế lại luồng chấm (Deepgram-only + DeepSeek + Azure ngầm)**: lý do + số liệu đo + quy tắc tách module ở [scoring-redesign-decision.md](./scoring-redesign-decision.md). Đọc TRƯỚC khi sửa luồng chấm practice.

---

## 1. Kiến trúc & sơ đồ thư mục

Module tự chứa UI + hooks + client lib + server evaluator + ledger, nhưng **đứng trên lõi `shared/`** cho pipeline chấm, recorder, content store, report kit và contract types.

```
features/practice/
├── index.ts                  # PUBLIC API barrel cho peer (helper isomorphic)
├── pages/                    # PracticeIndex, PracticePart1/2/3 (client pages)
├── layouts/                  # PracticeFocusShell + PracticeFocusHeaderContext
├── components/
│   ├── practice/             # shell câu hỏi, panel evaluation, paywall overlay, timeline + bộ Practice Optimize:
│   │                         #   PracticeCorrectedAnswer (track-changes), PracticeEdgeCaseCard,
│   │                         #   PracticeCriterionCard + PracticeCriterionPanels (4 panel riêng) + PracticeFixPopover, PracticeBandTrend,
│   │                         #   Practice{Pronunciation,Fluency,Vocab,Grammar}Insight,
│   │                         #   PracticeCoachAnswer, PracticeWordDrill
│   └── practice-part3/       # PracticeConsole, SupportRail, TopicOverview, TopicSidebar (+ index.ts)
├── hooks/                    # useSpeakingPractice (core), usePracticeAttemptHistory,
│                             #   usePracticeQuotaPurchaseFlow, usePracticeRecorderControls
├── providers/                # PracticeSessionStore (kho attempts khóa theo sessionKey, mount ở gốc app)
├── lib/                      # practice-client, practice-evaluation, practice-persistence,
│                             #   practice-report-adapter, practice-response, practice-retry,
│                             #   practice-topic-runtime, practice-assistant
├── server/                   # evaluate-practice, practice-route-handlers, practice-charge-ledger
├── data/                     # practice-catalog (static fallback), practice-part3 (UI meta)
└── types/                    # practice-evaluation.ts = re-export shim của shared
```

Luồng tổng:

```
app/(focus-practice)/practice/part-{1,2,3}  ──►  PracticePart{1,2,3} (pages)
                                                       │ useSpeakingPractice (core hook)
                                                       ▼
            lib/practice-client  ──HTTP──►  app/api/app/speaking/practice/{assessments,vocab,grammar,rewrites,word-cefr,word-drill}
                                                       │ server/practice-route-handlers
                                                       ▼
                          server/evaluate-practice ──► shared/server/evaluation-pipeline (Azure + 4 LLM step)   [baseline]
                          server/evaluate-practice-deepgram ──► practice-deepgram/calibration/quickscore       [PRACTICE_PIPELINE=deepgram]
                          server/practice-pronunciation ──► evaluateAzureWithReference (Phát âm: chạy NGẦM sau band)
                                                       │ server/practice-charge-ledger ──► Firestore practice_scoring_charges
                                                       ▼
                          persist ──► shared/lib (app-bff-client / company-api-client) ──► NineSpeak
```

---

## 2. Public API barrel — [features/practice/index.ts](../../../features/practice/index.ts)

Bề mặt duy nhất cho **module peer** (peer chỉ import từ `@/features/practice`). Hiện chỉ khai báo trước các helper isomorphic (chưa peer nào tiêu thụ); **không gom page/route-handler** (tránh phình bundle + rủi ro server-only):

```ts
export {
  roundBandScore, practicePartLabels, practiceCriteriaMeta,
  normalizeCriteriaScores, formatDurationLabel,
  countTranscriptWords, calculateSpeechRateWpm,
} from "@/features/practice/lib/practice-evaluation";
```

> `app/` (host) được phép import trực tiếp page/layout/route-handler của module — quan hệ host↔module, không phải import chéo peer.

---

## 3. Routes (app/)

### 3.1 Pages — route group `(focus-practice)` (chế độ tập trung)
| Route | File | Page component |
|---|---|---|
| `/practice` | [page.tsx](<../../../app/(focus-practice)/practice/page.tsx>) | `PracticeIndex` |
| `/practice/part-1` | [page.tsx](<../../../app/(focus-practice)/practice/part-1/page.tsx>) | `PracticePart1` (nhận prop `topUpPacks`) |
| `/practice/part-2` | [page.tsx](<../../../app/(focus-practice)/practice/part-2/page.tsx>) | `PracticePart2` |
| `/practice/part-3` | [page.tsx](<../../../app/(focus-practice)/practice/part-3/page.tsx>) | `PracticePart3` |
| layout | [layout.tsx](<../../../app/(focus-practice)/layout.tsx>) | `PracticeFocusShell` |

`topUpPacks` được inject từ server (`getResolvedPracticeTopUpPacks()` của shared) vào page Part 1/2/3.

### 3.2 API route handlers (BFF)
Tất cả là wrapper mỏng (`runtime = "nodejs"`, `dynamic = "force-dynamic"`) trỏ về handler trong [server/practice-route-handlers.ts](../../../features/practice/server/practice-route-handlers.ts):

| Endpoint | Handler | Chấm | Ledger | Lượt |
|---|---|---|---|---|
| `POST /api/app/speaking/practice/assessments` | `handlePracticeAssessmentsPost` | **Gated** `PRACTICE_PIPELINE`: baseline=Azure+calibration+quickScore (`evaluatePracticeAudio`); `deepgram`=Deepgram+DeepSeek (`evaluatePracticeAudioDeepgram`, KHÔNG phát âm) | ✅ `scoreDone` | **Trừ tại đây** (cả 2 nhánh) |
| `POST /api/app/speaking/practice/pronunciation` | `handlePracticePronunciationPost` | **MỚI** — Azure scripted detail-only (chạy ngầm sau band hoặc khi bấm tab Phát âm; `audioKey` lượt mới / `audioUrl` R2 cho history) → ghi `pronunciationSentences` (map RIÊNG) + `mispronouncedWords`, **KHÔNG đụng** `transcriptSentences`; KHÔNG đổi band | ❌ | miễn phí |
| `POST /api/app/speaking/practice/vocab` | `handlePracticeVocabPost` | `runVocabReview` (nhận lockedScore) | ❌ | miễn phí |
| `POST /api/app/speaking/practice/grammar` | `handlePracticeGrammarPost` | `runGrammarReview` (shared, danh sách lỗi) → `generatePracticeGrammarAnalysis` (RIÊNG, **GỘP 1 call**: tách câu + nhãn lỗi/độ phức `grammarSentenceBreakdown` + gợi ý cấu trúc) | ❌ | miễn phí |
| `POST /api/app/speaking/practice/fluency` | `handlePracticeFluencyPost` | `generatePracticeFluencyReview` (RIÊNG — mạch lạc + liên từ + gợi ý) | ❌ | miễn phí |
| `POST /api/app/speaking/practice/rewrites` | `handlePracticeRewritesPost` | `generatePracticeAnswerRewrite` | ❌ | miễn phí |
| `POST /api/app/speaking/practice/word-cefr` | `handlePracticeWordCefrPost` | `runWordCefrAnalysis` (background) | ❌ | fail-silent (trả `{}`) |
| `POST /api/app/speaking/practice/word-drill` | `handlePracticeWordDrillPost` | `assessPracticeWordDrill` (Azure 1 từ) | ❌ | miễn phí (drill phát âm) |
| `GET /api/app/speaking/practice/catalog` · `/topics` · `/topics/[topicId]` | handler của **onboarding-dashboard** (`app-data-route-handlers`) | content | — | — |

Route legacy còn giữ (backward compat): `POST /api/speaking/practice/evaluate` → `handlePracticeAssessmentsPost`; `POST /api/speaking/practice/rewrite` → `handlePracticeRewritesPost`. Cũng có alias `GET /api/app/practice/{catalog,topics,topics/[topicId]}` trỏ về handler onboarding-dashboard.

> **Lưu ý ranh giới**: catalog/topics là dữ liệu nội dung do **onboarding-dashboard** (BFF chung) phục vụ, không phải server của Practice — Practice chỉ tiêu thụ client-side qua app-bff-client của shared.

---

## 4. Data model / types

### 4.1 Contract types — sống ở shared
`features/practice/types/practice-evaluation.ts` là **re-export shim**:
```ts
export * from "@/features/shared/types/practice-evaluation";
```
Định nghĩa thật ở [shared/types/practice-evaluation.ts](../../../features/shared/types/practice-evaluation.ts). Các type chính:
- `PracticePartSlug` (`part-1|2|3`), `PracticePartKey` (`Part 1|2|3`)
- `IELTSCriteriaScores`, `PracticeEvaluationResult` (object trung tâm, gồm transcript, band, criteria, corrections, annotations, các `*Status`...). Field phát âm (luồng Deepgram):
  - `transcriptSentences` — transcript "chính": mang `cefrLevel` (/word-cefr, cho tab Từ vựng) + `pauseDurationMs` (Deepgram, cho tab Trôi chảy). **Phát âm KHÔNG ghi vào đây.**
  - `pronunciationSentences?` — **map RIÊNG** cho tab Phát âm (word-level `pronunciationScore` Azure). Đọc qua `resolvePronunciationSentences()` = `pronunciationSentences ?? transcriptSentences` (bản cũ/baseline fallback → tương thích ngược).
  - `pronunciationReviewStatus?` (`idle|pending|completed|failed`) + `pronunciationReviewError?` — trạng thái chấm phát âm ngầm/lazy (persist + rehydrate).
  - `_debugTimings?: Record<string, number>` — **client-only, KHÔNG persist**; chỉ gắn ở staging cho popup đo thời gian (bấm đúp điểm overall).
- `PracticeCorrections` (lexical/grammar/pronunciation), `PracticeGeneratedResponse`
- `PracticeRewriteMode` (`improve|expand|vocab|grammar|depth`); `rewriteStatus?: Partial<Record<PracticeRewriteMode, ...>>` (computed-key per mode); `PracticeRewriteChange` / `PracticeRewritePhrase` (coach answer: changes + cụm đáng học + translation)
- `PracticeWordDrillResult` (`pronunciationScore` + `phonemes: MockExamPronunciationPhoneme[]`) — kết quả drill 1 từ
- `PracticeAttempt` / `PracticeAttemptStatus` (dùng chung vì history-client của shared cần biết shape)
- Payload types: `EvaluatePracticeAudioPayload`, `EvaluatePracticeFeedbackPayload`, `EvaluatePracticeRewritePayload`
- Nhiều type tham chiếu `MockExam*` từ `shared/types/mock-exam` (transcript sentence/word, annotation, mispronounced word...).

> **Practice Optimize — pause 3 mức (§6.2):** panel Trôi chảy tính mức ngắt nghỉ (short/medium/long, ngưỡng PRD #11: ≲0.5s / ≤1.0s / >1.0s) **trực tiếp từ `pauseDurationMs`** (đã có sẵn trên `MockExamTranscriptWord`) ngay trong `PracticeFluencyInsight` — **không thêm field/scoring vào `shared`**, không đụng Mock Test.

`usePracticeAttemptHistory` và `practice-catalog`/`practice-topic-runtime` cũng **re-export** `PracticeAttempt(Status)` / `PracticePartKey` từ shared để giữ tương thích import qua đường dẫn module.

### 4.2 Data nội bộ module
- [data/practice-catalog.ts](../../../features/practice/data/practice-catalog.ts): type catalog (`PracticeTopic`, `PracticeQuestion`...) + bộ dữ liệu tĩnh mẫu (`practiceTopicsData`, `practicePartTabs`). Catalog runtime thật lấy từ NineSpeak; bộ tĩnh là cấu trúc/fallback.
- [data/practice-part3.ts](../../../features/practice/data/practice-part3.ts): UI meta cho Part 3 (wave heights, review scores, checklist, quick frames, topic meta).

---

## 5. Server

### 5.1 [server/evaluate-practice.ts](../../../features/practice/server/evaluate-practice.ts) (`"server-only"`)
- `evaluatePracticeAudio(...)` — pipeline chính:
  1. `evaluateSpeechAssessment(audioFile)` (Azure, shared) → phrases → transcript + word-level scores + aggregates.
  2. Build transcript sentences/words, weak words, pause stats, speech rate (helper cục bộ + shared pause classifiers).
  3. `runPronunciationCalibration(...)` (shared) → band phát âm. **Fail → throw** (fail-fast; code thực tế re-throw dù chú thích còn nói "fallback").
  4. `runQuickScore(...)` (shared) với `pronunciationBand` đã khoá → `lockedScore` (overallBand, criteria, cefr, summary, improvements).
  5. Trả `PracticeEvaluationResult` đầy đủ (corrections/annotations để rỗng — sẽ do /vocab,/grammar điền).
- `generatePracticeAnswerRewrite(...)` — gọi `requestStructuredOpenAiJson` với scope prompt **`practice.answerRewrite`** (prompt RIÊNG của Practice, mock-test không có; định tuyến theo 3 goal), trả coach answer `{answer, summary, changes, phrasesToLearn, translation}` theo `mode` (`PracticeRewriteMode`) + `targetBand`.
- `assessPracticeWordDrill(audioFile)` — gọi `evaluateSpeechAssessment` (Azure unscripted, shared, READ-only) cho **1 từ**, trả `PracticeWordDrillResult` (điểm phát âm + phoneme). Không qua calibration/quickScore, không ledger.

### 5.1b Luồng chấm MỚI Deepgram-only (practice-owned, gated `PRACTICE_PIPELINE=deepgram`)
Viết RIÊNG trong `features/practice/server/`, **KHÔNG đụng** `shared/server/evaluation-pipeline.ts` (mock-test dùng chung). Chỉ tái dùng *primitive* shared ổn định: `requestStructuredOpenAiJson`, `resolveAiScoringConfig`, `evaluateAzureWithReference`/`splitWavIntoSegments`, R2, alpha-logger. Lý do/số liệu: [scoring-redesign-decision.md](./scoring-redesign-decision.md).
- [practice-deepgram.ts](../../../features/practice/server/practice-deepgram.ts) — Deepgram REST: `transcribeFromUrl` (production, Deepgram tự kéo từ presigned GET R2 → function không chạm bytes) + `transcribeFromBytes` (bench). Trả transcript + word timestamp + confidence.
- [practice-calibration.ts](../../../features/practice/server/practice-calibration.ts) — `runPracticeCalibration`: tính pause từ timestamp Deepgram + DeepSeek calibrate band phát âm (prompt riêng [practice-scoring-prompts.ts](../../../features/practice/server/practice-scoring-prompts.ts)). Trả `{ band, rationale, speechRateWpm, pauseStats }`.
- [practice-quickscore.ts](../../../features/practice/server/practice-quickscore.ts) — `runPracticeQuickScore`: LLM chấm 3 tiêu chí; overall/cefr tính lại server; KHÔNG feedback/improvements.
- [evaluate-practice-deepgram.ts](../../../features/practice/server/evaluate-practice-deepgram.ts) — `evaluatePracticeAudioDeepgram`: Deepgram → calibrate → quickScore → `PracticeEvaluationResult` (mispronouncedWords=[], word score=0, `pronunciationReviewStatus="idle"`; feedbackContext có good/bad pause + filler cho /fluency). alpha-log từng phase.
- [practice-pronunciation.ts](../../../features/practice/server/practice-pronunciation.ts) — `evaluatePracticePronunciation`: Azure scripted (reference=transcript) từ audioKey **hoặc** audioUrl (history) → word-level + phoneme + từ yếu; tái dùng builder export từ `evaluate-practice.ts`.
- Bench hiệu chỉnh calibrate: [practice-calibration-bench.ts](../../../features/practice/server/practice-calibration-bench.ts) + `/debug/practice-calibration-bench` (gate `PRACTICE_PIPELINE_DEBUG_ENABLED`).

### 5.2 [server/practice-route-handlers.ts](../../../features/practice/server/practice-route-handlers.ts)
Mỗi handler: verify Firebase token → rate-limit (`assertRateLimit`) → parse input (presigned JSON `audioKey` hoặc multipart FormData) → `resolvePracticePrompt` (shared) → chạy evaluator → log alpha → trả JSON. Xử lý lỗi phân nhánh: auth → 401/500, request-guard (gồm 402 `grading_quota_exceeded`) → status tương ứng, prompt-override invalid → 400, AI error → `resolveAiUserFacingError`, còn lại → 500.
- `/assessments`: thêm bước `validateNineSpeakGradingOrThrow` (kiểm quota trước) và gọi `chargeIfReady` (ledger) sau khi có điểm; ledger fail **không re-throw**. **Gate `PRACTICE_PIPELINE`** chỉ **swap evaluator** (`evaluatePracticeAudioDeepgram` vs `evaluatePracticeAudio`) — quota/ledger/log/no_speech/persist **bao quanh cả 2 nhánh** (không bị phá). Ở deepgram+presigned, handler **KHÔNG download bytes** (truyền `audioKey` → Deepgram `transcribeUrl` tự kéo R2 → tiết kiệm băng thông Vercel).
- `/pronunciation` (MỚI, Azure detail-only — chạy ngầm sau band hoặc lazy khi mở tab): verify token + rate-limit → nhận `{ evaluation, audioKey?, audioUrl? }` → `evaluatePracticePronunciation` (Azure scripted, reference=transcript; audioKey download R2 hoặc audioUrl fetch trực tiếp cho history) → ghi vào **map RIÊNG `pronunciationSentences`** + `mispronouncedWords` (KHÔNG đụng `transcriptSentences`), set `pronunciationReviewStatus="completed"`. **KHÔNG** quota/ledger/đổi band.
- `/vocab`, `/grammar`: nhận `evaluation` từ client → tái dựng `lockedScore` (`lockedScoreFromEvaluation`) → `runVocabReview`/`runGrammarReview` → merge corrections + annotations (giữ phần đối lập để 2 response parallel không đè nhau). **Không** chạm ledger.
- `/word-cefr`: background, log `warn`, lỗi vẫn trả `{ wordCefrLevels: {} }`.
- `/assessments` (B2): khi Azure trả transcript rỗng/không có lời → trả **HTTP 422 code `no_speech`** (client nhận diện qua `isPracticeNoSpeechError`, không trừ lượt).
- `handlePracticeWordDrillPost`: verify token + rate-limit + parse audio → `assessPracticeWordDrill` → JSON. **Không** quota/ledger.

### 5.3 [server/practice-charge-ledger.ts](../../../features/practice/server/practice-charge-ledger.ts) (`"server-only"`)
- Collection **`practice_scoring_charges`**, doc id = `sessionId`. Schema v1, TTL theo trạng thái (pending 7d / charged 90d / failed 180d / voided 30d).
- `chargeIfReady(...)`: 2-phase commit lite. Phase 1 (Firestore transaction): set step `scoreDone`, nếu complete & `pending` → khoá `charging`. Phase 2: gọi `deductOrThrow` (NineSpeak deduct với idempotencyKey = `ninespeakRequestId`) → set `charged` / `charge-failed` / `skipped-no-token`. Idempotent + race-safe.
- State: `pending → charging → charged | charge-failed | skipped-no-token` (+ `voided` admin).

---

## 6. State (client)

### 6.1 [hooks/useSpeakingPractice.ts](../../../features/practice/hooks/useSpeakingPractice.ts) — hook trái tim (~1000 dòng)
- Quản lý recorder (`useMockExamRecorder` shared), status, evaluation (qua `evaluationRef` + `setEvaluation` để merge atomic ngoài setState).
- `runEvaluation`: upload presigned → `/assessments` → set band → **fire `/vocab` + `/grammar` parallel** (`Promise.allSettled`) + `/word-cefr` **và `/fluency`** fire-and-forget RIÊNG (ngoài gate, không trễ nút "Cải thiện câu"). Khi cả vocab+grammar `completed` → set `correctionsStatus="completed"` + `detailStatus="completed"`, gọi `onEvaluation(isFinal:true)`.
- **Trôi chảy & Ngữ pháp dùng LLM (không heuristic/hardcode)**: `/fluency` trả `fluencyCoherenceLevel|Note|LinkersUsed|LinkerSuggestions` (thay `detectConnectors`/`coherenceSummary`); `/grammar` trả thêm `grammarSentenceBreakdown[]` (tách câu + `hasError` + `isComplex` từ LLM) — client đếm % chính xác/% câu phức + vạch màu TỪ list này (thay Azure-phrase + regex + substring). Bài cũ chưa có field → UI **fallback heuristic** để không vỡ history. Cả hai là **prompt RIÊNG practice** (`practice.fluencyReview`, `practice.grammarStructureSuggestions` — call này nay GỘP tách câu + gợi ý cấu trúc), KHÔNG đụng `runGrammarReview`/shared dùng chung với mock-test.
- **Phát âm chạy NGẦM (luồng deepgram)**: `requestPronunciationReview(evaluation, requestId?, audioUrl?)`. Lượt vừa ghi: `runEvaluation` tự gọi NGẦM (qua `requestPronunciationReviewRef`) **ngay sau khi 4 phân tích DeepSeek (vocab/grammar/fluency/word-cefr) settle** — (base đủ dữ liệu trước khi phát âm chạy). **Phát âm ghi vào MAP RIÊNG `pronunciationSentences`** (Azure word-level pronunciationScore), **KHÔNG đụng `transcriptSentences`** (mang `cefrLevel` của /word-cefr + `pauseDurationMs`) → tab Từ vựng/Trôi chảy KHÔNG bao giờ bị ghi đè (chart "cấp độ từ vựng" không biến mất). Tab Phát âm + report đọc qua `resolvePronunciationSentences()` = `pronunciationSentences ?? transcriptSentences` (bản cũ/baseline không có map riêng → fallback, tương thích ngược). Per-word pronunciation coloring ở report dùng **annotations từ `mispronouncedWords`** (text-match), không phụ thuộc map. Đẩy `pronunciationReviewStatus="pending"` vào store ngay → mở tab Phát âm hiện **loading TOÀN PHẦN** (`PracticeCriterionCard`), không chỉ 1 dòng. Khi xong đẩy kết quả qua `onEvaluation`. Mở tab khi vẫn `idle`/`failed` (lượt history hoặc eager lỗi) thì click vẫn fire lazy như cũ (`handleOpenCriterion`, dùng `audioUrl` R2). Lấy `audioKey` theo `requestId` từ `audioKeyByRequestRef`. `pronunciationReviewStatus` được **persist + rehydrate** (practice-persistence + history-client). Analytics `PronunciationAnalysisLoadedEvent`.
- Cơ chế an toàn: `requestId` (=sessionId) chống stale (`latestRequestIdRef`), **ref-based mutex** chống double-fire, **sequence counter** per-tab chống race khi spam retry, **debounce 350ms** cho persist.
- Bắt 402 → tạo `pending` evaluation + persist pending (không throw).
- **Edge case (Practice Optimize)**: state `edgeCase` (`mic_denied|no_device|too_short|no_speech`) + `classifyRecorderStartError` (phân loại lỗi mở mic) → render `PracticeEdgeCaseCard`; B2 no_speech bắt từ 422; **B4 auto-stop** effect dừng ghi khi chạm `maxRecordingSeconds` (truyền từ page theo Part).
- `rewriteStatus` dùng **computed-key** `[mode]` cho 5 mode (`PracticeRewriteMode`); `runRewrite`/`requestAnswerRewrite` nhận `mode`.
- Public API: `startRecording/stopRecording`, `retrySavedAttempt`, `requestDetailedEvaluation`, `requestAnswerRewrite`, `retryVocabReview`, `retryGrammarReview`, `reset`, `beforeunload` guard.
- **[staging-only] đo thời gian xử lý**: `timeApi(key, promise)` bọc từng call (upload, assessments, word-cefr, fluency, vocab, grammar, pronunciation, rewrite_*) ghi ms vào `debugTimingsRef`; `recordEndToBandMs` = tổng recordEnd→hiện điểm. `syncEvaluation` đính `_debugTimings` (field client-only, **KHÔNG persist**) vào evaluation **chỉ khi** `isStagingEnvironment()`. Panel: bấm đúp điểm overall → popup `DebugTimingsPopup`. Prod không gắn gì (gate `stagingDebug`).

### 6.2 Hooks khác
- [usePracticeAttemptHistory](../../../features/practice/hooks/usePracticeAttemptHistory.ts): **selector mỏng** trên `PracticeSessionStore` (xem 6.4). Chữ ký `usePracticeAttemptHistory(sessionKey, status)` — đọc/ghi slice attempts theo `sessionKey`; giữ nguyên shape trả về (`attempts`, `latestAttempt`, `recorderState`, `register/update/hydrate/clearAttempts`), cờ `isImprovement` (so band lượt trước).
- [usePracticeQuotaPurchaseFlow](../../../features/practice/hooks/usePracticeQuotaPurchaseFlow.ts): state machine paywall (`idle→offer→purchase-status/retrying/still-blocked`). Subscription dùng `usePaymentOrderFlow` (Billing barrel) + auto retry sau paid; top-up dùng Messenger checkout.
- [usePracticeRecorderControls](../../../features/practice/hooks/usePracticeRecorderControls.ts): wrapper điều khiển recorder cho page. `recordActionLabel` theo state: `recording`→"Nhấn để dừng", `processing`→"Đang chấm", có ≥1 lượt→"Ghi âm lại", còn lại→"Bắt đầu ghi".
- **Timer đếm giờ khi ghi**: `useSpeakingPractice` trả `recordingDurationSeconds` (realtime từ `useMockExamRecorder`, ~250ms). Ba page truyền xuống `PracticeQuestionShell` qua prop `recordingSeconds`; shell hiển thị đồng hồ `M:SS` phía trên nhãn khi `isRecording` — **không** hiện giới hạn thời gian (thực tế không giới hạn).

### 6.4 [providers/PracticeSessionStore.tsx](../../../features/practice/providers/PracticeSessionStore.tsx) — kho phiên giữ qua điều hướng
- Mount ở **gốc app** ([app/providers.tsx](../../../app/providers.tsx)), đứng ngoài mọi route group → KHÔNG unmount khi điều hướng rời màn luyện. Nhờ đó attempts + audio (blob objectURL) + trạng thái `pending` được giữ khi user sang `/pricing` (CTA "Khám phá lợi ích PRO" trong paywall) rồi back lại; luồng auto chấm-lại sau khi mua chạy bình thường.
- State khóa theo **`sessionKey`** = `buildPracticeSessionKey({ part, topicId, questionId, historySessionId })` → `part:topic:question:(live | history:<id>)`. Tách hẳn luồng **luyện trực tiếp** (scope `live`) khỏi **xem lại từ Lịch sử** (scope `history:<sessionId>`) nên không lẫn dữ liệu; đổi câu hỏi = đổi key = slice rỗng (thay cho `clearAttempts` cưỡng bức trước đây).
- **FIFO evict**: giữ tối đa `MAX_SESSIONS = 3` phiên gần nhất (theo thứ tự tạo), revoke blob của phiên bị loại. Blob lifecycle do store quản lý (revoke khi `clearAttempts`/evict/unmount), **không** revoke theo unmount của page — đây là lý do trước đây back lại bị mất.
- Trang Part tự **mở lại paywall** khi back: nếu lượt mới nhất của key `live` đang `pending` và chưa mở offer trong lần mount này → gọi `openOfferForAttempt`. Bỏ qua khi đang ở luồng xem lại Lịch sử (`routeSessionId`) — vì vậy lượt `pending` mở từ Lịch sử KHÔNG tự bung paywall mà rơi vào overlay mặc định của timeline (card "đang chờ chấm" + nút "Chấm lại" → `onRetryPending` mới bung paywall).
- **UI màn pending**: overlay mặc định ([PracticeAttemptTimeline.tsx](../../../features/practice/components/practice/PracticeAttemptTimeline.tsx)) chỉ in câu tiếng Việt cố định, **không** render `attempt.pendingReason` (reason thô tiếng Anh từ backend). `PracticeQuestionShell` có prop `showRecordButton` (mặc định `true`); cả 3 Part truyền `showRecordButton={!(routeSessionId && attempts.some(a => a.status === "pending"))}` — chỉ ẩn nút ghi âm ở màn pending mở **từ Lịch sử**. Live mode vẫn giữ nút (nút mic = "Ghi âm lại" khi đã có lượt), lượt đã chấm xem từ Lịch sử cũng giữ. Shell còn nhận prop `recordingSeconds` để hiển thị đồng hồ đếm giờ khi đang ghi (xem 6.3).
- **Giới hạn đã biết (Tầng 1)**: chỉ bền theo vòng đời tab (chưa persist server → refresh mất); `activeQuestionIndex` vẫn reset về 0 khi remount nên với Part nhiều câu, attempt pending ở câu sâu được giữ trong store nhưng chỉ hiện lại khi quay đúng câu đó.

### 6.3 lib (client)
- [practice-client.ts](../../../features/practice/lib/practice-client.ts): fetch tới các endpoint; `PracticeRequestError` + `isPracticeQuotaExceededError` (402+`grading_quota_exceeded`) + `isPracticeNoSpeechError` (422+`no_speech`); presigned vs inline; gắn token + alpha-log headers; `fetchGradingWithAuthRetry` (refresh NineSpeak token 1 lần khi 401); `fetchWithTimeout` (AbortController per-endpoint: assessments 45s, vocab/grammar 60s, rewrite 30s, word-cefr/word-drill 30s); `assessPracticeWord(audioFile)` POST `/word-drill`.
- [practice-tts.ts](../../../features/practice/lib/practice-tts.ts): `speakEnglish(text, {rate})` (Web Speech API) + `useSpeechSynthesisAvailable()` (dùng `useSyncExternalStore` để né `react-hooks/set-state-in-effect` của React 19).
- [practice-text-heuristics.ts](../../../features/practice/lib/practice-text-heuristics.ts): `computeGrammarSentenceHeuristic` (câu đơn/phức), `detectConnectors` (liên từ), `isLikelyOffTopic` (overlap câu hỏi < 12%) — tính **cục bộ client**, không đụng scoring shared.
- [practice-evaluation.ts](../../../features/practice/lib/practice-evaluation.ts): **re-export `roundBandScore` từ shared/lib/band-score** + `practicePartLabels`, `practiceCriteriaMeta`, `normalizeCriteriaScores`, format/word/wpm helpers (đây là nguồn của barrel `index.ts`).
- [practice-persistence.ts](../../../features/practice/lib/practice-persistence.ts): persist session/result/analysis lên NineSpeak (company-api-client shared) + upload audio; chống gửi trùng bằng signature.
- [practice-response.ts](../../../features/practice/lib/practice-response.ts): type guard `isPracticeEvaluationResult` validate payload server.
- [practice-report-adapter.ts](../../../features/practice/lib/practice-report-adapter.ts): map `PracticeEvaluationResult` → shape report kit (`MockExam*`) dùng chung.
- [practice-topic-runtime.ts](../../../features/practice/lib/practice-topic-runtime.ts): tải/cache (TTL 15s) + phân trang topic detail; page size theo Part (1/3 → 3 câu, 2 → 1 câu).
- [practice-assistant.ts](../../../features/practice/lib/practice-assistant.ts): dựng nội dung Assistant Rail (câu mẫu + từ vựng) từ `MockExamSampleAnswer` theo target band.
- [practice-retry.ts](../../../features/practice/lib/practice-retry.ts): `restorePracticeAudioFileFromUrl` để chấm lại lượt đã lưu.

---

## 7. Phụ thuộc shared & billing (import cụ thể) + boundary

### 7.1 Từ `@/features/shared/...` (import sâu, READ-only)
| Thứ dùng | Đường dẫn |
|---|---|
| Pipeline 5 step | `shared/server/evaluation-pipeline` (`runPronunciationCalibration`, `runQuickScore`, `runVocabReview`, `runGrammarReview`, `runWordCefrAnalysis`, `buildAnnotationsFromEvaluation`, `vocabReviewToLexicalHighlights`, `grammarReviewToHighlights`, `QuickScoreResult`) |
| Speech / Azure | `shared/server/speech-assessment`, `shared/lib/azure-pronunciation`, `shared/lib/pronunciation-band` |
| Prompt/AI cfg | `shared/server/prompt-resolvers` (`resolvePracticePrompt`), `shared/server/openai-*`, `shared/lib/ai-scoring-settings(-browser)`, `shared/lib/prompt-overrides(-browser)` |
| Grading/quota | `shared/server/ninespeak-grading` (`validate/deductNineSpeakGradingOrThrow`), `shared/server/request-guards` |
| Recorder/audio | `shared/hooks/useMockExamRecorder`, `shared/lib/audio-presigned-upload`, `shared/lib/audio-upload-*` |
| Content/history | `shared/lib/api/app-bff-client`, `shared/lib/api/company-api-client`, `shared/lib/history-client`, `shared/lib/ninespeak-session-refresh` |
| Pricing/CEFR/band | `shared/server/pricing-config`, `shared/lib/top-up-packs`, `shared/lib/subscription-plans`, `shared/lib/cefr-mapping`, `shared/lib/band-score` |
| Types/providers/components | `shared/types/{practice-evaluation,mock-exam,app-api,ninespeak-api}`, `shared/providers/AuthProvider`, `shared/components/AssistantUnavailablePanel`, report kit |

### 7.2 Từ `@/features/billing` (CHỈ barrel)
`usePaymentOrderFlow` (trong `usePracticeQuotaPurchaseFlow`), `usePremiumPackages` + `SubscriptionPaymentModals` (trong PracticePart1/2/3). **Cấm** path nội bộ billing.

### 7.3 Boundary ESLint ([eslint.config.mjs](../../../eslint.config.mjs))
Cho `features/practice/**`:
```
blockModule("mock-test")            // cấm import mock-test
blockModule("onboarding-dashboard") // cấm import onboarding-dashboard
onlyBarrel("billing")               // billing chỉ qua @/features/billing
```
Import sâu vào `shared/` được phép. Vi phạm → `npm run lint` fail. (Đã verify: không có import mock-test/onboarding trong module.)

---

## 8. Env liên quan

Module **hầu như không đọc `process.env` trực tiếp** — hành vi đến qua dịch vụ shared. Biến tác động (đọc tại `shared`):

| Biến | Tác dụng với Practice |
|---|---|
| `PRACTICE_EVALUATION_PROVIDER`, `NEXT_PUBLIC_PRACTICE_EVALUATION_PROVIDER` | Chọn provider chấm Practice |
| `PRACTICE_TOPUP_3_VND`, `PRACTICE_TOPUP_6_VND`, `PRACTICE_TOPUP_10_VND` | Giá gói lẻ lượt chấm (qua pricing-config) |
| `AI_GRADING_ENGINE`, `OPENAI_*`, `DEEPSEEK_API_KEY`, `AI_SCORING_*` | Engine + tham số LLM (pipeline chung) |
| `AZURE_SPEECH_*`, `SPEECH_ASSESSMENT_PROVIDER`, `DEEPGRAM_*` | Phân tích giọng nói |
| `PRACTICE_PIPELINE` | **CHỈ practice** — `deepgram` → luồng chấm MỚI (Deepgram+DeepSeek, Azure chạy ngầm); trống → baseline. Rollback tức thì. KHÔNG ảnh hưởng mock-test |
| `PRACTICE_PIPELINE_DEBUG_ENABLED` | bật bench hiệu chỉnh calibrate `/debug/practice-calibration-bench` |
| `CLOUDFLARE_R2_*`, `NEXT_PUBLIC_AUDIO_UPLOAD_MODE` | Upload/lưu audio |
| `NEXT_PUBLIC_NINESPEAK_API_*` | Persistence/history qua NineSpeak |

Chi tiết & quy ước thêm biến: [env-and-modules](../../env-and-modules.md). **Không** tạo `env/practice.env`; thêm cờ qua `shared/lib/feature-flags.ts`.

---

## 9. Gotchas + nơi sửa khi mở rộng/debug

| Việc | Sửa ở đâu |
|---|---|
| Thêm/đổi tiêu chí, field evaluation | `shared/types/practice-evaluation.ts` (contract chung — cần tương thích ngược; mock-test cũng dùng `MockExam*`) |
| Đổi prompt rewrite | scope `practice.answerRewrite` + `generatePracticeAnswerRewrite` ([evaluate-practice.ts](../../../features/practice/server/evaluate-practice.ts)) |
| Đổi prompt chấm band/vocab/grammar | **KHÔNG** trong Practice — ở `shared/server/config/openai-prompts.json` + evaluation-pipeline (refactor hạ tầng) |
| Sửa logic trừ lượt | [practice-charge-ledger.ts](../../../features/practice/server/practice-charge-ledger.ts); chỉ trừ ở bước có điểm; vocab/grammar/rewrite/word-cefr không chạm ledger |
| Sửa luồng paywall | [usePracticeQuotaPurchaseFlow.ts](../../../features/practice/hooks/usePracticeQuotaPurchaseFlow.ts); subscription qua billing barrel |
| Sửa drill luyện âm | [PracticeWordDrill.tsx](../../../features/practice/components/practice/PracticeWordDrill.tsx) (UI) + `assessPracticeWordDrill`/`handlePracticeWordDrillPost` + `/word-drill` route. Không thêm ledger |
| Sửa edge case / off-topic | `edgeCase`+`classifyRecorderStartError` trong `useSpeakingPractice`, [PracticeEdgeCaseCard.tsx](../../../features/practice/components/practice/PracticeEdgeCaseCard.tsx), heuristic ở [practice-text-heuristics.ts](../../../features/practice/lib/practice-text-heuristics.ts). B3/B5 đã bỏ (không detect tin cậy) |
| Thêm phân tích tiêu chí / pause-level | Insight components + `practice-text-heuristics`; pause 3 mức tính từ `pauseDurationMs` — **KHÔNG** thêm field/scoring vào shared (giữ Mock Test nguyên) |
| Thêm/sửa analytics event Practice | `shared/lib/analytics-events/practice.ts` (Practice-owned). Component sâu đọc `practice_session_id` qua `readPracticeAnalyticsSessionId()` (sessionStorage) — khỏi thread props. Event mới: `play_pronunciation_audio`, `start_word_drill`, `save_vocab_phrase`, `practice_result_delivered` (time_to_result_ms §14.6 + band_change) |
| Debug race/stale UI | `useSpeakingPractice`: chú ý `latestRequestIdRef`, mutex, sequence counter, debounce 350ms |
| Debug "không trừ lượt" | Kiểm `sessionId` có gửi không; có `X-NineSpeak-Access-Token` không (else `skipped-no-token`); xem alpha-log `scoring-charge-*` |
| Debug calibration fail | Code **fail-fast re-throw** (chú thích "fallback" đã lỗi thời — đừng tin chú thích); banner + retry là đúng |
| Catalog/topics trống | Đó là handler của **onboarding-dashboard** (`app-data-route-handlers`), không phải server Practice |
| Sửa ranh giới import | Tuân ESLint: cấm mock-test/onboarding, billing chỉ barrel; chạy `npm run lint` + `npx tsc --noEmit` |
| Kiểm build | `npm run build` (có metadata), không phải `next build` thô |

**Quan trọng**: trong [evaluate-practice.ts](../../../features/practice/server/evaluate-practice.ts), block calibration có biến `azurePronunciationBand` và chú thích nói "fallback về công thức Azure" — nhưng nhánh `catch` thực tế **`throw error`** (Sprint 3 M3). Hành vi runtime là **fail-fast**, khớp với scoring-pipeline-overview. Chú thích là di tích cần dọn, không phản ánh code.
