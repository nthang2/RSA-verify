# PROGRESS

## Phase hiện tại: 0
## Task tiếp theo: P0.2

## Chờ người dùng
- P0.2: chọn phiên bản crate `rsa` (mặc định: bản 0.9.x stable mới nhất) và các tính năng cần kiểm chứng (PKCS#1 v1.5 / OAEP / PSS; ký / mã hóa).

## Hàng đợi (theo thứ tự)
- P0.2 Ghim phiên bản `rsa` + tính năng vào `extract-config.yml`
- P0.3 Cài toolchain (elan/Lean, Charon, Aeneas), ghi phiên bản
- P0.4 Viết `scripts/extract.sh`, chạy trích xuất
- P0.5 Lập `inventory.md`
- P0.6 Tải test vector (Wycheproof, RFC 8017) + bộ chuyển JSON → Lean
- P0.7 Viết `scripts/check.sh`, `vectors.sh`, `mutants.sh`, `progress.py`
  - Ghi chú: phần tạo `CLAUDE.md` / `PROGRESS.md` / `ASSUMPTIONS.md` / `DEVIATIONS.md` của P0.7 đã làm sớm (chuyển vào P0.1); P0.7 chỉ còn viết các script.
- P0.8 Báo cáo khả thi `review/P0-feasibility.md`

## Đang xác thực / chờ review
- (trống)

## Xong
- 2026-11-17 P0.1 `git init` + khung thư mục + `.gitignore` — `e8c4272`
- 2026-11-17 P0.1 (bổ sung) tạo `CLAUDE.md`, `PROGRESS.md`, `ASSUMPTIONS.md`, `DEVIATIONS.md`; sửa RSA_VERIFY_PLAN.md cho khớp — commit `docs: add working rules and tracking files`

## BLOCKED / SPEC-ISSUE / POSSIBLE-BUG
- (trống)

## Số liệu (dán output scripts/progress.py)
- (chưa có — `scripts/progress.py` viết ở P0.7)
