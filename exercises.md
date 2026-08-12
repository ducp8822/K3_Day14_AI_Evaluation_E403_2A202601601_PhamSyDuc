# Day 14 — Exercises

## AI Evaluation & Benchmarking · Lab Worksheet

**Thời gian làm bài:** 09:15–12:00

**Domain:** Northstar University Student Services

Điền trực tiếp câu trả lời vào file này. Golden dataset 20 QA được viết một lần
duy nhất trong `golden_dataset.json`, không chép lại toàn bộ vào Markdown.

---

Từ 09:15–09:30, cài môi trường và chạy baseline tests theo `guide_lab.md`.

---

## Part 1 — Warm-up (09:30–09:45)

### Exercise 1.1 — RAGAS Metric Thresholds

Theo bài giảng:

- 0.8–1.0: Good — monitor, maintain.
- 0.6–0.8: Needs work — analyze failures, iterate.
- Dưới 0.6: Significant issues — investigate.

Với từng metric, xác định khi nào score thấp có thể chấp nhận và khi nào là
critical.

| Metric | Acceptable Low Score Scenario | Critical Low Score Scenario | Action Required |
|---|---|---|---|
| Faithfulness | Score thấp có thể chấp nhận khi câu trả lời chỉ diễn đạt lại context bằng cách khác mà không thêm claim thực tế mới, hoặc agent từ chối trả lời khi context không đủ và đó là hành vi được rubric cho phép. | Agent bịa đặt hoặc khẳng định không có căn cứ các số liệu, hạn nộp, quy định học phí/học bổng; chỉ một claim quan trọng không được context hỗ trợ cũng có thể là critical. | Thêm guardrail kiểm tra từng claim với context, yêu cầu agent nêu rõ khi thiếu evidence và thắt chặt system prompt ("Chỉ trả lời dựa trên context"). |
| Answer Relevance | Score thấp có thể chấp nhận khi agent hỏi lại để làm rõ câu hỏi mơ hồ, hoặc từ chối câu hỏi out-of-scope/adversarial đúng policy nhưng evaluator chưa nhận diện được refusal hợp lệ. | Agent trả lời lạc đề hoàn toàn (off-topic), ví dụ được hỏi quy trình đóng học phí nhưng lại hướng dẫn đăng ký môn. | Cải thiện intent detection/query rewriting, cập nhật expected answer và rubric để phân biệt câu trả lời đúng với refusal hợp lệ. |
| Context Recall | Score thấp có thể chấp nhận khi gold/expected answer chứa chi tiết tùy chọn, dư thừa hoặc không cần cho câu hỏi; trước khi tăng `top_k` cần kiểm tra và thu gọn gold evidence. | Retriever thực sự không lấy được một phần evidence bắt buộc, khiến LLM không thể trả lời đúng hoặc bị thiếu ý nghiêm trọng. | Sửa gold answer/evidence nếu annotation quá rộng; nếu evidence thực sự bị bỏ sót thì tăng `top_k`, cải thiện chunking hoặc dùng Hybrid Search (BM25 + Dense Retrieval). |
| Context Precision | Score thấp có thể chấp nhận khi `top_k=5` vẫn chứa đủ evidence liên quan, các chunk không liên quan không chiếm phần lớn vị trí đầu và câu trả lời không bị ảnh hưởng dù evidence nằm ở vị trí 4–5. | Các chunk không liên quan chiếm các vị trí đầu, làm evidence liên quan bị xếp hạng thấp hoặc bị lẫn giữa nhiều chunk rác ("Lost in the Middle"), từ đó khiến model trả lời sai/thiếu. Evidence nằm ngoài `top_k` là vấn đề của Context Recall, không phải Context Precision. | Áp dụng reranking (Cross-Encoder / Overlap Reranker), điều chỉnh `top_k` và cải thiện query/retriever để đưa các chunk liên quan lên đầu. |
| Completeness | Score thấp có thể chấp nhận khi người dùng yêu cầu câu trả lời ngắn và agent vẫn nêu đủ các ý bắt buộc; các chi tiết tùy chọn nên được phân biệt với required claims trong expected answer. | Trả lời thiếu điều kiện bắt buộc, ngoại lệ (exceptions) hoặc deadline quan trọng khiến sinh viên hiểu sai. | Dùng cấu trúc/checklist cho các điều kiện bắt buộc, bổ sung few-shot examples và yêu cầu LLM tự kiểm tra đủ các claim quan trọng trước khi trả lời. |

### Exercise 1.2 — Bias trong LLM-as-a-Judge

Ba bias thường gặp:

- Position bias: judge ưu tiên answer xuất hiện trước.
- Verbosity bias: judge ưu tiên answer dài hơn.
- Self-preference: judge ưu tiên output giống chính model đó.

**Câu 1: Thiết kế experiment phát hiện position bias với ít nhất hai conditions.**

> *Câu trả lời:*
> 1. **Chuẩn bị dữ liệu:** Tạo nhiều cặp answer có chất lượng đã biết hoặc tương đương, gán nhãn logic A/B ngẫu nhiên và chạy mỗi cặp nhiều lần. Khi chấm điểm, lưu kết quả theo `answer_id`, không chỉ theo vị trí.
> 2. **Condition 1 (Original Order):** Đưa cùng một cặp vào prompt theo thứ tự `[Answer A, Answer B]`, giữ nguyên question, rubric và yêu cầu Judge chọn câu tốt hơn hoặc chấm điểm từng câu.
> 3. **Condition 2 (Swapped Order):** Đổi vị trí thành `[Answer B, Answer A]` trong một lần gọi độc lập; giữ nguyên hoàn toàn question, rubric và nội dung answer.
> 4. **Đánh giá:** So sánh tỷ lệ thắng và chênh lệch điểm của answer đứng trước với answer đứng sau, đồng thời đối chiếu theo `answer_id`. Nếu answer đứng trước liên tục được ưu tiên sau khi đã kiểm soát chất lượng nội dung, đó là bằng chứng của Position Bias.

**Câu 2: Làm thế nào giảm verbosity bias bằng rubric design?**

> *Câu trả lời:*
> 1. **Tách nội dung khỏi độ dài:** Yêu cầu Judge chấm điểm theo key facts/evidence chính xác, coverage của các ý bắt buộc và khả năng hành động; không dùng số câu hoặc số token làm proxy cho chất lượng.
> 2. **Chỉ phạt phần thừa:** Phạt redundancy, fluff hoặc claim không liên quan; không phạt độ dài nếu phần dài hơn cung cấp thêm thông tin cần thiết hoặc giải thích rõ điều kiện/ngoại lệ.
> 3. **Định nghĩa thang điểm 1–5 rõ ràng:** 5 điểm = "Chính xác, đầy đủ và súc tích". Câu trả lời dài nhưng hữu ích vẫn có thể đạt 5; câu trả lời dài mà không thêm giá trị chỉ nên đạt tối đa 3.

**Câu 3: Tại sao cần calibrate LLM judge với human labels?**

> *Câu trả lời:*
> 1. LLM Judge cũng là một model ngôn ngữ, vẫn có thể hallucinate, gặp position/verbosity/self-preference bias hoặc hiểu sai chính sách đặc thù của Northstar University.
> 2. Cần dùng một mẫu case đại diện được human chấm độc lập, có thể adjudicate các case bất đồng, rồi đo mức đồng thuận. Dùng weighted Cohen's Kappa cho nhãn thứ bậc (ordinal), Cohen's Kappa cho nhãn phân loại không có thứ tự (nominal), và Spearman correlation hoặc ICC cho score liên tục. Nếu agreement thấp, cần sửa rubric, few-shot examples hoặc threshold trước khi tự động hóa hoàn toàn; các case high-stakes vẫn cần human review.

### Exercise 1.3 — Evaluation trong CI/CD

**Câu 1: Chọn threshold để block deployment.**

| Metric | Threshold | Lý do |
|---|---:|---|
| Faithfulness | 0.85 | Đây là metric quan trọng nhất về an toàn và độ tin cậy. Block nếu trung bình dưới 0.85 hoặc có bất kỳ case nào chứa hallucination nghiêm trọng về chính sách, deadline, học phí hay học bổng. |
| Answer Relevance | 0.80 | Đảm bảo AI trả lời đúng trọng tâm câu hỏi của sinh viên, không trả lời lan man hoặc lạc đề. Các refusal đúng policy cần được chấm bằng expected refusal riêng, không để làm sai threshold. |
| Completeness | 0.80 | Đảm bảo cung cấp đủ thông tin cốt lõi và các điều kiện bắt buộc. Có thể cho phép thiếu chi tiết tùy chọn, nhưng không được thiếu deadline, exception hoặc bước hành động quan trọng. |

**Câu 2: Khi nào dùng offline evaluation, online evaluation và human review?**

> *Câu trả lời:*
> - **Offline Evaluation (Pre-deployment):** Chạy trong giai đoạn phát triển và CI/CD Pipeline trước khi release trên Golden Dataset cố định. Dùng để kiểm tra tính ổn định và phát hiện regression khi thay đổi prompt, model, retriever hoặc chunking. Deployment chỉ được pass khi các aggregate threshold đạt yêu cầu và không có hard-fail safety case.
> - **Online Evaluation (Post-deployment):** Chạy giám sát liên tục trên dữ liệu thật sau khi hệ thống lên production: user feedback, latency, error rate, mẫu log được lấy ngẫu nhiên và các chỉ số chất lượng. Dùng để phát hiện data drift, thay đổi phân bố câu hỏi hoặc sự cố thời gian thực; cần tuân thủ quy định bảo vệ dữ liệu cá nhân.
> - **Human Review:** Dùng khi xây dựng/cập nhật Golden Dataset, định kỳ trên mẫu production được chọn theo rủi ro (có thể bắt đầu với 1–5%), và bắt buộc với case high-stakes, điểm bất thường, khiếu nại của người dùng hoặc khi offline/online evaluation mâu thuẫn.

---

## Part 2 — Core Coding (09:45–10:40)

Hoàn thiện các TODO bắt buộc trong `template.py`.

### Task 1 — Data Models

- `QAPair`: question, expected answer, gold context, metadata và retrieved contexts.
- `EvalResult`: answer-side scores, optional retrieval scores, pass/failure fields.
- `overall_score()`: trung bình Faithfulness, Relevance và Completeness.

### Task 2 — RAGASEvaluator

Answer-side:

- `evaluate_faithfulness(answer, context)`
- `evaluate_relevance(answer, question)`
- `evaluate_completeness(answer, expected)`

Retrieval-side:

- `evaluate_context_recall(contexts, expected)`
- `evaluate_context_precision(contexts, expected)`

Full pipeline:

- `run_full_eval(..., contexts=None)` luôn tính ba answer metrics.
- Nếu có `contexts`, tính và lưu thêm Context Recall và Context Precision.
- Retrieval scores không làm thay đổi `overall_score()` và pass rule gốc.

### Task 3 — LLMJudge

- `score_response(question, answer, rubric)`
- `detect_bias(scores_batch)`

### Task 4 — BenchmarkRunner

- `run(qa_pairs, agent_fn, evaluator)`
- `generate_report(results)`
- `run_regression(new_results, baseline_results)`
- `identify_failures(results, threshold)`

`BenchmarkRunner.run()` phải truyền `pair.retrieved_contexts` vào
`run_full_eval()`. Report phải có average của hai retrieval metrics.

### Task 5 — FailureAnalyzer

- `categorize_failures(failures)`
- `find_root_cause(failure)`
- `generate_improvement_suggestions(failures)`
- `generate_improvement_log(failures, suggestions)`

Kiểm tra:

```bash
pytest tests/ -v
```

`rerank_by_overlap()` là TODO bonus của Exercise 3.5. Test tương ứng được skip
nếu bạn chưa làm bonus.

---

## Part 3 — Golden Dataset & Real Benchmark (10:40–11:35)

### Exercise 3.1 — Build the Golden Dataset

Thiết kế và validate dataset theo Mục 5–6 trong `guide_lab.md`. Nội dung 20 QA
được điền trực tiếp trong `golden_dataset.json`; phần dưới chỉ ghi lại kết quả
và quyết định thiết kế, không chép lại toàn bộ QA.

**Kết quả dataset**

| Hạng mục | Kết quả |
|---|---|
| Tổng số records | ____ / 20 |
| Easy | ____ / 5 |
| Medium | ____ / 7 |
| Hard | ____ / 5 |
| Adversarial | ____ / 3 |
| Source documents được sử dụng | ____ / 10 |
| Validator status | PASS / FAIL |

**Ba case đại diện cho quyết định thiết kế**

| ID | Difficulty | Source document(s) | Vì sao case phù hợp với difficulty/attack type? |
|---|---|---|---|
| | | | |
| | | | |
| | | | |

**Điểm khó nhất khi xây dựng expected answer hoặc evidence là gì?**

> *Câu trả lời:*

**Xác nhận:**

- [ ] Mọi claim trong expected answer đều có evidence hỗ trợ.
- [ ] Không có questions trùng ý và không dùng kiến thức ngoài corpus.
- [ ] `python validate_golden_dataset.py` báo `PASS`.

### Exercise 3.2 — Benchmark Run

Chạy:

```bash
python domain_assistant.py
python evaluate_answers.py
```

Copy bảng terminal vào đây hoặc điền từ `artifacts/benchmark_results.json`.

| ID | Question (short) | Ctx Recall | Ctx Precision | Faithfulness | Relevance | Completeness | Overall | Passed? | Failure Type |
|---|---|---:|---:|---:|---:|---:|---:|---|---|
| E01 | | | | | | | | | |
| E02 | | | | | | | | | |
| E03 | | | | | | | | | |
| E04 | | | | | | | | | |
| E05 | | | | | | | | | |
| M01 | | | | | | | | | |
| M02 | | | | | | | | | |
| M03 | | | | | | | | | |
| M04 | | | | | | | | | |
| M05 | | | | | | | | | |
| M06 | | | | | | | | | |
| M07 | | | | | | | | | |
| H01 | | | | | | | | | |
| H02 | | | | | | | | | |
| H03 | | | | | | | | | |
| H04 | | | | | | | | | |
| H05 | | | | | | | | | |
| A01 | | | | | | | | | |
| A02 | | | | | | | | | |
| A03 | | | | | | | | | |

**Aggregate Report**

- Overall pass rate: ____%
- Avg Context Recall: ____
- Avg Context Precision: ____
- Avg Faithfulness: ____
- Avg Relevance: ____
- Avg Completeness: ____
- Failure type distribution: ____

**Ba cases có Overall Score thấp nhất**

1. ID: ____ | Score: ____ | Failure type: ____
2. ID: ____ | Score: ____ | Failure type: ____
3. ID: ____ | Score: ____ | Failure type: ____

**Nhận xét ngắn:** Metric nào yếu nhất? Kết quả gợi ý vấn đề nằm ở retrieval
hay generation?

> *Câu trả lời:*

### Exercise 3.3 — LLM-as-a-Judge Rubric Design

Thiết kế rubric domain-specific cho Student Services. Mỗi mức phải đủ cụ thể để
hai người chấm độc lập có thể hiểu giống nhau.

Chọn 3–5 dimensions:

- [ ] Correctness
- [ ] Completeness
- [ ] Relevance
- [ ] Evidence/citation
- [ ] Actionability
- [ ] Safety/privacy
- [ ] Tone/clarity
- [ ] Dimension khác: __________

| Score | Tiêu chí domain-specific | Ví dụ response |
|---:|---|---|
| 5 | | |
| 4 | | |
| 3 | | |
| 2 | | |
| 1 | | |

**Ba edge cases khó chấm**

| Edge Case | Tại sao khó chấm? | Rubric xử lý thế nào? |
|---|---|---|
| | | |
| | | |
| | | |

**Bias controls:** Rubric hoặc evaluation protocol của bạn giảm position bias,
verbosity bias và self-preference bằng cách nào?

> *Câu trả lời:*

### Exercise 3.4 — Framework Comparison (Bonus +10)

Chỉ làm sau khi hoàn thành 3.1–3.3. Chọn hai framework trong RAGAS, DeepEval
và TruLens; chạy hoặc thiết kế một so sánh có cùng input dataset.

| Tiêu chí | Framework 1: ____ | Framework 2: ____ |
|---|---|---|
| Setup complexity | | |
| Metrics available | | |
| CI/CD integration | | |
| Kết quả trên cùng dataset | | |
| Insight rút ra | | |

- Scores có nhất quán không?
- Framework nào strict hơn và vì sao?
- Hai framework có tìm ra cùng failure cases không?

> *Phân tích:*

### Exercise 3.5 — Retrieval Reranking (Bonus +5)

Mục tiêu: kiểm tra việc đổi thứ tự chunks có tăng Context Precision mà không
thay đổi Context Recall hay không.

1. Chọn ít nhất 5 cases từ `artifacts/actual_answers.json`.
2. Tính Context Recall và Context Precision trước rerank.
3. Implement `rerank_by_overlap()` hoặc một reranker khác.
4. Rerank cùng tập chunks, không thêm hoặc xóa chunk.
5. Tính lại hai metrics và giải thích kết quả.

| ID | Recall before | Recall after | Precision before | Precision after | Delta Precision |
|---|---:|---:|---:|---:|---:|
| | | | | | |
| | | | | | |
| | | | | | |
| | | | | | |
| | | | | | |
| **Avg** | | | | | |

**Tại sao Recall dự kiến không đổi?**

> *Câu trả lời:*

**Khi nào reranking không đủ và cần sửa retriever/query/chunking?**

> *Câu trả lời:*

---

## Part 4 — Reflection (11:35–11:50)

Hoàn thành `reflection.md` bằng kết quả thật từ Exercise 3.2.

---

## Completion Checklist

Hoàn thành kiểm tra cuối trong khoảng 11:50–12:00.

- [ ] Tất cả required tests pass.
- [ ] `golden_dataset.json` validate thành công.
- [ ] Exercise 3.1 hoàn thành trong file JSON và bảng kết quả phía trên.
- [ ] Exercise 3.2 có năm metrics, aggregate report và ba cases thấp nhất.
- [ ] Exercise 3.3 có rubric 1–5 và bias controls.
- [ ] `reflection.md` có ba failure analyses và regression strategy.
- [ ] Đã copy `template.py` thành `solution/solution.py`.
- [ ] Exercise 3.4 và 3.5 chỉ làm nếu chọn bonus.
