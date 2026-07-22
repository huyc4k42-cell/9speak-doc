# TECH — Module Onboarding / Dashboard (Shell + BFF)

> Đặc tả kỹ thuật cho module `features/onboarding-dashboard/`. Đọc kèm [PRD.md](./PRD.md). Quy tắc module hóa: [../../README-modules.md](../../README-modules.md). Bản đồ env: [../../env-and-modules.md](../../env-and-modules.md). Context AI cục bộ: [../../../features/onboarding-dashboard/AGENTS.md](../../../features/onboarding-dashboard/AGENTS.md).

---

## 1. Kiến trúc & sơ đồ thư mục

Module là **composition root / shell + BFF**: lắp ráp khung UI và cung cấp tầng server cho `/api/app/*`.

```
features/onboarding-dashboard/
├── index.ts                  # composition root — KHÔNG export gì cho peer
├── AGENTS.md                 # context AI cục bộ
├── layouts/
│   └── AppShell.tsx          # sidebar + topbar + guard auth/onboarding (client)
├── pages/
│   ├── Auth.tsx              # export Login / Register / ForgotPassword
│   ├── Onboarding.tsx        # export OnboardingFlow (+ shim OnboardingGoal/MicCheck)
│   ├── Dashboard.tsx
│   ├── History.tsx
│   ├── Performance.tsx       # route /hieu-suat
│   ├── SettingsPage.tsx      # gói đăng ký + thanh toán (billing barrel)
│   ├── AiScoringSettingsPage.tsx   # /ai-settings (admin)
│   ├── PromptOverrideLabPage.tsx   # /settings/prompt-lab
│   ├── AlphaLogs.tsx               # /settings/logs
│   ├── BackendAuthDebugPage.tsx    # /settings/backend-auth-debug, /debug/backend-auth
│   ├── Resources.tsx               # /resources
│   └── Community.tsx               # /community
├── components/
│   └── AnalyticsScripts.tsx        # nạp ở app/layout.tsx
└── server/                   # BFF (server-only)
    ├── app-backend.ts        # auth + rate-limit + proxy company API
    ├── app-backend-mock.ts   # mock backend in-memory (dev); dùng @/features/mock-test
    └── app-data-route-handlers.ts  # handler dùng chung cho nhiều route /api/app/*
```

`app/` là **host**: import trực tiếp page/layout/handler của module — quan hệ host↔module, không bị ràng buộc boundary ESLint.

---

## 2. `index.ts` — composition root

[features/onboarding-dashboard/index.ts](../../../features/onboarding-dashboard/index.ts) chỉ chứa `export {};`. Module không có public API hướng peer: app/ import sâu trực tiếp (`@/features/onboarding-dashboard/pages/...`, `.../layouts/AppShell`, `.../server/...`). Đây là khác biệt cốt lõi so với `billing`/`mock-test`/`practice` (có barrel public).

---

## 3. Routes `app/`

### 3.1 Pages — route group `(auth)`
| Route | File app/ | Component module |
|---|---|---|
| `/login` | `app/(auth)/login/page.tsx` | `Login` (Auth.tsx) |
| `/register` | `app/(auth)/register/page.tsx` | `Register` (Auth.tsx) |
| `/forgot-password` | `app/(auth)/forgot-password/page.tsx` | `ForgotPassword` (Auth.tsx) |
| `/onboarding/flow` | `app/(auth)/onboarding/flow/page.tsx` | `OnboardingFlow` (Onboarding.tsx) |

### 3.2 Pages — route group `(shell)` (bọc bởi AppShell)
`app/(shell)/layout.tsx` render `<AppShell buildInfo={...} aiScoringBadgeEnabled={...}>`.

| Route | Component module |
|---|---|
| `/dashboard` | `Dashboard` |
| `/history` | (History.tsx) |
| `/hieu-suat` | `Performance` |
| `/settings` | `SettingsPage` |
| `/settings/logs` | `AlphaLogsPage` |
| `/settings/prompt-lab` | `PromptOverrideLabPage` |
| `/settings/backend-auth-debug` | `BackendAuthDebugPage` |
| `/ai-settings` | `AiScoringSettingsPage` |
| `/resources` | `Resources` |
| `/community` | `Community` |
| `/debug/backend-auth` | `BackendAuthDebugPage` |

> Các route `(shell)` khác (`/practice`, `/mock-exam`, `/pricing`, `/report`, `/ai-simulation`) thuộc module `practice`/`mock-test`/`billing` — chỉ dùng AppShell làm khung.

### 3.3 Components nạp ở root app
- `app/layout.tsx` → `AnalyticsScripts`.
- `app/providers.tsx` → `FeedbackSurveyWidget` (widget thuộc **shared**: [shared/components/feedback/FeedbackSurveyWidget.tsx](../../../features/shared/components/feedback/FeedbackSurveyWidget.tsx)).

### 3.4 API app-data (`app/api/app/**`) ↔ BFF
Hai nhóm handler:

**(a) Qua [server/app-data-route-handlers.ts](../../../features/onboarding-dashboard/server/app-data-route-handlers.ts)** — các route gọi `handleSpeaking*`:

| Route handler | Hàm |
|---|---|
| `app/api/app/practice/catalog`, `app/api/app/speaking/practice/catalog` | `handleSpeakingPracticeCatalogGet` |
| `app/api/app/practice/topics`, `.../speaking/practice/topics` | `handleSpeakingPracticeTopicsGet` |
| `app/api/app/practice/topics/[topicId]`, `.../speaking/practice/topics/[topicId]` | `handleSpeakingPracticeTopicDetailGet` |
| `app/api/app/mock-exam/catalog`, `.../speaking/mock-exams/catalog` | `handleSpeakingMockExamCatalogGet` |
| `app/api/app/sessions`, `.../speaking/sessions` (GET/POST) | `handleSpeakingSessionsGet` / `handleSpeakingSessionsPost` |
| `app/api/app/sessions/[sessionId]`, `.../speaking/sessions/[sessionId]` (GET/PUT) | `handleSpeakingSessionGet` / `handleSpeakingSessionPut` |

**(b) Route gọi trực tiếp `app-backend` + `app-backend-mock`** (không qua route-handlers chung):

| Route | Import từ module |
|---|---|
| `app/api/app/bootstrap` | `getMockDashboardBootstrap`, `getOrCreateMockUserProfile` |
| `app/api/app/user/profile` | `getOrCreateMockUserProfile`, `updateMockUserProfile` |
| `app/api/app/user/performance` | `getMockPerformance`, `getOrCreateMockUserProfile` |
| `app/api/app/user/onboarding` | `getOrCreateMockUserProfile`, `saveMockOnboarding` |
| `app/api/app/onboarding/[id]` | `getMockOnboardingDefinition`, `getOrCreateMockUserProfile` |
| `app/api/app/logs` | `createMockAlphaLog`, `listMockAlphaLogsMerged`, `getOrCreateMockUserProfile` |
| `app/api/app/feedback/status` | `authenticateAppDataRequest` (+ shared `feedback-survey-store`) |

> Các route `app/api/app/uploads/*`, `app/api/app/feedback` (POST), `app/api/internal/*` không thuộc module này.

---

## 4. BFF — cơ chế chọn backend

Mỗi handler:
1. `authenticateAppDataRequest({ request, scope, limit })` — lấy bearer token (`getBearerToken`), verify Firebase (`verifyFirebaseIdToken`), áp `assertRateLimit` theo `uid`. Thiếu token → `throw "UNAUTHORIZED"` → 401.
2. `getAppBackendMode()` (re-export từ [shared/lib/feature-flags](../../../features/shared/lib/feature-flags.ts)): trả `"company"` nếu `process.env.APP_BACKEND_API_MODE === "company"`, ngược lại `"mock"`.
3. **Company mode** → `proxyCompanyApiRequest(request)`: build URL từ `COMPANY_API_BASE_URL` + `pathname+search`, copy headers (xóa `host`/`content-length`), `fetch` no-store, stream response trả về.
4. **Mock mode** → `hydrateMockUserProfile(...)` rồi gọi hàm tương ứng trong [app-backend-mock.ts](../../../features/onboarding-dashboard/server/app-backend-mock.ts).

[app-backend.ts](../../../features/onboarding-dashboard/server/app-backend.ts) là `server-only`. [app-backend-mock.ts](../../../features/onboarding-dashboard/server/app-backend-mock.ts) cũng `server-only`, lưu state in-memory (`Map`) cho profile/onboarding/sessions/alpha-logs; seed từ `shared/data/mock`; build catalog từ `shared/server/question-store`.

**Phụ thuộc peer trong mock backend:** `app-backend-mock.ts` import `{ buildOverallCriteria, isMockExamReportReady } from "@/features/mock-test"` — **qua barrel**, để tóm tắt phiên mock exam thành `SessionSummary` đồng nhất với practice.

---

## 5. Phụ thuộc & ranh giới

### 5.1 Phụ thuộc shared (import sâu trực tiếp — hợp lệ)
| Thành phần | Đường dẫn | Dùng ở |
|---|---|---|
| `AuthProvider` (`useAuth`) | [shared/providers/AuthProvider.tsx](../../../features/shared/providers/AuthProvider.tsx) | AppShell, Dashboard, Onboarding, Auth, SettingsPage |
| `FeedbackSurveyProvider` | [shared/providers/FeedbackSurveyProvider.tsx](../../../features/shared/providers/FeedbackSurveyProvider.tsx) | Dashboard |
| `history-client` | [shared/lib/history-client.ts](../../../features/shared/lib/history-client.ts) | History, Performance |
| `app-data-client` | [shared/lib/app-data-client.ts](../../../features/shared/lib/app-data-client.ts) | client fetch BFF |
| `app-bff-client` | [shared/lib/api/app-bff-client.ts](../../../features/shared/lib/api/app-bff-client.ts) | Onboarding (`fetchOnboardingDefinition`), Performance (`fetchPerformance`) |
| `company-api-client` | [shared/lib/api/company-api-client.ts](../../../features/shared/lib/api/company-api-client.ts) | SettingsPage (`userApi.init`) |
| feature-flags, entity-id, analytics*, subscription-plans, ai-scoring-settings-browser | `shared/lib/*` | nhiều trang |
| `question-store`, `azure-speech`, `alpha-logger`, `alpha-log-runtime`, `openai-structured-output`, `request-guards` | `shared/server/*` | BFF |
| types `app-api`, `mock-exam`, `ninespeak-api` | `shared/types/*` | toàn module |
| `firebase/admin` (`getBearerToken`, `verifyFirebaseIdToken`) | [lib/firebase/admin](../../../lib/firebase/admin.ts) | app-backend.ts |

### 5.2 Phụ thuộc peer — CHỈ qua barrel
| Barrel | Dùng ở | Mục đích |
|---|---|---|
| `@/features/mock-test` | [server/app-backend-mock.ts](../../../features/onboarding-dashboard/server/app-backend-mock.ts) | `buildOverallCriteria`, `isMockExamReportReady` |
| `@/features/billing` | [pages/SettingsPage.tsx](../../../features/onboarding-dashboard/pages/SettingsPage.tsx) | `usePaymentOrderFlow`, `usePremiumPackages`, `SubscriptionPaymentModals` |

### 5.3 Boundary ESLint
Cấu hình tại [eslint.config.mjs](../../../eslint.config.mjs), block cho `features/onboarding-dashboard/**`:
```js
onlyBarrel("mock-test"),   // cho @/features/mock-test, cấm @/features/mock-test/*
onlyBarrel("practice"),
onlyBarrel("billing"),
```
- Peer chỉ được import qua barrel `@/features/<mod>`; đụng path nội bộ (`@/features/mock-test/server/...`) → lint error.
- Chiều ngược: `mock-test`/`practice`/`billing` đều `blockModule("onboarding-dashboard")` → không ai import ngược vào module này.

### 5.4 Vòng đời token & refresh (audit — khớp API spec 2.3)
> Nguồn: [shared/lib/ninespeak-client.ts](../../../features/shared/lib/ninespeak-client.ts) + [shared/providers/AuthProvider.tsx](../../../features/shared/providers/AuthProvider.tsx). Refresh-token flow **đã cài đúng** đặc tả [API_DOCS-2.md §2.3](../../../references/docs/API_DOCS-2.md).

- **Lưu trữ** (localStorage): access = `ninespeak.auth.token`, refresh = `ninespeak.auth.refreshToken` (cặp), Google idToken cache = `9speak.google.idToken`. `getStoredNineSpeakSession()` trả `null` nếu **thiếu 1 trong 2** token.
- **Đăng nhập**: `signInWithGoogle` (popup/redirect) → cache Google idToken → `bootstrapServerAuth` → `ensureAuthenticatedUser` → `POST /auth/google` → **persist CẢ access + refresh** (`setStoredNineSpeakSession`).
- **401 tại `request()`**: refresh **transparent, single-flight** (nhiều 401 đồng thời chỉ gọi `/auth/refresh` 1 lần) → persist cặp token mới → **retry đúng 1 lần**; chỉ khi refresh cũng fail mới `onUnauthorized` (xoá phiên). Cover cả bootstrap `GET /user/init` lẫn 401 giữa phiên. `onNineSpeakSessionRefreshed` đồng bộ React state để không gửi lại token cũ.
- **Mở lại web hôm sau** (access TTL ~1 ngày): Firebase persistence **local** (indexedDB) giữ user → `onAuthStateChanged` → `ensureAuthenticatedUser` đọc session đã lưu → `/user/init` 401 → refresh ngầm → OK, **không** phải đăng nhập lại.
- **Fallback cuối** (cả access + refresh hết hạn): thử `/auth/google` bằng Google idToken **cache** (`getCachedGoogleCredentialIdToken`) — token này ~1h nên có thể đã hết hạn → khi đó buộc đăng nhập lại. (Điểm có thể gia cố sau: lấy Google idToken mới trước khi bỏ cuộc.)
- **Lưu ý migration**: phiên tạo **trước** khi có tính năng refresh chỉ có `ninespeak.auth.token` (chưa có `refreshToken`) → `getStoredNineSpeakSession()` = `null` → lần mở đầu tiên sau deploy có thể phải đăng nhập lại **1 lần**; sau đó cặp token mới được lưu và refresh hoạt động bình thường.
- **Cách validate runtime**: đăng nhập → DevTools ▸ Application ▸ Local Storage có đủ 2 key → hỏng giá trị `ninespeak.auth.token` (giữ `refreshToken`) → reload → Network thấy `POST /auth/refresh` 200 rồi `/user/init` retry 200, KHÔNG bị đá về login.

---

## 6. Env liên quan

| Biến | Đọc tại | Vai trò |
|---|---|---|
| `APP_BACKEND_API_MODE` | shared `feature-flags` (`getAppBackendMode`) | `company` → proxy backend thật; khác → mock |
| `COMPANY_API_BASE_URL` | [app-backend.ts](../../../features/onboarding-dashboard/server/app-backend.ts) | base URL proxy lên backend công ty (NineSpeak/Railway) |
| `NEXT_PUBLIC_NINESPEAK_API_BASE_URL`, `NEXT_PUBLIC_NINESPEAK_API_TIMEOUT_MS` | shared ninespeak-client | persistence/history/report qua NineSpeak (client) |
| `FIREBASE_ADMIN_PROJECT_ID`, `FIREBASE_ADMIN_CLIENT_EMAIL`, `FIREBASE_ADMIN_PRIVATE_KEY` | `lib/firebase/admin` | verify bearer token ở BFF |
| `NEXT_PUBLIC_FIREBASE_*` | shared auth client | Firebase Auth phía client |
| `ALPHA_LOG_DIR`, `VERCEL` | [app-backend-mock.ts](../../../features/onboarding-dashboard/server/app-backend-mock.ts) | thư mục lưu alpha log persisted |

Chi tiết & quy ước thêm env: [../../env-and-modules.md](../../env-and-modules.md).

---

## 7. Gotchas + nơi sửa

- **Thêm trang shell mới:** tạo page trong `features/onboarding-dashboard/pages/`, rồi tạo `app/(shell)/<route>/page.tsx` import nó. Thêm mục nav vào `primaryNavigation`/`secondaryNavigation` trong [AppShell.tsx](../../../features/onboarding-dashboard/layouts/AppShell.tsx); nếu cần slug analytics, cập nhật `slugFromPath`.
- **Thêm endpoint BFF:** thêm hàm vào [app-data-route-handlers.ts](../../../features/onboarding-dashboard/server/app-data-route-handlers.ts) (theo mẫu: authenticate → branch company/mock) + hàm mock trong [app-backend-mock.ts](../../../features/onboarding-dashboard/server/app-backend-mock.ts), rồi tạo `app/api/app/<...>/route.ts` gọi handler. Client gọi qua `app-data-client`/`app-bff-client` (shared) — **không** fetch tay.
- **Đổi cơ chế chọn backend:** sửa `getAppBackendMode()` ở shared `feature-flags` (không hardcode trong handler). Company proxy logic ở `proxyCompanyApiRequest`.
- **Cần helper từ peer:** chỉ thêm import từ barrel `@/features/<mod>`; nếu barrel chưa export → yêu cầu owner module đó export tại `index.ts` của họ. Tuyệt đối không import path nội bộ (lint sẽ fail).
- **Guard auth/onboarding** nằm tập trung ở AppShell — không nhân bản ở từng trang.
- **Quota header:** sau hành động đổi lượt phải gọi `refreshProfile()` (AuthProvider) để header cập nhật.
- **Mock state là in-memory** — không dùng làm nguồn dữ liệu thật; reset khi restart.
- **Trang debug** (`/debug/backend-auth`, prompt-lab, logs, backend-auth-debug) là công cụ nội bộ/admin, không phải luồng người dùng thật.
- **Build kiểm tra** bằng `npm run build` (có metadata) + `npx tsc --noEmit` + `npx eslint <file>` cho file đã đổi.
