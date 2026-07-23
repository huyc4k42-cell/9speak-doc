# Screen Spec — Practice Search

**Module:** Practice  
**Trang:** Practice Index (`/practice`)  
**Trạng thái:** Ready for dev handoff  
**Ngày chốt:** 2026-07-13  

---

## Tóm tắt

Bổ sung thanh search cross-part vào màn Practice Index. User gõ từ khóa → dropdown kết quả tức thì → click thẳng vào phòng luyện. Không thay thế luồng duyệt topic cũ — chỉ bổ sung đường tắt.

---

## Mục lục

1. [Requirements](#1-requirements)
2. [API Contract](#2-api-contract)
3. [Analytics](#3-analytics)
4. [Non-functional](#4-non-functional)
5. [Quyết định thiết kế đã chốt](#5-quyết-định-thiết-kế-đã-chốt)

---

## 1. Requirements

### Feature A — Search bar + Dropdown (Desktop)

Thanh search đặt phía trên tab Part, cạnh dropdown filter Part. Khi user gõ ≥ 3 ký tự, sau debounce 300ms, gọi API và hiển thị kết quả dạng dropdown overlay ngay bên dưới.

**Cases:**

| Case | Scenario | Expected Behavior |
|---|---|---|
| ✅ Happy | Gõ ≥ 3 ký tự, có kết quả | Dropdown mở, skeleton 300ms → tối đa 8 kết quả |
| ✅ Happy | Click 1 kết quả | Navigate thẳng `/practice/part-X?topicId=Y&questionId=Z` |
| ✅ Happy | Xóa hết query | Dropdown đóng, layout gốc hiện lại |
| ✅ Happy | Đang có filter "Part 1" + gõ query | Chỉ trả kết quả thuộc Part 1 |
| ⚠️ Corner | Gõ < 3 ký tự | Không gọi API, không mở dropdown |
| ⚠️ Corner | Không có kết quả | Empty state: "Không tìm thấy câu hỏi nào cho '[query]'" |
| ⚠️ Corner | API lỗi / timeout | Trạng thái lỗi nhẹ trong dropdown, không crash trang |
| ⚠️ Corner | User gõ rất nhanh | Debounce 300ms — chỉ gọi API sau khi ngừng gõ |
| ⚠️ Corner | Nhấn Escape | Dropdown đóng, focus trả về input |

**AC-A1: Trigger search**
- **Given:** User gõ vào thanh search
- **When:** Query ≥ 3 ký tự VÀ đã 300ms kể từ lần gõ cuối
- **Then:** Gọi API `?q=[query]&part=[filter hiện tại]` VÀ dropdown mở với skeleton rows

**AC-A2: Không trigger khi query ngắn**
- **Given:** User gõ
- **When:** Query < 3 ký tự
- **Then:** Không gọi API VÀ dropdown không mở

**AC-A3: Hiển thị kết quả**
- **Given:** API trả về kết quả
- **When:** Data về client
- **Then:** Tối đa 8 items VÀ mỗi item gồm: tên câu hỏi + nhãn Part + tên topic VÀ skeleton được thay bằng data thật

**AC-A4: Empty state**
- **Given:** API trả về 0 kết quả
- **When:** Data về client
- **Then:** Dropdown hiện "Không tìm thấy câu hỏi nào cho '[query]'" — không hiện gợi ý thêm

**AC-A5: Navigate khi click**
- **Given:** Dropdown đang hiển thị kết quả
- **When:** User click 1 item
- **Then:** Navigate đến `/practice/part-[X]?topicId=[Y]&questionId=[Z]` VÀ dropdown đóng

**AC-A6: Xóa query**
- **Given:** Dropdown đang mở
- **When:** User xóa hết input
- **Then:** Dropdown đóng hoàn toàn VÀ layout gốc hiện lại bình thường

**AC-A7: Keyboard navigation**
- **Given:** Dropdown đang mở với ≥ 1 kết quả
- **When:** Nhấn ↓ / ↑ → focus di chuyển qua từng item
- **When:** Nhấn Enter → navigate (giống AC-A5)
- **When:** Nhấn Escape → dropdown đóng, focus trả về input

**AC-A8: Filter Part ảnh hưởng search**
- **Given:** Filter đang chọn "Part 2"
- **When:** User gõ query ≥ 3 ký tự
- **Then:** API gọi với `part=Part 2` VÀ kết quả chỉ có Part 2 VÀ filter không tự reset

**AC-A9: Loading state**
- **Given:** Search đã trigger (AC-A1)
- **When:** API chưa trả về
- **Then:** Dropdown hiện skeleton rows đúng hình dạng item — không để trống, không dùng spinner đơn lẻ

---

### Feature B — Full-screen Search (Mobile)

Trên mobile, thanh search là input button. Tap vào → mở overlay full-screen, input focus tự động, keyboard hiện. Logic search (debounce, ngưỡng, kết quả) giống desktop. Click kết quả → navigate + đóng overlay. "Hủy" → đóng, trả về Index.

**Cases:**

| Case | Scenario | Expected Behavior |
|---|---|---|
| ✅ Happy | Tap thanh search | Mở full-screen, input focus, keyboard hiện |
| ✅ Happy | Gõ ≥ 3 ký tự, có kết quả | List tối đa 8 kết quả |
| ✅ Happy | Tap 1 kết quả | Navigate vào phòng luyện, overlay đóng |
| ✅ Happy | Bấm "Hủy" | Overlay đóng, trở về Index |
| ⚠️ Corner | Không có kết quả | Empty state như desktop |
| ⚠️ Corner | Xóa hết query | List kết quả ẩn, overlay KHÔNG đóng (user gõ tiếp được) |

**AC-B1: Mở full-screen**
- **Given:** User ở Practice Index, mobile
- **When:** Tap thanh search
- **Then:** Full-screen mở VÀ input focus VÀ keyboard hiện

**AC-B2: Đóng full-screen**
- **Given:** Full-screen đang mở
- **When:** Bấm "Hủy" HOẶC back button hệ thống
- **Then:** Full-screen đóng VÀ về Index VÀ query không được lưu lại

**AC-B3: Xóa query trên mobile**
- **Given:** Full-screen đang mở, có kết quả
- **When:** User xóa hết input
- **Then:** List kết quả ẩn VÀ full-screen KHÔNG đóng

---

### Feature C — Filter Part tích hợp Search

Dropdown "Tất cả Part / Part 1 / Part 2 / Part 3" nằm cạnh thanh search. Có hai tác dụng đồng thời:
1. Khi không search: lọc danh sách topic theo Part
2. Khi search: thu hẹp phạm vi search xuống Part được chọn

**AC-C1: Filter áp dụng ngay cho cả topic list và search**
- **Given:** Tab Part 1 active, filter "Tất cả Part"
- **When:** User chọn "Part 2" trong filter
- **Then:** Danh sách topic hiện Part 2 VÀ nếu dropdown search đang mở thì kết quả search refresh theo Part 2

**AC-C2: Filter không reset khi search**
- **Given:** Filter đang chọn "Part 1"
- **When:** User bắt đầu gõ
- **Then:** Filter giữ nguyên "Part 1", không tự reset về "Tất cả Part"

---

## 2. API Contract

Endpoint mới (BFF):

```
GET /api/app/speaking/practice/search

Query params:
  q     : string  — required, min 3 chars
  part  : "Part 1" | "Part 2" | "Part 3"  — optional (omit = all parts)
  limit : number  — default 8, max 8

Response 200:
{
  results: Array<{
    questionId  : string,
    questionText: string,
    topicId     : string,
    topicTitle  : string,
    part        : "Part 1" | "Part 2" | "Part 3"
  }>,
  total: number
}
```

> **Dependency cần xác nhận trước khi dev:** NineSpeak backend đã có search endpoint chưa? Nếu chưa, BFF phải tự search trên catalog local. BFF contract trên đã đủ để frontend và backend làm song song.

Match scope: tên topic (`title`, `category`) + nội dung câu hỏi (`question.text`). Search server-side vì client không có toàn bộ catalog.

---

## 3. Analytics

| Event | Trigger | Properties |
|---|---|---|
| `search_performed` | Sau debounce 300ms, API được gọi | `query`, `part_filter: "all"\|"Part 1"\|"Part 2"\|"Part 3"`, `result_count` |
| `search_result_clicked` | Click 1 item trong dropdown/list | `query`, `part`, `topic_id`, `question_id`, `result_position` (1-indexed) |
| `search_no_results` | API trả về 0 kết quả | `query`, `part_filter` |
| `search_abandoned` | Gõ ≥ 3 ký tự rồi xóa/đóng mà không click | `query`, `result_count`, `time_spent_ms` |

> `search_no_results` → input trực tiếp cho content team để bổ sung catalog.

---

## 4. Non-functional

**Performance**
- API response ≤ 500ms p95
- LCP Practice Index không tăng quá 200ms so với baseline
- Debounce client-side 300ms — không spam request

**Accessibility (WCAG 2.2 AA)**
- Input: `role="combobox"`
- Dropdown: `role="listbox"`
- Mỗi item: `role="option"`
- `aria-activedescendant` cập nhật khi keyboard navigation
- Contrast text trong dropdown ≥ 4.5:1
- Mobile: focus trap trong full-screen overlay

**Compatibility**

| Platform | Min |
|---|---|
| Chrome | 110+ |
| Safari | 16+ |
| Firefox | 110+ |
| iOS Safari | 16+ |
| Android Chrome | 110+ |

---

## 5. Quyết định thiết kế đã chốt

| # | Topic | Quyết định |
|---|---|---|
| 1 | Phạm vi match | Topic title + question text — server-side |
| 2 | Cross-part | Search luôn cross-part; mỗi kết quả gắn nhãn Part |
| 3 | Desktop UI | Dropdown overlay ngay dưới thanh search |
| 4 | Mobile UI | Full-screen overlay khi tap |
| 5 | Item content | Tên câu hỏi + nhãn Part + tên topic. Không hiện trạng thái luyện tập ở MVP |
| 6 | Trigger | Debounce 300ms, ngưỡng tối thiểu 3 ký tự |
| 7 | Số kết quả | Tối đa 8, không phân trang |
| 8 | Click kết quả | Navigate thẳng `/practice/part-X?topicId=Y&questionId=Z` |
| 9 | Empty state | Chỉ "Không tìm thấy câu hỏi nào cho '[query]'" — không gợi ý thêm |
| 10 | Filter Part | Áp dụng đồng thời cho topic list + search scope |
| 11 | Filter behavior | Không tự reset khi user gõ |
| 12 | Loading | Skeleton rows |
| 13 | Query rỗng | Desktop: dropdown đóng. Mobile: list ẩn nhưng overlay giữ nguyên |
| 14 | Keyboard nav | ↑↓ di chuyển, Enter chọn, Escape đóng |
