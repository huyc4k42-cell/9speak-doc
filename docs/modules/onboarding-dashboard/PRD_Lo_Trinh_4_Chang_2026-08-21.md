# PRD — Lộ trình 4 chặng cá nhân hoá (Dashboard)

> Tài liệu nghiệp vụ thuần business. Không chứa chi tiết kỹ thuật (tên file, API, kiến trúc, code) — các quyết định kỹ thuật được ghi riêng ở tài liệu TECH khi bước vào giai đoạn triển khai.

**Trạng thái:** Draft v1 — sẵn sàng review
**Ngày viết:** 2026-08-21
**Người viết:** Arthur (qua AI hỗ trợ)
**Liên quan:** Module Onboarding/Dashboard, Module Mock Test, Module Practice

---

## Mục lục

1. [Goal](#1-goal)
2. [Context](#2-context)
3. [Metrics](#3-metrics)
4. [Requirements](#4-requirements)
5. [User Types](#5-user-types)
6. [Analytics](#6-analytics)
7. [Timelines](#7-timelines)
8. [Dependencies](#8-dependencies--approvals)
9. [Appendix](#9-appendix)

---

## 1. Goal

### 1.1 Business / Product Goal
- Tăng **activation**: biến câu trả lời onboarding (mục tiêu, deadline, khó khăn, hình thức ưa thích) thành một kế hoạch học cụ thể ngay từ lần đầu vào Dashboard, thay vì để người dùng tự loay hoay chọn việc cần làm.
- Tăng **retention**: cho người dùng lý do quay lại đều đặn — mỗi lần vào Dashboard đều thấy rõ "đang ở đâu, làm gì tiếp theo".

### 1.2 User Benefits
- Biết ngay band xuất phát thật của mình (không phải tự đoán).
- Có lộ trình cụ thể theo đúng khó khăn lớn nhất và cách học ưa thích của bản thân, không phải một lộ trình chung chung.
- Luôn biết chặng tiếp theo cần làm gì, không bị "mất phương hướng" giữa hàng chục bài luyện/thi thử có sẵn.
- Không bao giờ bị kẹt vĩnh viễn: mở khoá chặng tiếp theo dựa trên **nỗ lực đã bỏ ra** (đã làm đủ task), không dựa trên việc phải đạt band mục tiêu — tránh trường hợp người học mãi không tiến bộ đủ nhanh mà bị khoá chặng.

---

## 2. Context

### 2.1 Usage Data / UXR Insights
- Người dùng sau onboarding hiện được đưa thẳng vào Dashboard với các đề xuất rời rạc (card Thi thử, card Luyện tập) nhưng không có một **lộ trình liền mạch** nối các hoạt động này thành từng chặng có mục tiêu rõ ràng.
- Dữ liệu onboarding (mục tiêu band, deadline, khó khăn lớn nhất, hình thức học ưa thích) hiện được thu thập nhưng chưa được dùng để cá nhân hoá trải nghiệm sau đó — thu thập xong rồi "cất tủ".
- Rủi ro đã biết: nếu gate chặng tiếp theo bằng việc phải đạt band mục tiêu, người học tiến bộ chậm sẽ bị kẹt vĩnh viễn ở một chặng — đây là lý do cốt lõi khiến quyết định mở khoá theo nỗ lực (task hoàn thành) thay vì theo band được chọn ngay từ đầu.

### 2.2 Recommendation
- Xây một widget **Lộ trình 4 chặng** trên Dashboard, sinh tự động từ dữ liệu onboarding đã có sẵn (không hỏi lại người dùng thêm câu nào mới), thể hiện: đã ở đâu (band xuất phát) → đang làm gì (chặng hiện tại + task) → còn bao xa tới đích (band mục tiêu).
- Toàn bộ nội dung lộ trình đóng băng mặc định sau khi sinh — chỉ tính toán lại khi người dùng chủ động đổi mục tiêu band hoặc deadline, có xác nhận trước khi tính lại, để lộ trình luôn ổn định và đáng tin, không "nhảy số" âm thầm.

---

## 3. Metrics

### 3.1 Measurement Mechanism
- **Sample:** Toàn bộ người dùng đã hoàn tất onboarding (có dữ liệu mục tiêu/khó khăn/hình thức học), so sánh trước–sau khi widget lộ trình lên production.
- **Method:** Theo dõi cohort theo thời gian (trước/sau launch) qua dữ liệu sử dụng thực tế; chưa cần A/B test cho V1 vì đây là thay thế trải nghiệm Dashboard mặc định, không phải tính năng chọn tham gia.

### 3.2 North Star Metric
- **Tỉ lệ hoàn thành Chặng 1 (đo band xuất phát) trong vòng 7 ngày kể từ khi tài khoản có lộ trình.**
  - Baseline hiện tại: **0%** — tính năng chưa tồn tại, người dùng hiện không có luồng "đo band xuất phát" bắt buộc/gợi ý rõ ràng nào trong 7 ngày đầu.

### 3.3 Success Metrics
- % người dùng mở/tương tác với widget lộ trình trong lần đầu vào Dashboard sau onboarding.
- % người dùng hoàn thành ít nhất 1 task bắt buộc trong chặng đang làm, trong 7 ngày đầu.
- Retention D7/D30 của nhóm có lộ trình so với nhóm trước khi có lộ trình (cùng giai đoạn onboarding).

### 3.4 Other Metrics
- % người dùng đi tiếp Thi thử full IELTS Speaking từ nút trong Chặng 1 (so với vào Thi thử qua đường khác).
- % người dùng hoàn thành đủ task bắt buộc để tự mở khoá chặng kế tiếp (không tính người bỏ giữa chừng).
- Thời gian trung bình từ lúc tạo lộ trình tới lúc hoàn thành Chặng 1.

### 3.5 Guardrail / Maintain Metrics
- Số lượt Thi thử / Luyện tập tổng thể trên toàn app **không giảm** so với trước khi có widget (đảm bảo lộ trình không "hút" người dùng ra khỏi các hoạt động họ vẫn đang làm tốt ngoài lộ trình).
- Tỉ lệ người dùng report/complain về việc "bị kẹt", "không hiểu vì sao chặng bị khoá" giữ ở mức thấp (theo dõi qua kênh hỗ trợ, không có ngưỡng cứng cho V1 — theo dõi định tính).

---

## 4. Requirements

### 4.1 Functional Requirements

#### Feature: Widget Lộ trình 4 chặng trên Dashboard

**Design Resources:**
- Figma UI Design: TBD
- Figma Prototype: TBD
- Copywriting (VN + EN): TBD — hiện chỉ có bản tiếng Việt trong bản demo dựng sẵn

**Nguyên lý cốt lõi:**

| Nguyên lý | Mô tả |
|---|---|
| Một trục duy nhất | Lộ trình chia 4 chặng theo band: mỗi chặng tăng 0.5 band từ band xuất phát tới band mục tiêu. Nếu khoảng cách (band mục tiêu − band xuất phát) ≤ 1.5, các chặng dư ra chuyển thành mục tiêu "giữ ổn định phong độ" thay vì tăng band tiếp. |
| Đo band trước, học sau | Chặng 1 luôn là "đo band xuất phát" — bài Thi thử full IELTS Speaking (trọn 3 Part) là task duy nhất mở khoá Chặng 2. Nếu người dùng đã có bài thi thử trong 30 ngày gần nhất, dùng luôn kết quả đó làm band xuất phát thay vì bắt thi lại. |
| Mở khoá theo nỗ lực | Chặng tiếp theo mở khi người dùng **hoàn thành đủ số lượng task bắt buộc** của chặng hiện tại — không yêu cầu phải đạt band mục tiêu mới được mở. Band mục tiêu chỉ hiển thị để đối chiếu (VD: "Mục tiêu 6.5 · Bạn đang ở 5.5"), không phải điều kiện khoá/mở. |
| Task đếm số lượng | Mỗi task hiển thị dạng tiến độ đếm (VD: 2/5 câu), không phải việc tick-một-lần, và không có ngưỡng band ẩn nào để hoàn thành task. |
| Cá nhân hoá theo khó khăn | Nội dung task trong các chặng sau Chặng 1 ưu tiên theo khó khăn lớn nhất người dùng chọn ở onboarding, theo thứ tự: Bí ý → Phản xạ chậm → Thiếu từ vựng → Thiếu tự tin → Không chắc đúng/sai — mỗi người chỉ nhận 1 nhánh nội dung tương ứng khó khăn của mình. |
| Cá nhân hoá theo hình thức ưa thích | Hình thức học ưa thích ở onboarding quyết định thêm 1 task tuỳ chọn và vị trí ưu tiên của nó trong danh sách việc cần làm. Nếu hình thức ưa thích là "Nói chuyện với AI" (không áp dụng được cho task lộ trình), tự động thay bằng task luyện theo chủ đề. |
| Cường độ theo deadline | Deadline thi không đổi mốc band của từng chặng, chỉ đổi **nhãn thời lượng hiển thị** và **cường độ** (số câu/tần suất mỗi task). Deadline càng gấp, cường độ mỗi task càng cao để kịp tiến độ; band mục tiêu từng chặng không bị hạ để "cho vừa" lịch gấp. |
| Cảnh báo lịch gấp | Khi thời lượng một chặng bị rút xuống dưới 4 tuần do deadline gấp, hiển thị cảnh báo rõ ràng cho người dùng biết lịch đang gấp so với mục tiêu (không âm thầm hạ mốc band hay số câu để che đi việc lịch không khả thi). |
| Đóng băng mặc định | Lộ trình đã sinh giữ nguyên, không tự tính lại. Chỉ tính lại khi người dùng chủ động đổi band mục tiêu hoặc deadline, có hộp thoại xác nhận trước khi tính lại. Chặng đã hoàn thành giữ nguyên kết quả, không bị tính lại. Band xuất phát (đo ở Chặng 1) khoá vĩnh viễn, không đổi sau đó. |
| Không xoá, chỉ thu gọn | Người dùng có thể thu gọn/mở rộng hiển thị lộ trình trên Dashboard, không có tuỳ chọn xoá lộ trình. |

**Phạm vi ngoài (Out of scope cho V1):**
- Không thêm mốc thời hạn "12 tháng" vào onboarding — giữ nguyên các mốc hiện có (1/3/6 tháng/Chưa xác định).
- Không dùng tính năng AI Simulation làm task trong lộ trình.
- Task "Bật thông báo nhắc học" ở Chặng 1 chỉ dừng ở mức xin quyền thông báo từ trình duyệt (demo trải nghiệm) — **chưa gửi thông báo nhắc học thật** trong V1. Task tự ẩn nếu trình duyệt đã từng bị người dùng từ chối quyền.

**Cases:**

| Case Type | Scenario | Expected Behavior |
|-----------|----------|-------------------|
| ✅ Happy | Người dùng mới hoàn tất onboarding, chưa từng thi thử | Lộ trình hiển thị với Chặng 1 "Đo điểm xuất phát" đang active, 3 chặng sau ở trạng thái khoá với mục tiêu band hiển thị mờ/chưa xác định (trừ chặng cuối luôn hiện đúng band mục tiêu đã chọn). |
| ✅ Happy | Người dùng đã có bài thi thử trong 30 ngày gần nhất | Chặng 1 tự động đánh dấu hoàn thành, dùng kết quả đó làm band xuất phát, Chặng 2 chuyển thành active với task cá nhân hoá theo khó khăn + hình thức ưa thích. |
| ✅ Happy | Người dùng hoàn thành đủ task bắt buộc của chặng đang làm | Chặng đang làm chuyển trạng thái hoàn thành, chặng kế tiếp tự mở khoá active. |
| ⚠️ Corner | Khoảng cách band mục tiêu − band xuất phát ≤ 1.5 | Số chặng "tăng band" giảm tương ứng, chặng dư chuyển thành mục tiêu "giữ ổn định" thay vì đặt mốc band cao hơn không thực tế. |
| ⚠️ Corner | Deadline rút ngắn khiến 1 chặng còn dưới 4 tuần | Hiện cảnh báo lịch gấp kèm giải thích, không hạ mốc band/số câu để che giấu. |
| ⚠️ Corner | Người dùng đổi band mục tiêu hoặc deadline sau khi lộ trình đã sinh | Hiện hộp thoại xác nhận trước khi tính lại; nếu xác nhận, chặng chưa hoàn thành được tính lại, chặng đã hoàn thành và band xuất phát giữ nguyên. |
| ⚠️ Corner | Trình duyệt đã từng chặn quyền thông báo | Task "Bật thông báo" tự ẩn khỏi danh sách việc cần làm, không hiện task chết không bấm được. |
| ⚠️ Corner | Hình thức ưa thích onboarding là "Nói chuyện với AI" | Task tuỳ chọn cá nhân hoá tự động fallback sang luyện theo chủ đề, không hiển thị task rỗng/không áp dụng được. |
| ⚠️ Corner | Người dùng chưa hoàn tất onboarding hoặc đã bấm "Bỏ qua" | Không hiển thị widget lộ trình cho tới khi có đủ dữ liệu mục tiêu band + deadline (áp theo luồng hỏi lại band hiện có của Dashboard). |

##### Acceptance Criteria

**AC-1: Sinh lộ trình lần đầu sau onboarding**
- **Given:** Người dùng vừa hoàn tất onboarding với đầy đủ mục tiêu band, deadline, khó khăn lớn nhất, hình thức ưa thích AND chưa có lộ trình nào được tạo trước đó
- **When:** Người dùng vào Dashboard lần đầu
- **Then:** Widget lộ trình 4 chặng hiển thị với Chặng 1 active là "Đo điểm xuất phát" AND các chặng 2–4 hiển thị khoá kèm mục tiêu band tương ứng (trừ mục tiêu của chặng cuối luôn bằng band mục tiêu đã chọn)

**AC-2: Tái sử dụng bài thi thử gần đây làm band xuất phát**
- **Given:** Người dùng có bài Thi thử full IELTS Speaking đã hoàn thành trong 30 ngày gần nhất AND chưa có lộ trình được tạo
- **When:** Lộ trình được sinh
- **Then:** Chặng 1 tự động đánh dấu hoàn thành với band xuất phát lấy từ bài thi thử đó AND Chặng 2 chuyển active AND band xuất phát bị khoá không đổi

**AC-3: Mở khoá chặng theo nỗ lực, không theo band**
- **Given:** Người dùng đang ở một chặng active AND đã hoàn thành đủ số lượng tất cả task bắt buộc của chặng đó
- **When:** Task bắt buộc cuối cùng được hoàn thành
- **Then:** Chặng hiện tại chuyển trạng thái hoàn thành AND chặng kế tiếp mở khoá active AND việc mở khoá không bị chặn dù band thực tế đo được thấp hơn mục tiêu của chặng vừa hoàn thành

**AC-4: Cảnh báo lịch gấp**
- **Given:** Deadline người dùng chọn khiến thời lượng của ít nhất 1 chặng còn dưới 4 tuần
- **When:** Lộ trình được sinh hoặc tính lại
- **Then:** Hệ thống hiển thị cảnh báo nêu rõ lịch đang gấp so với mục tiêu AND mốc band cùng số lượng task của các chặng không bị hạ thấp để che đi sự gấp gáp

**AC-5: Đổi mục tiêu/deadline sau khi đã có lộ trình**
- **Given:** Người dùng đã có lộ trình đang chạy với ít nhất 1 chặng đã hoàn thành
- **When:** Người dùng đổi band mục tiêu hoặc deadline
- **Then:** Hệ thống hiện hộp thoại xác nhận trước khi tính lại AND nếu người dùng xác nhận, chỉ các chặng chưa hoàn thành được tính lại AND chặng đã hoàn thành cùng band xuất phát giữ nguyên không đổi

**AC-6: Task thông báo tự ẩn khi bị chặn quyền**
- **Given:** Trình duyệt của người dùng đã từng từ chối cấp quyền thông báo
- **When:** Người dùng xem danh sách task của Chặng 1
- **Then:** Task "Bật thông báo nhắc học" không hiển thị trong danh sách việc cần làm

### 4.2 Non-Functional Requirements

#### Performance
- Widget lộ trình phải hiển thị cùng lúc với phần còn lại của Dashboard, không tạo độ trễ tải trang riêng biệt cảm nhận được.

#### Compatibility
- Hoạt động trên các trình duyệt/thiết bị Dashboard hiện đang hỗ trợ (không có yêu cầu mở rộng riêng cho tính năng này).

#### Accessibility
- Trạng thái chặng (đang làm / đã xong / đã khoá) phải phân biệt được không chỉ bằng màu sắc (kèm nhãn chữ/icon), để người dùng khó phân biệt màu vẫn đọc được lộ trình.

---

## 5. User Types

| User Type | Định nghĩa | Hành vi widget |
|-----------|-----------|-----------------|
| Chưa đo band xuất phát | Đã hoàn tất onboarding nhưng chưa có bài Thi thử full nào (hoặc bài gần nhất đã quá 30 ngày) | Chặng 1 active yêu cầu Thi thử full; các chặng sau hiển thị khoá, mục tiêu band mờ |
| Đã đo band xuất phát | Có bài Thi thử full trong 30 ngày, band xuất phát đã xác định | Chặng 1 hiển thị hoàn thành với band xuất phát; chặng tiếp theo active với task cá nhân hoá theo khó khăn + hình thức ưa thích |
| Khoảng cách mục tiêu ngắn (≤1.5 band) | Band mục tiêu − band xuất phát ≤ 1.5 | Số chặng "tăng band" ít hơn 3, chặng dư chuyển thành mục tiêu giữ ổn định |
| Lịch thi gấp | Deadline khiến ít nhất 1 chặng dưới 4 tuần | Hiện cảnh báo lịch gấp, cường độ task tăng, mốc band không đổi |
| Chưa hoàn tất onboarding / đã bỏ qua | Chưa có đủ mục tiêu band + deadline | Không hiển thị widget lộ trình cho tới khi có đủ dữ liệu |

---

## 6. Analytics

> Instrumentation (event/property) được quản lý ở tài liệu tracking plan riêng, không lặp lại trong PRD này.

### 6.2 Dashboard

| Dashboard | Metrics | Priority |
|-----------|---------|----------|
| Roadmap Activation & Retention | Tỉ lệ hoàn thành Chặng 1 trong 7 ngày, % mở widget lần đầu, retention D7/D30 theo nhóm có/không có lộ trình | P0 |
| Roadmap Funnel | Tỉ lệ chuyển từ chặng này sang chặng kế theo thời gian, điểm rơi (chặng nào rớt nhiều nhất) | P1 |

---

## 7. Timelines

| Milestone | Target Date | Status |
|-----------|------------|--------|
| Chốt PRD | 2026-08-21 | ✅ Xong (tài liệu này) |
| Design (Figma) | TBD | ⬜ Chưa bắt đầu |
| Development & Testing Handoff | TBD | ⬜ Chưa bắt đầu |
| Review | TBD | ⬜ Chưa bắt đầu |
| Release | TBD | ⬜ Chưa bắt đầu |

---

## 8. Dependencies / Approvals

### 8.1 Internal
| Dependency | Owner | Status |
|-----------|-------|--------|
| Xác nhận nội dung task theo từng nhánh khó khăn (5 biến thể) | Đội nội dung/Arthur | ⬜ Pending |
| Bản dịch tiếng Anh cho toàn bộ copy lộ trình | TBD | ⬜ Pending |

### 8.2 External
| Dependency | Platform | Status |
|-----------|----------|--------|
| Hạ tầng gửi thông báo đẩy thật (để nâng task "Bật thông báo" từ demo lên chức năng thật) | Web push | ⬜ Chưa có kế hoạch — ngoài phạm vi V1 |

---

## 9. Appendix

| Document | Link | Description |
|----------|------|--------------|
| Bản demo trải nghiệm | Prototype nội bộ (2 trạng thái: chưa đo band / đã đo band 6.0) | Dùng để review luồng UI trước khi vào Figma chính thức |
| Glossary | — | **Chặng**: một giai đoạn trong lộ trình 4 phần. **Band xuất phát**: điểm IELTS Speaking đo được lần đầu qua Thi thử full. **Task bắt buộc**: việc phải hoàn thành đủ số lượng để mở khoá chặng kế tiếp. **Task tuỳ chọn**: việc không bắt buộc, không ảnh hưởng mở khoá. |
