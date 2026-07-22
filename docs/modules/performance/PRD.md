# PRD — Performance Progress Page (MVP)

**Sản phẩm:** 9Speak — IELTS Speaking Practice
**Phiên bản:** MVP v1.0
**Ngày:** 2026-06-21
**Trạng thái:** Draft — Sẵn sàng bàn giao dev

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

- Tăng retention bằng cách cho user bằng chứng rõ ràng rằng luyện tập đang có tác dụng → giảm churn do "cảm giác không tiến bộ"
- Hợp nhất 2 nav item (Lịch sử + Hiệu suất) thành 1 → giảm cognitive load sidebar, tăng discoverability của trang performance
- Thiết lập nền tảng data visualization để mở rộng sang V2 (per-criterion trend, milestone rewards)

### 1.2 User Benefits

- User thấy được xu hướng band score theo thời gian (8 tuần) — không chỉ snapshot tĩnh
- User biết chính xác criterion nào cần tập trung nhất dựa trên gap so với mục tiêu cá nhân
- User truy cập lịch sử phiên học trong cùng 1 trang, không cần navigate riêng

---

## 2. Context

### 2.1 UXR Insights & Usage Data

- **Insight 1:** Trang `/hieu-suat` hiện không có trong sidebar → user phải biết URL mới vào được. Đây là trang có giá trị cao nhưng bị ẩn hoàn toàn.
- **Insight 2:** Performance page hiện tại hiển thị snapshot tĩnh (band score hôm nay) — không trả lời được câu hỏi "Tôi có đang tiến bộ không?" theo thời gian.
- **Insight 3:** Triple redundancy trong design cũ (radar chart + criteria cards + focus area) cùng nói một điều → cognitive overload, không có clear hierarchy.
- **Insight 4:** Recommendation logic bị lỗi — suggest cải thiện criterion có điểm cao nhất thay vì thấp nhất → mất trust của user.
- **Insight 5:** Progress bar scale 0–9 sai với thang điểm IELTS thực tế (4.0–9.0) → misleading về vị trí thực của user.

### 2.2 Recommendation

- Hợp nhất Lịch sử vào Tiến trình dưới dạng tab-based page, thêm vào sidebar nav
- Thay snapshot tĩnh bằng area chart time-series 8 tuần rolling
- Đơn giản hóa criteria section: bỏ radar chart, bỏ progress bar, chỉ giữ score + delta badge
- Fix recommendation logic: focus vào criterion có gap lớn nhất so với `targetBand` của user

---

## 3. Metrics

### 3.1 Measurement Mechanism

- **Sample:** Tất cả user đã có ít nhất 1 completed session
- **Method:** Mixpanel event tracking + so sánh trước/sau deploy

### 3.2 North Star Metric

**Sessions per active user per week** — trang Tiến trình phải drive user quay lại luyện tập, không chỉ xem

### 3.3 Success Metrics

- Tỷ lệ user ghé thăm `/hieu-suat` tăng ≥ 40% so với baseline (do thêm vào sidebar)
- Tỷ lệ user click vào recommendation card ≥ 15% trong số user xem trang
- Bounce rate từ trang Tiến trình giảm ≥ 20% (user đọc xong rồi đi luyện tập, không thoát app)

### 3.4 Supporting Metrics

- `performance_screen_viewed` event count per week
- `recommendation_card_clicked` rate theo từng loại (practice / mock-exam / ai-simulation)
- Tab switch rate: bao nhiêu % user click sang tab "Lịch sử" sau khi xem "Tổng quan"

### 3.5 Guardrail Metrics (không được giảm)

- Session completion rate toàn app không giảm > 3%
- Dashboard page view không giảm (trang Tiến trình không cannibalize Dashboard)
- `/history` 404 rate = 0 (redirect phải hoạt động đúng)

---

## 4. Requirements

### 4.1 Functional Requirements

---

#### Feature 1: Sidebar Navigation Update

**Mô tả:** Thay "Lịch sử luyện tập" (`/history`, icon `CalendarClock`) bằng "Tiến trình" (`/hieu-suat`, icon `TrendingUp`) trong `primaryNavigation` của `AppShell.tsx`.

| Case | Scenario | Expected Behavior |
|---|---|---|
| ✅ Happy | User click "Tiến trình" trong sidebar | Navigate tới `/hieu-suat`, tab "Tổng quan" mặc định |
| ✅ Happy | User đang ở trang khác | Active indicator highlight đúng "Tiến trình" khi path là `/hieu-suat/*` |
| ⚠️ Corner | User bookmark cũ `/history` | Redirect 301 về `/hieu-suat?tab=history` |
| ⚠️ Corner | Analytics slug | `slugFromPath` phải map `/hieu-suat` → `performance_screen` |

**AC-1.1: Sidebar nav item**
- **Given:** User đã login và đang ở bất kỳ trang nào trong AppShell
- **When:** User nhìn vào sidebar
- **Then:** Nav item "Tiến trình" với icon `TrendingUp` hiển thị ở vị trí thứ 2 (sau "Trang chủ") AND nav item "Lịch sử luyện tập" không còn xuất hiện

**AC-1.2: Redirect từ /history**
- **Given:** User navigate tới URL `/history` (bookmark cũ, link share)
- **When:** Next.js xử lý request
- **Then:** User được redirect tới `/hieu-suat?tab=history` với HTTP 301 AND không thấy 404

---

#### Feature 2: Tab-Based Page Structure

**Mô tả:** `/hieu-suat` dùng Radix UI `Tabs` với 2 tabs: "Tổng quan" và "Lịch sử". Tab state được sync với URL searchParam `?tab=overview|history`. Tab "Lịch sử" lazy-mount (chỉ fetch khi user click lần đầu).

| Case | Scenario | Expected Behavior |
|---|---|---|
| ✅ Happy | Vào `/hieu-suat` không có param | Tab "Tổng quan" active mặc định |
| ✅ Happy | Vào `/hieu-suat?tab=history` | Tab "Lịch sử" active ngay lập tức, fetch session list |
| ✅ Happy | Click tab "Lịch sử" lần đầu | Mount HistoryPage component, fetch data |
| ✅ Happy | Click lại tab "Tổng quan" | Không re-fetch, giữ state của "Tổng quan" |
| ⚠️ Corner | Back/forward browser | Tab state đồng bộ với URL, đúng tab được restore |
| ⚠️ Corner | Mobile: tab labels dài | Tab bar scroll horizontal, không wrap |

**AC-2.1: Tab navigation**
- **Given:** User đang ở `/hieu-suat`
- **When:** User click tab "Lịch sử"
- **Then:** URL đổi thành `/hieu-suat?tab=history` (không full reload) AND nội dung "Lịch sử" được render AND tab "Lịch sử" có `aria-selected="true"`

**AC-2.2: Lazy load tab Lịch sử**
- **Given:** User vào `/hieu-suat` lần đầu (tab Tổng quan)
- **When:** User chưa click tab "Lịch sử"
- **Then:** Không có network request nào tới session list API

---

#### Feature 3: Area Chart — Band Trend

**Mô tả:** `Recharts` `AreaChart` hiển thị band score trung bình theo tuần trong 8 tuần gần nhất. Data tính client-side từ `listSpeakingSessions()`, aggregate theo tuần (UTC+7). Không có time filter. Static label "8 tuần gần nhất" bên cạnh chart title.

**Component mới:** `BandTrendChart`

**Data pipeline:**

```
listSpeakingSessions()
  → filter: status === "completed" AND createdAt trong 8 tuần gần nhất (UTC+7)
  → group by ISO week (UTC+7)
  → tính avg(overallBand) mỗi tuần
  → output: [{week: "T1", band: 6.2}, {week: "T2", band: 6.5}, ...]
```

**Chart spec:**

- X-axis: `T1` → `T8` (T8 = tuần hiện tại)
- Y-axis: 4.0 → 9.0, gridlines mỗi 0.5 band
- Primary line: `overallBand`, stroke `#2563eb`, area fill `rgba(37,99,235,0.08)`
- Dashed line: `targetBand` từ `profile.targetBand`, stroke `#94a3b8`, `strokeDasharray="4 4"`
- Label "Mục tiêu X.X" trên dashed line: color `#475569` (slate-600, đạt contrast 4.5:1)
- Dot: chỉ hiện tại data points, ẩn tại tuần không có session
- Weeks không có session: gap trong line (không nối điểm)
- `<ResponsiveContainer width="100%" height={220}>`

**Stat row dưới chart** (4 pills):

| Pill | Background | Text color |
|---|---|---|
| "Band hiện tại X.X" | `bg-blue-50` (#eff6ff) | `text-blue-800` (#1e40af) |
| "Mục tiêu X.X" | `bg-slate-100` | `text-slate-700` (#475569) |
| "Còn lại X.X band" | `bg-orange-50` (#fff7ed) | `text-orange-800` (#9a3412) — 7.2:1 ✅ |
| "Xu hướng ↑/↓/→" | `bg-green-50` / `bg-red-50` | `text-green-800` / `text-red-800` — 9.1:1 ✅ |

| Case | Scenario | Expected Behavior |
|---|---|---|
| ✅ Happy | User có sessions trong 8 tuần | Chart hiện line đầy đủ, stat row có data thật |
| ✅ Happy | Một tuần không có session | Gap trong line, không nối điểm qua tuần trống |
| ⚠️ Corner | User mới, 0 sessions | Empty state: icon + "Hoàn thành bài luyện đầu tiên để bắt đầu theo dõi" + CTA |
| ⚠️ Corner | Chỉ có 1 session | Hiện 1 dot, không có line. Stat row hiện data từ session đó |
| ⚠️ Corner | `listSpeakingSessions()` fail | Chart ẩn, hiện error state nhẹ: "Không thể tải dữ liệu. Thử lại." |
| ⚠️ Corner | `profile.targetBand` = 0 (chưa set) | Ẩn dashed target line, ẩn pill "Còn lại". Hiện link "Đặt mục tiêu →" |

**AC-3.1: Chart render với data**
- **Given:** User có ít nhất 2 completed sessions trong 8 tuần qua
- **When:** Trang `/hieu-suat` tab "Tổng quan" load
- **Then:** Area chart hiển thị với data thật AND dashed target line hiển thị nếu `targetBand > 0` AND stat row hiển thị 4 pills với giá trị đúng

**AC-3.2: Empty state**
- **Given:** User chưa có completed session nào
- **When:** Tab "Tổng quan" load
- **Then:** Chart được thay thế bằng empty state có icon + copy + button "Bắt đầu luyện tập" link tới `/practice`

**AC-3.3: Accessibility chart**
- **Given:** Bất kỳ state nào của chart
- **When:** Screen reader focus vào chart container
- **Then:** `aria-label` mô tả: "Biểu đồ xu hướng band điểm 8 tuần gần nhất. Band hiện tại: X.X. Mục tiêu: X.X."

---

#### Feature 4: Criteria Cards (Simplified)

**Mô tả:** 4 cards trong grid 2×2 (mobile) / 1×4 (desktop `lg:`). Mỗi card: tên criterion + score (24px bold) + delta badge. **Không có progress bar.** Màu semantic giữ nguyên.

**Data source:** `fetchPerformance()` → `response.criteria`

**Delta badge logic:**

| Condition | Display | Style |
|---|---|---|
| `delta > 0` | `"+{delta}"` | `bg-emerald-50 text-emerald-700` |
| `delta < 0` | `"{delta}"` | `bg-red-50 text-red-700` |
| `delta === 0` hoặc null | `"Không đổi"` | `bg-slate-100 text-slate-600`, `aria-label="Không có thay đổi so với kỳ trước"` |

**Criteria color mapping:**

| Criterion | Left border | Text |
|---|---|---|
| Fluency & Coherence | `emerald-500` | `text-emerald-600` |
| Lexical Resource | `amber-500` | `text-amber-600` |
| Grammatical Range | `blue-500` | `text-blue-600` |
| Pronunciation | `purple-500` | `text-purple-600` |

| Case | Scenario | Expected Behavior |
|---|---|---|
| ✅ Happy | Có đủ data 4 criteria | 4 cards hiện đủ, score đúng, badge đúng |
| ⚠️ Corner | Một criterion chưa có data | Card hiện score "–" và badge "Chưa có dữ liệu" |

**AC-4.1: Criteria cards render**
- **Given:** `fetchPerformance()` trả về criteria data đầy đủ
- **When:** Tab "Tổng quan" hiển thị
- **Then:** 4 cards hiển thị với đúng score, đúng màu semantic, đúng delta badge AND không có progress bar nào

---

#### Feature 5: Focus Area — Smart Criterion Selection

**Mô tả:** Card "Bạn nên tập trung vào" hiển thị criterion có gap lớn nhất so với `targetBand`.

**Logic chọn focus criterion:**

```
gap = profile.targetBand - criterionScore  (cho cả 4 criteria)
focus = criterion có gap lớn nhất
```

**Fallback:** Nếu `targetBand = 0` (chưa set) → focus vào criterion có điểm thấp nhất tuyệt đối.

**Data source:** `profile.targetBand` (AuthProvider) + `fetchPerformance()` → `response.criteria` + `response.focusArea.actionItems[]`

| Case | Scenario | Expected Behavior |
|---|---|---|
| ✅ Happy | targetBand set, có đủ criteria | Focus vào criterion có gap lớn nhất. actionItems từ backend |
| ⚠️ Corner | targetBand = 0 (chưa set) | Fallback: focus vào criterion thấp nhất. Hiện nudge "Đặt mục tiêu để nhận gợi ý chính xác hơn →" |
| ⚠️ Corner | `actionItems[]` từ backend = [] | Ẩn checklist, chỉ hiện criterion name và description |
| ⚠️ Corner | Tất cả 4 criteria đều = targetBand | Hiện state "Bạn đã đạt mục tiêu trên tất cả tiêu chí!" với emerald styling |

**AC-5.1: Focus criterion logic**
- **Given:** `profile.targetBand = 7.5`, criteria = `{fluency: 6.5, vocabulary: 6.0, grammar: 6.5, pronunciation: 7.0}`
- **When:** Component tính focus criterion
- **Then:** Focus criterion = "Từ vựng" (gap = 7.5 − 6.0 = 1.5, lớn nhất) AND `actionItems[]` từ backend cho Vocabulary được hiển thị AND tối đa 3 items

**AC-5.2: actionItems cap**
- **Given:** Backend trả về `focusArea.actionItems` có 5 items
- **When:** Render focus area card
- **Then:** Chỉ 3 items đầu tiên được hiển thị, items thứ 4–5 bị bỏ qua

---

#### Feature 6: Recommendation Cards

**Mô tả:** 3 cards từ `fetchPerformance()` → `response.recommendations`. Thêm client-side filter: loại bỏ recommendation nào match với criterion đang có điểm cao nhất.

**Fix logic:**

```
highestScoringCriterion = max(criteria.fluency, criteria.vocabulary, criteria.grammar, criteria.pronunciation)
recommendations = response.recommendations.filter(r => r.criterion !== highestScoringCriterion)
// Nếu sau filter còn < 3 items → lấy generic recommendation từ backend hoặc dùng fallback
```

**AC-6.1: Recommendation không recommend điểm cao nhất**
- **Given:** Pronunciation = 7.0 (cao nhất), backend trả về recommendation "Sửa lỗi phát âm"
- **When:** Component render recommendations
- **Then:** "Sửa lỗi phát âm" KHÔNG xuất hiện trong danh sách 3 recommendations

---

#### Feature 7: Milestone Toast

**Mô tả:** Khi `fetchPerformance()` resolve và `response.overview.previousBandDelta > 0`, trigger `sonner` toast. **Không dùng banner tĩnh trên trang.**

**Toast spec:**

```
Title: "🎯 Band của bạn vừa tăng lên {currentBand}!"
Sub-text: "Tiếp tục luyện tập để giữ vững đà này."
Duration: 6000ms, dismissable
Position: bottom-right
Dismiss button: aria-label="Đóng thông báo"
```

**Dedup logic:**

```
key = `shown-milestone-band-${Math.floor(currentBand * 2)}`
TTL = 24h (lưu timestamp vào localStorage)
→ Không show lại cho cùng milestone trong 24h
```

**AC-7.1: Toast trigger**
- **Given:** `fetchPerformance()` trả về `previousBandDelta > 0` AND localStorage không có key dedup
- **When:** Data load xong
- **Then:** Toast xuất hiện góc bottom-right trong 6 giây AND localStorage key được set với timestamp hiện tại

**AC-7.2: Toast dedup**
- **Given:** User đã thấy toast "Band 6.5" trong 24h qua (localStorage key tồn tại)
- **When:** User reload trang Tiến trình
- **Then:** Toast KHÔNG xuất hiện lại

---

#### Feature 8: Tab Lịch sử (Lazy Load)

**Mô tả:** Tab "Lịch sử" mount `HistoryPage` component lần đầu user click. Sau đó giữ mounted (không unmount khi switch tab) để tránh re-fetch.

**Component reuse:** `HistoryPage` (hiện có trong `onboarding-dashboard`, không sửa logic)

**AC-8.1: Lazy mount**
- **Given:** User vào trang `/hieu-suat` tab "Tổng quan"
- **When:** User click tab "Lịch sử" lần đầu
- **Then:** `HistoryPage` được mount, fetch `listHydratedHistoryEntries()` bắt đầu, loading state hiển thị

**AC-8.2: Không re-fetch khi switch tab**
- **Given:** User đã xem tab "Lịch sử" (data đã loaded)
- **When:** User switch sang "Tổng quan" rồi switch lại "Lịch sử"
- **Then:** Không có network request mới, data cũ được hiển thị ngay

---

### 4.2 Non-Functional Requirements

#### Performance

- `listSpeakingSessions()` aggregate (tối đa 80 sessions × 8 tuần) phải hoàn thành trong < 50ms client-side
- Tab switch animation < 150ms
- Chart render < 200ms sau khi data sẵn sàng

#### Compatibility

| Platform | Minimum |
|---|---|
| Chrome | 110+ |
| Safari | 16+ |
| Firefox | 110+ |
| iOS Safari | 16+ |
| Mobile responsive | 375px / Tablet 768px / Desktop 1280px |

#### Accessibility — WCAG 2.1 AA

| Element | Requirement |
|---|---|
| Chart container | `role="img"` + `aria-label` mô tả data |
| Tabs | Radix UI Tabs (built-in `role="tablist"`, `role="tab"`, `aria-selected`) |
| Badge "Không đổi" | `aria-label="Không có thay đổi so với kỳ trước"` |
| Stat pills | Dark text trên light background — minimum 4.5:1 |
| Target line label | `color: #475569` (slate-600), KHÔNG dùng slate-400 |
| Toast dismiss button | `aria-label="Đóng thông báo"` |

#### Dark Mode

- Tất cả background tokens dùng CSS variables (`bg-card`, `bg-muted`, `text-foreground`)
- Chart area fill trong dark mode: `rgba(96,165,250,0.12)`
- Không dùng hardcode `text-slate-900` hay `bg-slate-100`

---

## 5. User Types

| User Type | Định nghĩa | Behavior đặc biệt |
|---|---|---|
| New user | Chưa có completed session nào | Empty state trên chart + empty state trên tab Lịch sử |
| Active user | Có ít nhất 1 completed session | Full experience, chart có data |
| User chưa set targetBand | `profile.targetBand = 0` | Focus area dùng fallback (lowest score), hiện nudge đặt mục tiêu |
| User đã đạt targetBand | `currentBand >= targetBand` | Focus area hiện congratulation state, suggest nâng mục tiêu |

---

## 6. Analytics

### 6.1 Mixpanel Events

| Event | Trigger | Properties |
|---|---|---|
| `performance_screen_viewed` | Mount tab "Tổng quan" | `tab: "overview"`, `has_data: boolean`, `current_band: number` |
| `history_tab_viewed` | Click tab "Lịch sử" lần đầu | `session_count: number` |
| `band_trend_chart_viewed` | Chart render thành công | `weeks_with_data: number`, `current_band: number`, `target_band: number` |
| `recommendation_card_clicked` | Click recommendation card | `recommendation_kind: string`, `recommendation_title: string`, `position: 1\|2\|3` |
| `focus_area_viewed` | Focus area card visible | `focus_criterion: string`, `gap_to_target: number` |
| `milestone_toast_shown` | Toast xuất hiện | `band_reached: number` |
| `milestone_toast_dismissed` | User click X | `band_reached: number`, `time_visible_ms: number` |

### 6.2 Dashboard (Mixpanel)

| Dashboard | Metrics | Priority |
|---|---|---|
| Performance Page Health | `performance_screen_viewed` / DAU, tab switch rate | P0 |
| Recommendation Effectiveness | `recommendation_card_clicked` rate by kind | P1 |

---

## 7. Timelines

| Milestone | Target | Status |
|---|---|---|
| Dev handoff (PRD này) | 2026-06-21 | ✅ Done |
| Implementation | TBD | ⬜ |
| QA & Review | TBD | ⬜ |
| Merge vào `dev` branch | TBD | ⬜ |
| Release lên `main` | TBD | ⬜ |

---

## 8. Dependencies

### 8.1 Internal

| Dependency | Chi tiết | Status |
|---|---|---|
| `fetchPerformance()` API | Đã có. Response phải có `focusArea.actionItems[]` | ✅ Available |
| `listSpeakingSessions()` | Đã có. Dùng cho chart aggregate | ✅ Available |
| `profile.targetBand` | Từ AuthProvider | ✅ Available |
| `sonner` toast | Đã có trong stack | ✅ Available |
| `recharts` | Đã có trong `package.json` | ✅ Available |
| Radix UI Tabs | Đã có trong stack | ✅ Available |
| `HistoryPage` component | Trong module `onboarding-dashboard` | ✅ Available |

### 8.2 Out of Scope (MVP)

- Endpoint `/performance/history` mới — không cần, dùng session list
- Per-criterion time-series chart — V2
- Push notification cho milestone — V2
- Streak freeze mechanic — V2

---

## 9. Appendix

### Accessibility Color Verification

| Element | Foreground | Background | Ratio | WCAG AA |
|---|---|---|---|---|
| "Còn lại X.X band" pill | `#9a3412` (orange-800) | `#fff7ed` (orange-50) | 7.2:1 | ✅ Pass |
| "Xu hướng" pill | `#14532d` (green-800) | `#f0fdf4` (green-50) | 9.1:1 | ✅ Pass |
| Target line label | `#475569` (slate-600) | `#ffffff` | 4.6:1 | ✅ Pass |
| ~~Target line label cũ~~ | ~~`#94a3b8` (slate-400)~~ | `#ffffff` | ~~2.0:1~~ | ❌ Fail |

### Glossary

| Term | Định nghĩa |
|---|---|
| Band | Điểm IELTS Speaking (thang 4.0–9.0, bước 0.5) |
| Criteria | 4 tiêu chí IELTS: Fluency & Coherence, Lexical Resource, Grammatical Range & Accuracy, Pronunciation |
| targetBand | Band mục tiêu user tự đặt khi onboarding (từ `profile.targetBand`) |
| Gap | `targetBand − criterionScore` — khoảng cách từ điểm hiện tại đến mục tiêu |
| UTC+7 | Múi giờ Việt Nam, dùng để tính tuần cho chart aggregate |
| Lazy-mount | Component chỉ được tạo khi cần, không unmount sau khi đã mount |
