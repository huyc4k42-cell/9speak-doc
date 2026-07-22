# Logging & Observability

> Mục tiêu: khi khách hàng gặp lỗi, đội ngũ có thể **trace nhanh** nguyên nhân gốc (lỗi gì, ở khúc nào, API nào, module nào) trong khi màn hình khách chỉ hiện thông báo chung. Tài liệu này mô tả cơ chế log hiện tại, sink lỗi bền vững mới (Firestore), cách xem, ảnh hưởng hiệu năng, cấu hình, và lộ trình.

---

## 1. Kiến trúc log (sau cải tiến)

Có **3 đích ghi (sink)**, mỗi cái cho một mục đích khác nhau:

| Sink | Dùng cho | Bền vững | Bật khi |
|---|---|---|---|
| **Firestore `error_logs`** | **Trace lỗi production** (chính) | ✅ có TTL | error/warn, luôn bật khi Firebase Admin cấu hình |
| **Vercel Logs** (`console.error`) | Live tail / debug nhanh | ✅ theo gói Vercel | mọi `console.error` |
| **Alpha logs** (buffer in-memory + file local) | Debug session/dev sâu | ❌ ephemeral | `ALPHA_LOG_MODE` ≠ off |

Luồng: mọi điểm bắt lỗi gọi `appendAlphaLog(level: "error", ...)` (đã có sẵn ở các tầng giàu thông tin: AI provider, speech, charge ledger, route handlers). Hàm này:
1. **Luôn** chuyển bản ghi error/warn sang [`recordErrorLog`](../features/shared/server/error-logger.ts) → Firestore `error_logs` (fire-and-forget) + 1 dòng `console.error` (Vercel) — **độc lập với `ALPHA_LOG_MODE`**.
2. Sau đó mới chạy logic alpha-buffer/file như cũ (chỉ cho dev/session).

→ Lỗi nay được ghi bền vững **bất kể** alpha mode, kể cả khi khách không bật session debug.

### Thông tin được bắt cho mỗi lỗi
`timestamp`, `level`, `source` (server/client), `event` (vd `ai-provider-response-failed`), `message`, `module` (suy ra từ route), `route`, `statusCode`, `retryCount`, `requestId`, `sessionId`, `userId`, `part`, `questionId`, `payloadSnippet` (**body lỗi thật từ provider**, vd `"insufficient balance"`), `detailsText` (đã cắt gọn), `expireAt` (TTL).

**Ví dụ đúng nhu cầu:** khách dùng Practice nhận "Lỗi máy chủ AI". Trace: query `error_logs` theo `userId` + thời điểm → thấy `event=ai-provider-response-failed`, `statusCode=402`, `payloadSnippet` chứa `insufficient balance`, `provider=deepseek` → biết ngay DeepSeek hết số dư.

---

## 2. Xem log ở đâu

| Nơi | Cách xem | Ghi chú |
|---|---|---|
| **Firebase Console → Firestore → `error_logs`** | Lọc theo `userId`, `requestId`, `module`, `statusCode`, `timestamp` | Nguồn chính để trace lỗi khách. Sắp theo `createdAt` desc. |
| **Vercel → Project → Logs** | Tìm chuỗi `[error-log]` hoặc `requestId` | Live tail, retention theo gói. |
| **App `/settings/logs`** | UI alpha (buffer in-memory) | Chỉ dev/session, **không** dùng để trace lỗi production. |

**Quy trình support đề xuất:** khách báo lỗi (kèm thời điểm) → mở `error_logs`, lọc `userId == <uid khách>` quanh thời điểm → đọc `event/statusCode/payloadSnippet`. Nếu UI có hiện `requestId` (Mã lỗi) thì lọc thẳng theo `requestId`.

> **Correlation:** response lỗi 5xx của các route chấm điểm (Practice/Mock) nay kèm field `requestId`. Có thể hiện "Mã lỗi: {requestId}" trên UI để khách đọc cho support (xem Roadmap).

---

## 3. Ảnh hưởng hiệu năng

Thiết kế để **không** làm chậm/giật request người dùng:
- **Fire-and-forget**: ghi Firestore KHÔNG được `await` trong hot path (`void writeErrorLogDoc(...).catch(...)`).
- **Chỉ error/warn**: bỏ qua toàn bộ log info/success → ít ghi, **tối ưu chi phí Firestore**.
- **Rate-limit** theo chữ ký lỗi (`event:statusCode`): tối đa 30 ghi/phút/chữ ký → chống write-storm khi provider sập (vừa giảm chi phí vừa tránh nghẽn).
- **Bounded size**: `payloadSnippet`/`detailsText` cắt còn ≤ 4000 ký tự, `message` ≤ 1000 → document nhỏ.
- **Keepalive an toàn (client)**: log client POST `/api/app/logs` qua `fetch`. `keepalive: true` chỉ bật khi body `< 32KB` (spec Fetch giới hạn tổng body keepalive 64KB/trang — body lớn làm tràn quota, fetch bị reject, Safari còn báo nhầm `access control checks`). Snippet/details phía client cũng cắt còn ≤ 8000 ký tự (`ALPHA_LOG_CLIENT_MAX_STRING_LENGTH`) để giữ body nhỏ. Xem `features/shared/lib/alpha-log-client.ts`.
- **TTL**: field `expireAt` (30 ngày) — bật TTL policy để Firestore tự xoá, không phình kho.

> Lưu ý kỹ thuật còn lại (tối ưu sau, không bắt buộc): `createAlphaLogRecord` đang serialize `details` sớm cho cả log info/success — xem Roadmap.

---

## 4. Cấu hình

| Env | Ý nghĩa | Mặc định |
|---|---|---|
| `ERROR_LOG_FIRESTORE` | `on`/`off` bật ghi lỗi vào Firestore | bật khi `FIREBASE_ADMIN_*` đã cấu hình |
| `ALPHA_LOG_MODE` | `off`/`session`/`persist` cho alpha logs (debug) | Vercel: `session`, local: `persist` |

**Bắt buộc khi bật Firestore sink:** đã cấu hình `FIREBASE_ADMIN_PROJECT_ID`, `FIREBASE_ADMIN_CLIENT_EMAIL`, `FIREBASE_ADMIN_PRIVATE_KEY`.

### Cần làm thủ công 1 lần trên Firebase
1. **Bật TTL policy** cho collection `error_logs` trên field `expireAt` (Firestore → TTL) để tự xoá log cũ.
2. (Khuyến nghị) Tạo composite index `userId ASC, createdAt DESC` và `requestId ASC` để query support nhanh.
3. **Security rules**: `error_logs` chỉ ghi/đọc từ server (Admin SDK bỏ qua rules); KHÔNG mở đọc cho client.

---

## 5. Độ phủ hiện tại

Instrumentation đã phủ các luồng chính (đều đi qua `appendAlphaLog` → nay vào `error_logs`):
- **AI/LLM**: `ai-provider-response-failed` (OpenAI/DeepSeek, kèm body + statusCode) — [openai-structured-output.ts](../features/shared/server/openai-structured-output.ts).
- **Speech**: Azure/Deepgram/Whisper ([azure-speech](../features/shared/server/azure-speech.ts), [deepgram-speech](../features/shared/server/deepgram-speech.ts), [openai-transcribe](../features/shared/server/openai-transcribe.ts)).
- **Chấm điểm**: [evaluate-practice](../features/practice/server/evaluate-practice.ts), [evaluate-mock-exam-segmented](../features/mock-test/server/evaluate-mock-exam-segmented.ts).
- **Route handlers**: [practice](../features/practice/server/practice-route-handlers.ts), [mock](../features/mock-test/server/mock-exam-route-handlers.ts) (kèm `requestId` trong response 5xx).
- **Ledger lượt chấm**: [mocktest](../features/mock-test/server/mocktest-charge-ledger.ts), [practice](../features/practice/server/practice-charge-ledger.ts).
- **Client**: lỗi UI (`window.onerror`, `unhandledrejection`, `console.error`) qua [ClientDiagnosticsProvider](../features/shared/providers/ClientDiagnosticsProvider.tsx) → POST `/api/app/logs` (hiện vào alpha sink; xem Roadmap để đẩy sang `error_logs`).

---

## 6. Roadmap (chưa làm)

- **Hiện `requestId` (Mã lỗi) trên UI** ở các toast lỗi hệ thống để khách đọc cho support.
- **Lỗi client → `error_logs`**: cho `POST /api/app/logs` (level error) ghi thẳng `error_logs` thay vì chỉ alpha buffer.
- **Viewer nội bộ cho support**: trang/endpoint (có phân quyền admin) đọc `error_logs` từ Firestore (thay vì phải vào Firebase console).
- **Error taxonomy chuẩn**: map provider error → cause rõ ràng (402 balance / 429 rate-limit / 401 auth / timeout…).
- **Alerting**: cảnh báo (Slack/email) khi tỉ lệ lỗi tăng đột biến hoặc gặp lỗi billing-provider.
- **Redaction PII** trước khi ghi (rà soát `detailsText`/`payloadSnippet`).
- **Tối ưu**: bỏ eager-serialize `details` cho log info/success trong `createAlphaLogRecord`.
- (Tùy chọn) Cân nhắc dịch vụ ngoài (Sentry/Axiom) nếu cần grouping/alerting mạnh hơn — hiện ưu tiên Firestore để tối ưu chi phí.
