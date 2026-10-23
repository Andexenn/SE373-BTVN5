---
name: refund-policy
description: Tra cứu và thẩm định yêu cầu hoàn tiền cho đơn hàng theo ngày mua của khách hàng. Dùng khi người dùng hỏi về hoàn tiền, điều kiện hoàn tiền, phí hoàn hoặc kiểm tra đơn hàng có đủ điều kiện hoàn tiền hay không.
---

# Refund policy

Hướng dẫn tra cứu và áp dụng chính sách hoàn tiền cho đơn hàng.

## Các bước

1. **Kiểm tra thông tin đầu vào**:
   - Xác định đủ 3 thông tin cần thiết:
     - Ngày mua hàng.
     - Ngày yêu cầu hoàn tiền (dùng ngày được nêu trong câu hỏi của người dùng, không dùng ngày hiện tại của máy).
     - Trạng thái kích hoạt sản phẩm (đã kích hoạt hay chưa kích hoạt).
   - Nếu thiếu bất kỳ thông tin nào trong 3 thông tin trên (ví dụ chưa nêu trạng thái kích hoạt), dừng lại và hỏi người dùng để làm rõ; không tự giả định hoặc kết luận trước khi có đủ thông tin.

2. **Tìm và đọc tài liệu chính sách**:
   - Dùng `list_files` liệt kê các file trong thư mục `data/policies/`.
   - Dùng `read_file` đọc nội dung các file chính sách tìm được để xác định phạm vi hiệu lực theo ngày mua.
   - Chọn đúng chính sách áp dụng dựa trên **ngày mua** của khách hàng.

3. **Thẩm định điều kiện hoàn tiền**:
   - Tính số ngày đã qua: chênh lệch số ngày lịch giữa ngày yêu cầu hoàn tiền và ngày mua. Nếu số ngày bằng đúng giới hạn quy định thì vẫn đủ điều kiện về thời gian.
   - Kiểm tra điều kiện kích hoạt: sản phẩm đã kích hoạt thì không được hoàn tiền.
   - Xác định mức phí hoàn tiền (nếu đủ điều kiện) theo quy định trong chính sách đã chọn.

4. **Đọc template và trả lời**:
   - Dùng `read_file` đọc file template tại `skills/refund-policy/references/answer-template.md`.
   - Trả lời người dùng theo đúng cấu trúc của template, bao gồm: chính sách áp dụng, số ngày đã qua, kết luận đủ/không đủ điều kiện, phí hoàn tiền (nếu đủ điều kiện) và đường dẫn file chính sách làm căn cứ.
