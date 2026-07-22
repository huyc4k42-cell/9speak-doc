# Practice Optimize — Đánh giá vấn đề tiềm tàng & Kế hoạch triển khai theo phase

> **Mục đích:** Nối tiếp bản *Gap Analysis* (mục A–K). Tài liệu này (1) bổ sung các **vấn đề tiềm tàng** chưa được nêu, (2) chốt các **ràng buộc xuyên suốt**, (3) đưa ra **kế hoạch triển khai chia phase** — mỗi phase kèm điểm chạm code, thay đổi `shared/`, quyết định "Claude tự chốt" cần ghi báo cáo, **cập nhật tài liệu (DoD)**, và tiêu chí nghiệm thu.
> **Nguồn:** `9Speak - PRD-2.docx` (**CANON, v0.2 2026-06-20** — bản mới nhất, gồm 11 doc con #01–#11 + rubric học thuật #11), `PRACTICE_OPTIMIZE_FULL_SPEC.md`, `PRACTICE_ACCEPTANCE_CRITERIA.md`, `practice-flow-optimized.html`, gap analysis (A–K), code thực tế module `practice` + `shared`.
> **Ngày:** 2026-06-22. **Trạng thái:** Draft chờ duyệt (chưa code). Đã đối chiếu PRD-2.docx → xem Phần 0.

---

## Phần 0 — Đối chiếu với PRD-2.docx (bản canon mới nhất)

**Kết luận: KHÔNG mâu thuẫn.** PRD-2 là **superset chính thức** của các spec `.md` (cùng luồng, cấu trúc FR/AC/BR + thêm rubric học thuật). Map 1-1 với phase: #04→P1, #08→P2, #05→P3, #05+#11→P4, #07→P5, #02/#10/#09→P6, #06→P7.

**PRD-2 (đặc biệt #11 rubric) CUNG CẤP SẴN nhiều ngưỡng tôi từng để "Claude tự chốt"** → dùng theo PRD, không tự bịa:
- Pause 3 mức: 🟢 ≲0.5s · 🟡 0.5–1.0s · 🔴 >1.0s (🔢 tunable). ⚠️ **Lệch code hiện tại** (`transcript-pause-classifier` cho "good" tới ~1200ms) → Phase 0 **recalibrate** theo PRD.
- Tốc độ nói: <100 chậm · 110–150 tốt · >170 nhanh (từ/phút, loại filler).
- Phát âm theo từ (GOP): Tốt ≥75% · Khá 50–74% · Cần <50% (khớp code).
- Câu phức "đủ đa dạng": ≥35–40% số câu + chính xác; error taxonomy = đúng enum `GrammarSubtype` đã có; checklist cấu trúc nâng cao (Quan hệ/ĐK1-2-3/Bị động/Đảo ngữ/Cleft → ✓/⚠/💡).
- Từ vựng "cao": B2+ ≥ 25–30%; cờ lặp từ.

**Bổ sung từ PRD-2 cần fold vào plan (không phá plan):**
1. **Sub-tiêu chí (3 sub/criterion)** + band descriptor (#11). #11 để ngỏ "model trả 3 sub-score hay derive". AC #05 **không bắt buộc hiện sub-score** → mặc định **derive/rubric nội bộ**, không thêm field bắt buộc (tránh phình output — xem L2). Nâng cấp sau nếu cần.
2. **Nguồn nội dung** (mẹo/câu mẫu/từ vựng + nghĩa VN + DB collocation/paraphrase) — #01/#03/#11 để ngỏ tĩnh-hay-AI; có **CMS doc riêng** (`CMS_REQUIREMENT_DASHBOARD_USER`) → **dependency** cho coach (#07)/drill (#06), khớp gap B3/E2.
3. **IPA UK/US** (#06) + **off-topic có điểm tham khảo?** (#10) — vẫn để ngỏ, khớp đề xuất "cảnh báo chung" + ước-lượng của tôi.

**PRD-2 xác nhận đúng các rủi ro đã nêu:** #02 max-duration/auto-stop ở recorder (=L5 shared), #08 "so sánh trong phiên hay xuyên phiên" còn ngỏ (=L9), #10 edge + charge (đã chốt), progressive reveal + p50/p95 (=§14).

---

## Phần 1 — Vấn đề tiềm tàng BỔ SUNG (ngoài gap analysis A–K)

> Gap analysis đã xử lý tốt mặt **sản phẩm/nội dung** của B–K. Dưới đây là các rủi ro **kiến trúc & vận hành** sẽ làm vỡ kế hoạch nếu không xử lý trước.

### L1. Gần như MỌI tính năng mới đụng `features/shared/` (contract + pipeline) — KHÔNG phải việc "trong module"
Theo `AGENTS.md`/`docs/README-modules.md`: `shared/` là **READ-ONLY với module**, đổi nó là **breaking change cần duyệt + tương thích ngược**. Nhưng:
- Track-changes (§3), grammar theo câu (D), coherence + liên từ (C4), pause 3 mức (C1), rewrite-diff (E1), tín hiệu edge case (F) → đều cần **field mới trong `shared/types/{practice-evaluation,mock-exam}.ts`** và/hoặc **đổi prompt trong `shared/server/config/openai-prompts.json` + `evaluation-pipeline.ts`**.
- `evaluation.quickScore` / `vocabReview` / `grammarReview` / `wordCefr` là **prompt DÙNG CHUNG với Mock Test**. Sửa output grammar/quickScore để thêm dữ liệu cho Practice **sẽ tác động Mock Test**.
- **Hệ quả kế hoạch:** tách riêng **Phase 0 (nền tảng shared)** làm trước, giữ tất cả field **optional + additive**, không đổi nghĩa field cũ. Mọi thay đổi shared phải build xanh **cả mock-test** và giữ nguyên hành vi mock-test (regression check). Đây là lý do không thể "làm Practice một mình".

### L2. Mâu thuẫn "không thêm lần gọi model" (§12) ⟷ "output giàu hơn" → rủi ro **truncation**
Spec yêu cầu nhồi thêm (grammar theo câu, coherence, connectors, relevance, rewrite-diff) vào **các call có sẵn** để không tăng latency. Nhưng output dài thêm → chạm `AI_SCORING_MAX_TOKENS`, dễ **cụt JSON** (doc đã ghi DeepSeek hay drift/cụt). 
- **Xử lý:** với mỗi call mở rộng schema, đo token thực tế, nâng `maxTokens`/`retryMaxTokens` tương ứng; cân nhắc **giảm `maxErrors`/độ dài explanation** để bù; thêm test schema-validation. Nếu một call phình quá → chấp nhận tách (đánh đổi latency) thay vì cụt.

### L3. Track-changes (§3) — vị trí lỗi đã có, nhưng phụ thuộc chất lượng `wordIds`
Tin tốt: `MockExamTranscriptAnnotation` có `sentenceId` + `wordIds[]` → render inline **theo word, không cần match chuỗi**. Rủi ro còn lại:
- Annotation hiện map từ `corrections` qua `buildAnnotationsFromEvaluation`; cần xác minh **mọi** lexical/grammar error đều ra được `wordIds` đúng (lỗi nhiều từ, lỗi trải qua ranh giới câu, lỗi "thiếu từ" không có word để gạch). 
- **Xử lý:** với lỗi không gắn được `wordIds` (vd thiếu mạo từ) → cơ chế **anchor "chèn giữa 2 từ"** hoặc fallback hiển thị ở cuối câu; định nghĩa rõ trong contract. Đây là rủi ro UI P0 lớn nhất, cần spike nhỏ ở Phase 0.

### L4. `word-cefr` đang **fire-and-forget, fail-silent** — §6.3 biến nó thành tính năng lõi
Khi transcript tô màu CEFR là nội dung chính của card Từ vựng, "fail im lặng" không còn chấp nhận được.
- **Xử lý:** nâng `wordCefrStatus` lên ngang vocab/grammar (pending/completed/failed + retry), nhưng **vẫn không chặn** band tổng. Lưu ý: đây vốn đã là **1 call LLM** riêng (đếm vào chi phí — nhưng đã tồn tại, không phát sinh mới).

### L5. Recorder dùng chung (`useMockExamRecorder` ở `shared/hooks`) — auto-stop/max-duration (B4) là **shared change**
B4 (tự dừng ở giới hạn + đếm ngược) và xử lý quyền mic (A1/A2) nằm ở recorder dùng chung với Mock Test.
- **Xử lý:** thêm **tham số tùy chọn** (`maxDurationMs`, callback `onAutoStop`, expose `permissionState`/`deviceError`) — mặc định giữ nguyên hành vi cũ để mock-test không đổi. Không hard-code ngưỡng trong recorder.

### L6. Edge case "thật" (im lặng / nhiễu / sai ngôn ngữ / lệch đề) — chi phí & thời điểm phát hiện
- **Im lặng / quá ngắn:** suy từ transcript rỗng + word offsets (đã có) → rẻ, làm được client/post-Azure.
- **Sai ngôn ngữ:** Azure trả transcript rác/độ tin thấp cho non-English → heuristic (tỉ lệ từ không match, confidence). Không có language-detect riêng → chấp nhận "có vẻ…".
- **Lệch đề (C1) & nhiễu:** chỉ biết **sau khi đã gọi Azure** (đã phát sinh chi phí Azure). Lệch đề cần tín hiệu liên quan đề bài.
- **Xử lý (khớp gap F):** đợt đầu **cảnh báo chung** ("có vẻ lệch đề"), KHÔNG nêu tên chủ đề. Để tránh call mới, **fold 1 field `relevance` vào output `quickScore`** (đang fake bằng `relevanceScore` theo duration — thay bằng tín hiệu thật từ LLM đã chạy). Ngưỡng & quy tắc → ghi báo cáo (xem Phần 4).

### L7. Charge ledger — quy tắc mới phải **idempotent** và không trừ nhầm
Hiện trừ khi `scoreDone` thành công. Yêu cầu mới (gap H): drill 1 từ không trừ; lượt không-ra-điểm (quá ngắn/sai ngôn ngữ) không trừ; lệch đề **vẫn ra điểm** → cần chốt có trừ hay không.
- **Xử lý:** drill 1 từ đi qua **endpoint riêng KHÔNG chạm ledger** (giống vocab/grammar). Edge case chặn **trước** khi gọi quickScore (quá ngắn) → đương nhiên không trừ. Lệch đề: đề xuất **vẫn trừ** (vì đã tốn Azure + quickScore và vẫn trả điểm) nhưng cần user chốt — ghi báo cáo.

### L8. Backward-compat history (I2) + persistence round-trip
Field mới optional, nhưng phải **round-trip qua NineSpeak persistence** (`practice-persistence` + `aiFeedbackPerformance`) và **rehydrate** đúng khi mở Lịch sử. Bài cũ thiếu field → UI hiện dòng "không có dữ liệu cho lượt này" (gap I2), **tuyệt đối không chấm lại hàng loạt**.
- **Xử lý:** mỗi field mới đi kèm guard "có data?"; thêm cờ phân biệt "bài cũ" (thiếu cả cụm) vs "đang chờ" (pending). Verify serializer không nuốt field lạ.

### L9. Mini trend xuyên phiên (§5) cần dữ liệu nhiều lượt của **cùng câu hỏi**
Trong phiên: `PracticeSessionStore` đã giữ. Khi **mở lại sau refresh / từ Lịch sử**: cần lịch sử các lượt cùng `questionId` để vẽ trend + tính delta.
- **Xử lý:** xác nhận `history-client` trả được chuỗi lượt theo câu; nếu không, trend chỉ hiện trong phiên (ghi rõ giới hạn). Không đầu tư backend mới ở đợt này.

### L10. Thiếu **timeout phía client** → nguy cơ "treo vô hạn" (§14.5)
Code hiện không có timeout rõ ở `useSpeakingPractice` cho kết quả đầu/phân tích sâu.
- **Xử lý:** thêm timeout (đề xuất ~45s kết quả đầu, ~60s phân tích sâu) + nút thử lại; đo & track thời gian (§14.6).

### L11. Compact UI (§9) là **đại tu cột giao diện** — surface lớn, rủi ro regress layout
Đụng hầu hết component trong `features/practice/components/practice/**`. Nên tách phase riêng, làm sau khi cấu trúc dữ liệu/khối chức năng ổn định, để không phải sửa style 2 lần.

### L12. TTS trình duyệt (gap K1) không đồng đều
`speechSynthesis` thiếu giọng en-US trên một số máy VN → "nghe mẫu/âm chuẩn" có thể câm.
- **Xử lý:** feature-detect giọng en; nếu thiếu → ẩn nút/ą fallback thông báo. Chấp nhận cho đợt đầu, ghi vào known-limitations.

### L13. Analytics (§13) — tuân thủ PII
Nhiều event mới. Theo memory & `docs/env-and-modules.md`: **email/tên chỉ Mixpanel, không GA4**. Event mới phải đi qua wrapper analytics của shared, không nhét PII.

---

## Phần 2 — Ràng buộc xuyên suốt (áp cho mọi phase)

1. **Quy trình nhánh (bắt buộc):** mỗi phase = 1+ nhánh con tách từ `dev` (`feat/practice-...`). KHÔNG sửa thẳng `main`/`dev`. Build xanh (`npm run build` + `tsc --noEmit` + `eslint` file đổi) trước commit. **KHÔNG tự merge** khi đụng `shared/` — chờ owner duyệt.
2. **Tài liệu là một phần của DoD:** kết thúc mỗi phase phải cập nhật tài liệu liên quan (liệt kê trong từng phase) **cùng nhịp commit code**.
3. **Shared additive-only:** field/scope mới đều optional, không đổi nghĩa cũ; build + smoke-test **mock-test** mỗi lần đụng shared.
4. **Không thêm lần gọi model** trừ khi bất khả kháng (§12); ưu tiên fold vào call sẵn + tra tĩnh + lazy-load.
5. **An toàn dữ liệu:** không chấm lại hàng loạt history; ledger idempotent; không log/secret; PII chỉ Mixpanel.
6. **"Claude tự quyết" ⇒ ghi báo cáo:** mọi định nghĩa nghiệp vụ Claude tự chốt (D, F, H, ngưỡng…) ghi vào **`PRACTICE_OPTIMIZE_DECISIONS_REPORT.md`** (Phần 4) khi triển khai xong phase tương ứng.

---

## Phần 3 — Kế hoạch theo phase

> Thứ tự = giảm rủi ro: nền tảng shared & P0 trước, rồi P1 theo cụm dữ liệu, P2 sau. Map ưu tiên spec: Phase 1 = Đợt 1 (P0); Phase 2–6 = Đợt 2 (P1); Phase 7 = Đợt 3 (P2).

### PHASE 0 — Nền tảng `shared` (contract + pipeline) & spike track-changes  ✅ **DONE (2026-06-22, nhánh `feat/practice-shared-foundation`, chờ duyệt merge)**
> Đã làm: pause 3 mức theo trường độ (`pauseDurationLevel`, tách khỏi scoring) + spike track-changes/token. Build xanh. Chi tiết & phát hiện: [PRACTICE_OPTIMIZE_DECISIONS_REPORT.md](./PRACTICE_OPTIMIZE_DECISIONS_REPORT.md). **Đính chính:** contract field khác (grammar theo câu, coherence, rewrite-diff) **dời sang phase tiêu thụ** (additive khi cần) thay vì nhồi hết ở Phase 0 — tránh field chết mock-test phải nhìn.

**Mục tiêu:** mở đường an toàn cho mọi phase sau; không có UI người dùng đổi.
**Phạm vi/điểm chạm:**
- `shared/types/practice-evaluation.ts` + `mock-exam.ts`: thêm (optional) field cho: sentence-level grammar (đơn/phức + có/không lỗi), coherence + connectors, pause 3 mức, rewrite-diff metadata, relevance/edge signals. Anchor track-changes (mở rộng annotation nếu cần cho lỗi "chèn").
- `shared/lib/transcript-pause-classifier.ts`: thêm mức **"medium"** (`none|good|medium|bad`) + ngưỡng cấu hình.
- **Spike** (nhỏ): xác minh `buildAnnotationsFromEvaluation` ra `wordIds` đúng cho các loại lỗi; chốt cơ chế anchor lỗi "thiếu từ".
**Quyết định ghi báo cáo:** ngưỡng pause 3 mức (good/medium/long ms); cấu trúc anchor lỗi chèn.
**Cập nhật tài liệu:** `scoring-pipeline-overview.md` (pause 3 mức), `docs/modules/practice/TECH.md` §4 (contract mới), `docs/modules/shared/TECH.md` nếu có; ghi rõ "additive, mock-test không đổi hành vi".
**Nghiệm thu:** `npm run build` xanh cho cả practice + mock-test; type mới optional; pause classifier có test 3 mức.

### PHASE 1 — Đợt 1 / P0: Bố cục mới + Câu đã sửa + Edge case mic/quá ngắn
**Map spec:** §2 (bỏ tab + bỏ "xem chi tiết"), §3 (track-changes + popup), §8 A1/A2/B1.
**Phạm vi/điểm chạm (chủ yếu trong `features/practice/`):**
- `components/practice/PracticeEvaluationPanel.tsx`: bỏ 3 tab + nút "Xem chi tiết"/modal; dựng khối: điểm tổng → 4 thẻ điểm → "Câu trả lời đã sửa lỗi" (render inline theo `wordIds`, popup giải thích) → nút cải thiện (chỗ giữ).
- `hooks/useSpeakingPractice.ts`: bỏ luồng `requestDetailedEvaluation`/`detailStatus` (hoặc giữ no-op tương thích); skeleton "AI đang phân tích & sửa lỗi…".
- Edge A1/A2: dùng `permissionState`/`deviceError` từ recorder (cần Phase L5 nhẹ ở đây hoặc tối thiểu bắt lỗi getUserMedia) → card chuyên biệt + "Hướng dẫn cấp quyền". B1 (quá ngắn) đã có chặn <3s → đổi thành **card chuyên biệt** thay banner.
- Component mới: `CorrectedAnswer`, `EdgeCaseCard` (tái dùng cho phase sau).
**Quyết định ghi báo cáo:** ngưỡng "quá ngắn" (giây/từ, theo Part) — đề xuất tính theo **offset từ cuối − từ đầu** (thời gian nói thực), không lấy độ dài file.
**Cập nhật tài liệu:** `docs/modules/practice/PRD.md` §3.3/§8 (bỏ tab, thêm câu-đã-sửa, edge case), `TECH.md` §6 (bỏ detailStatus), `features/practice/AGENTS.md` (component mới).
**Nghiệm thu:** checklist §2/§3/§10 (A1,A2,B1) trong `PRACTICE_ACCEPTANCE_CRITERIA.md`.

### PHASE 2 — Đợt 2 / P1: So sánh lượt (delta + động viên + mini trend + auto-collapse)
**Map spec:** §5, §7 (Đợt2 cũ G). 
**Phạm vi:** `usePracticeAttemptHistory` (đã có `isImprovement`) + timeline components: chip delta band + delta 4 thẻ, dòng động viên (tăng/giảm/bằng), **mini trend cố định trên khu chat**, lượt cũ auto-collapse + bỏ bong bóng transcript lặp.
**Phụ thuộc:** L9 (dữ liệu nhiều lượt). Trong phiên đủ; xuyên phiên xác nhận history-client.
**Cập nhật tài liệu:** `PRD.md` §3.5/§3.6, `TECH.md` §6.2 (timeline/trend).
**Nghiệm thu:** §5/§9 acceptance.

### PHASE 3 — Đợt 2 / P1: Phân tích 4 tiêu chí (rail + panel rộng) — phần TÁI DÙNG dữ liệu sẵn có
**Map spec:** §6.1 (phát âm: phân bố Tốt/Khá/Yếu, top từ, nhóm theo âm, transcript theo band, danh sách lỗi 2 nút nghe), §6.2 (tốc độ wpm + gauge + pause 3 mức + filler), §6.3 (CEFR coloring + phân bố). **Bỏ** tô chữ con (gap B1), **bỏ** so trọng âm (gap B2).
**Phạm vi:**
- UI: pattern "rail gọn + ⛶ panel rộng" cho 4 thẻ (mở ở cột AI hỗ trợ). Phát âm/Fluency dùng dữ liệu Azure đã có (phoneme/IPA/score/timestamp/sound-insights). 
- Tốc độ nói: đổi sang **thời gian nói thực** (offset cuối − đầu) thay vì duration file (gap F note).
- Pause 3 mức: tiêu thụ output Phase 0.
- CEFR: nâng `wordCefrStatus` thành first-class (L4) + retry; transcript tô màu + phân bố + card paraphrase (từ `lexicalSuggestions` đã có).
- Per-word audio "nghe lại" (clip theo timestamp, client) + "âm chuẩn" (TTS, L12).
**Quyết định ghi báo cáo:** ngưỡng band từ (Tốt≥75/Khá50–75/Yếu<50 — đã ở spec, xác nhận), mức pause.
**Cập nhật tài liệu:** `PRD.md` §3.3, `TECH.md` §5/§6, `scoring-pipeline-overview.md` (word-cefr first-class).
**Nghiệm thu:** §6/§7.1/§7.2/§7.3 acceptance.

### PHASE 4 — Đợt 2 / P1: Ngữ pháp chi tiết + Coherence/Connectors (đụng prompt shared)
**Map spec:** §6.4 (câu đơn/phức, câu không lỗi, cấu trúc nâng cao trạng thái, lỗi theo loại, gợi ý có ví dụ — gap D), C4 (mạch lạc + liên từ đã dùng + gợi ý).
**Phạm vi (đụng `shared/` — cần duyệt):**
- `openai-prompts.json` + `evaluation-pipeline.ts`: mở rộng `grammarReview` (sentence-level classification + cấu trúc nâng cao trạng thái) và **fold coherence/connectors** vào `grammarReview` hoặc `quickScore` (chọn nơi ít phình nhất; xem L2). Field mới optional.
- UI ngữ pháp panel + phần Mạch lạc trong card Fluency.
- Kiểm regression Mock Test (cùng prompt).
**Quyết định ghi báo cáo (gap D — bắt buộc):** định nghĩa "câu không lỗi", quy tắc phân loại đơn/phức, danh sách cấu trúc "nên thử" theo band; nêu rõ UI là "đánh giá tương đối".
**Cập nhật tài liệu:** `scoring-pipeline-overview.md` §6.4 (schema grammar mới + coherence), `PRD.md`/`TECH.md` practice, và **note tác động mock-test**.
**Nghiệm thu:** §6.4/§7.4 acceptance + mock-test không regress.

### PHASE 5 — Đợt 2 / P1: Coach cải thiện câu (3 mục tiêu + diff metadata) (đụng prompt shared/Practice)
**Map spec:** §7 (E1): 3 mục tiêu (nâng từ vựng/đa dạng ngữ pháp/phát triển ý), diff tô màu + lý do/kỹ thuật, "đã nâng gì", "cụm đáng học", nghe mẫu, thử nói lại. Bỏ số band cụ thể (gap E, §2.5).
**Phạm vi:**
- `practice.answerRewrite` (prompt RIÊNG Practice — an toàn hơn): đổi từ 2 mode `{improve,expand}` → **3 goal** trả `answer` + **`changes[]`** (vị trí/loại/lý do/kỹ thuật) + cụm đáng học + dịch. **Gọi đúng mục tiêu user bấm, mỗi lần 1 call** (gap E1). Rewrite **không trừ lượt** (giữ nguyên).
- `evaluate-practice.ts` + route `/rewrites` + types + UI coach card/panel.
- "Cụm đáng học → flashcard": **chưa làm** (gap E2, chờ backend) nhưng giữ chỗ UI + nút (disabled/"sắp có") — kế hoạch chi tiết ghi ở Phần 4.
**Quyết định ghi báo cáo:** schema `changes[]`; ánh xạ "đã nâng gì" suy từ metadata (không call thêm).
**Cập nhật tài liệu:** `scoring-pipeline-overview.md` §6.5 (rewrite diff), `PRD.md` §3.4, `TECH.md` (prompt rewrite mới).
**Nghiệm thu:** §8 acceptance (coach).

### PHASE 6 — Đợt 2 / P1: Edge case còn lại + Max-duration + Timeout + Compact UI + Analytics
**Map spec:** §8 B2/B3/B4/B5/C1, §9 compact, §13 analytics, §14 performance/timeout.
**Phạm vi:**
- B2/B3/B5/C1: tín hiệu thật (transcript rỗng/confidence/relevance fold vào quickScore — L6). C1 cảnh báo chung, vẫn hiện transcript, chỉ cho ghi lại.
- B4: max-duration + countdown auto-stop ở recorder (L5, shared param) + nhãn "Đã tự dừng".
- L10 timeout client + §14.6 track thời gian.
- §9 compact restyle (L11) — làm cuối để không sửa style 2 lần.
- §13 analytics events (L13, PII-safe).
**Quyết định ghi báo cáo (gap F/H):** ngưỡng quá-ngắn/quá-dài theo Part; ngưỡng "lệch đề"; tín hiệu im lặng/nhiễu/sai-ngôn-ngữ; thứ tự ưu tiên khi nhiều lỗi; **quy tắc trừ lượt** (drill không trừ; không-ra-điểm không trừ; lệch đề trừ/không — chờ user).
**Cập nhật tài liệu:** `PRD.md` §8 (bảng edge case + recorder), `TECH.md` (recorder param, timeout), `env-and-modules.md` nếu thêm cờ, `docs/luot-cham-diem-cho-product.md` (quy tắc trừ lượt mới), analytics doc.
**Nghiệm thu:** §10/§11/§13/§14 acceptance.

### PHASE 7 — Đợt 3 / P2: Drill luyện âm từng từ + flashcard (khi backend sẵn)
**Map spec:** §6.1 drill (MẪU vs BẠN phoneme + hướng dẫn khẩu hình + "Ghi âm ngay" chấm riêng từ), E2 flashcard.
**Phạm vi:**
- Drill popup (đè panel rộng): từ + IPA + nghĩa + MẪU/BẠN (phoneme đã có từ Azure NBest) + hướng dẫn khẩu hình (**bộ tĩnh Claude biên soạn** — gap B3) + nghĩa VN (từ điển có sẵn, else AI).
- "Ghi âm ngay": **endpoint chấm 1 từ** (Azure scripted assessment với referenceText = từ đó) — **không chạm ledger** (L7); popup chấm đè lên popup cũ (gap B5).
- Flashcard (E2): triển khai khi backend lưu xong.
**Quyết định ghi báo cáo:** nguồn nghĩa VN; phạm vi bộ hướng dẫn khẩu hình.
**Cập nhật tài liệu:** `PRD.md`/`TECH.md` (endpoint drill, ledger note), bộ hướng dẫn khẩu hình (tài liệu nội dung riêng).
**Nghiệm thu:** §7.5 acceptance + ledger: drill không trừ lượt.

---

## Phần 4 — Sổ "Claude tự quyết" (điền dần thành báo cáo)

> Tạo file `PRACTICE_OPTIMIZE_DECISIONS_REPORT.md` và điền khi mỗi phase xong. Các mục **bắt buộc** user yêu cầu báo cáo:

> **Cập nhật sau đối chiếu PRD-2:** nhiều mục dưới đã được **PRD #11 (rubric) cung cấp ngưỡng** → không còn là "Claude tự chốt", chỉ cần *áp dụng + ghi giá trị thực tế đã calibrate*. Vẫn ghi báo cáo để truy vết.

| Mã | Quyết định cần chốt | Phase | Trạng thái |
|---|---|---|---|
| D-1 | Định nghĩa "câu không lỗi" (mức lỗi nào tính là lỗi) | 4 | 🟦 PRD #11 định hướng; chốt mức cụ thể |
| D-2 | Quy tắc phân loại câu đơn / câu phức | 4 | 🟦 PRD #11: phức ≥35–40% — áp dụng |
| D-3 | Danh sách cấu trúc "nên thử" theo band | 4 | 🟦 PRD #11 checklist — áp dụng |
| F-1 | Ngưỡng "quá ngắn" (giây/từ, theo Part) + dùng thời gian nói thực | 1/6 | ⬜ chờ (PRD để ngỏ ngưỡng theo Part) |
| F-2 | Ngưỡng "lệch đề" + chỉ cảnh báo chung (không nêu chủ đề) | 6 | ⬜ chờ (PRD #10 để ngỏ điểm tham khảo) |
| F-3 | Tín hiệu im lặng / nhiễu / sai ngôn ngữ | 6 | ⬜ chờ |
| F-4 | Thứ tự ưu tiên khi nhiều lỗi cùng xảy ra | 6 | ⬜ chờ |
| C1-1 | Ngưỡng pause 3 mức (good/medium/long ms) | 0/3 | 🟦 PRD #11: ≲0.5 / 0.5–1.0 / >1.0s — recalibrate code |
| SUB-1 | Sub-tiêu chí (3/criterion): model trả hay derive? | 3/4 | ⬜ chờ (mặc định derive, không field bắt buộc) |
| IPA-1 | Chuẩn IPA hiển thị UK/US + cơ chế đổi | 7 | ⬜ chờ |
| H-1 | Drill 1 từ không trừ lượt | 7 | ⬜ chờ |
| H-2 | Lượt không-ra-điểm (quá ngắn/im lặng, chặn trước chấm) **không trừ**; lượt **lệch đề / sai ngôn ngữ** (đã chấm, vẫn ra điểm) **VẪN TRỪ** — *user chốt 2026-06-22* | 6 | ✅ **đã chốt** |
| B3-1 | Nguồn nghĩa VN + phạm vi bộ hướng dẫn khẩu hình | 7 | ⬜ chờ |

**Đã chốt (user, 2026-06-22):** edge case **chặn trước khi gọi quickScore** (quá ngắn, im lặng) → không trừ. Edge case **phát hiện sau khi đã chấm** (lệch đề, sai ngôn ngữ — đã tốn Azure + quickScore + vẫn trả điểm) → **vẫn trừ 1 lượt**. Hệ quả Phase 6: không cần thêm logic hoàn/miễn-trừ cho lệch-đề/sai-ngôn-ngữ; chỉ đảm bảo các lỗi *pre-score* không bao giờ chạm ledger (vốn đã đúng).

---

## Phụ lục — Bảng "feasibility" nhanh (đã verify trong code)

| Yêu cầu | Dữ liệu đã có? | Ghi chú |
|---|---|---|
| Track-changes theo từ (§3) | ✅ `annotation.wordIds[]` | Rủi ro: lỗi "thiếu từ" cần anchor chèn (L3) |
| % điểm từng từ, phân bố Tốt/Khá/Yếu (§6.1) | ✅ `word.pronunciationScore` | Đã tính good/fair/poor |
| IPA đúng + "âm bạn nói" (B4/B5) | ✅ `referenceIpa` + NBest phonemes | "Bạn nói" là ước lượng — trình bày dạng ước lượng |
| Nhóm theo âm (§6.1) | ✅ `buildPronunciationSoundInsights` | Đã có top weak/strong sounds |
| Timestamp từ → nghe lại/clip | ✅ `startTimeSeconds/endTimeSeconds` | Clip client-side |
| Trọng âm vị trí (B2) | ❌ Azure không trả | **Bỏ** đợt đầu |
| Tô chữ con trong từ (B1) | ❌ không map âm→chữ | **Bỏ**, tô cả từ/âm tiết IPA |
| Pause 3 mức (C1) | ⚠️ hiện 2 mức | Thêm "medium" (Phase 0) |
| Vị trí filler inline (C3) | ⚠️ Azure hay lọc filler | Hiện tổng trước; vị trí nếu kiểm chứng được |
| CEFR từng từ (§6.3) | ✅ `evaluation.wordCefr` | Nâng thành first-class (L4) |
| Grammar theo câu (D) | ❌ chưa sinh | Mở rộng `grammarReview` (Phase 4, shared) |
| Coherence + connectors (C4) | ❌ chưa sinh | Fold vào call sẵn (Phase 4) |
| Rewrite diff metadata (E1) | ❌ chỉ {answer,summary} | Mở rộng `practice.answerRewrite` (Phase 5) |
| Tốc độ nói thực (F note) | ⚠️ đang dùng duration file | Đổi sang offset cuối−đầu (Phase 3) |
| Flashcard (E2) | ❌ chưa có chỗ lưu | Chờ backend (Phase 7) |
