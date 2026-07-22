# Mục lục tài liệu

Tổng hợp tài liệu nghiệp vụ/kỹ thuật của repo. Quy ước: tài liệu **active** ở `docs/`; tài liệu tham khảo ngoài (API, analytics, payment...) ở `references/docs/`. Plan/spec đã hoàn thành thì xóa (lịch sử nằm trong git); nếu cần giữ để tra cứu, đặt ở `docs/archive/`.

## Active — `docs/`

| File | Nội dung | Đối tượng |
|---|---|---|
| [scoring-pipeline-overview.md](scoring-pipeline-overview.md) | Luồng API & từng prompt chấm điểm Mock/Practice (sau Phase 1–6) | Dev + Product |
| [luot-cham-diem-cho-product.md](luot-cham-diem-cho-product.md) | Cơ chế lượt chấm: kiểm tra/trừ/thanh toán/cộng lượt + test case | Product |
| [environments.md](environments.md) | Tách 2 môi trường staging/production: file env, flavor local, deploy Vercel, checklist | Dev |
| [README-modules.md](README-modules.md) | Hướng dẫn phát triển độc lập theo Module & quy tắc làm việc cho AI agent | Dev |
| [env-and-modules.md](env-and-modules.md) | Biến env theo module: inventory, bản đồ env↔module, vì sao không tách env, phát hiện legacy | Dev |
| [logging-and-observability.md](logging-and-observability.md) | Cơ chế log & trace lỗi: Firestore `error_logs`, Vercel logs, alpha logs; cách xem, hiệu năng, cấu hình, roadmap | Dev + Support |
| [modules/README.md](modules/README.md) | **Bộ tài liệu PRD + Đặc tả kỹ thuật cho từng module** (mock-test, practice, billing, onboarding-dashboard, shared) | Dev + Product + AI |

## Context cho AI agent (không nằm trong docs/ nhưng liên quan)

| File | Nội dung |
|---|---|
| [../AGENTS.md](../AGENTS.md) | Quy trình git bắt buộc, tech stack, bản đồ project, glossary, gotchas |
| [../WORKING_WITH_CLAUDE.md](../WORKING_WITH_CLAUDE.md) | Quy trình chuẩn làm việc với Claude Code |
| [../features/shared/AGENTS.md](../features/shared/AGENTS.md) | Đặc tả kỹ thuật & Hạn ngạch hạ tầng chung (Read-only) |
| [../features/practice/AGENTS.md](../features/practice/AGENTS.md) | Đặc tả kỹ thuật & Ledger luyện tập (Module 2) |
| [../features/mock-test/AGENTS.md](../features/mock-test/AGENTS.md) | Đặc tả kỹ thuật & Ledger thi thử (Module 1) |
| [modules/mock-test/gap.md](modules/mock-test/gap.md) | **Gap report "Phản hồi chi tiết"** — đối chiếu tài liệu học thuật ↔ hiện trạng (đã làm / chưa làm + lý do) |
| [../features/billing/AGENTS.md](../features/billing/AGENTS.md) | Đặc tả kỹ thuật & Cổng thanh toán (Module 3) |
| [../features/onboarding-dashboard/AGENTS.md](../features/onboarding-dashboard/AGENTS.md) | Đặc tả kỹ thuật Shell, Auth, Merged History (Module 4) |

## Tham khảo — `references/docs/`

| File | Nội dung |
|---|---|
| [../references/docs/API_DOCS-2.md](../references/docs/API_DOCS-2.md) | Tài liệu API |
| [../references/docs/payment-api.md](../references/docs/payment-api.md) | API thanh toán |
| [../references/docs/word-cefr-background-implementation.md](../references/docs/word-cefr-background-implementation.md) | Triển khai CEFR cho từ vựng |
| [../references/docs/analytics-event-logging-guide.md](../references/docs/analytics-event-logging-guide.md) | Hướng dẫn log analytics event |
| [../references/docs/analytics-instrumentation-guide.md](../references/docs/analytics-instrumentation-guide.md) | Hướng dẫn gắn instrumentation |
| [../references/docs/analytics-implementation-assessment.md](../references/docs/analytics-implementation-assessment.md) | Đánh giá triển khai analytics |
| [../references/docs/analytics-implementation-assessment-feedback.md](../references/docs/analytics-implementation-assessment-feedback.md) | Feedback đánh giá analytics |
| [../references/docs/analytics-implementation-report-phase-1.md](../references/docs/analytics-implementation-report-phase-1.md) | Báo cáo analytics Phase 1 |

> Các file `analytics-events-requirement.{json,done.json,todo.json}` trong `references/docs/` là artifact theo dõi công việc (todo/done), không phải đặc tả ổn định — cân nhắc dọn khi xong.
