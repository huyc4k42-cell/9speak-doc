# Shared Core — Đặc tả Kỹ thuật (TECH)

> Đặc tả kỹ thuật cho lõi `features/shared/`. Đối tượng: AI agent + người đầu mối module phát triển/sửa lỗi với nhận thức về ràng buộc liên-module. Mô tả năng lực ở cấp sản phẩm → [PRD.md](./PRD.md).

---

## 1. Sơ đồ thư mục & vai trò từng nhóm

```
features/shared/
├── lib/         # client-side libs (an toàn cho cả client & server logic không secret)
├── server/      # SERVER-ONLY (import "server-only") — secret, SDK, pipeline
├── types/       # contract types (nguồn sự thật cho app-api / ninespeak-api / mock-exam / practice)
├── data/        # nội dung client-side (mock-exam definition, question bank)
├── providers/   # React context cross-cutting (Auth, Analytics, FeedbackSurvey, Diagnostics)
├── hooks/       # React hooks dùng chung (recorder, audio, report interactions...)
└── components/  # UI tái dùng (waveform, panels, report kit)
```

| Nhóm | Tính chất | Quy tắc |
|---|---|---|
| `lib/` | Client-safe (gồm `api/` re-export). Một số đọc `process.env` công khai (`NEXT_PUBLIC_*`). | Không chứa secret server. |
| `server/` | **Server-only**. Mọi file mở đầu `import "server-only"`. Đọc secret env (OpenAI/Azure/Deepgram key). | **Cấm** import vào client component. |
| `types/` | Type thuần, không runtime. | Nguồn sự thật; module re-export qua shim. |
| `data/` | JSON + getter client. | Nội dung tĩnh dùng chung. |
| `providers/`, `hooks/`, `components/` | `"use client"` React. | Cross-cutting UI/state. |

Ranh giới enforce: [eslint.config.mjs](../../../eslint.config.mjs) — block `features/shared/**` import từ 4 feature module.

---

## 2. Từng dịch vụ: API chính, file, luồng dữ liệu

### 2.1 Evaluation Pipeline (4 bước) — [evaluation-pipeline.ts](../../../features/shared/server/evaluation-pipeline.ts)

Hàm chính (tất cả async, trả `Promise`):

| Hàm | Input type | Output type |
|---|---|---|
| `runPronunciationCalibration` | `PronunciationCalibrationInput` | `PronunciationCalibrationResult` `{ band, rationale }` |
| `runQuickScore` | `QuickScoreInput` | `QuickScoreResult` `{ overallBand, criteria, cefrLevel, feedbackSummary, improvements }` |
| `runVocabReview` | `VocabReviewInput` | `VocabReviewResult` `{ lexicalErrors, lexicalSuggestions, cefrDistribution }` |
| `runGrammarReview` | `GrammarReviewInput` | `GrammarReviewResult` `{ grammarErrors, complexStructureErrors, errorSummary, complexStructureUsage, generalFeedback }` |
| `runWordCefrAnalysis` | `WordCefrAnalysisInput` | `WordCefrAnalysisResult` `{ wordCefrLevels }` (background, fail-safe) |

**Luồng dữ liệu (mỗi module gọi từ evaluator của mình):**

```
Azure/Deepgram/OpenAI speech assessment  →  AzureSpeechAssessmentResponse
        │  (transcript + 5 aggregate scores + word-level + pause stats)
        ▼
runPronunciationCalibration()  →  band (LOCK)        ── fail → throw (fail-fast, M3)
        ▼
runQuickScore(pronunciationBand = locked)  →  lockedScore
        │  server recompute overallBand = mean(4 criteria) → chống drift
        │  cefrLevel = cefrFromOverallBand(recomputed)
        ▼  (lockedScore truyền nguyên vào 2 bước sau, CẤM rescore)
   ┌──────────────┴──────────────┐
runVocabReview(lockedScore)   runGrammarReview(lockedScore)   ← module chạy parallel
   └──────────────┬──────────────┘
        ▼
mapping helpers → MockExam* highlights / annotations (report UI)
```

**Chi tiết kỹ thuật quan trọng:**
- Mọi bước gọi qua wrapper `requestStructuredOpenAiJson` ([openai-structured-output.ts](../../../features/shared/server/openai-structured-output.ts)) với JSON schema strict + `parseContent` validator + clamp (`clampBand` về [0,9] half-step, `clampPercent` về [0,100]).
- `runQuickScore`: schema **không** chứa `pronunciation` trong criteria — server inject `pronunciationBand` đã lock rồi **recompute** overallBand (safety net, line ~358-374).
- Provider-aware calibration:
  - `openai-azure` → `effectiveCompletenessScore = accuracyScore` (vì scripted mode completeness ~100 giả).
  - `deepgram` → dùng `pronunciationCalibration.deepgramSystem` prompt riêng (bỏ qua confidence giả).
- `runWordCefrAnalysis`: lọc wordId hallucinated (chỉ giữ id có trong input), bỏ duplicate.

**Mapping helpers (output → report shape):** `findWordMatchesForText`, `vocabReviewToLexicalHighlights`, `grammarReviewToHighlights`, `buildAnnotationsFromEvaluation`.

### 2.2 Speech Assessment + provider selection — [speech-assessment.ts](../../../features/shared/server/speech-assessment.ts)

```
evaluateSpeechAssessment(audioFile)
   └─ getSpeechAssessmentProvider()   ← đọc env SPEECH_ASSESSMENT_PROVIDER (default "azure")
         ├─ "azure"        → evaluateWithAzureSpeechAssessment        (azure-speech.ts)
         ├─ "openai-azure" → evaluateWithOpenAiAzureSpeechAssessment  (openai-azure-speech.ts)
         └─ "deepgram"     → evaluateWithDeepgramSpeechAssessment     (deepgram-speech.ts)
   →  tất cả trả CÙNG shape AzureSpeechAssessmentResponse (pipeline downstream không phân biệt)
```

Hỗ trợ:
- [azure-speech.ts](../../../features/shared/server/azure-speech.ts): `evaluateWithAzureSpeechAssessment`, `evaluateAzureWithReference` (scripted), `assertAzureSpeechConfigured`, `shouldFallbackFromAzureSpeech`, `AzureSpeechAssessmentError`.
- [deepgram-speech.ts](../../../features/shared/server/deepgram-speech.ts): `evaluateWithDeepgramSpeechAssessment`, `assertDeepgramConfigured`, `DeepgramSpeechAssessmentError`.
- [openai-transcribe.ts](../../../features/shared/server/openai-transcribe.ts): `transcribeWithOpenAI`, `normalizeReferenceText`, `OpenAiTranscribeError`.
- [wav-segmentation.ts](../../../features/shared/server/wav-segmentation.ts): `splitWavIntoSegments`.

### 2.3 OpenAI structured output wrapper — [openai-structured-output.ts](../../../features/shared/server/openai-structured-output.ts)

`requestStructuredOpenAiJson<T>({ engine, apiKey, baseUrl, model, timeoutMs, initialMaxTokens, retryMaxTokens, schema, messages, parseContent })`:
- `engine === "deepseek"` → `response_format: { type: "json_object" }` + prepend system message nhét schema (DeepSeek không hỗ trợ strict `json_schema`).
- Khác → `response_format: { type: "json_schema", json_schema: schema }`.
- `AbortController` theo `timeoutMs`; retry tối đa 2 attempt (attempt 2 dùng `retryMaxTokens` lớn hơn).
- Lỗi → `OpenAiStructuredOutputError`; quy đổi UI-facing qua `resolveAiUserFacingError`.
- Config nguồn: [openai-config.ts](../../../features/shared/server/openai-config.ts) — `resolveAiScoringConfig(overrides)`, `aiGradingEngine` (env `AI_GRADING_ENGINE`), key/baseUrl/model/maxTokens/timeout.
- Prompt: [openai-prompt-config.ts](../../../features/shared/server/openai-prompt-config.ts) (`buildSystemPrompt`, `renderPromptTemplate`, `openAiPromptConfig`) đọc [config/openai-prompts.json](../../../features/shared/server/config/openai-prompts.json); override qua Prompt Lab ([lib/prompt-overrides.ts](../../../features/shared/lib/prompt-overrides.ts)). Limits: [config/vocab-limits.ts](../../../features/shared/server/config/vocab-limits.ts).

### 2.4 NineSpeak grading / quota — [ninespeak-grading.ts](../../../features/shared/server/ninespeak-grading.ts)

```
validateNineSpeakGradingOrThrow({ request, examType })
   └─ token từ header "x-ninespeak-access-token"
        ├─ không token → bypass (canGrade:true, reason:"missing-access-token")
        ├─ canGrade=false → throw RequestGuardError(402)
        └─ NineSpeakApiError 401 → throw RequestGuardError(401, code "ninespeak_auth_expired")

deductNineSpeakGradingOrThrow({ request, examType, isFree })
        ├─ không token → { deducted: false }   (ledger KHÔNG ghi "charged" giả)
        └─ ok → { deducted: true }
```

`examType: NineSpeakGradingExamType = "mocktest" | "practice"`. Charge **state** do ledger module quản lý (không ở shared) — xem [PRD §2(3)](./PRD.md).

> **Gate quota nằm ở đâu (chống hiểu nhầm):** `validate` chạy **trên server tại thời điểm request chấm**, *không* khoá nút ghi âm phía client. UX thực tế: người dùng vẫn ghi âm + bấm chấm → request lên server → nếu hết lượt nhận **402 `grading_quota_exceeded`** → client bắt lỗi này, chuyển attempt sang **trạng thái `pending`** (không phải `failed`) + mở **paywall** (Practice: [usePracticeQuotaPurchaseFlow](../practice/TECH.md); Mock Test: [useMockExamQuotaPurchaseFlow](../mock-test/TECH.md)). Mua xong → `refreshProfile` cộng lượt → tự chấm lại bài pending (đi chung 1 lượt theo ledger `sessionId`).

### 2.5 Request guards — [request-guards.ts](../../../features/shared/server/request-guards.ts)

`RequestGuardError`, `isRequestGuardError`, `assertAudioUpload(...)`, `assertRateLimit(...)`. Route handler module dùng để validate audio + rate-limit trước khi vào pipeline.

### 2.6 Logging & trace lỗi — chi tiết: [docs/logging-and-observability.md](../../logging-and-observability.md)

- **Alpha logging (debug ephemeral)** — [alpha-logger.ts](../../../features/shared/server/alpha-logger.ts) + [alpha-log-runtime.ts](../../../features/shared/server/alpha-log-runtime.ts): `createAlphaLogRecord(input)` → `appendAlphaLog(record)` ghi **buffer in-memory + file JSONL local** (KHÔNG phải Firestore), gated theo `ALPHA_LOG_MODE`. `runWithAlphaLogContext` bọc request (AsyncLocalStorage) → log con kế thừa `sessionId`/`userId`. Client controls: [lib/alpha-log-controls.ts](../../../features/shared/lib/alpha-log-controls.ts) (header `x-alpha-log-session`).
- **Error logging (bền vững)** — [error-logger.ts](../../../features/shared/server/error-logger.ts): `appendAlphaLog` tự gọi `recordErrorLog` cho mọi bản ghi error/warn (độc lập alpha mode) → ghi **Firestore `error_logs`** fire-and-forget (rate-limit + TTL) + mirror `console.error` ra **Vercel Logs**. Là kênh trace lỗi production. Bật/tắt: `ERROR_LOG_FIRESTORE`.

### 2.7 NineSpeak client + BFF clients (client-side)

- [lib/ninespeak-client.ts](../../../features/shared/lib/ninespeak-client.ts): `createNineSpeakClient`, singleton `nineSpeakClient`, sub-API (`authApi`, `gradingApi`, `mocktest*Api`, `practice*Api`, `paymentsApi`...), storage helpers (`getStoredNineSpeakSession`, `setStoredNineSpeakSession`, `clearStoredNineSpeakSession`), `NineSpeakApiError`, `isOnboarded`.
  - **Refresh token transparent (chống "mở web hôm sau bị đăng nhập lại"):** access token TTL ngắn (~1 ngày). `request()` tự xử lý 401 ở **mọi** API call: gọi `POST /auth/refresh` (single-flight — nhiều 401 đồng thời chỉ refresh 1 lần) lấy cặp token mới, persist localStorage, rồi **retry đúng 1 lần** với token mới — TRƯỚC khi `onUnauthorized` xoá phiên. Cover cả bootstrap (`/user/init`) lẫn 401 **giữa phiên** (tab mở qua đêm rồi thao tác, không cần reload). Chỉ khi refresh token cũng hết hạn/invalid mới clear + fallback đăng nhập Google → buộc login.
    - Config: `nineSpeakClient` truyền `getRefreshToken` + `persistRefreshedSession`. Tránh đệ quy/loop bằng `skipAuthRefresh` (cho `/auth/refresh`, `/auth/google`, `/auth/logout`) và `isRetryAfterRefresh` (lần retry không refresh nữa). `authApi.refresh(refreshToken)` là helper public.
    - **Đồng bộ React state:** `persistRefreshedSession` gọi `notifyNineSpeakSessionRefreshed` → `AuthProvider` đăng ký `onNineSpeakSessionRefreshed` để `setSession` token mới (tránh consumer cứ gửi token cũ → refresh lặp). `ensureAuthenticatedUser` đọc lại session mới nhất sau `init` để trả về đúng session.
    - Client server-side (`serverNineSpeakClient` không config) **không** bật transparent refresh — đúng vì server nhận token qua header, không có localStorage.
- [lib/api/company-api-client.ts](../../../features/shared/lib/api/company-api-client.ts): **re-export** từ `ninespeak-client` (entry-point gọn).
- [lib/api/app-bff-client.ts](../../../features/shared/lib/api/app-bff-client.ts): **re-export** từ [lib/app-data-client.ts](../../../features/shared/lib/app-data-client.ts) — `fetchDashboardBootstrap`, `fetchPracticeCatalog`, `fetchUserProfile`, `upsertExamSession`, `savePracticeSession`...
- [lib/history-client.ts](../../../features/shared/lib/history-client.ts): `listHydratedRecentSessions`, `listHydratedHistoryEntries`, `fetchHydratedMockExamSession`, `fetchHydratedPracticeHistoryAttempt` — gộp Mock Test + Practice từ NineSpeak rồi sắp theo thời gian (dùng cho `/history`, `/hieu-suat`).

### 2.8 Content store — [question-store.ts](../../../features/shared/server/question-store.ts), [prompt-resolvers.ts](../../../features/shared/server/prompt-resolvers.ts), [data/](../../../features/shared/data/mock-exam-definition.ts)

Server đọc [part1.json](../../../features/shared/server/part1.json)/[part2.json](../../../features/shared/server/part2.json)/[part3.json](../../../features/shared/server/part3.json) → `MockExamDefinition`, sample answer theo band, vocabulary bucket. `prompt-resolvers` cung cấp prompt/định nghĩa cho pipeline. Client dùng `data/` (`getMockExamDefinition`, flow-mode getters).

### 2.9 Providers / Hooks / Components

| Loại | Export chính | File |
|---|---|---|
| Provider | `AuthProvider` + `useAuth` | [providers/AuthProvider.tsx](../../../features/shared/providers/AuthProvider.tsx) |
| Provider | `AnalyticsProvider` | [providers/AnalyticsProvider.tsx](../../../features/shared/providers/AnalyticsProvider.tsx) |
| Provider | `FeedbackSurveyProvider` + `useFeedbackSurvey` (luồng exit-intent hiện TẮT — cờ `EXIT_INTENT_SURVEY_ENABLED`) | [providers/FeedbackSurveyProvider.tsx](../../../features/shared/providers/FeedbackSurveyProvider.tsx) |
| Provider | `ClientDiagnosticsProvider` | [providers/ClientDiagnosticsProvider.tsx](../../../features/shared/providers/ClientDiagnosticsProvider.tsx) |
| Hook | `useMockExamRecorder` | [hooks/useMockExamRecorder.ts](../../../features/shared/hooks/useMockExamRecorder.ts) |
| Hook | `useQuestionAudio` | [hooks/useQuestionAudio.ts](../../../features/shared/hooks/useQuestionAudio.ts) |
| Hook | `useMockExamReportInteractions` | [hooks/useMockExamReportInteractions.ts](../../../features/shared/hooks/useMockExamReportInteractions.ts) |
| Hook | `useReducedMotionPreference` | [hooks/useReducedMotionPreference.ts](../../../features/shared/hooks/useReducedMotionPreference.ts) |
| Hook | `useRequireTargetBand` | [hooks/useRequireTargetBand.tsx](../../../features/shared/hooks/useRequireTargetBand.tsx) |
| Component | `AudioWaveBars`, `AssistantUnavailablePanel`, `BandScorePromptDialog`, `PromptOverrideWorkbench` | [components/](../../../features/shared/components/AudioWaveBars.tsx) |
| Component | `FeedbackSurveyWidget` (UI khảo sát, nạp ở `app/providers.tsx`; còn `ExitIntentCard` giữ lại nhưng đang tắt) | [components/feedback/FeedbackSurveyWidget.tsx](../../../features/shared/components/feedback/FeedbackSurveyWidget.tsx) |
| Component (report kit) | `MockExamReportChrome`, `MockExamReportNavigation`, `MockExamReportSections`, `PauseMarker` | [components/reports/](../../../features/shared/components/reports/MockExamReportChrome.tsx) |

---

## 3. Contract types — ai phụ thuộc

| File | Đại diện | Người tiêu thụ |
|---|---|---|
| [types/app-api.ts](../../../features/shared/types/app-api.ts) | `UserProfile`, `SessionSummary`, `DashboardBootstrapResponse`, `PracticeTopicSummary`, onboarding... | client BFF (`app-data-client`) + Onboarding/Dashboard server (`/api/app/*`) |
| [types/ninespeak-api.ts](../../../features/shared/types/ninespeak-api.ts) | `NineSpeakApiResponse`, `NineSpeakGrading*`, `NineSpeak{Mocktest,Practice}*`, `NineSpeakPayment*`, `NineSpeakAuthSession` | `ninespeak-client`, `ninespeak-grading`, persistence từng module |
| [types/mock-exam.ts](../../../features/shared/types/mock-exam.ts) | `MockExamDefinition`, `MockExamSession`, `MockExamSpeakingCriteria`, `AzureSpeechAssessmentResponse`, `MockExam*Highlight/Annotation`, `MockExamCefrDistribution` | Mock Test + pipeline mapping + speech assessment |
| [types/practice-evaluation.ts](../../../features/shared/types/practice-evaluation.ts) | `IELTSCriteriaScores`, `PracticeEvaluationResult`, `PracticeCorrections`, `Grammar*`/`Vocab*Subtype`, review status | Practice + pipeline |

> Quy ước: nguồn sự thật là `shared/types`. Mock Test/Practice **re-export qua shim** nội bộ — đổi type ở shared lan toả tới cả hai. Vì vậy mọi đổi type phải **additive** (field optional) để giữ backward-compat.

---

## 4. Ràng buộc boundary

- **`shared` KHÔNG import feature module** — ESLint `blockModule(...)` cho `features/shared/**` ([eslint.config.mjs](../../../eslint.config.mjs) line 43-50). Vi phạm → `npm run lint` fail.
- **Toolbox nhiều entry-point, KHÔNG barrel tổng** — module import sâu trực tiếp file cần dùng. Không tạo `features/shared/index.ts` gom tất cả.
- **Server-only tách bạch** — file `server/` mở đầu `import "server-only"`; không import vào client component (sẽ leak secret + lỗi build).
- **Ledger không ở shared** — charge state Firestore là của module (collection riêng `mocktest_scoring_charges` / `practice_scoring_charges`); shared chỉ cấp deduct/validate primitive.

---

## 5. Biến môi trường liên quan

`shared/server` là nơi đọc thực sự của hầu hết secret. Chi tiết bản đồ env ↔ module: [docs/env-and-modules.md](../../env-and-modules.md). Tóm tắt:

| Nhóm | Biến chính | Đọc tại |
|---|---|---|
| AI / LLM | `AI_GRADING_ENGINE`, `OPENAI_API_KEY`, `DEEPSEEK_API_KEY`, `AI_SCORING_MODEL`, `AI_SCORING_MAX_TOKENS`, `AI_SCORING_RETRY_MAX_TOKENS`, `AI_SCORING_TIMEOUT_MS`, `OPENAI_TRANSCRIPTION_MODEL` | `server/openai-config.ts`, `openai-transcribe.ts` |
| Speech | `SPEECH_ASSESSMENT_PROVIDER`, `AZURE_SPEECH_KEY/REGION/LANGUAGE`, `DEEPGRAM_API_KEY/MODEL/LANGUAGE` | `speech-assessment.ts`, `azure-speech.ts`, `deepgram-speech.ts` |
| Flags / pricing | `MOCK_EXAM_PART13_FLOW_MODE`, `PRACTICE_EVALUATION_PROVIDER`, `PRACTICE_TOPUP_*_VND`, `NEXT_PUBLIC_NINESPEAK_API_*` | `lib/feature-flags.ts`, `server/pricing-config.ts` |
| Analytics / log | `NEXT_PUBLIC_MIXPANEL_*`, `NEXT_PUBLIC_GA4_*`, `ALPHA_LOG_MODE` | `lib/analytics.ts`, `server/alpha-log-runtime.ts` |

> Module **không nên** đọc `process.env` trực tiếp — thêm cờ qua `lib/feature-flags.ts` rồi tiêu thụ giá trị.

---

## 6. Server-only constraints

- Tất cả file `server/` chứa secret/SDK → `import "server-only"` (ví dụ [evaluation-pipeline.ts](../../../features/shared/server/evaluation-pipeline.ts) line 1).
- KHÔNG import `server/*` từ component `"use client"`, hook client, hay `lib/*` client-safe.
- Khi cần dữ liệu server xuống client → đi qua route handler (`/api/app/*`, `/api/speaking/*`) với contract type ở `types/`.

---

## 7. Gotchas & nơi cần sửa khi mở rộng

| Tình huống | Nơi sửa | Lưu ý backward-compat |
|---|---|---|
| Thêm tiêu chí / field vào kết quả chấm điểm | [evaluation-pipeline.ts](../../../features/shared/server/evaluation-pipeline.ts) + schema + parser + [types/practice-evaluation.ts](../../../features/shared/types/practice-evaluation.ts)/[types/mock-exam.ts](../../../features/shared/types/mock-exam.ts) | Field mới **optional**; cả Mock Test + Practice nhận thay đổi cùng lúc. |
| Đổi prompt chấm điểm | [config/openai-prompts.json](../../../features/shared/server/config/openai-prompts.json) (qua [openai-prompt-config.ts](../../../features/shared/server/openai-prompt-config.ts)) | Whitespace system prompt bị collapse 1 dòng khi gửi; userTemplate giữ format. Override Prompt Lab không persist. |
| Thêm speech provider | [speech-assessment.ts](../../../features/shared/server/speech-assessment.ts) (branch mới) + adapter trả `AzureSpeechAssessmentResponse` | Phải trả đúng shape chung; cân nhắc calibration provider-aware (như openai-azure/deepgram). |
| Đổi engine LLM | [openai-config.ts](../../../features/shared/server/openai-config.ts) + wrapper deepseek branch | DeepSeek dùng `json_object` + schema-in-system; giữ 2 lớp ép JSON. |
| Đổi contract API 9Speak | [types/ninespeak-api.ts](../../../features/shared/types/ninespeak-api.ts) + [ninespeak-client.ts](../../../features/shared/lib/ninespeak-client.ts) | Mọi module persistence phụ thuộc — additive only. |
| Đổi quota/grading logic | [ninespeak-grading.ts](../../../features/shared/server/ninespeak-grading.ts) | Giữ contract `{ deducted }`; ledger module phụ thuộc semantics "không token → deducted=false". |
| Đổi shape report UI | mapping helpers trong [evaluation-pipeline.ts](../../../features/shared/server/evaluation-pipeline.ts) + [components/reports/](../../../features/shared/components/reports/MockExamReportSections.tsx) | Mock Test report consume trực tiếp; kiểm tra cả history cũ (legacy fallback). |

**Nhấn mạnh:** vì **mọi feature module phụ thuộc `shared`**, mọi thay đổi core phải **tương thích ngược**. Đổi signature/shape không tương thích = gãy build/runtime nhiều module cùng lúc. Khi bắt buộc breaking change → làm thành đợt refactor hạ tầng có chủ đích, cập nhật toàn bộ consumer trong cùng PR, chạy `npm run build` + `npx tsc --noEmit` + `npm run lint`.

---

## 8. Liên kết

- Năng lực cấp sản phẩm → [PRD.md](./PRD.md)
- Quy tắc module → [docs/README-modules.md](../../README-modules.md)
- Context lõi (AI) → [features/shared/AGENTS.md](../../../features/shared/AGENTS.md)
- Pipeline nghiệp vụ chi tiết → [docs/scoring-pipeline-overview.md](../../scoring-pipeline-overview.md)
- Env ↔ module → [docs/env-and-modules.md](../../env-and-modules.md)
