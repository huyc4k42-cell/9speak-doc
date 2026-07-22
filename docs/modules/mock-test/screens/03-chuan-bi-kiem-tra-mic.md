# Mock Test V2 — §3 Chuẩn bị & Kiểm tra Mic — v0.3 (đối chiếu code thật + chốt qua grill-me)

> Ngày cập nhật: 2026-07-07 · Trạng thái: Chính thức · Đã triển khai (đối chiếu [`useMicTest.ts`](../../../../features/mock-test/hooks/useMicTest.ts) + [`MockExamFull.tsx`](../../../../features/mock-test/pages/exams/MockExamFull.tsx)) · Thay thế toàn bộ nội dung bản v0.2 (Google Doc, 2026-06-24).

## Đổi log so với v0.2

| # | Thay đổi | Lý do |
|---|---|---|
| 1 | Thay mô hình 2 trạng thái lỗi mơ hồ ("mic bị từ chối/không có thiết bị") bằng **3 trạng thái lỗi riêng biệt** với copy/CTA khác nhau: `blocked` (quyền bị từ chối), `no-device` (không tìm thấy mic), `hardware-error` (mic đang bị app khác chiếm dụng) | Đối chiếu code thật (`useMicTest.ts`) — 3 lỗi có nguyên nhân và cách khắc phục khác nhau, gộp chung sẽ hướng dẫn sai cho user |
| 2 | Thêm trạng thái **`failed`** riêng cho "test hết 10 giây mà chưa nói đủ" (khác với lỗi quyền/thiết bị) — đây chính là câu trả lời cho câu hỏi mở cũ "mic OK nhưng không có tín hiệu" | Mockup cũ luôn thành công nên tài liệu chưa mô tả nhánh này; code thật đã có sẵn |
| 3 | Xác nhận qua grill-me: nếu quyền mic trình duyệt đã ở trạng thái "granted" từ trước (phiên trước, hoặc từ Practice — quyền theo domain, không theo phiên), màn chuẩn bị **tự động vào `success`, không bắt test giọng lại** | Quyết định UX có chủ đích: giảm ma sát cho user quay lại thi nhiều đề trong ngày; đảo ngược mô tả BR-03 cũ ("mỗi lần vào đều reset về idle") — mô tả cũ chưa từng khớp code |
| 4 | Bỏ câu hỏi mở "Timeout xin quyền" — xác nhận **không có timeout riêng cho popup xin quyền** (trình duyệt tự xử lý); chỉ có timeout **10 giây cho bài test giọng nói** (`maxRecordingSeconds`, mặc định 10s) | Đối chiếu code: `getUserMedia` chỉ resolve/reject theo lựa chọn của user trên popup, không có app-level timeout |
| 5 | Xác nhận qua grill-me: **có** hiển thị mức âm thanh trực quan (sóng phản ứng theo giọng nói thật, dùng chung engine với phòng thi) + thanh tiến độ theo thời lượng đã nói đủ — không phải số dB | Trả lời câu hỏi mở cũ "có VU meter không" |

---

## Goal

**Business/Product Goal:** Đảm bảo user có mic hoạt động thật trước khi vào thi — giảm rủi ro thi xong mới phát hiện không ghi được (mất bài, không chấm được, phải liên hệ hỗ trợ). Đồng thời giảm ma sát cho user quay lại thi nhiều đề trong cùng ngày (không bắt test lại nếu quyền đã cấp).

**User Benefits:**
- User lần đầu: được hướng dẫn rõ ràng từng loại sự cố (quyền/thiết bị/chiếm dụng/không nói đủ) kèm cách khắc phục cụ thể, thay vì 1 thông báo lỗi chung chung.
- User quay lại (đã cấp quyền mic trước đó): vào thẳng "sẵn sàng" ngay, không phải lặp lại bước nói thử mỗi lần chọn đề mới.

---

## 3.1 Tổng quan

**Mô tả ngắn:** Màn trước khi vào phòng thi: giới thiệu bài thi, checklist, và **kiểm tra micro** bắt buộc trước khi cho phép bắt đầu.

**Mục đích / vấn đề giải quyết:** Giảm rủi ro thi xong mới phát hiện mic hỏng (mất bài, không chấm được). Test mic trước đảm bảo điều kiện ghi âm hợp lệ và tạo tâm thế sẵn sàng.

**Phạm vi:**
- Trong phạm vi: card giới thiệu "Thi Thử IELTS Speaking", checklist, nút Kiểm tra Mic, 5 trạng thái mic thật (idle/requesting/recording/success + 4 nhánh kết quả), mở khoá nút "Bắt đầu thi", chặn vào thi khi mic lỗi, tùy biến nội dung theo chế độ thi, auto-skip khi quyền đã cấp.
- Ngoài phạm vi: chấm điểm, nội dung từng Part (#04), cấu hình thiết bị hệ điều hành.

**Đối tượng người dùng & bên liên quan:** Học viên; Dev FE (`getUserMedia`/quyền mic); QA.

---

## 3.2 Yêu cầu chức năng

| Mã | Yêu cầu | Đầu vào | Đầu ra | Ghi chú |
|---|---|---|---|---|
| FR-01 | Hiển thị card giới thiệu + checklist trước khi thi | Vào màn pre | Card giới thiệu | |
| FR-02 | Cho người dùng **Kiểm tra Mic** (nút, bắt đầu ghi thử) | Bấm "Kiểm tra Mic" | Bắt đầu vòng test | |
| FR-03 | Thể hiện đúng vòng trạng thái: `idle` → `requesting` → `recording` → (`success` \| `failed` \| `blocked` \| `no-device` \| `hardware-error`) | Sự kiện mic thật (permission + audio signal) | UI trạng thái tương ứng | 5 kết quả cuối, không phải nhị phân |
| FR-04 | **Disabled** nút "Bắt đầu thi" cho tới khi mic = `success` | Trạng thái mic | Nút khoá/mở | |
| FR-05 | Khi quyền mic bị từ chối (`blocked`), hiển thị **"Quyền truy cập micro bị từ chối"** + CTA "Thử lại", **chặn** vào thi | Lỗi quyền (NotAllowedError...) | Card đỏ + "Thử lại" | Tự phục hồi khi user cấp quyền qua Settings (poll `permissions.query`/focus/visibility) |
| FR-06 | Khi không tìm thấy thiết bị mic (`no-device`), hiển thị **"Không tìm thấy microphone"** + "Thử lại", **chặn** vào thi | NotFoundError/DevicesNotFoundError | Card amber + "Thử lại" | |
| FR-07 | Khi mic đang bị ứng dụng khác chiếm dụng (`hardware-error`), hiển thị **"Microphone đang bị chiếm dụng"** + "Thử lại", **chặn** vào thi | NotReadableError/TrackStartError | Card cam + "Thử lại" | Gợi ý đóng Zoom/Meet |
| FR-08 | Khi hết 10 giây test mà chưa nói đủ (`failed`), hiển thị **"Chưa nhận diện được âm thanh"** + nút "Kiểm tra Mic" để thử lại | Timeout 10s, chưa đạt ngưỡng nói | Card đỏ + retry | Ngưỡng: nói ≥700ms (mức âm > baseline) rồi im ≥250ms mới tính "ổn định"; chưa đạt khi hết 10s → failed |
| FR-09 | Khi mic `success`, cho phép **Bắt đầu thi** → vào phần đầu của chế độ đang chọn | Bấm "Bắt đầu thi" | Vào Part 1 (full) hoặc Part X (chế độ lẻ) | |
| FR-10 | Màn chuẩn bị phải **tùy biến theo chế độ thi** (full / Part 1 / Part 2 / Part 3): tiêu đề, mô tả, checklist, thẻ phần, nhãn nút bắt đầu | `examMode` | Nội dung theo chế độ | |
| FR-11 | Nếu quyền mic của trình duyệt **đã "granted" từ trước** (không cần theo phiên hiện tại), tự động chuyển thẳng `idle` → `success` khi vào màn, **không bắt test giọng lại** | `navigator.permissions` state = granted | Vào thẳng `success`, nút "Bắt đầu thi" mở khoá ngay | Quyết định đã chốt qua grill-me (2026-07-07) |

**Nội dung màn chuẩn bị theo chế độ (FR-10):**

| Chế độ | Tiêu đề | Thẻ phần | Checklist riêng | Nút bắt đầu |
|---|---|---|---|---|
| Full | Thi thử IELTS Speaking | 3 thẻ Part 1·2·3 | Sẵn sàng ~15 phút | Sẵn sàng bắt đầu |
| Part 1 | Luyện Part 1 · Phỏng vấn | 1 thẻ Part 1 | ~5 phút Part 1 | Bắt đầu Part 1 |
| Part 2 | Luyện Part 2 · Cue card | 1 thẻ Part 2 | Chuẩn bị giấy/bút (1' chuẩn bị) | Bắt đầu Part 2 |
| Part 3 | Luyện Part 3 · Thảo luận | 1 thẻ Part 3 | ~5 phút Part 3 | Bắt đầu Part 3 |

**Quy tắc nghiệp vụ (Business rules):**
- **BR-01:** Không cho bắt đầu thi nếu chưa test mic thành công (`status !== "success"` → CTA disabled).
- **BR-02:** Test mic là **test thật** (không chỉ kiểm tra permission): cần phát hiện mức âm thanh vượt ngưỡng nói liên tục ≥700ms rồi có khoảng lặng ≥250ms mới chốt "ổn định". Test tự dừng sau **10 giây** nếu chưa đạt → `failed`. Không chấm điểm, không lưu bản ghi thử.
- **BR-03:** **Chế độ thi (`examMode`)** do scope đã chọn ở màn Chọn đề quyết định (#01 — tab bar/card CTA). 2 mục checklist chung (mic đã kiểm tra · không gian yên tĩnh) + 1 mục riêng theo chế độ.
- **BR-04 (chốt qua grill-me):** Nếu quyền mic trình duyệt đã "granted" (persist theo domain, không theo phiên/session — có thể từ lần thi trước hoặc từ Practice), màn chuẩn bị **tự động vào `success` ngay khi mount, không yêu cầu test giọng lại**. Đánh đổi đã cân nhắc: chấp nhận rủi ro hiếm gặp "mic vừa hỏng/rút ra nhưng permission vẫn granted" để đổi lấy trải nghiệm nhanh cho user thi nhiều đề trong ngày.
- **BR-05:** 3 loại lỗi thiết bị/quyền (`blocked`/`no-device`/`hardware-error`) phải phân biệt rõ bằng copy + màu sắc riêng (đỏ/amber/cam) — không gộp chung thành 1 "lỗi mic".

**Phân quyền theo vai trò:**

| Vai trò | Được làm gì |
|---|---|
| Mọi user đã vào được phòng thi | Test mic và bắt đầu thi |

---

## 3.3 Cơ chế hoạt động

**Luồng người dùng (workflow):**
1. Từ chọn đề → vào màn chuẩn bị.
2. Nếu quyền mic đã "granted" từ trước → tự động vào `success` (bỏ qua bước 3-4, xem FR-11/BR-04).
3. Ngược lại: đọc checklist → bấm "Kiểm tra Mic" → xin quyền (`requesting`).
4. Cấp quyền → ghi thử (`recording`, tối đa 10s) → nói đủ ngưỡng + im lặng ngắn → `success`; hoặc hết 10s chưa đủ → `failed`.
5. Nút "Bắt đầu thi" mở khoá khi `success` → bấm → vào Part 1 (hoặc Part đang chọn theo chế độ).

**Use case chính:**
- **Bối cảnh A (lần đầu trong ngày):** user chưa từng cấp quyền mic → bấm "Kiểm tra Mic" → cấp quyền → nói thử → `success` → bắt đầu thi.
- **Bối cảnh B (quay lại thi đề khác):** quyền mic đã "granted" từ lượt trước → vào màn là thấy ngay "Microphone hoạt động ổn định", không cần thao tác gì → bắt đầu thi luôn.
- **Kịch bản thay thế / ngoại lệ:** từ chối quyền → `blocked`; rút mic ra → `no-device`; mic đang mở ở Zoom/Meet → `hardware-error`; nói không đủ trong 10s → `failed`. Cả 4 nhánh đều **không** vào được thi, đều có CTA "Thử lại".

**Luồng dữ liệu:** Yêu cầu quyền mic (`getUserMedia`) → phân tích tần số qua `AnalyserNode` (throttle ~16fps) → tính mức âm trung bình mỗi frame, so ngưỡng → cộng dồn thời lượng "có giọng" / "im lặng sau khi đã nói đủ" → cập nhật `micStatus` + `progress` (0→1) + `levels` (sóng trực quan). Không lưu bản ghi thử lên server hay client.

**Tích hợp:** Web Audio API (`getUserMedia`, `AudioContext`, `AnalyserNode`) · Permissions API (`navigator.permissions.query`, có fallback probe cho Safari không hỗ trợ) · Phòng thi (#04).

*Sơ đồ phù hợp: state diagram `idle → requesting → recording → {success | failed | blocked | no-device | hardware-error}`, có cạnh tắt `idle → success` khi permission đã granted.*

---

## 3.4 Xử lý ngoại lệ và trạng thái

| Trường hợp | Hành vi mong đợi / thông báo |
|---|---|
| Quyền mic bị từ chối (`blocked`) | Card đỏ "Quyền truy cập micro bị từ chối" + "Cấp quyền trong cài đặt trình duyệt rồi thử lại" + nút "Thử lại"; chặn thi; tự chuyển về `idle` khi phát hiện quyền được cấp lại (không cần reload) |
| Không có thiết bị mic (`no-device`) | Card amber "Không tìm thấy microphone" + "Cắm tai nghe hoặc microphone rồi thử lại" + "Thử lại"; chặn thi |
| Mic đang bị chiếm dụng (`hardware-error`) | Card cam "Microphone đang bị chiếm dụng" + "Đóng Zoom, Meet hoặc app dùng mic rồi thử lại" + "Thử lại"; chặn thi |
| Đang xin quyền (`requesting`) | Trạng thái "Đang yêu cầu quyền truy cập mic…" |
| Test hết 10s chưa nói đủ (`failed`) | Card đỏ "Chưa nhận diện được âm thanh" + "Để lại 1 đoạn thoại rõ ràng và thử lại ngay" + nút "Kiểm tra Mic" |
| Quyền đã granted từ trước | Bỏ qua bước ghi thử, vào thẳng `success` — "Microphone hoạt động ổn định" |

**Các trạng thái giao diện:** `idle` / `requesting` (loading) / `recording` (sóng + progress chạy) / `success` / `failed` / `blocked` / `no-device` / `hardware-error`.

---

## 3.5 Yêu cầu giao diện (UI/UX)

- Card tối (focus mode): tiêu đề, checklist, khối test mic (đổi UI theo 1 trong 8 trạng thái ở trên), nút bắt đầu (xám khi khoá → xanh khi mở).
- Khối test mic hiển thị **sóng trực quan phản ứng theo giọng nói thật** (dùng chung engine tính sóng với phòng thi — mọi thanh nhảy theo âm lượng, im lặng → phẳng) + **thanh tiến độ** chạy theo thời lượng đã nói đủ (đầy 100% khi đạt ngưỡng 700ms).
- Mỗi trạng thái lỗi có icon + màu riêng: `blocked`/`failed` = đỏ, `no-device` = amber, `hardware-error` = cam — không dùng chung 1 icon "lỗi mic".
- **Tùy biến theo chế độ:** đọc `examMode` → đổi tiêu đề/mô tả/checklist/thẻ phần/nhãn nút bắt đầu; chế độ Part lẻ chỉ hiện 1 thẻ phần.

---

## 3.6 Tiêu chí chấp nhận

**AC-01: Chưa cấp quyền mic**
- **Given:** Quyền mic chưa từng được cấp (permission != granted)
- **When:** User vào màn chuẩn bị
- **Then:** Nút "Bắt đầu thi" ở trạng thái disabled AND trạng thái mic là `idle`

**AC-02: Test mic thành công**
- **Given:** User đã bấm "Kiểm tra Mic" và cấp quyền mic
- **When:** User nói liên tục ≥700ms rồi ngừng ~250ms
- **Then:** Trạng thái mic chuyển `recording` → `success`

**AC-03: Vào thi sau khi mic success**
- **Given:** Trạng thái mic là `success`
- **When:** User bấm nút "Bắt đầu thi"
- **Then:** Hệ thống điều hướng vào Part 1 (hoặc Part đang chọn theo chế độ)

**AC-04: Quyền mic bị từ chối**
- **Given:** User bấm "Kiểm tra Mic"
- **When:** Quyền mic bị từ chối (permission denied)
- **Then:** Hiển thị card "Quyền truy cập micro bị từ chối" + nút "Thử lại" AND không cho vào thi

**AC-05: Không có thiết bị mic**
- **Given:** User bấm "Kiểm tra Mic"
- **When:** Trình duyệt không tìm thấy thiết bị mic nào
- **Then:** Hiển thị card "Không tìm thấy microphone" + nút "Thử lại" AND không cho vào thi

**AC-06: Mic bị ứng dụng khác chiếm dụng**
- **Given:** User bấm "Kiểm tra Mic"
- **When:** Mic đang được một ứng dụng khác (Zoom/Meet...) sử dụng, không thể truy cập
- **Then:** Hiển thị card "Microphone đang bị chiếm dụng" + nút "Thử lại" AND không cho vào thi

**AC-07: Test hết thời gian mà chưa nói đủ**
- **Given:** User đang trong quá trình test mic (`recording`)
- **When:** Test chạy hết 10 giây mà chưa đạt ngưỡng nói (≥700ms liên tục)
- **Then:** Trạng thái chuyển `failed` với thông báo "Chưa nhận diện được âm thanh" AND cho phép thử lại

**AC-08: Đang xin quyền**
- **Given:** User vừa bấm "Kiểm tra Mic" lần đầu (permission chưa được quyết định)
- **When:** Trình duyệt đang chờ user phản hồi popup xin quyền
- **Then:** Hiển thị trạng thái loading rõ ràng ("Đang yêu cầu quyền truy cập mic…")

**AC-09: Quyền đã cấp từ trước — auto-skip**
- **Given:** Quyền mic của trình duyệt đã ở trạng thái "granted" từ trước (phiên trước hoặc app khác cùng domain)
- **When:** User vào màn chuẩn bị
- **Then:** Hệ thống tự động hiển thị `success` ngay AND không yêu cầu bấm "Kiểm tra Mic" hay nói thử lại

**AC-10: Nội dung tùy biến theo chế độ Part lẻ**
- **Given:** User vào màn chuẩn bị ở chế độ Part X (không phải full)
- **When:** Màn chuẩn bị render
- **Then:** Tiêu đề/mô tả/thẻ phần/nút bắt đầu hiển thị đúng theo Part đó (Part 2 có thêm mục checklist "chuẩn bị giấy/bút") AND chế độ full hiển thị đủ 3 thẻ Part

---

## Non-functional Requirements

**Hiệu năng:** cập nhật sóng/tiến độ throttle ~16fps (60ms/frame) để tránh re-render dày; test tối đa 10 giây trước khi tự kết luận.

**Bảo mật/Quyền riêng tư:** không lưu/đẩy bản ghi thử lên server hay lưu trữ client; chỉ dùng `AnalyserNode` để đo mức âm tức thời, không giữ audio buffer.

**Khả năng phục hồi:** trạng thái `blocked` tự dò quyền được cấp lại qua `Permissions API` (Chrome/Edge/Firefox) hoặc probe `getUserMedia` khi tab quay lại visible/focus (Safari, không hỗ trợ Permissions API cho microphone) — không cần reload trang.

---

## User Types

Màn này không phân biệt theo gói (free/premium) hay lịch sử thi — mọi user đã vào tới bước chuẩn bị đều theo cùng 1 luồng test mic. Điểm khác biệt duy nhất là **quyền mic đã cấp hay chưa** (ảnh hưởng có bỏ qua bước test hay không — xem FR-11/BR-04), không phải một "loại user" theo nghĩa sản phẩm.

---

## 3.7 Phần bổ sung

**Giả định (Assumptions):** Trình duyệt hỗ trợ `getUserMedia` + `AudioContext`/`AnalyserNode`; quyền mic là theo domain (persist qua nhiều phiên), không phải theo từng lượt thi.

**Phụ thuộc (Dependencies):** Web Audio API + Permissions API của trình duyệt · Phòng thi (#04).

---

## 3.8 Câu hỏi mở — đã chốt (2026-07-07, grill-me)

| Câu hỏi | Đã chốt tại |
|---|---|
| Mockup luôn thành công — cần trạng thái failed/no-device thực tế? | Đã có sẵn trong code: 4 trạng thái kết quả lỗi riêng biệt (`blocked`/`no-device`/`hardware-error`/`failed`) — §3.2 FR-05→08 |
| Có hiển thị VU meter khi test không? | Có — sóng phản ứng theo giọng nói thật + thanh tiến độ theo thời lượng nói đủ (không phải số dB) — §3.5 |
| Có nhớ kết quả test trong phiên để không phải test lại mỗi lần? | Có, nhưng ở mức quyền trình duyệt (persist theo domain): quyền đã granted → tự vào `success`, không bắt nói lại — chốt giữ nguyên hành vi này (đánh đổi ma sát thấp hơn cho rủi ro hiếm) — FR-11/BR-04 |
| "Mic OK nhưng không có tín hiệu" / "Timeout xin quyền" cần chốt hành vi gì? | Không có timeout riêng cho popup xin quyền (trình duyệt xử lý); chỉ có timeout 10s cho bài test giọng nói → hết hạn mà chưa đủ ngưỡng nói thì vào `failed` — FR-08 |

Không còn câu hỏi mở nào cho §3 tại thời điểm này.
