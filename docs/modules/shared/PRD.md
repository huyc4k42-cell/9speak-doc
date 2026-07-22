# Shared Core — Catalog Năng lực Nền tảng (PRD)

> **Lưu ý về thể loại tài liệu:** `shared/` là **lõi hạ tầng**, không phải tính năng người dùng. Vì vậy "PRD" ở đây = **mô tả NĂNG LỰC / DỊCH VỤ** mà `shared` cung cấp cho các feature module (Mock Test, Practice, Billing, Onboarding/Dashboard), kèm input/output, người tiêu thụ và ràng buộc. Đặc tả kỹ thuật chi tiết (hàm, luồng, file) nằm ở [TECH.md](./TECH.md).
>
> Đối tượng đọc: AI agent + người đầu mối module. Mục tiêu: hiểu `shared` cung cấp gì, dùng thế nào, và **ràng buộc liên-module** trước khi phát triển / sửa lỗi.

---

## 1. Vai trò & Nguyên tắc

`features/shared/` là **toolbox nhiều entry-point** gom toàn bộ hạ tầng kỹ thuật, contract type, nội dung và UI/hook/provider dùng chung giữa các module. Nguyên tắc cốt lõi:

| Nguyên tắc | Nội dung | Cơ chế enforce |
|---|---|---|
| **Lõi không phụ thuộc tính năng** | `shared/` **KHÔNG** được import bất kỳ feature module nào (`mock-test`, `practice`, `billing`, `onboarding-dashboard`). | ESLint `no-restricted-imports` — [eslint.config.mjs](../../../eslint.config.mjs) (block 4 module cho `features/shared/**`) |
| **Read-only với module owner** | Owner module **không tự ý sửa** `shared/`. Đổi core phải qua đợt refactor hạ tầng. | Quy ước + [.github/CODEOWNERS](../../../.github/CODEOWNERS) |
| **Tương thích ngược bắt buộc** | Mọi thay đổi `shared/` phải backward-compatible vì **mọi module phụ thuộc**. | Review + nguyên tắc |
| **Toolbox, không barrel tổng** | Không có `index.ts` gom tất cả. Module import **sâu trực tiếp** vào file cần dùng (vd `@/features/shared/lib/band-score`). | Cấu trúc thư mục |
| **Server-only tách bạch** | File trong `server/` chứa secret/SDK — bắt đầu bằng `import "server-only"`, cấm import vào client component. | `server-only` package |

Sơ đồ vị trí trong repo: xem [docs/README-modules.md](../../README-modules.md) và [features/shared/AGENTS.md](../../../features/shared/AGENTS.md).

---

## 2. Danh mục Dịch vụ / Năng lực

### (1) Pipeline chấm điểm (Evaluation Pipeline)

Bộ 4 bước LLM chấm điểm IELTS Speaking, **dùng chung** Mock Test + Practice. File: [evaluation-pipeline.ts](../../../features/shared/server/evaluation-pipeline.ts).

| Bước | Hàm | Input | Output | Ràng buộc |
|---|---|---|---|---|
| **Pronunciation Calibration** | `runPronunciationCalibration` | Azure aggregate scores (accuracy/fluency/prosody/completeness/pronunciation) + transcript stats + top weak words | `{ band 0–9, rationale (VN) }` | Gọi **trước** quickScore. Band kết quả **lock** cho 3 bước sau. **Fail-fast**: LLM throw → propagate (không fallback Azure cứng). |
| **Quick Score** | `runQuickScore` | transcript + metrics + `pronunciationBand` (đã lock) | `overallBand`, `criteria` (4 tiêu chí), `cefrLevel`, `feedbackSummary`, `improvements` | Chấm 3 tiêu chí Fluency/Lexical/Grammar; **cấm rescore pronunciation**. Server **recompute** overallBand từ 4 criteria để chống drift. |
| **Vocab Review** | `runVocabReview` | transcript + `lockedScore` + limits | `lexicalErrors` (≤10), `lexicalSuggestions` (≤4), `cefrDistribution` | **Không rescore band.** |
| **Grammar Review** | `runGrammarReview` | transcript + `lockedScore` | `grammarErrors`, `complexStructureErrors`, `errorSummary`, `complexStructureUsage`, `generalFeedback` | **Không rescore band.** |
| **Word CEFR (phụ)** | `runWordCefrAnalysis` | transcript + danh sách wordId | `wordCefrLevels` map | Background, fire-and-forget; caller **phải fail-safe** (lỗi không ảnh hưởng UI chính). |

**Ai dùng:** `features/practice/server/evaluate-practice.ts`, `features/mock-test/server/evaluate-mock-exam-segmented.ts`. Tổng quan luồng nghiệp vụ: [docs/scoring-pipeline-overview.md](../../scoring-pipeline-overview.md).

**Helper mapping** (output pipeline → shape report UI): `vocabReviewToLexicalHighlights`, `grammarReviewToHighlights`, `buildAnnotationsFromEvaluation`, `findWordMatchesForText` — cùng file.

### (2) Speech Processing (Pronunciation Assessment + Transcribe)

Lớp seam thống nhất: mọi engine trả về cùng shape `AzureSpeechAssessmentResponse` nên pipeline phía sau không cần biết provider.

| Năng lực | Hàm / File | Provider | Ghi chú |
|---|---|---|---|
| **Provider selection** | `evaluateSpeechAssessment`, `getSpeechAssessmentProvider` — [speech-assessment.ts](../../../features/shared/server/speech-assessment.ts) | chọn theo env `SPEECH_ASSESSMENT_PROVIDER` (default `azure`) | seam dùng chung Practice + Mock Test |
| **Azure Pronunciation Assessment** | `evaluateWithAzureSpeechAssessment` — [azure-speech.ts](../../../features/shared/server/azure-speech.ts) | Azure Cognitive Services Speech | accuracy/fluency/completeness/prosody + phoneme-level |
| **OpenAI+Azure (scripted)** | `evaluateWithOpenAiAzureSpeechAssessment` — [openai-azure-speech.ts](../../../features/shared/server/openai-azure-speech.ts) | OpenAI transcribe → Azure scripted | calibration trung hòa completeness về accuracy |
| **Deepgram STT** | `evaluateWithDeepgramSpeechAssessment` — [deepgram-speech.ts](../../../features/shared/server/deepgram-speech.ts) | Deepgram | STT fallback; calibration dùng system prompt riêng (bỏ qua confidence) |
| **OpenAI transcribe** | `transcribeWithOpenAI` — [openai-transcribe.ts](../../../features/shared/server/openai-transcribe.ts) | OpenAI Whisper | reference text cho scripted assessment |
| **WAV segmentation** | `splitWavIntoSegments` — [wav-segmentation.ts](../../../features/shared/server/wav-segmentation.ts) | — | cắt/chuẩn hoá WAV trước khi gửi API |

**Ai dùng:** evaluator của Mock Test + Practice (qua `evaluateSpeechAssessment`).

### (3) NineSpeak Client + Quota / Grading

| Năng lực | Hàm / File | Input → Output | Ai dùng |
|---|---|---|---|
| **NineSpeak API client (client-side)** | `createNineSpeakClient`, `nineSpeakClient`, các sub-API (`authApi`, `gradingApi`, `mocktest*Api`, `practice*Api`, `paymentsApi`...) — [ninespeak-client.ts](../../../features/shared/lib/ninespeak-client.ts) | persistence/history/report/payment qua backend 9Speak | tất cả module (qua re-export [api/company-api-client.ts](../../../features/shared/lib/api/company-api-client.ts)) |
| **Quota validate (server)** | `validateNineSpeakGradingOrThrow` — [ninespeak-grading.ts](../../../features/shared/server/ninespeak-grading.ts) | `{ request, examType }` → grading state; throw `RequestGuardError(402)` khi hết lượt; `401` → mã `ninespeak_auth_expired` | route handler Mock Test + Practice (kiểm tra **trước** khi gọi AI đắt) |
| **Quota deduct (server)** | `deductNineSpeakGradingOrThrow` — [ninespeak-grading.ts](../../../features/shared/server/ninespeak-grading.ts) | `{ request, examType, isFree }` → `{ deducted }`. Không token → `deducted=false` (ledger không ghi "charged" giả) | charge ledger từng module |
| **Session refresh** | [ninespeak-session-refresh.ts](../../../features/shared/lib/ninespeak-session-refresh.ts) | refresh token khi 401 | AuthProvider / client |

> **Ledger thuộc module, KHÔNG thuộc shared:** Firestore charge ledger nằm ở [mock-test/server/mocktest-charge-ledger.ts](../../../features/mock-test/server/mocktest-charge-ledger.ts) và [practice/server/practice-charge-ledger.ts](../../../features/practice/server/practice-charge-ledger.ts) (collection riêng để tránh xung đột state). `shared` chỉ cung cấp **deduct/validate primitive**.

### (4) Content Store (đề bài & ngân hàng câu hỏi)

| Năng lực | File | Cung cấp |
|---|---|---|
| **Question store (server)** | [question-store.ts](../../../features/shared/server/question-store.ts) + [part1.json](../../../features/shared/server/part1.json) / [part2.json](../../../features/shared/server/part2.json) / [part3.json](../../../features/shared/server/part3.json) | định nghĩa câu hỏi Part 1/2/3, sample answer theo band, vocabulary bucket |
| **Prompt resolvers** | [prompt-resolvers.ts](../../../features/shared/server/prompt-resolvers.ts) | `resolvePracticePrompt`, `resolveMockExamPrompt`, `resolveMockExamDefinition`, `resolveMockExamQuestion` |
| **Mock exam definition (client)** | [data/mock-exam-definition.ts](../../../features/shared/data/mock-exam-definition.ts), [data/mock-exam.ts](../../../features/shared/data/mock-exam.ts), [data/mock.ts](../../../features/shared/data/mock.ts) | `getMockExamDefinition`, flow-mode getters, TTS flags |
| **Question bank** | [data/mock-exam-question-bank.json](../../../features/shared/data/mock-exam-question-bank.json) | ngân hàng câu hỏi mock exam |

**Ai dùng:** Mock Test + Practice (đề bài, sample answer, prompt cho LLM).

### (5) Auth & Providers (cross-cutting)

| Provider | File | Cung cấp |
|---|---|---|
| **AuthProvider** | [providers/AuthProvider.tsx](../../../features/shared/providers/AuthProvider.tsx) | `useAuth()` — Firebase auth + NineSpeak session, bearer token |
| **AnalyticsProvider** | [providers/AnalyticsProvider.tsx](../../../features/shared/providers/AnalyticsProvider.tsx) | init Mixpanel/GA4, page-view tracking |
| **FeedbackSurveyProvider** | [providers/FeedbackSurveyProvider.tsx](../../../features/shared/providers/FeedbackSurveyProvider.tsx) | `useFeedbackSurvey()` — khảo sát feedback (UI: [components/feedback/FeedbackSurveyWidget.tsx](../../../features/shared/components/feedback/FeedbackSurveyWidget.tsx)). Chỉ còn luồng **result-trigger**; luồng **exit-intent** hiện TẮT (cờ `EXIT_INTENT_SURVEY_ENABLED`). |
| **ClientDiagnosticsProvider** | [providers/ClientDiagnosticsProvider.tsx](../../../features/shared/providers/ClientDiagnosticsProvider.tsx) | capture lỗi/diagnostics client |

**Ai dùng:** lắp ở composition root (Onboarding/Dashboard) + mọi page cần auth/analytics.

### (6) Analytics & Logging

| Năng lực | File | Cung cấp |
|---|---|---|
| **Analytics core** | [lib/analytics.ts](../../../features/shared/lib/analytics.ts) + [lib/analytics-events/](../../../features/shared/lib/analytics-events/index.ts) | `trackDefinedAnalyticsEvent`, `identifyAnalyticsUser`, schema sự kiện per-domain (dashboard, login, mock-test, onboarding, payment, practice, feedback) |
| **Marketing / converters / timing / counters** | [analytics-marketing.ts](../../../features/shared/lib/analytics-marketing.ts), [analytics-converters.ts](../../../features/shared/lib/analytics-converters.ts), [analytics-timing.ts](../../../features/shared/lib/analytics-timing.ts), [analytics-counters.ts](../../../features/shared/lib/analytics-counters.ts) | tracking phụ trợ |
| **Alpha logging (debug ephemeral)** | [server/alpha-logger.ts](../../../features/shared/server/alpha-logger.ts), [server/alpha-log-runtime.ts](../../../features/shared/server/alpha-log-runtime.ts) | `appendAlphaLog`, `createAlphaLogRecord`, `runWithAlphaLogContext` — ghi **buffer in-memory + file local** (gated `ALPHA_LOG_MODE`), context qua AsyncLocalStorage |
| **Error logging (bền vững)** | [server/error-logger.ts](../../../features/shared/server/error-logger.ts) | `recordErrorLog` (auto gọi từ `appendAlphaLog` cho error/warn) → Firestore `error_logs` fire-and-forget + Vercel `console.error`; trace lỗi production. Xem [logging-and-observability.md](../../logging-and-observability.md) |
| **Alpha log controls (client)** | [lib/alpha-log-controls.ts](../../../features/shared/lib/alpha-log-controls.ts), [lib/alpha-log-client.ts](../../../features/shared/lib/alpha-log-client.ts) | header `x-alpha-log-session`, bật/tắt logging |

> **Ràng buộc PII:** email/name **chỉ** gửi Mixpanel, không GA4 (xem auto-memory). Khi mở rộng analytics, tuân thủ ràng buộc này.

**Ai dùng:** tất cả module + route handler (alpha log để debug pipeline).

### (7) Storage / CEFR / Band Utils

| Năng lực | File | Cung cấp |
|---|---|---|
| **Band score** | [lib/band-score.ts](../../../features/shared/lib/band-score.ts) | `roundBandScore`, `TARGET_BAND_OPTIONS` |
| **CEFR mapping / colors / distribution** | [lib/cefr-mapping.ts](../../../features/shared/lib/cefr-mapping.ts), [lib/cefr-colors.ts](../../../features/shared/lib/cefr-colors.ts), [lib/cefr-distribution.ts](../../../features/shared/lib/cefr-distribution.ts) | `cefrFromOverallBand`, màu CEFR cho UI |
| **Pronunciation band (reference)** | [lib/pronunciation-band.ts](../../../features/shared/lib/pronunciation-band.ts) | `calculateIeltsPronunciationBand` — **chỉ log reference**, không còn fallback |
| **Audio upload** | [lib/audio-presigned-upload.ts](../../../features/shared/lib/audio-presigned-upload.ts), [lib/audio-upload-client.ts](../../../features/shared/lib/audio-upload-client.ts), [lib/audio-upload-mode.ts](../../../features/shared/lib/audio-upload-mode.ts) | presigned upload R2 (audio recording) |
| **Pause / transcript stats** | [lib/transcript-pause-classifier.ts](../../../features/shared/lib/transcript-pause-classifier.ts), [lib/transcript-pause-stats.ts](../../../features/shared/lib/transcript-pause-stats.ts), [lib/azure-pronunciation.ts](../../../features/shared/lib/azure-pronunciation.ts) | phân loại good/bad pause, filler count |
| **Top-up / pricing / subscription** | [lib/top-up-packs.ts](../../../features/shared/lib/top-up-packs.ts), [server/pricing-config.ts](../../../features/shared/server/pricing-config.ts), [lib/subscription-plans.ts](../../../features/shared/lib/subscription-plans.ts) | gói lẻ lượt chấm, giá VND động qua env |

### (8) Contract Types

| File | Hợp đồng | Ai phụ thuộc |
|---|---|---|
| [types/app-api.ts](../../../features/shared/types/app-api.ts) | request/response của BFF `/api/app/*` (UserProfile, SessionSummary, Dashboard, Practice catalog...) | mọi module gọi BFF + Onboarding/Dashboard (server) |
| [types/ninespeak-api.ts](../../../features/shared/types/ninespeak-api.ts) | hợp đồng API 9Speak (auth, grading, mocktest/practice persistence, payment) | ninespeak-client + grading + module persistence |
| [types/mock-exam.ts](../../../features/shared/types/mock-exam.ts) | shape mock exam (session, criteria, transcript annotation, highlight...) | Mock Test + pipeline output mapping |
| [types/practice-evaluation.ts](../../../features/shared/types/practice-evaluation.ts) | shape practice (criteria, corrections, rewrite, review status...) | Practice + pipeline |

> Mock Test / Practice **re-export** từ các type này qua shim nội bộ — nguồn sự thật là `shared/types`.

---

## 3. Quy tắc Tiêu thụ (cho module owner)

**Được phép:**
- Import **sâu trực tiếp** vào file `shared` cần dùng: `@/features/shared/lib/...`, `@/features/shared/server/...`, `@/features/shared/types/...`, `@/features/shared/hooks/...`, `@/features/shared/components/...`, `@/features/shared/providers/...`, `@/features/shared/data/...`.
- Gọi pipeline (`runQuickScore`, `runVocabReview`...) trong evaluator của module mình, **bọc `try-catch`**, ghi `appendAlphaLog`, trả mã lỗi thân thiện cho client.
- Tái dùng contract type, content store, util band/CEFR thay vì tự định nghĩa lại.

**KHÔNG được phép:**
- Sửa file trong `shared/` mà không đảm bảo backward-compat.
- Import file `server/` (server-only) vào client component.
- Kỳ vọng `shared` import ngược feature module (ESLint chặn — sẽ fail `npm run lint`).
- Nhồi logic nghiệp vụ riêng module vào `shared` (giữ lõi trung lập).

**Khi cần năng lực mới từ shared:** nêu rõ trong PR, thiết kế additive (thêm hàm/field optional), không đổi chữ ký cũ. Nếu chỉ là cờ cấu hình → thêm vào [lib/feature-flags.ts](../../../features/shared/lib/feature-flags.ts) rồi tiêu thụ giá trị (không `process.env` trực tiếp ở module).

---

## 4. SLA / Đảm bảo

| Đảm bảo | Nội dung |
|---|---|
| **Fail-fast pipeline** | Pronunciation calibration fail → propagate error (không fallback band Azure sai lệch). quickScore fail → throw cả request. Vocab/Grammar fail **độc lập** từng bước (module quyết retry per-tab). |
| **Lock-once scoring** | Chỉ `runQuickScore` quyết band; vocab/grammar nhận `lockedScore`, **cấm rescore** → không trôi điểm. |
| **Idempotent quota** | `deductNineSpeakGradingOrThrow` không có token → `deducted=false`, không ghi charge giả. State doc do ledger module quản lý (2-phase commit lite). |
| **Provider-agnostic speech** | Mọi engine trả cùng `AzureSpeechAssessmentResponse`; đổi provider qua env, rollback tức thì. |
| **Backward-compat bắt buộc** | Mọi đổi `shared` phải tương thích ngược — mọi module phụ thuộc. Field mới = optional; hàm mới = thêm, không đổi signature cũ. |
| **Server-only an toàn** | Secret/SDK chỉ trong `server/` (`import "server-only"`), không leak xuống bundle client. |

---

## 5. Liên kết

- Đặc tả kỹ thuật chi tiết → [TECH.md](./TECH.md)
- Quy tắc module & ranh giới → [docs/README-modules.md](../../README-modules.md)
- Context lõi (AI) → [features/shared/AGENTS.md](../../../features/shared/AGENTS.md)
- Pipeline chấm điểm (nghiệp vụ) → [docs/scoring-pipeline-overview.md](../../scoring-pipeline-overview.md)
- Env ↔ module → [docs/env-and-modules.md](../../env-and-modules.md)
