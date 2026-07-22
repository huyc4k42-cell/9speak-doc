# Biến môi trường & Module hóa (Env & Modules)

> Mục tiêu tài liệu: trả lời câu hỏi *"tách module có cần tách biến env không?"* và cung cấp bản đồ biến env ↔ module để mỗi đầu mối biết biến nào ảnh hưởng tới module mình. **Không thay đổi cách tổ chức env hiện tại** — đây là tài liệu rà soát + đề xuất.

---

## 1. Kết luận nhanh

**KHÔNG cần tách biến env theo module.** Sau khi module hóa, gần như toàn bộ env được đọc tập trung ở **`features/shared/` (lõi)** và **`lib/` (hạ tầng repo: Firebase, Cloudflare R2, build-meta)** — các *feature module* (`mock-test`, `practice`, `billing`, `onboarding-dashboard`) **hầu như không đọc `process.env` trực tiếp**. Chúng nhận hành vi thông qua dịch vụ của `shared` (evaluation-pipeline, ninespeak-client, speech SDK, pricing-config…) chứ không tự đọc cấu hình.

Vì người tiêu thụ env đã tập trung, việc cắt env thành file-theo-module sẽ **giả tạo và rủi ro** (cùng một key như `OPENAI_API_KEY` phục vụ cả Mock Test lẫn Practice qua pipeline chung). Cách tổ chức hiện tại — **theo flavor triển khai** — là phù hợp và **giữ nguyên để tương thích ngược**.

---

## 2. Cách env đang được tổ chức & nạp

Tổ chức theo **flavor (môi trường triển khai)**, không theo module:

| File | Vai trò |
|---|---|
| `env/shared.env` | Biến **dùng chung GIỮA CÁC FLAVOR** (staging & production): khoá AI, Azure, R2, Mixpanel… |
| `env/staging.env` | Ghi đè/riêng cho staging |
| `env/production.env` | Ghi đè/riêng cho production |
| `env/*.env.example` | Bản mẫu (commit được, không chứa secret) |

Nạp bằng `dotenv-cli`, **file sau ghi đè file trước**:
```bash
npm run dev          # dotenv -e env/staging.env -e env/shared.env -- next dev
npm run build:prod   # dotenv -e env/production.env -e env/shared.env -- build-with-meta
npm run env:push:staging   # đẩy shared.env + staging.env lên Vercel
```

> ⚠️ **Lưu ý đặt tên:** `shared.env` ở đây nghĩa là *"chung giữa các flavor"*, **không** phải *"module shared"* (`features/shared/`). Hai khái niệm khác nhau — đừng nhầm.

---

## 3. Bản đồ biến env → tầng đọc → module bị ảnh hưởng

Cột **"Đọc tại"**: tầng thực sự gọi `process.env`. Cột **"Ảnh hưởng module"**: module mà hành vi của biến tác động tới (để owner biết, dù biến được đọc ở `shared`).

### 3.1 Chấm điểm AI / LLM — đọc tại `features/shared/server`
| Biến | Ảnh hưởng | Ghi chú |
|---|---|---|
| `AI_GRADING_ENGINE` | mock-test, practice | chọn engine (openai/deepseek) |
| `OPENAI_API_KEY`, `OPENAI_API_BASE_URL`, `OPENAI_BASE_URL` | mock-test, practice | khoá/endpoint OpenAI |
| `DEEPSEEK_API_KEY` | mock-test, practice | khoá DeepSeek (engine thay thế) |
| `AI_SCORING_MODEL`, `AI_SCORING_MAX_TOKENS`, `AI_SCORING_RETRY_MAX_TOKENS`, `AI_SCORING_TIMEOUT_MS` | mock-test, practice | tham số gọi LLM (pipeline chung) |
| `OPENAI_TRANSCRIPTION_MODEL` | mock-test, practice | Whisper transcribe |

### 3.2 Phân tích giọng nói — đọc tại `features/shared/server`
| Biến | Ảnh hưởng | Ghi chú |
|---|---|---|
| `AZURE_SPEECH_KEY`, `AZURE_SPEECH_REGION`, `AZURE_SPEECH_LANGUAGE` | mock-test, practice | Azure Pronunciation Assessment |
| `SPEECH_ASSESSMENT_PROVIDER` | mock-test, practice | chọn provider speech (đọc ở `speech-assessment.ts`) |
| `DEEPGRAM_API_KEY`, `DEEPGRAM_MODEL`, `DEEPGRAM_LANGUAGE` | mock-test, practice | STT Deepgram (fallback CŨ + luồng chấm MỚI của practice) |
| `PRACTICE_PIPELINE` | **CHỈ practice** | `deepgram` → luồng chấm MỚI (Deepgram transcribeUrl → DeepSeek calibrate+quickScore, Azure chạy ngầm sau band, fallback lazy khi mở tab Phát âm). Trống → baseline Whisper+Azure. Rollback tức thì. KHÔNG ảnh hưởng mock-test. Đọc ở `practice-route-handlers.ts` |
| `PRACTICE_PIPELINE_DEBUG_ENABLED` | practice (debug) | `true` → bật `/debug/practice-calibration-bench` (hiệu chỉnh calibrate). Off ở production |
| `PRACTICE_EVALUATION_PROVIDER`, `NEXT_PUBLIC_PRACTICE_EVALUATION_PROVIDER` | practice | ⚠️ legacy — code KHÔNG còn đọc (có thể xóa) |
| `SPEECHSUPER_APPLICATION_ID`, `SPEECHSUPER_SECRET_KEY`, `SPEECHSUPER_USER_ID`, `SPEECHSUPER_BASE_HOST` | — | ⚠️ **Không còn được tham chiếu trong code** (legacy/dead). Xem mục 5. |

### 3.3 Luồng nghiệp vụ
| Biến | Đọc tại | Ảnh hưởng |
|---|---|---|
| `MOCK_EXAM_PART13_FLOW_MODE`, `NEXT_PUBLIC_MOCK_EXAM_PART13_FLOW_MODE` | shared (feature-flags) | **mock-test** (chế độ chấm Part 1/3) |
| `PRACTICE_TOPUP_3_VND`, `PRACTICE_TOPUP_6_VND`, `PRACTICE_TOPUP_10_VND` | shared (pricing-config, đọc động qua `priceEnvKey`) | **billing** + **practice/mock-test** (giá gói lẻ lượt chấm) |
| `NEXT_PUBLIC_NINESPEAK_API_BASE_URL`, `NEXT_PUBLIC_NINESPEAK_API_TIMEOUT_MS` | shared, onboarding-dashboard | tất cả (persistence/history/report qua NineSpeak). **Lưu ý:** `..._BASE_URL` còn dùng để suy ra môi trường (chứa "staging" → staging) cho whitelist gói billing — xem `feature-flags.getAppEnvironment`. |

### 3.4 Hạ tầng repo — đọc tại `lib/` (root), `next.config.ts`, `scripts/`
| Biến | Ảnh hưởng |
|---|---|
| `CLOUDFLARE_R2_ACCOUNT_ID`, `CLOUDFLARE_R2_ACCESS_KEY_ID`, `CLOUDFLARE_R2_SECRET_ACCESS_KEY`, `CLOUDFLARE_R2_BUCKET_NAME`, `CLOUDFLARE_R2_PUBLIC_BASE_URL` | tất cả (upload audio) — đọc động trong `lib/cloudflare-r2.ts`, chấp nhận alias `CLOUDFLARE_*`/`R2_*` |
| `NEXT_PUBLIC_AUDIO_UPLOAD_MODE` | tất cả (chế độ upload audio) |
| `NEXT_PUBLIC_FIREBASE_*` (API_KEY, APP_ID, AUTH_DOMAIN, MESSAGING_SENDER_ID, PROJECT_ID, STORAGE_BUCKET) | auth client (mọi module qua `shared` AuthProvider) |
| `FIREBASE_ADMIN_PROJECT_ID`, `FIREBASE_ADMIN_CLIENT_EMAIL`, `FIREBASE_ADMIN_PRIVATE_KEY` | xác thực bearer-token server (BFF) |
| `APP_BUILD_ID`, `APP_BUILD_NOTE`, `APP_RELEASE_VERSION`, `NEXT_PUBLIC_SITE_URL` | build metadata / SEO |

### 3.5 Analytics & Logging — đọc tại `features/shared`
| Biến | Ảnh hưởng |
|---|---|
| `NEXT_PUBLIC_MIXPANEL_TOKEN`, `NEXT_PUBLIC_MIXPANEL_API_HOST`, `NEXT_PUBLIC_GA4_MEASUREMENT_ID`, `NEXT_PUBLIC_ANALYTICS_ENABLED`, `NEXT_PUBLIC_ANALYTICS_DEBUG` | tất cả (tracking) |
| `ALPHA_LOG_MODE`, `NEXT_PUBLIC_ALPHA_LOG_MODE` | tất cả (alpha logging — debug ephemeral) |
| `ERROR_LOG_FIRESTORE` | tất cả (bật ghi lỗi bền vững vào Firestore `error_logs`; mặc định bật khi Firebase Admin đã cấu hình). Xem `docs/logging-and-observability.md`. |

---

## 4. "Biến env ảnh hưởng tới module của tôi" (bảng tra cho owner)

| Module | Biến cần nắm (dù được đọc ở shared) |
|---|---|
| **Mock Test** | `MOCK_EXAM_PART13_FLOW_MODE`, `NEXT_PUBLIC_MOCK_EXAM_PART13_FLOW_MODE`, (chung) AI/Speech/R2/NineSpeak |
| **Practice** | `PRACTICE_EVALUATION_PROVIDER`, `NEXT_PUBLIC_PRACTICE_EVALUATION_PROVIDER`, `PRACTICE_TOPUP_*`, (chung) AI/Speech/R2/NineSpeak |
| **Billing** | `PRACTICE_TOPUP_3_VND`, `PRACTICE_TOPUP_6_VND`, `PRACTICE_TOPUP_10_VND` (gói premium subscription lấy qua API `/premium-packages`, không qua env) |
| **Onboarding/Dashboard** | `NEXT_PUBLIC_NINESPEAK_API_BASE_URL`, `NEXT_PUBLIC_NINESPEAK_API_TIMEOUT_MS`, Firebase admin (BFF) |
| **Shared (lõi)** | tất cả khoá AI/Azure/Deepgram/Mixpanel/R2/Firebase — đây là nơi đọc thực sự |

> Quy ước khi thêm biến: nếu biến cấu hình một **dịch vụ lõi** (AI, speech, storage, analytics) → đặt ở `env/shared.env` và đọc trong `features/shared`. Nếu chỉ đổi giữa flavor → đặt ở `env/{staging,production}.env`. Module **không nên** tự `process.env` — thêm cờ qua `features/shared/lib/feature-flags.ts` rồi tiêu thụ giá trị đó.

---

## 5. Phát hiện & xử lý

1. **`SPEECHSUPER_*` (4 biến) không còn được tham chiếu** trong toàn bộ source. ✅ **Đã xử lý:** gỡ khỏi `env/shared.env.example` (thay bằng ghi chú `[LEGACY]`) và khỏi `env/shared.env` local. Code đã xác nhận 0 tham chiếu nên không ảnh hưởng runtime.
2. **Đặt tên gây nhầm:** ✅ **Đã xử lý:** thêm chú thích ở đầu `env/shared.env.example` nêu rõ *"SHARED = chung giữa các flavor, KHÔNG phải module `features/shared/`"*.
3. **Nhóm theo section bằng comment:** ✅ Template `env/shared.env.example` đã được tổ chức theo section `[B1]…[B9]` (AI / Whisper / Azure / selector / Deepgram / mock-flow / NineSpeak / top-up / R2).
4. **Giữ nguyên mô hình nạp theo flavor.** Không tạo `env/<module>.env` — sẽ phá tính tập trung và dễ lệch khoá dùng chung.

> Lưu ý: file `.env` thật (`env/shared.env`, `staging.env`, `production.env`) bị **gitignore** (chứa secret) nên thay đổi chỉ ở bản template `*.example` được commit; khi deploy nhớ đồng bộ secret thật tương ứng.
