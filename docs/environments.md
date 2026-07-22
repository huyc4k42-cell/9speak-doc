# Môi trường: Staging & Production

Tài liệu vận hành việc tách 2 môi trường của 9-Speaking. Đối tượng: dev/người deploy.

## 1. Mô hình tổng quan

Repo có **2 môi trường**, ánh xạ 1-1 với 2 nhánh dài hạn và 2 project Vercel:

| Môi trường | Nhánh git | Vercel project | Domain | Backend (Railway) | Firebase |
|---|---|---|---|---|---|
| **Production** | `main` | `9-speaking` | ielts.9speak.vn | `backend-9speak-production-5279…` | `speak-dba61` |
| **Staging** | `dev` | `9-speaking-staging` | staging.9speak.vn | `backend-9speak-staging-production…` | (project staging riêng) |

DeepSeek/OpenAI, Whisper, Azure, Deepgram, SpeechSuper, Cloudflare R2 **dùng chung** cả 2 môi trường.

## 2. Hai nguồn env tách biệt (rất quan trọng)

| Nơi chạy | Env đến từ đâu |
|---|---|
| **Local** (`npm run dev`, `build:*`) | các file trong `env/` (qua `dotenv-cli`) |
| **Vercel** (push git) | **Dashboard** của từng Vercel project — KHÔNG đọc file `env/` (đã gitignore) |

→ Sửa file `env/` chỉ ảnh hưởng máy local. Muốn đổi env trên Vercel phải sửa Dashboard (hoặc dùng `npm run env:push:*`).

## 3. Cấu trúc file env

```
env/
  shared.env       # Nhóm B — DÙNG CHUNG (AI/speech/R2/cờ luồng/giá top-up)
  staging.env      # Nhóm A+C — riêng STAGING
  production.env   # Nhóm A+C — riêng PRODUCTION
  *.env.example    # bản mẫu (commit được, không secret)
.env.local.backup  # backup .env.local cũ (gitignored, để tham chiếu)
```

**Phân nhóm biến:**
- **Nhóm A** (khác môi trường): `NEXT_PUBLIC_NINESPEAK_API_BASE_URL`, toàn bộ `NEXT_PUBLIC_FIREBASE_*`, `FIREBASE_ADMIN_*`.
- **Nhóm B** (dùng chung → `shared.env`): AI grading, Whisper, Azure, Deepgram, SpeechSuper, `MOCK_EXAM_PART13_FLOW_MODE`, R2, `PRACTICE_TOPUP_*`, timeout backend.
- **Nhóm C** (khác môi trường): `NEXT_PUBLIC_SITE_URL`, Analytics (GA4/Mixpanel — **mã riêng** mỗi môi trường), Alpha log (staging bật, prod tắt).

> Pricing subscription (`PRICING_*_MONTHS_VND`) đã bỏ — giá lấy từ API `GET /api/v1/premium-packages`. Chỉ còn `PRACTICE_TOPUP_*` (có fallback cứng trong code).

## 4. Chạy local theo "flavor"

| Lệnh | Flavor | Backend + Firebase |
|---|---|---|
| `npm run dev` | **staging** (mặc định) | Railway/Firebase staging |
| `npm run dev:prod` | production | Railway/Firebase prod |
| `npm run build:staging` | staging | build local với env staging |
| `npm run build:prod` | production | build local với env prod |
| `npm run build` | (không flavor) | **dành cho Vercel** — lấy env từ Dashboard |

Bắt đầu lần đầu:
```bash
cp env/shared.env.example     env/shared.env
cp env/staging.env.example    env/staging.env
cp env/production.env.example env/production.env
# rồi điền secret vào các file .env (đã gitignore)
```

## 5. Deploy (tự động qua Git)

Không deploy thủ công bằng `vercel --prod`. Mỗi Vercel project bám 1 nhánh:

- Push/merge vào **`dev`** → project `9-speaking-staging` tự build & deploy staging.
- Merge **`dev → main`** → project `9-speaking` tự build & deploy production.

Cross-deploy được chặn bằng **Ignored Build Step** (xem mục 7) để mỗi project chỉ build đúng nhánh của nó.

## 6. Nạp/đồng bộ env lên Vercel

Thay vì click Dashboard từng biến:
```bash
vercel link            # link thư mục với project staging (1 lần)
npm run env:push:staging   # nạp shared.env + staging.env lên project đang link
```
Script `scripts/push-env.mjs` in rõ project đang link + bắt xác nhận trước khi ghi. Bỏ qua placeholder `<ĐIỀN SAU>`. Chạy không `--yes` = dry-run.

> **Không** chạy `env:push:prod` lên project production đang chạy trừ khi cố ý ghi đè — env prod trên Dashboard là nguồn chân lý.

## 7. Checklist dựng staging (một lần)

1. **Firebase staging**: tạo project → bật Google Auth → lấy web config (6 biến `NEXT_PUBLIC_FIREBASE_*`) + service account JSON (3 biến `FIREBASE_ADMIN_*`) → điền vào `env/staging.env`.
2. **Vercel project**: Add New Project → import cùng repo → tên `9-speaking-staging` → **Production Branch = `dev`** → Node 24.x → Build Command = `npm run build`.
3. **Ignored Build Step** (Settings → Git):
   - `9-speaking` (prod): `if [ "$VERCEL_GIT_COMMIT_REF" = "main" ]; then exit 1; else exit 0; fi`
   - `9-speaking-staging`: `if [ "$VERCEL_GIT_COMMIT_REF" = "dev" ]; then exit 1; else exit 0; fi`
4. **Domain**: thêm `staging.9speak.vn` vào project staging → tạo DNS theo hướng dẫn Vercel.
5. **Firebase Authorized domains** (project staging): thêm `staging.9speak.vn`.
6. **Nạp env**: `vercel link` (chọn project staging) → `npm run env:push:staging`.
7. **Verify**: push commit nhỏ lên `dev` → mở `staging.9speak.vn` → test login Google + 1 lượt chấm.

## 8. FAQ

**Các thay đổi này có ảnh hưởng production đang chạy không?**
Không. Vercel lấy env từ Dashboard (file `env/` đã gitignore, không lên repo); script `build` giữ nguyên; `.env.local` chỉ đổi tên thành backup (vốn đã gitignore). Production redeploy với cùng build + cùng env → hành vi y hệt.

**Vì sao staging set env vào target `production` của Vercel?**
Vì nhánh production của project `9-speaking-staging` là `dev` — deploy từ `dev` là "Production" trong phạm vi project đó.

**Firebase auth trên `staging.9speak.vn` cấu hình thế nào?**
Code đã hỗ trợ: regex `*.9speak.vn` trong `lib/firebase/client.ts` dùng host hiện tại làm `authDomain`, còn `next.config.ts` proxy `/__/auth/*` về `NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN` (= `<staging>.firebaseapp.com`). Chỉ cần thêm `staging.9speak.vn` vào Authorized domains của Firebase staging.
