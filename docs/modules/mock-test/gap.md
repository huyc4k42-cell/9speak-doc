# Gap — Report "Phản hồi chi tiết" (4 tiêu chí IELTS Speaking)

> **Cập nhật:** 2026-06-27 · **Trạng thái:** đang triển khai (nhánh `feat/mocktest-exam-v2`).
> Đối chiếu **tài liệu học thuật "Report Phản hồi chi tiết" (v0.1)** + mockup `mocktest-flow-mockup.html`
> với hiện trạng code (`features/shared/components/reports/MockExamReportSections.tsx` + `MockExamReport.tsx`).
> Mục đích: theo dõi mục nào ĐÃ làm, mục nào CHƯA làm được và **vì sao** (đặc biệt phần cần mở pipeline/content).

**Chú thích:** ✅ đã làm · 🔵 làm được client-side (data đã có) · 🔴 cần data/pipeline/content (chưa làm được).

---

## 1. Trôi chảy & Mạch lạc

| Mục (tài liệu 🖥️) | Trạng thái | Ghi chú |
|---|---|---|
| Tốc độ nói + thước đo + nhận xét | ✅ | `FluencySection` (thước ~80–200; tài liệu nói 60–200 — lệch nhẹ, có thể chỉnh) |
| Chips **N ngắt tốt / vừa / dài / filler** (3 mức) | ✅ | `computeFluencyChipCounts` + `resolvePauseTier` (good/bad + độ dài → 3 mức) |
| Chấm 3 màu ngắt trong transcript | ✅ | 1 chấm cuối mỗi câu (`sentencePauseTier`) |
| Mạch lạc: *Liên từ đã dùng* + *gợi ý thêm* | ✅ | `CoherencePanel` (`cohesiveDevices` + gợi ý tĩnh) |
| Mạch lạc: **badge** ("Tròn ý nhưng chưa sâu") + **nhận xét chiều sâu** | 🔴 | Pipeline chưa trả tín hiệu đánh giá độ sâu ý/coherence rating |
| **Filler "uhm" inline** trong transcript | 🔴 | Pipeline chỉ trả **đếm** filler (`fluencyProfile.fillerCounts`), **không có vị trí** từng từ |

## 2. Từ vựng

| Mục | Trạng thái | Ghi chú |
|---|---|---|
| Transcript **gạch chân theo màu CEFR** | ✅ | `cefrTextColorClass` + `underline decoration-current` |
| Phân bố cấp độ **CEFR 6 mức A1–C2** | ✅ | `CefrProfileCard` derive từ `word.cefrLevel`, fallback `lexicalProfile.cefrDistribution` |
| Gợi ý nâng cấp (paraphrase) 3 lựa chọn | ✅ | `LexicalHighlightCard` (`replacementOptions`) |
| **Thành ngữ** (cụm đáng học + nghĩa) | 🔴 | Chưa có field idiom riêng trong `lexicalProfile`; cần pipeline gắn nhãn idiom/collocation |

## 3. Ngữ pháp

| Mục | Trạng thái | Ghi chú |
|---|---|---|
| Sentence-card đơn/phức + lỗi/không (viền) | ✅ | Tab Ngữ pháp card-per-câu (AC-06) |
| Độ chính xác **X/Y câu** + tỉ lệ đơn/phức | ✅ | "X/Y câu" derive từ annotation grammar; đơn/phức = `classifySentenceComplexity` (heuristic) |
| Lỗi theo loại | ✅ | gom `grammarProfile.highlights[].subtype` |
| Cấu trúc nâng cao ✓ Dùng đúng / ⚠ Chưa đúng | ✅ | `complexStructureUsage.successful/failed` |
| Cấu trúc nâng cao **💡 Nên thử** | 🔴 | Data chỉ có attempted/successful/failed — không có danh sách "nên thử" (chưa dùng) |
| **Gợi ý nâng cấp cấu trúc** (câu mẫu) | 🔴 | Pipeline không sinh câu mẫu nâng cấp cấu trúc |
| Phân loại đơn/phức **chính xác (LLM)** | 🔵 | Hiện heuristic regex client-side; có thể nâng classifier LLM ở bước grammar finalize |

## 4. Phát âm

| Mục | Trạng thái | Ghi chú |
|---|---|---|
| Highlight từ trong transcript theo **3 mức** (tốt ≥75 / khá 50–75 / cần cải thiện <50) | ✅ | `pronWordTone(word.pronunciationScore)` — khớp legend; từ yếu vẫn bấm mở popup |
| **✗ sau từ "cần sửa"** (khá + cần cải thiện) trong transcript | ✅ | gắn ✗ sau từ <75; từ tốt không có ✗; "Nhóm theo âm" không gắn ✗ |
| Lưới "Lỗi phát âm (nặng → nhẹ)": từ + ×số lần + % | ✅ | sort theo `pronunciationScore` |
| ✓ Đúng `/ipa/` + nghe (🔊) | ✅ | `referenceIpa` + Google TTS (`buildTtsAudioUrl`) |
| ✗ Bạn nói **+ IPA as-said** + nghe (🔊) | ✅ | IPA as-said tái dựng từ NBestPhonemes (`buildSpokenIpaFromPhonemes`); nghe đúng đoạn user nói (`onPlayPrecise`) |
| Ngữ cảnh "…câu" | ✅ | câu chứa từ (`sentenceByWordId`) |
| Luyện › → mini-drill | ✅ | `onPracticeWord` → ghi âm + chấm Azure (miễn phí) |
| Nhóm theo âm (phoneme + ví dụ) | ✅ | `pronunciationProfile.topWeakSounds` |
| Nhóm theo âm **+ cách đặt khẩu hình/lưỡi** | 🔴 | Cần **bảng coaching theo phoneme** (content tĩnh) — chưa có |

---

## 5. Tồn đọng xuyên suốt (cross-cutting)

| Mục | Trạng thái | Ghi chú |
|---|---|---|
| Commit/merge | ⏳ | Toàn bộ đang trên `feat/mocktest-exam-v2`, **chưa commit** — chờ duyệt |
| Xác minh **runtime** | ⏳ | drill Azure thật, karaoke timing, round-trip lịch sử panel mới, progress local — cần test trên app thật |
| Progress bài thi | ✅ (local) | đã làm / đang làm dở lưu `localStorage`; **chưa đồng bộ BE** (đúng yêu cầu hiện tại) |
| Part 1/2/3 scope | ⏳ | phòng thi/chuẩn bị theo từng Part **chưa wiring** (CTA Part vẫn vào full) — mốc PRD §3.5, không thuộc report |

---

## 6. Việc cần phía pipeline/content để mở khoá nhóm 🔴

Để hoàn thiện 100% theo tài liệu học thuật, cần bổ sung **đầu ra pipeline** hoặc **content tĩnh**:

1. **Coherence rating + depth signal** (Trôi chảy): model trả mức "đủ sâu / liệt kê" + 1 câu nhận xét → badge Mạch lạc.
2. **Filler vị trí** (Trôi chảy): đánh dấu offset/wordId của filler trong transcript (không chỉ đếm).
3. **Idiom/collocation tagging** (Từ vựng): danh sách cụm đáng học + nghĩa.
4. **Structure suggestions** (Ngữ pháp): danh sách "nên thử" + câu mẫu nâng cấp cấu trúc.
5. **Phoneme coaching map** (Phát âm): bảng tĩnh "cách đặt khẩu hình/lưỡi" theo nhóm âm (`/ð/ /θ/`, `/æ/`, âm cuối `/t/ /d/`…).
6. *(Tùy chọn)* **Sentence complexity classifier LLM** thay heuristic đơn/phức.

> Mọi mục ✅/🔵 ở trên **build xanh** (`npm run build` status=0). Nhóm 🔴 **không làm được client-only** — chờ quyết định mở pipeline/thêm content.
