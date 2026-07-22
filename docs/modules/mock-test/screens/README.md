# Mock Test — Đặc tả chi tiết theo từng màn hình (Screens)

> Mỗi file dưới đây là **spec đầy đủ, chính thức, chỉ 1 bản hiện hành** cho một chức năng/màn hình trong luồng Mock Test. Không giữ song song "bản gốc" + "bản V2" — mọi thay đổi sửa thẳng vào file, kèm bảng **Đổi log** ở đầu file để biết version nào đổi gì/vì sao (lịch sử chi tiết hơn nằm ở git log).
>
> Đây là **nguồn chi tiết** cho dev khi triển khai; [`../PRD.md`](../PRD.md) chỉ tóm tắt + trạng thái as-built và link ra đây. Đặc tả kỹ thuật (route/API/data model) nằm ở [`../TECH.md`](../TECH.md).
>
> Nguồn gốc: chuyển thể từ Google Doc tổng ("Mocktest V2") — doc đó đang trộn base + patch V2 làm 2 khối tách rời cho mỗi màn, gây khó theo dõi bản hiện hành. Từ nay, **repo là nguồn chính thức**; Google Doc chỉ còn giá trị tham khảo lịch sử.

## Danh sách 9 màn hình

| # | File | Chức năng | Trạng thái |
|---|---|---|---|
| 01 | [01-chon-de-goi-y.md](01-chon-de-goi-y.md) | Chọn đề & Gợi ý — engine gợi ý, thẻ trạng thái, filter, lưới đề | ✅ Chính thức (v2.0, 2026-07-03) — đã gộp bản V2; **chưa triển khai trên code** |
| 02 | [02-cau-hoi-tu-chon.md](02-cau-hoi-tu-chon.md) | Câu hỏi tự chọn (thêm/xoá/luyện) | ✅ Chính thức (v0.1, không có bản V2) |
| 03 | [03-chuan-bi-kiem-tra-mic.md](03-chuan-bi-kiem-tra-mic.md) | Chuẩn bị & Kiểm tra Mic | ✅ Chính thức (v0.3, 2026-07-07) — đối chiếu code thật + chốt hết câu hỏi mở qua grill-me |
| 04 | [04-phong-thi-giam-khao.md](04-phong-thi-giam-khao.md) | Phòng thi — Giám khảo dẫn dắt (3 Part, ghi âm, phụ đề) | ✅ Chính thức (v0.3, 2026-07-07) — đối chiếu code thật + chốt 5 quyết định qua grill-me (resume im lặng, xác nhận huỷ 2 bước, chặn huỷ sau khi trừ lượt...); **chế độ Part lẻ + toàn bộ #05 vẫn là spec đích chưa build** |
| 05 | [05-edge-case-khi-thi.md](05-edge-case-khi-thi.md) | Xử lý Edge case khi thi (B/A/D/C) | ✅ Chính thức (v0.2, 2026-07-07) — đối chiếu code thật + chốt qua grill-me. Nhóm B (B1/B2/B3/B5) 0% xây; D1 (mất mic giữa lúc ghi) đánh dấu **🔴 rủi ro dữ liệu, ưu tiên cao hơn nhóm B** (nộp bài hỏng với tín hiệu giả thành công); C1 tách 2 điểm lỗi (live trong thi vs finalize ở Processing) |
| 06 | [06-processing-cham-diem.md](06-processing-cham-diem.md) | Processing — Chấm điểm | ✅ Chính thức (v0.2, 2026-07-07) — đối chiếu code thật + chốt qua grill-me. Đã bổ sung màn "Chúc mừng" (3.6s, xác nhận qua code) vào luồng; sửa lại FR-06 "progressive reveal" cũ vốn không đúng thực tế (Processing luôn chờ chấm xong hoàn toàn mới chuyển report — quyết định chủ đích, không phải thiếu sót) |
| 07 | [07-report-tong-quan.md](07-report-tong-quan.md) | Report — Tổng quan & Accordion tiêu chí | ✅ Chính thức (v0.2, 2026-07-07) — đối chiếu code thật, chốt dứt điểm cả 3 câu hỏi mở cũ (sub-score và descriptor đều do AI sinh động theo hiệu suất thật, không phải bảng tra tĩnh; mapping band→CEFR đã có sẵn) |
| 08 | [08-report-phan-hoi-chi-tiet.md](08-report-phan-hoi-chi-tiet.md) | Report — Phản hồi chi tiết theo Part/câu | ✅ Chính thức (v0.2, 2026-07-07) — đối chiếu code thật, chốt dứt điểm 4 câu hỏi mở cũ (nguồn dữ liệu từng tiêu chí: rule/AI/pipeline/tĩnh — xem bảng §8.2). Gap học thuật chi tiết vẫn ở [gap.md](../gap.md) |
| 09 | [09-report-hoc-thuat.md](09-report-hoc-thuat.md) | Học thuật: rubric + band descriptor + ngưỡng 4 tiêu chí IELTS | ✅ Chính thức (v0.2, 2026-07-07) — tách khỏi Google Doc, đối chiếu code thật; 4/7 câu hỏi mở đã chốt qua research; 3 câu còn treo (calibration ngưỡng, IPA chuẩn UK/US, phoneme taxonomy tiếng Việt) |

## Quy ước chung mỗi file

Các file đã qua quy trình chuẩn (01, 03-08) theo khung sau (số mục X.1-X.8 ứng với số màn hình X):

- **Đổi log** — bảng thay đổi so với bản trước, kèm lý do/bằng chứng code
- **Goal** — Business/Product Goal + User Benefits (mượn từ `/create-prd`)
- **X.1 Tổng quan** · **X.2 Yêu cầu chức năng** (FR + Business rules + phân quyền) · **X.3 Cơ chế hoạt động** · **X.4 Xử lý ngoại lệ và trạng thái** · **X.5 Yêu cầu giao diện (UI/UX)** · **X.6 Tiêu chí chấp nhận** (mỗi AC dạng bullet **Given/When/Then**, không phải bảng phẳng)
- **Non-functional Requirements** · **User Types** (mượn từ `/create-prd`)
- **X.7 Phần bổ sung** (giả định, phụ thuộc, câu hỏi mở thật sự còn treo)
- **X.8 Câu hỏi mở — đã chốt** (bảng câu hỏi cũ + quyết định/bằng chứng chốt tại đâu)

File **02** vẫn theo template 7 mục cũ (chuyển thể thô từ Google Doc, chưa qua quy trình trên).

## Việc còn lại

**✅ Hoàn tất 8/9 (01, 03, 04, 05, 06, 07, 08, 09)** — mỗi file đã qua đủ quy trình chuẩn: đối chiếu code thật (research agent + writer agent) + grill-me trực tiếp với product owner để chốt mọi vấn đề còn mơ hồ. Đây là nguồn spec chính thức duy nhất cho dev — không còn cần tra Google Doc gốc cho 8 file này.

File 09 khác biệt: là tài liệu rubric/tham chiếu, không phải "màn hình", nên không theo template 7 mục FR/BR/AC — giữ bố cục (a)–(f) mỗi tiêu chí IELTS và bổ sung annotation đối chiếu code ✅/⚠️/❌.

Còn lại:
- **02 (Câu hỏi tự chọn)** — chưa xử lý theo quy trình mới; dự án chưa triển khai tới tính năng này nên tạm hoãn theo yêu cầu.

**Khoảng trống hành vi đã xác nhận qua code (chưa khớp giữa spec và implementation, không phải lỗi tài liệu):**
- Chế độ Part lẻ (examMode chỉ chạy 1 Part) — chưa build ở cả màn Chọn đề (01) lẫn Phòng thi (04); CTA Part vẫn dẫn vào full.
- Toàn bộ Edge case Nhóm B (B1/B2/B3/B5, file 05) — 0% xây dựng, không có hạ tầng phát hiện độ dài/im lặng/nhiễu/ngôn ngữ.
- D1 (mất mic giữa lúc ghi, file 05) — 🔴 rủi ro dữ liệu cao hơn nhóm B: hệ thống âm thầm nộp bài với dữ liệu hỏng mà báo tín hiệu thành công giả, cần ưu tiên xử lý trước các case khác.
