# TECH — Streak Widget

**Module:** streak
**Trạng thái:** Chưa implement — tài liệu này cần được điền khi bắt đầu triển khai

> Đọc kèm [PRD.md](PRD.md) để hiểu nghiệp vụ trước khi đọc spec kỹ thuật.

---

## Vị trí trong codebase

> ⚠️ **Cần xác định:** PRD tham chiếu `features/speaking/` nhưng cấu trúc hiện tại dùng `features/onboarding-dashboard/` cho Dashboard. Cần quyết định widget này nằm trong module nào trước khi implement.

Module này ảnh hưởng:
- `Dashboard.tsx` (page) — thêm StreakWidget vào header right column
- Practice completion screen — thêm streak banner
- Mock Exam processing screen — thêm streak banner
- `styles/sample-ui/theme.css` — thêm streak color tokens

---

## Components / Hooks cần tạo

| File | Action | Mô tả |
|---|---|---|
| `StreakWidget.tsx` | **Tạo mới** | Collapsed widget (6 states) + expanded panel |
| `useStreak.ts` | **Tạo mới** | Computation logic, localStorage cache, re-fetch trigger |
| `Dashboard.tsx` | Sửa nhỏ | Thêm StreakWidget vào header right column |
| Practice completion screen | Sửa | Thêm streak banner (xác định file path khi implement) |
| Mock Exam processing screen | Sửa | Thêm streak banner (xác định file path khi implement) |
| `styles/sample-ui/theme.css` | Sửa | Thêm streak color tokens |

---

## Hook Interface

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
```

---

## Data Source

Không cần endpoint mới. Chỉ dùng `listSpeakingSessions()` — cần fields `createdAt` và `status`.

---

## Caching Strategy

```
localStorage key: "streak-cache"
Value: { currentStreak, bestStreak, totalDays, totalSessions, startDate, lastUpdated }
Invalidate: sau mỗi session submit (event-based, không time-based)
Render: optimistic từ cache, fetch background
```

---

## Timezone

Bắt buộc dùng `date-fns-tz` và `toZonedTime(date, 'Asia/Ho_Chi_Minh')`. Không dùng `new Date()` local timezone.

> Confirm `date-fns-tz` có trong `package.json` trước khi implement.

---

## Env & Dependencies

| Package | Mục đích | Status |
|---|---|---|
| `date-fns-tz` | UTC+7 timezone conversion | ⬜ Cần confirm |
| `sonner` | Milestone toast | ✅ Đã có |
| `localStorage` | Streak cache + dedup | ✅ Browser native |

---

## Color Tokens cần thêm vào theme.css

```css
--streak-active-bg: #fff7ed;
--streak-active-border: #fed7aa;
--streak-active-border-strong: #f97316;
--streak-icon: #f97316;
--streak-at-risk-bg: #fffbeb;
--streak-at-risk-border: #fcd34d;
--streak-at-risk-text: #b45309;
--streak-milestone-badge-bg: #fff7ed;
--streak-milestone-badge-text: #9a3412;
```

---

## Gotchas

- Streak tính từ **thời điểm submit audio**, không chờ pipeline — `status IN [processing, completed, failed]` đều qualify
- Nhiều session trong 1 ngày = 1 đơn vị (không cộng dồn)
- At-risk threshold: **18:00 UTC+7** hardcode
- Milestone toast dedup: **permanent** (không có TTL) — `streak-milestone-shown-{days}`
- Milestone widget state dedup: **24h TTL** — khác với toast
- Expanded panel: position absolute, cần handle viewport flip khi panel bị cắt bởi bottom edge
