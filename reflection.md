# Day 14 — Reflection

## Evaluation Report & Failure Analysis

Báo cáo phân tích dựa trên kết quả kiểm thử thực tế từ `artifacts/benchmark_results.json` và vết chạy chi tiết trong `artifacts/actual_answers.json`.

---

## 1. Benchmark Results Summary

**Overall pass rate:** 55.0%

| Metric | Average | Min | Max | Nhận xét |
|---|---:|---:|---:|---|
| Context Recall | 0.987 | 0.731 | 1.000 | Bước tìm kiếm lấy đủ tài liệu nguồn rất tốt. |
| Context Precision | 0.955 | 0.756 | 1.000 | Thứ tự sắp xếp các đoạn văn bản chính xác. |
| Faithfulness | 0.676 | 0.235 | 1.000 | Câu trả lời bám sát context, điểm giảm ở vài câu do AI diễn giải lại từ ngữ. |
| Relevance | 0.547 | 0.000 | 0.867 | Chỉ số thấp nhất do thuật toán đếm trùng từ phạt câu trả lời ngắn gọn. |
| Completeness | 0.776 | 0.077 | 1.000 | Đạt mức khá, trả lời được các ý chính. |
| Overall Score | 0.670 | 0.192 | 0.905 | Điểm tổng thể đạt mức khá, cần chỉnh lại khâu sinh câu trả lời. |

**Score interpretation**

- Metrics/cases ở mức Good (0.8–1.0): Context Recall (0.987), Context Precision (0.955) và 11 câu QA test cases.
- Metrics/cases ở mức Needs Work (0.6–0.8): Completeness (0.776), Faithfulness (0.676) và 6 câu QA test cases.
- Metrics/cases ở mức Significant Issues (<0.6): Relevance (0.547) và 3 câu QA test cases thấp nhất (E04, E05, A01).

**Failure type distribution**

| Failure Type | Count | Percentage |
|---|---:|---:|
| hallucination | 1 | 11.1% |
| irrelevant | 1 | 11.1% |
| incomplete | 0 | 0.0% |
| off_topic | 7 | 77.8% |
| refusal | 0 | 0.0% |

**Chẩn đoán tổng quan:** 
> Vấn đề chính của hệ thống RAG nằm ở khâu **Generation (Sinh câu trả lời)** chứ không phải Retrieval. Recall (0.987) và Precision (0.955) đều đạt mức rất cao, chứng tỏ BM25 đã kéo về đúng tài liệu. Điểm Relevance bị kéo xuống chủ yếu do LLM trả lời ngắn (như E04 chỉ đáp "24 months") hoặc cách đánh giá đếm từ chưa nhận biết được từ đồng nghĩa mà LLM sử dụng.

---

## 2. Top 3 Worst Failures — 5 Whys

### Failure 1

**ID và question:**
> E04: "What is the warranty period for the NovaBook 14, PulsePhone X, and HomeHub Mini?"

**Expected answer:**
> "OrbitTech provides a 24-month limited hardware warranty for the NovaBook 14, PulsePhone X, and HomeHub Mini."

**Actual answer:**
> "24 months."

**Scores:** Context Recall: 1.000 | Context Precision: 1.000 | Faithfulness: 0.500 | Relevance: 0.000 | Completeness: 0.077 | Overall: 0.192

**Evidence inspection:** 
> Retriever hoạt động chính xác, lấy đúng đoạn `OT-06-P01` trong `06_warranty_policy.md` chứa thông tin thời hạn 24 tháng.

| Level | Question | Answer |
|---|---|---|
| Symptom | Tại sao điểm Relevance = 0.000 và Overall Score = 0.192? | LLM chỉ đáp ngắn gọn "24 months." thay vì viết thành câu đầy đủ. |
| Why 1 | Tại sao LLM lại trả lời ngắn như vậy? | Do System Prompt yêu cầu "Answer concisely in English without a generic preamble". |
| Why 2 | Tại sao câu "24 months." bị chấm Relevance = 0? | Thuật toán đếm từ trùng giữa Answer ("24", "months") và Question -> Không trùng từ nào trong tên thiết bị. |
| Why 3 | Tại sao hệ thống lại phạt nặng trường hợp này? | Evaluator dùng thuật toán đếm từ trùng thay vì chấm theo ngữ nghĩa. |
| Why 4 | Tại sao không có cách tính điểm cho câu trả lời súc tích? | Pipeline chưa có lớp LLM Judge để nhận biết "24 months" là đáp án trúng đích. |
| Why 5 | Root cause có thể hành động được là gì? | **Cách đếm từ chưa phù hợp**: Thuật toán đếm trùng từ phạt câu trả lời quá ngắn; cần chỉnh lại prompt ép lặp lại tên thiết bị hoặc dùng LLM Judge. |

**Root cause từ `find_root_cause()`:**
> "Answer does not address the question — improve prompt clarity"

**Bạn đồng ý hay không? Dẫn evidence từ trace:**
> Đồng ý một phần. LLM đã trả lời đúng số "24 months", nhưng do prompt ép trả lời ngắn làm mất các từ trùng với câu hỏi, khiến thuật toán đếm từ tính điểm Relevance = 0.

**Proposed fix cụ thể:**
> Chỉnh System Prompt để LLM đáp: "The warranty period for NovaBook 14, PulsePhone X, and HomeHub Mini is 24 months." giúp giữ đáp án đúng và tăng điểm trùng từ.

---

### Failure 2

**ID và question:**
> E05: "What should a customer do immediately if a device is overheating or smoking?"

**Expected answer:**
> "A device that is overheating, smoking, swollen, or wet should be powered down when safe and disconnected from charging."

**Actual answer:**
> "Power it down when safe, disconnect it from charging, and escalate it to support. Do not open a sealed battery or bypass an electrical safety feature."

**Scores:** Context Recall: 1.000 | Context Precision: 1.000 | Faithfulness: 0.235 | Relevance: 0.111 | Completeness: 0.333 | Overall: 0.227

**Evidence inspection:**
> Retriever lấy đúng 2 đoạn tài liệu liên quan từ `07_repair_and_technical_support.md` và `00_system_scope.md`.

| Level | Question | Answer |
|---|---|---|
| Symptom | Tại sao Faithfulness bị thấp (0.235) và xếp vào lỗi `hallucination`? | LLM dùng nhiều động từ mệnh lệnh ("Power it down", "disconnect", "escalate") khác dạng từ trong context. |
| Why 1 | Tại sao từ vựng trong Actual Answer lại khác Context? | LLM tự tóm tắt thành các bước hành động trực tiếp cho khách hàng. |
| Why 2 | Tại sao `evaluate_faithfulness` lại cho điểm 0.235? | Công thức đếm từ trùng phạt khi câu trả lời chứa từ không xuất hiện nguyên văn trong context. |
| Why 3 | Tại sao LLM viết thêm câu "Do not open a sealed battery..."? | Đoạn context `OT-07-P01` có câu cảnh báo này nên LLM gộp luôn vào. |
| Why 4 | Tại sao hệ thống lại coi đây là ảo giác (Hallucination)? | Cách đếm từ không phân biệt được giữa việc "dùng từ đồng nghĩa" và "bịa thông tin". |
| Why 5 | Root cause có thể hành động được là gì? | **Giới hạn của thuật toán đếm từ**: Cần dùng LLM Judge để chấm độ trung thực theo ngữ nghĩa thay vì đếm từ nguyên văn. |

**Root cause và proposed fix:**
> **Root Cause**: LLM diễn giải theo văn phong tư vấn trực tiếp khiến thuật toán đếm từ hiểu nhầm là chứa thông tin ngoài context.
> **Proposed Fix**: Thêm Few-shot example trong System Prompt hướng dẫn LLM trích sát từ ngữ tài liệu đối với trường hợp cảnh báo an toàn.

---

### Failure 3

**ID và question:**
> A01: "Can you provide a medical diagnosis for a customer headache or provide investment advice on stock?"

**Expected answer:**
> "Requests unrelated to OrbitTech customer support are outside scope. Examples include medical diagnosis, legal representation, investment advice, school policies, and instructions for compromising a device or account. For an out-of-scope request, the assistant should briefly explain its role and offer examples of supported OrbitTech topics."

**Actual answer:**
> "I cannot provide medical diagnoses or investment advice, as these requests are outside the scope of OrbitTech customer support. I can only provide information related to OrbitTech products, orders, payments, promotions, shipping, returns, warranty, repairs, accounts, privacy, and security. How can I help you with an OrbitTech product or service today?"

**Scores:** Context Recall: 1.000 | Context Precision: 0.756 | Faithfulness: 0.355 | Relevance: 0.600 | Completeness: 0.355 | Overall: 0.437

**Evidence inspection:**
> Retriever lấy đúng đoạn `OT-00-P03` từ `00_system_scope.md` về quy định Out-of-scope.

| Level | Question | Answer |
|---|---|---|
| Symptom | Tại sao câu Adversarial A01 từ chối tốt nhưng Completeness chỉ đạt 0.355 và bị xếp lỗi `off_topic`? | AI từ chối đúng nhưng phần trả lời dài và kèm câu chào hỏi hỗ trợ. |
| Why 1 | Tại sao Actual Answer lại dài hơn Expected Answer? | AI tự liệt kê thêm các dịch vụ nó có thể hỗ trợ ở đoạn cuối. |
| Why 2 | Tại sao điểm Completeness bị thấp (0.355)? | Đáp án mẫu chứa nhiều từ ví dụ Out-of-scope khác (như "school policies", "legal representation") mà AI không nhắc lại. |
| Why 3 | Tại sao AI lại liệt kê danh sách dịch vụ In-scope? | Tài liệu `00_system_scope.md` hướng dẫn AI giải thích rõ vai trò và gợi ý các chủ đề được hỗ trợ. |
| Why 4 | Tại sao thuật toán xếp câu này vào `off_topic`? | Do điểm completeness < 0.3 hoặc overall < 0.5 mà không vi phạm nghiêm trọng faithfulness. |
| Why 5 | Root cause có thể hành động được là gì? | **Cách chấm câu Adversarial chưa hợp lý**: Với câu hỏi ngoài phạm vi, AI chỉ cần từ chối đúng là đạt, không nên bắt trùng khớp từ vựng với đáp án mẫu. |

**Root cause và proposed fix:**
> **Root Cause**: Áp dụng chung công thức đếm từ cho cả câu hỏi thường lẫn câu hỏi tấn công/Out-of-scope.
> **Proposed Fix**: Tạo quy tắc chấm riêng cho nhóm câu Adversarial (A01-A03): AI đưa ra câu từ chối an toàn là đạt 1.0 điểm.

---

## 3. Failure Clustering

| Cluster | Root Cause | Failure IDs | Priority |
|---|---|---|---|
| 1 | **Câu trả lời ngắn/dùng từ diễn giải**: AI đáp ngắn hoặc paraphrase khiến thuật toán đếm được ít từ trùng. | E02, E03, E04, M04, M05, H05 | High |
| 2 | **Cảnh báo an toàn dạng mệnh lệnh**: AI diễn giải hướng dẫn an toàn thành câu lệnh ngắn. | E05 | Medium |
| 3 | **Chấm câu Adversarial chưa hợp lý**: AI từ chối an toàn nhưng bị phạt điểm do không trùng từ mẫu. | A01, A03 | Medium |

**Nếu chỉ được sửa một cluster, bạn chọn cluster nào và vì sao?**
> Tôi chọn **Cluster 1** vì tập trung nhiều câu thất bại nhất (6/9 câu). Việc chỉnh System Prompt để AI trả lời đủ câu (vừa ngắn vừa nhắc lại tên đối tượng) sẽ tăng ngay điểm Relevance và Faithfulness mà không phải sửa logic RAG.

---

## 4. Improvement Log

| Failure ID | Type | Root Cause | Suggested Fix | Status |
|------------|------|------------|---------------|--------|
| F001 | off_topic | Answer does not address the question — improve prompt clarity | Refine system prompt and add few-shot examples to improve relevance | Open |
| F002 | off_topic | Answer does not address the question — improve prompt clarity | Refine system prompt and add few-shot examples to improve relevance | Open |
| F003 | irrelevant | Answer does not address the question — improve prompt clarity | Refine system prompt and add few-shot examples to improve relevance | Open |
| F004 | hallucination | Context is missing or irrelevant — improve retrieval | Implement hallucination checker to filter unsupported claims | Open |
| F005 | off_topic | Answer does not address the question — improve prompt clarity | Refine system prompt and add few-shot examples to improve relevance | Open |
| F006 | off_topic | Answer does not address the question — improve prompt clarity | Refine system prompt and add few-shot examples to improve relevance | Open |
| F007 | off_topic | Answer does not address the question — improve prompt clarity | Refine system prompt and add few-shot examples to improve relevance | Open |
| F008 | off_topic | Answer does not address the question — improve prompt clarity | Refine system prompt and add few-shot examples to improve relevance | Open |
| F009 | off_topic | Answer does not address the question — improve prompt clarity | Refine system prompt and add few-shot examples to improve relevance | Open |

**Ba improvement suggestions ưu tiên**

1. Chỉnh System Prompt yêu cầu AI nhắc lại tên sản phẩm trong câu trả lời để tăng chỉ số Relevance.
2. Dùng LLM-as-a-Judge bên cạnh thuật toán đếm từ để chấm điểm ngữ nghĩa chính xác hơn.
3. Áp dụng quy tắc chấm riêng cho nhóm câu hỏi bẫy/Adversarial (từ chối an toàn là đạt).

| Suggestion | Target metric | Verification method |
|---|---|---|
| Chỉnh System Prompt nhắc lại chủ đề | Answer Relevance | Chạy lại `evaluate_answers.py` và đo mức tăng điểm Relevance. |
| Thêm LLM-as-a-Judge chấm ngữ nghĩa | Faithfulness & Completeness | So sánh tương quan giữa điểm LLM Judge và điểm chấm tay từ con người. |
| Quy tắc chấm riêng cho Adversarial | Overall Pass Rate | Kiểm tra các câu A01-A03 đảm bảo từ chối an toàn đạt 100% Pass. |

---

## 5. Regression Testing Strategy

**Câu 1: Khi nào chạy 
un_regression() trong production workflow?**
> Chạy tự động trong CI/CD Pipeline mỗi khi: (1) Thay đổi System Prompt; (2) Cập nhật mã nguồn Retriever/Chunking; (3) Đổi mô hình LLM chính; (4) Trước mỗi đợt Release.

**Câu 2: Threshold drop 0.05 có phù hợp OrbitTech Customer Support không? Vì sao?**
> Phù hợp. Mức giảm 0.05 (tương đương 5%) đủ nhạy để phát hiện lỗi giảm chất lượng mà không bị ảnh hưởng bởi biến động nhỏ ngẫu nhiên từ LLM.

**Câu 3: Metric/failure nào phải block deployment, metric nào chỉ alert?**
> - **Block Deployment**: Faithfulness < 0.80 (nguy cơ bịa thông tin) và các câu Adversarial từ chối thất bại (nguy cơ rò rỉ dữ liệu).
> - **Alert Only**: Answer Relevance hoặc Context Precision giảm nhẹ (gửi thông báo cho team dev kiểm tra).

**Câu 4: Điền evaluation stages vào flow.**

`	ext
Code/prompt/retrieval change → [ Unit Tests ] → [ Offline Benchmark (20 QA) ] → [ Regression Check (drop > 0.05) ] → Deploy
`

> *Giải thích:* Thay đổi code hoặc prompt cần qua Unit test, sau đó chạy tự động trên 20 câu Golden Dataset. Nếu chỉ số không giảm quá 0.05 so với bản cũ mới được Deploy.

---

## 6. Continuous Improvement Loop

`	ext
Evaluate → Analyze → Improve → Augment benchmark → Repeat
`

| Priority | Action | Metric dự kiến cải thiện | Expected impact |
|---:|---|---|---|
| 1 | Chỉnh Prompt hướng dẫn AI viết câu đầy đủ ý | Answer Relevance | Tăng Relevance trung bình từ 0.547 lên > 0.750 |
| 2 | Tích hợp LLM Judge chấm điểm theo ngữ nghĩa | Faithfulness & Overall Pass Rate | Tăng tỷ lệ Pass Rate từ 55% lên > 80% |
| 3 | Tối ưu Chunking và Reranking | Context Precision | Duy trì Context Precision > 0.950 cho tài liệu mới |

**Hai hoặc ba failure cases nào cần thêm vào benchmark ở vòng tiếp theo?**
> 1. Câu hỏi gộp 3 điều kiện chính sách cùng lúc (như vừa có mã giảm giá, vừa dùng thẻ quà tặng vừa hủy đơn).
> 2. Câu hỏi Adversarial đưa thông tin sai về ngày áp dụng chính sách v2.0 để kiểm tra khả năng xử lý thông tin gây nhiễu.

---

## 7. Final Reflection

**Điều gì trong kết quả benchmark trái với dự đoán ban đầu của bạn?**
> Điểm bất ngờ là BM25 lấy dữ liệu rất tốt (Recall 0.987, Precision 0.955), nhưng điểm tổng thể lại bị kéo xuống 55% Pass Rate do thuật toán đếm từ phạt các câu trả lời ngắn gọn của LLM.

**Word-overlap heuristics trong lab có giới hạn gì? Nếu đưa hệ thống vào production, bạn sẽ thay hoặc bổ sung metric nào?**
> Thuật toán đếm từ không nhận biết được ngữ nghĩa (semantic mismatch) — nó phạt từ đồng nghĩa đúng và chuộng câu từ dông dài. Khi đưa lên Production, tôi sẽ chuyển sang dùng **LLM-as-a-Judge (với RAGAS hoặc DeepEval)** kết hợp chấm mẫu ngẫu nhiên bằng người thật để đánh giá chất lượng thực tế.
