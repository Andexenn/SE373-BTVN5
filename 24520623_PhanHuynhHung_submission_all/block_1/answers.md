# Comprehensive Answers to Lab Assignment Questions
## Bảng trả lời câu hỏi bài tập (Block 1 & Block 2)

---

## Block 1: Policy Lookup by Version (Tra cứu chính sách đúng phiên bản)

### Question 1:
> **Why do we need a tool to list files and a skill to guide policy selection?**  
> *(Vì sao cần tool để tìm file và skill để hướng dẫn chọn chính sách?)*

**Answer:**
1. **Tool to list files (`list_files`) provides Environmental Perception *(Khả năng cảm nhận môi trường)*:**
   - In dynamic environments, documents may be added, removed, or renamed at runtime *(thời gian chạy)*.
   - Without an inspection tool like `list_files`, the agent operates blindly *(hoạt động mù quáng)* and must rely on hardcoded filenames predetermined by developers.
   - `list_files` allows the agent to discover files dynamically in `data/policies/` without hardcoding specific filenames in system prompts or tool source code.

2. **Skill to guide policy selection (`refund-policy`) provides Procedural Knowledge *(Kiến thức quy trình nghiệp vụ)*:**
   - While tools provide low-level primitives *(các thao tác nguyên thủy cơ bản)* like reading and writing bytes, skills encapsulate multi-step domain workflows *(quy trình nghiệp vụ nhiều bước)*:
     - Checking if all required inputs (purchase date, refund request date, activation status) are present before concluding;
     - Reading policy validity intervals and selecting the exact policy based on purchase date (not request date);
     - Calculating calendar day differences accurately;
     - Formatting the final response according to the defined template (`answer-template.md`).
   - Using skills leverages **Progressive Disclosure *(Tiết lộ lũy tiến / nạp theo nhu cầu)***: Only metadata is kept in the initial system prompt catalog, saving context tokens and preventing context pollution until the skill is actually needed.

---

### Question 2:
> **If the agent does not yet have a tool to find files, can modifying the prompt solve the renamed file requirement? Explain.**  
> *(Nếu agent chưa có tool tìm file, việc sửa prompt có giải quyết được yêu cầu đổi tên file không? Giải thích.)*

**Answer:**
- **NO, prompt engineering cannot solve this requirement.**
- **Explanation:**
  - A system prompt or user prompt is **static context *(ngữ cảnh tĩnh)*** established prior to execution.
  - If a policy file is renamed at runtime (e.g. from `policy-from-oct.md` to `chinh-sach-tu-thang-10.md` or a UUID-based filename), a static prompt cannot anticipate or discover the new filename.
  - The model has no psychic ability *(khả năng thần giao cách cảm / đoán trước)* to divine what files exist on disk without a perception tool.
  - Attempting to solve this by modifying the prompt would require a human to inspect the disk and manually hardcode the new filename into the prompt each time a file changes. This completely violates the **Autonomous Agency *(tính tự chủ của tác tử AI)*** paradigm, turning an autonomous agent back into a brittle, human-dependent system.

---

## Block 2: Workload Overload Inspection (Kiểm tra quá tải theo người)

### Question 1:
> **Which part is calculated by the script and which part is interpreted by the model?**  
> *(Phần nào do script tính, phần nào do model diễn giải?)*

**Answer:**
1. **Part calculated by the script (`check_csv.py` - Deterministic Computation / Tính toán tất định):**
   - **Data Parsing & Schema Validation *(Phân tích cú pháp & xác thực lược đồ)***: Checking column structure, verifying header names, and identifying field count mismatches.
   - **ID Deduplication Rule *(Quy tắc khử trùng lặp mã công việc)***: Registering the first occurrence of each `task_id` (`first_seen`) and rejecting all subsequent duplicates, regardless of whether the first occurrence was valid.
   - **Exclusion Categorization *(Phân loại dòng bị loại)***: Systematically recording each rejected line with standardized reason codes (`wrong_field_count`, `missing_task_id`, `duplicate_id`, `missing_owner`, `invalid_hours`).
   - **Arithmetic Aggregation & Threshold Logic *(Tổng hợp số học & logic ngưỡng)***: Accurately summing valid hours for each owner and evaluating `total_hours > max_hours`.
   - **Rationale**: LLMs are notorious for arithmetic inaccuracies, off-by-one errors, and hallucinations *(ảo giác)* when computing sums over multi-line tables. Offloading deterministic math to a Python script guarantees 100% mathematical precision and reproducibility.

2. **Part interpreted by the model (Cognitive Orchestration & Synthesis / Điều phối nhận thức & tổng hợp):**
   - **Intent Understanding *(Hiểu ý định người dùng)***: Parsing natural language requests (e.g. "Ai làm vượt quá 8 giờ?").
   - **Catalog Matching & Tool Invocation *(Khớp danh mục kỹ năng & gọi công cụ)***: Identifying the relevant skill (`csv-quality`), inspecting its `SKILL.md`, and invoking `bash` with CLI arguments (`--input`, `--max-hours`).
   - **Interaction Protocol *(Giao thức tương tác)***: Detecting missing threshold parameters and pausing to ask the user clarification questions instead of guessing.
   - **Report Formatting & Advisory Recommendations *(Định dạng báo cáo & đưa ra khuyến nghị)***: Taking the structured JSON output from stdout and synthesizing it into an executive Markdown report adhering to `report-template.md`, including actionable insights (e.g., suggesting fixes for invalid hours or missing owners).

---

### Question 2:
> **If we modify the script but do not update the skill and references, what errors or omissions could occur in the report?**  
> *(Nếu sửa script nhưng không cập nhật skill và reference, báo cáo có thể sai hoặc thiếu thông tin gì?)*

**Answer:**
1. **Execution Failure *(Lỗi thực thi / crash chương trình)***:
   - The updated `check_csv.py` script makes `--max-hours` a mandatory argument. If `SKILL.md` is not updated, the agent will execute the old command format: `python check_csv.py --input data/workload.csv`.
   - The script will fail immediately with non-zero exit code (exit 2) and error output on stderr (`the following arguments are required: --max-hours`), preventing the agent from receiving any analysis data.
2. **Loss of Interactive Boundary Guardrails *(Mất khả năng chặn lỗi khi thiếu thông tin)***:
   - Without the guidance in `SKILL.md`, if the user does not specify a threshold (e.g., "Kiểm tra ai quá tải?"), the agent would either hallucinate a default threshold (e.g., 8 hours) or try reading the raw CSV directly via `read_file`, computing incorrect hours by eye.
3. **Missing Critical Report Sections *(Báo cáo thiếu các mục thông tin trọng yếu)***:
   - If `references/report-template.md` is not synchronized, the agent will follow the obsolete template:
     - **Missing Owner Workload Table *(Thiếu bảng tổng giờ theo người)***: The report would not list `hours_by_owner`.
     - **Missing Overload Determination *(Thiếu kết luận người quá tải)***: The report would fail to highlight `overloaded_owners`.
     - **Missing Excluded Rows Audit Table *(Thiếu bảng kiểm tra các dòng bị loại)***: The report would omit the table detailing rejected rows with standardized reason codes.
   - The final output would only show generic file quality counts without fulfilling the business requirement of workload analysis.
