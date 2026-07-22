# Mock Test — Đặc tả kỹ thuật (TECH)

> Đối tượng đọc: AI agent và đầu mối kỹ thuật module `features/mock-test/`. Cặp với [PRD.md](./PRD.md). Quy tắc ranh giới module: [../../README-modules.md](../../README-modules.md). Pipeline chấm điểm: [../../scoring-pipeline-overview.md](../../scoring-pipeline-overview.md). Env: [../../env-and-modules.md](../../env-and-modules.md).

---

## 1. Kiến trúc & sơ đồ thư mục

```
features/mock-test/
├── index.ts                  # PUBLIC API barrel cho peer (helper báo cáo isomorphic)
├── AGENTS.md                 # context cục bộ cho AI
├── pages/
│   ├── exams/                # MockExamFull.tsx, AISimulation.tsx
│   ├── reports/              # MockExamReport.tsx
│   └── *PreviewPage.tsx      # preview cho app/debug/* (KHÔNG phải luồng user)
├── layouts/                  # FocusShell.tsx
├── providers/                # MockExamSessionProvider, FocusShellActionsProvider
├── hooks/                    # điều phối phiên thi (xem §8)
├── components/               # UI phòng thi + components/exams/* (paywall/processing/purchase)
│   └── index.ts              # barrel UI nội bộ (re-export, gồm AudioWaveBars từ shared)
├── lib/                      # client libs: mock-exam-client, -report, -session-state, -retry...
├── data/                     # exam-demo.ts, mock-exam-demo-session.ts (demo nội bộ)
├── types/                    # mock-exam.ts (shim re-export từ shared)
└── server/                   # SERVER-ONLY: evaluate-mock-exam-segmented, route handlers, ledger
```

Luồng dữ liệu:
```
app/ (routing host) ──► pages/ ──► hooks/ + providers/ (state) ──► lib/mock-exam-client (fetch)
                                                                          │
                                          app/api/.../route.ts ──► server/mock-exam-route-handlers
                                                                          ├─► server/evaluate-mock-exam-segmented (shared evaluation-pipeline + speech SDK)
                                                                          └─► server/mocktest-charge-ledger (Firestore + NineSpeak deduct)
```

---

## 2. Public API barrel — [features/mock-test/index.ts](../../../features/mock-test/index.ts)

Bề mặt duy nhất cho module **peer** (hiện `onboarding-dashboard` mock backend dùng). Chỉ export helper báo cáo **isomorphic** (dùng được cả client/server), từ `lib/mock-exam-report`:

| Export | Vai trò |
|---|---|
| `isMockExamReportReady` | Báo cáo đã sẵn sàng hiển thị? |
| `isMockExamComplete` | Phiên thi đã hoàn tất chưa? |
| `isMockExamFullyGraded` | Đã chấm đủ mọi part chưa? |
| `hasMockExamPendingParts` | Còn part đang chờ chấm? |
| `computeScoringProgress` | Tiến độ chấm |
| `buildOverallCriteria` | Dựng 4 tiêu chí tổng |

> Barrel **cố tình KHÔNG gom** page/layout/route-handler — `app/` (host) import trực tiếp các file đó (quan hệ host↔module), tránh anti-pattern barrel-file của Next gây phình bundle / rò server-only.

---

## 3. Routes trong `app/`

### 3.1 Pages
| Route | File app/ | Mount component |
|---|---|---|
| `/mock-exam/select` | [app/(focus-mock)/mock-exam/select/page.tsx](../../../app/(focus-mock)/mock-exam/select/page.tsx) | `MockExamSelect` (màn chọn đề & Hero, full-screen shell `(focus-mock)`, không sidebar) |
| `/mock-exam/full` | [app/(focus)/mock-exam/full/page.tsx](../../../app/(focus)/mock-exam/full/page.tsx) | `MockExamFull` (truyền `topUpPacks`; đọc `?examId` → giám khảo riêng của đề + dựng `presetDefinition` đúng câu hỏi của đề qua [server/mock-exam-definition-from-catalog.ts](../../../features/mock-test/server/mock-exam-definition-from-catalog.ts)) |
| `/ai-simulation/examiner` | [app/(focus)/ai-simulation/examiner/page.tsx](../../../app/(focus)/ai-simulation/examiner/page.tsx) | `AISimulation` |
| `/mock-exam` | [app/(shell)/mock-exam/page.tsx](../../../app/(shell)/mock-exam/page.tsx) | redirect → `/mock-exam/select` |
| `/ai-simulation` | [app/(shell)/ai-simulation/page.tsx](../../../app/(shell)/ai-simulation/page.tsx) | redirect → `/ai-simulation/examiner` |
| `/report/[sessionId]` | [app/(shell)/report/[sessionId]/page.tsx](../../../app/(shell)/report/[sessionId]/page.tsx) | `MockExamReport` |
| `/debug/mock-exam-*` | [app/debug/](../../../app/debug) | preview nội bộ (KHÔNG phải luồng user) |

### 3.2 API route handlers
Tất cả route handler chỉ là vỏ mỏng gọi vào [server/mock-exam-route-handlers.ts](../../../features/mock-test/server/mock-exam-route-handlers.ts) (`runtime = "nodejs"`, `dynamic = "force-dynamic"`):

| Route | Handler | Mục đích |
|---|---|---|
| `POST /api/app/speaking/mock-exams/parts/[part]/assessments` | `handleMockExamPartAssessmentsPost` | Chấm 1 câu (Part 1/3 Azure-only) hoặc cả part atomic (Part 2) |
| `POST .../finalizations/score` | `handleMockExamPartFinalizationScorePost` | Calibration + Quick Score (Part 1/3) |
| `POST .../finalizations/vocab` | `handleMockExamPartFinalizationVocabPost` | Vocab Review (cần lockedScore) |
| `POST .../finalizations/grammar` | `handleMockExamPartFinalizationGrammarPost` | Grammar Review (cần lockedScore) |
| `POST .../finalizations/word-cefr` | `handleMockExamPartFinalizationWordCefrPost` | Phân phối CEFR theo từ |
| `POST /api/app/speaking/mock-exams/finalizations/overall-summary` | `handleMockExamOverallBreakdownPost` | Report "Tổng quan" v2 — synthesis breakdown toàn bài (sub-score + descriptor 4 tiêu chí). **KHÔNG part-scoped, KHÔNG ledger, KHÔNG lockedScore.** |
| `POST /api/app/speaking/mock-exams/drills/pronunciation` | `handleMockExamPronunciationDrillPost` ([server/mock-exam-drill.ts](../../../features/mock-test/server/mock-exam-drill.ts)) | Mini-drill "Luyện ›" — chấm phát âm 1 từ (Azure scripted `evaluateAzureWithReference`). **MIỄN PHÍ — KHÔNG ledger, KHÔNG session/analysis.** |
| `POST .../finalizations` (atomic) | `handleMockExamPartFinalizationsPost` | Legacy atomic (backward-compat) |
| `GET /api/app/speaking/mock-exams/catalog` | `handleSpeakingMockExamCatalogGet` (**ở onboarding-dashboard** BFF) | Catalog đề thi |
| `GET /api/app/mock-exam/catalog` | `handleSpeakingMockExamCatalogGet` (alias cũ) | Catalog đề thi |
| `POST /api/speaking/mock-exam/evaluate` | `handleMockExamPartAssessmentsPost` (qua `?part=`) | Alias legacy assessments |
| `POST /api/speaking/mock-exam/finalize-segmented-part` | `handleMockExamPartFinalizationsPost` (qua `?part=`) | Alias legacy finalize |

> Catalog handler nằm ở `onboarding-dashboard/server` (composition root) — đây là trường hợp host/BFF, không phải import chéo từ mock-test.

---

## 4. Data model / types

- **Contract types**: [types/mock-exam.ts](../../../features/mock-test/types/mock-exam.ts) là **shim** `export * from "@/features/shared/types/mock-exam"`. Định nghĩa thật ở [features/shared/types/mock-exam.ts](../../../features/shared/types/mock-exam.ts) (`MockExamSession`, `MockExamPart`, `MockExamPartEvaluation`, `MockExamQuestionAssessmentPayload`, `MockExamFlowMode`...). **Không thêm type mới** trong shim.
- **Định nghĩa đề & ngân hàng câu hỏi (shared)**: [features/shared/data/mock-exam-definition.ts](../../../features/shared/data/mock-exam-definition.ts), [features/shared/data/mock-exam.ts](../../../features/shared/data/mock-exam.ts), [features/shared/data/mock.ts](../../../features/shared/data/mock.ts); JSON `part{1,2,3}.json` + `question-store` ở [features/shared/server/](../../../features/shared/server).
- **Data nội bộ module**: [data/exam-demo.ts](../../../features/mock-test/data/exam-demo.ts), [data/mock-exam-demo-session.ts](../../../features/mock-test/data/mock-exam-demo-session.ts) (dữ liệu demo cho preview / dev).

---

## 5. Server layer

### 5.1 Evaluator — [server/evaluate-mock-exam-segmented.ts](../../../features/mock-test/server/evaluate-mock-exam-segmented.ts)
Các export chính:
| Function | Dùng cho |
|---|---|
| `evaluateMockExamSegmentedQuestion` | Chấm 1 câu Part 1/3 — chỉ Azure Speech, lưu `azureAggregates` |
| `evaluateMockExamSegmentedPartScore` | Finalize /score: calibration + quickScore |
| `evaluateMockExamSegmentedPartVocab` | Finalize /vocab |
| `evaluateMockExamSegmentedPartGrammar` | Finalize /grammar |
| `evaluateMockExamSegmentedPartWordCefr` | Finalize /word-cefr |
| `evaluateMockExamSegmentedPart` | Legacy atomic (4 bước) |
| `evaluateMockExamOpenAiAzurePart` | Part 2 whole-part atomic (Azure + 4 bước OpenAI) |

Các function này gọi pipeline lõi ở [features/shared/server/evaluation-pipeline.ts](../../../features/shared/server/evaluation-pipeline.ts) (`runPronunciationCalibration`, `runQuickScore`, `runVocabReview`, `runGrammarReview`) + speech SDK shared ([azure-speech](../../../features/shared/server/azure-speech.ts), [deepgram-speech](../../../features/shared/server/deepgram-speech.ts), [speech-assessment](../../../features/shared/server/speech-assessment.ts)) + [openai-structured-output](../../../features/shared/server/openai-structured-output.ts) + [prompt-resolvers](../../../features/shared/server/prompt-resolvers.ts).

### 5.2 Route handlers — [server/mock-exam-route-handlers.ts](../../../features/mock-test/server/mock-exam-route-handlers.ts)
Xác thực bearer-token (request-guards của shared), parse multipart, route theo `part`, gọi evaluator, ghi alpha-log per-phase, gọi ledger. Xem các handler ở §3.2.

### 5.3 Synthesis "Tổng quan" v2 — [server/evaluate-mock-exam-overall-breakdown.ts](../../../features/mock-test/server/evaluate-mock-exam-overall-breakdown.ts)
`runMockExamOverallBreakdownSynthesis(input)` — **1 call LLM** mức TOÀN BÀI: nhận overallBand + cefrLevel + 4 band (đã lock) + per-part `{part, band, feedbackSummary}` → mỗi tiêu chí trả **3 sub-score** (phân rã band, *không re-score* — trung bình lệch ≤0.25, mỗi sub ≤1.0) + **descriptor tiếng Việt**. Prompt mock-only `evaluation.overallBreakdown` ([config/openai-prompts.json](../../../features/shared/server/config/openai-prompts.json), đọc thẳng `.system`, **không** qua prompt-override). Nhãn sub-score gắn server từ `mock-exam-criteria-display`. Handler `handleMockExamOverallBreakdownPost` (auth + rate-limit, **KHÔNG ledger**).
- **Persist eager** (phòng thi): `usePersistExamSession` → `syncMockExamSession` → `ensureMockAnalysis` ghi `metadata.overallBreakdown`. ⚠️ `buildRemoteSyncSnapshot` **phải** chứa tín hiệu `overallBreakdown` (status + updatedAt); thiếu → snapshot không đổi → breakdown KHÔNG xuống DB (bug cũ).
- **Persist fallback** (report mở từ lịch sử): synth lại → `persistMockExamOverallBreakdown` ([mock-exam-persistence.ts](../../../features/mock-test/lib/mock-exam-persistence.ts)) tìm Analysis theo `localSessionKey`, GIỮ NGUYÊN resultIds/metadata cũ, chỉ chèn `overallBreakdown` (idempotent). Hydrate qua `parsePersistedOverallBreakdown` ([history-client.ts](../../../features/shared/lib/history-client.ts)).

### 5.4 Logging
Per-phase event qua [features/shared/server/alpha-logger.ts](../../../features/shared/server/alpha-logger.ts): `mock-exam-pronunciation-calibration-*`, `mock-exam-quick-score-*`, `mock-exam-vocab-review-*`, `mock-exam-grammar-review-*`, `mock-exam-finalize-{score,vocab,grammar}-*`, `mock-exam-overall-breakdown-*`, ledger `scoring-charge-*`. UI ở `/settings/logs`.

---

## 6. Pipeline phân đoạn (tóm tắt kỹ thuật)

```
Part 1/3:  per-question POST /assessments (Azure only, lưu azureAggregates)
           └─► hết part: POST /finalizations/score  (calibration + quickScore, trả band tổng)
                   └─► client: Promise.allSettled([ /finalizations/vocab, /finalizations/grammar ])
Part 2:    1× POST /assessments  → atomic: Azure + calibration + quickScore + vocab + grammar
```
Per-step status trên `MockExamPartEvaluation`: `partVocabReviewStatus` / `partGrammarReviewStatus` ∈ `pending|completed|failed`. Chi tiết bước/prompt: [../../scoring-pipeline-overview.md](../../scoring-pipeline-overview.md) §3.

---

## 7. Ledger — mocktest charge ledger

File: [server/mocktest-charge-ledger.ts](../../../features/mock-test/server/mocktest-charge-ledger.ts).
- **Firestore collection**: `mocktest_scoring_charges` (tách biệt Practice).
- **Doc ID**: `{sessionId}` (mã phiên thi client) — xem `chargeId()` trong file.
- **State machine**: `pending` → `charging` → `charged` | `charge-failed` (2-phase commit lite, idempotent + race-safe).
- **Trừ lượt**: khi mọi visibleParts có `scoreDone`. Vocab/Grammar sau đó không trừ thêm.
- **Deduct thật** gọi qua NineSpeak ([features/shared/server/ninespeak-grading.ts](../../../features/shared/server/ninespeak-grading.ts)); Firestore admin qua [lib/firebase/admin.ts](../../../lib/firebase/admin.ts).
- **Reconcile docs kẹt** (crash giữa phase): cron admin — xem [app/api/internal/cron/charge-reconcile-report/route.ts](../../../app/api/internal/cron/charge-reconcile-report/route.ts) và (general cleanup) `app/api/internal/cron/cleanup-stuck-charges/route.ts`.

> ⚠️ Lưu ý: doc thực tế dùng doc ID = `{sessionId}`. (Bảng "Tóm tắt" trong scoring-pipeline-overview ghi dạng `{userId}__mocktest__{sessionId}` — nếu sửa ledger hãy lấy code làm chuẩn.)

---

## 8. State — hooks & providers

### 8.1 Providers
- [MockExamSessionProvider](../../../features/mock-test/providers/MockExamSessionProvider.tsx) — context phiên thi, state machine qua [lib/mock-exam-session-state.ts](../../../features/mock-test/lib/mock-exam-session-state.ts) (`createMockExamSession`, `applyQuestion*`, `applyPart*`, `applySegmentedPartFinalizing`...).
- [FocusShellActionsProvider](../../../features/mock-test/providers/FocusShellActionsProvider.tsx) — đăng ký handler `cancelExam` / `beforeExit` cho FocusShell.

### 8.2 Hooks chính
| Hook | Vai trò |
|---|---|
| [useMockExamRecorder](../../../features/shared/hooks/useMockExamRecorder.ts) (**shared**) | Recorder lõi (ghi âm) — dùng chung Mock/Practice |
| [useExamRecordingController](../../../features/mock-test/hooks/useExamRecordingController.ts) | Điều khiển ghi âm trong phòng thi, giới hạn thời gian, auto-next |
| [useQuestionTtsAutoRecord](../../../features/mock-test/hooks/useQuestionTtsAutoRecord.ts) | TTS đọc đề + auto bật ghi âm |
| [useSegmentedPartFinalization](../../../features/mock-test/hooks/useSegmentedPartFinalization.ts) | Auto chạy /score → allSettled([/vocab,/grammar]); expose retry per-tab; sequence counter chống race |
| [useOverallBreakdownSynthesis](../../../features/mock-test/hooks/useOverallBreakdownSynthesis.ts) | Report "Tổng quan" v2 — eager synthesis breakdown toàn bài khi đủ part lock band (1 call, idempotent theo sessionId). Store-agnostic (callback `onSession`); dùng ở `MockExamFull` (eager, commit qua provider `replaceSession`) + fallback ở `MockExamReport` (commit optimisticSession + app-snapshot + `persistMockExamOverallBreakdown` xuống Analysis metadata) |
| [useExamSessionSync](../../../features/mock-test/hooks/useExamSessionSync.ts) | Đồng bộ phiên |
| [usePersistExamSession](../../../features/mock-test/hooks/usePersistExamSession.ts) | Lưu phiên qua NineSpeak |
| [useMockExamReportSession](../../../features/mock-test/hooks/useMockExamReportSession.ts) | Đọc phiên cho báo cáo |
| [useMockExamQuotaPurchaseFlow](../../../features/mock-test/hooks/useMockExamQuotaPurchaseFlow.ts) | Paywall/mua lượt (qua billing barrel) |
| [useMockExamBeforeUnloadGuard](../../../features/mock-test/hooks/useMockExamBeforeUnloadGuard.ts) | Chặn reload/đóng tab khi đang chấm |
| [useMicTest](../../../features/mock-test/hooks/useMicTest.ts) | Kiểm tra mic chuẩn bị. Expose `levels` (sóng âm chỉ nảy khi có âm thanh thật, không bounce CSS) + `progress` 0..1 (thanh progress chạy theo thời lượng nói — heuristic, không ASR). User nói vài câu → progress đầy (`REQUIRED_VOICED_MS`) + lặng ngắn → `success` (tích "mic ổn định"). Chỉ có tiếng lẻ tẻ → `failed` |

### 8.3 Client libs đáng chú ý
[lib/mock-exam-client.ts](../../../features/mock-test/lib/mock-exam-client.ts) (`evaluateMockExamAudio`, `finalizeMockExamSegmentedPart{Score,Vocab,Grammar}`, `finalizeMockExamOverallBreakdown`, legacy atomic), [lib/mock-exam-report.ts](../../../features/mock-test/lib/mock-exam-report.ts) (helper export qua barrel + `buildOverallScoreAccordion` cho "Tổng quan" v2), [lib/mock-exam-criteria-display.ts](../../../features/mock-test/lib/mock-exam-criteria-display.ts) (nhãn 4 tiêu chí + 3 sub-score cố định — isomorphic, dùng cả server synthesis & UI), [lib/mock-exam-retry.ts](../../../features/mock-test/lib/mock-exam-retry.ts), [lib/mock-exam-response.ts](../../../features/mock-test/lib/mock-exam-response.ts), [lib/exam-flow.ts](../../../features/mock-test/lib/exam-flow.ts), [lib/mock-exam-navigation.ts](../../../features/mock-test/lib/mock-exam-navigation.ts), [lib/mock-exam-assistant.ts](../../../features/mock-test/lib/mock-exam-assistant.ts).

### 8.4 Catalog đề & lời thoại giám khảo (màn chọn đề + lời dẫn)
- **Data**: [data/mocktest-exam-v1.json](../../../features/mock-test/data/mocktest-exam-v1.json) (`version: 2`) — 81 đề full. Mỗi đề: `questionIds` (Part 1 = 5 câu, Part 2 = 1 cue card, Part 3 = 3 câu), `topics` (id), `access` (free/premium), `examinerName`, **`leadIns` riêng theo đề** (`intro`/`toPart2`/`toPart3`). Lời bất biến ở `commonScripts`: `acknowledgements` (12 câu, chọn ngẫu nhiên), `part2Start`, `ending`.
- [lib/mock-exam-select-catalog.ts](../../../features/mock-test/lib/mock-exam-select-catalog.ts): nạp + định hình catalog theo scope cho màn chọn đề.
- [lib/mock-exam-progress-store.ts](../../../features/mock-test/lib/mock-exam-progress-store.ts) + [hooks/useMockExamProgress.ts](../../../features/mock-test/hooks/useMockExamProgress.ts): **progress bài thi lưu LOCAL** (`localStorage` key `mock-exam-progress-v1`) — đã làm / đang làm dở (+ band, resume). `markMockExamInProgress` (lúc `goToExam`) · `markMockExamDone` (lúc `MockExamFull` → `completed`); đọc reactive qua `useMockExamProgress` (`useSyncExternalStore` + event nội bộ + `storage`). `useMockExamSelectSignals` nạp `perExamStatus`/`resume` từ đây (bỏ stub). Key = `catalogId` (`buildMockExamCatalogId`: examId full / `examId::part` Part lẻ → khớp `perExamStatus[item.id]`).
- [lib/mock-exam-scripts.ts](../../../features/mock-test/lib/mock-exam-scripts.ts): `getMockExamCommonScripts()`, `getMockExamLeadIns(examId)` (lời dẫn riêng từng đề), `withExaminerName()` (thay `{examiner}`), `pickAcknowledgement()`.
- [lib/text-to-speech-service.ts](../../../lib/text-to-speech-service.ts) (**root `lib/`**): `TextToSpeechService` + `buildTtsAudioUrl` (proxy) — Google Translate TTS (tách đoạn ≤200 ký tự, phát tuần tự, `stop()`). ⚠️ **Phát qua proxy same-origin** [app/api/tts/route.ts](../../../app/api/tts/route.ts): gọi `translate_tts` thẳng từ `<audio>` client bị **404 (Referer cross-site)**; proxy fetch server-side (không Referer) → 200.
- **Phụ đề & nhịp narration** (`MockExamFull` + `MockExamLiveStage` + `ExamPreparationScreen`): toggle "Phụ đề" (`showQuestion`) **persist** qua đổi câu/part; phụ đề = câu đang nói (gate `showQuestion` cho câu hỏi, `showQuestion && narrationText` cho narration). **Cue card Part 2 KHÔNG bám toggle** (`showCueCard`): narrate → ẩn (chỉ avatar + phụ đề), đọc xong → hiện (animation). Đếm ngược prep2 gate `!isNarrating` (vào thẳng → đếm ngay). "Qua câu" (`handleSkipQuestion`) vẫn phát ack; `moveToNextPart()` cho câu cuối.
- **Màn chúc mừng** ([MockExamCelebrationScreen.tsx](../../../features/mock-test/components/exams/MockExamCelebrationScreen.tsx)): `completed` → `MockExamFull` chèn `completionStage` `"celebrate"`→`"processing"` **trước** màn xử lý. Đọc `ending` xong (`!isNarrating`) → pháo hoa (CSS `animate-firework-spark` ở `app/globals.css`) + `setTimeout(CELEBRATION_HOLD_MS=3600)` → `"processing"`. Redirect `/report/[id]` gate `completionStage==="processing"`. Scoring chạy nền suốt celebrate.
- **Công cụ test audio (chỉ `isAdmin`, VẪN trừ lượt)** — nộp audio sẵn, chấm không cần ghi âm. `useExamRecordingController` export `submitExternalAudio(file,{advanceOnCapture})` (duration `<audio>`, MIN 3s, **pipeline thật**: presigned R2 → `evaluateMockExamAudio`; recording cũ không đổi). "Tải audio" (`accept="audio/*"`) → `submitExternalAudio`. "Qua câu" → `getTestAnswerAudioUrl(questionId)` ([data/test-answer-audio.ts](../../../features/mock-test/data/test-answer-audio.ts) → `public/test-audio/mock-exam/`, chỉ đề `forecast-full-337`; thiếu file → bỏ qua).
  - ⚠️ **Audio chấm BẮT BUỘC WAV PCM mono 16-bit 16000 Hz** — 2 lớp chặn: `assertAudioUpload` (mp3/m4a → 400) + Azure `validateAudioDiagnostics` ([azure-speech.ts](../../../features/shared/server/azure-speech.ts), sai sample rate → 502). Recorder đã 16kHz; audio test `submitExternalAudio` **tự chuẩn hoá** (`normalizeAudioToAzureWav`); `npm run gen:test-answers` resample OpenAI TTS 24kHz→16kHz.
- **R2 audio key** = `user-response/{mode}/{uid}/{part}/{topicId}/{questionId}/{randomUUID}.{ext}`. Mock giờ truyền `topicId` thật (`activeQuestion.topicId`) qua `uploadSpeakingAudioPresigned` + `primeMockExamAttemptPersistence` → hết `unknown-topic`. **Không** trùng key với Practice: segment `mode` khác (`mock` vs `practice`) + filename `crypto.randomUUID()` (mỗi upload 1 object riêng, không ghi đè). `topicId` chỉ là path lưu trữ — **không** ảnh hưởng chấm điểm (route vẫn đặt `question.topicId = questionId`).
  - `questionTitle` client = metadata thôi (Part 1/3 server tự suy `Câu N`; Part 2 = `topicTitle`) — không phải input chấm chính.
- **Màn chọn đề** ([components/select/MockExamSelectScreen.tsx](../../../features/mock-test/components/select/MockExamSelectScreen.tsx)): đã **bỏ thanh demo/stub admin** (`DemoBar`/`DemoState`/`mergeDemoSignals`) — chỉ render `useMockExamSelectSignals` thật + check rỗng; bỏ prop `isAdmin`.

### 8.5 Report "Phản hồi chi tiết v2" — rendering (sections shared + wiring Mock)
- **Wiring Mock**: [pages/reports/MockExamReport.tsx](../../../features/mock-test/pages/reports/MockExamReport.tsx) — card "Phản hồi chi tiết": `<h3>` + **part label** + cue card Part 2 (amber) → tabs Part (ẩn ở **Part lẻ**) → **tabs pill** 4 tiêu chí (Trôi chảy→Từ vựng→Phát âm→Ngữ pháp) → grid 50/50 (trái card-per-câu, phải panel tiêu chí).
- **Sections (shared)**: [shared/components/reports/MockExamReportSections.tsx](../../../features/shared/components/reports/MockExamReportSections.tsx) + [MockExamReportNavigation.tsx](../../../features/shared/components/reports/MockExamReportNavigation.tsx). ⚠️ **Practice DÙNG CHUNG** `TranscriptViewer`/`AnnotationPopup`/`AnnotationLegend` → mọi thay đổi phải **additive**: Practice giữ `renderMode="transcript"` (mặc định), Mock bật `renderMode="question-cards"`.
- **Props bật từ Mock** (đều **optional** → Practice không truyền = ẩn/giữ cũ): `resolveQuestionAudio(qi) → {url, offsetSeconds}` · `onPlayPrecise(start,end,weakWord)` (phát đúng đoạn user nói) · `onPracticeWord(word,ipa)` (mở mini-drill) · `onSpeakReference(word)` (Mock truyền **Google TTS**; Practice giữ `speechSynthesis`).
- **MockExamAnswerCard** (module-level, giữ state player ổn định): card per-câu + `<audio>` (play/pause + seek + clock) + **karaoke** rAF (từ đang đọc `scale-[1.16]`+bg-amber; `reducedMotion` tắt scale). ⚠️ **Karaoke offset**: audio per-câu bắt đầu ở 0 nhưng `word.startTimeSeconds` **tuyệt đối** (aggregate) → active khi `currentTime ∈ [start−offset, end−offset)`, `offset = segment.startSeconds` (0 nếu dùng audio cả part). Không trừ → chỉ câu 1 đúng.
- **Tô theo tiêu chí** (`renderWord`/`renderSentence`):
  - **Fluency**: chấm ngắt **sau MỖI từ có pause** (`resolvePauseTier` = pauseClassification + độ dài, 3 mức tốt/vừa/dài) — KHÔNG gộp 1 chấm/câu.
  - **Lexical**: gạch chân màu **CEFR** (`cefrTextColorClass`) + panel **CEFR 6 mức A1–C2** (`CefrProfileCard` từ `word.cefrLevel`, fallback `lexicalProfile.cefrDistribution`).
  - **Grammar**: **sentence-card** đơn/phức (`classifySentenceComplexity` — heuristic regex, KHÔNG gọi pipeline) + có/không lỗi (annotation grammar); độ chính xác **"X/Y câu"**.
  - **Pronunciation**: tô **mọi từ 3 mức** (`pronWordTone`: ≥75 emerald / 50–75 amber / <50 rose) + **✗ sau từ <75** ("Nhóm theo âm" KHÔNG gắn ✗); lưới "Lỗi phát âm (nặng→nhẹ)" + **IPA as-said** (`buildSpokenIpaFromPhonemes` — **shared additive** trong [azure-pronunciation.ts](../../../features/shared/lib/azure-pronunciation.ts), tái dựng từ Azure **NBestPhonemes** `nbestPhonemeCount=5`; ẩn nếu trùng `referenceIpa`) + ✓ Đúng `/referenceIpa/` (🔊 Google TTS) + ngữ cảnh câu + "Luyện ›".
- **Mini-drill**: [components/reports/MockExamPronunciationDrillModal.tsx](../../../features/mock-test/components/reports/MockExamPronunciationDrillModal.tsx) — ghi âm (`useMockExamRecorder`) → [lib/mock-exam-drill-client.ts](../../../features/mock-test/lib/mock-exam-drill-client.ts) POST `/drills/pronunciation` (§3.2, **KHÔNG ledger**). "Phát âm mẫu"/"đúng" = **Google TTS** (`buildTtsAudioUrl` qua `/api/tts`) — **chỉ Mock**.
- 📌 **Chưa làm / cần pipeline-content**: [gap.md](./gap.md) (badge Mạch lạc, filler vị trí, idiom, "💡 Nên thử" + câu mẫu cấu trúc, bảng khẩu hình theo phoneme).

---

## 9. Phụ thuộc shared & billing + boundary ESLint

### 9.1 Import từ `shared` (import sâu — READ-ONLY, không sửa)
Nhóm chính module đang dùng:
- **Providers**: `shared/providers/AuthProvider`, `FeedbackSurveyProvider`.
- **Hooks**: `shared/hooks/useMockExamRecorder`, `useRequireTargetBand`, `useReducedMotionPreference`, `useMockExamReportInteractions`.
- **lib**: `feature-flags`, `band-score`, `api/app-bff-client`, `api/company-api-client`, `top-up-packs`, `subscription-plans`, `audio-upload-client`, `audio-presigned-upload`, `audio-upload-mode`, `analytics*`, `alpha-log*`, `ai-scoring-settings*`, `prompt-overrides*`, `pronunciation-band`, `transcript-pause-*`, `azure-pronunciation`, `history-client`, `ninespeak-session-refresh`, `mic-permission`, `messenger-checkout`, `entity-id`, `api-response`.
- **server**: `evaluation-pipeline`, `azure-speech`, `deepgram-speech`, `speech-assessment`, `openai-structured-output`, `openai-prompt-config`, `prompt-resolvers`, `request-guards`, `ninespeak-grading`, `alpha-logger`, `alpha-log-runtime`.
- **components**: `AudioWaveBars`, `AssistantUnavailablePanel`, `PromptOverrideWorkbench`, `reports/MockExamReport{Sections,Navigation,Chrome}`.
- **data**: `mock-exam-definition`, `mock-exam`, `mock`.
- **types**: `mock-exam`, `ninespeak-api`.

### 9.2 Import từ `billing` — CHỈ qua barrel `@/features/billing`
| File | Import |
|---|---|
| [components/exams/MockExamPurchaseStatusScreen.tsx](../../../features/mock-test/components/exams/MockExamPurchaseStatusScreen.tsx) | `PurchaseStatusBackdrop` |
| [components/exams/MockExamQuotaPaywallScreen.tsx](../../../features/mock-test/components/exams/MockExamQuotaPaywallScreen.tsx) | `PricingPlanCards`, `PurchaseReportBlurSurface` |
| [hooks/useMockExamQuotaPurchaseFlow.ts](../../../features/mock-test/hooks/useMockExamQuotaPurchaseFlow.ts) | `usePaymentOrderFlow` |
| [pages/exams/MockExamFull.tsx](../../../features/mock-test/pages/exams/MockExamFull.tsx) | `SubscriptionPaymentModals`, `usePremiumPackages` |

### 9.3 Ranh giới (ESLint `eslint.config.mjs`)
- **CẤM** import `@/features/practice/*` và `@/features/onboarding-dashboard/*`.
- **CẤM** path nội bộ của Billing (`@/features/billing/hooks/*`, `.../components/*`) — chỉ barrel.
- **Được** import sâu vào `shared/` (lõi READ-ONLY).
- `shared/` không bao giờ import ngược vào feature module.
- Phân công review: `.github/CODEOWNERS`.

---

## 10. Env liên quan

Module **không tự đọc `process.env`** (trừ flow flag qua shared). Chi tiết & bản đồ đầy đủ: [../../env-and-modules.md](../../env-and-modules.md).

| Env | Đọc tại | Ảnh hưởng |
|---|---|---|
| `MOCK_EXAM_PART13_FLOW_MODE`, `NEXT_PUBLIC_MOCK_EXAM_PART13_FLOW_MODE` | shared `feature-flags` | chế độ chấm Part 1/3 |
| `AI_GRADING_ENGINE`, `OPENAI_*`, `DEEPSEEK_API_KEY`, `AI_SCORING_*` | shared/server | LLM chấm điểm |
| `AZURE_SPEECH_*`, `DEEPGRAM_*`, `SPEECH_ASSESSMENT_PROVIDER` | shared/server | phát âm/STT |
| `PRACTICE_TOPUP_*_VND` | shared `pricing-config` | giá gói lẻ lượt chấm |
| `NEXT_PUBLIC_NINESPEAK_API_*` | shared / onboarding-dashboard | persistence/history |
| `CLOUDFLARE_R2_*`, Firebase admin | `lib/` | upload audio, auth/ledger |

---

## 11. Gotchas + nơi sửa khi thêm tính năng / sửa lỗi

- **Sửa logic chấm Part 1/3**: evaluator [evaluate-mock-exam-segmented.ts](../../../features/mock-test/server/evaluate-mock-exam-segmented.ts) + handler [mock-exam-route-handlers.ts](../../../features/mock-test/server/mock-exam-route-handlers.ts) + hook [useSegmentedPartFinalization.ts](../../../features/mock-test/hooks/useSegmentedPartFinalization.ts).
- **Đổi prompt/anchor band**: KHÔNG sửa ở module — prompt nằm ở `shared/server/config/openai-prompts.json`; runtime override qua Prompt Lab (`/settings/prompt-lab`).
- **Sửa logic trừ lượt**: chỉ ledger [mocktest-charge-ledger.ts](../../../features/mock-test/server/mocktest-charge-ledger.ts); KHÔNG đụng collection của Practice.
- **Thêm năng lực thanh toán**: yêu cầu owner Billing export thêm tại `features/billing/index.ts`; module chỉ tiêu thụ qua barrel.
- **Cần tiện ích dùng chung mới** (recorder/report kit/util): đề xuất đưa vào `shared` (tương thích ngược), KHÔNG copy vào module hay import từ practice.
- **Báo cáo partial/retry per-tab**: UI ở `shared/components/reports/*` (wire callback) + trang [MockExamReport.tsx](../../../features/mock-test/pages/reports/MockExamReport.tsx).
- **Sửa report "Phản hồi chi tiết" (rendering)**: sections ở `shared/components/reports/MockExamReportSections.tsx` **dùng chung Practice** → đổi phải **additive** (xem §8.5); wiring Mock ở `MockExamReport.tsx`. Nhớ: **karaoke offset = `segment.startSeconds`**; "Phát âm đúng/mẫu" Google TTS **chỉ Mock**; **mini-drill KHÔNG trừ ledger**. Việc chưa làm được (cần pipeline/content) đã liệt kê ở [gap.md](./gap.md) — đừng implement "chay" client.
- **Debug "tab Từ vựng đỏ"**: filter alpha-log theo user → `requestId` → tìm `mock-exam-vocab-review-*-failed`.
- **Doc ID ledger**: lấy code làm chuẩn (`{sessionId}`), không lấy ví dụ trong overview.
- **Lan sang peer**: nếu thay đổi đòi sửa `practice`/`onboarding-dashboard`/`billing` nội bộ → dừng lại, đó là dấu hiệu vi phạm boundary; xử lý qua barrel hoặc shared.
- **Build/lint**: `npm run build` (có metadata) + `npx tsc --noEmit` + `npx eslint <file đã đổi>` trước khi commit. ESLint sẽ chặn import chéo vi phạm boundary.
