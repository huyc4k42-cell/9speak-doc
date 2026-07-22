# PRD — Streak Widget (MVP)

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

- Tăng **daily retention** bằng cách tạo ra habit loop rõ ràng: học → thấy chuỗi tăng → muốn học tiếp
- Tạo **cost of inaction**: user thấy chuỗi sắp bị mất → có động lực học hôm nay thay vì bỏ qua
- Thiết lập nền tảng dữ liệu hành vi học tập để sau này phục vụ personalization và milestone rewards (V2)

### 1.2 User Benefits

- User thấy ngay bằng chứng rằng họ đang duy trì thói quen học đều đặn — không cần vào trang Tiến trình
- User được nhắc nhở nhẹ nhàng khi sắp hết ngày mà chưa học (at-risk state)
- User được ghi nhận khi đạt cột mốc (3/7/14/30 ngày) — tăng cảm giác thành tựu

---

## 2. Context

### 2.1 UXR Insights & Usage Data

- **Insight 1:** Retention tuần 2 của user mới thấp — user không thấy "tôi có đang tiến bộ không?" và không có lý do cụ thể để quay lại hôm sau ngoài động lực nội tại.
- **Insight 2:** Duolingo, Headspace, và Notion đều dùng streak như cơ chế retention hàng đầu. Habit formation qua streak đã được validate rộng rãi trong edtech.
- **Insight 3:** Dashboard hiện tại thiếu **persistent motivational signal** — user vào dashboard, thấy 2 card action (Thi thử / Luyện tập) và không có gì cho thấy họ đang "on track".
- **Insight 4:** Widget nhỏ gọn ở góc không làm phân tán attention khỏi core CTA (Bắt đầu thi / Bắt đầu luyện) trong khi vẫn cung cấp context liên tục.

### 2.2 Recommendation

- Thêm Streak Widget compact (160px) vào góc top-right Dashboard, bên trái "Mục tiêu" Band card
- Widget tồn tại **mọi lúc** trên Dashboard — không ẩn, không cần navigate riêng
- Click để expand panel thống kê, không chiếm màn hình
- Streak tính từ **thời điểm submit audio** (không chờ pipeline hoàn thành) — user không bị phạt vì lỗi server

---

## 3. Metrics

### 3.1 Measurement Mechanism

- **Sample:** Tất cả user đã có ít nhất 1 session (new user không đủ điều kiện đo streak retention)
- **Method:** Mixpanel event tracking, cohort retention analysis (D7, D14, D30)

### 3.2 North Star Metric

**D7 retention rate** — % user quay lại học trong 7 ngày đầu sau lần đầu. Streak widget là cơ chế chính để cải thiện con số này.

### 3.3 Success Metrics

- D7 retention tăng ≥ 10% so với baseline trước khi có streak
- Tỷ lệ user có streak ≥ 3 ngày trong 30 ngày đầu ≥ 30%
- `streak_widget_expanded` rate ≥ 20% trong số user nhìn thấy widget (scroll position metric)

### 3.4 Supporting Metrics

- Số session trung bình/tuần tăng so với pre-streak cohort
- `streak_at_risk_cta_clicked` rate (bao nhiêu % user at-risk click "Luyện ngay")
- Tỷ lệ user đạt milestone 7 ngày trong 30 ngày đầu
- `streak_milestone_toast_shown` count by milestone level (3/7/14/30)

### 3.5 Guardrail Metrics (không được giảm)

- Dashboard load time không tăng > 200ms (widget fetch là background, non-blocking)
- Session completion rate không giảm (widget không distract khỏi core flow)
- Core CTA click rate (Bắt đầu thi / Luyện tập) không giảm > 5%

---

## 4. Requirements

### 4.1 Functional Requirements

---

#### Feature 1: Streak Computation Logic

**Mô tả:** Tính streak từ dữ liệu session hiện có. Không cần backend endpoint mới.

**Data source:** `listSpeakingSessions()` — lấy tất cả session của user

**Thuật toán:**

```
qualifying session = bất kỳ session nào đã submit audio thành công
                   = status IN ["processing", "completed", "failed"]
                   (loại trừ: draft, cancelled, không có audio)

ngày học = toZonedTime(session.createdAt, 'Asia/Ho_Chi_Minh').date
         = UTC+7 hardcode (không theo device timezone)

streak = số ngày liên tiếp (kể từ hôm nay hoặc hôm qua) có ít nhất 1 qualifying session

nhiều session/ngày = 1 ngày, không cộng dồn

streak break = không có qualifying session trong ngày hôm qua (UTC+7)
```

**Ví dụ:**

| Ngày (UTC+7) | Sessions | Tính streak? |
|---|---|---|
| 21/06 (hôm nay) | 1 completed | ✅ |
| 20/06 | 2 failed, 1 processing | ✅ (failed vẫn tính) |
| 19/06 | 0 | ❌ Streak break |
| 18/06 | 1 completed | Chuỗi cũ, không liên quan |

→ Streak hiện tại = 2 ngày (20/06 + 21/06)
→ Best streak = xét toàn bộ lịch sử

**Caching:**

```
localStorage key: "streak-cache"
Value: { currentStreak, bestStreak, totalDays, totalSessions, startDate, lastUpdated }
TTL logic: invalidate sau mỗi session submit (không dùng time-based TTL)
Optimistic render: hiện cache ngay, fetch background để update
```

| Case | Scenario | Expected Behavior |
|---|---|---|
| ✅ Happy | User có session hôm nay | `currentStreak = N`, widget shows active state |
| ✅ Happy | User có session hôm qua, chưa có hôm nay | `currentStreak = N`, state = "active-not-yet" |
| ✅ Happy | Nhiều session cùng ngày | Ngày đó = 1 đơn vị, không nhân đôi |
| ✅ Happy | Session `failed` | Vẫn tính — lỗi server không phạt user |
| ✅ Happy | Session `processing` (mock exam vừa submit) | Vẫn tính — optimistic |
| ⚠️ Corner | User mới, 0 sessions | `currentStreak = 0`, state = "new" |
| ⚠️ Corner | Miss 1 ngày | `currentStreak = 0`, state = "broken", bestStreak giữ nguyên |
| ⚠️ Corner | `listSpeakingSessions()` fail | Hiện từ cache. Nếu không có cache: hiện widget skeleton, không crash |
| ⚠️ Corner | Timezone device khác UTC+7 | Luôn dùng UTC+7, bất kể device timezone |

**AC-1.1: Streak computation đúng timezone**
- **Given:** User submit session lúc 23:50 ngày 21/06 theo giờ Việt Nam (UTC+7)
- **When:** Streak được tính
- **Then:** Session đó được ghi nhận vào ngày 21/06 UTC+7, không phải 22/06 UTC

**AC-1.2: Failed session vẫn tính**
- **Given:** User submit mock exam lúc 20:00, pipeline trả về `failed` lúc 20:05
- **When:** Widget tính streak
- **Then:** Ngày hôm đó được ghi nhận là "đã học" — streak không bị break

**AC-1.3: Optimistic update sau submit**
- **Given:** User hoàn thành 1 session bất kỳ (practice hoặc mock exam)
- **When:** Audio được submit thành công (trước khi pipeline chạy)
- **Then:** localStorage cache được invalidate → widget re-fetch và hiện state "active" ngay lập tức

---

#### Feature 2: Collapsed Widget — 6 States

**Mô tả:** Widget 160px fixed width (desktop) / full-width (mobile), luôn hiển thị ở góc top-right Dashboard. Click để toggle expanded panel.

**Layout Desktop:**
```
[StreakWidget 160px] [gap 12px] [Mục tiêu card 150px]
```

**Layout Mobile (≤ 768px):**
```
[StreakWidget full-width]
[Mục tiêu card full-width]
(stack dọc, streak trên)
```

---

**State 1 — New User (0 qualifying sessions)**

```
bg: #f8fafc | border: 1.5px dashed #e2e8f0 | border-radius: 12px | padding: 12px 14px

Row 1: [Flame icon outline #94a3b8 18px] ["Bắt đầu chuỗi" 13px semibold #475569] [Chevron ▼ #94a3b8]
Row 2: 7 circles — 6 empty (bg #f1f5f9, border 1.5px #e2e8f0, 10px)
        + 1 today circle RIGHTMOST (bg transparent, border 2px #f97316)
Row 3: "Luyện ngay →" 11px #2563eb
```

---

**State 2 — Active, đã học hôm nay**

```
bg: #fff7ed | border: 1.5px solid #fed7aa | border-radius: 12px | padding: 12px 14px

Row 1: [Flame SVG filled #f97316 18px] ["12 ngày" 18px bold #030213] [Chevron ▼ #9a3412]
Row 2: 6 filled dots (bg #f97316, 10px)
        + today dot RIGHTMOST: bg #f97316 + 1px white inner ring + 2px #fb923c outer ring (total 14px)
Row 3: "từ ngày 07/06" 10px #94a3b8
```

---

**State 3 — Active, chưa học (trước 18:00 UTC+7)**

```
Giống State 2 NGOẠI TRỪ today dot (rightmost):
  bg transparent | border 2px solid #f97316 | kích thước 10px (không glow)
Row 3: "từ ngày 07/06" 10px #94a3b8
```

---

**State 4 — At-Risk (sau 18:00 UTC+7, chưa học)**

```
bg: #fffbeb | border: 2px solid #fcd34d | border-radius: 12px | padding: 12px 14px

Row 1: [Flame SVG filled #fb923c 18px] ["12 ngày" 18px bold #030213]
        [pulse dot 6px filled #fcd34d, margin-left 4px] [Chevron ▼]
Row 2: Giống State 3 (today dot = orange ring, empty)
Row 3: "Luyện ngay →" 11px bold #b45309  ← CTA thay sub-label
```

Pulse dot: CSS animation `pulse` 1.5s infinite (opacity 1 → 0.4 → 1).

---

**State 5 — Broken (ngày sau khi miss)**

```
bg: #f8fafc | border: 1.5px solid #e2e8f0 | border-radius: 12px | padding: 12px 14px

Row 1: [Flame SVG outline #94a3b8 18px] ["Kỷ lục: 12 ngày" 13px #475569] [Chevron ▼ #94a3b8]
Row 2: 7 circles all empty (bg #f1f5f9, border #e2e8f0)
        + today circle RIGHTMOST (bg transparent, border 2px #f97316)
Row 3: "Bắt đầu lại hôm nay →" 11px #2563eb
```

---

**State 6 — Milestone (24h sau khi đạt 3/7/14/30 ngày)**

```
Giống State 2 (Active, đã học) THÊM:
- Border: 2px solid #f97316 (mạnh hơn #fed7aa bình thường)
- Badge absolute top:-8px right:-8px:
    "🎉 7 ngày" | bg #fff7ed | border 1px #fed7aa | text #9a3412 10px bold
    border-radius 20px | padding 2px 8px
Row 3: "Kỷ lục mới!" 10px bold #9a3412  ← thay "từ ngày DD/MM" trong 24h
```

Milestone dedup: `localStorage.setItem('streak-milestone-shown-{days}', Date.now())`
Sau 24h: xóa key → sub-label trở về "từ ngày DD/MM".

---

**Accessibility cho tất cả collapsed states:**

```tsx
<Flame aria-hidden="true" />
<span className="sr-only">Chuỗi học:</span>

<div role="img" aria-label="7 ngày gần nhất: 6 ngày đã học, hôm nay chưa học" />

<button
  aria-label="Mở rộng thống kê streak"
  aria-expanded={isOpen}
  aria-controls="streak-panel"
/>

<a href="/practice" aria-label="Luyện ngay để giữ chuỗi học hôm nay">
  Luyện ngay →
</a>
```

| Case | Scenario | Expected Behavior |
|---|---|---|
| ✅ Happy | User ở State 2 (active) | Widget orange, số ngày đúng, today dot sáng nhất |
| ✅ Happy | User click widget | Panel mở, `aria-expanded="true"` |
| ✅ Happy | User nhấn Escape | Panel đóng |
| ⚠️ Corner | User submit session → State 3 → State 2 | Widget update optimistic ngay, không reload page |
| ⚠️ Corner | Đồng hồ qua 18:00 khi user đang xem Dashboard | State không tự đổi sang At-risk (chỉ update khi re-fetch hoặc user interact) |
| ⚠️ Corner | Mobile 375px | Widget full-width, row 1 horizontal, chevron right-aligned |

**AC-2.1: State switching**
- **Given:** User ở State 3 (chưa học hôm nay) và hoàn thành 1 practice session
- **When:** User quay lại Dashboard
- **Then:** Widget hiển thị State 2 (active, đã học hôm nay) với today dot filled AND streak count tăng thêm 1 nếu đây là ngày đầu tiên trong chuỗi mới

**AC-2.2: Milestone badge**
- **Given:** Streak của user vừa đạt đúng 7 ngày
- **When:** Widget render
- **Then:** Badge "🎉 7 ngày" hiển thị góc top-right widget AND sub-label "Kỷ lục mới!" AND border mạnh hơn (2px solid #f97316)

---

#### Feature 3: Expanded Panel

**Mô tả:** Panel 340px dropdown dưới widget, mở/đóng khi click widget. Không phải modal — anchor vào bottom edge của widget. Bao gồm: 4 stat tiles + calendar tháng hiện tại + so sánh tháng trước + footer copy.

**Panel ID:** `streak-panel` (referenced bởi `aria-controls` trên trigger)

```
Container: width 340px | bg white | border 1.5px #e2e8f0 | border-radius 12px
           shadow: 0 8px 24px rgba(0,0,0,0.10) | padding 16px
           position: absolute, top = widget.bottom + 8px, right = 0

── HEADER ──
Left: Flame SVG 16px #f97316 + "Thống kê streak" 14px bold #030213
Right: Chevron ▲ 14px #94a3b8, aria-label="Thu gọn thống kê streak"

── STATS 2×2 GRID (margin-top 14px) ──
4 tiles, 2 columns, gap 10px
Each tile: bg #f8fafc | border-radius 10px | padding 12px 14px

Tile 1 (top-left):   streak number 26px bold #f97316 / "Chuỗi hiện tại" 11px #717182
Tile 2 (top-right):  best streak 26px bold #2563eb / "Dài nhất" 11px #717182
                     Unit label "ngày" 10px #94a3b8 (cả 2 tile trên)
Tile 3 (bottom-left): total days 26px bold #030213 / "Tổng ngày học" 11px #717182
Tile 4 (bottom-right): total sessions 26px bold #030213 / "Tổng phiên" 11px #717182
                       (tổng tất cả session types, không breakdown)

── DIVIDER ──
1px solid #f1f5f9, margin 14px 0

── MONTHLY CALENDAR HEATMAP ── (⚠️ xem note - có thể bỏ MVP)
[xem spec đầy đủ ở bên dưới]

── FOOTER COPY ──
"✦ Bạn đang xây dựng thói quen học tốt." 12px italic #717182

── CTA BUTTON (chỉ hiện khi state = at-risk hoặc broken) ──
margin-top 12px, full width
bg #2563eb | text white 13px semibold | border-radius 8px | padding 10px
"Luyện tập ngay →"
aria-label: "Luyện tập ngay để giữ chuỗi học"
```

**Monthly Calendar Heatmap spec:**

```
HEADER ROW: "Tháng 6, 2026" 12px bold #030213 | "18 phiên" 12px bold #f97316

COLUMN HEADERS: T2 T3 T4 T5 T6 T7 CN — 10px #94a3b8

GRID: 7 columns (T2→CN), 4–5 rows, cell 16×16px, gap 4px, border-radius 4px
ROW LABELS: "T1"–"T5" bên phải grid, 10px #94a3b8

CELL TYPES:
  Padding (ngoài tháng): transparent
  Tương lai: bg #f5f7fa
  0 session: bg #f1f5f9
  1 session: bg #bfdbfe (blue-200)
  2–3 sessions: bg #60a5fa (blue-400)
  4+ sessions: bg #3b82f6 (blue-500)
  Hôm nay: bg theo session count + 2px ring #f97316

TOOLTIP hover: "{DD/MM}: {N} phiên học"
bg #030213 | text white 11px | border-radius 6px | padding 4px 8px

COMPARISON STAT:
  delta > 0: "↑ +{delta}% so với tháng {M-1}"  #16a34a
  delta < 0: "↓ −{|delta|}% so với tháng {M-1}" #dc2626
  delta = 0: "= Tương đương tháng {M-1}"         #717182
  prev = 0:  "Tháng trước: chưa học • Tháng này: {N} phiên 🎉" #2563eb
  no history: "Tháng đầu tiên — tiếp tục nào! 💪" #717182 italic

LEGEND: "ít" + 4 cells (#f1f5f9→#bfdbfe→#60a5fa→#3b82f6, 10×10px) + "nhiều"
```

**At-risk variant của panel:**
- Panel bg: `#fffbeb`, border: `1.5px solid #fcd34d`
- Footer copy: `"⚠️ Còn ít thời gian để giữ chuỗi hôm nay."` `#92400e`
- CTA button hiển thị

| Case | Scenario | Expected Behavior |
|---|---|---|
| ✅ Happy | User click widget lần đầu | Panel mở animation slide-down 150ms |
| ✅ Happy | User click lại widget | Panel đóng |
| ✅ Happy | User click ngoài panel | Panel đóng |
| ✅ Happy | State at-risk | Panel bg amber, CTA hiển thị |
| ✅ Happy | User hover cell đã qua | Tooltip "{DD/MM}: N phiên học" xuất hiện |
| ⚠️ Corner | User mới, 0 sessions tháng này | Tất cả cells #f1f5f9, today có orange ring |
| ⚠️ Corner | Tháng có 5 tuần | Row T5 hiển thị đúng, padding ô còn lại |
| ⚠️ Corner | Mobile < 340px | Panel width = calc(100vw - 32px), grid tự scale |
| ⚠️ Corner | Panel bị cắt bởi viewport bottom | Panel flip lên trên widget |

**AC-3.1: Stats hiển thị đúng data**
- **Given:** User có currentStreak = 12, bestStreak = 20, totalDays = 45, totalSessions = 68
- **When:** User mở panel
- **Then:** 4 tiles hiển thị đúng 12 / 20 / 45 / 68 với đúng màu (orange / blue / black / black)

**AC-3.4: Panel đóng khi click outside**
- **Given:** Expanded panel đang mở
- **When:** User click bất kỳ element nào ngoài panel và ngoài widget trigger
- **Then:** Panel đóng với animation fade-out 100ms AND `aria-expanded="false"` được update

---

#### Feature 4: Milestone Toast

**Mô tả:** Khi streak đạt đúng 3, 7, 14, hoặc 30 ngày sau khi submit session, trigger `sonner` toast. Toast chỉ hiện 1 lần mỗi milestone (không TTL — permanent dedup).

```
Toast content:
  Title: "🔥 Chuỗi {N} ngày!"
  Body: copy theo milestone:
    3 ngày:  "Bạn đã học đều đặn 3 ngày. Hãy tiếp tục!"
    7 ngày:  "1 tuần không nghỉ — thói quen đang hình thành."
    14 ngày: "2 tuần liên tiếp. Bạn đang tiến bộ thật sự."
    30 ngày: "30 ngày! Đây là thành tích đáng tự hào."
  Duration: 6000ms, dismissable | Position: bottom-right
  Dismiss: aria-label="Đóng thông báo"
```

**Dedup:**

```typescript
const milestoneKey = `streak-milestone-shown-${milestoneValue}`;
if (!localStorage.getItem(milestoneKey)) {
  toast(...);
  localStorage.setItem(milestoneKey, Date.now().toString());
}
// Không có TTL — milestone toast chỉ hiện 1 lần duy nhất
// Widget milestone STATE (badge + border) có TTL 24h riêng
```

**AC-4.1: Toast trigger đúng milestone**
- **Given:** User vừa hoàn thành session làm streak = 7
- **When:** Widget re-compute streak
- **Then:** Toast "🔥 Chuỗi 7 ngày!" xuất hiện AND localStorage key `streak-milestone-shown-7` được set AND Widget chuyển sang State 6

**AC-4.2: Toast không repeat**
- **Given:** User đã thấy toast milestone 7 ngày (localStorage key tồn tại)
- **When:** User học thêm và streak vẫn ≥ 7
- **Then:** Toast KHÔNG xuất hiện lại

---

#### Feature 5: Streak Banner trong Practice/Mock Flow

**Mô tả:** Sau khi hoàn thành session, hiển thị streak context nhỏ trên màn hình kết quả/processing. Không block flow, không cần interaction.

| Context | Vị trí | Copy |
|---|---|---|
| Practice completion screen | Banner nhỏ phía trên score card | `"🔥 Chuỗi {N} ngày — bài luyện hôm nay đã được ghi nhận."` |
| Mock Exam processing screen | Banner dưới tiêu đề "Đang xử lý bài thi..." | `"🔥 Chuỗi {N} ngày — bài thi đã nộp, kết quả sẽ có trong ít phút."` |
| Practice/Mock focus mode | **Không hiển thị** — giữ màn hình tập trung clean |

**Banner spec:**
```
bg #fff7ed | border 1px #fed7aa | border-radius 8px | padding 10px 14px
Flame icon 14px #f97316 (aria-hidden) + text 13px #92400e
```

**AC-5.1: Banner sau practice**
- **Given:** User hoàn thành Practice Part 1 với streak hiện tại = 5
- **When:** Màn hình kết quả hiển thị
- **Then:** Banner "🔥 Chuỗi 5 ngày — bài luyện hôm nay đã được ghi nhận." xuất hiện phía trên score

**AC-5.2: Banner KHÔNG xuất hiện trong focus mode**
- **Given:** User đang trong màn hình tập trung luyện tập (mic recording, countdown...)
- **When:** Bất kỳ lúc nào trong flow
- **Then:** Banner streak hoàn toàn không hiển thị

---

### 4.2 Non-Functional Requirements

#### Performance

- `listSpeakingSessions()` fetch là **background, non-blocking** — không block Dashboard render
- Widget render từ localStorage cache < 50ms (optimistic)
- Background fetch và re-render sau khi data về < 200ms
- Panel open/close animation ≤ 150ms

#### Accessibility — WCAG 2.1 AA

| Element | Requirement |
|---|---|
| Flame SVG icon | `aria-hidden="true"` + `<span className="sr-only">Chuỗi học:</span>` |
| 7-dot row | `role="img"` + `aria-label` mô tả số ngày đã học và hôm nay |
| Today dot | `aria-label="Hôm nay"` |
| Widget trigger button | `aria-label="Mở rộng thống kê streak"` + `aria-expanded={isOpen}` + `aria-controls="streak-panel"` |
| Panel container | `role="region"` + `aria-label="Thống kê streak"` + `id="streak-panel"` |
| "Luyện ngay" link | `aria-label="Luyện ngay để giữ chuỗi học hôm nay"` |
| Milestone badge | `aria-label="Đạt chuỗi {N} ngày"` |
| Toast dismiss button | `aria-label="Đóng thông báo"` |
| Pulse dot (at-risk) | `aria-hidden="true"` (decorative only) |

#### Dark Mode

| Element | Light | Dark |
|---|---|---|
| Widget active bg | `#fff7ed` | `#1a0f00` |
| Widget active border | `#fed7aa` | `#92400e` |
| Streak number | `#030213` | `#ffffff` |
| Flame icon | `#f97316` | `#f97316` |
| Dots filled | `#f97316` | `#f97316` |
| Dots empty | `#f1f5f9` | `#27272a` |
| Panel bg | `white` | `#141414` |
| Panel border | `#e2e8f0` | `#2a2a2a` |
| Stat tile bg | `#f8fafc` | `#1a1a1a` |
| Current streak number | `#f97316` | `#fb923c` (orange-400) |
| Best streak number | `#2563eb` | `#60a5fa` (blue-400) |
| Calendar — ngày tương lai | `#f5f7fa` | `#1f1f1f` |
| Calendar — 0 session | `#f1f5f9` | `#27272a` |
| Calendar — 1 session | `#bfdbfe` | `#1e3a5f` |
| Calendar — 2–3 sessions | `#60a5fa` | `#1d4ed8` |
| Calendar — 4+ sessions | `#3b82f6` | `#3b82f6` |
| Calendar — today ring | `#f97316` | `#f97316` |
| Comparison stat ↑ | `#16a34a` | `#4ade80` |
| Comparison stat ↓ | `#dc2626` | `#f87171` |

---

## 5. User Types

| User Type | Định nghĩa | Behavior |
|---|---|---|
| New user | 0 qualifying sessions | State 1: "Bắt đầu chuỗi", dashed border, empty dots |
| Active user (today) | Có session hôm nay | State 2: orange widget, today dot filled |
| Active user (not yet) | Có session hôm qua, chưa có hôm nay, trước 18:00 | State 3: orange widget, today dot empty |
| At-risk user | Chưa học hôm nay, sau 18:00 | State 4: amber widget, CTA visible, panel amber |
| Broken user | Miss ≥ 1 ngày | State 5: grey widget, "Kỷ lục: N ngày", start over |
| Milestone user | Vừa đạt 3/7/14/30 ngày | State 6: badge + "Kỷ lục mới!", toast triggered |

---

## 6. Analytics

### 6.1 Mixpanel Events

| Event | Trigger | Properties |
|---|---|---|
| `streak_widget_viewed` | Widget visible trong viewport | `state: "new"\|"active"\|"at-risk"\|"broken"\|"milestone"`, `current_streak: number` |
| `streak_widget_expanded` | User click để mở panel | `state`, `current_streak` |
| `streak_widget_collapsed` | User click để đóng panel | `time_open_ms: number` |
| `streak_at_risk_cta_clicked` | Click "Luyện ngay →" trong at-risk state | `current_streak`, `time_of_day_hour: number` |
| `streak_broken_cta_clicked` | Click "Bắt đầu lại" trong broken state | `previous_best_streak: number` |
| `streak_milestone_toast_shown` | Toast xuất hiện | `milestone_days: 3\|7\|14\|30` |
| `streak_milestone_toast_dismissed` | User click X | `milestone_days`, `time_visible_ms: number` |

### 6.2 Dashboard (Mixpanel)

| Dashboard | Metrics | Priority |
|---|---|---|
| Streak Health | D7/D14/D30 retention by streak cohort | P0 |
| At-Risk Conversion | `streak_at_risk_cta_clicked` / `streak_widget_viewed` where state=at-risk | P0 |
| Milestone Funnel | % users reaching 3 → 7 → 14 → 30 days | P1 |

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
| `listSpeakingSessions()` | Dùng để compute streak. Cần `createdAt` + `status` fields | ✅ Available |
| `date-fns-tz` | `toZonedTime(date, 'Asia/Ho_Chi_Minh')` cho UTC+7 | Cần confirm có trong package.json |
| `sonner` toast | Milestone toast | ✅ Available |
| `localStorage` | Streak cache + milestone dedup | ✅ Browser native |
| Practice completion screen | Cần add streak banner — xác định file path khi implement | ⬜ |
| Mock Exam processing screen | Cần add streak banner — xác định file path khi implement | ⬜ |

### 8.2 Out of Scope (MVP)

- Streak freeze mechanic — V2
- Push notification nhắc nhở học (cần service worker) — V2
- Session type breakdown trong stats — V2
- Sidebar badge cho streak — V2
- Social proof percentile dynamic — V2
- Streak sharing / social — V3

---

## 9. Appendix

### Hook Interface

```typescript
interface StreakData {
  currentStreak: number;
  bestStreak: number;
  totalDays: number;
  totalSessions: number;
  startDate: string | null;    // ISO string UTC+7
  practicedToday: boolean;
  state: StreakState;
  isLoading: boolean;
}

type StreakState = "new" | "active" | "active-not-yet" | "at-risk" | "broken" | "milestone";
// at-risk = sau 18:00 UTC+7 + practicedToday = false
// milestone = 24h sau khi currentStreak IN [3, 7, 14, 30] + localStorage dedup chưa expire
```

### Milestone Toast Copy

| Milestone | Toast Title | Toast Body |
|---|---|---|
| 3 ngày | "🔥 Chuỗi 3 ngày!" | "Bạn đã học đều đặn 3 ngày. Hãy tiếp tục!" |
| 7 ngày | "🔥 Chuỗi 7 ngày!" | "1 tuần không nghỉ — thói quen đang hình thành." |
| 14 ngày | "🔥 Chuỗi 14 ngày!" | "2 tuần liên tiếp. Bạn đang tiến bộ thật sự." |
| 30 ngày | "🔥 Chuỗi 30 ngày!" | "30 ngày! Đây là thành tích đáng tự hào." |

### Glossary

| Term | Định nghĩa |
|---|---|
| Qualifying session | Session có audio submit thành công (status: processing / completed / failed). Draft và cancelled không tính |
| Streak | Số ngày liên tiếp có ít nhất 1 qualifying session, tính theo UTC+7 |
| Best streak | Chuỗi dài nhất trong toàn bộ lịch sử user |
| At-risk | Trạng thái sau 18:00 UTC+7 mà user chưa học hôm nay |
| Milestone | Cột mốc streak đặc biệt: 3 / 7 / 14 / 30 ngày |
| UTC+7 | Múi giờ Việt Nam cố định (Asia/Ho_Chi_Minh), dùng để tính ranh giới ngày |
| Optimistic update | Cập nhật UI từ localStorage cache ngay lập tức, không chờ API response |
