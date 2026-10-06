# Báo cáo Lab: Self evolving Agentic

## 1. Thông tin nhóm và cấu hình

| Họ tên | Mã sinh viên | Phần đóng góp |
|---|---|---|
| Mai Quang Dũng | 2A202602966 | 100% (Hoàn thiện harness, chạy đo lường, thiết kế subagents, tự sinh skill, phân tích và báo cáo) |

- Nhà cung cấp và mô hình: `openai:gpt-4o-mini`, nhiệt độ (`LAB_TEMPERATURE`): 0, `recursion_limit`: 60
- Phiên bản Deep Agents: `deepagents==0.7.21`, hệ điều hành: Windows 11 (tích hợp POSIX utilities qua Git/usr/bin), chạy trực tiếp.
- Số lần chạy tác vụ đã dùng / ngân sách: 15 / 30
- Commit của tag `freeze`: (Sẽ cập nhật sau khi gắn tag)

## 2. Giả thuyết (commit TRƯỚC tag `freeze`, Phần 4.0)

- H1 (subagents so với baseline): Dự đoán điều kiện `subagents` sẽ có điểm tương đương hoặc chỉ nhỉnh hơn nhẹ so với `baseline` trên các tác vụ đánh giá, nhưng tiêu thụ lượng token cao hơn từ 1.5x đến 3x và thời gian thực thi lâu hơn. Căn cứ từ tập học: việc chia nhỏ bài toán sang subagent làm phân mảnh ngữ cảnh (subagent là stateless, chỉ nhìn thấy nội dung được chuyển tiếp), tác tử chính thường ưu tiên tự thực hiện các bước để tránh rủi ro mất mát thông tin hoặc gặp khó khăn khi tổng hợp kết quả.
- H2 (skills-auto so với baseline): Dự đoán điều kiện `skills-auto` sẽ đạt điểm cải thiện đáng kể trên các tác vụ có quy ước kỹ thuật lặp lại từ tập học (như quản lý changelog, viết regression test, tuân thủ docstring), nhưng sẽ KHÔNG cải thiện được các quy ước kiểm tra mới phát sinh riêng trong tập đánh giá. Căn cứ: curator chỉ tổng hợp kinh nghiệm từ vết thất bại của tập học; các quy ước tổ chức mới ở tập đánh giá chưa hề xuất hiện trong dữ liệu huấn luyện nên skill tự sinh không thể bao quát trước.
- H3 (tác vụ học so với tác vụ đánh giá): Dự đoán điểm trung bình trên tập học sẽ cao hơn đáng kể so với tập đánh giá trên điều kiện `skills-auto`, trong khi ở `baseline` và `subagents` độ chênh lệch sẽ nhỏ hơn. Căn cứ: điều kiện `skills-auto` được tối ưu hóa theo phản hồi lỗi của tập học nên sẽ có hiện tượng thích ứng ngữ cảnh cụ thể (domain context adaptation); khi sang tập đánh giá với dữ liệu mới và quy ước bổ sung mới, khoảng cách điểm sẽ thể hiện rõ ranh giới giữa khả năng tổng quát hóa kỹ thuật và các quy tắc đặc thù.

## 3. Làm quen Deep Agents (Phần 0.3)

1. Tác tử mặc định có 9 công cụ:
   - Nhóm thao tác tệp: `ls`, `read_file`, `write_file`, `edit_file`, `delete`, `glob`, `grep`.
   - Shell: `execute`.
   - Đa tác tử (subagents): `task`.
   Công cụ cho phép chạy lệnh hệ thống là `execute`.
2. Mô tả của công cụ `task` cho biết subagent `general-purpose` là một tác tử đa năng dùng để nghiên cứu câu hỏi phức tạp, tìm kiếm file/nội dung và thực hiện các tác vụ nhiều bước khi tác tử chính chưa tự tin tìm đúng ngay. Subagent này có đầy đủ các công cụ như tác tử chính.
   Về mặt ngữ cảnh: Subagent mặc định là phi trạng thái (stateless) - nó CHỈ nhìn thấy prompt được tác tử chính truyền sang (`the agent sees only the prompt you give it and returns a single final report`) và không nhìn thấy lịch sử hội thoại trước đó của tác tử chính trừ khi được kế thừa rõ ràng. Báo cáo của subagent không được hiển thị trực tiếp cho người dùng mà do tác tử chính tổng hợp lại.
3. Trích dẫn câu hướng dẫn hành vi:
   - Từ mô tả công cụ `task`: *"Tell the agent whether to create content, analyze, or only research, since it can't necessarily see the user's intent unless it inherits your conversation, as noted per agent type below."*
   - Từ mô tả công cụ `execute`: *"You MUST avoid using search commands like find and grep. Instead use the grep, glob tools to search. Use read_file rather than cat/head/tail."*

## 4. Đường cơ sở và phân loại lỗi (Phần 2.2)

| Tác vụ | Check thất bại | Nhóm lỗi (A-G) | Bằng chứng (trích ngắn từ `detail` hoặc vết) |
|---|---|---|---|
| `code-learn` | `tests_not_modified` | A | `the original files in tests/ must not be modified (new test files are allowed)` |
| `code-learn` | `parse_price_all_formats` | D | `wrong for: ['(12.00)']` (chưa xử lý định dạng số âm đặt trong ngoặc đơn) |
| `code-learn` | `csv_quoting_follows_docstring` | D | `to_csv_row returned 'Desk, large "oak",10.00,2'` (chưa escape dấu ngoặc kép theo docstring) |
| `code-learn` | `rule_type_hints` | E | `RULE: every public function ... has type annotations on all parameters and on the return value.` |
| `code-learn` | `rule_regression_tests` | E | `RULE: add tests/test_regressions.py with one test function per bug you fixed (at least 3)` |
| `code-learn` | `rule_changelog` | E | `RULE: record each fix in CHANGELOG.md under the heading '## Unreleased'` |
| `data-learn` | `north_q1_revenue` | D | `north_q1_revenue: wrong value (got 0)` (thiếu thư viện xử lý dữ liệu phức tạp / tính sai múi giờ và ngày) |
| `data-learn` | `north_q1_orders` | D | `north_q1_orders: wrong value (got 0)` |
| `data-learn` | `missing_amount_orders` | D | `missing_amount_orders: wrong value (got 0)` |
| `data-learn` | `duplicate_rows_removed` | D | `duplicate_rows_removed: wrong value (got 0)` |
| `data-learn` | `rule_money_in_cents` | E | `RULE: money values in answer.json are integer cents (1606.67 USD is written 160667).` |
| `data-learn` | `rule_meta_block` | E | `RULE: answer.json has an object meta = {"source": <input file name>, ...}` |
| `data-learn` | `rule_clean_csv` | E | `RULE: write workspace/clean.csv with the header order_id,timestamp_utc,region,amount_cents...` |
| `logs-learn` | `entry_count` | D | `wrong number of entries (got 10)` |
| `logs-learn` | `timestamps_utc` | D | `4/25 timestamps match` |
| `logs-learn` | `exception_fields` | D | `21 wrong exception values` |
| `logs-learn` | `repeat_counts` | D | `21 wrong repeat_count values` |
| `logs-learn` | `counts_by_service` | D | `counts_by_service: wrong values` |
| `logs-learn` | `rule_service_names` | E | `RULE: service names in the output are lower-case with '-' replaced by '_'` |
| `logs-learn` | `rule_sorted_errors` | E | `RULE: errors is sorted by service, then by timestamp_utc, ascending.` |
| `logs-learn` | `rule_schema_header` | E | `RULE: the top-level object has "schema_version": 2 and "generated_by": "log-triage".` |

Nhận xét:
- Nhóm lỗi chiếm đa số tuyệt đối là **Nhóm E (Vi phạm quy ước tổ chức - convention rules)** và **Nhóm D (Bỏ sót định dạng dữ liệu bẩn/biên dữ liệu)**.
- Các lỗi Nhóm E xảy ra ở 100% các check có tiền tố `rule_*` vì những quy ước này hoàn toàn không nằm trong `instruction.md` ban đầu của đề bài mà thuộc về hệ thống kiểm định nội bộ của tổ chức.
- Một bộ skill tự sinh có thể **phòng ngừa rất hiệu quả nhóm lỗi E và một phần nhóm D**, bởi vì curator có thể đọc các thông báo `RULE:` từ phản hồi thất bại của lần chạy trước để đúc kết thành các hướng dẫn rõ ràng trong `SKILL.md` (ví dụ quy định cập nhật CHANGELOG, viết test hồi quy, chuẩn hóa định dạng, kiểm tra docstring).

## 5. Điều kiện `subagents` (Phần 2.3)

- Các subagent đã định nghĩa:
  1. `explorer`: Chuyên gia khám phá code base và dữ liệu, đọc tệp, tìm kiếm từ khóa và tóm tắt cấu trúc. Chỉ đọc và phân tích, không sửa đổi dữ liệu.
  2. `implementer`: Chuyên gia thực thi và lập trình, áp dụng các thay đổi chính xác vào mã nguồn, tệp cấu hình và ghi dữ liệu đầu ra theo yêu cầu.
  3. `reviewer`: Chuyên gia đánh giá và kiểm thử, chạy test suites, kiểm tra cú pháp, xác thực định dạng file đầu ra và đối chiếu với các quy ước trước khi hoàn tất.
- `subagent_calls` ở từng tác vụ:
  - `code-learn`: 0 cuộc gọi. Tác tử chính nhận thấy có thể đọc file và dùng `edit_file` trực tiếp nên không chia việc.
  - `data-learn`: 1 cuộc gọi. Tác tử chính gọi subagent để thực hiện đọc và phân tích file `sales.csv`.
  - `logs-learn`: 0 cuộc gọi. Tác tử chính tự đọc `app.log` và ghi `errors.json`.
- Thông tin khi giao việc: Tác tử chính đã truyền tóm tắt yêu cầu phân tích dữ liệu nhưng do subagent không nhận `BASE_PROMPT` và ngữ cảnh trước đó, subagent cần nhận đầy đủ chỉ dẫn về đường dẫn tương đối và quy ước đầu ra.
- Ảnh hưởng đến token và thời gian: Trong `data-learn`, `subagents` tiêu thụ 132,264 tokens (so với 33,388 tokens của baseline - gấp ~4 lần) và mất 80.1s (so với 18.6s của baseline). Điểm số tăng từ 1/8 lên 3/8, cho thấy có cải thiện điểm nhờ subagent nhưng chi phí tính toán tăng mạnh.

## 6. Self-evolving: skill do curator sinh (Phần 3)

- Số lần chạy curator: 1 lần. Số skill bị xóa: 0 skill (tất cả 3 skill sinh ra đều đạt chuẩn định dạng và có giá trị áp dụng thực tiễn).

| Skill | Tổng quát hay riêng cho tác vụ học? | Đúng hay sai (nêu chỗ sai nếu có) | Độ dài, `description` và `skills_read` ở Phần 3.4 |
|---|---|---|---|
| `adhere-to-docstring-specifications` | Tổng quát cho việc lập trình và kiểm thử theo docstring | Đúng: hướng dẫn đọc kỹ tham số, kiểu trả về và viết test kiểm chứng | 10 dòng; kích hoạt khi implement hàm theo docstring; `skills_read = 0` |
| `avoid-parallel-file-mutations` | Tổng quát cho việc quản lý chỉnh sửa tệp tin | Đúng: hướng dẫn sửa tệp tuần tự, kiểm tra từng bước tránh ghi đè lỗi | 10 dòng; kích hoạt khi sửa đổi nhiều tệp; `skills_read = 0` |
| `maintain-changelog-and-test-requirements` | Tổng quát cho quy trình release và regression testing | Đúng: hướng dẫn ghi nhận changelog, viết test regression chuyên biệt | 10 dòng; kích hoạt khi sửa lỗi và kiểm thử; `skills_read = 0` |

## 7. Kết quả so sánh (Phần 4.3, 4.4)

(Sẽ cập nhật sau khi chạy hoàn tất tập đánh giá và đóng băng skill)

## 8. Phân tích

(Sẽ hoàn thiện sau khi có bảng số liệu mục 7)

## 9. Hạn chế và tính hợp lệ

1. Số lượng tác vụ nhỏ (3 họ tác vụ, 6 bài toán) khiến kết quả có thể chịu phương sai lớn từ tính ngẫu nhiên của mô hình ngôn ngữ (temperature = 0 vẫn có thể có độ trôi nhẹ do batching hoặc API provider).
2. Tác tử chỉ chạy 1 lần trên mỗi cấu hình (single-run), chưa thực hiện lặp lại nhiều seed để tính khoảng tin cậy thống kê chuẩn xác.
3. Quy ước tổ chức (`RULE:*`) được thiết kế đặc thù trong bộ test của lab, do đó khả năng học của curator phụ thuộc mạnh vào việc thông báo lỗi từ lần chạy trước có mô tả chi tiết quy ước hay không.

## 10. Kết luận

(Sẽ hoàn thiện sau khi tổng hợp toàn bộ kết quả)

## Phụ lục

- Lệnh đã chạy:
  1. `pytest tests/test_01_provided.py tests/test_02_agent.py tests/test_03_runner.py tests/test_04_curator.py`
  2. `python -m lab.runner --condition baseline --tasks data-learn`
  3. `python -m lab.runner --condition baseline --tasks code-learn logs-learn`
  4. `python -m lab.runner --condition subagents --tasks learn`
  5. `python -m lab.curator`
  6. `python -m lab.runner --condition skills-auto --tasks learn`
