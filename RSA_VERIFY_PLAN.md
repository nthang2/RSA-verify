# RSA_VERIFY_PLAN.md — Kế hoạch kiểm chứng crate `rsa` (RustCrypto) bằng Lean 4

> File này dành cho Claude Code. Đặt nó ở thư mục gốc của repo `rsa-verify/`.
> Mỗi phiên làm việc, Claude Code đọc file này + `CLAUDE.md` + `PROGRESS.md`,
> làm **đúng một task tiếp theo**, cập nhật `PROGRESS.md`, commit, rồi dừng.

---

## 0. Mục tiêu và nguyên tắc

**Mục tiêu:** chứng minh bằng Lean 4 rằng code Rust thật của crate `rsa` (không viết lại)
thỏa các đặc tả toán học. Pipeline: Rust → Charon → Aeneas → Lean 4 → định lý cho từng hàm.

**Nguyên tắc bất biến:**
1. **Không sửa tay** code Rust trong `src/` và code Lean sinh tự động trong `Extracted/Types.lean`, `Extracted/Funs.lean`.
   Nếu Rust phải đổi để dịch được → tạo một patch trong `translation-patches/` (mỗi patch một hàm, có ghi lý do).
2. **Không tin một nguồn duy nhất cho đặc tả.** Claude Code soạn nháp; đặc tả phải qua **cổng xác thực** (mục 4)
   — gồm kiểm tra bằng test vector bên ngoài, code có lỗi cài sẵn, và đối chiếu với một bản viết độc lập —
   rồi mới được người dùng duyệt. Đặc tả đã duyệt bị khóa trong `specs.lock` và không bao giờ bị sửa để chứng minh cho dễ.
3. **Chứng minh do Claude Code viết**, máy (Lean kernel + `scripts/check.sh`) kiểm tra.
4. **Làm từ dưới lên:** toán → hàm lá → padding → lõi RSA → API công khai. Không làm tầng trên khi tầng dưới chưa xong.
5. **Mọi giả định phải hiện rõ:** giả định về crate ngoài chỉ nằm trong `Extracted/FunsExternal.lean` / `TypesExternal.lean`
   và được liệt kê trong `ASSUMPTIONS.md`. Không dùng `axiom` ở nơi nào khác.
6. **Mọi bất đồng đều được ghi lại** trong `DEVIATIONS.md` (giữa Claude, ChatGPT, test vector, RFC và người dùng), kèm cách giải quyết.

**Vai trò:**
- **Người dùng:** chọn phạm vi, ra quyết định, duyệt đặc tả, là người duy nhất được ghi `APPROVED`.
- **Claude Code:** soạn đặc tả, chạy cổng xác thực, viết chứng minh, duy trì repo.
- **ChatGPT (reviewer độc lập):** phản biện đặc tả, dịch ngược đặc tả, viết bản đặc tả độc lập, đề xuất bản code có lỗi.
  Người dùng chuyển qua lại nội dung bằng các file trong `review/`.

---

## 1. Cấu trúc repo

```
rsa-verify/
├── RSA_VERIFY_PLAN.md        # file này (chỉ sửa khi người dùng yêu cầu)
├── CLAUDE.md                 # luật làm việc (tạo ở Phase 0)
├── PROGRESS.md               # trạng thái hiện tại + hàng đợi việc (nguồn sự thật)
├── ASSUMPTIONS.md            # danh sách mọi giả định/axiom ngoài
├── DEVIATIONS.md             # nhật ký bất đồng & cách giải quyết
├── inventory.md              # danh sách mọi hàm Rust, trạng thái từng hàm
├── specs.lock                # hash của các phát biểu định lý đã duyệt
├── extract-config.yml        # commit upstream + phiên bản Charon/Aeneas/Lean
├── src/                      # crate rsa, đúng commit đã ghim
├── translation-patches/      # NNN-<ten-ham>.patch + lý do
├── Extracted/
│   ├── Types.lean, Funs.lean                 # SINH TỰ ĐỘNG
│   └── TypesExternal.lean, FunsExternal.lean # viết tay: giả định cho crate ngoài
├── Rsa/Math/                 # định nghĩa toán CHẠY ĐƯỢC: số học modulo, CRT, I2OSP/OS2IP, RFC 8017
├── Rsa/Specs/                # định lý cho từng hàm, theo cây module của src/
├── Validation/
│   ├── Vectors/              # dữ liệu test vector (Wycheproof, RFC 8017) + bộ chạy
│   ├── Witnesses/            # ví dụ cụ thể thỏa tiền điều kiện
│   ├── Sanity/               # các hệ quả hiển nhiên suy ra từ đặc tả
│   ├── Mutants/              # bản sao hàm có lỗi cài sẵn (không nằm trong build chính)
│   ├── Stubs/                # cài đặt "giả" phải bị đặc tả từ chối
│   ├── Independent/          # bản đặc tả viết độc lập (ChatGPT) + chứng minh tương đương
│   └── reports/<Ten>.md      # biên bản cổng xác thực cho từng đặc tả
├── review/                   # gói gửi ChatGPT + phản hồi + quyết định của người dùng
└── scripts/
    ├── extract.sh            # chạy Charon + Aeneas, áp patch, sinh Extracted/
    ├── vectors.sh            # chạy đặc tả/code trên test vector
    ├── mutants.sh            # kiểm tra mọi mutant đều bị "giết"
    ├── check.sh              # gatekeeper (mục 5)
    └── progress.py           # in số liệu tiến độ
```

---

## 2. Các Phase và điều kiện qua cổng

Mỗi phase có **điều kiện qua cổng (gate)**. Chỉ chuyển phase khi gate đạt và người dùng xác nhận.

### Phase 0 — Dựng pipeline & đánh giá khả thi
- [ ] P0.1 `git init`; tạo khung thư mục ở mục 1.
- [ ] P0.2 **Hỏi người dùng** phiên bản `rsa` cần kiểm chứng (mặc định: bản 0.9.x stable mới nhất) và các tính năng đang dùng (PKCS#1 v1.5 / OAEP / PSS, ký / mã hóa). Ghim commit vào `extract-config.yml`.
- [ ] P0.3 Cài toolchain: Lean (elan), Charon, Aeneas (hoặc `cargo hax into lean`). Ghi phiên bản chính xác vào `extract-config.yml`. Đọc tài liệu/tutorial của Aeneas để nắm ký hiệu định lý và tactic của phiên bản đã cài.
- [ ] P0.4 Viết `scripts/extract.sh`, chạy trích xuất. Ghi lại mọi lỗi dịch.
- [ ] P0.5 Lập `inventory.md`: bảng mọi hàm Rust (trừ test) với cột: module, tên hàm, dịch được?, gọi tới crate ngoài nào, tầng (0–5), trạng thái.
- [ ] P0.6 Tải test vector: Wycheproof (các file `rsa_oaep_*`, `rsa_pkcs1_*`, `rsa_pss_*`, `rsa_signature_*` phù hợp với tính năng đã chọn) và ví dụ trong RFC 8017 vào `Validation/Vectors/` (ghi nguồn + license). Viết bộ chuyển JSON → dữ liệu Lean.
- [ ] P0.7 Viết `scripts/check.sh`, `vectors.sh`, `mutants.sh`, `progress.py`. Tạo `CLAUDE.md` (mục 7), `PROGRESS.md` (mục 8), `ASSUMPTIONS.md`, `DEVIATIONS.md`.
- [ ] P0.8 Viết báo cáo khả thi `review/P0-feasibility.md`: % hàm dịch được, các hàm cần patch, các crate ngoài cần giả định (bigint, sha2/digest, rand_core, subtle, zeroize…), đề xuất phạm vi.

**Gate 0:** `lake build` xanh trên code sinh tự động; các script chạy được; test vector đã nạp; người dùng đã chọn
(a) coi thư viện bigint là hộp đen (giả định trong FunsExternal) hay (b) dịch luôn bigint; và phạm vi tính năng.

### Phase 1 — Thư viện toán `Rsa/Math/` (Lean thuần, chưa đụng code Rust)
Mọi định nghĩa ở đây phải **chạy được** (`#eval` được, không `noncomputable`) để kiểm tra bằng test vector.
- RSA trên ℕ và định lý đúng đắn: với khóa hợp lệ (p, q nguyên tố khác nhau, e·d ≡ 1 mod (p−1)(q−1) hoặc mod λ(n) — theo đúng cách crate sinh d) và m < n thì (mᵉ mod n)ᵈ mod n = m. **Không** giả định gcd(m, n) = 1.
- CRT: giải mã CRT (dp, dq, qinv) bằng cᵈ mod n.
- I2OSP / OS2IP theo RFC 8017 và định lý nghịch đảo.
- Định nghĩa thuần hàm của EME-PKCS1-v1_5, EME-OAEP, EMSA-PSS, MGF1 theo RFC 8017 (hàm băm là tham số).

**Gate 1:** mọi định nghĩa Math đã qua cổng xác thực mức đầy đủ (mục 4) — đặc biệt V3 (test vector) và V7 (bản độc lập) — và được duyệt; định lý Math chứng minh xong, không `sorry`.

### Phase 2 — Hàm lá (tầng 1)
Chuyển đổi byte ↔ số, so sánh, tiện ích, MGF1… Mỗi hàm: "kết quả = định nghĩa trong Rsa/Math" + "không panic với đầu vào hợp lệ".

### Phase 3 — Padding (tầng 2)
Mã hóa/giải mã padding PKCS#1 v1.5, OAEP, PSS. Mỗi cặp cần:
- round-trip: `decode (encode m) = ok m` với mọi m có độ dài hợp lệ;
- khớp RFC: encode = định nghĩa trong Rsa/Math;
- từ chối: decode trả lỗi với mọi đầu vào không đúng định dạng RFC.

### Phase 4 — Lõi RSA (tầng 3)
Phép toán RSA thô, giải mã CRT, blinding. Kết quả (đọc thành số) = mᵉ mod n / cᵈ mod n, dưới các giả định bigint trong `FunsExternal.lean` (nếu chọn (a)).

### Phase 5 — API công khai (tầng 4)
`encrypt`/`decrypt`, `sign`/`verify`, kiểm tra khóa (`validate`). Định lý cuối: với mọi khóa hợp lệ và thông điệp hợp lệ,
decrypt(encrypt m) = m và verify(sign m) = ok; các nhánh lỗi đúng như tài liệu.

### Phase 6 — Tùy chọn
(b) dịch và kiểm chứng thư viện bigint; hoặc chứng minh bảo mật dạng trò chơi cho OAEP/PSS trên đặc tả Math.

**Ngoài phạm vi (ghi rõ trong báo cáo cuối):** tính chất hằng thời gian / kênh phụ (ví dụ lỗ hổng Marvin, RUSTSEC-2023-0071),
chất lượng RNG, kiểm tra số nguyên tố xác suất, và mọi điều nằm ngoài các định lý đã khóa.

---

## 3. Vòng đời của MỖI đặc tả (Phase 1–5)

Trạng thái: `todo → draft → validated → approved → proved` (thêm `blocked` khi kẹt).

1. **Đọc:** code Rust của hàm, code Lean đã dịch trong `Funs.lean`, định lý của các hàm nó gọi, mục RFC liên quan.
2. **Soạn (draft):** tạo `Rsa/Specs/<module>/<Ham>.lean`, chứng minh để `sorry`, đầu file ghi `-- STATUS: DRAFT`.
   Tự kiểm tra bằng checklist ở mục 6.
3. **Chạy cổng xác thực (mục 4)** ở mức tương ứng với tầng của hàm. Ghi kết quả vào `Validation/reports/<Ham>.md`.
   Nếu một bước thất bại → sửa draft, chạy lại. Khi mọi bước bắt buộc xanh → `-- STATUS: VALIDATED`.
4. **Gói review:** gom tối đa ~5 đặc tả `VALIDATED` vào `review/R<NNN>/` (mẫu ở mục 9). **DỪNG và báo người dùng.**
5. **Người dùng duyệt:** dán phản hồi ChatGPT vào thư mục review, rồi ghi `APPROVED: <tên>` hoặc `CHANGE: <tên> — <yêu cầu>` vào `review/R<NNN>/decision.md`.
   Nếu `CHANGE` → quay lại bước 2 (và chạy lại cổng xác thực).
6. **Khóa:** `-- STATUS: APPROVED`; `scripts/check.sh --lock <file>` ghi hash phát biểu vào `specs.lock`.
7. **Chứng minh:** thay `sorry` bằng chứng minh, dùng định lý của các hàm con (không unfold lại hàm con).
8. **Kiểm tra & commit:** `scripts/check.sh` xanh; commit `prove: <module>::<ham>`; cập nhật `inventory.md`, `PROGRESS.md`.

**Khi kẹt** (quá ~3 hướng tiếp cận):
- Thử **bác bỏ** mục tiêu bằng phản ví dụ nhỏ (`#eval`, `decide`). Nếu đặc tả sai → `SPEC-ISSUE` trong PROGRESS, báo người dùng. **Không tự sửa đặc tả đã khóa.**
- Thiếu định lý hàm con / giả định ngoài → `BLOCKED` kèm lý do, chuyển hàm khác cùng tầng.
- Nghi code Rust có lỗi thật → `POSSIBLE-BUG` kèm phản ví dụ cụ thể, báo người dùng ngay.

---

## 4. Cổng xác thực đặc tả (bắt buộc trước khi người dùng duyệt)

Mục đích: bắt lỗi đặc tả của **cả AI lẫn con người**, bằng những phép thử không phụ thuộc vào phán đoán của một ai.

| Bước | Tên | Làm gì | Trọng tài |
|---|---|---|---|
| V1 | Witness | Dựng ≥1 đầu vào cụ thể thỏa tiền điều kiện (`Validation/Witnesses/`), chứng minh nó thỏa. Chống giả thiết mâu thuẫn. | Lean |
| V2 | Hệ quả hiển nhiên | Chứng minh 2–5 tính chất "ai cũng biết phải đúng" suy ra từ đặc tả (vd. độ dài output decode ≤ k − 11; encode rồi decode ra lại m). Không suy ra được → đặc tả thiếu. | Lean |
| V3 | Test vector | Chạy định nghĩa Math (và code đã dịch, nếu chạy được) trên Wycheproof / RFC 8017. Mọi case "valid" phải khớp output, mọi case "invalid" phải bị từ chối. Lệch → ghi DEVIATIONS. | Dữ liệu bên ngoài |
| V4 | Mutant | Tạo ≥3 bản sao hàm có lỗi cài sẵn trong `Validation/Mutants/` (vd. bỏ kiểm tra byte 0x00, so sánh sai độ dài, lệch chỉ số, bỏ kiểm tra hash label). Mỗi mutant phải bị "giết": chứng minh được phủ định của đặc tả, hoặc có phản ví dụ cụ thể. Mutant sống sót → đặc tả quá yếu. Danh sách mutant lấy từ Claude + ChatGPT + người dùng. | Lean |
| V5 | Stub | Cài đặt "giả" (luôn trả lỗi, luôn trả input, trả hằng số…) phải **không** thỏa đặc tả, trừ khi đặc tả ghi rõ vì sao được phép. | Lean |
| V6 | Dịch ngược | ChatGPT, trong **cuộc chat mới, không có ngữ cảnh**, giải thích đặc tả Lean bằng tiếng Việt. Người dùng so với RFC/ý định. | Người dùng |
| V7 | Bản độc lập | ChatGPT, trong **cuộc chat mới**, chỉ đọc RFC/mô tả hàm (không thấy bản của Claude) và tự viết đặc tả Lean → `Validation/Independent/<Ham>_gpt.lean`. Claude Code chứng minh hai bản tương đương, hoặc chỉ ra input làm chúng khác nhau → ghi DEVIATIONS, người dùng quyết. | Lean + người dùng |

**Mức bắt buộc theo tầng:**
- Phase 1 (Math), Phase 3 (Padding), Phase 5 (API): **V1–V7 đầy đủ.**
- Phase 2 (hàm lá), Phase 4 (lõi): V1, V2, V4, V5; V3 khi có vector; V6 bắt buộc; V7 tùy người dùng chọn.
- Hàm tiện ích rất nhỏ (getter, wrapper): V1, V5; người dùng có thể miễn phần còn lại (ghi lý do vào report).

**Biên bản `Validation/reports/<Ham>.md`:** checklist V1–V7 (đạt / miễn + lý do), số vector chạy/đạt, danh sách mutant và cách bị giết, link tới các mục DEVIATIONS liên quan.

---

## 5. Gatekeeper `scripts/check.sh`

Phải fail (exit ≠ 0) nếu bất kỳ điều nào sai:
1. `lake build` lỗi.
2. Chạy lại `scripts/extract.sh` cho ra `Types.lean`/`Funs.lean` khác bản đã commit (code sinh bị sửa tay).
3. Có `sorry`/`admit` trong định lý mà inventory ghi `proved`.
4. Có `axiom` ngoài `Extracted/*External.lean`; hoặc có `native_decide`.
5. Phát biểu của định lý trong `specs.lock` bị thay đổi (so hash phần phát biểu, từ tên định lý đến `:=`).
6. `#print axioms` của mọi định lý `proved` chỉ chứa `propext`, `Classical.choice`, `Quot.sound` và giả định có tên trong `ASSUMPTIONS.md`.
7. Một đặc tả `APPROVED` mà biên bản xác thực thiếu bước bắt buộc, hoặc `vectors.sh` / `mutants.sh` không xanh.

`scripts/progress.py` in: tổng số hàm; số hàm có định lý; số định lý chứng minh xong; số `sorry`; số giả định ngoài;
số định lý phụ thuộc gián tiếp vào `sorry`/giả định; số vector đạt; số mutant bị giết/tổng; số mục DEVIATIONS còn mở; tiến độ theo tầng.

---

## 6. Checklist khi soạn đặc tả

1. **Tiền điều kiện có quá mạnh không?** Chỉ đòi điều kiện code thật sự cần (khớp `assert!`, `unwrap`, nhánh lỗi, overflow). Hàm gọi nó có thỏa được không?
2. **Hậu điều kiện có đủ mạnh không?** Một kết quả sai (nhân đôi, mất phần tử, sai thứ tự, sai byte) có lọt qua không?
3. **Mọi nhánh lỗi đã được nêu chưa?** Đặc biệt trong decode/verify.
4. **Có phụ thuộc kích thước `usize` (32/64-bit) không?** Nếu có, ghi rõ.
5. **Khớp RFC 8017 / tài liệu crate không?** Trích số mục RFC.
6. **Dùng lại định nghĩa trong `Rsa/Math/`**, không định nghĩa lại khái niệm toán trong file Specs.

---

## 7. Nội dung `CLAUDE.md` (tạo ở P0.7)

```markdown
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
```

---

## 8. Mẫu `PROGRESS.md` (tạo ở P0.7)

```markdown
# PROGRESS

## Phase hiện tại: 0
## Task tiếp theo: P0.1

## Chờ người dùng
- (trống)

## Hàng đợi (theo thứ tự)
- P0.1 ... P0.8

## Đang xác thực / chờ review
- <Ham>: V1 ✓ V2 ✓ V3 ✗(2 vector lệch) V4 … — review/R001

## Xong
- (ngày) task — commit

## BLOCKED / SPEC-ISSUE / POSSIBLE-BUG
- (trống)

## Số liệu (dán output scripts/progress.py)
```

---

## 9. Gói review `review/R<NNN>/` (người dùng chuyển cho ChatGPT)

Mỗi gói gồm các file riêng. **Mỗi file dùng một cuộc chat ChatGPT mới**, để các bước độc lập không nhìn thấy nhau.

**`A-review.md` — phản biện đặc tả**
```markdown
Bạn là chuyên gia kiểm chứng hình thức (Lean 4) và RSA (RFC 8017). Rà soát CÁC ĐẶC TẢ dưới đây, không cần viết chứng minh.
Với mỗi định lý trả lời:
1. Tiền điều kiện có mạnh hơn mức code cần không? Nơi gọi có thỏa được không?
2. Hậu điều kiện có để lọt kết quả sai nào không? Cho ví dụ cụ thể.
3. Thiếu nhánh lỗi nào không? 4. Khác RFC 8017 ở đâu (trích mục)?
5. Đề xuất thêm ≥3 bản code có lỗi (mutant) mà đặc tả PHẢI bắt được.
6. Kết luận: OK / CẦN SỬA (kèm phát biểu thay thế).

## Định lý: <tên>
### Code Rust gốc / ### Code Lean do Aeneas sinh ra / ### Định nghĩa Math liên quan
### Đặc tả đề xuất / ### Kết quả cổng xác thực (tóm tắt V1–V5)
```

**`B-backtranslate.md` — dịch ngược (V6)**: chỉ chứa phát biểu Lean + các định nghĩa nó dùng, KHÔNG có RFC, tên ý nghĩa hay mục đích.
```markdown
Giải thích bằng tiếng Việt, thật cụ thể, định lý Lean dưới đây khẳng định điều gì:
với đầu vào nào, kết quả phải thế nào, khi nào được trả lỗi. Không đoán mục đích; chỉ nói đúng điều được viết.
<phát biểu + định nghĩa>
```

**`C-independent.md` — bản độc lập (V7)**: chỉ chứa mô tả hàm + trích RFC + kiểu dữ liệu Lean của hàm đã dịch, KHÔNG có đặc tả của Claude.
```markdown
Viết đặc tả Lean 4 (chỉ phát biểu, không chứng minh) cho hàm dưới đây theo RFC 8017.
Dùng đúng chữ ký kiểu đã cho. Nêu rõ tiền điều kiện, hậu điều kiện, và mọi trường hợp trả lỗi.
<mô tả + trích RFC + chữ ký kiểu>
```

Người dùng dán câu trả lời tương ứng vào `A-response.md`, `B-response.md`, `C-response.lean`, rồi viết `decision.md`:
```
APPROVED: <tên>
CHANGE: <tên> — <yêu cầu>
```
Claude Code chỉ coi một đặc tả là đã duyệt khi có dòng `APPROVED` do người dùng ghi trong `decision.md`.

---

## 10. Tài liệu đọc thêm (tùy chọn, chỉ để tham khảo)
- Tài liệu và tutorial Lean của Aeneas / hax (`cargo hax into lean`).
- Một số dự án công khai kiểm chứng code Rust mật mã bằng Lean (ví dụ các dự án dùng Charon + Aeneas cho SPQR, curve25519-dalek, SymCrypt) — có thể xem để tìm ý tưởng về tactic hay cách tổ chức; dự án này tự quyết định cấu trúc và quy ước của mình.
- RFC 8017 (PKCS #1 v2.2); Project Wycheproof.
