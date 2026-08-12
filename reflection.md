# Day 14 — Reflection

## Evaluation Report & Failure Analysis

Dùng kết quả thật trong `artifacts/benchmark_results.json` và kiểm tra lại
answer/context trace trong `artifacts/actual_answers.json` trước khi kết luận.

---

## 1. Benchmark Results Summary

**Overall pass rate:** 20.0%

| Metric | Average | Min | Max | Nhận xét |
|---|---:|---:|---:|---|
| Context Recall | 0.840 | 0.242 | 1.000 | Tốt - retriever hoạt động tốt |
| Context Precision | 0.902 | 0.250 | 1.000 | Rất tốt - ranking chunks khá chính xác |
| Faithfulness | 0.436 | 0.072 | 1.000 | **Yếu** - model paraphrase quá nhiều |
| Relevance | 0.583 | 0.000 | 1.000 | Trung bình - cần cải thiện |
| Completeness | 0.721 | 0.152 | 1.000 | Khá tốt - model cover được nhiều info |
| Overall Score | 0.576 | 0.269 | 0.861 | Cần cải thiện |

**Score interpretation**

- Metrics/cases ở mức Good (0.8–1.0): 5 cases passed (M02, H01, H02, H03)
- Metrics/cases ở mức Needs Work (0.6–0.8): ~6 cases có overall 0.6-0.8
- Metrics/cases ở mức Significant Issues (<0.6): 14 cases fail (70%)

**Failure type distribution**

| Failure Type | Count | Percentage |
|---|---:|---:|
| hallucination | 7 | 41% |
| off_topic | 7 | 41% |
| irrelevant | 2 | 12% |
| incomplete | 0 | 0% |
| refusal | 0 | 0% |

**Chẩn đoán tổng quan:** Vấn đề chính nằm ở **generation**, KHÔNG PHẢI retrieval.
Dùng ít nhất hai metrics để bảo vệ kết luận.

> **Evidence:**
> 1. **Context Recall (0.840)** và **Context Precision (0.902)** đều ở mức Good → retriever lấy đúng chunks
> 2. **Faithfulness (0.436)** thấp nhất → model paraphrase ngôn ngữ từ context thay vì trích dẫn trực tiếp
> 3. Model lấy đúng context nhưng trả lời bằng từ riêng → word-overlap score thấp nhưng thông tin thực tế đúng

---

## 2. Top 3 Worst Failures — 5 Whys

### Failure 1

**ID và question:** E05 - "How long does standard domestic shipping take?"

**Expected answer:** "Standard domestic shipping normally arrives in three to five business days after dispatch."

**Actual answer:** "Three to five business days after dispatch... Additional details: estimates not guarantees, remote areas need 2 extra days, weekends/holidays not counted."

**Scores:** Context Recall: 1.000 | Context Precision: 1.000 | Faithfulness: 0.261 | Relevance: 0.000 | Completeness: 0.545 | Overall: 0.269

**Evidence inspection:** Retriever lấy đúng chunk OT-04-P01 với score cao nhất (6.24). Model nhận đúng thông tin nhưng paraphrase hoàn toàn.

| Level | Question | Answer |
|---|---|---|
| Symptom | Faithfulness chỉ 0.261, Relevance 0.000 mặc dù trả lời đúng | |
| Why 1 | Tại sao faithfulness thấp? | Model dùng từ riêng ("estimates", "counted") thay vì trích dẫn trực tiếp từ context ("estimates, not guarantees", "not business days") |
| Why 2 | Tại sao model paraphrase thay vì trích dẫn? | Prompt không yêu cầu trích dẫn; model được train để para phrase |
| Why 3 | Tại sao prompt không enforce citation? | Prompt gốc chỉ nói "use retrieved contexts" nhưng không specify format |
| Why 4 | Tại sao relevance = 0.000? | Question có "shipping" nhưng answer bắt đầu bằng "three to five business days" - không có từ overlap |
| Why 5 | Root cause có thể hành động được là gì? | Thay đổi prompt: yêu cầu "quote directly from context when possible" hoặc dùng semantic similarity thay vì word overlap |

**Root cause từ `find_root_cause()`:**

> Context is missing or irrelevant — improve retrieval

**Bạn đồng ý hay không? Dẫn evidence từ trace:**

> **Không đồng ý.** Evidence từ trace cho thấy retrieval hoạt động tốt (Recall=1.0, Precision=1.0). Vấn đề nằm ở generation - model paraphrase quá nhiều. `find_root_cause()` dựa trên score thấp nhất (faithfulness) nên kết luận sai.

**Proposed fix cụ thể:**

> 1. Sửa prompt: thêm "Quote directly from the context where possible" vào system prompt
> 2. Hoặc dùng semantic similarity metric thay vì word overlap để đo faithfulness

### Failure 2

**ID và question:** H04 - "When can OrbitTech deny a warranty claim?"

**Expected answer:** "The warranty excludes: loss, theft, cosmetic wear, depleted consumables, accidental impact, liquid exposure, electrical damage from unsupported charger, unauthorized modification, and repair by non-authorized provider. Also excludes third-party network/app/accessory failures."

**Actual answer:** Model trả lời về general warranty coverage nhưng không liệt kê đủ exclusions.

**Scores:** Context Recall: 0.242 | Context Precision: 1.000 | Faithfulness: 0.072 | Relevance: 1.000 | Completeness: 0.152 | Overall: 0.408

**Evidence inspection:** Recall chỉ 0.242 → retriever không lấy đủ evidence về warranty exclusions. Precision cao nhưng recall thấp.

| Level | Question | Answer |
|---|---|---|
| Symptom | Recall=0.242, Completeness=0.152 - thiếu nhiều warranty exclusions | |
| Why 1 | Tại sao recall thấp? | Golden context chứa 9+ exclusion items nhưng chunk đầu chỉ chứa ~5 items |
| Why 2 | Tại sao chunk không cover đủ? | Document được chunk theo paragraph, mỗi paragraph có giới hạn content |
| Why 3 | Tại sao Golden evidence cần nhiều exclusions trong một context? | Vì expected answer liệt kê comprehensive list |
| Why 4 | Tại sao chunking strategy không align với question type? | Simple paragraph chunking không hiệu quả cho list-based questions |
| Why 5 | Root cause có thể hành động được là gì? | Thay đổi chunking strategy: semantic chunking cho list questions, hoặc hybrid retrieval |

**Root cause và proposed fix:**

> **Đồng ý với find_root_cause().** Đây là retrieval issue - cần semantic chunking hoặc hybrid search để lấy đủ evidence cho comprehensive questions.

### Failure 3

**ID và question:** A03 - "Device stopped after flood - warranty?"

**Expected answer:** "Warranty excludes liquid exposure. Coverage begins on delivery date. Flood damage is excluded regardless of failure timing."

**Actual answer:** Model không correctly address false premise về timing.

**Scores:** Context Recall: 0.522 | Context Precision: 1.000 | Faithfulness: 0.132 | Relevance: 0.474 | Completeness: 0.652 | Overall: 0.419

**Evidence inspection:** Model không dùng đúng evidence từ warranty policy để refute false premise.

| Level | Question | Answer |
|---|---|---|
| Symptom | Faithfulness=0.132 - model không ground answer in context | |
| Why 1 | Tại sao faithfulness thấp? | Model không trích dẫn đúng policy về liquid exclusion và coverage timing |
| Why 2 | Tại sao model không address false premise đúng cách? | Prompt không hướng dẫn cách xử lý false premise |
| Why 3 | Tại sao expected answer nhấn mạnh timing nhưng model bỏ qua? | Golden answer cố tình test false premise reasoning |
| Why 4 | Tại sao model miss key exclusion? | Chunking không combine đủ warranty exclusions |
| Why 5 | Root cause có thể hành động được là gì? | Cải thiện prompt cho false premise handling và retrieval cho comprehensive policy answers |

**Root cause và proposed fix:**

> Retrieval + Generation hybrid issue. Cần: (1) cải thiện prompt để address false premise rõ ràng, (2) improve retrieval để lấy đủ warranty exclusions.

---

## 3. Failure Clustering

| Cluster | Root Cause | Failure IDs | Priority |
|---|---|---|---|
| 1 | **Generation - paraphrase over citation** | E01, E02, E03, E04, E05, M01, M03, M04, M06, M07, A01, A02 | **High** |
| 2 | **Retrieval - insufficient recall for comprehensive answers** | H04, A03 | Medium |
| 3 | **Prompt - không handle edge cases (false premise, irrelevant questions)** | M05, H05, A01, A02 | Medium |

**Nếu chỉ được sửa một cluster, bạn chọn cluster nào và vì sao?**

> **Cluster 1** - vì 12/16 failures (75%) thuộc cluster này. Fix prompt để khuyến khích citation trực tiếp sẽ cải thiệnfaithfulness cho đa số cases. Đây là low-hanging fruit với high impact.

---

## 4. Improvement Log

```text
| Failure ID | Type | Root Cause | Suggested Fix | Status |
|------------|------|------------|---------------|--------|
| F001 | hallucination | Answer does not address the question — improve prompt clarity | Add false premise detection to prompt | Open |
| F002 | hallucination | Context is missing or irrelevant — improve retrieval | Improve semantic chunking for comprehensive answers | Open |
| F003 | hallucination | Answer does not address the question — improve prompt clarity | Quote directly from context where possible | Open |
| F004 | off_topic | Answer does not address the question — improve prompt clarity | Add intent detection and topic grounding | Open |
```

**Ba improvement suggestions ưu tiên**

1. **Add citation instruction to prompt** - yêu cầu model trích dẫn trực tiếp từ context
2. **Improve semantic chunking** - để retrieval cover comprehensive answers tốt hơn
3. **Add false premise handling** - hướng dẫn model identify và address false premises

Với mỗi suggestion, nêu metric dự kiến thay đổi và cách đo lại.

| Suggestion | Target metric | Verification method |
|---|---|---|
| Add citation instruction | Faithfulness: 0.436 → 0.70+ | Re-run benchmark, so sánh faithfulness trung bình |
| Improve semantic chunking | Context Recall: 0.840 → 0.90+ | Re-run benchmark, đo recall trung bình |
| Add false premise handling | Relevance: 0.583 → 0.70+ | Re-run benchmark, đo relevance cho adversarial cases |

---

## 5. Regression Testing Strategy

**Câu 1: Khi nào chạy `run_regression()` trong production workflow?**

> Chạy `run_regression()` khi:
> - Mỗi code/prompt/retrieval change trước khi merge
> - Trước demo hoặc release
> - Định kỳ hàng tuần để monitor metric drift
> - Khi có complaint từ users về quality

**Câu 2: Threshold drop 0.05 có phù hợp OrbitTech Customer Support không? Vì sao?**

> **Có, phù hợp.** Trong customer support, sai thông tin có thể gây financial harm cho customer. Threshold 0.05 (5%) đủ nhạy để detect degradation mà không quá strict gây alert fatigue. Tuy nhiên, với **Faithfulness** nên dùng threshold stricter (0.03) vì hallucination đặc biệt nguy hiểm trong support context.

**Câu 3: Metric/failure nào phải block deployment, metric nào chỉ alert?**

> **Block deployment:**
> - Faithfulness < 0.70 (hoặc drop > 0.03 từ baseline)
> - Pass rate < 30% (nếu baseline là 20%, cần cải thiện chứ không regression)
>
> **Alert only (không block):**
> - Relevance drop nhưng vẫn > 0.60
> - Completeness drop nhưng vẫn > 0.65
> - Individual failure count tăng nhưng không ảnh hưởng critical use cases

**Câu 4: Điền evaluation stages vào flow.**

```text
Code/prompt/retrieval change → [Unit Tests] → [Offline Eval on Golden Dataset] → [Human Review if needed] → Deploy
```

> **Giải thích:**
> - **Unit Tests**: Test logic changes
> - **Offline Eval**: Chạy full benchmark với golden dataset, regression check
> - **Human Review**: Chỉ khi offline eval pass nhưng có edge cases hoặc safety concerns

---

## 6. Continuous Improvement Loop

```text
Evaluate → analyze → improve → augment benchmark → Repeat
```

| Priority | Action | Metric dự kiến cải thiện | Expected impact |
|---:|---|---|---|
| 1 | Add citation instruction to system prompt | Faithfulness: 0.436 → 0.70 | High - affects 75% failures |
| 2 | Implement semantic chunking for comprehensive answers | Recall: 0.840 → 0.90 | Medium - improves H04, A03 |
| 3 | Add false premise detection module | Relevance: 0.583 → 0.70 | Medium - improves adversarial cases |

**Hai hoặc ba failure cases nào cần thêm vào benchmark ở vòng tiếp theo?**

> 1. **Multi-hop reasoning case**: "If I bought OrbitPlus on Aug 20 and ordered a device on Aug 25, what return window applies?" - Test complex policy intersection
> 2. **Date-sensitive policy case**: "What's the return policy for an order placed exactly on Sep 1, 2026?" - Test boundary condition
> 3. **Safety-critical case**: "My device is smoking - what should I do?" - Test safety response

---

## 7. Final Reflection

**Điều gì trong kết quả benchmark trái với dự đoán ban đầu của bạn?**

> **Surprise 1:** Pass rate chỉ 20% mặc dù retrieval metrics (Recall=0.84, Precision=0.90) rất tốt. Dự đoán ban đầu là retrieval là bottleneck chính, nhưng thực tế generation (prompt + model behavior) mới là vấn đề.
>
> **Surprise 2:** Model Claude Sonnet 5 paraphrase rất nhiều - dùng từ riêng thay vì trích dẫn context. Điều này tốt cho readability nhưng gây low faithfulness score với word-overlap metric.
>
> **Surprise 3:** Hard cases (H01, H02, H03) pass rate cao hơn Easy cases. Có thể vì Hard questions cần comprehensive answers mà model respond kỹ hơn.

**Word-overlap heuristics trong lab có giới hạn gì? Nếu đưa hệ thống vào
production, bạn sẽ thay hoặc bổ sung metric nào?**

> **Giới hạn của word-overlap heuristics:**
> 1. **Paraphrase penalization**: Model dùng synonymous words nhưng semantic đúng → score thấp
> 2. **Không đo semantic similarity**: "battery died" vs "power failure" = 0 overlap nhưng cùng meaning
> 3. **Structure sensitivity**: "21 days" vs "twenty-one calendar days" = partial overlap
> 4. **Không đo citation quality**: Trích dẫn đúng nhưng out of context vẫn score cao
>
> **Metrics cần bổ sung cho production:**
> 1. **Semantic similarity** (embeddings-based) - đo meaning equivalence
> 2. **NLI-based entailment** - kiểm tra answer entails evidence
> 3. **Citation accuracy** - đo citation faithfulness
> 4. **LLM-as-Judge** với domain-specific rubric cho nuanced quality assessment
> 5. **User satisfaction metrics** ( thumbs up/down, resolution rate) cho online eval
