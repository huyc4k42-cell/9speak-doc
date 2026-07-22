# Tài liệu Module (PRD + Đặc tả kỹ thuật)

Mỗi module có **2 tài liệu** dành cho cả con người (đầu mối phụ trách) và AI (đọc để phát triển/sửa lỗi đúng ràng buộc):

- **PRD.md** — Mô tả nghiệp vụ chi tiết: mục tiêu, người dùng, tính năng, user journey, business rules, edge cases, phạm vi.
- **TECH.md** — Đặc tả kỹ thuật: kiến trúc, public API, routes, data model, server/pipeline, state, phụ thuộc & ràng buộc boundary, env, gotchas.

> Đọc kèm: [README-modules.md](../README-modules.md) (bản đồ & quy tắc tách biệt), [env-and-modules.md](../env-and-modules.md) (biến env theo module), [scoring-pipeline-overview.md](../scoring-pipeline-overview.md) (chi tiết pipeline chấm điểm). `AGENTS.md` trong mỗi `features/<module>/` là context ngắn Claude tự nạp; bộ PRD/TECH này là tài liệu đầy đủ.

| Module | Vai trò | PRD | TECH |
|---|---|---|---|
| **shared** | Lõi hạ tầng dùng chung (read-only) | [PRD](shared/PRD.md) | [TECH](shared/TECH.md) |
| **mock-test** | Thi thử Mock Exam & AI Simulation | [PRD](mock-test/PRD.md) | [TECH](mock-test/TECH.md) |
| **practice** | Luyện tập Part 1/2/3 | [PRD](practice/PRD.md) | [TECH](practice/TECH.md) |
| **billing** | Thanh toán & Paywall | [PRD](billing/PRD.md) | [TECH](billing/TECH.md) |
| **onboarding-dashboard** | Shell, Onboarding, Dashboard, History, Settings, BFF | [PRD](onboarding-dashboard/PRD.md) | [TECH](onboarding-dashboard/TECH.md) |
| **performance** | Trang Tiến trình (`/hieu-suat`) — band trend chart, criteria, focus area, recommendations; hợp nhất Lịch sử dưới dạng tab | [PRD](performance/PRD.md) | [TECH](performance/TECH.md) |
| **streak** | Streak Widget trên Dashboard — chuỗi ngày học, 6 states, expanded panel, milestone toast | [PRD](streak/PRD.md) | [TECH](streak/TECH.md) |

> Một số module có thêm thư mục **`screens/`** (vd [mock-test/screens](mock-test/screens/README.md)): mỗi file là spec chi tiết, chính thức, chỉ 1 bản hiện hành cho một màn hình/chức năng cụ thể — `PRD.md` của module chỉ tóm tắt + link ra đây, không lặp lại nội dung.

## Ràng buộc quan trọng giữa module (tóm tắt — chi tiết trong README-modules.md)

- `shared` KHÔNG phụ thuộc bất kỳ feature module nào (enforce bằng ESLint).
- `mock-test` và `practice` KHÔNG import lẫn nhau; phần chung nằm ở `shared`.
- `billing` chỉ lộ qua barrel `@/features/billing`; module khác không đụng nội bộ.
- `onboarding-dashboard` là composition root: được dùng module khác **qua barrel**, và không module nào import ngược vào nó.
