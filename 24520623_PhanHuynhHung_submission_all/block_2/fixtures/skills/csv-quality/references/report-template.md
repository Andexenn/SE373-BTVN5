# Báo cáo chất lượng dữ liệu và quá tải công việc: `{đường dẫn CSV}`

Công cụ: `skills/csv-quality/scripts/check_csv.py` | Ngưỡng quá tải: {max_hours} giờ

## 1. Tổng quan chất lượng dữ liệu
| Chỉ số | Giá trị |
|---|---|
| Số dòng dữ liệu (không tính header) | {row_count} |
| Dòng thiếu owner | {missing_owner_count} |
| Dòng hours không hợp lệ | {invalid_hours_count} |
| Số task_id bị lặp (distinct) | {duplicate_id_count} ({duplicate_ids}) |

## 2. Thống kê giờ làm việc theo người
| Người phụ trách | Tổng giờ hợp lệ | Trạng thái (> {max_hours} giờ) |
|---|---|---|
| {owner} | {total_hours} | {Quá tải / Bình thường} |

## 3. Danh sách người quá tải
- {owner}: {total_hours} giờ (vượt ngưỡng {max_hours} giờ)
*(Nếu không có ai quá tải: "Không có ai quá tải.")*

## 4. Các dòng bị loại khỏi tính toán
| Dòng | task_id | Lý do loại |
|---|---|---|
| {line} | {task_id} | {reasons} |

## 5. Chi tiết lỗi chất lượng
| Line | Cột | Loại | task_id | Mô tả |
|---|---|---|---|---|
| {line} | {column} | {type} | {task_id} | {message} |

## 6. Khuyến nghị
- {Cách sửa đề xuất cho từng nhóm lỗi; không tự sửa file nguồn}
