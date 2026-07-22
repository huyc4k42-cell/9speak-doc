# Hướng dẫn phát triển độc lập theo Module (Developer Module Guide)

Tài liệu này mô tả cấu trúc đã module hóa của hệ thống IELTS Speaking và **quy tắc tách biệt** để nhiều lập trình viên (hoặc nhiều Claude subagents) phát triển song song mà không xung đột mã nguồn. Ranh giới này được **enforce tự động** bằng ESLint (`eslint.config.mjs`) và phân công bằng `.github/CODEOWNERS`.

---

## 📂 Sơ đồ cấu trúc Module

```
features/
├── shared/                   # [LÕI dùng chung] — READ-ONLY với module owner
│   ├── lib/                  # ninespeak-client, analytics, cefr-*, band-score,
│   │                         #   app-data-client, history-client, top-up-packs, api/...
│   ├── server/               # SERVER-ONLY: evaluation-pipeline, speech SDK (Azure/
│   │                         #   OpenAI/Deepgram), question-store + part{1,2,3}.json,
│   │                         #   prompt-resolvers, scoring guards, alpha-logger...
│   ├── types/                # contract types: app-api, ninespeak-api, mock-exam,
│   │                         #   practice-evaluation
│   ├── data/                 # nội dung dùng chung: mock-exam-definition,
│   │                         #   mock-exam-question-bank.json, mock-exam, mock
│   ├── providers/            # AuthProvider, FeedbackSurveyProvider, Analytics,
│   │                         #   ClientDiagnostics (provider cross-cutting)
│   ├── hooks/                # useMockExamRecorder, useQuestionAudio,
│   │                         #   useReducedMotionPreference, useRequireTargetBand...
│   └── components/           # UI tái dùng: AudioWaveBars, AssistantUnavailablePanel,
│                             #   reports/* (report kit), BandScorePromptDialog...
│
├── mock-test/                # [MODULE 1] Thi thử Mock Exam & AI Simulation
│   ├── index.ts              # PUBLIC API cho peer (helpers báo cáo isomorphic)
│   ├── components/ data/ hooks/ layouts/ pages/ providers/ server/ types/ lib/
│
├── practice/                 # [MODULE 2] Luyện tập Part 1/2/3
│   ├── index.ts              # PUBLIC API cho peer
│   └── components/ data/ hooks/ layouts/ pages/ server/ types/ lib/
│
├── billing/                  # [MODULE 3] Thanh toán & Paywall
│   ├── index.ts              # PUBLIC API (CONTRACT) — module khác CHỈ import từ đây
│   └── components/ hooks/ pages/
│
└── onboarding-dashboard/     # [MODULE 4] Dashboard, Shell, Onboarding, History, Settings
    ├── index.ts              # composition root — không có public API hướng peer
    └── components/ layouts/ pages/ server/ (BFF /api/app/*)
```

> `app/` chỉ là tầng routing (host). `app/` được phép import trực tiếp page/layout/
> route-handler của bất kỳ module nào — đó là quan hệ host↔module, không phải import
> chéo giữa peer.

---

## 🛡️ Nguyên tắc thiết kế & Tách biệt Logic

### 1. Lõi `shared/` — READ-only & không phụ thuộc tính năng
- `features/shared/` là hạ tầng kỹ thuật + contract type + nội dung + UI/hook/provider dùng chung.
- **Quy tắc cứng (ESLint enforce):** `shared/` **KHÔNG** được import từ bất kỳ feature module nào (`mock-test`, `practice`, `billing`, `onboarding-dashboard`). Lõi không bao giờ phụ thuộc tính năng.
- Module owner **không tự ý sửa** `shared/`. Cần đổi core thì làm qua đợt refactor hạ tầng, **bắt buộc tương thích ngược**.
- `shared/` là **toolbox nhiều entry-point** (`lib/`, `server/`, `types/`, `hooks/`, `components/`, `providers/`, `data/`): module import sâu trực tiếp vào file cần dùng (không có barrel tổng).

### 2. Mock Test & Practice — hai module ngang hàng, KHÔNG dính nhau
- **ESLint enforce:** `mock-test` và `practice` **KHÔNG** được import lẫn nhau, cũng **KHÔNG** import `onboarding-dashboard`.
- Mọi thứ hai bên dùng chung (recorder, waveform, report kit, audio hook, content store, band util, contract type...) đã được đưa về `shared/`.
- Prompt & evaluator chấm điểm tách riêng từng module (`mock-test/server/...`, `practice/server/...`) để cải tiến độc lập.

### 3. Firestore Ledgers độc lập
- Mock Test ghi quota tại collection `mocktest_scoring_charges`; Practice tại `practice_scoring_charges`. Không dùng chung để tránh xung đột trạng thái/lock.

### 4. Billing — Public API Contract (`@/features/billing`)
- Billing chỉ lộ ra một bề mặt công khai duy nhất tại **`features/billing/index.ts`**: `usePaymentOrderFlow`, `usePremiumPackages`, `PricingPlanCards`, `SubscriptionPaymentModals`, `PurchaseStatusBackdrop`, `PricingPage`.
- **ESLint enforce:** module khác **CHỈ** được `import ... from "@/features/billing"`, **cấm** đụng path nội bộ (`@/features/billing/hooks/*`, `@/features/billing/components/*`). Nhờ vậy owner Billing tự do refactor nội bộ miễn giữ contract.

### 5. Onboarding/Dashboard — composition root
- Là module shell/host nội bộ: chứa `AppShell`, các trang dashboard/onboarding/history/settings và **BFF** (`server/` cho `/api/app/*`, gồm `app-backend` thật và `app-backend-mock` dev).
- Được phép dùng module tính năng khác để lắp ráp/giả lập backend, **nhưng chỉ qua barrel** `@/features/<module>` (ESLint cấm path nội bộ của peer).
- Không module peer nào được import ngược vào `onboarding-dashboard` (hạ tầng dùng chung như `AuthProvider`, `history-client`, `app-data-client` đã nằm ở `shared/`).

### 6. Lịch sử & Hiệu suất chung
- `/history` và `/hieu-suat` dùng `features/shared/lib/history-client.ts` để gộp dữ liệu Mock Test + Practice từ NineSpeak API rồi sắp theo thời gian.

---

## 🔒 Enforcement & phân công

- **Ranh giới**: `eslint.config.mjs` cấu hình `no-restricted-imports` theo từng thư mục module. Vi phạm → `npm run lint` báo lỗi kèm thông điệp trỏ về tài liệu này.
- **Chủ sở hữu**: `.github/CODEOWNERS` map mỗi `features/<module>/` cho một owner/team → PR tự gọi review đúng người.

---

## 🤖 Hướng dẫn làm việc với Claude (AI Guidelines)

Khi giao việc cho Claude, hãy giới hạn workspace để Claude phản hồi nhanh và không sửa nhầm ngoài phạm vi:

- **Module Mock Test**: chỉ sửa trong `features/mock-test/`, đọc `features/shared/` làm tham chiếu, không sửa `shared/`.
- **Module Practice**: chỉ trong `features/practice/`.
- **Module Billing**: chỉ trong `features/billing/`; khi cần thêm khả năng cho module khác thì export thêm tại `features/billing/index.ts`.
- **Module Onboarding/Dashboard**: chỉ trong `features/onboarding-dashboard/`.

*Ví dụ prompt:*
> "Thêm phân tích ngữ pháp nâng cao vào Practice Part 1. Chỉ chỉnh sửa trong `features/practice/`. Dùng tiện ích chung tại `features/shared/` (không sửa). Không đụng module khác."

> Nếu một thay đổi đòi sửa `shared/`, hãy nêu rõ và đảm bảo **tương thích ngược** trước khi làm.
