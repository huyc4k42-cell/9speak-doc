# Practice Module — PRD (Mô tả nghiệp vụ)

> Tài liệu nghiệp vụ chi tiết cho **module Practice** (`features/practice/`) — luồng luyện tập IELTS Speaking từng câu theo Part 1/2/3. Đối tượng đọc: AI agent và đầu mối phụ trách module, cần hiểu đầy đủ nghiệp vụ + ràng buộc liên-module (đặc biệt phụ thuộc vào `features/shared/` và `features/billing/`).
>
> Cặp tài liệu: xem thêm [TECH.md](./TECH.md) cho đặc tả kỹ thuật.
>
> Nền tảng tham chiếu: [README-modules](../../README-modules.md), [AGENTS.md của module](../../../features/practice/AGENTS.md), [scoring-pipeline-overview](../../scoring-pipeline-overview.md), [env-and-modules](../../env-and-modules.md).

---

## 1. Tổng quan & mục tiêu

Practice là module giúp học viên **luyện từng câu hỏi IELTS Speaking** theo đúng cấu trúc 3 Part của kỳ thi, nhận **feedback AI tức thì** sau mỗi lượt nói và **cải thiện dần** qua nhiều lượt làm lại.

Khác với Mock Test (thi thử trọn bộ, chấm cả phiên), Practice tập trung vào **vòng lặp ngắn**: chọn câu hỏi → ghi âm → chấm → xem lỗi → viết lại mẫu → thu lại. Mỗi câu hỏi là một đơn vị độc lập, người dùng có thể lặp lại bao nhiêu lần tuỳ thích (trong giới hạn lượt chấm).

Mục tiêu nghiệp vụ:

- Cho phép luyện **đúng cấu trúc từng Part** (Part 1 hỏi-đáp nhanh, Part 2 cue card nói dài, Part 3 thảo luận chiều sâu).
- Trả band điểm + 4 tiêu chí IELTS (Fluency & Coherence, Lexical Resource, Grammatical Range & Accuracy, Pronunciation) **ngay sau khi nói**.
- Bóc lỗi và gợi ý nâng cấp **Từ vựng** + **Ngữ pháp**, tô màu CEFR từng từ, đánh dấu từ phát âm yếu; cho **luyện lại phát âm từng từ** (drill) ngay trên kết quả.
- Cung cấp **câu trả lời mẫu của coach** (rewrite) theo nhiều mục tiêu (*Hay hơn* / *Dài & sâu hơn* / *Tự nhiên hơn*) kèm đối chiếu thay đổi, cụm đáng học, dịch nghĩa và nghe mẫu.
- Lưu **lịch sử lượt làm**, hiện **delta + xu hướng** và đánh dấu lượt nào là **cải thiện** so với lượt trước.

Module là một **đơn vị độc lập** về prompt chấm điểm, API endpoint và sổ đối soát quota (ledger riêng `practice_scoring_charges`).

---

## 2. Người dùng & vai trò

| Vai trò | Mô tả | Quyền trong Practice |
|---|---|---|
| **Học viên (đăng nhập)** | Người dùng chính, đã xác thực Firebase | Chọn topic/câu hỏi, ghi âm, chấm điểm (trừ lượt), xem lịch sử, rewrite, nạp lượt |
| **Khách chưa đăng nhập** | Chưa có Firebase ID token | Bị chặn ở mọi endpoint chấm (HTTP 401). Không dùng được tính năng cốt lõi |
| **Học viên hết lượt chấm** | Đã hết quota NineSpeak | Vẫn ghi âm + lưu lượt ở trạng thái `pending`, nhưng phải nạp lượt (paywall) để chấm |
| **Admin / Prompt Lab** | Nội bộ, qua trang `debug/` | Gửi `promptOverrides` + `aiScoringOverrides` để thử nghiệm prompt (không phải luồng người dùng thật) |

Yêu cầu xác thực: mọi request chấm điểm cần **Firebase ID token** (bearer). Trừ lượt cần thêm **NineSpeak access token** (`X-NineSpeak-Access-Token`); nếu không có token → chấm vẫn chạy nhưng ledger ghi `skipped-no-token` (không trừ).

---

## 3. Phạm vi tính năng chi tiết

### 3.1 Chọn topic / câu hỏi
- Trang chủ Practice ([PracticeIndex](../../../features/practice/pages/PracticeIndex.tsx)) tải **catalog** chủ đề luyện tập từ NineSpeak (qua BFF `/api/app/speaking/practice/catalog`), nhóm theo từng Part (tab "Part 1" / "Part 2" / "Part 3").
- Mỗi topic có danh sách câu hỏi; chọn câu hỏi sẽ điều hướng tới trang luyện tập tập trung của Part tương ứng kèm `topicId` + `questionId` (+ `sessionId` khi resume) trên URL.
- Nội dung câu hỏi đến từ **content store của shared** qua BFF; có cache TTL 15s phía client ([practice-topic-runtime](../../../features/practice/lib/practice-topic-runtime.ts)). Phân trang câu hỏi theo Part: Part 1 và Part 3 lấy 3 câu/trang, Part 2 lấy 1 câu/trang.

### 3.2 Ghi âm
- Dùng recorder dùng chung `useMockExamRecorder` (shared) bọc trong [usePracticeRecorderControls](../../../features/practice/hooks/usePracticeRecorderControls.ts) / [useSpeakingPractice](../../../features/practice/hooks/useSpeakingPractice.ts).
- Ghi WAV phía client; chặn đoạn ghi **< 3 giây** (Azure thường trả transcript rỗng với audio quá ngắn).
- Upload audio: ưu tiên **presigned R2** (client → R2 một lần, dùng chung cho cả chấm điểm lẫn lưu lịch sử, né giới hạn body 4.5MB). Nếu presigned lỗi → fallback gửi bytes inline.

### 3.3 Chấm điểm (Quick Score + Vocab/Grammar parallel)
- Sau khi dừng ghi, client gọi `/assessments`. Có **2 luồng chấm**, chọn bằng env `PRACTICE_PIPELINE` (chỉ practice, không ảnh hưởng mock-test — xem [scoring-redesign-decision.md](./scoring-redesign-decision.md)):
  - **Baseline (mặc định):** **Azure Speech** → **Pronunciation calibration (LLM)** → **Quick Score (LLM)** → band + 4 tiêu chí + tóm tắt + improvements.
  - **Mới `PRACTICE_PIPELINE=deepgram` (tối ưu tốc độ/chi phí):** **Deepgram** (transcript + timestamp + confidence, qua `transcribeUrl` từ R2 — function không tải bytes) → **DeepSeek calibrate** band phát âm (từ timestamp/pause) → **DeepSeek Quick Score** → band + 4 tiêu chí. **KHÔNG** sinh tóm tắt/improvements (UI không hiển thị) và **KHÔNG** chấm phát âm chi tiết ở bước này — band ra ngay từ DeepSeek, **phát âm (Azure) chạy NGẦM tự động sau đó** (không chặn hot-path, không chờ user bấm tab — §3.5). Bỏ Azure khỏi hot-path ⇒ tới-band nhanh ~3×. Calibrate/quickScore/rewrite/vocab/grammar đều DeepSeek.
- Cả 2 luồng đều: **"điểm khoá" (locked score)**, **trừ lượt qua ledger** (`scoreDone`) sau khi có điểm, **kiểm quota + paywall** (402) trước, **lưu history** để xem lại — logic dùng chung, không phụ thuộc luồng.
- Ngay khi có điểm, client **fire song song** 2 request: `/vocab` (phân tích từ vựng) và `/grammar` (phân tích ngữ pháp) bằng `Promise.allSettled`. Hai tab populate độc lập, không chờ tuần tự.
- Song song nữa: `/word-cefr` (tô màu CEFR từng từ) chạy **fire-and-forget**, fail im lặng, không ảnh hưởng UI chính.

### 3.4 Câu trả lời "đã sửa lỗi" (track-changes)
- Ngay trong khu kết quả, sau khi vocab+grammar xong, hiện khối **"Câu trả lời của bạn — đã sửa lỗi"** ([PracticeCorrectedAnswer](../../../features/practice/components/practice/PracticeCorrectedAnswer.tsx)): lỗi từ vựng (critical) + ngữ pháp được **gạch bỏ đỏ** và **bản sửa xanh inline** (track-changes), bấm vào → popup giải thích (dùng `AnnotationPopup` của shared).
- Lỗi **"thiếu từ"** (missing-word, không có `wordIds`) được neo vào từ kế cận trong `correctedText` qua `findGrammarAnchorMapping` (xử lý cục bộ trong Practice, không đụng scoring shared) để vẫn render được.

### 3.5 Phân tích từng tiêu chí (criterion cards)
> ⚙️ **Luồng `PRACTICE_PIPELINE=deepgram`:** kết quả ban đầu **KHÔNG có chi tiết phát âm** (band ra trước, transcript chưa tô màu phát âm). **Phát âm (Azure scripted) chạy NGẦM tự động** ngay sau khi các phân tích DeepSeek (vocab/grammar/fluency/word-cefr) xong — KHÔNG chờ user bấm tab. Mở tab **Phát âm** mà chưa xong → hiện **loading TOÀN BỘ box** ("Đang chấm phát âm…") tới khi có kết quả; xong → điểm phát âm từng từ + phoneme + từ yếu. **Azure CHỈ hiển thị chi tiết, KHÔNG tính/đổi band tổng.** Để tab **Từ vựng** (đếm CEFR từ `word.cefrLevel`) và tab **Phát âm** KHÔNG ghi đè nhau, hai bên dùng **2 map riêng**: `transcriptSentences` (cefr + pause, từ Deepgram/word-cefr) vs `pronunciationSentences` (điểm phát âm Azure) — bản cũ/baseline không có map riêng thì tab Phát âm tự fallback `transcriptSentences` (tương thích ngược). Tab **Trôi chảy** lấy pause good/bad + filler từ **timestamp Deepgram**. Lượt history vẫn chấm phát âm được (dùng URL audio R2). Mở tab khi status `idle`/`failed` (history/eager lỗi) thì click vẫn gọi như cũ. Luồng baseline thì phát âm có sẵn ngay.

- 4 thẻ điểm IELTS là **nút bấm** → mở **card phân tích tiêu chí ở cột "AI hỗ trợ"** ([PracticeCriterionCard](../../../features/practice/components/practice/PracticeCriterionCard.tsx) qua [PracticeAssistantRail](../../../features/practice/components/practice/PracticeAssistantRail.tsx)). Không còn 3 tab "Phân tích bài nói" + modal "Xem chi tiết" cũ.
- Nội dung từng card (insight component riêng):
  - **Phát âm** ([PracticePronunciationInsight](../../../features/practice/components/practice/PracticePronunciationInsight.tsx)): card rail GỌN bám mockup `pronCard` — **3 pill** Tốt/Khá/Yếu + **"Từ cần sửa nhất"** (list gọn: từ ×N + % màu, click → drill) + box tím **"Âm yếu nhất"**. Panel "Bản đầy đủ" bám mockup `openPronModal` 2 cột: TRÁI = **"Câu trả lời theo độ chuẩn"** (transcript tô màu TỪNG TỪ theo `word.pronunciationScore`: đỏ <50 / vàng 50–75 / xanh ≥75, click từ yếu → drill) + audio + legend · PHẢI = **"Lỗi phát âm (nặng → nhẹ)"** (thẻ ✓ Đúng /ipa/ + ✗ Bạn nói /ipa/ + nút nghe Mẫu/Bạn + câu ngữ cảnh + nút **"Luyện ›"**) + **"Nhóm theo âm"** (gom theo phoneme yếu nhất, chip từ → drill). Mảnh tái dùng (`PronunciationBandTranscript/PronunciationBandLegend/PronunciationMistakeCards/PronunciationSoundCards`). *Phần "Trọng âm & Ngữ điệu" trong mockup KHÔNG dựng vì practice chưa expose prosody/stress ra client (đã chốt bỏ).*
  - **Trôi chảy** ([PracticeFluencyInsight](../../../features/practice/components/practice/PracticeFluencyInsight.tsx)): card rail GỌN bám mockup `fluencyCard` — **Mạch lạc** (badge + note) → **Trôi chảy** (badge chất lượng ngắt nghỉ) → ô **tốc độ nói gọn** (wpm + nhãn màu, KHÔNG phải gauge) → **pause pills** 3 mức + filler. Panel "Bản đầy đủ" bám mockup `openFluModal` 2 cột: TRÁI = head + transcript ngắt nghỉ + audio nghe lại + pills + legend · PHẢI = **speed gauge** đầy đủ + **coherence block** (badge + note + liên từ đã dùng + gợi ý). Dữ liệu dẫn xuất chung qua `buildFluencyView`; mảnh UI tái dùng (`FluencyHead/FluencySpeedGaugeCard/FluencyCoherenceBlock/FluencyPauseCounts/FluencyLegend/FluencyPauseTranscript`). Ngắt nghỉ **3 mức** + filler đo thật từ `pauseDurationMs`/Azure; **mạch lạc + liên từ** lấy từ **LLM `/fluency`** (đang chạy → shimmer; bài cũ → fallback heuristic detectConnectors/coherenceSummary).
  - **Từ vựng** ([PracticeVocabInsight](../../../features/practice/components/practice/PracticeVocabInsight.tsx)): card rail GỌN bám mockup `vocabCard` — badge cấp độ + **cefrBars** (phân bố CEFR) + note + **gợi ý nâng cấp DẠNG 1 DÒNG** (`word → alt1, alt2`, tối đa 2; KHÔNG legend, KHÔNG thẻ to ở rail). Panel "Bản đầy đủ" bám mockup `openVocModal` 2 cột: TRÁI = transcript **tô màu CEFR từng từ** + legend + audio · PHẢI = **"Phân bố cấp độ (CEFR)"** (cefrBars + note) + **"Gợi ý nâng cấp (paraphrase)"** (hint vàng + thẻ word/"Thử dùng:"/chips). Dữ liệu chung qua `buildVocabView`; mảnh tái dùng (`VocabCefrBars/VocabSuggestionCards/CefrLegend`). Phân bố CEFR **đếm deterministic từ nhãn per-word `/word-cefr`** (không dùng số ước lượng LLM; chưa có nhãn → shimmer).
  - **Ngữ pháp** ([PracticeGrammarInsight](../../../features/practice/components/practice/PracticeGrammarInsight.tsx)): card rail GỌN bám mockup `gramCard` — badge "Độ chính xác & đa dạng" + **độ chính xác** (câu không lỗi + bar) + **độ đa dạng câu** (đơn/phức) + **1 lỗi thường gặp gọn** + **1 gợi ý gọn**. Panel "Bản đầy đủ" bám mockup `openGramModal` 2 cột: TRÁI = **"Câu trả lời theo câu"** (vạch màu xanh/cam + nhãn đơn/phức + lỗi/không lỗi) + legend · PHẢI = thẻ **Độ chính xác/đa dạng** + **Cấu trúc nâng cao** (✅ dùng đúng / ⚠ chưa đúng / 💡 nên thử) + **Lỗi theo loại** (subtype + N lỗi) + **Gợi ý nâng cấp cấu trúc** (thẻ theo bài). Tách câu + nhãn **câu lỗi/không lỗi** + **câu đơn/phức** bằng **LLM `/grammar` (`grammarSentenceBreakdown`)**; client **đếm** sau khi LLM tách & gắn nhãn (không để AI tự đưa số). Bài cũ chưa có breakdown → fallback heuristic (`isComplexSentence` + so khớp đoạn lỗi). Mảnh tái dùng (`buildGrammarView/GrammarAccuracyBar/GrammarComplexityBars/GrammarStructuresList/GrammarErrorTypesList/GrammarSuggestionCards/GrammarSentenceList`).
- Mỗi card có nút **"⛶ Bản đầy đủ"** mở **panel rộng 2 cột RIÊNG theo tiêu chí** ([PracticeCriterionPanel](../../../features/practice/components/practice/PracticeCriterionPanels.tsx) — 4 panel: phát âm/trôi chảy/từ vựng/ngữ pháp), cột trái = transcript riêng từng tiêu chí (band/ngắt nghỉ/CEFR/danh sách câu) + audio nghe lại, cột phải = phân tích. Click chỗ sửa ở "đã sửa lỗi" → **FixPopover** bám con trỏ (badge + ❌Sai→✓Đúng), mỗi lỗi 1 popup riêng. Áp dụng cho **cả Part 1, 2, 3**.

#### 3.5.1 Drill luyện âm từng từ
- Từ card Phát âm, bấm **"Luyện"** ở một từ → mở popup drill ([PracticeWordDrill](../../../features/practice/components/practice/PracticeWordDrill.tsx)): nghe mẫu (TTS) + xem phoneme MẪU → ghi âm từ đó → server gọi Azure chấm và trả **điểm phát âm + phoneme của BẠN** (tô màu theo độ chuẩn).
- Drill dùng endpoint riêng `/word-drill`, **không trừ lượt**, không ledger, không chấm lại band cả câu. Chỉ trả điểm + phân tích âm, không đánh giá sâu hơn.

### 3.6 Coach answer (gợi ý viết lại)
- Khi cả Từ vựng + Ngữ pháp đã xong (`correctionsStatus = "completed"`), bật **3 nút mục tiêu** (`REWRITE_GOAL_BUTTONS`): **Hay hơn** (improve), **Dài & sâu hơn** (expand/depth), **Tự nhiên hơn** (vocab/grammar). Gọi `/rewrites` với `mode` tương ứng → server dùng prompt riêng của Practice (`practice.answerRewrite`, định tuyến theo 3 goal) sinh câu mẫu.
- Kết quả hiện trong [PracticeCoachAnswer](../../../features/practice/components/practice/PracticeCoachAnswer.tsx): card rail GỌN bám mockup `impCardRail` — tiêu đề **"Câu tốt hơn · {mode}"** (màu theo mode) + "bấm từ tô màu để xem vì sao" + **câu nâng cấp** với **chỗ thay đổi tô màu → bấm hiện popup vì sao** ([PracticePopover](../../../features/practice/components/practice/PracticePopover.tsx), khớp `showImpTag`) + **chips loại đã nâng** + 2 nút **"⛶ Bản đầy đủ"** / **"🎙 Thử nói lại"**. Nút "Bản đầy đủ" mở **popup 2 cột** bám mockup `openImpModal`: TRÁI = câu nâng cấp (tagged) + **legend loại** + **nghe mẫu** (TTS) + **dịch nghĩa** (translation) · PHẢI = **"Bạn đã nâng cấp gì"** (list thay đổi theo loại) + **"Cụm đáng học"** (phrasesToLearn — **chỉ hiển thị, nút Lưu đã ẩn**, chờ backend flashcard) + **"Thử nói lại theo gợi ý"**. **Mỗi mode chỉ hiện thay đổi đúng phạm vi của nó** (vocab→từ vựng/collocation; grammar→ngữ pháp; depth→ý/liên kết; improve/expand→tổng quát) — không lẫn loại khác.
- **Mỗi mode sinh một câu KHÁC nhau, không hội tụ:** prompt `practice.answerRewrite` bắt model sửa **tối thiểu, đúng 1 chiều** và **giữ nguyên các chiều khác** (vocab: giữ cấu trúc câu, chỉ đổi từ; grammar: giữ từ vựng, chỉ đổi cấu trúc; depth: giữ câu cũ, chỉ thêm lý do/ví dụ/kết luận). Rewrite chạy ở `temperature: 0.5` (cao hơn mặc định 0.2) để vocab vs grammar không ra cùng một bản. Trước đây 2 mode hay trùng kết quả vì prompt yêu cầu viết lại cả câu + temp thấp → model hội tụ về 1 bản "tốt nhất".
- Nút **"Lưu"** từ vựng đã **ẩn** (chờ backend flashcard); trước đây hiện toast *"Tính năng đang được phát triển"*. Cụm từ vẫn hiển thị để học.
- Rewrite **không trừ lượt** và **không chấm lại điểm**.

### 3.7 Lịch sử attempt
- [usePracticeAttemptHistory](../../../features/practice/hooks/usePracticeAttemptHistory.ts) giữ danh sách lượt làm **của câu hỏi hiện tại** trong phiên, mỗi lượt có audio playback + evaluation.
- Lịch sử dài hạn được persist lên NineSpeak; khi mở lại câu hỏi (resume qua `sessionId`), client rehydrate lượt cũ từ history-client (shared).
- Ở chế độ mở lại từ Lịch sử (`?sessionId=...`), ẩn 2 nút điều hướng "Câu trước"/"Câu tiếp" (chỉ xem đúng lượt đó). Lượt đã chấm xong vẫn cho ghi âm lượt mới cho cùng câu; riêng lượt `pending` (đang chờ chấm) thì **ẩn nút ghi âm** (xem §5.3).

### 3.8 Cải thiện (improvement) + delta & xu hướng
- Mỗi lượt mới so band tổng với **lượt liền trước**; nếu cao hơn → đánh cờ `isImprovement` để UI hiển thị "tiến bộ".
- Khu kết quả hiện **delta** so lượt đã chấm liền trước: chip band tổng (▲/▼/–) + delta 4 thẻ tiêu chí + **dòng động viên** theo xu hướng. Có **mini-trend "Tiến bộ band tổng"** ([PracticeBandTrend](../../../features/practice/components/practice/PracticeBandTrend.tsx)) bám mockup `trendCard` (nhãn 2 dòng + đường band qua các lượt với **chấm + nhãn giá trị từng điểm**, trục y tự co theo dải dữ liệu ±0.4 + khối phải **"Lần 1 → N"** & tổng chênh band), sticky phía trên khu chat khi ≥2 lượt. **Auto-collapse:** chỉ lượt mới nhất mở rộng, lượt cũ tự thu gọn (vẫn xem điểm+delta, bấm mở lại đầy đủ). Delta/trend tính trên lượt trong phiên (xuyên phiên phụ thuộc history rehydrate — xem TECH/report).
- Học viên thường lặp: nói → xem lỗi → đọc coach answer → thu lại để cải thiện.

---

## 4. User journey từng bước (theo Part)

### 4.1 Điểm chung (mọi Part)
1. Vào trang chủ Practice, chọn tab Part và chủ đề.
2. Chọn câu hỏi → điều hướng tới trang luyện tập tập trung của Part.
3. Nhấn ghi âm → nói → dừng (≥ 3s). **Nhãn nút ghi âm theo trạng thái**: chờ ghi = **"Bắt đầu ghi"**; đang ghi = **"Nhấn để dừng"** kèm **đồng hồ đếm giờ `M:SS`** (thời gian đã trôi, **không** giới hạn thời gian); đang chấm = nút xám + **"Đang chấm"**; đã có ≥1 lượt = **"Ghi âm lại"**.
4. Hệ thống xử lý: hiện band + 4 thẻ điểm trước, khối "đã sửa lỗi" và phân tích Từ vựng/Ngữ pháp populate song song; biểu đồ CEFR shimmer rồi điền dần.
5. Bấm thẻ điểm để mở card phân tích tiêu chí (luyện âm từng từ ở card Phát âm), nghe lại audio, dùng 3 nút mục tiêu để xem coach answer. **Hover 1 thẻ điểm** hiện tooltip **"Xem phân tích"**.
6. Thu lại để cải thiện (xem delta/xu hướng) hoặc chuyển câu kế tiếp.

### 4.2 Part 1 — Hỏi-đáp nhanh ([PracticePart1](../../../features/practice/pages/PracticePart1.tsx))
- Mỗi câu hỏi ngắn, trả lời trực tiếp 2-4 câu. Page tải nhiều câu (3 câu/trang), điều hướng câu trước/câu sau trong topic, có thể tải thêm câu (`loadMorePracticeTopicQuestions`).
- Có **Assistant Rail** gợi ý câu mẫu + từ vựng theo target band ([practice-assistant](../../../features/practice/lib/practice-assistant.ts)).

### 4.3 Part 2 — Cue card nói dài ([PracticePart2](../../../features/practice/pages/PracticePart2.tsx))
- Một cue card (1 câu/trang), người dùng nói liền mạch ~1.5-2 phút (1 đoạn ghi âm dài duy nhất).
- Tips/cue points hiển thị để dẫn dắt; vẫn chấm theo cùng pipeline (1 lần `/assessments`).

### 4.4 Part 3 — Thảo luận chiều sâu ([PracticePart3](../../../features/practice/pages/PracticePart3.tsx))
- Câu hỏi trừu tượng, đòi lập luận hai chiều. Page dùng [PracticeConsole](../../../features/practice/components/practice-part3/PracticeConsole.tsx) + SupportRail/TopicOverview/TopicSidebar quản lý hỏi-đáp luân phiên.
- 3 câu/trang giống Part 1; có khung gợi ý cấu trúc lập luận ([practice-part3 data](../../../features/practice/data/practice-part3.ts)).

> Khác biệt cốt lõi giữa các Part nằm ở **UI/nhịp luyện tập** (số câu/trang, assistant, cue card vs console) — **pipeline chấm điểm dùng chung một đường** (Azure → calibration → quickScore → vocab+grammar parallel), chỉ khác `partLabel` truyền vào prompt.

---

## 5. Business rules

### 5.1 Pipeline chấm (band lock, không rescore)
- Thứ tự bắt buộc trong `/assessments`: **Azure Speech** → **Pronunciation calibration (LLM)** → **Quick Score (LLM)**. Band phát âm do calibration quyết và **bị khoá** trước khi Quick Score chạy; Quick Score **bị cấm rescore** phát âm.
- **Vocab Review** và **Grammar Review** nhận `lockedScore` (band đã khoá) làm input và **không được rescore band** — chỉ bóc lỗi + gợi ý. Chi tiết: [scoring-pipeline-overview](../../scoring-pipeline-overview.md).
- **Calibration fail = fail-fast**: nếu LLM calibration lỗi, server **propagate error** (không fallback band Azure) → client hiện banner + cho retry. (Đây là hành vi thực tế của code; chú thích "fallback" còn sót trong source đã lỗi thời.)

### 5.2 Quota / lượt chấm (ledger riêng)
- Mỗi phiên chấm có `sessionId` (do client sinh, chính là `requestId`) → ghi sổ Firestore collection **`practice_scoring_charges`** (doc id = `sessionId`).
- **1 lượt Practice = 1 lần có điểm**: trừ ngay khi bước có điểm (`scoreDone`) thành công. `/vocab` + `/grammar` **không chạm ledger** → retry 2 tab này sau khi đã trả tiền là **miễn phí**.
- Ledger **không bao giờ block UI**: nếu trừ lượt lỗi, user vẫn nhận evaluation + lưu lịch sử; đối soát sau qua `sessionId`.
- Không có `sessionId` → bỏ qua ledger, **không trừ** (tránh trừ trùng/lệch). Không có NineSpeak token → `skipped-no-token`.

### 5.3 Paywall khi hết lượt (top-up / subscription)
- Khi `/assessments` trả **HTTP 402** với code `grading_quota_exceeded`: client **không báo lỗi đỏ** mà tạo một lượt `pending` (lưu audio + lý do), rồi mở **paywall** ([usePracticeQuotaPurchaseFlow](../../../features/practice/hooks/usePracticeQuotaPurchaseFlow.ts)).
- **Auto-paywall có điều kiện**: khi mở màn practice có lượt `pending`, chỉ **tự bung paywall** khi client thấy **hết lượt** (`hasScoringCredits=false`, suy từ `profile`: `remainingPracticeUses`/`freeRemainingPracticeUses`/premium còn hạn). Nếu **đã có lượt** (vd user vừa mua premium) → **không** nhồi paywall, để overlay "Chấm lại" hiển thị.
- **Nút "Chấm lại" = CHẤM LẠI THẬT** (`retryPendingAttempt`): re-POST `/assessments` → validate lại với NineSpeak. Có lượt → bài được chấm ngay (pending → completed); **chỉ khi vẫn bị chặn (402)** mới mở paywall. Trong lúc chấm hiện spinner "Đang chấm lại". (Trước đây nút này chỉ mở paywall, không gọi API — đã sửa.)
- **Màn pending "đang chờ chấm"** (khi mở lượt `pending`, paywall chưa/không tự bung): hiện card tiếng Việt "Bài nói đã được lưu và đang chờ chấm" + dòng hướng dẫn bấm "Chấm lại" + nút **"Chấm lại"**. **KHÔNG** in `reason` thô từ backend (tránh lòi text tiếng Anh kiểu "no practice grading quota remaining") và **ẩn nút ghi âm** ở đáy. Việc ẩn nút ghi âm **chỉ áp dụng cho lượt `pending` mở từ Lịch sử** (`sessionId`); ở luyện trực tiếp nút mic vẫn còn (đóng vai "thử lại lần nữa"), và lượt đã chấm xem từ Lịch sử cũng vẫn còn nút.
- 2 hướng nạp:
  - **Subscription** (gói premium): luồng QR qua `usePaymentOrderFlow` của Billing (barrel). Sau khi thanh toán, **tự chấm lại** lượt `pending` đó.
  - **Top-up packs** (gói lẻ lượt chấm): hiện qua Messenger checkout + màn purchase-status; user xác nhận → chấm lại thủ công.
- Giá top-up đến từ env `PRACTICE_TOPUP_3_VND` / `PRACTICE_TOPUP_6_VND` / `PRACTICE_TOPUP_10_VND` (đọc qua pricing-config của shared, inject vào page như prop `topUpPacks`).
- **Giữ phiên khi rời màn xem bảng giá**: từ paywall, CTA "Khám phá lợi ích PRO" mở trang `/pricing`. Phiên đang luyện (attempts + audio + lượt `pending`) được giữ qua điều hướng nhờ `PracticeSessionStore` mount ở gốc app; khi back lại, nội dung khôi phục nguyên vẹn và **paywall tự mở lại** cho lượt `pending` — không phải ghi âm lại. (Tầng 1: bền theo vòng đời tab; refresh thì mất. Xem TECH §6.4.)

### 5.4 Provider chấm
- `PRACTICE_EVALUATION_PROVIDER` / `NEXT_PUBLIC_PRACTICE_EVALUATION_PROVIDER` chọn provider chấm Practice. Hiện `scoringProvider` của kết quả luôn là `"azure"` (Azure Speech làm nguồn phát âm; LLM grading qua engine chung `AI_GRADING_ENGINE`).

---

## 6. Trạng thái & vòng đời

### 6.1 Recorder status (`PracticeRecorderStatus`)
`idle` → `recording` → `processing` → `scored` (hoặc `error`).

### 6.2 PracticeAttempt
- `status`: `completed` (đã chấm) | `pending` (chưa chấm do hết lượt).
- Mỗi attempt giữ `evaluation` (`PracticeEvaluationResult`) + `audioUrl` + cờ `isImprovement`.

### 6.3 Trạng thái con của một evaluation
| Trường | Giá trị | Ý nghĩa |
|---|---|---|
| `feedbackStatus` | `pending` / `completed` | Đã có band hay chưa (pending = đang chờ thanh toán) |
| `vocabReviewStatus` / `grammarReviewStatus` | `idle`/`pending`/`completed`/`failed` | Trạng thái 2 tab phân tích (retry độc lập) |
| `wordCefrStatus` | `idle`/`pending`/`completed`/`failed` | Tô màu CEFR từng từ (background) |
| `correctionsStatus` | `idle`/`pending`/`completed` | Gate cho nút rewrite (chỉ bật khi cả vocab+grammar xong) |
| `rewriteStatus.{improve,expand,vocab,grammar,depth}` | `idle`/`pending`/`completed`/`failed` | Trạng thái từng mode coach answer (3 nút mục tiêu định tuyến vào các mode này) |

### 6.4 Ledger state machine
`pending` → `charging` → `charged` | `charge-failed` | `skipped-no-token` (`voided` do admin). Hoàn thành khi `scoreDone = true`.

### 6.5 Persistence qua NineSpeak
- Mỗi lần state thay đổi (có điểm, vocab xong, grammar xong, rewrite...) → persist lên NineSpeak qua [practice-persistence](../../../features/practice/lib/practice-persistence.ts) + `savePracticeSession` (shared BFF client).
- Có **debounce 350ms** để gộp các persist gần nhau (vocab + grammar về gần đồng thời) tránh race.

---

## 7. Ràng buộc / phụ thuộc liên-module

### 7.1 Phụ thuộc `features/shared/` (lõi READ-only, import sâu được)
- **Pipeline chấm**: `runPronunciationCalibration`, `runQuickScore`, `runVocabReview`, `runGrammarReview`, `runWordCefrAnalysis` (`shared/server/evaluation-pipeline`).
- **Recorder/audio**: `useMockExamRecorder`, presigned upload, audio-upload-mode (`shared/hooks` + `shared/lib`).
- **Content store**: catalog/topic detail qua app-bff-client + history-client (`shared/lib`).
- **Report kit & types**: `MockExam*` types (`shared/types/mock-exam`), report adapter dùng chung.
- **Contract types**: tất cả type Practice sống ở `shared/types/practice-evaluation` (module chỉ re-export).
- **Tiện ích band/CEFR**: `roundBandScore`, `cefrFromOverallBand`, pricing-config, top-up-packs.
- **Quy tắc cứng**: module **không được sửa** `shared/`; thay đổi core phải làm qua refactor hạ tầng, tương thích ngược.

### 7.2 Phụ thuộc `features/billing/` (chỉ qua barrel)
- Chỉ import từ `@/features/billing`: `usePaymentOrderFlow`, `usePremiumPackages`, `SubscriptionPaymentModals`.
- **Cấm** chạm path nội bộ của Billing. Không có hook `usePaywall`.

### 7.3 Ranh giới module (ESLint enforce)
- **Cấm** import `mock-test` và `onboarding-dashboard` (peer ngang hàng, không dính nhau).
- Bề mặt cho peer chỉ qua [features/practice/index.ts](../../../features/practice/index.ts).

---

## 8. Edge cases

| Tình huống | Hành vi |
|---|---|
| Mic bị chặn / không có thiết bị (A1/A2) | Card chuyên biệt [PracticeEdgeCaseCard](../../../features/practice/components/practice/PracticeEdgeCaseCard.tsx) (code `mic_denied`/`no_device`) thay banner đỏ chung, kèm hướng dẫn cấp quyền |
| Ghi âm < 3s (B1) | Chặn trước khi gọi API (code `too_short`), card "ghi quá ngắn", quay về `error` |
| Audio không có transcript / không có lời (B2) | Server trả 422 code `no_speech` khi transcript rỗng; client nhận diện qua `isPracticeNoSpeechError` → card `no_speech` "không nghe thấy giọng nói", user thu lại (không trừ lượt) |
| Ghi quá dài (B4) | Tự dừng ghi khi chạm `maxRecordingSeconds` theo Part (P1 90s / P2 120s / P3 90s) |
| Trả lời lạc đề (C1) | Banner cảnh báo nhẹ khi `isLikelyOffTopic` (độ trùng từ với câu hỏi < 12%, heuristic cục bộ); vẫn chấm bình thường |
| Tạp âm / sai ngôn ngữ (B3/B5) | **Đã bỏ** — không có tín hiệu detect tin cậy (chốt 2026-06-23) |
| Hết lượt (402) | Tạo lượt `pending`, lưu audio, mở paywall — không báo lỗi đỏ |
| Calibration LLM fail | Fail-fast, propagate → banner + retry cả lượt (không dùng band Azure sai) |
| `/vocab` hoặc `/grammar` fail | Banner đỏ riêng tab đó + nút "Thử lại"; band tổng và tab kia không bị ảnh hưởng |
| `/word-cefr` fail | Im lặng, không ảnh hưởng luồng chính. Biểu đồ "Cấp độ từ vựng" ([PracticeVocabInsight](../../../features/practice/components/practice/PracticeVocabInsight.tsx)) đếm trực tiếp từ nhãn per-word — nếu không có nhãn nào thì hiện "Chưa phân tích được…" (KHÔNG fallback số ước lượng LLM `vocabCefrDistribution`) |
| Mất mạng / upload presigned lỗi | Fallback gửi bytes inline; nếu vẫn lỗi → banner retry |
| Spam click retry | Sequence counter chống response stale (chỉ lượt mới nhất thắng) |
| Double-fire `/assessments` cùng tick | Ref-based mutex chặn |
| Đóng tab / reload khi đang chấm | `beforeunload` cảnh báo "đang chấm điểm, rời trang sẽ mất tiến độ" |
| Trừ lượt (ledger) fail | Không block: user vẫn nhận điểm + lưu history; đối soát qua `sessionId` |
| Resume câu hỏi cũ (`sessionId` trên URL) | Rehydrate lượt từ NineSpeak history |

---

## 9. Out of scope

- **Mock Test / AI Simulation** — thuộc module `mock-test`, chấm cả phiên 3 part (Practice không đụng).
- **Thanh toán / quản lý gói** — thuộc `billing`; Practice chỉ tiêu thụ qua barrel.
- **Dashboard / History tổng hợp / Settings / Onboarding** — thuộc `onboarding-dashboard`; `/history` và `/hieu-suat` gộp dữ liệu Practice + Mock Test qua history-client của shared.
- **Định nghĩa prompt chung & speech SDK** — sống ở `shared/server` (Practice chỉ gọi); chỉ prompt `practice.answerRewrite` là riêng module.
- **Sửa core pipeline / contract types** — phải qua refactor hạ tầng `shared/`, không làm trong module.
