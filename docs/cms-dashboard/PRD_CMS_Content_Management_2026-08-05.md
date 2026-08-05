# PRD: 9Speak CMS — Quản lý & Thêm đề (Content Management)

| Field | Value |
|---|---|
| **Version** | 1.0 (Draft) |
| **Date** | 2026-08-05 |
| **Author** | Product Team |
| **Audience** | Product, Content Team, Stakeholders |
| **Status** | Draft — Pending Review |
| **Quan hệ tài liệu** | **Tách biệt** khỏi PRD CMS Dashboard v2 (dashboard + quản lý user). Bám card đề Thi thử ở [`01-chon-de-goi-y.md`](../modules/mock-test/screens/01-chon-de-goi-y.md) §1.3. |

> **Phạm vi tài liệu:** PRD này mô tả **nhu cầu nghiệp vụ và hành vi sản phẩm** mong muốn. **Không** bàn giải pháp kỹ thuật, kiến trúc, hay tính khả thi triển khai — những phần đó do đội kỹ thuật đánh giá trong tài liệu riêng (xem [§8](#8-dependencies--approvals)).
>
> Nguồn quyết định: 1 phiên grill-me (2026-08-05, 16 câu) — xem [Decision Summary](#91-grill-me-decision-summary).

---

## Mục lục

1. [Goal](#1-goal)
2. [Context](#2-context)
3. [Metrics](#3-metrics)
4. [Requirements](#4-requirements)
5. [User Types](#5-user-types)
6. [Analytics](#6-analytics)
7. [Timelines](#7-timelines)
8. [Dependencies & Approvals](#8-dependencies--approvals)
9. [Appendix](#9-appendix)

---

## 1. Goal

### 1.1 Business / Product Goal

- **Trao quyền tự chủ nội dung cho Content Team**: tự thêm/sửa/phát hành đề Thi thử và câu hỏi Luyện tập, **không phải nhờ đội kỹ thuật** cho từng thay đổi — rút ngắn thời gian từ "soạn xong" đến "học viên thấy" từ nhiều ngày xuống trong ngày.
- **Tăng tốc mở rộng kho đề**: gỡ nút cổ chai vận hành để tăng số đề/chủ đề phát hành mỗi tháng, nuôi discovery ("Đề gợi ý", tag New/Hot) và tạo lý do quay lại cho học viên.
- **Giảm rủi ro lỗi nội dung ra sản phẩm**: thay quy trình sửa nội dung thủ công (dễ sai cấu trúc đề, phá trải nghiệm phòng thi) bằng công cụ có kiểm tra hợp lệ + trạng thái Nháp/Phát hành.

### 1.2 User Benefits (Nội bộ — Content Team)

- Tạo trọn 1 đề Thi thử (đủ 3 Part) qua giao diện, không cần chạm tới hệ thống kỹ thuật.
- Soạn dở được lưu (Nháp) và xem trước trước khi phát hành — không sợ đề lỗi lọt ra học viên.
- Sửa đề/câu đã phát hành (sửa lỗi chữ, đổi khoá Premium) ngay lập tức, không chờ lịch của đội kỹ thuật.
- Quản lý câu hỏi tập trung: sửa 1 câu ở ngân hàng → đúng ở mọi đề/chủ đề đang dùng câu đó.

### 1.3 Benefit gián tiếp cho học viên

- Kho đề/chủ đề mới nhiều hơn, cập nhật nhanh hơn → nội dung "sống", tag New phản ánh đúng kỳ.
- Ít lỗi nội dung hơn nhờ kiểm tra hợp lệ ở khâu phát hành.

---

## 2. Context

### 2.1 Vấn đề hiện tại

- **Content Team không tự chủ được nội dung.** Mọi thay đổi đề Thi thử / câu hỏi Luyện tập hiện đều **phải nhờ đội kỹ thuật** thực hiện. Việc này chậm (chờ theo lịch phát triển), tốn nguồn lực kỹ thuật cho công việc lẽ ra thuộc content, và khiến kho đề khó mở rộng đều đặn.
- **Không có nơi soạn nháp an toàn.** Hiện nội dung gần như chỉ ở 2 trạng thái: "chưa có" hoặc "đã ra sản phẩm". Thiếu vùng đệm để soạn dở, review, rồi mới phát hành.
- **Dễ sai cấu trúc.** Đề Thi thử có cấu trúc bắt buộc (Part 1 gồm 5 câu, Part 2 gồm 1 cue card, Part 3 gồm 3 câu). Chỉnh sửa thủ công không được kiểm tra hợp lệ → rủi ro đề sai lọt ra học viên.
- **Đã có tín hiệu nhu cầu content.** Màn Tìm kiếm của Luyện tập ghi nhận các lượt "không có kết quả" như đầu vào để content team bổ sung câu hỏi — tức nhu cầu "thêm nội dung nhanh" là có thật, nhưng chưa có công cụ để hành động.

### 2.2 Hướng đi

- Xây **khu vực Quản lý nội dung trong CMS** với **2 vùng tách biệt**, luồng soạn thảo độc lập: *Quản lý đề Thi thử* và *Quản lý câu hỏi Luyện tập*.
- Áp mô hình **Nháp → Phát hành** kèm **kiểm tra hợp lệ** để bảo toàn tính đúng đắn của đề trước khi ra học viên.
- Đặt Content Team làm người dùng chính, với quyền soạn/phát hành nội dung tách khỏi quyền quản trị người dùng/thanh toán.

---

## 3. Metrics

> **Lưu ý baseline:** Đây là công cụ nội bộ mới, **chưa có baseline định lượng**. Giai đoạn đầu (30–60 ngày) dùng để thiết lập baseline; chưa đặt target cứng.

### 3.1 Measurement Mechanism

- **Sample:** Toàn bộ Content Editor + admin dùng CMS (nhóm nhỏ, nội bộ).
- **Method:** Đối chiếu vận hành trước/sau (số đề phát hành/tháng, thời gian trung bình soạn→phát hành) + ghi nhận hành vi thao tác trong CMS (§6).

### 3.2 North Star Metric

**Số nội dung Content Team tự phát hành thành công qua CMS mỗi tháng** (không cần đội kỹ thuật).
= số đề Thi thử phát hành + số chủ đề Luyện tập phát hành, do Content Editor thực hiện, trong kỳ.

- **Ý nghĩa:** đo trực tiếp mục tiêu "tự chủ nội dung". Bằng 0 nghĩa là feature thất bại dù đã có công cụ.
- **Baseline:** 0 (hiện 100% qua đội kỹ thuật). → Target đầu: **> 0 và tăng dần**.

### 3.3 Success Metrics

| Metric | Định nghĩa | Baseline | Target |
|---|---|---|---|
| Tỷ lệ nội dung tự phát hành | Nội dung phát hành qua CMS / tổng nội dung mới trong kỳ | 0% | Đa số nội dung mới đi qua CMS |
| Thời gian soạn → phát hành (median) | Từ lúc tạo Nháp đến lúc Phát hành 1 đề Thi thử | Hiện: nhiều ngày (chờ đội kỹ thuật) | Trong ngày |
| Tỷ lệ phát hành hợp lệ ngay lần đầu | Phát hành thành công / số lần bấm Phát hành | TBD | Cao (kiểm tra hợp lệ bắt lỗi sớm ở Nháp) |

### 3.4 Other Metrics

- Số câu hỏi mới thêm vào ngân hàng / kỳ.
- Số đề/chủ đề được **sửa** (không chỉ tạo mới) / kỳ — chứng tỏ CMS thay được việc chỉnh sửa thủ công.
- Số đề nằm ở trạng thái Nháp quá lâu (>14 ngày) — tín hiệu ma sát soạn thảo.
- Số lần phát hành bị chặn, phân theo loại lỗi (thiếu câu / sai cấu trúc / thiếu trường bắt buộc).

### 3.5 Guardrail Metrics

- **Không đề sai cấu trúc ra sản phẩm:** 0 đề Thi thử phát hành sai cấu trúc 5+1+3.
- **Không mất kết quả cũ:** 0 kết quả bài thi/lịch sử của học viên bị ảnh hưởng khi sửa/xóa đề.
- **Không lộ nội dung Nháp:** 0 lần đề Nháp/đã ẩn xuất hiện với học viên.

---

## 4. Requirements

### 4.1 Functional Requirements

Khu vực Quản lý nội dung gồm **2 vùng độc lập** — **F1. Quản lý đề Thi thử** và **F2. Quản lý câu hỏi Luyện tập** — dùng chung **F3. Ngân hàng câu hỏi** và **F4. Phân quyền**.

---

#### F1. Quản lý đề Thi thử

##### F1.1 Danh sách đề

| Mã | Yêu cầu |
|---|---|
| FR-M01 | Hiển thị danh sách đề: số đề, tên chủ đề, khoá (Free/Premium), trạng thái (Nháp / Đã phát hành / Đã ẩn / Đã xóa), kỳ phát hành, ngày cập nhật |
| FR-M02 | Lọc theo trạng thái + tìm theo tên/số đề |
| FR-M03 | Mặc định ẩn đề đã xóa; có tùy chọn "Hiện đề đã xóa" để khôi phục |

##### F1.2 Tạo / sửa đề

| Mã | Yêu cầu |
|---|---|
| FR-M04 | Form tạo đề gồm: **tên chủ đề (bắt buộc)**, **khoá Free/Premium (bắt buộc)**, số đề (tự đánh, cho sửa), kỳ phát hành (mặc định theo thời điểm phát hành, cho đổi). Tag Hot/New **không** nhập tay — hệ thống tự gắn |
| FR-M05 | Chọn câu hỏi cho 3 Part từ **ngân hàng câu hỏi**: Part 1 chọn 5 câu, Part 2 chọn 1 cue card, Part 3 chọn 3 câu |
| FR-M06 | Trong lúc chọn câu, cho phép **thêm câu mới vào ngân hàng** rồi chọn ngay, không rời form |
| FR-M07 | **Lời dẫn giám khảo** (giới thiệu, chuyển sang Part 2/Part 3) và **tên giám khảo**: tùy chọn; để trống → dùng lời dẫn/giám khảo mặc định |
| FR-M08 | Lưu Nháp bất kỳ lúc nào, kể cả khi chưa đủ câu / còn thiếu trường |
| FR-M09 | Sau khi tạo, **định danh đề và cấu trúc 3 Part được khoá** (không sửa); các trường khác vẫn sửa được |

##### F1.3 Phát hành / Ẩn / Xóa

| Mã | Yêu cầu |
|---|---|
| FR-M10 | Nút **Phát hành**: kiểm tra hợp lệ (BR-M01) → đạt thì đề xuất hiện với học viên; không đạt → chặn + liệt kê trường lỗi |
| FR-M11 | Nút **Ẩn (gỡ phát hành)**: đề biến khỏi danh sách học viên thấy, vẫn còn trong CMS. Có **hộp thoại xác nhận** |
| FR-M12 | Nút **Xóa**: ẩn khỏi cả CMS (giữ để khôi phục), kết quả bài thi cũ vẫn xem được. Có **hộp thoại xác nhận** |
| FR-M13 | **Xem trước**: hiển thị card đề đúng như học viên thấy (tag/khoá/tên) + danh sách 9 câu ở dạng đọc |

##### Business Rules — Thi thử

- **BR-M01 (kiểm tra hợp lệ khi phát hành):** Chỉ cho phát hành khi **Part 1 đúng 5 câu, Part 2 đúng 1 cue card, Part 3 đúng 3 câu, có tên chủ đề, có khoá Free/Premium**. Thiếu/thừa → chặn, chỉ rõ trường lỗi. (Nháp không áp ràng buộc này.)
- **BR-M02 (tag tự gắn):** Tag **Hot** (đề được bắt đầu nhiều nhất gần đây) và **New** (đề thuộc kỳ hiện tại) do hệ thống **tự tính**, không do content nhập. Content chỉ đặt khoá Free/Premium và kỳ.
- **BR-M03 (khoá sau khi tạo):** Định danh đề và cấu trúc 3 Part không đổi được sau khi tạo (là cơ sở gắn với kết quả/lịch sử của học viên).
- **BR-M04 (sửa đề đã phát hành):** Kết quả các bài **đã hoàn thành được giữ nguyên** — sửa đề không làm thay đổi kết quả/lịch sử cũ. Đề sau sửa chỉ áp cho người **bắt đầu mới**. Người **đang làm dở** mà đề đã đổi câu → tiến độ dở được **làm lại từ đầu với bản mới**, kèm thông báo cho học viên.
- **BR-M05 (số đề):** Tự đánh khi tạo, cho sửa tay; cảnh báo nhẹ nếu trùng số với đề đang phát hành (không chặn).
- **BR-M06 (kỳ phát hành):** Mặc định theo thời điểm phát hành; cho đổi để nhập bù đề cũ hoặc chuẩn bị trước đề kỳ sau. Kỳ quyết định tag New.

##### Acceptance Criteria — Thi thử

**AC-M01: Lưu Nháp đề chưa đủ câu**
- **Given:** Content Editor đang tạo đề, mới chọn 3/5 câu Part 1
- **When:** Bấm "Lưu Nháp"
- **Then:** Đề được lưu ở trạng thái Nháp AND không báo lỗi AND đề không xuất hiện với học viên

**AC-M02: Phát hành bị chặn khi sai cấu trúc**
- **Given:** Đề có Part 1 = 4 câu (thiếu 1) HOẶC Part 2 = 0 cue card HOẶC thiếu khoá Free/Premium
- **When:** Bấm "Phát hành"
- **Then:** Hệ thống chặn AND hiển thị trường lỗi cụ thể (vd "Part 1 cần đúng 5 câu, hiện 4") AND trạng thái vẫn là Nháp

**AC-M03: Phát hành thành công**
- **Given:** Đề đủ 5+1+3 câu AND có tên chủ đề AND có khoá
- **When:** Bấm "Phát hành"
- **Then:** Trạng thái đổi "Đã phát hành" AND đề xuất hiện với học viên AND hiện thông báo xác nhận

**AC-M04: Thêm câu mới trong lúc tạo đề**
- **Given:** Đang chọn câu Part 3, ngân hàng chưa có câu cần dùng
- **When:** Bấm "Thêm câu mới", nhập nội dung, lưu
- **Then:** Câu mới vào ngân hàng dùng chung AND tự được chọn vào đề đang soạn AND form không mất dữ liệu đang nhập

**AC-M05: Khoá cấu trúc sau khi tạo**
- **Given:** Đề đã được tạo
- **When:** Mở form sửa
- **Then:** Định danh đề và cấu trúc 3 Part ở dạng chỉ đọc AND tên/câu hỏi/khoá/lời dẫn/kỳ vẫn sửa được

**AC-M06: Sửa đề không đụng kết quả cũ**
- **Given:** Đề đã phát hành, đã có học viên hoàn thành và có kết quả
- **When:** Sửa 1 câu Part 1 rồi phát hành lại
- **Then:** Kết quả bài đã hoàn thành **giữ nguyên nội dung câu cũ** AND học viên bắt đầu mới nhận bản đã sửa

**AC-M07: Làm lại tiến độ dở khi đề đổi**
- **Given:** Học viên có 1 bài đề X đang làm dở AND đề X đã bị đổi câu sau đó
- **When:** Học viên quay lại làm tiếp đề X
- **Then:** Tiến độ dở được làm lại từ đầu với bản mới AND hiển thị thông báo đề đã cập nhật

**AC-M08: Ẩn đề có xác nhận**
- **Given:** Đề đang "Đã phát hành"
- **When:** Bấm "Ẩn"
- **Then:** Hiện hộp thoại xác nhận AND chỉ khi xác nhận đề mới chuyển "Đã ẩn" AND biến khỏi danh sách học viên thấy AND vẫn còn trong CMS

**AC-M09: Xóa vẫn giữ kết quả cũ**
- **Given:** Đề từng được học viên làm (có kết quả/lịch sử)
- **When:** Bấm "Xóa" và xác nhận
- **Then:** Đề ẩn khỏi CMS mặc định AND kết quả cũ vẫn xem được AND có thể khôi phục qua "Hiện đề đã xóa"

**AC-M10: Xem trước trước khi phát hành**
- **Given:** Đề ở trạng thái Nháp
- **When:** Bấm "Xem trước"
- **Then:** Hiển thị card đề đúng tag/khoá/tên như học viên thấy AND danh sách 9 câu dạng đọc

---

#### F2. Quản lý câu hỏi Luyện tập

| Mã | Yêu cầu |
|---|---|
| FR-P01 | Danh sách chủ đề Luyện tập: tên, Part, số câu, thời lượng, trạng thái (Nháp / Đã phát hành / Đã ẩn / Đã xóa) — nhóm theo Part |
| FR-P02 | Tạo/sửa chủ đề: **tên (bắt buộc)**, **Part (bắt buộc, khoá sau khi tạo)**, phân loại (tùy chọn), danh sách câu (chọn từ ngân hàng / thêm mới) |
| FR-P03 | Chủ đề Luyện tập **không** có khoá Free/Premium riêng, **không** tag Hot/New/Forecast/Premium (form tối giản, khác Thi thử) |
| FR-P04 | Thời lượng ("x phút") **hệ thống tự ước lượng theo số câu**, không nhập tay |
| FR-P05 | Kiểm tra hợp lệ khi phát hành: **tên + Part + ít nhất 1 câu** |
| FR-P06 | Phát hành / Ẩn / Xóa + Xem trước: **hành vi giống Thi thử** (F1.3), chỉ khác bộ trường |

##### Business Rules — Luyện tập

- **BR-P01 (đơn vị nội dung):** Chủ đề Luyện tập **gắn 1 Part cụ thể** (vd "Advertisement — Part 1"). Cùng tên chủ đề ở Part khác = một mục độc lập.
- **BR-P02 (khoá sau khi tạo):** Định danh chủ đề + Part không đổi được sau khi tạo.
- **BR-P03 (hợp lệ):** Phát hành cần tên + Part + ít nhất 1 câu. Thiếu → chặn, báo trường lỗi.
- **BR-P04 (sửa/xóa):** Giống Thi thử — sửa không đụng kết quả/lịch sử cũ; Ẩn + Xóa (giữ để khôi phục).

##### Acceptance Criteria — Luyện tập

**AC-P01: Tạo chủ đề gắn Part**
- **Given:** Content Editor ở vùng Luyện tập
- **When:** Tạo chủ đề "Advertisement", chọn Part 1, thêm 5 câu, phát hành
- **Then:** Chủ đề xuất hiện dưới tab Part 1 với "5 câu · [thời lượng tự tính]" AND không có nhãn khoá/tag nào

**AC-P02: Part khoá sau khi tạo**
- **Given:** Chủ đề đã tạo với Part 1
- **When:** Mở form sửa
- **Then:** Part ở dạng chỉ đọc AND tên/câu hỏi/phân loại sửa được

**AC-P03: Chặn phát hành chủ đề rỗng**
- **Given:** Chủ đề có 0 câu
- **When:** Bấm Phát hành
- **Then:** Chặn AND báo "cần ít nhất 1 câu" AND trạng thái vẫn Nháp

**AC-P04: Thời lượng tự tính**
- **Given:** Chủ đề có N câu
- **When:** Chủ đề hiển thị trong CMS và với học viên
- **Then:** Thời lượng = giá trị tự ước lượng theo N, không phải trường nhập tay

---

#### F3. Ngân hàng câu hỏi

| Mã | Yêu cầu |
|---|---|
| FR-B01 | Ngân hàng câu hỏi là nơi quản lý câu tập trung, tách theo Part; tạo đề/chủ đề bằng cách chọn câu từ đây |
| FR-B02 | Thêm/sửa câu hỏi; **sửa lan toả** — đổi nội dung 1 câu → mọi đề/chủ đề đang dùng câu đó đổi theo |
| FR-B03 | Khi sửa 1 câu, hiển thị **"đang được dùng ở N đề / M chủ đề"** để người soạn biết phạm vi ảnh hưởng |
| FR-B04 | **Xóa câu đang được dùng → chặn**, buộc gỡ khỏi các đề/chủ đề trước; câu chưa ai dùng thì xóa được |

##### Acceptance Criteria — Ngân hàng

**AC-B01: Sửa câu lan toả**
- **Given:** Câu Q đang được dùng ở 3 đề Thi thử và 1 chủ đề Luyện tập
- **When:** Sửa nội dung Q và lưu
- **Then:** Cả 3 đề và 1 chủ đề đều phản ánh nội dung mới AND trước khi lưu hệ thống hiển thị "đang được dùng ở 3 đề / 1 chủ đề"

**AC-B02: Chặn xóa câu còn được dùng**
- **Given:** Câu Q đang được ít nhất 1 đề/chủ đề dùng
- **When:** Bấm xóa Q
- **Then:** Hệ thống chặn AND liệt kê nơi đang dùng AND gợi ý gỡ khỏi các nơi đó trước

---

#### F4. Phân quyền

| Mã | Yêu cầu |
|---|---|
| FR-A01 | Chỉ **Content Editor** (vai trò mới) và **admin** thấy & dùng khu vực Quản lý nội dung — tách khỏi quyền quản trị người dùng/thanh toán |
| FR-A02 | Ghi **nhật ký thay đổi (audit log)**: ai, khi nào, tạo/sửa/phát hành/ẩn/xóa nội dung gì |

---

### 4.2 Non-Functional Requirements

- **Hiệu năng cảm nhận:** danh sách và form tải nhanh, thao tác không giật; là công cụ nội bộ dùng trên desktop.
- **Tương thích:** trình duyệt desktop phổ biến bản mới (Chrome/Safari/Firefox), màn hình ≥ 1280px.
- **Khả dụng (Accessibility):** nhãn trường rõ, thông báo lỗi đọc được bằng trình đọc màn hình, tương phản đạt chuẩn; hộp thoại xác nhận đóng được bằng bàn phím.

---

## 5. User Types

| User Type | Định nghĩa | Hành vi feature |
|---|---|---|
| **Content Editor** | Vai trò mới, soạn/phát hành nội dung | Toàn quyền vùng Quản lý nội dung (tạo/sửa/phát hành/ẩn/xóa đề Thi thử + chủ đề Luyện tập + ngân hàng câu). Không thấy quản trị người dùng/thanh toán |
| **Admin** | Quản trị CMS toàn phần | Như Content Editor + các quyền admin khác đã có ở CMS |
| **Người xem dashboard khác** (PM/Growth…) | Chỉ xem dashboard | **Không** thấy/không dùng vùng Quản lý nội dung |
| **Học viên** | Ngoài CMS | Chỉ thấy nội dung **đã phát hành**; không bao giờ thấy Nháp/Đã ẩn/Đã xóa |

---

## 6. Analytics

Những hành vi cần đo (phục vụ thiết lập baseline vận hành content):

- Số đề/chủ đề **được tạo Nháp**, **được phát hành**, **được sửa**, **bị ẩn/xóa** — theo kỳ, theo người soạn.
- Số lần **phát hành bị chặn** và **loại lỗi** (thiếu câu / sai cấu trúc / thiếu trường) — để cải thiện form soạn thảo.
- Số **câu mới thêm vào ngân hàng** và số lần **sửa câu lan toả** (kèm phạm vi "dùng ở N đề/chủ đề").
- **Thời gian từ Nháp đến Phát hành** của mỗi đề.

> Bảng sự kiện chi tiết (tên sự kiện, thuộc tính) sẽ do bước thiết kế tracking riêng chuẩn hoá — không thuộc phạm vi PRD này.

---

## 7. Timelines

| Milestone | Target | Status |
|---|---|---|
| PRD review & sign-off | TBD | ⬜ |
| Đánh giá khả thi & ước lượng (đội kỹ thuật) | TBD | ⬜ |
| Thiết kế UI/UX | TBD | ⬜ |
| Xây dựng & QA | TBD | ⬜ |
| UAT với Content Team | TBD | ⬜ |
| Release | TBD | ⬜ |

---

## 8. Dependencies & Approvals

| # | Dependency | Owner | Status |
|---|---|---|---|
| 1 | **Khả thi kỹ thuật & phương án triển khai** (nơi lưu, cách nội dung phát hành lên app, migrate nội dung hiện có nếu cần) — đánh giá trong tài liệu kỹ thuật riêng, **ngoài phạm vi PRD này** | Tech Lead | ⬜ |
| 2 | **Thống nhất & cấp vai trò "Content Editor"** trong CMS | Product + Admin | ⬜ |
| 3 | **Chuẩn hoá tracking** cho các hành vi ở §6 | Product / Data | ⬜ |
| 4 | **Duyệt UI/UX** cho form soạn đề + soạn chủ đề Luyện tập | Product + Design | ⬜ |

### Câu hỏi mở (nghiệp vụ)

1. Công thức ước lượng thời lượng chủ đề Luyện tập theo số câu (thời lượng chuẩn / câu)?
2. Số đề trùng: cảnh báo nhẹ là đủ, hay cần bắt buộc duy nhất trong một kỳ?
3. Nhật ký thay đổi cần chi tiết tới mức nào (chỉ tạo/phát hành/xóa, hay cả từng trường thay đổi)?

---

## 9. Appendix

### 9.1 Grill-Me Decision Summary

| # | Topic | Quyết định |
|---|---|---|
| 1 | Phạm vi | CMS nội dung chung, **2 vùng tách biệt** (đề Thi thử + câu hỏi Luyện tập), luồng soạn độc lập, không dùng chung form |
| 2 | Soạn câu hỏi Thi thử | **Lai**: chọn câu từ ngân hàng; thiếu thì thêm câu mới vào ngân hàng ngay trong luồng tạo đề |
| 3 | Trạng thái phát hành | **Nháp → Đã phát hành**; Nháp chỉ thấy trong CMS + xem trước, không lộ ra học viên |
| 4 | Lời dẫn + giám khảo | **Tùy chọn, có mặc định**: trống thì dùng lời dẫn/giám khảo mặc định; muốn cá nhân hoá thì tự điền |
| 5 | Metadata thẻ | Số đề **tự đánh, cho sửa**; kỳ **mặc định theo thời điểm phát hành, cho đổi**; content nhập tên/khoá/kỳ; tag Hot/New tự gắn |
| 6 | Đơn vị Luyện tập | **Chủ đề gắn 1 Part cụ thể**; cùng tên ở Part khác = mục khác |
| 7 | Kiểm tra hợp lệ | **Chặn cứng**: Thi thử đúng 5+1+3 câu + tên + khoá; Luyện tập tên + Part + ≥1 câu; Nháp vẫn lưu khi chưa đủ |
| 8 | Khoá sau khi tạo | Định danh đề/chủ đề + cấu trúc (3 Part / Part) không đổi; trường khác sửa được |
| 9 | Sửa đề đang dùng | Không đụng kết quả/lịch sử đã hoàn thành; áp cho người bắt đầu mới; đang làm dở mà đề đổi → **làm lại bản mới** (báo học viên) |
| 10 | Xóa | **Giữ để khôi phục** (kết quả cũ vẫn xem được) + bước nhẹ hơn **Ẩn (gỡ phát hành)**; không xóa vĩnh viễn |
| 11 | Thông báo | Tạo/Sửa → thông báo nhẹ; **Xóa & Ẩn → hộp thoại xác nhận** rồi thông báo |
| 12 | Phân quyền | **Content Editor (vai trò mới) + admin**; tách khỏi quản trị người dùng/thanh toán |
| 13 | Xem trước | **Card + xem đủ câu hỏi** ở v1; chạy thử phòng thi thật = mong muốn **v2** |
| 14 | Sửa/xóa câu ngân hàng | **Sửa lan toả**, cảnh báo "đang dùng ở N đề/chủ đề"; **xóa câu còn được dùng → chặn** |
| 15 | Nội dung hiện có | Nội dung đang có cần được đưa vào CMS để quản lý/sửa được (phương án cụ thể do đội kỹ thuật quyết) |
| 16 | Khoá/tag Luyện tập | Luyện tập **tối giản**: không khoá per-chủ-đề, không tag; thời lượng **tự tính theo số câu** (khác Thi thử giàu metadata) |

### 9.2 Out of scope v1

- Tab "Đề thi custom" (học viên tự tạo — không thuộc kho đề CMS).
- Audio đáp án mẫu cho câu hỏi.
- **Xem trước động** (chạy thử phòng thi/phòng luyện với đề nháp) → v2.
- **Lịch phát hành theo kỳ tự động** (đề kỳ sau tự lên) → v2.
- Khoá/tag riêng cho từng chủ đề Luyện tập.
- Phân quyền nhiều cấp chi tiết trong CMS (v1 chỉ tách Content Editor vs admin).

### 9.3 Tài liệu liên quan

| Document | Link | Mô tả |
|---|---|---|
| Card đề Thi thử (screen spec) | [`01-chon-de-goi-y.md`](../modules/mock-test/screens/01-chon-de-goi-y.md) | §1.3 — nguồn của các trường/tag trên card đề |
| CMS Dashboard v2 | [`PRD_CMS_Dashboard_v2_2026-07-25.md`](./PRD_CMS_Dashboard_v2_2026-07-25.md) | PRD anh em — dashboard + quản lý user (tách biệt PRD này) |
| Glossary | — | cue card, Part 1/2/3, kỳ phát hành, tag Hot/New/Forecast/Premium, khoá Free/Premium |

---

*PRD tạo từ grill-me session 2026-08-05 (16 câu). Mọi thay đổi quyết định cần cập nhật cả §9.1 Decision Summary.*
