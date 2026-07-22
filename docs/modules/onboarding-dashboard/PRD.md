# PRD — Module Onboarding / Dashboard (Shell + BFF)

> Tài liệu nghiệp vụ chi tiết cho module `features/onboarding-dashboard/`. Dành cho AI agent và đầu mối phụ trách đọc để hiểu **đầy đủ phạm vi, luồng và ràng buộc** trước khi phát triển/sửa lỗi.
>
> Tài liệu kỹ thuật đi kèm: [TECH.md](./TECH.md). Bối cảnh module hóa: [../../README-modules.md](../../README-modules.md), [../../env-and-modules.md](../../env-and-modules.md). Context AI cục bộ: [../../../features/onboarding-dashboard/AGENTS.md](../../../features/onboarding-dashboard/AGENTS.md).

---

## 1. Tổng quan & mục tiêu

Module này là **composition root / shell** của toàn bộ ứng dụng IELTS Speaking. Khác với các module tính năng (`mock-test`, `practice`, `billing`) tập trung vào một nghiệp vụ, module này chịu trách nhiệm **lắp ráp khung hệ thống** và cung cấp các trang "ngoài phòng thi":

| Nhóm chức năng | Vai trò |
|---|---|
| **System Shell** | Khung sidebar + topbar ([AppShell](../../../features/onboarding-dashboard/layouts/AppShell.tsx)) bọc mọi trang trong route group `(shell)`; điều hướng, hiển thị hồ sơ + lượt chấm, guard đăng nhập/onboarding. |
| **Onboarding** | Khảo sát mục tiêu/trình độ lần đầu, lưu vào profile, mở khóa vào app. |
| **Dashboard** | Trang trung tâm sau đăng nhập: band hiện tại, mục tiêu, đề xuất làm bài. |
| **History & Performance** | Lịch sử **hợp nhất** (mock + practice) và biểu đồ hiệu suất theo thời gian. |
| **Settings** | Thông tin gói đăng ký + thanh toán (qua billing), AI settings, prompt lab, logs, debug. |
| **Auth** | Login / Register / Forgot password. |
| **BFF (Backend-for-Frontend)** | Tầng server `/api/app/*`: xác thực Firebase, proxy lên backend công ty (NineSpeak) hoặc giả lập bằng mock backend khi dev. |

**Mục tiêu sản phẩm:** cho người học một "ngôi nhà" thống nhất để bắt đầu (onboarding), nắm tình hình (dashboard/history/performance), quản lý gói (settings) và điều hướng sang các phòng luyện/thi (do module khác đảm nhận).

---

## 2. Người dùng & vai trò

| Vai trò | Mô tả | Khác biệt trong module |
|---|---|---|
| **Khách (chưa đăng nhập)** | Chỉ truy cập được nhóm `(auth)`: login/register/forgot/onboarding. | AppShell guard sẽ `router.replace("/login")` nếu vào trang `(shell)`. |
| **Người dùng mới (chưa onboard)** | Đã đăng nhập nhưng `isOnboarded = false`. | AppShell ép redirect về `/onboarding/flow` cho tới khi hoàn tất hoặc bỏ qua. |
| **Người dùng thường** | Đã onboard, dùng app bình thường. | Thấy điều hướng chính (Trang chủ, Lịch sử, Luyện tập, Thi thử) + Bảng giá, Cài đặt. |
| **Admin** (`profile.userType === "admin"`) | Người vận hành/nội bộ. | Thấy thêm menu **AI Settings**, badge model AI ở topbar, khối build metadata (double-click để mở Alpha logs). Các trang prompt-lab/logs/backend-auth-debug phục vụ nhóm này. |

---

## 3. Phạm vi tính năng chi tiết

### 3.1 Onboarding flow
- Trang: [Onboarding.tsx](../../../features/onboarding-dashboard/pages/Onboarding.tsx) — export `OnboardingFlow`, cùng các shim `OnboardingGoal`, `OnboardingMicCheck` (đều redirect về `/onboarding/flow`).
- Route thực tế chỉ còn một bước gộp: `app/(auth)/onboarding/flow`. Các thư mục `goal`, `mic-check` vẫn tồn tại nhưng chuyển hướng về `flow`.
- Bộ câu hỏi **tải động** từ BFF (`fetchOnboardingDefinition`, định danh `id = 1`): lý do học, band mục tiêu + deadline (sub-question cùng màn), khó khăn (multi-select), phương pháp ưa thích (multi-select), v.v.
- Khi hoàn tất: gọi `updateProfile(...)` (qua AuthProvider) lưu `currentBand`, `targetBand`, `onboardingData.completed = true` rồi điều hướng `/dashboard`.
- Có nút **Bỏ qua**: ghi marker `onboardingData.skipped = true` (không gửi band) để vượt guard; band sẽ được hỏi lại sau qua popup khi dùng tính năng AI.
- Phát đầy đủ analytics: `onboarding_screen_viewed`, `click_onboarding_button` (kèm `is_skip`), `complete_onboarding` (kèm `time_to_completed`).

### 3.2 Dashboard
- Trang: [Dashboard.tsx](../../../features/onboarding-dashboard/pages/Dashboard.tsx).
- Hiển thị band hiện tại / mục tiêu (lấy từ `profile` của AuthProvider — **không** tự gọi `/api/app/user/profile`), nút đặt/sửa band mục tiêu (tái dùng `BandScorePromptDialog` của shared), đề xuất làm bài nhanh, link sang practice/mock/history.
- Tích hợp khảo sát phản hồi (`FeedbackSurveyProvider` shared): hiển thị survey đã lên lịch sau khi user thoát/hủy mock test.

### 3.3 History (lịch sử hợp nhất)
- Trang: [History.tsx](../../../features/onboarding-dashboard/pages/History.tsx).
- Gộp dữ liệu **Practice + Mock Test** qua [shared/lib/history-client.ts](../../../features/shared/lib/history-client.ts) (`listHydratedHistoryEntries`): truy vấn các endpoint NineSpeak (practice sessions/results + mocktest results/analyses), chuẩn hóa về `HydratedHistoryEntry`, sắp theo `createdAt` giảm dần, nhóm theo tháng.
- Mỗi mục dẫn link sang phòng tương ứng (vd `/practice/part-1?...`, `/report/...`).

### 3.4 Performance / Hiệu suất
- Trang: [Performance.tsx](../../../features/onboarding-dashboard/pages/Performance.tsx) (route `/hieu-suat`).
- Dùng `fetchPerformance(range)` (BFF) + `listHydratedRecentSessions` (history-client) để vẽ tổng quan band, delta, theo từng tiêu chí (FC/LR/GRA/P), focus area và recommendation, theo dải thời gian (`7day`/`30day`/...).

### 3.5 Settings
- **Trang chính** [SettingsPage.tsx](../../../features/onboarding-dashboard/pages/SettingsPage.tsx): thông tin gói đăng ký, mức sử dụng (lượt AI Speaking / Mock Test), nâng cấp gói. **Thanh toán đi qua barrel billing** `@/features/billing` (`usePaymentOrderFlow`, `usePremiumPackages`, `SubscriptionPaymentModals`). Sau khi thanh toán xong gọi `refreshProfile()` để cập nhật lượt.
- **AI Settings** [AiScoringSettingsPage.tsx](../../../features/onboarding-dashboard/pages/AiScoringSettingsPage.tsx) (route `/ai-settings`, chỉ admin trong nav): chọn engine/model chấm điểm cho phiên.
- **Prompt Lab** [PromptOverrideLabPage.tsx](../../../features/onboarding-dashboard/pages/PromptOverrideLabPage.tsx) (route `/settings/prompt-lab`): bọc `PromptOverrideWorkbench` của shared để thử/ghi đè prompt.
- **Logs** [AlphaLogs.tsx](../../../features/onboarding-dashboard/pages/AlphaLogs.tsx) (route `/settings/logs`): xem alpha log gộp (in-memory + buffer + file persist).
- **Backend Auth Debug** [BackendAuthDebugPage.tsx](../../../features/onboarding-dashboard/pages/BackendAuthDebugPage.tsx) (route `/settings/backend-auth-debug` và `/debug/backend-auth`): công cụ kiểm tra token/backend.
- **Resources** [Resources.tsx](../../../features/onboarding-dashboard/pages/Resources.tsx) và **Community** [Community.tsx](../../../features/onboarding-dashboard/pages/Community.tsx): trang nội dung phụ trợ.

### 3.6 Auth (login / register / forgot)
- Tất cả nằm trong một file [Auth.tsx](../../../features/onboarding-dashboard/pages/Auth.tsx) export `Login`, `Register`, `ForgotPassword`.
- Dùng Firebase Auth qua shared (`auth-browser`, AuthProvider). Phát analytics login (`login_screen_viewed`, `click_login_button`, `login_success`/`login_failed`, alias user).

---

## 4. User journey

### 4.1 Onboarding lần đầu
1. Khách → `/login` → đăng nhập Google/email (Auth.tsx).
2. AuthProvider phát hiện đã auth nhưng `isOnboarded = false` → AppShell guard redirect `/onboarding/flow`.
3. OnboardingFlow tải bộ câu hỏi từ BFF, user trả lời từng bước (có validation, có nút Bỏ qua).
4. Hoàn tất → `updateProfile` lưu profile → redirect `/dashboard`.

### 4.2 Xem dashboard / history / performance
1. User đã onboard vào `/dashboard`: thấy band, mục tiêu, đề xuất.
2. Vào `/history`: history-client gộp practice + mock, hiển thị theo tháng, click mở report/phòng tương ứng.
3. Vào `/hieu-suat`: biểu đồ theo tiêu chí + đề xuất theo dải thời gian.

### 4.3 Cập nhật settings / thanh toán
1. Vào `/settings`: xem gói + mức sử dụng.
2. Bấm nâng cấp → modal billing (`usePaymentOrderFlow`) → thanh toán.
3. Sau khi paid → `refreshProfile()` cập nhật lượt/gói trên header & trang.

---

## 5. Business rules

1. **Chế độ BFF (mock vs company/NineSpeak):** mọi handler `/api/app/*` kiểm tra `getAppBackendMode()`. Nếu `company` → **proxy** thẳng request lên backend công ty (NineSpeak/Railway) qua `proxyCompanyApiRequest`. Nếu không → dùng **mock backend** in-memory ([app-backend-mock.ts](../../../features/onboarding-dashboard/server/app-backend-mock.ts)) để dev/demo không cần backend thật. Cơ chế chọn xem [TECH.md](./TECH.md#4-bff--cơ-chế-chọn-backend).
2. **Bearer-token Firebase:** mọi request BFF phải có header `Authorization: Bearer <Firebase ID token>`. Server gọi Firebase Admin verify, lấy `uid`, đồng thời áp **rate limit** theo `uid`. Thiếu/không hợp lệ → 401.
3. **App-data API:** client tương tác BFF qua [shared/lib/app-data-client.ts](../../../features/shared/lib/app-data-client.ts) và [shared/lib/api/app-bff-client.ts](../../../features/shared/lib/api/app-bff-client.ts) (đã chuyển về shared) — không tự fetch tay.
4. **Quota rehydrate:** mỗi khi lượt chấm thay đổi (nộp bài practice/mock thành công, mua gói billing thành công), module phát sinh hành động phải gọi `refreshProfile()` từ AuthProvider để header AppShell hiển thị đúng lượt còn lại.
5. **Guard truy cập:** AppShell tự enforce — chưa auth → `/login`; auth nhưng chưa onboard → `/onboarding/flow`. Không cần guard riêng ở từng trang `(shell)`.

---

## 6. Ràng buộc (Constraints)

- **Composition root:** module được phép **lắp ráp / phụ thuộc module tính năng khác**, nhưng **CHỈ qua barrel** `@/features/<module>`:
  - `app-backend-mock` dùng `@/features/mock-test` (helper báo cáo isomorphic: `buildOverallCriteria`, `isMockExamReportReady`).
  - `SettingsPage` dùng `@/features/billing` (payment flow + modals + packages).
  - **Cấm** đụng path nội bộ của peer (`@/features/mock-test/server/...`) — ESLint chặn (xem [TECH.md](./TECH.md#5-phụ-thuộc--ranh-giới)).
- **Không module peer nào được import ngược** vào `onboarding-dashboard`. Hạ tầng dùng chung (AuthProvider, history-client, app-data-client, providers) đã nằm ở `shared/`.
- **`index.ts` không export gì cho peer** — không có public API hướng peer (app/ import trực tiếp page/layout/handler).
- **Không tự đọc `process.env`** cho cấu hình nghiệp vụ (trừ BFF server: `COMPANY_API_BASE_URL`, `ALPHA_LOG_DIR`, `VERCEL`); chế độ backend đọc qua `getAppBackendMode()` của shared.

---

## 7. Edge cases

- **Token hết hạn / thiếu:** BFF trả 401 với message tiếng Việt; client cần re-auth.
- **Company mode nhưng thiếu `COMPANY_API_BASE_URL`:** proxy ném lỗi "Thiếu COMPANY_API_BASE_URL...".
- **Mock mode:** dữ liệu là in-memory + seed → mất khi restart server; chỉ dùng dev/demo.
- **Onboarding API trả rỗng:** OnboardingFlow báo "Không thể tải câu hỏi onboarding từ API" thay vì hiển thị form trống.
- **History/Performance khi backend lỗi:** history-client trả mảng rỗng/lỗi → trang hiển thị trạng thái trống, không crash.
- **Mock exam catalog ở segmented flow:** BFF gọi `assertAzureSpeechConfigured()` — thiếu Azure config sẽ trả lỗi cấu hình.
- **Admin-only nav:** user thường không thấy `/ai-settings` trong sidebar nhưng route vẫn tồn tại (kiểm soát hiển thị, không phải bảo mật cứng).

---

## 8. Out of scope

- **Logic chấm điểm** (pipeline, prompt, speech SDK) — thuộc `shared/server` + `mock-test`/`practice`.
- **Nội dung phòng thi/luyện** (mock-exam, practice part 1/2/3, ai-simulation) — thuộc `mock-test`/`practice`.
- **Logic thanh toán nội bộ** (tạo order, polling trạng thái, pricing) — thuộc `billing` (module này chỉ tiêu thụ qua barrel).
- **Hạ tầng auth/analytics/history/app-data** — đã ở `shared/`; module chỉ tiêu thụ.
- **Quản lý env** — xem [../../env-and-modules.md](../../env-and-modules.md).
