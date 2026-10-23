---
name: csv-quality
description: Bắt buộc sử dụng khi người dùng yêu cầu kiểm tra bất kỳ file CSV nào (ví dụ data/workload.csv, data/tasks.csv), tìm người vượt giờ, kiểm tra quá tải, tính tổng giờ làm việc hoặc ghi báo cáo CSV. Đọc SKILL.md này trước để chạy script qua bash, không tự đọc trực tiếp file CSV.
---

# CSV quality

Kiểm tra chất lượng CSV công việc và xác định người quá tải bằng script chuyên dụng qua bash, tuyệt đối không tự đọc file để tính nhẩm bằng mắt.

## Chạy script

Dùng tool `bash` (cwd là workspace). Lệnh đầy đủ:

```bash
python skills/csv-quality/scripts/check_csv.py --input <đường dẫn CSV> --max-hours <ngưỡng>
```

Ví dụ: `python skills/csv-quality/scripts/check_csv.py --input data/workload.csv --max-hours 8`

Quy tắc về ngưỡng:
- Nếu yêu cầu kiểm tra quá tải hoặc kiểm tra người vượt giờ nhưng **chưa cung cấp ngưỡng giờ** (max hours), hãy dừng lại và hỏi người dùng để lấy ngưỡng trước khi kết luận; không tự ý giả định hoặc lấy ngưỡng từ cuộc trò chuyện cũ.
- Khi đã có ngưỡng (ví dụ vượt 8 giờ -> `--max-hours 8`), truyền giá trị số đó vào `--max-hours`.
- Không cần đọc source script để chạy.

## Kiểm tra kết quả

- `exit_code` 0: phân tích thành công. `stdout` là JSON gồm `row_count`, `missing_owner_count`, `invalid_hours_count`, `duplicate_id_count`, `duplicate_ids`, `issues`, `hours_by_owner`, `overloaded_owners`, `excluded_rows`. Dữ liệu có lỗi chất lượng hoặc người quá tải vẫn là exit 0.
- `exit_code` 1 hoặc khác 0: **lỗi thực thi** (file không tồn tại, thiếu cột bắt buộc, lỗi parse, thiếu tham số hoặc ngưỡng không hợp lệ). Đọc `stderr`, báo lỗi cho người dùng. Không bịa thống kê, không ghi báo cáo như thể đã phân tích thành công.
- `timed_out` true hoặc `ok` false: lệnh không chạy xong; báo lỗi, không suy đoán kết quả.

## Viết báo cáo

1. Dùng `read_file` đọc template tại `skills/csv-quality/references/report-template.md`.
2. Lấy mọi con số từ JSON của script:
   - Ngưỡng giờ tối đa (`max_hours`).
   - Tổng giờ làm việc theo từng người (`hours_by_owner`).
   - Danh sách người bị quá tải (`overloaded_owners`), chỉ gồm người có tổng giờ > ngưỡng.
   - Danh sách dòng bị loại (`excluded_rows`) kèm lý do cụ thể.
   - Các chỉ số chất lượng dữ liệu và chi tiết lỗi (`issues`).
3. Không sửa file CSV khi người dùng chỉ yêu cầu kiểm tra.
4. Ghi báo cáo bằng `write_file` vào đường dẫn người dùng yêu cầu (ví dụ `output/workload.md` hoặc `output/csv-quality.md`), rồi trả lời tóm tắt kết quả cho người dùng.
