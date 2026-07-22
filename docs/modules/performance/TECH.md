# TECH — Performance Progress Page

**Module:** performance
**Trạng thái:** Chưa implement — tài liệu này cần được điền khi bắt đầu triển khai

> Đọc kèm [PRD.md](PRD.md) để hiểu nghiệp vụ trước khi đọc spec kỹ thuật.

---

## Vị trí trong codebase

> ⚠️ **Cần xác định:** PRD tham chiếu `features/speaking/` nhưng cấu trúc hiện tại dùng `features/onboarding-dashboard/` cho trang Dashboard/History/Performance. Cần quyết định feature này nằm trong module nào (tạo `features/performance/` mới hay tích hợp vào `features/onboarding-dashboard/`) trước khi implement.

Module này ảnh hưởng:
- Trang `/hieu-suat` (Performance page) — hiện trong `onboarding-dashboard`
- Sidebar nav (AppShell) — hiện trong `onboarding-dashboard`
- Redirect `/history` → `/hieu-suat?tab=history` — `next.config.ts`

---

## Components cần tạo / sửa

| File | Action | Mô tả |
|---|---|---|
| `AppShell.tsx` | Sửa | Đổi nav item Lịch sử → Tiến trình (icon TrendingUp, path /hieu-suat) |
| `Performance.tsx` (page) | Sửa lớn | Thêm Radix Tabs wrapper, xóa radar chart, xóa progress bar, thêm BandTrendChart |
| `BandTrendChart.tsx` | **Tạo mới** | Recharts AreaChart với weekly aggregate, target line, stat row, empty state |
| `next.config.ts` | Sửa | Thêm redirect: `/history` → `/hieu-suat?tab=history` (HTTP 301) |
| `styles/sample-ui/theme.css` | Sửa | Thêm streak color tokens (xem PRD Appendix) |

---

## Data Sources

Không cần endpoint mới. Tất cả data từ:

| Source | Dùng cho |
|---|---|
| `listSpeakingSessions()` | Aggregate weekly band data cho chart |
| `fetchPerformance()` | Criteria scores, focus area, recommendations |
| `profile.targetBand` | Target line trên chart, focus criterion logic |
| `listHydratedHistoryEntries()` | Tab Lịch sử (lazy-mount, reuse HistoryPage) |

---

## State Management

- Tab state: sync với URL `?tab=overview\|history` (useSearchParams)
- Chart data: client-side aggregate từ session list
- HistoryPage: lazy-mount (mount lần đầu khi click tab, không unmount sau)
- Milestone toast dedup: `localStorage` key `shown-milestone-band-{N}`, TTL 24h

---

## Redirect

```typescript
// next.config.ts
redirects: async () => [
  {
    source: '/history',
    destination: '/hieu-suat?tab=history',
    permanent: true, // 301
  }
]
```

---

## Env & Dependencies

Không có env mới. Stack đã có:
- `recharts` (chart)
- Radix UI Tabs
- `sonner` (toast)

---

## Gotchas

- Chart aggregate phải dùng UTC+7 (`Asia/Ho_Chi_Minh`) — không dùng device timezone
- Focus criterion logic: `gap = targetBand - criterionScore`, chọn criterion có gap lớn nhất (không phải điểm thấp nhất nếu targetBand đã set)
- Recommendation filter: loại bỏ criterion có điểm **cao nhất** (fix lỗi logic cũ)
- Progress bar bị loại bỏ vĩnh viễn — không thêm lại
- Radar chart bị loại bỏ vĩnh viễn — không thêm lại
