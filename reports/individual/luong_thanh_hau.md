# Báo Cáo Cá Nhân — Lab Day 10: Data Pipeline & Observability

**Họ và tên:** Lương Thanh Hậu  
**Vai trò:** Cleaning / Logic Lead
**Ngày nộp:** ___________  
**Độ dài yêu cầu:** **400–650 từ** (ngắn hơn Day 09 vì rubric slide cá nhân ~10% — vẫn phải đủ bằng chứng)

---

> Viết **"tôi"**, đính kèm **run_id**, **tên file**, **đoạn log** hoặc **dòng CSV** thật.  
> Nếu làm phần clean/expectation: nêu **một số liệu thay đổi** (vd `quarantine_records`, `hits_forbidden`, `top1_doc_expected`) khớp bảng `metric_impact` của nhóm.  
> Lưu: `reports/individual/[ten_ban].md`

---

## 1. Tôi phụ trách phần nào? (80–120 từ)

**File / module:**

- `transform/cleaning_rules.py` (core cleaning + quarantine rules)
- `quality/expectations.py` (expectation checks + halt/warn gate)
- `reports/team/metric_impact.md` (đối chiếu impact của rule mới theo run)

**Kết nối với thành viên khác:**

Tôi nhận raw rows từ pipeline ingest của bạn ETL, sau đó chuẩn hoá và lọc nhiễu trước khi chuyển dữ liệu sạch cho phần index/embedding của nhóm. Tôi cũng phối hợp với bạn phụ trách đánh giá để map các rule clean sang expectation tương ứng (đặc biệt nhóm rule idempotency và near-duplicate), để nếu clean bị “lọt” thì expectation vẫn chặn ở cổng chất lượng. Ngoài ra, tôi gửi lại danh sách `reason` trong quarantine để bạn observability thống kê theo `quarantine_records`, từ đó cả nhóm đối chiếu với bảng `metric_impact` và xác nhận thay đổi không làm lệch hành vi retrieval.

**Bằng chứng (commit / comment trong code):**

- Commit: `c191539` — *update cleaning and expectation rules for dedup-safe embeddings*
- Trong `transform/cleaning_rules.py` có comment đánh dấu: “Rule mới 1/2/3”
- Trong `quality/expectations.py` có comment đánh dấu: “E7 (mới)” và “E8 (mới)”

---

## 2. Một quyết định kỹ thuật (100–150 từ)

Tôi chọn tách rõ mức độ `halt` và `warn` trong expectation để tránh dừng pipeline vì các tín hiệu chất lượng chưa nghiêm trọng. Cụ thể, tôi để các lỗi có thể gây sai index hoặc duplicate embedding ở mức `halt`: `unique_non_empty_chunk_id`, `no_canonical_text_duplicates`, `effective_date_iso_yyyy_mm_dd`, và stale policy markers. Ngược lại, rule `chunk_min_length_8` chỉ ở mức `warn` vì chunk ngắn chưa chắc làm hỏng dữ liệu, nhưng là cảnh báo để xem lại nguồn.

Quyết định quan trọng nhất là ưu tiên idempotency: trong clean tôi tạo `chunk_id` ổn định (`doc_id + text + seq` hash) và thêm dedupe sau transform bằng canonical text (bỏ khoảng trắng thừa, hạ chữ thường, bỏ ký tự nhiễu). Đến expectation, tôi kiểm tra lại uniqueness của `chunk_id` và canonical text như một lớp bảo vệ thứ hai. Cách này giúp hạn chế việc một nội dung gần giống nhau bị index lặp khi chạy lại pipeline nhiều lần.

---

## 3. Một lỗi hoặc anomaly đã xử lý (100–150 từ)

Anomaly tôi xử lý là near-duplicate sau khi làm sạch nội dung. Triệu chứng ban đầu: cùng một policy xuất hiện nhiều chunk gần giống nhau do khác dấu câu, khoảng trắng hoặc có migration note trong ngoặc. Khi vào index, các bản gần trùng này làm retrieval thiên lệch vì embedding bị nhân bản ngữ nghĩa.

Tôi phát hiện vấn đề này qua hai điểm: (1) tỷ lệ quarantine duplicate thấp hơn kỳ vọng nhưng kết quả retrieval vẫn có dấu hiệu lặp; (2) expectation mới `no_canonical_text_duplicates` fail ở lần test đầu (duplicate canonical > 0). Fix tôi áp dụng gồm 3 bước trong cleaning: thêm rule chặn `low_information_chunk`, xoá migration note bằng regex `_REFUND_MIGRATION_NOTE`, rồi dedupe lần 2 bằng `_canonical_text` sau transform. Sau đó tôi bổ sung E8 trong expectation để đảm bảo regression không quay lại. Sau fix, check canonical duplicate về 0 và pipeline không còn halt vì lỗi trùng gần-giống.

---

## 4. Bằng chứng trước / sau (80–120 từ)

`run_id=day10_dedupsafe_2026-04-15_01`  
Before: `quarantine_records=7 | duplicate_chunk_text_after_transform=3 | should_halt=True`

`run_id=day10_dedupsafe_2026-04-15_02`  
After: `quarantine_records=10 | duplicate_chunk_text_after_transform=0 | should_halt=False`

Diễn giải ngắn: số quarantine tăng vì tôi chủ động loại thêm chunk nhiễu/near-duplicate, đổi lại expectation `halt` đã pass ở lớp canonical dedupe và idempotent IDs. Điều này khớp mục tiêu “giảm rác vào index” trong metric impact của nhóm, dù tổng cleaned rows có thể giảm nhẹ.

---

## 5. Cải tiến tiếp theo (40–80 từ)

Nếu có thêm 2 giờ, tôi sẽ thêm một báo cáo phân rã `quarantine reason distribution` theo `doc_id` cho từng `run_id` (ví dụ: `low_information_chunk`, `duplicate_chunk_text_after_transform`, `invalid_effective_date_format`). Việc này giúp nhóm nhanh chóng biết rule nào đang “ăn” nhiều dữ liệu nhất để tinh chỉnh ngưỡng/rule theo từng tài liệu, thay vì chỉ nhìn tổng `quarantine_records`.
