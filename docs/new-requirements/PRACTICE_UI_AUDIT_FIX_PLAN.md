# Practice — Audit UI & Kế hoạch chỉnh cho khớp mockup

> Nguồn chuẩn: [practice-flow-optimized.html](./practice-flow-optimized.html). Tài liệu này rà soát
> hiện trạng UI trang kết quả luyện tập, chỉ ra **bug + lệch so mockup** (kèm root cause, file:line)
> và **kế hoạch sửa theo phase**. Tất cả là lỗi **UI/trình bày**, không phải logic chấm điểm.

---

## 0. Tóm tắt điều hành

| # | Vấn đề | Mức | Root cause | Nơi sửa |
|---|--------|-----|-----------|---------|
| B1 | Bấm từ highlight trong "Bản đầy đủ" KHÔNG hiện popup (bị đè) | **P0** | `AnnotationPopup` z-50 < `PracticeCriterionWidePanel` z-[60] (panel có `backdrop-blur` tạo stacking context đè lên popup) | WidePanel / AnnotationPopup |
| B2 | "Đã sửa lỗi": bấm từ nào cũng mở **chung 1 popup** | **P0** | Mỗi câu chỉ dựng **1 `representative`** annotation; mọi span đều gọi `onSelectAnnotation(representative)` | `PracticeCorrectedAnswer.tsx:130,190` |
| B3 | "Đã sửa lỗi": **thừa chữ "s"** (và ký tự lạ) ở cuối từ | **P0** | `replaceFirstCaseInsensitive` thay **substring** không theo word-boundary → để sót đuôi từ | `practice-corrected-diff.ts:13-18` |
| B4 | Popup giải thích lỗi **không giống** `fixpop` mockup | **P1** | Đang dùng `AnnotationPopup` (modal giữa màn, style mock-test) thay vì thẻ nhỏ bám con trỏ | shared `AnnotationPopup` |
| B5 | Các **box rail "AI hỗ trợ"** + **popup Bản đầy đủ** layout chưa khớp mockup (đặc biệt bố cục 2 cột per-tiêu-chí) | **P1** | WidePanel dùng **1 shell generic** (transcript trái + insight phải) thay vì layout riêng từng tiêu chí | WidePanel + 4 insight |
| B6 | Chuẩn hoá **z-index/stacking** toàn bộ overlay practice | **P1** | Mỗi overlay đặt z rời rạc (z-50/z-[60]/z-[70]) không theo 1 thang | nhiều file |

---

## 1. Kiến trúc hiện tại vs mockup (tổng quan)

**Mockup** dùng các overlay rời theo tầng rõ ràng:
- Scrim phân tích `z-40` → Modal phân tích (pron/flu/vocab/gram) `z-50` → Drill scrim `z-60` → Drill modal `z-70` → Toast `z-80`.
- `fixpop` (giải thích lỗi "đã sửa" + tag cải thiện): thẻ nhỏ **bám con trỏ**, `w-260px`, `z-50`, đóng khi click ngoài.
- Mỗi tiêu chí có **modal riêng** (`openPronModal/openFluModal/openVocModal/openGramModal`) với **bố cục 2 cột bespoke** (xem §3).

**Hiện tại** dùng:
- 1 component generic `PracticeCriterionWidePanel` (overlay `z-[60]`, `backdrop-blur-sm`) cho **cả 4 tiêu chí**: cột trái = `TranscriptViewer` (annotation chung), cột phải = insight.
- `AnnotationPopup` shared (mock-test) `z-50`: modal **giữa màn**, không phải thẻ bám con trỏ.
- `PracticeWordDrill` overlay `z-[70]`.
- "Đã sửa lỗi" = `PracticeCorrectedAnswer` (inline del đỏ + ins xanh) → click mở `AnnotationPopup` ở panel cha.

→ Hệ quả lệch: thang z không khớp (panel `z-[60]` > popup `z-50` gây **B1**); popup style khác mockup (**B4**); layout 2 cột generic không khớp bố cục riêng từng tiêu chí (**B5**); với phát âm còn **trùng transcript** (cột trái `TranscriptViewer` annotation + cột phải insight đã có sẵn band-transcript riêng).

---

## 2. Bug chi tiết + root cause

### B1 — Popup bị đè trong "Bản đầy đủ" (P0)
- [PracticeCriterionWidePanel.tsx:85](../../features/practice/components/practice/PracticeCriterionWidePanel.tsx#L85): overlay `fixed inset-0 z-[60] ... backdrop-blur-sm` → **tạo stacking context** ở mức 60.
- [MockExamReportSections.tsx:817](../../features/shared/components/reports/MockExamReportSections.tsx#L817): `AnnotationPopup` là `fixed inset-0 z-50`, render **sibling** ngay sau panel ([WidePanel.tsx:139](../../features/practice/components/practice/PracticeCriterionWidePanel.tsx#L139)).
- 60 > 50 → popup nằm **dưới** panel → bấm từ trong transcript của Bản đầy đủ thấy như "không hiện popup".
- **Fix**: nâng z của popup khi ở trong WidePanel lên **trên 60** (vd `z-[75]`), hoặc render qua **portal** ra `body` với z cao. Vì `AnnotationPopup` là **shared (mock-test dùng)**, KHÔNG sửa cứng z-50 của nó → thêm prop `zClassName`/`overlayClassName` (default giữ z-50) hoặc bọc portal riêng cho practice.

### B2 — "Đã sửa lỗi": bấm từ nào cũng mở chung 1 popup (P0)
- [PracticeCorrectedAnswer.tsx:130](../../features/practice/components/practice/PracticeCorrectedAnswer.tsx#L130): `representative = sentenceLevel ?? grammarAnns[0] ?? lexicalAnns[0]` — **1 annotation cho cả câu**.
- [:190](../../features/practice/components/practice/PracticeCorrectedAnswer.tsx#L190): mọi span "change" đều `onClick={() => onSelectAnnotation(representative)}`.
- → Câu có nhiều lỗi nhưng click từ nào cũng ra **cùng một** giải thích.
- **Fix**: map **từng span thay đổi → đúng annotation của nó**. Khi dựng `corrected` bằng cách áp từng correction, gắn `annotation.id` cho vùng ký tự tương ứng, rồi ở bước `groupDiff` giữ liên kết span→annotationId (vd diff theo từng correction, hoặc match `span.added/removed` về annotation theo `shortText/replacement`). Mỗi span mở popup riêng.

### B3 — "Đã sửa lỗi": thừa chữ "s"/ký tự lạ (P0)
- [practice-corrected-diff.ts:13-18](../../features/practice/lib/practice-corrected-diff.ts#L13-L18): `replaceFirstCaseInsensitive` dùng `indexOf` (**substring**, không word-boundary).
- Ví dụ lỗi: `shortText="advert"`, `replacement="advertisement"`, câu chứa từ số nhiều `"adverts"` → thay 6 ký tự "advert" thành "advertisement", **để sót "s"** → `"advertisements"`. Tương tự `shortText="go"` có thể khớp trong `"goes"/"good"`. → đây là nguồn "thừa s ở cuối từ" + ghép từ sai.
- **Fix**: thay bằng **thay theo ranh giới từ** (regex `\b` + `\bword\b` case-insensitive, có xử lý dấu câu), hoặc khớp theo **token** thay vì substring thô. Cần giữ nguyên dấu câu/han chế khớp nhầm trong từ khác.

### B4 — Popup giải thích chưa giống `fixpop` mockup (P1)
- Mockup `fixpop` ([html:124](./practice-flow-optimized.html), `showFix`/`showImpTag`): thẻ nhỏ `w-260px`, **bám con trỏ** `(clientX+10, clientY+12)`, nội dung: **badge loại** (Từ vựng/Ngữ pháp…) + dòng `❌ Sai: <gạch> → ✓ Đúng: <xanh>` + giải thích ngắn; đóng khi click ngoài.
- Hiện tại: `AnnotationPopup` modal **giữa màn** + scrim — khác hẳn.
- **Fix**: dựng **popover practice riêng** (thẻ nhỏ bám con trỏ) cho "đã sửa lỗi" + tag cải thiện, đúng `fixpop`. Không tái dùng AnnotationPopup mock-test cho khu vực này.

### B5 — Box rail + popup tiêu chí chưa khớp bố cục mockup (P1)
- Rail card (`PracticeCriterionCard`) đã gần khớp (header + điểm + insight + nút "Bản đầy đủ") nhưng cần đối chiếu từng block với `pronCard/fluencyCard/vocabCard/gramCard` mockup (thứ tự, nhãn, màu).
- WidePanel generic chưa khớp **bố cục 2 cột bespoke** từng tiêu chí (xem §3). Với **phát âm** còn **trùng transcript** (cột trái annotation + insight phải đã có band-transcript).
- **Fix**: cho mỗi tiêu chí một **layout riêng** trong WidePanel (hoặc tách 4 panel), đúng cột trái/phải + các section mockup; bỏ phần trùng.

### B6 — Chuẩn hoá z-index (P1)
- Hiện: `AnnotationPopup` 50, WidePanel 60, Drill 70, một số modal khác 50. Không nhất quán, dễ tái phát B1.
- **Fix**: định 1 thang z dùng chung practice (đề xuất §5).

---

## 3. Đối chiếu chi tiết theo khu vực (chuẩn = mockup)

### 3.1 Transcript highlight + click → drill
- Mockup: 4 mức highlight phát âm (ok/minor gạch chấm/light vàng phần/severe đỏ+IPA); click → `openPronDrill`. Transcript "độ chuẩn" theo band trong modal phát âm; transcript CEFR trong modal từ vựng; transcript ngắt nghỉ trong modal trôi chảy.
- Hiện: `PracticePronunciationInsight` đã có band-transcript + click → `PracticeWordDrill`; nhưng trong WidePanel cột trái lại là `TranscriptViewer` annotation (khác) → **trùng/khó hiểu**.
- Cần: thống nhất transcript hiển thị theo đúng từng tiêu chí như mockup, bỏ trùng.

### 3.2 "Câu trả lời — đã sửa lỗi"
- Mockup: inline `fix-del` (gạch đỏ) + `fix-ins` (xanh, click) → `fixpop` bám con trỏ (badge + ❌/✓ + giải thích).
- Hiện: inline del/ins đúng hướng; nhưng (a) **B2** click chung 1 popup, (b) **B3** thừa "s", (c) **B4** popup sai kiểu.

### 3.3 Rail "AI hỗ trợ" (card gọn)
- Mockup: mỗi tiêu chí 1 card gọn — header chấm màu + "Phân tích · <tên>" + điểm + 3-4 block tóm tắt + nút "⛶ Bản đầy đủ" + nút ✕.
- Hiện: `PracticeCriterionCard` có đủ khung; cần soát từng block tóm tắt cho khớp thứ tự/nhãn/màu mockup.

### 3.4 Popup "Bản đầy đủ" — bố cục 2 cột bespoke
| Tiêu chí | Cột trái (mockup) | Cột phải (mockup) | Hiện tại |
|---|---|---|---|
| Phát âm | Transcript theo độ chuẩn + audio player + legend | Lỗi (nặng→nhẹ) · Nhóm theo âm · Trọng âm & Ngữ điệu | Trái=annotation transcript; phải=insight (đã có band-transcript→trùng); thiếu audio player; thiếu Trọng âm/Ngữ điệu (đã chốt **không làm** prosody) |
| Trôi chảy | Transcript ngắt nghỉ + audio + legend/counts | Gauge tốc độ · Mạch lạc (badge+note+linkers+gợi ý) | 1 cột insight; thiếu audio player; bố cục chưa 2 cột |
| Từ vựng | Transcript tô màu CEFR + legend | Phân bố CEFR · Gợi ý nâng cấp | 1 cột insight; thiếu transcript CEFR riêng |
| Ngữ pháp | Danh sách câu (đơn/phức, lỗi/sạch) + legend | Độ chính xác · Cấu trúc nâng cao · Lỗi theo loại · Gợi ý | Trái=annotation transcript; phải=insight; danh sách câu nằm trong insight |

### 3.5 Drill từng từ
- Mockup `openPronDrill`: word + play + IPA + nghĩa; hàng "Mẫu"/"Bạn" phoneme tô màu; hộp hướng dẫn (target + guide + xem thêm); nút "Ghi âm ngay".
- Hiện: `PracticeWordDrill` — cần đối chiếu chi tiết bố cục/nhãn cho khớp (P2).

---

## 4. Kế hoạch sửa theo phase

> Nguyên tắc: **không đụng logic chấm/LLM**; chỉ UI. Hạn chế sửa shared (`AnnotationPopup`, `MockExamReportSections`) — ưu tiên thêm prop/portal hoặc dựng component practice riêng để không ảnh hưởng mock-test.

### Phase 0 — Sửa 3 bug chặn (P0) — *ưu tiên cao nhất, rủi ro thấp*
1. **B1**: cho `AnnotationPopup` nhận prop `overlayZClassName` (default `z-50`); WidePanel truyền `z-[75]`. (Hoặc portal ra body.) — không đổi hành vi mock-test.
2. **B3**: viết `replaceWordCaseInsensitive` thay theo ranh giới từ; thay `replaceFirstCaseInsensitive` trong `buildCorrected`. Thêm test nhỏ vài case (số nhiều, từ nằm trong từ khác).
3. **B2**: map span→annotation riêng trong `PracticeCorrectedAnswer` để mỗi vùng sửa mở đúng popup của nó.

### Phase 1 — Popup giải thích + bố cục panel (P1)
4. **B4**: dựng `PracticeFixPopover` (thẻ nhỏ bám con trỏ, badge + ❌/✓ + giải thích) cho "đã sửa lỗi" (và sau này tag cải thiện), thay `AnnotationPopup` ở khu vực này.
5. **B5**: cho mỗi tiêu chí một **layout 2 cột bespoke** trong WidePanel theo §3.4 (bỏ transcript trùng ở phát âm; thêm transcript CEFR cho từ vựng; thêm transcript ngắt nghỉ cho trôi chảy; thêm audio player nếu có `audioUrl`).
6. **B6**: chốt thang z (xem §5) và áp đồng bộ.

### Phase 2 — Hoàn thiện chi tiết (P2)
7. Soát từng card rail + drill modal cho khớp nhãn/màu/thứ tự mockup.
8. Audio player (nghe lại đoạn) trong panel nếu `audioUrl` có sẵn.

---

## 5. Thang z-index đề xuất (practice)

| Lớp | z | Ghi chú |
|---|---|---|
| Scrim panel Bản đầy đủ | `z-[60]` | giữ |
| Panel Bản đầy đủ | `z-[60]` | giữ |
| **Popup annotation / FixPopover (trong panel)** | **`z-[75]`** | > 60 để không bị đè (sửa B1) |
| Drill scrim/modal | `z-[80]/z-[85]` | drill luôn trên cùng panel + popup |
| Toast | `z-[90]` | trên cùng |

*(Nếu giữ Drill `z-[70]` thì popup phải < 70 và > 60 → `z-[65]`; nhưng để drill luôn trên popup, nên nâng drill lên 80+.)*

---

## 6. Câu hỏi cần chốt trước khi code
1. **B5 — cách dựng panel**: giữ **1 `PracticeCriterionWidePanel`** rồi rẽ nhánh layout theo `criterionKey`, hay tách **4 component panel riêng** (giống 4 `openXxxModal` mockup)? (Đề xuất: giữ 1 component, rẽ nhánh — ít trùng, dễ chia sẻ khung.)
2. **Audio player** trong panel: có cần làm ngay không, hay để Phase 2? (Cần `audioUrl` — đã truyền sẵn vào panel.)
3. **Prosody (Trọng âm & Ngữ điệu)** ở panel phát âm: giữ **không làm** như đã chốt? (mockup có, nhưng practice chưa expose dữ liệu).
4. Có muốn tôi làm **Phase 0 (3 bug P0) trước rồi review**, sau đó mới sang Phase 1 không? (Đề xuất: có — P0 rủi ro thấp, thấy kết quả ngay.)

---

## 7. ĐÃ THỰC HIỆN (chốt: làm hết một mạch · 4 panel riêng · audio Phase 1 · bỏ prosody)

- **B3** — `practice-corrected-diff.ts`: thêm `locateWordCaseInsensitive` + `replaceFirstWordCaseInsensitive` (thay theo RANH GIỚI TỪ) → hết "thừa s".
- **B2** — `PracticeCorrectedAnswer.tsx`: định vị từng correction theo word-boundary, mỗi vùng sửa gắn **đúng annotation của nó** (sentence-level vẫn 1 popup; fragment → mỗi lỗi 1 popup).
- **B4** — `PracticeFixPopover.tsx` (mới): thẻ nhỏ **bám con trỏ** (badge loại + ❌Sai→✓Đúng + giải thích), đóng khi click ngoài/Esc, clamp viewport, `z-[95]`. Corrected answer self-contained (bỏ phụ thuộc AnnotationPopup shared).
- **B5** — `PracticeCriterionPanels.tsx` (mới): **4 panel riêng** (pron/flu/vocab/grammar) bố cục 2 cột. Cột trái transcript riêng từng tiêu chí: band-transcript (phát âm) / pause-transcript (trôi chảy) / **CEFR-transcript** (từ vựng) / danh sách câu (ngữ pháp). Cột phải = insight (ẩn transcript trùng qua prop `hideBandTranscript`/`hidePauseTranscript`/`hideSentenceList`). Xoá `PracticeCriterionWidePanel.tsx`.
- **Audio player**: `<audio controls>` nghe lại bài nói trong mỗi panel (khi có `audioUrl`).
- **B1+B6** — panel KHÔNG còn dùng `AnnotationPopup` shared → hết cảnh "popup bị đè". Thang z: panel `z-[60]` · drill `z-[70]` (trên panel) · FixPopover `z-[95]`.
- **Prosody (Trọng âm & Ngữ điệu)**: giữ KHÔNG làm (practice chưa expose dữ liệu).

Verify: `npx tsc --noEmit` ✓ · `eslint` ✓ · `npm run build` status=0 ✓.

> Lưu ý còn lại (Phase 2 nếu cần): soát chi tiết nhãn/màu từng card rail + drill modal cho khít mockup; audio player tuỳ biến (hiện dùng native control).
