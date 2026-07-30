# PRD — MockTest ↔ Practice Connection Flow

| | |
|---|---|
| **Product** | 9Speak — IELTS Speaking |
| **Feature** | Kết nối MockTest → Practice (Loop closing) |
| **Scope** | MVP — Wave 1 |
| **Ngày** | 2026-07-25 |
| **Trạng thái** | Sẵn sàng triển khai |
| **Tài liệu liên quan** | [Spec đầy đủ](./MOCKTEST_PRACTICE_CONNECTION_SPEC.md) · [Handoff & decisions](./MOCKTEST_PRACTICE_CONNECTION_HANDOFF.md) |

---

## Mục lục

1. [Goal](#1-goal)
2. [Context](#2-context)
3. [Metrics](#3-metrics)
4. [Requirements](#4-requirements)
5. [User Types](#5-user-types)
6. [Analytics](#6-analytics)
7. [Timelines](#7-timelines)
8. [Dependencies](#8-dependencies)
9. [Appendix](#9-appendix)

---

## 1. Goal

### 1.1 Business / Product Goal

Đóng vòng lặp **Diagnose → Practice → Validate** — hiện tại 83.8% users không cross modules trong 7 ngày. Feature này tạo touchpoints dẫn user từ MockTest report sang luyện đúng điểm yếu, và từ Practice nhớ quay lại thi kiểm tra tiến bộ.

### 1.2 User Benefits

- **Sau MockTest:** Biết ngay phải luyện criterion nào — không phải tự đoán.
- **Khi vào Practice từ MockTest:** Hiểu context tại sao đang luyện criterion này.
- **Trên Performance page:** Nhận gợi ý action phù hợp theo trạng thái thực tế.

---

## 2. Context

### 2.1 Usage Data / UXR Insights

| Insight | Nguồn |
|---------|-------|
| 83.8% users không cross modules trong 7 ngày | Mixpanel — 7 ngày gần nhất |
| Average gap Practice → MockTest: ~14.5h | Mixpanel |
| Practice sidebar clicks 3× MockTest (30.3% vs 10.5%) | Mixpanel |
| Không có positive correlation giữa Practice sessions và MockTest score | Mixpanel — selection bias |

### 2.2 Recommendation

- Xây **touchpoint chủ động** ngay tại MockTest report dẫn sang Practice đúng criterion yếu.
- Tránh claim causation (không có causal backing) — chỉ hiển thị delta và gợi ý.
- Tạo **loop language** nhất quán (↻ / "Bước tiếp theo →") xuyên suốt các touchpoints.

---

## 3. Metrics

### 3.1 Measurement Mechanism

- **Sample:** Tất cả users thi MockTest và xem report hoàn chỉnh.
- **Method:** Event tracking qua Mixpanel. Đo trước/sau launch, không A/B test ở Wave 1.

### 3.2 North Star Metric

Cross-module rate: % users có cả MockTest lẫn Practice activity trong 7 ngày.

> **Baseline hiện tại:** 16.2% · **Target:** ≥25% sau 30 ngày

### 3.3 Success Metrics

| Metric | Baseline | Target | Thời gian đo |
|--------|----------|--------|--------------|
| CTA click-through rate (report → Practice) | ~0% | ≥20% | 2 tuần sau launch |
| Cross-module rate (7 ngày) | 16.2% | ≥25% | 30 ngày sau launch |
| Performance page Action Card click rate | TBD | ≥30% users vào `/hieu-suat` | 30 ngày sau launch |

### 3.4 Guardrail Metrics

Không được ảnh hưởng tiêu cực:
- MockTest completion rate (user không bỏ dở report vì bị distract bởi CTA mới)
- Practice session start rate (banner không gây friction)

---

## 4. Requirements

> **Constraint kiến trúc bắt buộc:** MockTest và Practice bị ESLint enforce không được import lẫn nhau. Cross-module = URL navigation only: `?source=mock-report&criterion=[criterion]&sessionId=[id]`

---

### 4.1 Functional Requirements

---

#### Feature A — MockTest Report: Practice CTA

**Mô tả:** Zone "Bước tiếp theo" đặt sau accordion 4 tiêu chí (cuối section Tổng quan), trước Part selector. Hiển thị 1 CTA dẫn sang Practice với criterion yếu nhất.

**Điều kiện hiển thị:** Report hoàn chỉnh — band + vocab/grammar đủ 4 tiêu chí. Ẩn khi report partial.

**Criterion yếu nhất:** Band thấp nhất. Tie-breaker cố định: `Vocabulary → Grammar → Fluency → Pronunciation`.

| Case | Scenario | Expected Behavior |
|------|----------|------------------|
| ✅ Happy | Report hoàn chỉnh, user còn lượt | CTA hiển thị: `"Luyện [Criterion] tại Practice →"` + sub-text `"Band [X.X] · [N] lượt còn lại"` |
| ✅ Happy | Premium user | Sub-text: `"Band [X.X] · Unlimited"` |
| ✅ Happy | User click CTA | Navigate `/practice?source=mock-report&criterion=[criterion]&sessionId=[id]` |
| ⚠️ Corner | 0 lượt còn lại | Label đổi thành `"Nạp lượt để luyện [Criterion] →"`, click → billing flow in-context |
| ⚠️ Corner | Report partial / đang chấm | Zone ẩn hoàn toàn |
| ⚠️ Corner | Tie giữa 2+ criterion | Dùng fallback order: Vocabulary → Grammar → Fluency → Pronunciation |

> **Wave 1 scope:** Không có Before/After delta card (deferred sang Wave 2). Luôn 1 CTA duy nhất.

**Acceptance Criteria:**

**AC-A01: Zone ẩn khi report chưa hoàn chỉnh**
- **Given:** Report đang chấm hoặc thiếu tiêu chí
- **When:** User mở MockTest report
- **Then:** Zone "Bước tiếp theo" không render

**AC-A02: CTA hiển thị đúng criterion và lượt**
- **Given:** Report hoàn chỉnh, `remainingPracticeUses > 0`
- **When:** User scroll đến cuối Tổng quan
- **Then:** Label `"Luyện [tên criterion] tại Practice →"` với criterion đúng AND sub-text `"Band [X.X] · [N] lượt còn lại"` (hoặc `"Unlimited"` nếu premium)

**AC-A03: Click CTA navigate đúng URL**
- **Given:** User còn lượt, click CTA
- **When:** Click
- **Then:** Navigate `/practice?source=mock-report&criterion=[criterion]&sessionId=[id]` với giá trị chính xác

**AC-A04: Upsell khi 0 lượt**
- **Given:** `remainingPracticeUses = 0`
- **When:** Zone render
- **Then:** Label `"Nạp lượt để luyện [Criterion] →"` AND click mở billing flow in-context (không navigate sang Practice)

**AC-A05: Tie-breaker nhất quán**
- **Given:** 2+ criterion cùng band thấp nhất
- **When:** Zone render
- **Then:** CTA hiển thị criterion đầu tiên trong fallback order (Vocabulary > Grammar > Fluency > Pronunciation)

---

#### Feature B — Practice: Context Banner

**Mô tả:** Banner xuất hiện ở đầu Practice khi user đến từ MockTest (URL params present). Clear params ngay sau render đầu.

| Case | Scenario | Expected Behavior |
|------|----------|------------------|
| ✅ Happy | Đến từ MockTest, sessionId hợp lệ | Banner: `"Đang luyện theo gợi ý từ phiên thi [ngày] · Xem lại báo cáo ›"` |
| ✅ Happy | User navigate sang câu hỏi khác | Banner biến mất |
| ⚠️ Corner | sessionId không thuộc user hiện tại | Banner không xuất hiện (silent fail) |
| ⚠️ Corner | User nhấn browser back | Banner không re-appear (params đã clear bằng `router.replace`) |

**Acceptance Criteria:**

**AC-B01: Banner xuất hiện và params được clear**
- **Given:** User đến từ MockTest với params `?source=mock-report&criterion=X&sessionId=Y`
- **When:** Practice page mount
- **Then:** Banner render với đúng nội dung AND URL đổi thành `/practice` (params clear) AND browser back không quay về URL có params

**AC-B02: Banner biến mất khi navigate**
- **Given:** Banner đang hiển thị
- **When:** User chọn câu hỏi khác
- **Then:** Banner không còn hiển thị

**AC-B03: Click link quay lại report**
- **Given:** Banner hiển thị, sessionId hợp lệ
- **When:** User click `"Xem lại báo cáo ›"`
- **Then:** Navigate sang MockTest report của đúng sessionId đó

---

#### Feature C — Performance Page `/hieu-suat`: Action Card

**Mô tả:** 1 action card gợi ý action phù hợp theo trạng thái MockTest hiện tại. Priority: A > C > D > B.

**Logic states (cooldown = 7 ngày):**

| State | Điều kiện | Copy | Destination |
|-------|-----------|------|------------|
| **A** | Chưa có MockTest nào | `"Thi thử để biết band thật của bạn →"` | MockTest |
| **C** | MockTest cuối: 7–30 ngày trước | `"Đã [X] ngày kể từ lần thi — thử lại để cập nhật band →"` | MockTest |
| **D** | MockTest cuối: >30 ngày trước | `"Đã lâu không kiểm tra — thi thử để cập nhật band →"` | MockTest |
| **B** | MockTest cuối: trong 7 ngày qua | `"Tiếp tục luyện [Criterion yếu nhất] →"` | Practice |
| Default | Không match | Không hiển thị card | — |

> State B dùng cùng fallback order criterion: Vocabulary → Grammar → Fluency → Pronunciation.  
> State B **không** nudge "thi lại" — user vừa mới thi.

**Acceptance Criteria:**

**AC-C01: Đúng 1 card theo priority**
- **Given:** Bất kỳ trạng thái nào
- **When:** Performance page load
- **Then:** Hiển thị đúng 1 card match state có priority cao nhất (A > C > D > B) OR không hiển thị card nào nếu không match

**AC-C02: Copy và destination đúng theo từng state**
- **Given:** MockTest gần nhất [X] ngày trước
- **When:** Card render
- **Then:** Copy và destination click đúng với state tương ứng (theo bảng trên)

**AC-C03: State B không nudge "thi lại"**
- **Given:** MockTest gần nhất < 7 ngày trước (State B)
- **When:** Card render
- **Then:** Copy là `"Tiếp tục luyện [Criterion] →"` AND không có text liên quan đến "thi lại" hay "thi thử"

---

#### Feature D — Mini-drill Differentiation Label

**Mô tả:** Thêm label phụ cho nút `"Luyện ›"` trong tab Phát âm để phân biệt với Practice CTA.

**Acceptance Criteria:**

**AC-D01: Label phụ đúng chỗ, không thay đổi behavior**
- **Given:** User mở tab Phát âm trong MockTest report
- **When:** Tab render
- **Then:** Nút `"Luyện ›"` hiển thị kèm `"(Miễn phí · 1 từ)"` AND hành vi nút giữ nguyên AND các tab khác không có label này

---

### 4.2 Non-Functional Requirements

#### Performance
- Zone "Bước tiếp theo" render trong cùng paint với phần còn lại của Tổng quan (không lazy load riêng).
- Context Banner render synchronously từ URL params — không gây layout shift.

#### Compatibility
- Theo thiết lập hiện tại của 9Speak (Next.js, mobile-first web app).
- `router.replace` sử dụng Next.js router — không tương thích với native browser history API.

#### Constraint kiến trúc
- MockTest và Practice không được import lẫn nhau (ESLint rule).
- Mọi cross-module interaction = URL params only.
- URL params clear bằng `router.replace` (không tạo history entry mới).

---

## 5. User Types

| User Type | Định nghĩa | Behavior |
|-----------|-----------|---------|
| User thường (có lượt) | `remainingPracticeUses > 0` | Thấy CTA với số lượt còn lại |
| Premium user | Unlimited practice | Sub-text hiển thị "Unlimited" |
| User hết lượt | `remainingPracticeUses = 0` | Thấy upsell CTA in-context |
| User lần đầu | Chưa có MockTest nào | Performance page State A |
| User active | MockTest trong 7 ngày | Performance page State B (cooldown) |
| User lapsed | MockTest >7 ngày trước | Performance page State C hoặc D |

---

## 6. Analytics

### 6.1 Instrumentation

| Event | Trigger | Properties |
|-------|---------|-----------|
| `mocktest_report_cta_viewed` | Zone "Bước tiếp theo" render | `criterion`, `remaining_uses`, `session_id` |
| `mocktest_report_cta_clicked` | Click Practice CTA | `criterion`, `remaining_uses`, `session_id`, `cta_type: "practice" \| "upsell"` |
| `practice_context_banner_viewed` | Banner render trong Practice | `source_session_id`, `criterion` |
| `practice_context_banner_clicked` | Click "Xem lại báo cáo ›" | `source_session_id` |
| `performance_action_card_viewed` | Card render trên `/hieu-suat` | `state: "A" \| "B" \| "C" \| "D"`, `days_since_last_mocktest` |
| `performance_action_card_clicked` | Click action card | `state`, `destination: "mocktest" \| "practice"` |

### 6.2 Dashboard

| Dashboard | Metrics | Priority |
|-----------|---------|----------|
| Loop closing funnel | CTA view → click → Practice start → MockTest repeat | P0 |
| Cross-module rate | % users cross modules / 7 ngày | P0 |
| Upsell engagement | Clicks trên upsell CTA / tổng views 0-lượt | P1 |

---

## 7. Timelines

| Milestone | Target | Status |
|-----------|--------|--------|
| Dev handoff (Wave 1) | TBD | ⬜ |
| QA & review | TBD | ⬜ |
| Release Wave 1 | TBD | ⬜ |
| Đánh giá metrics (2 tuần) | TBD | ⬜ |
| Wave 2 kickoff (Before/After card, Performance Card) | Sau khi có data Wave 1 | ⬜ |

---

## 8. Dependencies

### 8.1 Internal

| Dependency | Owner | Status | Blocking? |
|-----------|-------|--------|-----------|
| `remainingPracticeUses` exposed tại root app (NineSpeak profile) | Engineering | ✅ Có sẵn | No |
| `history-client` từ `features/shared/` (Performance page đang dùng) | Engineering | ✅ Có sẵn | No |
| Billing flow API cho upsell in-context | Engineering | ⬜ Cần confirm | Yes (Feature A upsell) |
| MockTest report expose criterion bands đủ 4 tiêu chí | Engineering | ✅ Có sẵn | No |

### 8.2 External / V2 Blocker

| Dependency | Mô tả | Blocking gì |
|-----------|-------|------------|
| BFF `/api/app/speaking/practice/catalog` thêm `criterionTags` | Practice catalog chưa có tags | Deep-link criterion-filtered (V2) |

---

## 9. Appendix

| Tài liệu | Mô tả |
|---------|-------|
| [Handoff & decisions](./MOCKTEST_PRACTICE_CONNECTION_HANDOFF.md) | Toàn bộ decisions đã chốt, conflict resolutions, business rules |
| [Spec đầy đủ V1](./MOCKTEST_PRACTICE_CONNECTION_SPEC.md) | FR/BR/AC chi tiết cho cả 4 components |
| [MockTest Report §07](../../modules/mock-test/screens/07-report-tong-quan.md) | Tổng quan section — nơi đặt Feature A |
| [MockTest Report §08](../../modules/mock-test/screens/08-report-phan-hoi-chi-tiet.md) | Tab Phát âm — nơi đặt Feature D |
| [Practice PRD](../../modules/practice/PRD.md) | Practice module scope — Feature B |
| [Performance PRD](../../modules/performance/PRD.md) | Performance page — Feature C |

### V2 Backlog (ngoài scope PRD này)

| Item | Blocker |
|------|---------|
| Before/After delta card trong MockTest report | Cần `previousSession.overallBand` từ history-client |
| Deep-link criterion-filtered trong Practice | BFF cần `criterionTags` |
| Dual-CTA Pronunciation | Phụ thuộc criterion filter |
| Mixpanel: `attempt_number`, `score_delta` instrumentation | Data layer work |
