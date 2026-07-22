# Lượt chấm điểm — Giải thích cho đội Product & Bộ test case

> Tài liệu này mô tả cách hệ thống **kiểm tra lượt, trừ lượt, thanh toán, cộng lượt** bằng ngôn ngữ đời thường (không kỹ thuật), kèm bộ test case để đội product tự kiểm thử.
> Cập nhật: 2026-06-10.

---

## PHẦN A — GIẢI THÍCH LUỒNG

### 1. Khái niệm cơ bản

**"Lượt chấm điểm" là gì?**
Mỗi lần người dùng nói xong một bài và bấm để hệ thống AI chấm điểm, hệ thống tiêu tốn **1 lượt chấm**. Hết lượt thì không chấm được nữa, phải mua thêm hoặc nâng cấp gói.

**Có 2 loại bài, đếm lượt riêng:**
- **Luyện tập (Practice):** mỗi câu hỏi luyện nói là 1 bài → tốn 1 lượt practice.
- **Thi thử (Mock test):** một đề thi thử có nhiều phần (Part 1, 2, 3). **Cả đề thi thử chỉ tính 1 lượt mock test** (không phải mỗi part một lượt).

**Có 2 "ví" lượt cho mỗi loại bài:**
- **Lượt miễn phí (free):** số lượt được tặng/khuyến mãi.
- **Lượt trả phí (đã mua):** số lượt có được sau khi mua gói.

Khi chấm, hệ thống tự ưu tiên dùng đúng "ví" phù hợp. Người dùng không cần chọn — hệ thống tự quyết định lần chấm này dùng lượt miễn phí hay lượt trả phí.

---

### 2. Luồng KIỂM TRA trước khi chấm

Trước khi AI bắt đầu chấm, hệ thống luôn hỏi: **"Người này còn lượt để chấm không?"**

- **Còn lượt** → cho phép chấm tiếp, và ghi nhớ lần này sẽ dùng "ví miễn phí" hay "ví trả phí".
- **Hết lượt** → **dừng ngay, không chấm**, hiện thông báo:
  - Với luyện tập: *"Bạn đã hết lượt chấm luyện tập. Vui lòng nâng cấp gói hoặc chấm lại sau."*
  - Với thi thử: *"Bạn đã hết lượt chấm thi thử. Vui lòng nâng cấp gói hoặc chấm lại sau."*
  - Kèm theo đó, màn hình mời người dùng mua thêm lượt / nâng cấp gói (paywall).

> **Lưu ý cách "chặn" hiển thị ra sao:** hệ thống **không khoá nút ghi âm từ đầu**. Người dùng vẫn **ghi âm và bấm chấm như bình thường**; việc kiểm tra lượt diễn ra ngay khi bấm chấm. Nếu hết lượt, bài **không được chấm** mà chuyển sang trạng thái **"đang chờ chấm" (pending)** kèm paywall — bài và audio **được giữ lại**, nên sau khi mua thêm lượt, **bài đang chờ tự được chấm lại**, không phải ghi âm lại từ đầu. (Vì vậy test case TC-1.2/1.3 mô tả "nói → bấm chấm → mới hiện thông báo", chứ không phải bị chặn ngay khi mở câu hỏi.)

**Trường hợp đặc biệt — phiên đăng nhập hết hạn:**
Nếu trong lúc chấm mà phiên đăng nhập 9Speak đã hết hạn, hệ thống báo *"Phiên đăng nhập 9Speak không còn hợp lệ. Vui lòng đăng nhập lại."* Hệ thống sẽ cố làm mới phiên rồi cho thử lại, không bắt đăng nhập lại từ đầu nếu tránh được.

---

### 3. Luồng TRỪ LƯỢT (thời điểm trừ rất quan trọng)

Đây là điểm cần đội product nắm kỹ vì ảnh hưởng trực tiếp tới trải nghiệm và khiếu nại tính tiền.

**Nguyên tắc cốt lõi: chỉ trừ lượt khi đã CÓ ĐIỂM.**

Một bài chấm thực ra có nhiều bước: chấm điểm → gợi ý từ vựng → gợi ý ngữ pháp. **Hệ thống trừ lượt ngay khi có điểm, KHÔNG chờ phần từ vựng/ngữ pháp xong.**

Hệ quả quan trọng:
- **Phần gợi ý từ vựng, gợi ý ngữ pháp, và "viết lại / cải thiện câu trả lời" đều MIỄN PHÍ** — không tốn thêm lượt. Nếu các phần này lỗi và người dùng bấm thử lại, **không bị trừ thêm lượt**.
- Nói cách khác: **người dùng trả tiền cho ĐIỂM, không trả cho các gợi ý kèm theo.**

**Với bài luyện tập (Practice):**
- Trừ 1 lượt practice ngay khi có điểm của bài đó.

**Với bài thi thử (Mock test):**
- Đề thi thử gồm Part 1, Part 2, Part 3.
- Hệ thống **chỉ trừ 1 lượt mock test khi CẢ BA part đều đã có điểm.**
- Nếu mới chấm được 1–2 part (chưa đủ cả 3) thì **chưa trừ lượt nào cả**.

**Cơ chế chống trừ trùng (rất quan trọng cho khiếu nại tính tiền):**
- Mỗi phiên làm bài có một "sổ ghi nợ" riêng. Hệ thống dùng sổ này để đảm bảo **một phiên chỉ bị trừ đúng 1 lần**, dù người dùng có bấm lại nhiều lần, mạng chập chờn gọi lại nhiều lần, hay mở nhiều tab.
- Khi đang trong quá trình trừ, phiên bị "khóa" để không ai trừ chồng lên.

**Nếu việc trừ lượt gặp lỗi (mạng/timeout/lỗi hệ thống):**
- **Người dùng VẪN nhận được kết quả chấm điểm và bài vẫn được lưu vào lịch sử.** Việc trừ lượt lỗi **không** chặn người dùng xem kết quả.
- Phiên đó được đánh dấu là "trừ lỗi" để bộ phận vận hành đối soát lại sau (xem Phần 5).
- Lý do thiết kế: người dùng đã nhận được dịch vụ (đã có điểm) thì phải được thấy kết quả; chuyện tiền nong tính lại nội bộ, không làm phiền người dùng.

---

### 4. Luồng CỘNG LƯỢT

Lượt được cộng vào tài khoản người dùng trong các trường hợp:

1. **Mua gói / nâng cấp gói (chính):** sau khi thanh toán thành công, hệ thống cập nhật lại hồ sơ người dùng và cộng số lượt practice + lượt mock test theo gói đã mua. (Chi tiết ở Phần 5.)
2. **Lượt miễn phí được tặng:** do hệ thống backend cấp (khuyến mãi, người dùng mới…). Phía app chỉ hiển thị số lượt còn lại, không tự cấp.

---

### 5. Luồng THANH TOÁN

Người dùng mua lượt qua **gói đăng ký (subscription)**, thanh toán bằng chuyển khoản QR.

#### 5.1. Mua gói đăng ký — qua chuyển khoản QR

Các bước người dùng trải qua:

1. **Chọn gói:** app hiển thị danh sách gói (giá, số lượt practice, số lượt mock test, thời hạn). *Giá và số lượt luôn lấy trực tiếp từ hệ thống, không có giá dự phòng — nếu không lấy được danh sách gói thì app chặn thanh toán và báo lỗi, tránh tạo đơn sai giá.*
2. **Tạo đơn:** app tạo một đơn hàng, nhận về **mã QR + thông tin chuyển khoản + đồng hồ đếm ngược** (hạn thanh toán).
3. **Người dùng quét QR và chuyển khoản.**
4. **App tự kiểm tra trạng thái đơn mỗi 3 giây** cho tới khi đơn chuyển sang "đã thanh toán".
5. **Khi đã thanh toán:**
   - App làm mới hồ sơ người dùng → số lượt mới được cộng vào.
   - Hiện popup thành công, **tự đóng sau 5 giây** (hoặc người dùng bấm nút).
   - Sau đó đưa người dùng đi tiếp theo đúng ngữ cảnh:
     - Đang ở trang bảng giá/cài đặt → hiển thị trạng thái thành công (gói + lượt mới).
     - Đang kẹt ở paywall lúc chấm → **tự động chấm lại** bài đang dở.

**Ba kết cục có thể xảy ra của một đơn:**
- **Đã thanh toán (paid):** cộng lượt, đi tiếp như trên.
- **Hết hạn (expired) hoặc đồng hồ về 0:** báo hết hạn, hủy đơn. Muốn mua lại phải **chọn gói từ đầu**.
- **Lỗi kỹ thuật** (không tạo được đơn / quá thời gian chờ / lỗi máy chủ / mất mạng): báo lỗi, mời thử lại.

**Về mã QR:** nếu ảnh QR không tải được → báo lỗi "không tải được QR". App **không** hiển thị thông tin chuyển khoản thủ công thay thế (tránh chuyển nhầm).

---

### 6. Tóm tắt một dòng cho mỗi luồng

| Luồng | Tóm tắt |
|---|---|
| Kiểm tra lượt | Trước khi chấm, hỏi "còn lượt không?". Hết → chặn + mời mua. |
| Trừ lượt | Chỉ trừ khi **đã có điểm**. Practice: 1 lượt/bài. Mock test: 1 lượt khi đủ cả 3 part. |
| Miễn phí kèm theo | Từ vựng, ngữ pháp, viết lại câu → **không tốn thêm lượt**. |
| Trừ lỗi | Người dùng **vẫn nhận kết quả**; phiên được đánh dấu để đối soát. |
| Cộng lượt | Sau thanh toán thành công, làm mới hồ sơ → cộng lượt theo gói. |
| Thanh toán | Gói đăng ký: QR chuyển khoản, poll mỗi 3s. |

---

## PHẦN B — BỘ TEST CASE

> Quy ước: **Điều kiện** = trạng thái cần chuẩn bị trước; **Các bước** = thao tác; **Kết quả kỳ vọng** = điều phải xảy ra.
> Gợi ý chuẩn bị: cần tài khoản test có thể điều chỉnh số lượt miễn phí / lượt trả phí (nhờ kỹ thuật set giúp), và một tài khoản hết sạch lượt.

### Nhóm 1 — Kiểm tra lượt trước khi chấm

**TC-1.1 — Còn lượt thì chấm được (Practice)**
- Điều kiện: tài khoản còn ≥ 1 lượt practice.
- Các bước: vào 1 câu luyện tập → nói/ghi âm → bấm chấm.
- Kết quả kỳ vọng: chấm thành công, hiện điểm bình thường.

**TC-1.2 — Hết lượt thì bị chặn (Practice)**
- Điều kiện: tài khoản hết sạch lượt practice (cả miễn phí lẫn trả phí).
- Các bước: vào 1 câu luyện tập → nói → bấm chấm.
- Kết quả kỳ vọng: **không chấm**, hiện thông báo "Bạn đã hết lượt chấm luyện tập…", và hiện màn hình mời mua/nâng cấp.

**TC-1.3 — Hết lượt thì bị chặn (Mock test)**
- Điều kiện: tài khoản hết sạch lượt mock test.
- Các bước: làm một đề thi thử → tới bước chấm.
- Kết quả kỳ vọng: **không chấm**, hiện thông báo "Bạn đã hết lượt chấm thi thử…", và hiện màn hình mời mua/nâng cấp.


### Nhóm 2 — Trừ lượt đúng thời điểm

**TC-2.1 — Practice trừ đúng 1 lượt khi có điểm**
- Điều kiện: ghi nhận số lượt practice trước khi chấm (ví dụ đang 5).
- Các bước: chấm xong 1 bài luyện tập → kiểm tra lại số lượt.
- Kết quả kỳ vọng: số lượt giảm đúng **1** (còn 4).

**TC-2.2 — Từ vựng / ngữ pháp KHÔNG trừ thêm lượt**
- Điều kiện: vừa chấm xong 1 bài (đã trừ 1 lượt), ghi nhận số lượt còn lại.
- Các bước: xem phần gợi ý từ vựng, gợi ý ngữ pháp; nếu có nút thử lại các phần này thì bấm thử lại.
- Kết quả kỳ vọng: số lượt **không đổi** so với sau khi chấm điểm.

**TC-2.3 — "Viết lại / cải thiện câu trả lời" KHÔNG trừ lượt**
- Điều kiện: đang xem kết quả một bài đã chấm.
- Các bước: dùng tính năng viết lại / cải thiện câu trả lời nhiều lần.
- Kết quả kỳ vọng: số lượt **không đổi**.

**TC-2.4 — Mock test chỉ trừ 1 lượt khi đủ cả 3 part**
- Điều kiện: ghi nhận số lượt mock test (ví dụ 3).
- Các bước: làm đề thi thử, chấm lần lượt Part 1 → kiểm tra lượt; Part 2 → kiểm tra lượt; Part 3 → kiểm tra lượt.
- Kết quả kỳ vọng: sau Part 1 và Part 2 **chưa trừ** (vẫn 3); chỉ sau khi part cuối cùng có điểm (đủ cả 3) mới trừ đúng **1** (còn 2).

**TC-2.5 — Mock test bỏ dở không bị trừ**
- Điều kiện: ghi nhận số lượt mock test.
- Các bước: làm đề thi thử nhưng chỉ chấm 1–2 part rồi thoát.
- Kết quả kỳ vọng: số lượt mock test **không đổi** (vì chưa đủ cả 3 part).


### Nhóm 4 — Thanh toán gói đăng ký (QR)

**TC-4.1 — Mua gói thành công cộng lượt**
- Điều kiện: tài khoản test (lý tưởng là đang hết/sắp hết lượt); ghi nhận số lượt hiện tại.
- Các bước: chọn gói → tạo đơn → quét QR và chuyển khoản đúng số tiền/nội dung → chờ.
- Kết quả kỳ vọng: trong vòng vài giây app báo "đã thanh toán"; popup thành công hiện ra rồi tự đóng sau 5s; số lượt practice + mock test **tăng đúng theo gói**; thời hạn gói cập nhật.

**TC-4.2 — Mua gói khi đang kẹt paywall → tự chấm lại**
- Điều kiện: đang ở màn paywall vì hết lượt, có bài đang chờ chấm.
- Các bước: mua gói thành công ngay tại paywall.
- Kết quả kỳ vọng: sau khi cộng lượt, **bài đang dở được tự động chấm lại**, không phải làm lại từ đầu.

**TC-4.3 — Đơn hết hạn**
- Điều kiện: tạo đơn nhưng không chuyển khoản.
- Các bước: chờ đồng hồ đếm ngược về 0 (hoặc đơn chuyển "hết hạn").
- Kết quả kỳ vọng: app báo **hết hạn**, dừng kiểm tra; muốn mua lại phải **chọn gói lại từ đầu**; **không** cộng lượt.

**TC-4.4 — Lỗi kỹ thuật khi tạo/kiểm tra đơn**
- Điều kiện: giả lập lỗi mạng/máy chủ khi tạo đơn hoặc khi đang poll.
- Các bước: chọn gói → tạo đơn (gặp lỗi) hoặc đang chờ thì mất mạng.
- Kết quả kỳ vọng: app báo **lỗi** (ví dụ "Yêu cầu thanh toán bị quá thời gian chờ", "Không tìm thấy gói hoặc đơn hàng"), dừng đếm; **không** cộng lượt; cho phép thử lại.

**TC-4.5 — QR không tải được**
- Điều kiện: giả lập ảnh QR lỗi.
- Các bước: tạo đơn, quan sát vùng hiển thị QR.
- Kết quả kỳ vọng: báo **"không tải được QR"**; **không** hiển thị thông tin chuyển khoản thủ công thay thế.

**TC-4.6 — Không lấy được bảng giá**
- Điều kiện: giả lập lỗi khi tải danh sách gói.
- Các bước: mở bảng giá / paywall.
- Kết quả kỳ vọng: **chặn thanh toán**, báo lỗi; **không** hiển thị giá cũ/giá đoán; không cho tạo đơn.

**TC-4.7 — Số tiền hiển thị đúng nguồn**
- Điều kiện: bảng giá tải được.
- Các bước: so sánh giá hiển thị với giá hệ thống trả về.
- Kết quả kỳ vọng: giá/credits/thời hạn **khớp đúng** dữ liệu hệ thống, không lệch.

### Nhóm 5 — Cộng lượt sau thanh toán

**TC-5.1 — Cộng lượt sau thanh toán phản ánh ngay**
- Điều kiện: vừa thanh toán thành công gói đăng ký.
- Các bước: kiểm tra số lượt ở màn hình chính / cài đặt sau khi popup đóng.
- Kết quả kỳ vọng: số lượt mới hiển thị **ngay**, không cần đăng nhập lại / tải lại app thủ công.

---

## PHẦN C — Lưu ý & rủi ro cần đội Product để mắt

1. **Trừ lỗi vẫn cho dùng:** đúng về trải nghiệm, nhưng tạo ra "nợ đối soát". Cần lịch chạy báo cáo đối soát định kỳ (hiện đang chạy thủ công).
2. **Mock test trừ theo cả đề:** cần truyền thông rõ cho người dùng để tránh hiểu nhầm "mỗi part một lượt".
