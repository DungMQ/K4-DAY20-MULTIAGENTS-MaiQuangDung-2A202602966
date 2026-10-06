# Báo cáo Lab: Self evolving Agentic

## 1. Thông tin nhóm và cấu hình

| Họ tên | Mã sinh viên | Phần đóng góp |
|---|---|---|
| Mai Quang Dũng | 2A202602966 | Harness, thiết kế subagent, chạy thí nghiệm, tự sinh skill và phân tích báo cáo |

- Mô hình: `openai:gpt-4o-mini`; `LAB_TEMPERATURE=0`; `recursion_limit=60`.
- Môi trường: Python 3.11, `deepagents==0.7.21`, Windows; chạy trực tiếp trong `.venv`, công cụ POSIX từ Git `usr/bin`.
- Có 21 lần chạy tác vụ có bản ghi: 18 kết quả chính thức và 3 kết quả phát triển tại `results/skills-auto-dev/`. Thêm một lượt chạy lại `logs-learn` bị dừng khi chờ API quá lâu, không có bản ghi mới; số lượt đã biết là ít nhất 22/30 theo ngân sách ghi trong bản nháp. Không suy ra các lượt cũ không còn lưu từ số thư mục hiện tại.
- Curator: bản nháp ghi nhận 1 lần gọi, 3 skill được giữ, không xóa skill; không có bản ghi riêng để kiểm chứng số lần gọi curator.
- Commit giả thuyết: `64879ef` (`hypotheses`); commit tag `freeze`: `ee859ac399143b281520595f7bcb542c826918d8`, lúc 17:39:04 ngày 06/10/2026, múi giờ Việt Nam.
- Kiểm tra cuối: `32 passed`; `checked 6 runs of skill conditions: OK`. Không sửa mã được cung cấp, bộ test, workspace gốc hoặc nội dung skill đã đóng băng khi hoàn thiện báo cáo.

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

Bảng dưới dùng toàn bộ check thất bại của ba tác vụ học ở `results/baseline/`. Nhãn G của `tests_not_modified` là vấn đề môi trường xuống dòng, được kiểm chứng bên dưới; không quy kết tác tử đã sửa test.

| Tác vụ | Check thất bại | Nhóm lỗi (A–G) | Bằng chứng từ `detail` hoặc vết |
|---|---|---|---|
| `code-learn` | `tests_not_modified` | G: xuống dòng Windows | the original files in tests/ must not be modified (new test files are allowed) |
| `code-learn` | `parse_price_all_formats` | D: định dạng số | wrong for: ['(12.00)'] |
| `code-learn` | `csv_quoting_follows_docstring` | D: định dạng CSV | to_csv_row returned 'Desk, large "oak",10.00,2' |
| `code-learn` | `rule_type_hints` | E: quy ước tổ chức | RULE: every public function (name not starting with '_') in the package has type annotations on all parameters and on the return value. |
| `code-learn` | `rule_regression_tests` | E: quy ước tổ chức | RULE: add tests/test_regressions.py with one test function per bug you fixed (at least 3); the file must pass. |
| `code-learn` | `rule_changelog` | E: quy ước tổ chức | RULE: record each fix in CHANGELOG.md under the heading '## Unreleased' as a bullet '- fix(<function name>): <short description>' (at least 3 bullets). |
| `data-learn` | `north_q1_revenue` | B: ghi giá trị giữ chỗ, chưa kiểm chứng | north_q1_revenue: wrong value (got 0) |
| `data-learn` | `north_q1_orders` | B: ghi giá trị giữ chỗ, chưa kiểm chứng | north_q1_orders: wrong value (got 0) |
| `data-learn` | `missing_amount_orders` | B: ghi giá trị giữ chỗ, chưa kiểm chứng | missing_amount_orders: wrong value (got 0) |
| `data-learn` | `duplicate_rows_removed` | B: ghi giá trị giữ chỗ, chưa kiểm chứng | duplicate_rows_removed: wrong value (got 0) |
| `data-learn` | `rule_money_in_cents` | E: quy ước tổ chức | RULE: money values in answer.json are integer cents (1606.67 USD is written 160667). |
| `data-learn` | `rule_meta_block` | E: quy ước tổ chức | RULE: answer.json has an object `meta` = {"source": <input file name>, "rows_in": <number of data rows in the input file, duplicates included>, "rows_used": <number of distinct orders with a known amount>}. |
| `data-learn` | `rule_clean_csv` | E: quy ước tổ chức | RULE: write workspace/clean.csv with the header order_id,timestamp_utc,region,amount_cents; one row per distinct order with a known amount; timestamp_utc as YYYY-MM-DDTHH:MM:SSZ (UTC); region in canonical spelling (North, South, East, West); amount in integer cents. |
| `logs-learn` | `entry_count` | D: xử lý log | wrong number of entries (got 10) |
| `logs-learn` | `timestamps_utc` | D: xử lý log | 4/25 timestamps match |
| `logs-learn` | `exception_fields` | D: xử lý log | 21 wrong `exception` values |
| `logs-learn` | `repeat_counts` | D: xử lý log | 21 wrong `repeat_count` values |
| `logs-learn` | `counts_by_service` | D: xử lý log | counts_by_service: wrong values |
| `logs-learn` | `rule_service_names` | E: quy ước tổ chức | RULE: service names in the output are lower-case with '-' replaced by '_' (payment-service -> payment_service). |
| `logs-learn` | `rule_sorted_errors` | E: quy ước tổ chức | RULE: `errors` is sorted by service, then by timestamp_utc, ascending. |
| `logs-learn` | `rule_schema_header` | E: quy ước tổ chức | RULE: the top-level object has "schema_version": 2 and "generated_by": "log-triage". |

Có 21 check thất bại: E có 9, D có 7, B có 4 và G có 1. E là nhóm lớn nhất nhưng không chiếm hơn một nửa; E và D cộng lại là 16/21. Baseline đạt 6/18 check kỹ thuật và 0/9 check quy ước trên tập học, nên dữ liệu không ủng hộ nhận xét rằng phần kỹ thuật đã gần hoàn chỉnh.

Ở `data-learn`, vết ghi nhận lệnh Python một dòng bị lỗi cú pháp, rồi `ModuleNotFoundError: No module named 'pandas'`; sau đó tác tử ghi `answer.json` với các giá trị 0 và nói rõ đó là giá trị giữ chỗ. Bốn check sai được xếp B vì kết quả chưa được tính/kiểm chứng; bản thân lỗi thiếu thư viện là vấn đề môi trường và không được tính như bằng chứng tác tử xử lý sai múi giờ. Tác tử có thể chọn thư viện chuẩn hoặc báo chưa hoàn thành thay vì coi file giữ chỗ là đầu ra hợp lệ.

Check `tests_not_modified` có kết quả âm ngay cả khi file được sao chép nguyên trạng: SHA-256 của `test_report.py` trong bản Git là `79e05f4c...e00d`, đúng hằng số checker; file trên Windows có 32 cặp CRLF, hash byte là `efb5e765...f19`. Chỉ chuyển CRLF thành LF trong bộ nhớ cho lại đúng hash gốc. Vết baseline không có lệnh sửa test. Đây là sai khác byte do checkout Windows, không phải bằng chứng vi phạm chỉ dẫn. Giữ nguyên kết quả chấm đã ghi; đề xuất dùng checkout LF/WSL cho thí nghiệm tiếp theo.

Chưa có bằng chứng rõ để quy check nào cho C (vá triệu chứng). Việc đọc docstring và sửa hàm dùng chung trong vết `code-learn` cũng là bằng chứng rằng tác tử không hoàn toàn bỏ đọc đặc tả. Skill có thể giúp các nhóm B, D, E nếu được đọc và chứa quy tắc đủ cụ thể; đây là kỳ vọng cần kiểm chứng, chưa phải hiệu quả đã quan sát.

## 5. Điều kiện `subagents` (Phần 2.3)

Ba vai trò trong `src/lab/subagents.py`: `explorer` đọc và báo cáo trước khi thay đổi; `implementer` thực hiện thay đổi khi có yêu cầu rõ; `reviewer` kiểm tra độc lập và không sửa file. Cách chia này tách khám phá, thực hiện và kiểm tra. `build_agent` nối `PATHS_NOTE` vào prompt của từng subagent vì chúng không tự nhận `BASE_PROMPT` của tác tử chính.

| Tác vụ | `subagent_calls` | Quan sát |
|---|---:|---|
| code-learn | 0 | Tác tử chính tự đọc file; điểm 0/10 |
| data-learn | 1 | Gọi `implementer` sau khi lệnh dùng pandas thất bại; điểm 3/8 |
| logs-learn | 0 | Tác tử chính tự tạo JSON; điểm 1/9 |
| code-eval | 0 | Không có giao việc trong vết chính |
| data-eval | 0 | Chạm giới hạn 60 bước, không có giao việc |
| logs-eval | 0 | Không có giao việc trong vết chính |

Ở `data-learn`, thông điệp giao việc truyền `workspace/sales.csv`, `workspace/answer.json`, năm chỉ số, khoảng thời gian UTC, cách xử lý `-999` và yêu cầu chuẩn hóa dữ liệu. Tuy nhiên, câu “following Acme's reporting conventions” không giải thích quy ước cụ thể; quy ước ẩn vốn chưa có trong instruction nên không thể kỳ vọng tác tử tự biết. Báo cáo của `implementer` cho hai chỉ số Q1 sai, còn tác tử chính không gọi reviewer hoặc chạy kiểm chứng sau khi nhận báo cáo. Vết chỉ chứa lời gọi và báo cáo cuối của subagent; không quan sát đầy đủ các bước bên trong.

`data-learn` tăng từ 1/8 lên 3/8, token tăng từ 33,388 lên 132,264 (3.96 lần), thời gian từ 18.6 lên 80.1 giây (4.31 lần). Hai check tăng là `missing_amount_orders` và `duplicate_rows_removed`; chưa chứng minh chính việc giao việc là nguyên nhân vì mỗi cấu hình chỉ chạy một lần. Trung bình tập học của subagents lại thấp hơn baseline (0.1620 so với 0.2120). Với 5/6 tác vụ không gọi subagent, nhãn cấu hình `subagents` không đồng nghĩa cả sáu lượt đều thực hiện đa tác tử.

## 6. Self-evolving: skill do curator sinh (Phần 3)

Giữ nguyên ba skill đã được curator sinh trước freeze. Mỗi file có 10 dòng: 4 dòng frontmatter và 6 dòng hướng dẫn. Cả ba qua `validate_skill`; hash bộ skill là `26ed3ea235bb7fd63b6f4914cbed48361d952801a0b297c5b5b9a1a8debd89a7`. Định dạng hợp lệ không bảo đảm nội dung đầy đủ hoặc an toàn về ngữ nghĩa.

| Skill | Mức tổng quát | Đánh giá nội dung | `description` và sử dụng |
|---|---|---|---|
| `adhere-to-docstring-specifications` | Tổng quát cho lập trình theo đặc tả | Đọc docstring, kiểm tra biên và viết test là hợp lý. Câu yêu cầu cập nhật docstring nếu hành vi thay đổi có thể khiến tác tử đổi đặc tả để hợp thức hóa code sai; trong lab phải giữ đặc tả. | Kích hoạt khi implement hàm, nhưng không nêu rõ sửa bug. 10 dòng; chưa có lần đọc được ghi nhận. |
| `avoid-parallel-file-mutations` | Tổng quát cho thao tác file | Tránh sửa cùng file đồng thời là hợp lý; yêu cầu mọi file phải được sửa bằng thao tác riêng khá hạn chế và không cần thiết cho các file độc lập. Không xử lý được lỗi dữ liệu, quy ước tiền hoặc schema. | Kích hoạt khi thay đổi nhiều file; 10 dòng; chưa có lần đọc được ghi nhận. |
| `maintain-changelog-and-test-requirements` | Tổng quát cho sửa bug và kiểm thử | Có hướng dẫn changelog và regression test nhưng bỏ sót type hints, tên file test bắt buộc và cú pháp changelog cụ thể trong phản hồi tập học. Không có quy ước cho data/logs. | Kích hoạt khi sửa bug/thay đổi; khá rộng nhưng thiếu các từ khóa data/logs. 10 dòng; chưa có lần đọc được ghi nhận. |

Phần 3.4 lưu tại `results/skills-auto-dev/`: điểm code/data/logs lần lượt 1/10, 1/8, 1/9; cả ba có `skills_read=0`. Sáu kết quả chính thức cũng đều có `skills_read=0`; không có lời gọi `read_file` tới `skills/.../SKILL.md` trong vết. Có thể metadata tên/description vẫn nằm trong system prompt theo cơ chế progressive disclosure, nhưng chưa có bằng chứng phần thân skill được áp dụng. Cơ chế chỉ nạp metadata trước rồi đọc nội dung khi cần được mô tả trong [tài liệu Skills của LangChain](https://docs.langchain.com/oss/python/deepagents/skills).

Test offline xác nhận `build_agent(use_skills=True)` nạp metadata và chỉ dẫn `FIRST action`. Dữ liệu thực tế cho thấy mô hình không làm theo yêu cầu đọc skill; không đủ bằng chứng để quy nguyên nhân duy nhất cho description hoặc loader. Không sửa tay skill để cải thiện kết quả sau freeze.

## 7. Kết quả so sánh (Phần 4.3, 4.4)

Bảng sau được sinh bằng `lab.compare.build_table(load_runs())`, tương đương đầu ra `python -m lab.compare` và khớp `report/table.md`:

| Task | baseline | subagents | skills-auto |
|---|---|---|---|
| code-learn | 4/10 | 0/10 | 0/10 |
| data-learn | 1/8 | 3/8 | 0/8 |
| logs-learn | 1/9 | 1/9 | 1/9 |
| code-eval | 1/11 | 2/11 | 1/11 |
| data-eval | 0/9 | 0/9 | 0/9 |
| logs-eval | 2/10 | 1/10 | 1/10 |
| **Mean score - learning tasks** | 0.21 | 0.16 | 0.04 |
| **Mean score - evaluation tasks** | 0.10 | 0.09 | 0.06 |
| **Mean tokens per run** | 125,539 | 108,679 | 105,573 |
| **Runs that read a skill** | 0/6 | 0/6 | 0/6 |

Điểm trung bình là trung bình điểm của từng tác vụ, không phải tỷ lệ tất cả check gộp lại. Thống kê kỹ thuật/quy ước từ `scripts/check_breakdown.py`:

```text
condition     role    technical  house rules  mean tokens  read a skill
baseline      eval      2/18         1/12         221,625      0/3
baseline      learn     6/18         0/9           29,453      0/3
subagents     eval      3/18         0/12         159,961      0/3
subagents     learn     4/18         0/9           57,396      0/3
skills-auto   eval      2/18         0/12         103,373      0/3
skills-auto   learn     1/18         0/9          107,774      0/3
```

Các lượt có `error` được giữ trong bảng chính để phản ánh kết quả dưới cùng giới hạn thực thi; không diễn giải chúng thành thất bại riêng của skill hoặc quy ước. Cả 18 lượt có `skills_modified=false`.

| Điều kiện | Tác vụ | Lỗi | Điểm |
|---|---|---|---|
| `baseline` | `code-eval` | `GraphRecursionError`, giới hạn 60 bước | 1/11 |
| `baseline` | `data-eval` | `GraphRecursionError`, giới hạn 60 bước | 0/9 |
| `skills-auto` | `code-eval` | `GraphRecursionError`, giới hạn 60 bước | 1/11 |
| `skills-auto` | `data-learn` | `GraphRecursionError`, giới hạn 60 bước | 0/8 |
| `subagents` | `data-eval` | `GraphRecursionError`, giới hạn 60 bước | 0/9 |

Có 5/18 lượt chạm giới hạn 60 bước. Không tăng giới hạn riêng cho một điều kiện hoặc lựa chọn lại kết quả theo điểm. `verify_freeze.py` ban đầu báo duy nhất `skills-auto/logs-learn` chạy trước tag; kết quả cũ đã nằm trong `skills-auto-dev`. Chạy lại đúng tác vụ này với cùng mô hình/nhiệt độ/giới hạn bước, timeout mỗi yêu cầu 25 giây và `max_retries=0`, rồi dùng bản ghi thành công sau freeze. Timeout chỉ giới hạn chờ API, không đổi prompt hoặc skill. Lượt chạy bằng CLI trước đó bị dừng khi chờ API quá lâu, không dùng để tính điểm. Kiểm tra cuối báo `checked 6 runs of skill conditions: OK`.

## 8. Phân tích

### 8.1. So sánh điểm và giả thuyết

| Vai trò | baseline | subagents | skills-auto |
|---|---:|---:|---:|
| Học, điểm trung bình | 0.2120 | 0.1620 | 0.0370 |
| Đánh giá, điểm trung bình | 0.0970 | 0.0939 | 0.0636 |

Không điều kiện nào cải thiện điểm trung bình so với baseline ở cả tập học hoặc tập đánh giá. Subagents giảm 0.0500 trên tập học và khoảng 0.0030 trên tập đánh giá; skills-auto giảm 0.1750 và khoảng 0.0333. Cải thiện riêng ở `subagents/data-learn` không chuyển thành cải thiện trung bình hay thành công ở `data-eval` (0/9).

H1 chỉ phù hợp ở dự đoán điểm gần tương đương trên tập đánh giá; dự đoán tăng token 1.5–3 lần không được xác nhận trên tập đánh giá, do baseline có hai lượt lặp dài. H2 không được xác nhận: skills-auto không cải thiện cả quy ước lặp lại. H3 không được xác nhận với skills-auto: điểm học 0.0370 thấp hơn điểm đánh giá 0.0636. Giữ nguyên H1–H3 đã commit trước freeze, không viết lại dự đoán theo kết quả đã thấy.

Căn cứ bổ sung khi diễn giải: subagent giúp cô lập ngữ cảnh nhưng cần thông tin giao việc đầy đủ theo [tài liệu Subagents của LangChain](https://docs.langchain.com/oss/python/deepagents/subagents). Tài liệu tham khảo này được đối chiếu khi hoàn thiện báo cáo; không phải bằng chứng nó đã được dẫn trong commit giả thuyết ban đầu. Phân tích cơ chế của phiên bản 0.7.21 dựa trực tiếp vào `scripts/tour.py`, test offline và vết thực nghiệm.

### 8.2. Check kỹ thuật và quy ước

Tập học: baseline đạt 6/18 kỹ thuật, subagents 4/18, skills-auto 1/18; cả ba đều 0/9 quy ước. Tập đánh giá: lần lượt 2/18, 3/18, 2/18 kỹ thuật và 1/12, 0/12, 0/12 quy ước. Không có bằng chứng skill giúp nhóm quy ước nào. Tỷ lệ kỹ thuật bị ảnh hưởng bởi check hash byte trên Windows, xử lý dữ liệu thiếu kiểm chứng và những lượt hết bước; không dùng các con số này như ước lượng thuần khả năng suy luận.

Các check mới `rule_version_bump`, `rule_sorted_keys_format`, `rule_source_line` đều không đạt trong skills-auto. Skill chỉ sinh từ phản hồi tập học, không có hướng dẫn về những quy ước mới này; đồng thời phần thân skill cũng chưa được đọc. Hai nguyên nhân này không thể tách tác động bằng dữ liệu hiện có.

### 8.3. Một check đạt và một check không đạt dưới cấu hình skill

Check `valid_structure` ở `skills-auto/logs-learn` đạt, nhưng baseline cũng đạt và không có đọc skill; chỉ có thể nói tác tử đã viết JSON đúng cấu trúc, không thể nói skill giúp nó đạt. Check `rule_changelog` ở `skills-auto/code-learn` không đạt dù có skill về changelog; `skills_read=0` và vết không có thao tác ghi changelog. Skill còn thiếu cú pháp chính xác `## Unreleased` và `- fix(...)`, nên kể cả đọc nó cũng chưa bảo đảm vượt check.

Vết chính thức `skills-auto/code-learn` chỉ có `ls`/`read_file`, rồi câu trả lời cuối nói đã cập nhật `parse_price`; không có `edit_file` hoặc `write_file`, và checker vẫn thấy các lỗi gốc. Đây là bằng chứng bổ sung cho nhóm F (báo hoàn thành sai sự thật). Không dùng một câu trả lời tự nhận đã sửa file để suy ra skill được làm theo.

### 8.4. Chi phí và hiệu quả

| Điều kiện | Token trung bình/học | Giây trung bình/học | Token trung bình/đánh giá | Giây trung bình/đánh giá | Điểm trung bình đánh giá / 100,000 token |
|---|---:|---:|---:|---:|---:|
| baseline | 29,453 | 17.5 | 221,626 | 72.6 | 0.0438 |
| subagents | 57,397 | 34.2 | 159,962 | 49.5 | 0.0587 |
| skills-auto | 107,774 | 21.8 | 103,373 | 47.2 | 0.0616 |

Định nghĩa hiệu quả ở cột cuối là tỷ số điểm trung bình với token trung bình, chuẩn hóa theo 100,000 token; không phải giá API hoặc tỷ lệ thành công. Skills-auto cao nhất theo tỷ số này (0.0616), nhưng điểm tuyệt đối thấp nhất và không đọc skill: chỉ số tốt hơn chủ yếu vì dùng ít token hơn baseline. Trên tập học, baseline tốt nhất: 0.7199 so với subagents 0.2823 và skills-auto 0.0344.

Trên tập đánh giá, subagents dùng ít token hơn baseline khoảng 27.8%, skills-auto khoảng 53.4%; baseline bị hai lượt hết bước kéo chi phí lên. Không thể kết luận đa tác tử tiết kiệm chi phí nói chung vì không lượt đánh giá nào thực sự gọi subagent. Với trường hợp duy nhất có giao việc (`data-learn`), tăng 2 check phải trả gần 4 lần token và hơn 4 lần thời gian, chưa cho thấy lợi ích đủ rõ trong thí nghiệm này.

### 8.5. Rò rỉ và quá khớp

Curator lọc `role == 'learn'`; test offline xác nhận prompt không chứa dữ liệu đánh giá và skill có tên không hợp lệ bị bỏ. Cả ba skill qua kiểm tra marker, không có đáp án hoặc nội dung riêng của tập đánh giá. Lịch sử có commit `hypotheses` trước tag; hash skill khớp và cả sáu lượt skills-auto chính thức chạy sau freeze. Những bằng chứng này hỗ trợ quy trình tách học/đánh giá, nhưng không chứng minh tuyệt đối chưa ai từng mở tài liệu đánh giá ngoài các bản ghi được lưu.

Nội dung skill thiên về code, thiếu các quy ước data/logs từ chính tập học. Đây là thiếu bao phủ, chưa phải bằng chứng quá khớp vào đáp án. Không quan sát được mẫu “skills-auto tăng điểm học nhưng giảm điểm đánh giá”; phần thân skill không được đọc nên chưa kiểm nghiệm được khả năng tổng quát hóa của chính các hướng dẫn.

### 8.6. Dao động khi dùng cùng bộ skill

| Tác vụ học | Phần 3.4 (`skills-auto-dev`) | Sau freeze (`skills-auto`) | Chênh lệch điểm |
|---|---:|---:|---:|
| code-learn | 1/10 | 0/10 | -0.1000 |
| data-learn | 1/8 | 0/8 | -0.1250 |
| logs-learn | 1/9 | 1/9 | +0.0000 |

Trung bình giảm từ 0.1120 xuống 0.0370, chênh lệch -0.0750 với cùng hash skill. Cả hai đợt đều không đọc skill. `data-learn` sau freeze có `GraphRecursionError`, nên chênh lệch gồm dao động hành vi và lỗi thực thi, không chỉ ngẫu nhiên thuần túy. Riêng `logs-learn` giữ điểm 1/9 nhưng nội dung/check chi tiết và token có thể khác. Chênh lệch 0.0750 lớn hơn khoảng cách trung bình đánh giá baseline–subagents (0.0030); đây là dấu hiệu các chênh lệch nhỏ thiếu ổn định, không phải khoảng tin cậy hoặc kiểm định thống kê.

## 9. Hạn chế và tính hợp lệ

1. Chỉ ba tác vụ cho mỗi vai trò và ba họ do giảng viên thiết kế; kết luận không đại diện cho mọi bài toán kỹ thuật, dữ liệu hoặc log thực tế.
2. Mỗi cặp điều kiện/tác vụ chính thức chỉ có một lượt; temperature 0 vẫn không bảo đảm hành vi lặp lại. Hai đợt tập học cùng skill khác nhau 0.0750 điểm trung bình, chưa đủ mẫu để tính độ tin cậy.
3. Chỉ một mô hình nhỏ (`gpt-4o-mini`); chưa so sánh với mô hình khác nên không thể quy kết giới hạn cho Deep Agents hoặc thiết kế skill nói chung.
4. Windows gây sai khác hash test do CRLF; thiếu pandas và cú pháp lệnh shell một dòng còn ảnh hưởng kết quả. Đây là yếu tố nhiễu môi trường, không thể loại khỏi số liệu cũ bằng cách sửa điểm thủ công.
5. Có 5 lượt hết giới hạn bước và 0/6 lượt chính thức đọc skill; thí nghiệm kiểm chứng việc cấu hình harness tốt hơn là hiệu quả nhân quả của nội dung skill. Cấu hình subagents cũng chỉ có một lượt thật sự giao việc.
6. Vết chỉ lưu luồng chính, mỗi khối bị cắt ở 1,500 ký tự; không thấy đầy đủ bước nội bộ subagent và system prompt thực tế. `tool_calls`/`subagent_calls` chỉ đếm luồng chính, trong khi token callback có cộng subagent; hai chỉ số không cùng phạm vi.
7. Một lượt logs sau freeze dùng timeout API 25 giây, không retry để tránh chờ kéo dài; phần suy luận vẫn giữ cùng cấu hình nhưng thời gian toàn trình không hoàn toàn đồng nhất với lượt cũ. Không dùng dao động thời gian làm kết luận tốc độ của skill.

## 10. Kết luận

Harness đạt 32 test offline, có đủ 18 kết quả chính thức và quy trình freeze đạt kiểm tra. Baseline có điểm trung bình cao nhất ở cả tập học và tập đánh giá trong dữ liệu hiện có. Skill tự sinh chưa được đọc, còn subagent chỉ được gọi một lần, nên chưa có bằng chứng hai cơ chế này cải thiện chất lượng. Các lượt hết bước, sai khác xuống dòng và dao động giữa hai đợt hạn chế sức mạnh của kết luận. Bước tiếp theo là giữ checkout LF trong WSL/Linux, kiểm tra việc đọc skill trên tập học trước freeze mới và chạy lặp trong một thí nghiệm riêng.

## Phụ lục

Chuỗi lệnh tái lập quy trình (lệnh học trước freeze; không chạy lại curator trên skill đã đóng băng khi chỉ kiểm tra bài nộp):

```bash
python -m pytest
python scripts/tour.py
python -m lab.runner --condition baseline --tasks data-learn
python -m lab.runner --condition baseline --tasks code-learn logs-learn
python -m lab.runner --condition subagents --tasks learn
python -m lab.curator
python -m lab.runner --condition skills-auto --tasks learn
# Sao lưu results/skills-auto thành results/skills-auto-dev trước khi chạy chính thức.
git add -A
git commit -m "hypotheses"
git commit --allow-empty -m "freeze skills"
git tag freeze
python -m lab.runner --condition baseline --tasks eval
python -m lab.runner --condition subagents --tasks eval
python -m lab.runner --condition skills-auto --tasks all
python scripts/verify_freeze.py
python -m lab.compare > report/table.md
python scripts/check_breakdown.py
```

Đây là chuỗi lệnh hướng dẫn tái lập; trình tự thực tế được đối chiếu bằng timestamp. Tập chính thức skills-auto đã có code/data-learn sau freeze nhưng logs-learn còn giữ bản trước freeze; lượt bổ sung chỉ chạy logs-learn. Không tạo tag mới, không di chuyển tag cũ và không thay đổi giả thuyết trong commit lịch sử.

Kiểm tra trên PowerShell Windows cần UTF-8 để `verify_freeze.py` đọc báo cáo tiếng Việt từ Git:

```powershell
$env:PYTHONUTF8 = '1'
$env:PYTHONIOENCODING = 'utf-8'
.\.venv\Scripts\python.exe -m pytest
.\.venv\Scripts\python.exe scripts/verify_freeze.py
.\.venv\Scripts\python.exe scripts/check_breakdown.py
```

Trong môi trường thực thi bị hạn chế quyền, pytest cần `--basetemp` trỏ tới một thư mục mới có quyền ghi; các binary Git POSIX cần chạy ngoài sandbox hạn chế signal pipe. Bộ test không bị sửa để vượt qua lỗi quyền môi trường. Lượt kiểm chứng cuối dùng thư mục tạm mới trong `.venv` và đạt `32 passed`.

Lượt bổ sung logs-learn gọi `run_task('logs-learn', 'skills-auto', ...)` với `ChatOpenAI` dùng lại model, key và nhiệt độ từ cấu hình có sẵn, `timeout=25`, `max_retries=0`; kết quả thành công được chuyển vào thư mục chính thức. Key không được đưa vào báo cáo. Ba bản ghi Phần 3.4 được giữ trong `results/skills-auto-dev/` để kiểm chứng dao động.

Không thực hiện thử thách mở rộng Phần 6. Tài liệu tham khảo: `GUIDE.md`, `RUBRIC.md`, `guides/pseudocode/05_skill_quality.md`, kết quả `scripts/tour.py` của phiên bản đã cài, cùng hai trang chính thức về Skills và Subagents đã dẫn ở mục 6 và 8. Các tài liệu trực tuyến được đối chiếu khi hoàn thiện báo cáo, không được trình bày như tài liệu đã tồn tại trong commit giả thuyết.
