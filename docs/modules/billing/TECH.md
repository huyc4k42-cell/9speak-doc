# Billing & Paywall — Đặc tả kỹ thuật (TECH)

> Module: `features/billing/` — **Thanh toán & Paywall**
> Tài liệu nghiệp vụ đi kèm: [PRD.md](./PRD.md).
> Tài liệu nền tảng: [README-modules](../../README-modules.md) · [env-and-modules](../../env-and-modules.md).

---

## 1. Kiến trúc & sơ đồ thư mục

Billing là module **client-side thuần** (mọi file `"use client"`). Không có `server/` riêng — mọi gọi backend đi qua client API của `shared/` (NineSpeak). Không có route handler `app/api/*` riêng cho thanh toán: app gọi **trực tiếp** backend NineSpeak.

```
features/billing/
├── index.ts                                   # PUBLIC API (CONTRACT) — bề mặt duy nhất cho peer
├── AGENTS.md                                   # context cục bộ cho AI agent
├── hooks/
│   ├── usePaymentOrderFlow.ts                  # điều phối luồng thanh toán QR (stages, poll, đếm ngược)
│   └── usePremiumPackages.ts                   # tải danh sách gói subscription từ API
├── components/pricing/
│   ├── PricingPlanCards.tsx                    # lưới thẻ giá gói premium
│   ├── SubscriptionPaymentModals.tsx          # modal QR / creating / paid / expired / error
│   ├── PurchaseReportBlurSurface.tsx          # skeleton "Performance Report" blur — nền dùng chung cho các màn paywall
│   └── PurchaseStatusBackdrop.tsx              # nền mờ "báo cáo bị khóa" cho màn paywall (dùng PurchaseReportBlurSurface)
└── pages/
    └── PricingPage.tsx                         # trang /pricing (ghép hooks + components)
```

Phụ thuộc ra ngoài (đều thuộc `shared/` hoặc hạ tầng repo):
- API: [shared/lib/api/company-api-client.ts](../../../features/shared/lib/api/company-api-client.ts) → `paymentsApi`, `premiumPackagesApi`, `NineSpeakApiError` (re-export từ [ninespeak-client.ts](../../../features/shared/lib/ninespeak-client.ts)).
- Auth: [shared/providers/AuthProvider.tsx](../../../features/shared/providers/AuthProvider.tsx) → `useAuth` (token, `refreshProfile`).
- Pricing model: [shared/lib/subscription-plans.ts](../../../features/shared/lib/subscription-plans.ts).
- Types: [shared/types/ninespeak-api.ts](../../../features/shared/types/ninespeak-api.ts).
- UI kit: [components/ui](../../../components/ui), `lib/utils` (`cn`), `lucide-react`, `sonner`.

---

## 2. Public API CONTRACT — [features/billing/index.ts](../../../features/billing/index.ts)

Đây là **bề mặt công khai duy nhất**. Liệt kê chính xác các export và ai dùng:

| Export | Kiểu | Mô tả | Người tiêu thụ (qua barrel `@/features/billing`) |
|---|---|---|---|
| `usePaymentOrderFlow` | hook | Điều phối luồng thanh toán QR subscription | [PricingPage](../../../features/billing/pages/PricingPage.tsx) (nội bộ), [useMockExamQuotaPurchaseFlow](../../../features/mock-test/hooks/useMockExamQuotaPurchaseFlow.ts), [usePracticeQuotaPurchaseFlow](../../../features/practice/hooks/usePracticeQuotaPurchaseFlow.ts), [SettingsPage](../../../features/onboarding-dashboard/pages/SettingsPage.tsx) |
| `PaymentMethodId` | type | `"qr"` | (qua `usePaymentOrderFlow`) |
| `PaymentFlowStage` | type | `"idle" \| "method" \| "creating" \| "awaiting" \| "paid" \| "expired" \| "error"` | (qua `usePaymentOrderFlow`) |
| `usePremiumPackages` | hook | Tải danh sách gói subscription từ API | PricingPage (nội bộ), [MockExamFull](../../../features/mock-test/pages/exams/MockExamFull.tsx), [PracticePart1/2/3](../../../features/practice/pages/PracticePart1.tsx), SettingsPage |
| `PricingPlanCards` | component | Lưới thẻ giá gói premium | PricingPage (nội bộ), [MockExamQuotaPaywallScreen](../../../features/mock-test/components/exams/MockExamQuotaPaywallScreen.tsx) |
| `SubscriptionPaymentModals` | component | Modal thanh toán QR (mọi stage) | PricingPage, MockExamFull, PracticePart1/2/3, SettingsPage |
| `PurchaseStatusBackdrop` | component | Nền mờ paywall "báo cáo bị khóa" (dùng `PurchaseReportBlurSurface` bên trong) | [MockExamPurchaseStatusScreen](../../../features/mock-test/components/exams/MockExamPurchaseStatusScreen.tsx) |
| `PurchaseReportBlurSurface` | component | Skeleton "Performance Report" blur (absolute inset-0) — đặt phía sau màn paywall để tạo cảm giác "đã có kết quả nhưng bị khóa" | [MockExamQuotaPaywallScreen](../../../features/mock-test/components/exams/MockExamQuotaPaywallScreen.tsx), `PurchaseStatusBackdrop` (nội bộ) |
| `PricingPage` | component | Trang `/pricing` | [app/(shell)/pricing/page.tsx](../../../app/(shell)/pricing/page.tsx) |

> ⚠️ **KHÔNG có `usePaywall`** (mô tả cũ hư cấu). Mỗi peer dựng paywall riêng trên các export trên.

---

## 3. Routes & API

### 3.1. Route app
| Route | File | Ghi chú |
|---|---|---|
| `/pricing` | [app/(shell)/pricing/page.tsx](../../../app/(shell)/pricing/page.tsx) | Wrapper mỏng: `import { PricingPage as PricingView } from "@/features/billing"`. Nằm trong route group `(shell)` (có sidebar/topbar). |

Không có route trong `(focus)`/`(focus-practice)` cho Billing; paywall được peer render **inline** trong luồng làm bài của họ (overlay), không phải route riêng.

### 3.2. API thanh toán / premium-packages
Gọi **trực tiếp** backend NineSpeak (không qua `app/api/*`). Định nghĩa tại [ninespeak-client.ts](../../../features/shared/lib/ninespeak-client.ts), re-export qua [company-api-client.ts](../../../features/shared/lib/api/company-api-client.ts):

| Client | Method | Endpoint NineSpeak | Dùng bởi |
|---|---|---|---|
| `premiumPackagesApi` | `list(token)` | `GET /api/v1/premium-packages` | `usePremiumPackages` |
| `paymentsApi` | `createOrder({ premiumPackageId }, token)` | `POST /api/v1/payments/orders` | `usePaymentOrderFlow` |
| `paymentsApi` | `getOrder(orderCode, token)` | `GET /api/v1/payments/orders/{orderCode}` | `usePaymentOrderFlow` (poll) |
| `paymentsApi` | `listOrders(...)` | `GET /api/v1/payments/orders` | (chưa dùng trong Billing) |

Type liên quan ([ninespeak-api.ts](../../../features/shared/types/ninespeak-api.ts)):
- `NineSpeakPremiumPackage` — `{ premiumPackageId, currency, price, practiceCredits, mocktestCredits, duration, durationUnit, ... }`.
- `NineSpeakPaymentOrder` — `{ orderCode, premiumPackageId, amount, currency, status, bankAccount, bankName, accountHolder?, transferContent, qrUrl, expiredAt, paidAt?, ... }`.
- `NineSpeakPaymentOrderStatus` — `"pending" | "paid" | "expired" | (string & {})`.

---

## 4. Hooks

### 4.1. [usePaymentOrderFlow](../../../features/billing/hooks/usePaymentOrderFlow.ts)
Điều phối toàn bộ luồng thanh toán QR subscription.

**Options** (`UsePaymentOrderFlowOptions`):
| Option | Kiểu | Vai trò |
|---|---|---|
| `refreshAfterPaid?` | `() => Promise<unknown> \| unknown` | Gọi khi đơn `paid` để rehydrate quota (thường `() => refreshProfile()`). Nút OK **chờ** tới khi resolve. |
| `onContinue?` | `(order: NineSpeakPaymentOrder \| null) => void \| Promise<void>` | Chạy tiếp theo ngữ cảnh khi đóng popup thành công (bấm OK hoặc tự đóng 5s). Ở paywall: dùng để **tự chấm lại** bài dở. |
| `continueLabel?` | `string` | Nhãn nút popup thành công (mặc định "Bắt đầu luyện tập"). |

**Stages** (`PaymentFlowStage`): `idle` → (`method` *bỏ qua*) → `creating` → `awaiting` → `paid` / `expired` / `error`.
- Hằng số: `POLL_INTERVAL_MS = 3000`, `SUCCESS_AUTO_CLOSE_SECONDS = 5`, `ENABLE_METHOD_SELECTION = false`.
- **`awaiting`**: effect chạy đồng hồ đếm ngược 1s (theo `order.expiredAt`) + poll `getOrder` tuần tự mỗi 3s. `paid` → dừng + `setStage("paid")`; `expired` (hoặc đếm về 0) → `expired`; lỗi → `error`.
- **`paid`**: effect gọi `refreshAfterPaid()` (set `isCreditUpdating`), xong thì mở `okCountdown = 5`; effect khác đếm 5→0 rồi `confirmContinue()`. `continueHandledRef` chặn `onContinue` chạy 2 lần.

**Giá trị trả về** (chọn lọc): `stage`, `selectedPackageId`, `paymentMethod`/`setPaymentMethod`, `order`, `secondsLeft`, `errorMessage`, `isCreditUpdating`, `okCountdown`, `continueLabel`, `isActive`, `open(premiumPackageId)`, `confirmMethod()`, `backToMethod()`, `close()`, `confirmContinue()`.

### 4.2. [usePremiumPackages](../../../features/billing/hooks/usePremiumPackages.ts)
Tải gói subscription. Nguồn: `premiumPackagesApi.list(token)` → `resolveSubscriptionPlans(...)`.
- **Không fallback**: lỗi/không token → `state: "error"`, `plans: []` (block luồng mua).
- Trả về: `state`, `plans: ResolvedSubscriptionPlan[]`, `errorMessage`, `reload()`, `isLoading`, `isError`.
- `plans` là **toàn bộ** gói đã resolve (kể cả gói ẩn). Mọi bề mặt UI phải lọc qua `getVisibleResolvedPlans(plans)` trước khi render — xem 4.3.

### 4.3. Whitelist & hiển thị gói ([subscription-plans.ts](../../../features/shared/lib/subscription-plans.ts))
`resolveSubscriptionPlan(pkg)` tra metadata trình bày theo `canonicalizePlanId(pkg.premiumPackageId)`:
- **Whitelist** `KNOWN_SUBSCRIPTION_PLAN_IDS` = `{pro_1m, pro_3m, pro_6m, pro_12m}` (key của map presentation). ID ngoài tập → "mồ côi" (`isOrphan=true`) → `buildFallbackPresentation` đặt `visibleInPricing=false` (ẩn, nhưng vẫn resolve để hiện tên gói user đang sở hữu).
- **`canonicalizePlanId`**: trên staging (suy ra từ `NEXT_PUBLIC_NINESPEAK_API_BASE_URL` chứa "staging" — xem `feature-flags.getAppEnvironment`/`isStagingEnvironment`) bỏ hậu tố `_test` (`pro_6m_test` → `pro_6m`); production giữ nguyên nên `_test` lọt vào sẽ thành mồ côi → ẩn. **`id`/`premiumPackageId` giữ nguyên giá trị gốc** để tạo đơn thanh toán đúng.
- **`getVisibleResolvedPlans(plans)`** lọc `visibleInPricing` — `pro_3m` đặt `false` nên không hiện. Áp dụng ở: `PricingPage`, paywall Mock Test (`MockExamFull`), paywall Practice (Part 1/2/3) và danh sách nâng cấp Settings (`SettingsPage`).

---

## 5. Components pricing

| Component | Vai trò | Props chính |
|---|---|---|
| [PricingPlanCards](../../../features/billing/components/pricing/PricingPlanCards.tsx) | Lưới thẻ giá (theme light/featured/dark, density default/compact). Mỗi thẻ có tên/mô tả/giá/feature/nút mua. | `plans`, `actionLabel`, `onSelectPlan(plan)`, `density` |
| [SubscriptionPaymentModals](../../../features/billing/components/pricing/SubscriptionPaymentModals.tsx) | Render đúng UI theo `flow.stage`: `method` (chọn phương thức — ẩn), `creating` (spinner), `awaiting` (QR 2 cột + đếm ngược + copy thông tin CK), `paid` (thành công + đếm tự đóng), `expired`, `error`. `QrImage` xử lý lỗi tải ảnh QR. | `flow` (= `ReturnType<typeof usePaymentOrderFlow>`), `plan` |
| [PurchaseStatusBackdrop](../../../features/billing/components/pricing/PurchaseStatusBackdrop.tsx) | Layout overlay cho màn paywall (overlay `fixed inset-0`): nền report blur (`PurchaseReportBlurSurface`) phủ TRỌN vùng cuộn + slot children/footer; heading/description và nút back đều **tùy chọn** (chỉ render khi truyền). | `heading?`, `description?`, `backLabel?`, `onBack?`, `backDisabled?`, `children`, `footer?` |
| [PurchaseReportBlurSurface](../../../features/billing/components/pricing/PurchaseReportBlurSurface.tsx) | Skeleton "Performance Report" blur (`absolute inset-0`, `pointer-events-none`). Đặt trong một ancestor `relative` để phủ kín vùng nền paywall. Là mockup tĩnh — KHÔNG phải report thật (report thật chỉ có sau khi mua + chấm điểm). | (không props) |

---

## 6. Phụ thuộc shared (import cụ thể) & ràng buộc boundary

### 6.1. Import shared được phép
```ts
// hooks/usePaymentOrderFlow.ts
import { NineSpeakApiError, paymentsApi } from "@/features/shared/lib/api/company-api-client";
import { useAuth } from "@/features/shared/providers/AuthProvider";
import type { NineSpeakPaymentOrder } from "@/features/shared/types/ninespeak-api";

// hooks/usePremiumPackages.ts
import { NineSpeakApiError, premiumPackagesApi } from "@/features/shared/lib/api/company-api-client";
import { resolveSubscriptionPlans, type ResolvedSubscriptionPlan } from "@/features/shared/lib/subscription-plans";
import { useAuth } from "@/features/shared/providers/AuthProvider";

// components + page: subscription-plans (getVisibleResolvedPlans, formatPlanPrice, ResolvedSubscriptionPlan), components/ui, lib/utils
```

### 6.2. Ràng buộc boundary (ESLint enforce — [eslint.config.mjs](../../../eslint.config.mjs))
- `features/billing/**` áp `blockModule("mock-test")`, `blockModule("practice")`, `blockModule("onboarding-dashboard")` → **Billing KHÔNG import module tính năng khác**. Chỉ được `shared/` + hạ tầng repo.
- Mọi module khác áp `onlyBarrel("billing")` → **chỉ** `import ... from "@/features/billing"`, cấm path nội bộ.
- Vi phạm → `npm run lint` fail kèm thông điệp trỏ về [README-modules](../../README-modules.md).

---

## 7. Env liên quan

| Biến | Đọc tại | Ảnh hưởng | Ghi chú |
|---|---|---|---|
| `PRACTICE_TOPUP_3_VND` / `_6_VND` / `_10_VND` | `shared/lib/top-up-packs.ts` (qua `priceEnvKey`, đọc động) | billing (UI top-up trên paywall) + practice/mock-test | Override giá gói lẻ; mặc định 79k/149k/229k. |
| (gói subscription) | — | billing | **Không qua env** — lấy từ API `GET /api/v1/premium-packages`. |
| `NEXT_PUBLIC_NINESPEAK_API_BASE_URL`, `NEXT_PUBLIC_NINESPEAK_API_TIMEOUT_MS` | shared (`ninespeak-client`) | tất cả | Endpoint/timeout backend NineSpeak (gián tiếp ảnh hưởng Billing). |

> Billing **không** đọc `process.env` trực tiếp. Top-up env được tiêu thụ qua `shared`. Chi tiết: [env-and-modules](../../env-and-modules.md).

---

## 8. Gotchas & hướng phát triển

**Gotchas**
- **Top-up ≠ luồng QR.** Subscription đi qua `usePaymentOrderFlow` (QR + poll); top-up đi qua **Messenger** (`buildPrefilledMessengerCheckoutUrl` trong [shared/lib/messenger-checkout.ts](../../../features/shared/lib/messenger-checkout.ts)), do **peer** xử lý — Billing chỉ cấp `PricingPlanCards` cho phần subscription cùng màn. (Top-up có thể đang tạm ẩn ở một số paywall.)
- **`refreshAfterPaid` do caller truyền**, không cố định `refreshProfile`. Nếu cộng lượt không phản ánh, kiểm tra callback caller chứ không phải Billing.
- **Bước `method` bị bỏ qua** (`ENABLE_METHOD_SELECTION = false`); code giữ lại cho tương lai khi có thêm phương thức.
- **Không fallback giá** ở `usePremiumPackages` là cố ý — đừng thêm giá mặc định (tránh tạo đơn sai giá).
- **`continueHandledRef`** chặn `onContinue` chạy đúp (bấm OK trùng đếm ngược). Đừng bỏ khi refactor.
- **Ảnh QR ngoài** (qr.sepay.vn): lỗi tải → nút tải lại, **không** lộ thông tin CK thủ công.

**Thêm khả năng mới cho peer**
1. Triển khai nội bộ trong `features/billing/` (hook/component/type mới).
2. **Export tại [index.ts](../../../features/billing/index.ts)** — đây là cách duy nhất để peer dùng. Không để peer chạm file nội bộ.
3. Giữ tương thích ngược các export đang có (peer phụ thuộc).

**Nơi sửa / debug**
| Triệu chứng | Nơi xem |
|---|---|
| Bảng giá trống/lỗi tải | `usePremiumPackages`, API `premiumPackagesApi.list`, token AuthProvider |
| Đơn không tạo được / poll lỗi | `usePaymentOrderFlow.createOrderFor` + effect `awaiting`, `paymentsApi.createOrder/getOrder` |
| Cộng lượt không phản ánh | callback `refreshAfterPaid` của caller → `refreshProfile` (AuthProvider) |
| QR/đếm ngược/copy sai | `SubscriptionPaymentModals` (`QrImage`, `formatCountdown`) |
| Paywall không tự chấm lại | `onContinue` của peer (vd `usePracticeQuotaPurchaseFlow`), không phải Billing |
| Lint báo lỗi import | [eslint.config.mjs](../../../eslint.config.mjs) — kiểm tra barrel/boundary |
</content>
