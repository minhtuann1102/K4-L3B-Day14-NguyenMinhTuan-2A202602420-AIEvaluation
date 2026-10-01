# Day 14 — Exercises

## AI Evaluation & Benchmarking · Lab Worksheet

**Thời gian làm bài:** 9:15–12:00

**Domain:** OrbitTech Store Customer Support

Điền trực tiếp câu trả lời vào file này. Golden dataset 20 QA được viết một lần
duy nhất trong `golden_dataset.json`, không chép lại toàn bộ vào Markdown.

---

Từ 9:15–9:30, cài môi trường và chạy baseline tests theo `guide_lab.md`.

---

## Part 1 — Warm-up (9:30–9:45)

### Exercise 1.1 — RAGAS Metric Thresholds

Theo bài giảng:

- 0.8–1.0: Good — monitor, maintain.
- 0.6–0.8: Needs work — analyze failures, iterate.
- Dưới 0.6: Significant issues — investigate.

Với từng metric, xác định khi nào score thấp có thể chấp nhận và khi nào là
critical.

| Metric | Acceptable Low Score Scenario | Critical Low Score Scenario | Action Required |
|---|---|---|---|
| Faithfulness |Khi bot chào hỏi xã giao hoặc thêm câu chúc khách hàng (những câu này hiển nhiên không có trong tài liệu đối chiếu). |Bot tự bịa chính sách đổi trả, bịa thời hạn bảo hành hoặc báo sai thông số kỹ thuật sản phẩm. |Khóa prompt bằng câu lệnh nghiêm ngặt "chỉ trả lời dựa trên context được cấp", hạ temperature về 0. |
| Answer Relevance |Khách hỏi trêu, hỏi ngoài lề và bot từ chối trả lời lịch sự (refusal đúng yêu cầu). |Khách hỏi về phí ship hay địa chỉ shop nhưng bot lại đi giải thích chính sách bảo hành. |Chỉnh lại prompt hướng dẫn bám sát câu hỏi, thêm bước phân loại ý định (intent classification). |
| Context Recall |Câu hỏi chỉ cần đúng 1 ý nhỏ là đủ kết luận (như kiểm tra xem shop có mở cửa Chủ Nhật không). |Khách hỏi điều kiện bảo hành pin nhưng hệ thống retrieve thiếu mất tài liệu chính sách pin. |Tăng số chunk lấy về (top_k), tăng kích thước chunk size và overlap để không bị đứt đoạn thông tin. |
| Context Precision |Lấy về nhiều đoạn văn bản, đoạn chứa đáp án đúng nằm ở vị trí thứ 2 hoặc 3 thay vì đứng đầu. |Mấy đoạn rác/nhiễu bị xếp lên top 1 khiến bot đọc nhầm và hiểu sai ngữ cảnh của khách. |Thêm bước Rerank (xếp hạng lại) để kéo đoạn thông tin liên quan nhất lên đầu. |
| Completeness |	Khách chỉ cần câu trả lời nhanh dạng xác nhận Có/Không, không cần lôi hết quy định ra đọc. |Khách hỏi các bước gửi hàng bảo hành mà bot chỉ chỉ được bước 1 rồi ngưng, thiếu mất 3 bước sau. |Thêm ví dụ mẫu (few-shot) trong prompt, yêu cầu bot trả lời dạng gạch đầu dòng để không bị sót ý. |

### Exercise 1.2 — Bias trong LLM-as-a-Judge

Ba bias thường gặp:

- Position bias: judge ưu tiên answer xuất hiện trước.
- Verbosity bias: judge ưu tiên answer dài hơn.
- Self-preference: judge ưu tiên output giống chính model đó.

**Câu 1: Thiết kế experiment phát hiện position bias với ít nhất hai conditions.**

> *Câu trả lời:* 
Mình sẽ chuẩn bị 1 tập câu hỏi và 2 câu trả lời A, B.

Lần 1: Cho LLM chấm theo thứ tự A trước, B sau.
Lần 2: Giữ nguyên văn bản nhưng đảo thứ tự lại thành B trước, A sau.
Nếu ở cả 2 lần, câu nào đứng ở vị trí đầu tiên cũng đều được chấm điểm cao hơn rõ rệt (dù nội dung không đổi), thì chắc chắn judge đang bị Position bias. Khi đó giải pháp là chạy cả 2 chiều rồi lấy điểm trung bình.

**Câu 2: Làm thế nào giảm verbosity bias bằng rubric design?**

> *Câu trả lời:* 
Thêm hẳn tiêu chí "ngắn gọn, súc tích" vào barem điểm. Trong rubric ghi rõ: chỉ cho điểm tối đa nếu trả lời đúng trọng tâm và không thừa thãi. Nếu câu trả lời dài dòng, lặp ý hoặc chém gió lan man thì trừ bớt 1–2 điểm.

**Câu 3: Tại sao cần calibrate LLM judge với human labels?**

> *Câu trả lời:* Vì LLM judge rất hay bị bias ngầm (như model OpenAI thường có xu hướng chấm điểm cao cho văn phong của chính OpenAI, hoặc model hay chấm nới tay). Cần so sánh điểm của LLM với điểm do người thật chấm để biết mức độ tin cậy được bao nhiêu %, từ đó mới dám giao cho nó tự động duyệt code/prompt.

### Exercise 1.3 — Evaluation trong CI/CD

**Câu 1: Chọn threshold để block deployment.**

| Metric | Threshold | Lý do |
|---|---:|---|
| Faithfulness |0.80 |	Làm bot chăm sóc khách hàng mà trả lời bịa đặt là toang ngay, rất dễ bị khách khiếu nại nên tiêu chí này phải siết chặt nhất. |
| Answer Relevance |0.75 |	Đảm bảo bot hiểu đúng và trả lời trúng câu hỏi của khách, tránh tình trạng "hỏi một đằng trả lời một nẻo". |
| Completeness |0.70 | Cần đủ ý chính cho khách hiểu, nhưng vẫn châm chước được vì khách thường sẽ nhắn hỏi thêm nếu chưa rõ.|

**Câu 2: Khi nào dùng offline evaluation, online evaluation và human review?**

> *Câu trả lời:*
Offline eval: Chạy tự động trong CI/CD trước khi deploy. Mỗi lần dev sửa prompt hoặc đổi code retriever thì chạy qua tập test 20 câu để xem điểm có bị tụt không.
Online eval: Chạy khi bot đã lên live thực tế. Theo dõi xem khách có bấm nút dislike (thumbs-down) không, hay có bao nhiêu người bực mình đòi gặp nhân viên hỗ trợ trực tiếp.
Human review: Định kỳ hàng tuần hoặc khi thấy có ca khách đánh giá 1 sao, người thật sẽ mở log ra đọc lại toàn bộ hội thoại để tìm nguyên nhân gốc rễ và bổ sung ca lỗi đó vào tập test dataset.

---

## Part 2 — Core Coding (9:45–10:40)

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

Thiết kế rubric domain-specific cho OrbitTech Customer Support. Mỗi mức phải
đủ cụ thể để hai người chấm độc lập có thể hiểu giống nhau.

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

### Exercise 3.4 — Framework Comparison (Bonus +5)

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
