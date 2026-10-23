# Báo cáo chất lượng dữ liệu và quá tải công việc: "data/workload.csv"

Công cụ: `skills/csv-quality/scripts/check_csv.py` | Ngưỡng quá tải: 9 giờ

## 1. Tổng quan chất lượng dữ liệu
| Chỉ số | Giá trị |
|---|---|
| Số dòng dữ liệu (không tính header) | 6 |
| Dòng thiếu owner | 1 |
| Dòng hours không hợp lệ | 1 |
| Số task_id bị lặp (distinct) | 1 (T02) |

## 2. Thống kê giờ làm việc theo người
| Người phụ trách | Tổng giờ hợp lệ | Trạng thái (> 9 giờ) |
|---|---|---|
| Lan | 9 | Bình thường |
| Minh | 3 | Bình thường |

## 3. Danh sách người quá tải
- Không có ai quá tải.

## 4. Các dòng bị loại khỏi tính toán
| Dòng | task_id | Lý do loại |
|---|---|---|
| 5 | T04 | invalid_hours |
| 6 | T02 | duplicate_id |
| 7 | T05 | missing_owner |

## 5. Chi tiết lỗi chất lượng
| Line | Cột | Loại | task_id | Mô tả |
|---|---|---|---|---|
| 5 | hours | invalid_hours | T04 | hours 'abc' không phải số hữu hạn không âm. |
| 6 | task_id | duplicate_id | T02 | task_id T02 đã xuất hiện ở line 3. |
| 7 | owner | missing_owner | T05 | owner trống. |

## 6. Khuyến nghị
- Kiểm tra và sửa giá trị hours tại line 5 để đảm bảo đó là số hợp lệ.
- Làm rõ và gán owner cho task ở line 7.
- Thay đổi task_id bị trùng lặp tại line 5 để tránh nhầm lẫn.