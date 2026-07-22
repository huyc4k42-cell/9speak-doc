# Practice scoring redesign — Decision record (2026-06)

> Tài liệu này ghi lại **vì sao** luồng chấm practice được thiết kế lại theo hướng Deepgram-only +
> DeepSeek + Azure chạy ngầm (detail-only). Dùng cho người vận hành/dev sau hiểu bối cảnh trước khi sửa. Số liệu đo là
> căn cứ ra quyết định — **đừng xoá**.

## 1. Bối cảnh & mục tiêu
- Tối ưu **tốc độ "ấn tượng đầu"** (thời gian tới band đầu tiên) khi user dùng practice.
- Tối ưu **chi phí Vercel** (băng thông function — không đẩy file audio qua lại).
- **Tách biệt khỏi mock-test**: practice viết logic chấm RIÊNG, chỉ dùng chung hạ tầng ổn định
  (gọi LLM HTTP+JSON+retry, wrapper Azure, R2, error handling). Lý do: có dev khác đang sửa mock-test
  độc lập — không được đụng `features/shared/` evaluation-pipeline dùng chung.

## 2. Số liệu đo (nền tảng quyết định)
Đo trên 3 mẫu TTS (giọng đọc tiếng Anh "sở thích đọc sách") ~10s/20s/30s, qua công cụ debug so 4 cấu hình:

**Tới-band** (speech + calibration + quickScore — chưa tính rewrite vì rewrite hiển thị sau band):
| Cấu hình | 10s | 20s | 30s |
|---|---|---|---|
| A · Whisper+Azure · DeepSeek | 10.1s | 18.6s | 18.9s |
| B · Deepgram+Azure · Gemini | 9.0s | 18.6s | 21.4s |
| **C · Deepgram-only · DeepSeek calibrate** | **5.0s** | **6.5s** | **7.0s** |
| D · Deepgram-only · heuristic (no calibrate) | 3.7s | 5.8s | 4.9s |

**Band phát âm** (Azure = chuẩn thật):
| Mẫu | Azure (A/B) | C calibrate | D heuristic |
|---|---|---|---|
| 10s | 8 | 6 | 9 |
| 20s | 8 | 6.5 | 9 |
| 30s | 8 | 6.5 | 9 |

Ghi chú phụ: rewrite là bước tốn nhất (~3-8s, tăng theo độ dài) nhưng nằm **sau** band; Gemini json_object
ban đầu lỗi shape (phải dùng json_schema); Deepgram SDK `transcribeFile` bị `SLOW_UPLOAD` (phải dùng REST/URL);
`gemini-1.5-*`/`gemini-2.0-*` đã bị Google 404 (model lite còn sống: `gemini-2.5-flash-lite`).

## 3. Quyết định & lý do
1. **Bỏ Azure khỏi hot-path** → KHÔNG còn chặn đường tới-band. Ban đầu để lazy (bấm tab mới gọi) nhưng
   chờ Azure lúc bấm thì lâu → đổi sang **chạy NGẦM tự động** ngay sau khi các phân tích DeepSeek xong
   (không chặn hot-path; mở tab sớm thì hiện loading toàn box). Azure scripted chậm và tăng theo độ dài
   audio (tới-band 18-21s với câu dài) → bỏ khỏi hot-path giúp nhanh ~3×; pre-warm ngầm để bấm tab đỡ chờ.
2. **Bỏ Gemini, dùng DeepSeek** cho calibrate/quickScore/rewrite. Data: Gemini **không** nhanh hơn
   (30s còn chậm hơn), lại thêm phụ thuộc + rủi ro JSON-schema. Không đáng.
3. **Dùng calibrate (C), KHÔNG dùng heuristic (D)**. Heuristic chấm phát âm từ confidence Deepgram →
   **luôn = 9 bất kể giọng** (không phân biệt được người phát âm kém) → nguy hiểm. Calibrate (DeepSeek
   suy luận từ transcript + timestamp) ít nhất có lý lẽ và bảo thủ.
4. **Azure (ngầm, detail-only) = chỉ hiển thị chi tiết, KHÔNG tính/cập nhật lại band.** Band khoá ở bước đầu là band
   cuối; tab Phát âm chỉ bổ sung phoneme/từ yếu/tô màu để xem.
5. **Lưu ý chất lượng**: không có Azure thì band phát âm ở hot-path **không đo được thật** (calibrate
   lệch thấp ~1.5 trên giọng TTS sạch; với học viên thật có thể gần hơn). → cần **hiệu chỉnh calibrate
   bằng audio người dùng thật** qua công cụ debug-bench (so calibrate vs Azure trên nhiều mẫu).

## 4. Kiến trúc đích
- Ghi âm → **R2 trực tiếp (presigned PUT)** + giữ blob ở browser (playback).
- `/assessments`: **Deepgram qua `transcribeUrl` + presigned GET** (function KHÔNG chạm bytes audio) →
  transcript + word timestamp + confidence → **DeepSeek calibrate** (truyền timestamp để chấm pausing) →
  **DeepSeek quickScore** (4 tiêu chí + overall) → hiển thị. Rewrite (DeepSeek) hiển thị tiếp.
- UI bước đầu: **chỉ điểm + rewrite, KHÔNG tô màu phát âm.**
- Sau band: Trôi chảy (pause từ timestamp Deepgram + LLM), Từ vựng/Ngữ pháp (DeepSeek), **Phát âm
  (Azure scripted detail-only) chạy NGẦM tự động** sau khi 4 phân tích trên xong (cắt 25s server-side từ R2).
- **2 map riêng** để các tab không ghi đè nhau: `transcriptSentences` (cefr + pause) cho Từ vựng/Trôi chảy
  vs `pronunciationSentences` (điểm Azure) cho Phát âm. Đọc qua `resolvePronunciationSentences()` (fallback
  `transcriptSentences` cho bản cũ/baseline). Per-word coloring ở report dùng annotations từ `mispronouncedWords`.
- Mỗi bước cập nhật session/history để xem lại.
- **(staging) đo thời gian**: bấm đúp điểm overall → popup recordEnd→band + chi tiết từng API (chỉ staging).

## 5. Quy tắc tách module (isolation)
- **Shared (chỉ dùng lại, không sửa):** `requestStructuredOpenAiJson`, `resolveAiScoringConfig`,
  `evaluateAzureWithReference`/`splitWavIntoSegments`, R2 helpers, `alpha-logger`, pause-math
  (`transcript-pause-*`).
- **Practice RIÊNG (`features/practice/server/`):** Deepgram client, calibration (Deepgram-based),
  quickScore, fluency/pause orchestration, endpoints, prompt namespace `practice.*`.
- **Tuyệt đối không sửa** `features/shared/server/evaluation-pipeline.ts` (mock-test dùng chung).

## 6. Trạng thái
Thử nghiệm + đo xong (branch `feat/practice-pipeline-compare-debug`). Đã **revert sạch mọi thay đổi
shared** về `dev` (chỉ giữ mẫu audio `public/debug-samples/`). Triển khai production theo §4-5 đang chờ làm.
