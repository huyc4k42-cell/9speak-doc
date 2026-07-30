# MockTest ↔ Practice Connection Flow — Handoff Document

**Trạng thái:** Decisions finalized (grill-me + conflict resolution done) — sẵn sàng viết spec  
**Session hoàn thành:** Discovery → Brainstorming → Design Critique → Conflict Resolution → Grill-me  
**Bước tiếp theo:** `/spec` — viết PRD/spec chính thức dựa trên handoff này

---

## 1. Bối cảnh dự án

### Kiến trúc
- **9Speak**: Next.js monorepo, luyện IELTS Speaking
- **MockTest module** (`features/mock-test/`): Thi thử 3 Part, chấm điểm phiên, báo cáo band + 4 tiêu chí
- **Practice module** (`features/practice/`): Luyện từng câu, chấm ngay, coach answer, drill phát âm, delta/trend
- **Shared module** (`features/shared/`): Pipeline scoring, recorder hooks, report kit, contract types, NineSpeak client, content store, history-client
- **Performance page** `/hieu-suat`: history-client từ shared, hiện tại passive display
- **ESLint enforced**: MockTest và Practice KHÔNG được import lẫn nhau — kết nối CHỈ qua URL navigation

### Tech constraints
- URL params pattern: `?source=mock-report&criterion=vocabulary&sessionId=xxx`
- Clear params: `router.replace` sau first render — không tạo history entry mới
- Practice catalog BFF `/api/app/speaking/practice/catalog`: hiện CHƯA có `criterionTags` trên topics → deep-link criterion-filtered = V2

### Data context (Mixpanel)
- 83.8% users không cross modules trong 7 ngày
- Average time gap Practice → MockTest: ~14.5h
- Không có positive correlation giữa Practice sessions và MockTest score (selection bias)
- Practice sidebar clicks 3× MockTest (30.3% vs 10.5%)

---

## 2. Tất cả quyết định đã chốt

### 2.1 MockTest Report — Zone "Bước tiếp theo"

#### Vị trí CTA
CTA đặt sau accordion 4 tiêu chí (cuối Tổng quan), trước Part selector. Trigger: report phải hoàn chỉnh — band + vocab/grammar đầy đủ (không hiển thị khi report partial).

#### Criterion selection
Chọn criterion yếu nhất theo band. Khi tie, fallback cố định: **Vocabulary → Grammar → Fluency → Pronunciation** (thứ tự này dùng nhất quán ở mọi nơi: MockTest report, Performance page, secondary CTA).

#### CTA label
`"Luyện [Criterion] tại Practice →"` + sub-text `"Band [X.X] · [N] lượt còn lại"`  
Premium user: sub-text `"Band [X.X] · Unlimited"`

#### CTA khi 0 lượt
CTA chuyển thành upsell in-context. Label: `"Nạp lượt để luyện [Criterion] →"`. Click mở billing flow ngay tại MockTest report — không navigate sang Practice rồi bị chặn.

#### Số lượng CTA (V1)
Luôn hiển thị đúng **1 CTA duy nhất** — không dual-CTA ngay cả khi Pronunciation yếu nhất. Dual-CTA Pronunciation deferred to V2 khi catalog có `criterionTags`.

#### Deep-link destination (V1)
Practice index tổng: `/practice?source=mock-report&criterion=[criterion]&sessionId=[id]`  
Criterion-filtered view → V2 khi BFF có `criterionTags`.

#### Loop Language
Tất cả cross-module touchpoints dùng shared icon (↻ hoặc label `"Bước tiếp theo →"`) nhất quán. Không dùng màu riêng — amber giữ nguyên cho warning/caveat.

---

### 2.2 Before/After Score Reveal

#### Điều kiện hiển thị
Chỉ hiển thị khi user đang xem session **mới nhất** (`isLatestSession` check). Ẩn hoàn toàn khi browse history.

#### Trigger render
Card + CTA cùng zone, render đồng thời khi user scroll đến cuối Tổng quan — không timer, không event tracking.

#### Nội dung card
Delta thuần túy — không claim causation:  
`"Lần trước: [X.X] → Lần này: [X.X] (+/−[Y])"`

Khi score giảm, thêm framing:  
`"Band có thể dao động giữa các lần thi — tiếp tục luyện để ổn định kết quả."`

---

### 2.3 Mini-drill Differentiation

Nút `"Luyện ›"` trong tab Phát âm (mini-drill, miễn phí) giữ nguyên, thêm label phụ `"(Miễn phí · 1 từ)"`.  
Không cạnh tranh với Practice CTA vì hai thứ nằm ở hai zones khác nhau (in-tab vs. cuối Tổng quan).

---

### 2.4 Practice — Context Banner

- Hiển thị khi Practice được mở từ MockTest report (URL params present)
- Content: `"Đang luyện theo gợi ý từ phiên thi [ngày] · Xem lại báo cáo ›"`
- Sau khi banner render lần đầu: URL params clear ngay bằng `router.replace`
- Banner biến mất khi user navigate sang câu hỏi khác
- Không re-appear khi dùng browser back
- Shared link không leak sessionId của người khác

---

### 2.5 Performance Page `/hieu-suat` — Action Card

Hiển thị đúng 1 card theo priority **A > C > D > B**:

| State | Điều kiện | Copy |
|-------|-----------|------|
| **A** | Chưa có MockTest nào | `"Thi thử để biết band thật của bạn →"` |
| **C** | MockTest từ 7–30 ngày trước | `"Đã [X] ngày kể từ lần thi — thử lại để cập nhật band →"` |
| **D** | MockTest từ >30 ngày trước | `"Đã lâu không kiểm tra — thi thử để cập nhật band →"` |
| **B** | MockTest trong 7 ngày qua (cool-down) | `"Tiếp tục luyện [criterion yếu nhất] →"` |
| Default | Không match state nào | Không hiển thị card |

State B KHÔNG hiển thị "thi lại" — chỉ "tiếp tục luyện". Criterion yếu nhất dùng cùng fallback order: Vocab > Grammar > Fluency > Pronunciation.

State C: không có điều kiện session count — **time-based only**.

---

## 3. Decisions đã drop

| Decision | Lý do |
|----------|-------|
| **BR-04** (≥5 sessions nudge) | Không có data backing, selection bias trong correlation data. Không queue V2. |
| **Pre-exam warm-up** | Timing fundamentally sai (peak anxiety), complexity cao, benefit không chắc chắn. Không queue V2. |
| **Before/After causation claim** | Không có causal backing. Card chỉ hiển thị delta thuần túy. |
| **Session count trong State C** | Dropped — time-based only là đủ và có data backing. |

---

## 4. Deferred to V2

| Item | Blocker |
|------|---------|
| Deep-link criterion-filtered view | BFF cần có `criterionTags` trên Practice catalog topics |
| Dual-CTA cho Pronunciation | Practice index cần criterion filter để 2 CTAs mới có giá trị |
| `attempt_number` & `score_delta` instrumentation | Cần instrument `mock_scoring_completed` event cho future data-backed thresholds |

---

## 5. Business Rules Summary (BR-01 → BR-07)

| BR | Touchpoint | Mô tả |
|----|-----------|--------|
| BR-01 | MockTest Report → Practice CTA | Sau report hoàn chỉnh, hiển thị CTA cho criterion yếu nhất |
| BR-02 | Practice CTA content | Label + sub-text lượt còn lại / Unlimited / Upsell khi 0 lượt |
| BR-03 | CTA deep-link | V1: Practice index; V2: criterion-filtered |
| BR-04 | ~~Session count nudge~~ | **DROPPED** |
| BR-05 | Quota display | Hiển thị lượt còn lại trong CTA |
| BR-06 | Performance page action card | 4 states A/C/D/B theo time-based logic |
| BR-07 | Context banner trong Practice | Hiển thị khi đến từ MockTest report via URL params |

---

## 6. Conflict Resolution

### Conflict 1 — Mini-drill vs. Practice CTA
**Vấn đề:** Cả hai đều là "luyện" — user không rõ khác gì nhau.  
**Giải pháp:** Mini-drill thêm label `"(Miễn phí · 1 từ)"`. Practice CTA ở zone riêng cuối Tổng quan. Tách biệt vật lý.

### Conflict 2 — Performance page cool-down vs. MockTest report CTA
**Vấn đề:** Cool-down 14 ngày quá dài — nếu user thi lại sau 10 ngày, Performance page vẫn nudge "thi lại" gây mâu thuẫn.  
**Giải pháp:** Cool-down = **7 ngày** (align với MockTest report State B logic). State B trên Performance page chỉ hiển thị "tiếp tục luyện", không nudge "thi lại".

### Conflict 3 — Pre-exam warm-up
**Vấn đề:** Màn intro MockTest là peak anxiety — suggest "luyện trước" lúc này sai timing.  
**Giải pháp:** **DROPPED** hoàn toàn. Alternative: text tĩnh "Hít thở sâu, tự tin bắt đầu." trong intro nếu cần.

---

## 7. Files liên quan

| File | Mục đích |
|------|---------|
| [`docs/modules/mock-test/PRD.md`](../modules/mock-test/PRD.md) | MockTest module scope, scoring, paywall, report structure |
| [`docs/modules/mock-test/screens/07-report-tong-quan.md`](../modules/mock-test/screens/07-report-tong-quan.md) | Tổng quan section spec |
| [`docs/modules/mock-test/screens/08-report-phan-hoi-chi-tiet.md`](../modules/mock-test/screens/08-report-phan-hoi-chi-tiet.md) | Part selector, criterion tabs, mini-drill spec |
| [`docs/modules/practice/PRD.md`](../modules/practice/PRD.md) | Practice module scope, drill loop, paywall |
| [`docs/modules/performance/PRD.md`](../modules/performance/PRD.md) | Performance page PRD |
| [`docs/new-requirements/Markdown/PRACTICE_OPTIMIZE_FULL_SPEC.md`](PRACTICE_OPTIMIZE_FULL_SPEC.md) | Practice optimize spec (layout, criterion cards, coach answer) |

---

## 8. Bước tiếp theo

**Cần làm ngay:** Viết spec chính thức — invoke `/anthropic-skills:discovery` → `/spec` với handoff file này làm input.

Scope của spec:
1. MockTest Report — Zone "Bước tiếp theo" (Before/After card + Practice CTA + upsell state)
2. Practice — Context Banner lifecycle
3. Performance Page — Action Card 4 states
4. Mini-drill differentiation label

**V2 backlog** (không thuộc scope spec này):
- Practice catalog BFF: thêm `criterionTags`
- Criterion-filtered deep-link
- Dual-CTA Pronunciation
- Mixpanel instrumentation: `attempt_number`, `score_delta`
