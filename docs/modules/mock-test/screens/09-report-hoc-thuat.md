# Mock Test V2 — §9 Tài liệu học thuật — Rubric & Band Descriptor (4 tiêu chí IELTS Speaking) — v0.2 (đối chiếu code thật)

> Ngày cập nhật: 2026-07-07 · Trạng thái: Chính thức · Đối chiếu [`band-score.ts`](../../../../features/shared/lib/band-score.ts), [`cefr-mapping.ts`](../../../../features/shared/lib/cefr-mapping.ts), [`pronunciation-band.ts`](../../../../features/shared/lib/pronunciation-band.ts), [`transcript-pause-classifier.ts`](../../../../features/shared/lib/transcript-pause-classifier.ts), [`transcript-pause-stats.ts`](../../../../features/shared/lib/transcript-pause-stats.ts), [`azure-pronunciation.ts`](../../../../features/shared/lib/azure-pronunciation.ts), [`cefr-distribution.ts`](../../../../features/shared/lib/cefr-distribution.ts), [`evaluation-pipeline.ts`](../../../../features/shared/server/evaluation-pipeline.ts), [`openai-prompts.json`](../../../../features/shared/server/config/openai-prompts.json), [`vocab-limits.ts`](../../../../features/shared/server/config/vocab-limits.ts), [`mock-exam.ts`](../../../../features/shared/types/mock-exam.ts) · Nguồn gốc: chuyển thể + nâng cấp từ Google Doc "Mocktest V2" (§9, v0.1, 2026-06-20) — gộp toàn bộ nội dung base vào, đối chiếu code thật, đóng các câu hỏi mở đã giải quyết được.

> **Bản quyền:** Các mô tả band dưới đây được **diễn đạt lại** theo tinh thần band descriptors công khai của IELTS, **không trích nguyên văn**. Khi đưa vào sản phẩm nên rà lại với chuyên gia IELTS.

> **Đây KHÔNG phải spec "màn hình"** — đây là tài liệu rubric/tham chiếu dùng chung cho Report (#07, #08) và pipeline chấm điểm. Không theo template 7 mục FR/BR/AC như các file 01–08.

## Đổi log so với v0.1 (Google Doc, 2026-06-20)

| # | Thay đổi | Lý do |
|---|---|---|
| 1 | **Sửa ngưỡng phát âm "tốt" từ ≥75% thành ≥70%** — khớp code thật | `pronunciation-band.ts:29-30`: good = >70, fair = 50-70, poor = <50. Doc cũ ghi 75% không đúng thực tế |
| 2 | **Bổ sung CEFR mapping ≤4.0→A2** — doc cũ thiếu vùng dưới B1 | `cefr-mapping.ts:16-38`: ≤4.0→A2 (bao gồm cả dưới 4.0). Doc cũ chỉ liệt kê từ 4.0–5.0→B1 trở lên |
| 3 | **Cập nhật mô tả phân loại ngắt (pause)**: code dùng heuristic context-aware phức tạp hơn 3 ngưỡng cứng trong doc | `transcript-pause-classifier.ts:3`: `GOOD_PAUSE_MAX_MS = 1200`, `GOOD_SENTENCE_END_MAX_MS = 2000`, `BAD_EXTREME_PAUSE_MS = 2500` — xét cả vị trí ranh giới mệnh đề, signposting, sentence-end |
| 4 | **Bổ sung annotation ✅/⚠️/❌ cho từng mục**: đánh dấu rõ phần nào code đã build, phần nào chỉ là spec đích | Tổng hợp từ [`gap.md`](../gap.md) + research trực tiếp, cho team thấy hiện trạng thật |
| 5 | **Đóng 4/7 câu hỏi mở** (§6 doc cũ) đã giải quyết qua file #07/#08 + code research | Xem bảng §9.8 |
| 6 | **Ghi rõ divergence quy tắc làm tròn**: code dùng `Math.round(value * 2) / 2` (banker's rounding) thay vì IELTS rule chính thức | `band-score.ts:20-22` — cần quyết định align về phía nào |
| 7 | **Bổ sung giới hạn pipeline thật**: MAX_VOCAB_ERRORS=10, MAX_VOCAB_SUGGESTIONS=4, grammar subtypes 16 loại (richer than doc's 7) | `vocab-limits.ts`, `evaluation-pipeline.ts:51-57`, `mock-exam.ts:398-416` |

---

## Mục đích

Định nghĩa nền tảng học thuật cho phần Report — Phản hồi chi tiết (#07, #08) — vừa là:
- **🎯 Rubric chấm** (đo cái gì, dấu hiệu phát hiện, ngưỡng, cách phân loại) cho team chấm/thiết kế prompt + dev
- **🖥️ Nguồn nội dung hiển thị** (mô tả band, lời khuyên) cho content/UX

> **Quy ước:** 🎯 *Rubric* = cách chấm/phát hiện · 🖥️ *Hiển thị* = nội dung cho người học · 🔧 *Ngưỡng* = tham số **cần hiệu chỉnh (tunable)** bằng dữ liệu thật.
>
> **Trạng thái code:** ✅ = đã build khớp · ⚠️ = build nhưng khác chi tiết · ❌ = chưa build (spec đích) · Xem chi tiết tại [`gap.md`](../gap.md).

---

## 9.0 Nền tảng chung

- **4 tiêu chí, trọng số ngang nhau (mỗi tiêu chí 25%):** Trôi chảy & Mạch lạc (Fluency & Coherence), Từ vựng (Lexical Resource), Ngữ pháp (Grammatical Range & Accuracy), Phát âm (Pronunciation). ✅
- **Band 0–9, bước 0.5.** Mỗi tiêu chí có **3 sub-tiêu chí** (hiển thị trong accordion #07). ✅ — `band-score.ts:20-22`: `Math.round(value * 2) / 2`; sub-scores do AI sinh, không phải tính toán độc lập
- **Điểm tổng (overall):** trung bình cộng 4 band tiêu chí. ✅ — `practice-quickscore.ts:110-112`

> ⚠️ **Divergence — quy tắc làm tròn:**
> - **IELTS chính thức:** phần lẻ .25 → làm tròn lên .5; .75 → làm tròn lên số nguyên (vd 6.25→6.5; 6.75→7.0)
> - **Code hiện tại:** `Math.round(value * 2) / 2` (banker's rounding) — khác ở biên 6.25 (IELTS: 6.5; banker's: 6.0)
> - **LLM prompt** (`openai-prompts.json`, quickScore system) mô tả đúng IELTS rule — nhưng code post-process không enforce
> - **Cần quyết định:** sửa code theo đúng IELTS, hay chấp nhận banker's rounding?

- **Quy đổi CEFR (xấp xỉ, để hiển thị):** ✅ — `cefr-mapping.ts:16-38`

| IELTS | CEFR |
|---|---|
| **≤ 4.0** | A2 |
| **4.5–5.0** | B1 |
| **5.5–6.5** | B2 |
| **7.0–8.0** | C1 |
| **≥ 8.5** | C2 |

> Lưu ý: doc cũ thiếu vùng ≤4.0→A2. Code xử lý cả `overallBand ≤ 0` → A2 (A1 hiếm khi xuất hiện trong IELTS Speaking context).

---

*Mỗi mục 1–4 dưới đây có cùng bố cục:* **(a) Đo gì · (b) Sub-tiêu chí · (c) Kỳ vọng theo band · (d) 🎯 Dấu hiệu & ngưỡng phân loại · (e) 🖥️ Nội dung hiển thị · (f) Ví dụ.**

---

## 9.1 Trôi chảy & Mạch lạc (Fluency & Coherence)

### (a) Đo gì

- **Fluency (trôi chảy):** khả năng nói liên tục với tốc độ và nỗ lực bình thường; tần suất/độ dài ngập ngừng; lặp lại; tự sửa.
- **Coherence (mạch lạc):** ý nối tiếp logic; dùng **liên từ/cụm nối (cohesive/linking devices)** hợp lý; **phát triển chủ đề** có chiều sâu (mở rộng, ví dụ) chứ không liệt kê rời rạc.

### (b) Sub-tiêu chí (hiển thị)

Độ trôi chảy · Liên kết ý · Phát triển chủ đề.

### (c) Kỳ vọng theo band (diễn đạt lại)

| Band | Kỳ vọng |
|---|---|
| **5** | Nói được nhưng ngắt/ngập ngừng nhiều, hay lặp & tự sửa; liên kết cơ bản (and, but, because), chủ đề phát triển hạn chế. |
| **6** | Nói khá dài, đôi lúc mất mạch do tìm từ/ngữ pháp; dùng liên từ đa dạng hơn nhưng chưa luôn phù hợp. |
| **7** | Nói dài không mấy khó khăn; ngập ngừng chủ yếu do **nội dung** (suy nghĩ ý) chứ ít do ngôn ngữ; liên kết linh hoạt, phát triển ý mạch lạc. |
| **8** | Trôi chảy, chỉ thỉnh thoảng lặp/tự sửa; mạch lạc tốt, triển khai ý đầy đủ và phù hợp. |

### (d) 🎯 Dấu hiệu & ngưỡng phân loại

**Phân loại khoảng ngắt (pause) — 3 mức:**

| Mức | Doc gốc 🔧 | Code thật | Trạng thái |
|---|---|---|---|
| 🟢 Ngắt tốt | ≲ 0.5s | `GOOD_PAUSE_MAX_MS = 1200` (1.2s), tối đa 2.0s nếu cuối câu (`GOOD_SENTENCE_END_MAX_MS`) | ⚠️ Code phức tạp hơn |
| 🟡 Ngắt vừa | ~0.5–1.0s | Không có mức "vừa" tách biệt — heuristic phân good/bad/none theo ngữ cảnh (ranh giới mệnh đề, signposting, vị trí sentence-end) | ⚠️ Khác cấu trúc |
| 🔴 Ngắt dài | > ~1.0s | `BAD_LEXICAL_SEARCH_MS = 1000` (>1s = bí từ), `BAD_EXTREME_PAUSE_MS = 2500` (>2.5s = extreme) | ⚠️ Ngưỡng khác |

> **Giải thích:** Code tại `transcript-pause-classifier.ts` dùng phân loại `PauseClassification = "none" | "good" | "bad"` — 3 giá trị nhưng ngữ nghĩa khác doc (không phải tốt/vừa/dài mà là không đáng kể/tự nhiên/có vấn đề). Heuristic xét cả vị trí ngắt (cuối câu vs giữa cụm), từ signposting trước ngắt, và độ dài tương đối.
>
> Khi hiển thị cho user, `computeFluencyChipCounts` gom thành **chips: N ngắt tốt / N ngắt vừa / N ngắt dài / N filler** ✅ — "vừa" ở UI derive từ good/bad + context, không phải ngưỡng cứng riêng.

**Filler:** ✅ — `transcript-pause-stats.ts:3-10` đếm 12 tiếng đệm: um, uh, er, ah, hmm, eh (+ biến thể). Trả `fillerCount` tổng.

> ❌ **Filler vị trí inline:** Pipeline chỉ trả **đếm** filler (`fluencyProfile.fillerCounts`), **không có vị trí** từng từ trong transcript — không thể highlight inline.

**Tốc độ nói (speech rate):** ⚠️

| Mục | Doc gốc 🔧 | Code thật |
|---|---|---|
| Thước đo | 60–200 wpm | Tính `Math.round((wordCount / durationSeconds) * 60)` — giá trị thô, không có thước cố định |
| Vùng tốt | ~110–150 wpm | LLM prompt nói "stable tempo around 130–170 wpm" — không có ngưỡng cứng client-side |
| Hiển thị | Thước + vùng xanh | `FluencySection` hiển thị thước ~80–200, chưa khớp hoàn toàn với doc (60–200) |

**Mạch lạc:**
- *Liên từ đã dùng* + *gợi ý thêm*: ✅ — `CoherencePanel` (`cohesiveDevices` + gợi ý tĩnh)
- *Độ sâu phát triển ý* + badge "Tròn ý nhưng chưa sâu": ❌ — Pipeline chưa trả tín hiệu đánh giá coherence depth

> ⚠️ **Liên từ không phải metric tự động:** Code không đếm linking words algorithmically — LLM nhận transcript rồi trả feedback định tính (mảng `cohesiveDevices`). Doc ngụ ý auto-detection, thực tế là AI qualitative.

### (e) 🖥️ Nội dung hiển thị

- **Tốc độ nói** + thước đo + câu nhận xét theo vùng. ✅
- **Mạch lạc:** gợi ý đào sâu + *Liên từ đã dùng* + *gợi ý thêm*. ✅ (badge đánh giá ❌)
- **Transcript:** chấm 3 màu pause + chips tổng hợp: N ngắt tốt / N ngắt vừa / N ngắt dài / N filler. ✅ (filler inline ❌)

### (f) Ví dụ

- 🔴 *Ngắt dài + filler:* "I think… *(1.4s)* um… the city is, you know, big." → gợi ý: chuẩn bị ý trước, giảm filler.
- ✅ *Mạch lạc tốt:* "I prefer the countryside **because** it's calmer, **and for instance**, I can grow my own vegetables there."

---

## 9.2 Từ vựng (Lexical Resource)

### (a) Đo gì

Độ rộng & chính xác của vốn từ: từ ít phổ biến/học thuật, **collocation**, **thành ngữ (idiomatic)**, khả năng **paraphrase**, và mức **phù hợp/chính xác nghĩa** (appropriacy).

### (b) Sub-tiêu chí

Độ đa dạng từ · Thành ngữ · Độ chính xác.

### (c) Kỳ vọng theo band

| Band | Kỳ vọng |
|---|---|
| **5** | Vốn từ đủ dùng cho chủ đề quen; hay lặp; paraphrase hạn chế; lỗi chọn từ làm giảm rõ nghĩa. |
| **6** | Đủ từ để bàn nhiều chủ đề & nói dài; dùng được một số từ ít phổ biến nhưng chưa ổn định; paraphrase được dù đôi khi vụng. |
| **7** | Dùng linh hoạt, có **từ ít phổ biến & thành ngữ**, có ý thức về văn phong/collocation; đôi khi chọn từ chưa khéo; paraphrase hiệu quả. |
| **8** | Vốn từ rộng, dùng tự nhiên & chính xác; thành ngữ & collocation thuần thục; paraphrase linh hoạt. |

### (d) 🎯 Dấu hiệu & ngưỡng phân loại

**Phân bố cấp độ từ (CEFR per word):** ✅ — Mỗi từ nội dung được gán A1–C2 qua AI (`MockExamTranscriptWord.cefrLevel`). Hiển thị qua `CefrProfileCard` + `cefr-distribution.ts` (gom 6 mức thành 4 nhóm: A, B1, B2, C, tính % bằng largest-remainder method).

| Mục | Doc gốc | Code thật | Trạng thái |
|---|---|---|---|
| Gán CEFR per word | A1–C2 | ✅ `word.cefrLevel` | ✅ |
| Phân bố % | 6 mức A1–C2 | 4 nhóm (A, B1, B2, C) | ⚠️ Gom nhóm khác |
| Tín hiệu "vốn từ cao": B2+ ≥ ~25–30% 🔧 | Có ngưỡng cứng | Không có rule — giá trị truyền cho LLM | ❌ |

**Độ đa dạng (lexical diversity):** Không có type-token ratio riêng. Cờ **lặp từ** khi một từ nội dung lặp nhiều lần — ❌ chưa implement.

**Collocation/idiom:** ❌ Phát hiện tự động KHÔNG có. Code có type `MockExamVocabularyBucket.collocations` và `idioms` nhưng chỉ lấy từ **sample answers** (nội dung tham chiếu), không phát hiện trong transcript người học.

**Lỗi chọn từ (precision):** ✅ — `evaluation-pipeline.ts:51-57` định nghĩa 5 loại lỗi: `wrong_meaning`, `wrong_collocation`, `wrong_register`, `literal_translation`, `wrong_word_form`. Schema enforce chỉ 5 loại này.

**Giới hạn pipeline:** ✅ — `vocab-limits.ts`: `MAX_VOCAB_ERRORS = 10`, `MAX_VOCAB_SUGGESTIONS = 4`.

### (e) 🖥️ Nội dung hiển thị

- **Phân bố cấp độ (CEFR):** thanh A–C (%) + nhận xét. ✅
- **Transcript:** tô màu + gạch chân từ theo CEFR (`cefrTextColorClass` + `underline decoration-current`). ✅
- **Gợi ý nâng cấp (paraphrase):** từ thường → 3–4 lựa chọn nâng cấp (`replacementOptions`). ✅
- **Thành ngữ** (cụm đáng học + nghĩa): ❌ — chưa có field idiom riêng trong `lexicalProfile`

### (f) Ví dụ

- ⬆️ *Nâng cấp:* "*happy*" (A2) → "*delighted / content*"; "*big difference*" → "*significant difference / world of difference*".
- ❌ *Sai collocation:* "*I did a mistake*" → "*made a mistake*".

---

## 9.3 Ngữ pháp (Grammatical Range & Accuracy)

### (a) Đo gì

**Độ đa dạng cấu trúc** (câu đơn vs phức/ghép; cấu trúc nâng cao) và **độ chính xác** (tần suất lỗi, mức ảnh hưởng đến nghĩa).

### (b) Sub-tiêu chí

Đa dạng cấu trúc · Độ chính xác thì · Độ chính xác ngữ pháp.

### (c) Kỳ vọng theo band

| Band | Kỳ vọng |
|---|---|
| **5** | Chủ yếu câu đơn/cụm cố định; câu phức ít & hay lỗi; lỗi thường gây hiểu sai. |
| **6** | Trộn câu đơn & phức nhưng độ linh hoạt hạn chế; lỗi còn nhiều nhưng hiếm khi gây khó hiểu. |
| **7** | Dùng đa dạng cấu trúc linh hoạt; **nhiều câu không lỗi**; còn vài lỗi nhỏ không hệ thống. |
| **8** | Đa dạng & phần lớn chính xác; chỉ thỉnh thoảng có lỗi nhỏ/không hệ thống. |

### (d) 🎯 Dấu hiệu & phân loại

**Phân loại câu:** Doc yêu cầu *Câu đơn (simple) / Câu ghép (compound) / Câu phức (complex)* → tỉ lệ đơn/phức.

| Mục | Doc gốc | Code thật | Trạng thái |
|---|---|---|---|
| Phân loại simple/compound/complex | Có, kèm tỉ lệ | `classifySentenceComplexity` — client heuristic regex, chỉ phân **đơn/phức** (không có compound riêng) | ⚠️ Đơn giản hóa |
| Tín hiệu band cao: ≥ ~35–40% câu phức 🔧 | Có ngưỡng cứng | Không có ngưỡng — giá trị truyền cho LLM | ❌ |

**Tỉ lệ câu không lỗi:**

| Mục | Doc gốc | Code thật | Trạng thái |
|---|---|---|---|
| Error-free clause ratio | Theo mệnh đề (clause) | `accurateSentencePercent` — theo câu, không theo mệnh đề | ⚠️ Đơn vị khác |
| Hiển thị | "3/4 câu không lỗi" | "X/Y câu" derive từ grammar annotation | ✅ (câu, không phải clause) |

**Phân loại lỗi (error taxonomy):** ✅ — Code mở rộng hơn doc gốc (16 loại thay vì 7):

| Nhóm doc gốc (7) | Code thật (`MockExamGrammarSubtype`, 16 loại) |
|---|---|
| Thì động từ (tense) | `verb_tense` |
| Mạo từ (a/the) | `article` |
| Giới từ | `preposition` |
| Hoà hợp chủ–vị (agreement) | `subject_verb_agreement` |
| Trật tự từ | `word_order` |
| Số ít/nhiều | `singular_plural` |
| Đại từ | (thuộc `subject_verb_agreement` hoặc riêng) |
| *(mở rộng)* | `relative_clause`, `conditional_1/2/3/mixed_conditional`, `modal_passive`, `inversion`, `cleft_sentence`, `reduced_relative`, `participle_phrase`, `gerund_infinitive`, `comparative_superlative` |

**Cấu trúc nâng cao (checklist):** ✅ — `complexStructureUsage` trong grammar profile:
- Code track: `attempted[]`, `successful[]`, `failed[]` — AI sinh qua `runGrammarReview()` (OpenAI structured output, `evaluation-pipeline.ts:784-792`)
- Bao gồm: mệnh đề quan hệ, điều kiện loại 1/2/3/mixed, bị động, đảo ngữ, cleft, reduced relative, participle phrase
- Trạng thái: ✓ *Dùng đúng* (successful) / ⚠ *Chưa đúng* (failed) — có 2/3 trạng thái
- 💡 *Nên thử* (chưa dùng): ❌ — Data chỉ có attempted/successful/failed, không có danh sách "nên thử"

### (e) 🖥️ Nội dung hiển thị

- **Câu trả lời theo câu:** mỗi câu là thẻ — nhãn *Câu đơn/Câu phức* + *Không lỗi/Có lỗi* (viền xanh/cam). ✅
- **Độ chính xác:** câu không lỗi X/Y + tỉ lệ Câu đơn/Câu phức. ✅
- **Cấu trúc nâng cao:** checklist ✓/⚠. ✅ (thiếu 💡)
- **Lỗi theo loại:** Thì/Mạo từ/Giới từ… + số lỗi. ✅ (16 loại, richer than doc)
- **Gợi ý nâng cấp cấu trúc** (câu mẫu nâng cấp): ❌ — Pipeline không sinh câu mẫu

### (f) Ví dụ

- ❌ *Lỗi thì:* "Last week I **carry** her bags" → "**carried**".
- 💡 *Nâng cấp:* "small things can make a big difference" → "**even small gestures can make a real difference**" (đa dạng cấu trúc + từ vựng).
- 💡 *Điều kiện loại 2:* "If I **had** more time, I **would** visit more often."

---

## 9.4 Phát âm (Pronunciation)

### (a) Đo gì

Khả năng tạo **âm (phonemes)** rõ; **trọng âm từ & câu (stress)**; **ngữ điệu (intonation)**; **ngắt nhịp/tiết tấu (chunking & rhythm)**; **nối âm (connected speech)**; và mức **dễ nghe hiểu (intelligibility)** — ảnh hưởng của chất giọng bản ngữ (L1).

### (b) Sub-tiêu chí

Độ rõ ràng · Ngắt nhịp & tiết tấu · Ngữ điệu & trọng âm.

### (c) Kỳ vọng theo band

| Band | Kỳ vọng |
|---|---|
| **5** | Có thể hiểu nhưng lỗi âm/trọng âm thường xuyên; người nghe phải cố gắng; L1 ảnh hưởng rõ. |
| **6** | Dùng được một số đặc điểm phát âm nhưng chưa đều; vài lỗi gây khó nghe cục bộ. |
| **7** | Dùng đa dạng đặc điểm phát âm, duy trì khá tốt; dễ hiểu xuyên suốt; L1 ít ảnh hưởng. |
| **8** | Đa dạng & linh hoạt, chỉ thỉnh thoảng lỡ; rất dễ hiểu; L1 ảnh hưởng tối thiểu. |

### (d) 🎯 Dấu hiệu & ngưỡng phân loại

**Điểm phát âm theo từ (per-word, GOP-based):** ✅ — Azure Cognitive Services `AccuracyScore` (0–100) lưu tại `MockExamTranscriptWord.pronunciationScore`.

| Mức | Doc gốc 🔧 | Code thật (`pronunciation-band.ts:29-30`) | Trạng thái |
|---|---|---|---|
| 🟢 Tốt | ≥ 75% | **> 70** | ⚠️ 70 thay vì 75 |
| 🟡 Khá | 50–74% | **50–70** | ⚠️ 70 thay vì 74 |
| 🔴 Cần cải thiện | < 50% | **< 50** | ✅ |

> Code dùng ngưỡng 70 (không phải 75). Đây là giá trị đang chạy production — doc cũ ghi sai.

**So sánh chuẩn (IPA):** ✅ — `azure-pronunciation.ts:53-101`:
- `buildReferenceIpaFromPhonemes()` — IPA đúng (reference)
- `buildSpokenIpaFromPhonemes()` — IPA bạn nói (từ Azure NBestPhonemes)
- Hiển thị trong UI: ✓ Đúng `/ipa/` + nghe (Google TTS `buildTtsAudioUrl`) + ✗ Bạn nói `/ipa/` + nghe đúng đoạn user nói (`onPlayPrecise`, dùng Azure word-level timestamps)

**Nhóm theo âm (phoneme grouping) — "sửa gốc rễ":** ✅ — `buildPronunciationSoundInsights` (`azure-pronunciation.ts:103-187`):
- Gom lỗi theo *âm* (phoneme) thay vì *từ*
- Trả top 3 weak sounds + top 3 strong sounds
- Mỗi nhóm: `sound` (phoneme), `averageAccuracy`, `occurrenceCount`, `exampleWord`, `exampleWordScore`, `exampleIpa`

| Mục | Doc gốc | Code thật | Trạng thái |
|---|---|---|---|
| Nhóm theo phoneme | ✓ | ✅ `topWeakSounds` | ✅ |
| Cách đặt khẩu hình/lưỡi | ✓ (mỗi nhóm kèm hướng dẫn) | ❌ Chưa có bảng coaching tĩnh | ❌ |
| Ví dụ từ cụ thể | ✓ | ✅ `exampleWord` | ✅ |

**Trọng âm & ngữ điệu:** ⚠️ — Azure trả `ProsodyScore` tổng hợp, dùng trong pronunciation band calibration nhưng **không có breakdown** chi tiết (vd trọng âm từ nào sai). Không itemize per-word.

**Tiết tấu/ngắt nhịp (chunking):** Chỉ có ở mức tổng hợp qua `ProsodyScore`, không phân tích cụ thể.

### (e) 🖥️ Nội dung hiển thị

- **Transcript:** highlight từ yếu (đỏ < 50 / vàng 50–70). ✅ (`pronWordTone`)
- **Lỗi phát âm (nặng → nhẹ):** lưới thẻ từ — từ + số lần + % + ✓ Đúng /ipa/ (nghe) + ✗ Bạn nói /ipa/ (nghe) + ngữ cảnh + **Luyện ›**. ✅
- **Nhóm theo âm (sửa gốc rễ):** mỗi nhóm phoneme + từ ví dụ. ✅ (thiếu hướng dẫn khẩu hình ❌)
- **Mini-drill "Luyện ›":** modal ghi âm lại → chấm Azure, **miễn phí** (không trừ lượt), kết quả standalone không ghi ngược report. ✅

### (f) Ví dụ

- 🔴 *Âm "th":* "the" /ðə/ bị thành /dɛ/ → đặt đầu lưỡi giữa hai hàm răng.
- 🟡 *Trọng âm:* "comfortable" — nhấn âm tiết đầu, không kéo dài giữa.
- 🔴 *Nguyên âm /æ/:* "and" /ænd/ → tránh thành /end/ (mở miệng rộng như "cat").

---

## 9.5 Bảng ngưỡng tổng hợp

> Tất cả ngưỡng dưới đây là **điểm khởi đầu** 🔧; phải calibrate theo phân phối điểm thật và đối chiếu band do giám khảo người chấm.

| Hạng mục | Ngưỡng đề xuất 🔧 | Code thật | Trạng thái |
|---|---|---|---|
| **Pause — tốt / vừa / dài** | ≲0.5s / ~0.5–1.0s / >~1.0s | Context-aware: good ≤1.2s (sentence-end ≤2.0s) / bad >1.0s bí từ, >2.5s extreme | ⚠️ Khác cấu trúc |
| **Tốc độ nói tốt** | ~110–150 từ/phút | Tính raw wpm, truyền LLM; prompt nói "130–170 wpm" | ⚠️ Soft rule |
| **Từ vựng "cao"** | B2+ ≥ ~25–30% tổng từ nội dung | Không có ngưỡng cứng | ❌ |
| **Câu phức "đủ đa dạng"** | ≥ ~35–40% số câu, kèm chính xác | Không có ngưỡng cứng | ❌ |
| **Phát âm theo từ** | Tốt ≥75% · Khá 50–74% · Cần <50% | **Tốt >70 · Khá 50–70 · Cần <50** | ⚠️ 70 thay vì 75 |
| **Lặp từ (cờ)** | một từ nội dung lặp ≥ N lần (N tunable) | Không implement | ❌ |

> **Lưu ý kiến trúc:** Phần lớn ngưỡng trên KHÔNG enforce bằng code cứng (hard-coded rule) — thay vào đó, giá trị thô được truyền cho LLM scoring call như tín hiệu (signal), LLM đưa ra band dựa trên tổng hợp nhiều tín hiệu. Chỉ pronunciation band calibration (`pronunciation-band.ts:37-95`) có ngưỡng cứng (vd `averageAccuracy >= 92 → band 8`), và đây là fallback — Phase D hiện dùng LLM.

---

## 9.6 Việc cần phía pipeline/content để mở khoá nhóm ❌

Để hoàn thiện 100% theo tài liệu học thuật, cần bổ sung **đầu ra pipeline** hoặc **content tĩnh** (tổng hợp từ [`gap.md`](../gap.md)):

| # | Mục | Tiêu chí | Loại cần |
|---|---|---|---|
| 1 | **Coherence rating + depth signal** | Trôi chảy | Pipeline — model trả mức "đủ sâu / liệt kê" + 1 câu nhận xét → badge Mạch lạc |
| 2 | **Filler vị trí inline** | Trôi chảy | Pipeline — đánh dấu offset/wordId của filler trong transcript (không chỉ đếm) |
| 3 | **Idiom/collocation tagging** | Từ vựng | Pipeline — danh sách cụm đáng học + nghĩa, phát hiện trong transcript người học |
| 4 | **Structure suggestions "Nên thử"** | Ngữ pháp | Pipeline — danh sách cấu trúc chưa dùng + câu mẫu nâng cấp |
| 5 | **Phoneme coaching map** | Phát âm | Content tĩnh — bảng "cách đặt khẩu hình/lưỡi" theo nhóm âm |
| 6 | *(Tùy chọn)* **Sentence complexity classifier LLM** | Ngữ pháp | Pipeline — thay heuristic đơn/phức hiện tại |

> Mọi mục ✅ ở trên **build xanh** (`npm run build` status=0). Nhóm ❌ **không làm được client-only** — chờ quyết định mở pipeline/thêm content.

---

## 9.7 Phần bổ sung

### Giả định

- Tất cả ngưỡng 🔧 là **điểm khởi đầu** — cần calibrate theo dữ liệu chấm thật + giám khảo người trước khi fix thành tham số production.
- Band descriptor viết lại theo tinh thần IELTS công khai, **không trích nguyên văn** — cần rà lại với chuyên gia IELTS trước khi phát hành chính thức.
- Sub-score (3 con số mỗi tiêu chí) là do AI sinh trực tiếp theo hiệu suất thật, không phải tính toán/derive từ band tiêu chí (đã xác nhận qua file #07, `evaluate-mock-exam-overall-breakdown.ts`).

### Phụ thuộc

- Report #07 (Tổng quan) — dùng band descriptor + CEFR mapping từ tài liệu này
- Report #08 (Phản hồi chi tiết) — dùng rubric + ngưỡng + taxonomy từ tài liệu này
- Pipeline chấm điểm (`evaluation-pipeline.ts`) — prompt LLM tham chiếu các band anchor từ tài liệu này
- [`gap.md`](../gap.md) — bản tracking chi tiết hiện trạng build theo từng UI element

### Câu hỏi mở thật sự còn treo

1. **Calibrate ngưỡng §9.5** — chờ dữ liệu chấm thật + giám khảo người
2. **Chuẩn IPA mặc định (UK/US)** — code hiện dùng Azure trả gì nhận nấy, chưa có cơ chế cho user đổi chuẩn
3. **Quy tắc gom nhóm âm cho người Việt** — code có `buildPronunciationSoundInsights` nhưng không ưu tiên/nhóm theo lỗi phổ biến của L1 tiếng Việt (vd /θ/ /ð/, âm cuối, /æ/–/e/, /ɪ/–/iː/)

---

## 9.8 Câu hỏi mở — đã chốt

| Câu hỏi (từ §6 doc gốc) | Quyết định | Bằng chứng |
|---|---|---|
| **Nguồn 3 sub-score**: model trả riêng hay derive từ band tiêu chí tổng? | **AI sinh trực tiếp** — LLM trả mảng đúng 3 số (`subScores: number[]`), server kẹp về thang IELTS; client chỉ render nguyên giá trị | File #07 Đổi log #1; `evaluate-mock-exam-overall-breakdown.ts:38-53, 79-91` |
| **Bộ band descriptor tiếng Việt chính thức** cho 3 sub-tiêu chí mỗi criterion | **AI sinh động theo từng user** — LLM nhận `partsJson` (band + feedbackSummary thật) làm ngữ cảnh, sinh free-text tiếng Việt ≤~55 từ; không có enum/lookup tĩnh theo band | File #07 Đổi log #2; `evaluate-mock-exam-overall-breakdown.ts:143-149`, prompt `openai-prompts.json:51` |
| **Cơ sở dữ liệu collocation/idiom & paraphrase** | **Paraphrase: AI sinh** — `replacementOptions` do LLM trả, 3–4 option/suggestion, max 4 suggestions. **Collocation/idiom: chưa auto-detect** — chỉ có từ sample answers | `evaluation-pipeline.ts:450-456`, `vocab-limits.ts` |
| **Cách phát hiện độ sâu phát triển ý** (coherence) | **Chưa implement** — giữ là spec đích, cần pipeline trả tín hiệu coherence depth | `gap.md` §1 — 🔴 |
| Calibrate toàn bộ ngưỡng ở §5 | **Chưa chốt** — chờ dữ liệu chấm thật + giám khảo người | Còn treo |
| Chọn chuẩn IPA mặc định (UK/US) | **Chưa chốt** — code dùng Azure mặc định, chưa có UI đổi chuẩn | Còn treo |
| Quy tắc gom nhóm theo âm cho người Việt | **Chưa chốt** — code gom generic, chưa ưu tiên theo L1 tiếng Việt | Còn treo |
