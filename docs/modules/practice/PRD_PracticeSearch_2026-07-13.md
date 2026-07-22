# PRD — Tính năng Search trên Practice Index

**Module:** Practice  
**Ngày:** 2026-07-13  
**Trạng thái:** Draft  
**Tác giả:** TBD  

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

- Giảm thời gian user tìm câu hỏi muốn luyện trên màn Practice Index — từ đó tăng tỉ lệ bắt đầu phiên luyện tập.
- Cải thiện khả năng khám phá nội dung: user có thể tìm câu hỏi cụ thể mà không cần biết nó thuộc topic hay Part nào.
- Tăng tần suất luyện tập bằng cách giảm friction ở điểm vào.

### 1.2 User Benefits

- Tìm ngay câu hỏi cụ thể bằng cách gõ từ khóa — không cần duyệt qua từng topic/Part.
- Nhảy thẳng vào phòng luyện sau một thao tác tìm kiếm, không qua màn trung gian.
- Filter nhanh theo Part khi đã biết câu hỏi thuộc phạm vi nào.

---

## 2. Context

### 2.1 Vấn đề hiện tại

- Màn Practice Index hiện tổ chức nội dung theo **3 tab Part → danh sách topic → danh sách câu hỏi** — user phải duyệt tuần tự, không có đường tắt.
- Catalog có thể có hàng trăm câu hỏi phân tán qua nhiều topic và Part; user nhớ nội dung câu hỏi (vd "Are you working or studying?") nhưng không nhớ nó thuộc topic "Daily Routine" hay Part mấy.
- Không có cơ chế search → user bị buộc phải scroll và duyệt thủ công, tốn thời gian, dễ bỏ cuộc.
- Data user hành vi hiện chưa đủ để có baseline chính xác (TBD sau khi triển khai analytics).

### 2.2 Hướng giải quyết

- Bổ sung thanh search cross-part với dropdown kết quả tức thì, kết hợp filter Part đã có sẵn trên UI.
- Kết quả search match cả tên topic lẫn nội dung câu hỏi — phủ đúng use case "nhớ câu hỏi, không nhớ topic".
- Click kết quả → navigate thẳng vào phòng luyện, không thêm bước xác nhận.

---

## 3. Metrics

### 3.1 Measurement Mechanism

- **Sample:** Toàn bộ user đã đăng nhập truy cập màn Practice Index.
- **Method:** Analytics event tracking (Mixpanel) — đo hành vi thực tế sau khi release.

### 3.2 North Star Metric

- **Tỉ lệ session luyện tập được bắt đầu** (click "Bắt đầu luyện tập" hoặc click câu hỏi trong kết quả search → vào phòng luyện) / tổng lượt vào Practice Index.

### 3.3 Success Metrics

| Metric | Baseline | Target |
|--------|----------|--------|
| Tỉ lệ user sử dụng search ít nhất 1 lần trong session | TBD | TBD — đo sau 30 ngày |
| Tỉ lệ `search_result_clicked` / `search_performed` (search-to-click conversion) | TBD | ≥ 60% |
| Tỉ lệ `search_no_results` / `search_performed` | TBD | ≤ 20% |

> Baseline TBD — chưa có data trước khi triển khai. Cần đo sau 2–4 tuần đầu để đặt target có ý nghĩa.

### 3.4 Other Metrics

- Phân phối query length (để calibrate ngưỡng 3 ký tự).
- Top 20 query không có kết quả → input cho content team bổ sung catalog.
- Tỉ lệ `search_abandoned` / `search_performed` — đo friction của search experience.

### 3.5 Guardrail Metrics

- Thời gian load màn Practice Index (LCP) không tăng quá 200ms so với baseline hiện tại.
- Tỉ lệ bắt đầu luyện tập qua luồng cũ (duyệt topic thủ công) không giảm — search là bổ sung, không thay thế.

---

## 4. Requirements

### 4.1 Functional Requirements

---

#### Feature A: Thanh Search + Dropdown kết quả (Desktop)

**Design Resources:**
- Figma UI Design: TBD
- Figma Prototype: TBD

---

**Mô tả hành vi:**

Thanh search đặt phía trên tab Part, ngay cạnh dropdown filter "Tất cả Part". Khi user gõ ≥ 3 ký tự, sau debounce 300ms, hệ thống gọi API search và hiển thị kết quả dạng dropdown overlay ngay bên dưới thanh search.

---

**Cases:**

| Case | Scenario | Expected Behavior |
|------|----------|-------------------|
| ✅ Happy | Gõ ≥ 3 ký tự, có kết quả | Dropdown mở, hiện skeleton 300ms, sau đó render tối đa 8 kết quả |
| ✅ Happy | Click vào 1 kết quả | Navigate thẳng đến `/practice/part-X?topicId=Y&questionId=Z` |
| ✅ Happy | Xóa hết query (rỗng) | Dropdown đóng hoàn toàn, layout gốc hiện lại |
| ✅ Happy | Gõ query + filter "Part 1" | Chỉ trả kết quả thuộc Part 1 |
| ⚠️ Corner | Gõ < 3 ký tự | Không gọi API, không mở dropdown |
| ⚠️ Corner | Query ≥ 3 ký tự, không có kết quả | Dropdown mở, hiện empty state "Không tìm thấy câu hỏi nào cho '[query]'" |
| ⚠️ Corner | API search lỗi / timeout | Dropdown hiện trạng thái lỗi nhẹ, không crash toàn trang |
| ⚠️ Corner | User gõ rất nhanh | Debounce 300ms — chỉ gọi API sau khi ngừng gõ, không spam request |
| ⚠️ Corner | Nhấn Escape khi dropdown đang mở | Dropdown đóng, focus trả về input |

---

**Acceptance Criteria:**

**AC-A1: Trigger search**
- **Given:** User đang ở màn Practice Index (desktop), đã gõ vào thanh search
- **When:** Độ dài query ≥ 3 ký tự VÀ đã trôi qua 300ms kể từ lần gõ cuối
- **Then:** Hệ thống gọi API search với params `q=[query]&part=[filter hiện tại]` VÀ dropdown mở ra hiển thị skeleton rows

**AC-A2: Không trigger khi query ngắn**
- **Given:** User đang gõ vào thanh search
- **When:** Độ dài query < 3 ký tự
- **Then:** Không gọi API VÀ dropdown không mở

**AC-A3: Hiển thị kết quả**
- **Given:** API search trả về kết quả
- **When:** Data về client
- **Then:** Dropdown hiển thị tối đa 8 kết quả VÀ mỗi item gồm: tên câu hỏi + nhãn Part (Part 1/2/3) + tên topic VÀ skeleton được thay thế bằng kết quả thật

**AC-A4: Empty state**
- **Given:** API search trả về 0 kết quả
- **When:** Data về client
- **Then:** Dropdown hiển thị text "Không tìm thấy câu hỏi nào cho '[query]'" VÀ không hiển thị gợi ý thêm

**AC-A5: Navigate khi click kết quả**
- **Given:** Dropdown đang hiển thị kết quả
- **When:** User click vào 1 item bất kỳ
- **Then:** Hệ thống navigate đến `/practice/part-[X]?topicId=[Y]&questionId=[Z]` tương ứng VÀ dropdown đóng lại

**AC-A6: Đóng dropdown khi xóa query**
- **Given:** Dropdown đang mở
- **When:** User xóa hết nội dung input (input rỗng)
- **Then:** Dropdown đóng hoàn toàn VÀ layout gốc (danh sách topic + panel phải) hiển thị lại bình thường

**AC-A7: Keyboard navigation**
- **Given:** Dropdown đang mở với ít nhất 1 kết quả
- **When:** User nhấn phím ↓ / ↑
- **Then:** Focus di chuyển xuống / lên qua từng item trong dropdown
- **When:** User nhấn Enter khi 1 item đang được focus
- **Then:** Navigate đến phòng luyện của item đó (giống AC-A5)
- **When:** User nhấn Escape
- **Then:** Dropdown đóng VÀ focus trả về input

**AC-A8: Filter Part ảnh hưởng đến search**
- **Given:** Dropdown filter đang chọn "Part 2"
- **When:** User gõ query ≥ 3 ký tự
- **Then:** API được gọi với `part=Part 2` VÀ kết quả chỉ chứa câu hỏi thuộc Part 2 VÀ filter không tự reset về "Tất cả Part"

**AC-A9: Loading state**
- **Given:** User đã trigger search (AC-A1)
- **When:** API chưa trả về response
- **Then:** Dropdown hiển thị skeleton rows đúng hình dạng item kết quả (không để trống hoặc spinner đơn lẻ)

---

#### Feature B: Full-screen Search (Mobile)

**Mô tả hành vi:**

Trên mobile, thanh search hiển thị dạng input button. Khi user tap vào, mở màn full-screen search riêng (overlay toàn màn hình) với input được focus tự động. Behavior search (debounce, ngưỡng, kết quả) giống desktop. Bấm kết quả → navigate, màn full-screen đóng. Bấm "Hủy" hoặc swipe down → đóng màn full-screen, trả về Index.

---

**Cases:**

| Case | Scenario | Expected Behavior |
|------|----------|-------------------|
| ✅ Happy | Tap vào thanh search | Mở màn full-screen, input được focus, keyboard mở |
| ✅ Happy | Gõ ≥ 3 ký tự, có kết quả | Hiện danh sách kết quả dạng list (tối đa 8 item) |
| ✅ Happy | Tap vào 1 kết quả | Navigate thẳng vào phòng luyện, màn search đóng |
| ✅ Happy | Bấm "Hủy" | Màn full-screen đóng, trả về Index |
| ⚠️ Corner | Không có kết quả | Empty state tương tự desktop |
| ⚠️ Corner | Xóa hết query | List kết quả ẩn, không đóng màn full-screen (user vẫn có thể gõ lại) |

---

**Acceptance Criteria:**

**AC-B1: Mở màn full-screen**
- **Given:** User đang ở Practice Index trên mobile
- **When:** User tap vào thanh search
- **Then:** Màn full-screen search mở ra VÀ input được focus tự động VÀ keyboard hiển thị

**AC-B2: Đóng màn full-screen**
- **Given:** Màn full-screen search đang mở
- **When:** User bấm "Hủy" HOẶC nhấn back button hệ thống
- **Then:** Màn full-screen đóng VÀ user trở về Practice Index VÀ query không được lưu lại

**AC-B3: Xóa query trên mobile**
- **Given:** Màn full-screen đang mở VÀ đang có kết quả
- **When:** User xóa hết nội dung input
- **Then:** Danh sách kết quả ẩn VÀ màn full-screen KHÔNG đóng (user vẫn có thể gõ tiếp)

---

#### Feature C: Filter Part tích hợp với Search

**Mô tả hành vi:**

Dropdown "Tất cả Part / Part 1 / Part 2 / Part 3" nằm cạnh thanh search. Control này có hai tác dụng:
1. Khi không search: lọc danh sách topic bên dưới theo Part.
2. Khi search: thu hẹp phạm vi search xuống Part được chọn.

Hai tác dụng hoạt động đồng thời — thay đổi filter luôn áp dụng ngay cho cả danh sách topic lẫn kết quả search đang hiển thị.

---

**Acceptance Criteria:**

**AC-C1: Filter độc lập với tab Part**
- **Given:** Tab Part 1 đang active VÀ dropdown filter đang là "Tất cả Part"
- **When:** User chọn "Part 2" trong dropdown filter
- **Then:** Danh sách topic bên dưới hiển thị topic của Part 2 VÀ nếu đang có dropdown search mở thì kết quả search cũng refresh theo Part 2

**AC-C2: Filter không reset khi search**
- **Given:** Dropdown filter đang chọn "Part 1"
- **When:** User bắt đầu gõ vào thanh search
- **Then:** Dropdown filter giữ nguyên "Part 1" (không tự reset về "Tất cả Part")

---

### 4.2 Non-Functional Requirements

#### Performance
- API search phải trả response trong **≤ 500ms** ở p95 để dropdown không gây cảm giác lag sau debounce 300ms.
- Không tăng LCP của màn Practice Index quá **200ms** so với baseline hiện tại.
- Debounce client-side đảm bảo không gửi quá 1 request mỗi 300ms.

#### Compatibility

| Platform | Phiên bản tối thiểu |
|----------|---------------------|
| Chrome | 110+ |
| Safari | 16+ |
| Firefox | 110+ |
| iOS Safari | 16+ |
| Android Chrome | 110+ |

#### Accessibility
- Dropdown tuân thủ ARIA pattern `combobox` — có `role="combobox"` trên input, `role="listbox"` trên dropdown, `role="option"` trên từng item.
- `aria-activedescendant` cập nhật khi keyboard navigation.
- Contrast ratio của text trong dropdown ≥ 4.5:1 (WCAG 2.2 AA).
- Màn full-screen mobile focus trap đúng chuẩn — không để focus thoát ra ngoài overlay khi đang mở.

#### API Contract (BFF)

Endpoint mới cần định nghĩa:

```
GET /api/app/speaking/practice/search
Query params:
  q: string (required, min 3 chars)
  part: "Part 1" | "Part 2" | "Part 3" | undefined (optional, undefined = all parts)
  limit: number (default 8, max 8)

Response:
{
  results: Array<{
    questionId: MockExamEntityId,
    questionText: string,
    topicId: MockExamEntityId,
    topicTitle: string,
    part: "Part 1" | "Part 2" | "Part 3"
  }>,
  total: number
}
```

> **Lưu ý:** Cần xác nhận với NineSpeak backend xem đã có search endpoint chưa. BFF contract trên định nghĩa trước để frontend và backend làm song song.

---

## 5. User Types

| User Type | Định nghĩa | Behavior với Search |
|-----------|-----------|---------------------|
| Học viên đã đăng nhập | User có Firebase session | Truy cập đầy đủ search — kết quả click → navigate vào phòng luyện |
| Khách chưa đăng nhập | Không có Firebase session | Search vẫn hoạt động (read-only catalog); click kết quả → redirect login trước khi vào phòng luyện (behavior hiện tại của toàn module) |

---

## 6. Analytics

### 6.1 Instrumentation

| Event Name | Trigger | Properties |
|-----------|---------|------------|
| `search_performed` | Sau debounce 300ms, API được gọi | `query: string`, `part_filter: "all" \| "Part 1" \| "Part 2" \| "Part 3"`, `result_count: number` |
| `search_result_clicked` | User click vào 1 item trong dropdown / list | `query: string`, `part: "Part 1" \| "Part 2" \| "Part 3"`, `topic_id: string`, `question_id: string`, `result_position: number` (1-indexed) |
| `search_no_results` | API trả về 0 kết quả | `query: string`, `part_filter: "all" \| "Part 1" \| "Part 2" \| "Part 3"` |
| `search_abandoned` | User gõ ≥ 3 ký tự rồi xóa query hoặc đóng dropdown mà không click kết quả nào | `query: string`, `result_count: number`, `time_spent_ms: number` |

> `search_no_results` là input trực tiếp cho content team để bổ sung câu hỏi còn thiếu trong catalog.

### 6.2 Dashboard

| Dashboard | Metrics | Priority |
|-----------|---------|----------|
| Practice Search Performance | search_performed / ngày, search-to-click rate, top 20 no-result queries | P1 — sau 2 tuần release |

---

## 7. Timelines

| Milestone | Target Date | Status |
|-----------|------------|--------|
| PRD finalized & sign-off | TBD | ⬜ Not started |
| Xác nhận backend search endpoint | TBD | ⬜ Not started |
| Design Figma (desktop + mobile) | TBD | ⬜ Not started |
| Development handoff | TBD | ⬜ Not started |
| QA & Testing | TBD | ⬜ Not started |
| Release | TBD | ⬜ Not started |

---

## 8. Dependencies

### 8.1 Internal

| Dependency | Owner | Status |
|-----------|-------|--------|
| Xác nhận NineSpeak backend có search endpoint hay cần xây mới | Backend team | ⬜ Pending |
| BFF route handler `/api/app/speaking/practice/search` | Backend/BFF owner | ⬜ Pending |
| Figma design cho dropdown desktop + full-screen mobile | Design | ⬜ Pending |
| Analytics events đăng ký vào Mixpanel tracking plan | Data/Analytics | ⬜ Pending |

### 8.2 External

Không có dependency external ở MVP này.

---

## 9. Appendix

### Quyết định thiết kế đã chốt (từ Grill Session 2026-07-13)

| # | Topic | Quyết định |
|---|-------|------------|
| 1 | Phạm vi search | Match cả tên topic (`title`, `category`) lẫn nội dung câu hỏi (`question.text`) — yêu cầu API server-side vì client không có toàn bộ câu hỏi |
| 2 | Cross-part | Search luôn cross-part (Part 1+2+3); mỗi kết quả gắn nhãn Part tương ứng |
| 3 | UI pattern desktop | Dropdown overlay ngay dưới thanh search — không thay thế layout gốc |
| 4 | UI pattern mobile | Full-screen search overlay khi tap vào thanh search |
| 5 | Item content | Mỗi item: tên câu hỏi + nhãn Part + tên topic. Không hiện trạng thái luyện tập ở MVP |
| 6 | Trigger | Debounce 300ms khi gõ; ngưỡng tối thiểu 3 ký tự |
| 7 | Số kết quả | Tối đa 8 kết quả, không có phân trang |
| 8 | Click kết quả | Navigate thẳng vào phòng luyện (`/practice/part-X?topicId=Y&questionId=Z`) |
| 9 | Empty state | Chỉ hiện "Không tìm thấy câu hỏi nào cho '[query]'", không có gợi ý thêm |
| 10 | Filter Part | Dropdown "Tất cả Part" có hai tác dụng: lọc danh sách topic + thu hẹp phạm vi search |
| 11 | Filter behavior | Search dùng filter đang chọn, không tự reset về "Tất cả Part" khi user gõ |
| 12 | Loading | Skeleton rows trong dropdown |
| 13 | Query rỗng | Dropdown đóng hoàn toàn |
| 14 | Keyboard nav | ↑↓ di chuyển, Enter chọn, Escape đóng |
| 15 | Analytics | 4 events: `search_performed`, `search_result_clicked`, `search_no_results`, `search_abandoned` |
| 16 | Backend | Cần xác nhận; BFF contract định nghĩa trước để làm song song |

### Tài liệu tham chiếu

| Tài liệu | Link |
|----------|------|
| PRD Module Practice | `docs/modules/practice/PRD.md` |
| Practice Optimize Full Spec | `docs/new-requirements/Markdown/PRACTICE_OPTIMIZE_FULL_SPEC.md` |
| App API Types | `features/shared/types/app-api.ts` |
| Practice Index (code hiện tại) | `features/practice/pages/PracticeIndex.tsx` |
