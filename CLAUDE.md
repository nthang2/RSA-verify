# Luật làm việc cho Claude Code — rsa-verify

Đầu mỗi phiên: đọc RSA_VERIFY_PLAN.md, CLAUDE.md, PROGRESS.md. Làm ĐÚNG MỘT task tiếp theo trong PROGRESS.md.
Cuối mỗi phiên: chạy scripts/check.sh, cập nhật PROGRESS.md + inventory.md, commit, tóm tắt cho người dùng, rồi DỪNG.

CẤM:
- Sửa src/, Extracted/Types.lean, Extracted/Funs.lean bằng tay.
- Sửa phát biểu định lý có STATUS: APPROVED hoặc có trong specs.lock.
- Dùng sorry/admit trong định lý đánh dấu proved; dùng native_decide; thêm axiom ngoài *External.lean.
- Tự chuyển phase, tự đánh dấu APPROVED, tự miễn một bước xác thực bắt buộc.
- Đưa bản đặc tả của mình cho ChatGPT trong các yêu cầu V6/V7 (hai bước này phải độc lập).
- Làm hàm ở tầng cao khi hàm con chưa có định lý approved.

BẮT BUỘC:
- Định lý cho hàm đã dịch dùng ký hiệu Hoare của thư viện Aeneas (dạng `f args ⦃ r => post ⦄`, kiểm tra cú pháp đúng với phiên bản đã cài), gắn thuộc tính để tactic tự dùng lại ở hàm gọi.
- Định nghĩa toán đặt trong Rsa/Math/, phải chạy được; Specs chỉ dùng lại.
- Mỗi giả định crate ngoài: ghi vào ASSUMPTIONS.md (tên, crate, nội dung, lý do).
- Mọi bất đồng (vector lệch, bản độc lập khác, reviewer phản đối): ghi vào DEVIATIONS.md.
- Khi cần người dùng quyết định: ghi vào PROGRESS.md mục "Chờ người dùng" và DỪNG.
- Ưu tiên lemma nhỏ; build từng file (lake build <Module>) trước khi build toàn bộ.
