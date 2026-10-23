# Báo cáo Phân tích Lab: Tra cứu chính sách đúng phiên bản (Block 1)

Cấu hình thực thi kiểm thử:
- **Model**: `openai/gpt-4o-mini` (OpenRouter API tương thích OpenAI Chat Completions, cấu hình qua file `.env` từ BT2).
- **Môi trường**: Python 3.12, LangChain 1.4.3, LangGraph 1.2.14, uv workspace (`agent-tools-skills-lab`).

---

## 1. Stage 00: Xác định giới hạn của agent

- **Câu hỏi kiểm tra**:
  > *"Tôi mua ngày 28/09/2026, yêu cầu hoàn ngày 06/10/2026, chưa kích hoạt. Tôi có được hoàn không?"*
- **Kết quả thực tế từ model (`openai/gpt-4o-mini`)**:
  > *"Tôi không có thông tin cụ thể về chính sách hoàn tiền cho sản phẩm bạn mua. Bạn nên kiểm tra điều kiện hoàn tiền của cửa hàng hoặc nhà cung cấp mà bạn đã mua để biết liệu bạn có đủ điều kiện hoàn trả hay không."*
- **Số lượt gọi**: 1 model call, 0 tool call.
- **Nhận xét**:
  - Agent tại Stage 00 hoàn toàn không được cấp tool (`tools: []`, `tools schema: 0`).
  - Agent tuân thủ system prompt: không bịa đặt quy định hay tự suy diễn kết quả mà thẳng thắn thừa nhận giới hạn thiếu công cụ truy xuất dữ liệu ngoài.
  - **Hạn chế xác định**: Agent thiếu khả năng cảm nhận môi trường (perception) và khả năng đọc file dữ liệu (`list_files`, `read_file`) để tra cứu văn bản chính sách.
- **Bằng chứng trong trace**:
  - File trace: `stage-00-chat/traces/20261007-224348_46d5f4f7_turn01_a45a0c5c.jsonl`
  - Seq 1: `user_submitted` ghi nhận câu hỏi đầu vào, `tools: []`.
  - Seq 2: `model_request` chứa `tools: []` và system prompt cấm bịa kết quả.
  - Seq 3: `model_response` trả về nội dung từ chối do thiếu dữ liệu, `tool_calls_requested: []`.
  - Seq 4: `run_completed` hoàn tất với câu trả lời trung thực.

---

## 2. Stage 01: Cài đặt và kiểm tra tool `list_files`

### Triển khai kỹ thuật
- **Tool logic (`_list`)**:
  - Tái sử dụng hàm `_resolve(workspace, path)` bảo vệ an toàn: chặn path traversal (`../`), đường dẫn tuyệt đối, symlink escape ra ngoài workspace.
  - Kiểm tra tồn tại và loại đường dẫn:
    - Nếu không tồn tại: trả lỗi `{"ok": false, "error": {"code": "DIRECTORY_NOT_FOUND", ...}}`.
    - Nếu là file: trả lỗi `{"ok": false, "error": {"code": "NOT_A_DIRECTORY", ...}}`.
  - Không duyệt đệ quy (chỉ duyệt con trực tiếp qua `target.iterdir()`).
  - Sắp xếp ổn định theo tên (`sorted`), trả metadata cho từng mục: `name`, `path` (tương đối workspace, dạng posix), `type` (`"file"` hoặc `"directory"`).
- **Đăng ký tool**:
  - Export trong `tools/__init__.py`.
  - Đăng ký vào mảng `TOOLS = [read_file, write_file, list_files]` trong `agent.py`.
  - Cập nhật mô tả năng lực trong `prompts.py` (không hardcode tên file/thư mục).
- **Kiểm thử tự động**:
  - Bộ unit test `test_files.py` (22/22 passed) và `test_agent.py` (2/2 passed) kiểm tra toàn diện chức năng đọc, ghi và liệt kê file.

### Kết quả kiểm tra tool trực tiếp
| STT | Trường hợp kiểm tra | Đường dẫn truyền vào | Kết quả kỳ vọng | Kết quả thực tế (JSON output) | Trạng thái |
|---|---|---|---|---|---|
| 1 | Thư mục hợp lệ | `data/policies` | Danh sách file kèm metadata | `{"ok": true, "path": "data/policies", "entries": [{"name": "policy-before-oct.md", "path": "data/policies/policy-before-oct.md", "type": "file"}, {"name": "policy-from-oct.md", "path": "data/policies/policy-from-oct.md", "type": "file"}]}` | **ĐẠT** |
| 2 | Đường dẫn là file | `data/policies/policy-before-oct.md` | Lỗi `NOT_A_DIRECTORY` | `{"ok": false, "error": {"code": "NOT_A_DIRECTORY", "message": "Đây là file, không phải thư mục: data/policies/policy-before-oct.md"}}` | **ĐẠT** |
| 3 | Thư mục không tồn tại | `data/not_exist` | Lỗi `DIRECTORY_NOT_FOUND` | `{"ok": false, "error": {"code": "DIRECTORY_NOT_FOUND", "message": "Không tìm thấy thư mục: data/not_exist"}}` | **ĐẠT** |
| 4 | Vượt workspace (traversal) | `../secret.txt` | Lỗi `PATH_OUTSIDE_WORKSPACE` | `{"ok": false, "error": {"code": "PATH_OUTSIDE_WORKSPACE", "message": "Đường dẫn thoát ra ngoài workspace: ../secret.txt"}}` | **ĐẠT** |

---

## 3. Stage 02: Thẩm định chính sách với Skill `refund-policy`

### Triển khai Skill
- Thư mục: `workspace/skills/refund-policy/`
  - `SKILL.md`: Frontmatter định danh (`name: refund-policy`, `description`). Thân file hướng dẫn: kiểm tra 3 thông tin đầu vào (ngày mua, ngày yêu cầu, kích hoạt); dùng `list_files` tại `data/policies/`; dùng `read_file` đọc chính sách và template; tính chênh lệch ngày lịch và đối chiếu điều kiện.
  - `references/answer-template.md`: Quy định cấu trúc trả lời chuẩn (chính sách áp dụng, số ngày đã qua, trạng thái kích hoạt, kết luận, phí hoàn tiền, căn cứ tài liệu).

### Kết quả các kịch bản kiểm tra
| Kịch bản | Yêu cầu kiểm tra | Kết quả đạt được | Bằng chứng trong Trace (`stage-02-skills/traces/`) |
|---|---|---|---|
| **Trường hợp A** (Mua trước ngày đổi CS) | Mua 28/09/2026, hoàn 06/10/2026, chưa kích hoạt | - Chính sách: Trước tháng 10 (`< 2026-10-01`)<br>- Số ngày: 8 ngày (hạn: 7 ngày)<br>- Trạng thái: Chưa kích hoạt<br>- Kết luận: **Không đủ điều kiện** (quá hạn 7 ngày)<br>- Phí: Không áp dụng<br>- Căn cứ: `policy-before-oct.md` | `traces/20261007-230709_c04f6dbb_turn01_5499e378.jsonl`<br>- Seq 4: `read_file(skills/refund-policy/SKILL.md)`<br>- Seq 8: `list_files(data/policies/)`<br>- Seq 12, 14: `read_file` đọc 2 chính sách<br>- Seq 17: Trả lời kết luận không đủ điều kiện |
| **Trường hợp B** (Mua từ ngày đổi CS) | Mua 02/10/2026, hoàn 12/10/2026, chưa kích hoạt | - Chính sách: Từ tháng 10 (`>= 2026-10-01`)<br>- Số ngày: 10 ngày (hạn: 14 ngày)<br>- Trạng thái: Chưa kích hoạt<br>- Kết luận: **Đủ điều kiện hoàn tiền**<br>- Phí: Không thu phí hoàn tiền (0đ)<br>- Căn cứ: `data/policies/policy-from-oct.md` | `traces/20261007-230718_b537e9bf_turn01_6a825a43.jsonl`<br>- Seq 4: Đọc skill `refund-policy`<br>- Seq 8, 9: `list_files` và đọc `answer-template.md`<br>- Seq 14, 15: Đọc policies<br>- Seq 19: Trả lời theo đúng chuẩn template: Đủ điều kiện, 0đ phí |
| **Đổi tên file** | Đổi tên 2 file chính sách thành `chinh-sach-truoc-thang-10.md` & `chinh-sach-tu-thang-10.md`, chạy lại Case B | - `list_files` tìm được danh sách tên file mới<br>- Đọc đúng file chính sách mới theo tên mới<br>- Kết luận không đổi: Đủ điều kiện hoàn tiền<br>- Căn cứ trích dẫn đúng file mới | `traces/20261007-230729_700241da_turn01_164bc352.jsonl`<br>- Seq 8: `list_files` tìm thấy file đã đổi tên<br>- Seq 12, 13: Đọc đúng 2 file theo tên mới<br>- Seq 17: Kết luận đúng theo chính sách mới |
| **Thiếu thông tin** | Hỏi mua 02/10/2026, hoàn 12/10/2026 (chưa nêu kích hoạt) | - Agent đọc `SKILL.md`, phát hiện thiếu trạng thái kích hoạt<br>- Dừng lại và hỏi người dùng: *"Để thẩm định yêu cầu của bạn, cần biết trạng thái kích hoạt của sản phẩm đã mua. Bạn có thể cung cấp thông tin này được không?"*<br>- Không tự giả định kết luận | `traces/20261007-230726_b005e470_turn01_1d82c754.jsonl`<br>- Seq 4: Đọc skill `refund-policy`<br>- Seq 6: Agent dừng lại ngay và hỏi người dùng trạng thái kích hoạt |

---

## 4. Trả lời câu hỏi cuối bài

### 1. Vì sao cần tool để tìm file và skill để hướng dẫn chọn chính sách?
- **Tool tìm file (`list_files`)** đóng vai trò **Perception (Cảm nhận môi trường)** động tại runtime: Cho phép agent quan sát danh sách file thực tế trong workspace mà không phụ thuộc vào việc lập trình viên biết trước tên file hay số lượng file.
- **Skill (`refund-policy`)** đóng vai trò **Procedural Knowledge (Quy trình nghiệp vụ)** chuyên biệt: Hướng dẫn agent logic nghiệp vụ (kiểm tra đủ 3 thông tin đầu vào, tra cứu theo ngày mua, tính khoảng cách ngày lịch và định dạng câu trả lời). Cơ chế nạp on-demand (tiết lộ lũy tiến / progressive disclosure) giúp system prompt ban đầu ngắn gọn, tối ưu token ngữ cảnh.

### 2. Nếu agent chưa có tool tìm file, việc sửa prompt có giải quyết được yêu cầu đổi tên file không? Giải thích.
- **KHÔNG thể giải quyết**.
- **Giải thích**: Prompt là ngữ cảnh tĩnh (static context) được thiết lập trước khi chạy. Nếu tên file bị đổi thành tên ngẫu nhiên tại runtime, prompt tĩnh không thể tự biết tên file mới để đưa vào câu lệnh `read_file`. Nếu không có tool cảm nhận môi trường như `list_files`, agent bắt buộc phải dựa vào việc con người can thiệp sửa prompt thủ công mỗi khi file đổi tên, làm mất tính tự chủ (autonomy) và khả năng thích ứng (adaptability) của hệ thống agentic AI.

---

# Báo cáo Phân tích Lab: Kiểm tra quá tải theo người (Block 2)

---

## 5. Stage 03: Chạy Bash với Python tính tổng giờ theo người

### Kịch bản và thực thi
- **Câu hỏi đầu vào**:
  > *"Dùng tool bash để chạy Python (python -c ...) tính tổng hours theo owner trong data/workload.csv. Không chỉnh sửa file đầu vào. Cho biết cách xử lý dòng lỗi và task_id trùng."*
- **Lệnh model thực hiện qua tool Bash**:
  ```bash
  python -c "import pandas as pd; df = pd.read_csv('data/workload.csv'); df = df.drop_duplicates(subset='task_id'); total_hours = df.groupby('owner')['hours'].sum(); print(total_hours)"
  ```
- **Kết quả lệnh tool trả về (stdout)**:
  ```
  owner
  Lan       45
  Minh    3abc
  ```
- **Nhận xét kết quả và phân tích dữ liệu**:
  - Do cột `hours` trong `data/workload.csv` chứa giá trị `'abc'` tại dòng 5 (`T04,Minh,abc`), Pandas/Python đọc toàn bộ cột dưới dạng chuỗi (`string/object`).
  - Phép tính `.sum()` đã thực hiện phép nối chuỗi (string concatenation) thay vì cộng số học:
    - Lan: `'4' + '5' = '45'` (thực tế Lan có 2 dòng hợp lệ T01: 4 và T02: 5). Nếu cộng trùng cả dòng 6 (T02, Lan, 5) sẽ là 14 giờ.
    - Minh: `'3' + 'abc' = '3abc'`.
  - Dòng 7 (`T05,,2`): thiếu tên owner, bị bỏ qua hoặc gây lỗi dữ liệu.
  - Model đã nhận diện được vấn đề: giá trị `'3abc'` không phải là số hợp lệ và khuyến nghị cần có logic chuẩn hóa, lọc dòng lỗi trước khi tính tổng.
- **Bằng chứng trong trace**:
  - File trace: `stage-03-bash/traces/20261007-233928_6d18a978_turn01_a73337c5.jsonl`
  - Seq 1: `user_submitted` nhận câu hỏi, danh sách tool `[read_file, write_file, bash]`.
  - Seq 2: `model_request` chứa context LangChain và model gpt-4o-mini.
  - Seq 3: `model_response` yêu cầu gọi tool `bash` chạy lệnh `python -c ...`.
  - Seq 4: `tool_started` và `tool_finished` trả về stdout với exit_code = 0.
  - Seq 7: `model_response` phân tích kết quả, chỉ ra chuỗi `3abc` và dòng lỗi.

---

## 6. Stage 04: Mở rộng Script `check_csv.py` và Cập nhật Skill `csv-quality`

### 6.1 Triển khai mã nguồn và quy tắc nghiệp vụ
1. **Script `workspace/skills/csv-quality/scripts/check_csv.py`**:
   - Thêm tham số bắt buộc `--max-hours`: Nhận số thực/nguyên không âm (`float >= 0`). Nếu thiếu tham số hoặc giá trị không hợp lệ (NaN, âm, chuỗi), in thông báo lỗi ra stderr và trả mã thoát `exit(2)`.
   - Chuẩn hóa: Dùng `.strip()` cho `task_id`, `owner`, `hours`. Phân biệt chữ hoa/thường của `owner`.
   - **Quy tắc Deduplication Task ID**: Lưu `first_seen[task_id] = line` cho lần xuất hiện đầu tiên (kể cả khi lần đầu chứa dữ liệu lỗi). Mọi lần xuất hiện sau của cùng `task_id` đều bị loại với lý do `duplicate_id` và **không được thay thế** bằng lần hợp lệ đầu tiên.
   - **Quy tắc loại dòng**: Dòng được cộng phải có đúng 3 trường, task_id không rỗng, owner không rỗng, hours là số hữu hạn không âm. Mỗi dòng bị loại xuất hiện 1 lần trong `excluded_rows` kèm mảng lý do được sắp xếp chuẩn: `wrong_field_count`, `missing_task_id`, `duplicate_id`, `missing_owner`, `invalid_hours`.
   - **Quá tải**: Tổng giờ `hours > max_hours` (bằng ngưỡng không tính là quá tải).
   - Bảo toàn các thống kê chất lượng dữ liệu ban đầu (`row_count`, `missing_owner_count`, `invalid_hours_count`, `duplicate_id_count`, `duplicate_ids`, `issues`).
2. **Skill `workspace/skills/csv-quality/SKILL.md` và `references/report-template.md`**:
   - `SKILL.md`: Frontmatter mô tả rõ ràng nhiệm vụ kiểm tra CSV, tìm người vượt giờ, kiểm tra quá tải. Hướng dẫn chạy `python skills/csv-quality/scripts/check_csv.py --input <path> --max-hours <ngưỡng>`. Quy định bắt buộc: Nếu câu hỏi chưa cung cấp ngưỡng, phải dừng lại hỏi người dùng; không tự ý lấy ngưỡng từ lịch sử cũ.
   - `report-template.md`: Bổ sung bảng tổng giờ theo người, danh sách người quá tải, và bảng các dòng bị loại kèm lý do chuẩn hóa.
   - Đồng bộ đầy đủ sang `fixtures/skills/csv-quality/`.
3. **Kiểm thử tự động**:
   - Viết test `test_first_id_occurrence_invalid_hours_rejects_subsequent_duplicate` trong `tests/test_check_csv.py` cho trường hợp đặc biệt (ID đầu tiên có hours lỗi).
   - Viết test `test_missing_max_hours_exit_nonzero` và `test_invalid_max_hours_exit_nonzero`.
   - **Kết quả Unit Tests**: Toàn bộ **67/67 tests PASSED** trong `stage-04-script-skill`.

### 6.2 Kết quả chạy Script trực tiếp
1. **Chạy với ngưỡng 8 (`--max-hours 8`) trên `data/workload.csv`**:
   ```json
   {
     "max_hours": 8.0,
     "hours_by_owner": {"Lan": 9.0, "Minh": 3.0},
     "overloaded_owners": [{"owner": "Lan", "total_hours": 9.0}],
     "excluded_rows": [
       {"line": 5, "task_id": "T04", "reasons": ["invalid_hours"]},
       {"line": 6, "task_id": "T02", "reasons": ["duplicate_id"]},
       {"line": 7, "task_id": "T05", "reasons": ["missing_owner"]}
     ]
   }
   ```
2. **Chạy với ngưỡng 9 (`--max-hours 9`) trên `data/workload.csv`**:
   ```json
   {
     "max_hours": 9.0,
     "hours_by_owner": {"Lan": 9.0, "Minh": 3.0},
     "overloaded_owners": [],
     "excluded_rows": [
       {"line": 5, "task_id": "T04", "reasons": ["invalid_hours"]},
       {"line": 6, "task_id": "T02", "reasons": ["duplicate_id"]},
       {"line": 7, "task_id": "T05", "reasons": ["missing_owner"]}
     ]
   }
   ```
3. **Chạy với ngưỡng 0 trên file biên `data/workload-edge.csv`**:
   - File đầu vào:
     ```csv
     task_id,owner,hours
     E01,Lan,abc
     E01,Lan,5
     E02,Minh,0
     ```
   - Kết quả JSON:
     ```json
     {
       "max_hours": 0.0,
       "hours_by_owner": {"Minh": 0.0},
       "overloaded_owners": [],
       "excluded_rows": [
         {"line": 2, "task_id": "E01", "reasons": ["invalid_hours"]},
         {"line": 3, "task_id": "E01", "reasons": ["duplicate_id"]}
       ]
     }
     ```
   - Nhận xét: Dòng 2 bị loại vì `invalid_hours`. Dòng 3 bị loại vì `duplicate_id` do E01 đã xuất hiện ở dòng 2. Lan không được cộng 5 giờ và không có tên trong `hours_by_owner`. Minh có 0 giờ, bằng ngưỡng nên không quá tải.

### 6.3 Kết quả 4 Kịch bản kiểm tra Agent
| Kịch bản | Yêu cầu kiểm tra | Hành vi của Agent & Kết quả | Bằng chứng trong Trace (`stage-04-script-skill/traces/`) |
|---|---|---|---|
| **1. Ngưỡng 8** | *"Kiểm tra data/workload.csv, người nào vượt 8 giờ? Ghi báo cáo vào output/workload.md."* | - Agent đọc `SKILL.md`<br>- Chạy lệnh Bash: `python skills/csv-quality/scripts/check_csv.py --input data/workload.csv --max-hours 8`<br>- Nhận JSON: Lan 9 giờ, Minh 3 giờ; Loại dòng 5 (`invalid_hours`), dòng 6 (`duplicate_id`), dòng 7 (`missing_owner`)<br>- Ghi báo cáo `output/workload.md`<br>- Kết luận: Lan quá tải (9 giờ) | `traces/20261007-234241_6e86a163_turn01_e88d67a5.jsonl`<br>- Seq 4: `read_file(skills/csv-quality/SKILL.md)`<br>- Seq 8: `bash(python check_csv.py ... --max-hours 8)`<br>- Seq 12: `write_file(output/workload.md)` |
| **2. Ngưỡng 9** | *"Kiểm tra data/workload.csv, người nào vượt 9 giờ? Ghi báo cáo vào output/workload.md."* | - Agent đọc `SKILL.md`<br>- Chạy Bash với `--max-hours 9`<br>- Đọc `report-template.md`<br>- Ghi báo cáo `output/workload.md`<br>- Kết luận: Không ai vượt quá 9 giờ làm việc | `traces/20261007-234254_94d2c199_turn01_d9479435.jsonl`<br>- Seq 4: `read_file(skills/csv-quality/SKILL.md)`<br>- Seq 8: `bash(python check_csv.py ... --max-hours 9)`<br>- Seq 12: `read_file(references/report-template.md)`<br>- Seq 16: `write_file(output/workload.md)` |
| **3. Thiếu ngưỡng** | *"Tính tổng giờ theo người trong data/workload.csv và xác định người quá tải."* | - Agent đọc `SKILL.md`<br>- Nhận diện yêu cầu thiếu ngưỡng giờ tối đa<br>- Dừng lại và hỏi người dùng: *"Bạn cần cung cấp ngưỡng giờ tối đa (max hours) để tôi có thể tính tổng giờ làm việc theo từng người và xác định người quá tải. Xin cho tôi biết ngưỡng giờ mà bạn muốn kiểm tra (ví dụ: 8 giờ)."*<br>- Không tự giả định ngưỡng 8 | `traces/20261007-234306_35071ffb_turn01_62c70e3c.jsonl`<br>- Seq 4: `read_file(skills/csv-quality/SKILL.md)`<br>- Seq 6: Model trả lời hỏi lại người dùng ngưỡng max-hours |
| **4. File không tồn tại** | *"Kiểm tra data/khong-ton-tai.csv, người nào vượt 8 giờ? Ghi báo cáo vào output/workload.md."* | - Script hoặc agent phát hiện file không tồn tại<br>- Phản hồi trung thực: *"File `data/khong-ton-tai.csv` không tồn tại, vì vậy tôi không thể thực hiện kiểm tra..."*<br>- Không bịa đặt tổng giờ hoặc ghi báo cáo rác | `traces/20261007-234311_953fa964_turn01_96149485.jsonl`<br>- Seq 2: `model_request`<br>- Seq 3: Model phát hiện file không tồn tại và trả lời ngay |

---

## 7. Trả lời câu hỏi cuối bài Block 2

### 1. Phần nào do script tính, phần nào do model diễn giải?
- **Phần do Script tính toán (Deterministic Execution)**:
  - Phân tích cú pháp file CSV, xác thực schema và định dạng từng trường.
  - Xử lý loại trừ trùng lặp ID công việc dựa trên lần xuất hiện đầu tiên (`first_seen`).
  - Lọc và gắn cờ toàn diện các dòng bị loại kèm danh sách mã nguyên nhân chuẩn hóa (`wrong_field_count`, `missing_task_id`, `duplicate_id`, `missing_owner`, `invalid_hours`).
  - Tính tổng số học chính xác của số giờ theo từng owner và so sánh với ngưỡng `max_hours` để xác định danh sách người quá tải.
  - Đảm bảo tính toán tin cậy tuyệt đối, tránh lỗi tính nhẩm hoặc ảo giác (hallucination) của mô hình ngôn ngữ.
- **Phần do Model diễn giải (Cognitive & Orchestration Reasoning)**:
  - Hiểu ý định của người dùng từ ngôn ngữ tự nhiên.
  - Quyết định kích hoạt skill `csv-quality` khi thấy yêu cầu kiểm tra dữ liệu CSV công việc.
  - Kiểm tra tính đầy đủ của thông tin đầu vào (ngưỡng giờ): nếu thiếu thì chủ động hỏi lại người dùng trước khi thực thi.
  - Điều phối tool (chạy script qua `bash`), tiếp nhận JSON kết quả từ `stdout`.
  - Nạp template báo cáo từ `references/report-template.md` và ánh xạ dữ liệu JSON thành văn bản Markdown chuyên nghiệp, sinh động kèm khuyến nghị thực tế.

### 2. Nếu sửa script nhưng không cập nhật skill và reference, báo cáo có thể sai hoặc thiếu thông tin gì?
- **Lỗi thực thi do thiếu tham số**: Script mới bắt buộc tham số `--max-hours`. Nếu `SKILL.md` không được cập nhật, model chỉ chạy lệnh cũ `python check_csv.py --input ...` dẫn đến script bị crash với exit code 2 (thiếu argument), agent không nhận được kết quả.
- **Mất khả năng xử lý tương tác khi thiếu thông tin**: Nếu không có chỉ dẫn trong `SKILL.md`, model có thể tự bịa ra một ngưỡng ngẫu nhiên (ví dụ 8 giờ) hoặc tự đọc file CSV bằng `read_file` rồi tính nhẩm sai quy tắc nghiệp vụ.
- **Báo cáo sai định dạng và thiếu thông tin quan trọng**: Nếu `references/report-template.md` không được đồng bộ, báo cáo Markdown do model tạo ra sẽ không có bảng tổng giờ làm việc theo người, không có danh sách người quá tải và thiếu bảng chi tiết các dòng bị loại kèm lý do, khiến người dùng không nhận được giá trị phân tích mới của hệ thống.

