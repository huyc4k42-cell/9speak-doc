# 📑 Tổng hợp tài liệu — 9Speak (CMS & Practice)

> Bản mục lục cho tất cả tài liệu & bản mẫu (mockup) đã xây dựng. Mở file tương ứng để xem chi tiết.
> **Cập nhật:** 2026-06-16.

---

## 1. Tất cả tài liệu (bảng tổng hợp)

| # | File | Loại | Mục đích | Đối tượng | Trạng thái |
|---|------|------|----------|-----------|-----------|
| **CMS** |
| 1 | [`CMS_REQUIREMENT_DASHBOARD_USER.md`](./CMS_REQUIREMENT_DASHBOARD_USER.md) | Requirement (PRD) | Yêu cầu CMS: Dashboard chỉ số + Quản lý User (read-only + nâng/đổi gói) | PM, Dev, Tech lead | ✅ Dùng được |
| 2 | [`cms-mockup.html`](./cms-mockup.html) | Mockup | Bản mẫu CMS Dashboard & Users (mở bằng trình duyệt) | Tất cả | ✅ |
| **PRACTICE — Mockup** |
| 3 | [`practice-flow-mockup.html`](./practice-flow-mockup.html) | Mockup | Luồng Practice **HIỆN TẠI** (baseline, bám UI thật) — để đối chiếu "before" | Tất cả | ✅ Tham chiếu |
| 4 | [`practice-flow-optimized.html`](./practice-flow-optimized.html) | Mockup | Luồng Practice **TỐI ƯU** (bản chính, đầy đủ tính năng) | Tất cả | ⭐ Bản chính |
| **PRACTICE — Tài liệu** |
| 5 | [`PRACTICE_MODULE_FEATURES.md`](./PRACTICE_MODULE_FEATURES.md) | Tài liệu tính năng | Mô tả **toàn bộ tính năng** (thuần product + đo lường) | Nội bộ dùng chung | ✅ |
| 6 | [`PRACTICE_ACCEPTANCE_CRITERIA.md`](./PRACTICE_ACCEPTANCE_CRITERIA.md) | Tiêu chí chấp nhận | Checklist nghiệm thu từng tính năng + **§14 thời gian trả kết quả** | QA, Dev | ✅ |
| 7 | [`PRACTICE_OPTIMIZE_FULL_SPEC.md`](./PRACTICE_OPTIMIZE_FULL_SPEC.md) | Spec cập nhật | Yêu cầu cập nhật + **dữ liệu/backend** + **hiệu năng** | Dev, Tech lead | ✅ |
| 8 | [`PRACTICE_FLOW_UPDATE_REQUIREMENT.md`](./PRACTICE_FLOW_UPDATE_REQUIREMENT.md) | Requirement (đầu) | Bản yêu cầu đầu tiên cho Practice | — | ⚠️ Bị #7 bao trùm |

> *Các file `AGENTS.md`, `CLAUDE.md`, `README.md`, `EVALUATION_PIPELINE_REDESIGN_PLAN_VI.md`, `HISTORY_PERSISTENCE_FLOW_VI.md` là của repo gốc, không thuộc bộ tài liệu này.*

---

## 2. Đọc từ đâu? (theo vai trò)

- **PM / Stakeholder:** mở mockup #4 để xem trực quan → đọc tính năng #5.
- **Dev:** #5 (tính năng) + #7 (dữ liệu/backend + hiệu năng) + #6 (nghiệm thu). Mockup #4 làm chuẩn UI.
- **QA:** #6 (acceptance criteria) là chính; đối chiếu hành vi trên mockup #4; đặc biệt **§14** (thời gian chờ).
- **Business (CMS):** #1 + mockup #2.

---

## 3. Quan hệ giữa các tài liệu Practice

```
practice-flow-optimized.html  ⭐ (bản mẫu chuẩn — mọi doc tham chiếu)
        │
        ├── PRACTICE_MODULE_FEATURES.md      → "có những tính năng gì + hành vi"
        ├── PRACTICE_ACCEPTANCE_CRITERIA.md  → "nghiệm thu thế nào + thời gian chờ"
        └── PRACTICE_OPTIMIZE_FULL_SPEC.md   → "cần sửa gì + dữ liệu/backend + hiệu năng"
                  └── (bao trùm) PRACTICE_FLOW_UPDATE_REQUIREMENT.md
```

- **#5 Features** = *cái gì* (mô tả tính năng, thuần product + analytics).
- **#6 Acceptance** = *đạt khi nào* (checklist QA + ngưỡng thời gian p50/p95).
- **#7 Full Spec** = *làm thế nào về dữ liệu* (before→after, yêu cầu backend, hiệu năng).
- **#8** là bản đầu, **#7 đã bao trùm** → ưu tiên dùng #7.

---

## 4. Tóm tắt nội dung từng tài liệu Practice

**#5 — Features:** Index (Forecast/Part/chủ đề) · Phòng luyện (câu hỏi, ghi âm, timeline, AI hỗ trợ) · Kết quả (band + 4 tiêu chí + câu đã sửa) · **Phân tích chi tiết 4 tiêu chí** · **Cải thiện câu (coach)** · So sánh qua các lượt · Edge case · Đo lường.

**#6 — Acceptance Criteria:** ~110 tiêu chí pass/fail theo 14 nhóm, gồm happy path + edge, và **§14 thời gian trả kết quả** (phản hồi tức thì, p50/p95, progressive reveal, timeout, giảm cảm giác chờ).

**#7 — Full Spec:** Before→After · yêu cầu chi tiết theo khu vực · **yêu cầu dữ liệu/backend (9 nhóm tín hiệu)** · **hiệu năng & ranh giới xử lý** (tính-1-lần / tra-tĩnh / lazy-load) · ưu tiên P0/P1/P2.

---

## 5. Điểm cốt lõi của bản tối ưu Practice
- Bỏ 3 tab → **"Câu trả lời đã sửa lỗi"** (track-changes, bấm xem giải thích) + **4 thẻ điểm = nút mở phân tích**.
- **Phân tích chi tiết 4 tiêu chí** (rail gọn / panel rộng đầy đủ): Phát âm (danh sách lỗi + nghe đối chiếu + nhóm âm + trọng âm/ngữ điệu + luyện từng từ), Trôi chảy & Mạch lạc (gauge tốc độ + ngắt nghỉ 3 mức), Từ vựng (CEFR + paraphrase), Ngữ pháp (độ chính xác/đa dạng + cấu trúc + lỗi theo loại).
- **Cải thiện câu = coach**: 3 mục tiêu, diff theo tiêu chí, cụm đáng học (lưu flashcard), nghe mẫu, thử nói lại.
- **So sánh qua các lượt**: delta + động viên + mini trend cố định; lượt cũ tự thu gọn.
- **Edge case khi nói** (A1–C1): card lỗi chuyên biệt, off-topic giữ transcript, quá dài tự dừng.
- **Hiệu năng**: progressive reveal + lazy-load — không bắt user chờ trắng màn hình.
