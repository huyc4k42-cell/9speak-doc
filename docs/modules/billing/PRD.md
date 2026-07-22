# Billing & Paywall — Mô tả nghiệp vụ (PRD)

> Module: `features/billing/` — **Thanh toán & Paywall**
> Đối tượng đọc: AI agent và đầu mối phụ trách module. Mục tiêu: hiểu đầy đủ nghiệp vụ thanh toán + paywall và các **ràng buộc giữa module** trước khi phát triển/sửa lỗi.
> Tài liệu kỹ thuật đi kèm: [TECH.md](./TECH.md).
> Tài liệu nền tảng: [README-modules](../../README-modules.md) · [env-and-modules](../../env-and-modules.md) · [Lượt chấm điểm cho Product](../../luot-cham-diem-cho-product.md).

---

## 1. Tổng quan & mục tiêu

Module Billing chịu trách nhiệm **toàn bộ hạ tầng thanh toán dịch vụ** của hệ thống IELTS Speaking và **paywall** (màn chặn mời mua khi người dùng hết lượt chấm). Có hai loại sản phẩm bán ra:

| Loại sản phẩm | Bản chất | Nguồn dữ liệu giá | Kênh thanh toán |
|---|---|---|---|
| **Gói premium (subscription)** | Gói dài hạn cộng cả lượt **practice** lẫn **mock test** + thời hạn sử dụng | API NineSpeak `GET /api/v1/premium-packages` (single source of truth, **không** fallback) | Chuyển khoản **QR** trong app (luồng `usePaymentOrderFlow`) |
| **Gói lẻ top-up lượt chấm** | Cộng nhanh vài lượt practice/mock test (3/6/10 lượt) | Định nghĩa tĩnh ở `shared/lib/top-up-packs.ts`, giá có thể override qua env `PRACTICE_TOPUP_*` | **Messenger** (mở link chat điền sẵn email) — *không* qua luồng QR |

Mục tiêu module:
- Hiển thị **bảng giá** gói premium ([PricingPage](../../../features/billing/pages/PricingPage.tsx) tại route `/pricing`).
- Điều phối **luồng đặt đơn thanh toán QR** cho subscription: tạo đơn → hiển thị QR + đếm ngược → poll trạng thái → cộng lượt khi `paid`.
- Cung cấp **bộ UI/hook dùng lại** để các module khác (Mock Test, Practice, Settings) tự dựng **paywall** của riêng họ khi người dùng hết lượt — qua một **CONTRACT công khai duy nhất**.

> ⚠️ **KHÔNG có hook `usePaywall`.** Đây là mô tả cũ hư cấu. Contract thật là barrel [features/billing/index.ts](../../../features/billing/index.ts). Mỗi peer tự dựng paywall của mình (vd `useMockExamQuotaPurchaseFlow`, `usePracticeQuotaPurchaseFlow`) **trên** các export của Billing.

---

## 2. Người dùng & vai trò

| Vai trò | Mô tả | Tương tác với Billing |
|---|---|---|
| **Học viên (end user)** | Người luyện IELTS Speaking | Xem bảng giá, mua gói premium qua QR, mua top-up qua Messenger, bị chặn paywall khi hết lượt và mua để mở khóa |
| **Đầu mối Billing (owner)** | Người/team sở hữu `features/billing/` | Phát triển/sửa module, giữ nguyên CONTRACT ở `index.ts` |
| **Đầu mối module peer** | Owner của Mock Test / Practice / Onboarding-Dashboard | **Chỉ** tiêu thụ Billing qua barrel `@/features/billing`; khi cần khả năng mới phải nhờ Billing export thêm |
| **Backend NineSpeak** | Hệ thống công ty (khác Firebase) | Cấp danh sách gói, tạo/đối soát đơn thanh toán, sinh mã QR, cập nhật lượt sau khi `paid` |

---

## 3. Phạm vi tính năng

### 3.1. Bảng giá / PricingPage
- Trang `/pricing` ([app/(shell)/pricing/page.tsx](../../../app/(shell)/pricing/page.tsx) → [PricingPage](../../../features/billing/pages/PricingPage.tsx)).
- Tải danh sách gói qua `usePremiumPackages`. Trạng thái: **loading** (spinner "Đang tải danh sách gói…"), **error** (thẻ đỏ + nút "Thử lại"), **success** (lưới thẻ giá `PricingPlanCards`).
- Nhấn "Mua ngay" trên một thẻ → mở luồng thanh toán QR cho gói đó.

### 3.2. Luồng đặt đơn thanh toán QR (payment order flow)
- Hook [usePaymentOrderFlow](../../../features/billing/hooks/usePaymentOrderFlow.ts) điều phối. UI hiển thị bằng [SubscriptionPaymentModals](../../../features/billing/components/pricing/SubscriptionPaymentModals.tsx).
- Tạo đơn → backend trả **mã QR + thông tin chuyển khoản (ngân hàng, số TK, chủ TK, nội dung CK) + hạn thanh toán (`expiredAt`)**.
- Modal QR hai cột: cột trái QR + đếm ngược + thông tin chuyển khoản (copy được); cột phải tên gói + mã đơn + tổng tiền.

### 3.3. Trạng thái thanh toán & polling
- Khi đang chờ (`awaiting`): tự **poll `getOrder` mỗi 3 giây** + đồng hồ đếm ngược theo `expiredAt`.
- Khi backend trả `paid`: dừng poll, hiển thị popup "Thanh toán thành công!", gọi `refreshAfterPaid` để rehydrate lượt, rồi **tự đóng sau 5 giây** (hoặc bấm nút).

### 3.4. Paywall overlay khi module khác hết lượt
- Billing **không** tự đặt paywall ở bất kỳ luồng nào. Module peer tự dựng paywall của họ và tái dùng các thành phần Billing:
  - [PricingPlanCards](../../../features/billing/components/pricing/PricingPlanCards.tsx) — lưới gói premium trong màn paywall.
  - [PurchaseStatusBackdrop](../../../features/billing/components/pricing/PurchaseStatusBackdrop.tsx) — nền mờ giả lập "báo cáo bị khóa" sau paywall.
  - [SubscriptionPaymentModals](../../../features/billing/components/pricing/SubscriptionPaymentModals.tsx) + [usePaymentOrderFlow](../../../features/billing/hooks/usePaymentOrderFlow.ts) — luồng QR khi chọn mua subscription tại paywall.

---

## 4. User journey

### 4.1. Mua gói subscription (từ trang bảng giá)
1. Vào `/pricing` → hệ thống tải gói qua `usePremiumPackages`.
2. Chọn gói → bấm "Mua ngay" → `usePaymentOrderFlow.open(premiumPackageId)`.
3. App tạo đơn (`creating`) → nhận QR (`awaiting`) → hiển thị QR + đếm ngược.
4. Người dùng quét QR & chuyển khoản. App **poll mỗi 3s**.
5. Backend xác nhận `paid` → popup thành công → `refreshProfile()` cộng lượt → tự đóng sau 5s.

### 4.2. Mua top-up (gói lẻ)
1. Tại màn paywall của Mock Test/Practice, người dùng chọn một gói top-up (3/6/10 lượt).
2. App **mở Messenger** với link điền sẵn email (`buildPrefilledMessengerCheckoutUrl`).
3. Người dùng chat & thanh toán thủ công qua kênh hỗ trợ.
4. Quay lại app, màn "purchase-status" cho phép bấm "đã thanh toán → chấm lại" để re-poll.
> Top-up **chưa có API tự động** → không đi qua luồng QR; do peer xử lý, Billing chỉ cung cấp `PricingPlanCards` cho phần subscription cùng màn.

### 4.3. Bị chặn paywall từ Mock Test / Practice
1. Người dùng hết lượt → khi chấm, hệ thống chặn (`grading_quota_exceeded` / HTTP `402` ở Practice; quota guard ở Mock Test).
2. Peer mở paywall của mình (`offer` stage), hiển thị `PricingPlanCards` + tùy chọn top-up.
3. Chọn gói premium → mở `usePaymentOrderFlow` (luồng QR) ngay trên nền paywall.
4. Thanh toán thành công → `refreshAfterPaid` cộng lượt → `onContinue` của peer **tự chấm lại bài đang dở** (không phải làm lại từ đầu).

---

## 5. Business rules

| # | Quy tắc | Chi tiết |
|---|---|---|
| BR-1 | **Nguồn gói subscription** | Lấy động từ API `GET /api/v1/premium-packages` (cần token). **Không fallback giá**: lỗi tải → chặn mua, báo lỗi (`state: "error"`, `plans: []`). Tránh tạo đơn sai giá. |
| BR-2 | **Nguồn gói top-up** | Định nghĩa tĩnh ở `shared/lib/top-up-packs.ts`; giá mặc định 79k/149k/229k, override qua env `PRACTICE_TOPUP_3_VND`/`_6_VND`/`_10_VND`. |
| BR-3 | **Các stage thanh toán** | `idle` → (`method` — *hiện bỏ qua*) → `creating` → `awaiting` → `paid` / `expired` / `error`. |
| BR-4 | **Polling** | Ở `awaiting`: poll `getOrder` **mỗi 3 giây**, tuần tự (không chồng request). Dừng khi `paid`/`expired`/`error` hoặc đếm ngược về 0. |
| BR-5 | **Hết hạn** | Backend trả `expired` **hoặc** đồng hồ (`expiredAt`) về 0 → `expired`. Muốn mua lại phải **chọn gói từ đầu** (đóng modal, mở lại). |
| BR-6 | **Cộng lượt** | Chỉ khi `paid`: gọi `refreshAfterPaid` (do caller truyền) → `refreshProfile()` của AuthProvider → lượt mới hiển thị ngay. |
| BR-7 | **Tự đóng popup thành công** | Sau khi `refreshAfterPaid` resolve, đếm ngược **5 giây** rồi chạy `onContinue` (đảm bảo chỉ chạy 1 lần dù bấm OK trùng với đếm ngược). |
| BR-8 | **QR lỗi tải** | Ảnh QR ngoài (qr.sepay.vn). Lỗi → hiện nút "Tải lại"; **KHÔNG** hiển thị thông tin chuyển khoản thủ công thay thế (tránh chuyển nhầm). |
| BR-9 | **Chỉ một phương thức** | Hiện chỉ QR (`PaymentMethodId = "qr"`); bước chọn phương thức (`method`) bị bỏ qua (`ENABLE_METHOD_SELECTION = false`), giữ code cho tương lai. |
| BR-10 | **Whitelist gói hiển thị** | UI **chỉ** hiển thị các gói có `premiumPackageId` thuộc whitelist `{pro_1m, pro_3m, pro_6m, pro_12m}`. ID ngoài tập này (gói lạ/nội bộ) bị coi là "mồ côi" → **ẩn** (`visibleInPricing=false`). Áp dụng cho **mọi** bề mặt: `/pricing`, paywall Mock Test/Practice, và danh sách nâng cấp ở Settings. |
| BR-11 | **Ẩn gói 3 tháng** | `pro_3m` (và biến thể `pro_3m_test`) đặt `visibleInPricing=false` → **không** hiện ở bảng giá/paywall/Settings; chỉ hiện 1/6/12 tháng. Gói vẫn được resolve để hiển thị tên nếu user đang sở hữu. |
| BR-12 | **Biến thể `_test` theo môi trường** | Môi trường được **suy ra từ backend đang trỏ tới** (`NEXT_PUBLIC_NINESPEAK_API_BASE_URL` chứa "staging" → staging) — không thêm biến env riêng. Trên **staging**, backend trả thêm ID hậu tố `_test` (vd `pro_6m_test`) → được canonical hóa về ID gốc để dùng đúng trình bày + whitelist. Trên **production**, hậu tố `_test` **không** được nhận → ID `_test` lọt vào sẽ bị ẩn. |

---

## 6. Ràng buộc giữa module (rất quan trọng)

Billing là **CONTRACT cho module khác**. Quy tắc đã được **ESLint enforce** ([eslint.config.mjs](../../../eslint.config.mjs)):

- Module khác (`mock-test`, `practice`, `onboarding-dashboard`) **CHỈ** được `import ... from "@/features/billing"` — **cấm** chạm path nội bộ (`@/features/billing/hooks/*`, `@/features/billing/components/*`, `@/features/billing/pages/*`).
- Billing **KHÔNG** được import bất kỳ module tính năng nào khác. Chỉ được import `shared/`.
- Khi peer cần khả năng mới → **thêm export tại [index.ts](../../../features/billing/index.ts)**, không để peer chạm file nội bộ. Nhờ vậy owner Billing tự do refactor nội bộ miễn giữ contract.

Người tiêu thụ hiện tại (xem chi tiết ở [TECH.md](./TECH.md)): Mock Test (paywall thi thử), Practice (paywall Part 1/2/3), Onboarding-Dashboard (Settings), app `/pricing`.

---

## 7. Edge cases

| Tình huống | Hành vi mong đợi |
|---|---|
| **Đơn hết hạn** (timeout/backend `expired`) | Báo "Đơn thanh toán đã hết hạn", hủy đơn, dừng poll, nút "Chọn lại gói". **Không** cộng lượt. |
| **Lỗi tạo/poll đơn** (404 / timeout / 5xx / mất mạng) | Stage `error`, thông điệp tiếng Việt (vd "Không tìm thấy gói hoặc đơn hàng", "Yêu cầu thanh toán bị quá thời gian chờ"), nút "Thử lại". **Không** cộng lượt. |
| **Hủy giao dịch** (bấm X / "Hủy giao dịch") | Đóng modal, `reset` về `idle`. Đơn dở dang không ảnh hưởng lượt. |
| **Hết phiên đăng nhập** (không có token) | Báo "Phiên đăng nhập không còn hợp lệ. Vui lòng đăng nhập lại." (cả tạo đơn lẫn tải gói). |
| **Không tải được bảng giá** | Chặn mua, hiện lỗi + "Thử lại". Không hiện giá cũ/giá đoán. |
| **QR không tải** | Nút tải lại QR; không lộ thông tin chuyển khoản thủ công thay thế. |
| **`refreshAfterPaid` lỗi** | Bỏ qua lỗi refresh, **vẫn** cho người dùng tiếp tục (không chặn hậu thanh toán). |

---

## 8. Out of scope

- **Kiểm tra/trừ lượt khi chấm** (quota guard, ledger Firestore): thuộc Mock Test / Practice (`mocktest_scoring_charges`, `practice_scoring_charges`) — xem [luot-cham-diem-cho-product](../../luot-cham-diem-cho-product.md). Billing chỉ **cộng lượt** sau thanh toán (gián tiếp qua `refreshProfile`).
- **Đối soát "trừ lỗi"** (reconciliation) — vận hành nội bộ, ngoài app.
- **Logic paywall cụ thể của từng module** (khi nào chặn, chấm lại bài nào) — do peer tự dựng trên contract Billing.
- **Top-up thanh toán tự động** — hiện qua Messenger thủ công, chưa có API.
- **Cấp lượt miễn phí / khuyến mãi** — do backend NineSpeak quyết định, app chỉ hiển thị.
</content>
</invoke>
