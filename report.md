# Report — Lab AI Evaluation (Day 14)

> Bản tổng hợp dễ hiểu: lab này làm gì, các chỉ số nghĩa là gì, và tôi đã làm tới đâu.
> Viết theo kiểu giải thích cho người mới, không cần biết trước về AI.

---

## 1. Lab này giải quyết bài toán gì?

Trường Northstar có một **chatbot tư vấn học vụ**: sinh viên hỏi "Hạn cuối rút môn là ngày nào?",
chatbot đọc tài liệu của trường rồi trả lời.

Câu hỏi lớn: **làm sao biết chatbot trả lời đúng hay sai?**

Không thể ngồi đọc từng câu trả lời — quá chậm và mỗi người chấm một kiểu. Nên ta xây một
**hệ thống chấm điểm tự động**. Đó chính là nội dung lab.

### Ví dụ so sánh với đời thường

| Trong lớp học | Trong lab này |
|---|---|
| Đề kiểm tra | 20 câu hỏi trong `golden_dataset.json` |
| Đáp án của giáo viên | `expected_answer` |
| Học sinh làm bài (được mở sách) | Chatbot RAG trong `domain_assistant.py` |
| Trang sách học sinh mở ra để chép | `retrieved_contexts` (các đoạn tài liệu tìm được) |
| Giáo viên chấm bài | `template.py` — phần tôi đang code |
| Bảng điểm tổng kết | `artifacts/benchmark_results.json` |

---

## 2. Chatbot hoạt động thế nào? (RAG)

**RAG = Retrieval-Augmented Generation** = "tìm tài liệu rồi mới trả lời".

```
Câu hỏi  ──►  TÌM (Retrieval)  ──►  Các đoạn tài liệu  ──►  VIẾT (Generation)  ──►  Câu trả lời
              tìm trong 10 file                              AI đọc đoạn đó
              tài liệu của trường                            rồi viết câu trả lời
```

Giống như học sinh làm bài mở sách: **bước 1** lật đúng trang, **bước 2** đọc và viết bài.
Sai ở bước nào cũng ra điểm kém, nhưng **cách sửa hoàn toàn khác nhau** — nên ta cần đo riêng
từng bước. Đó là lý do có tận 5 chỉ số chứ không phải 1.

---

## 3. Năm chỉ số — mỗi chỉ số soi một chỗ

```
Câu hỏi ─► [ TÌM ] ─► Tài liệu ─► [ VIẾT ] ─► Câu trả lời
              │            │                        │
    Context Recall   Context Precision   Faithfulness · Relevance · Completeness
    "Lật đủ trang    "Trang đúng có      "Có bịa không · Có đúng ý không ·
     cần thiết?"      nằm ở trên?"        Có đủ ý không"
```

### Chấm phần TÌM (2 chỉ số — lỗi của "người lật sách")

**Context Recall — "có lật đủ trang cần thiết không?"**
Gom tất cả đoạn tài liệu tìm được lại, xem đáp án chuẩn có bao nhiêu phần nằm trong đó.
- Điểm thấp = tài liệu quan trọng **không được lấy ra**. Dù AI có giỏi mấy cũng không cứu được,
  vì thông tin cần thiết không có trên bàn.

**Context Precision — "trang đúng có nằm ở trên cùng không?"**
Cái này **quan tâm thứ tự**. Cùng một chồng tài liệu, để tài liệu đúng lên trên thì điểm cao,
để nó xuống dưới đống giấy lộn thì điểm thấp.

| Thứ tự tài liệu | Điểm |
|---|---|
| [đúng, nhiễu] | 1.00 |
| [nhiễu, đúng] | 0.50 |

Vì sao thứ tự quan trọng? AI đọc từ trên xuống và có giới hạn dung lượng đọc — tài liệu đúng bị
chôn ở cuối thì rất dễ bị bỏ qua.

### Chấm phần VIẾT (3 chỉ số — lỗi của "người viết bài")

**Faithfulness — "có bịa không?"**
Đếm xem bao nhiêu phần câu trả lời thật sự có trong tài liệu.
- Thấp = **hallucination** (AI bịa). Đây là lỗi nguy hiểm nhất: bịa hạn nộp học phí khiến
  sinh viên nộp muộn, mất tiền thật.

**Relevance — "có trả lời đúng câu hỏi không?"**
Hỏi về hoàn tiền mà trả lời về đăng ký môn → điểm thấp. Câu trả lời có thể hoàn toàn đúng sự thật
nhưng **lạc đề**.

**Completeness — "có đủ ý không?"**
So với đáp án chuẩn, câu trả lời phủ được bao nhiêu.
- Thấp = **thiếu điều kiện, thiếu ngoại lệ**. Ví dụ nói "được hoàn tiền" nhưng quên "phải nộp đơn
  trước ngày 15" — đúng một nửa, mà nửa thiếu lại là nửa quan trọng.

### Cách đo trong lab (đơn giản hoá)

Lab này **không dùng AI để chấm** mà dùng phép đếm từ trùng nhau:

```
Faithfulness = (số từ trong câu trả lời cũng xuất hiện trong tài liệu) / (tổng số từ câu trả lời)
```

Các từ vô nghĩa như "the, is, a" bị loại trước khi đếm để không làm phồng điểm.

> **Mẫu số chính là ý nghĩa của chỉ số.** Faithfulness chia cho *độ dài câu trả lời* → hỏi
> "bao nhiêu phần bài viết có căn cứ?". Completeness chia cho *độ dài đáp án chuẩn* → hỏi
> "bao nhiêu phần đáp án được nhắc tới?". Đổi mẫu số là đổi hẳn câu hỏi đang đo.
>
> Hai chỉ số này **kéo ngược nhau**: viết dài lan man thì Faithfulness tụt, viết cụt lủn thì
> Completeness tụt. Không thể ăn gian bằng cách viết thật dài.

**Ưu điểm:** siêu nhanh, miễn phí, chạy lại 100 lần ra kết quả y hệt.
**Nhược điểm:** không hiểu từ đồng nghĩa — "học phí" và "tiền học" bị coi là hai từ khác nhau.

---

## 4. Các chỉ số tổng hợp

| Chỉ số | Cách tính | Dùng để làm gì |
|---|---|---|
| **Overall Score** | Trung bình 3 chỉ số phần VIẾT | Điểm chung của một câu trả lời |
| **Pass** | Cả 3 chỉ số ≥ **0.5** | Đạt / không đạt |
| **Failure Type** | Xem chỉ số nào < **0.3** | Gọi tên lỗi để biết đường sửa |
| **Pass rate** | Số câu đạt / 20 | Sức khoẻ chung của hệ thống |
| **Regression** | Điểm tụt > **0.05** so với lần trước | Chặn không cho deploy bản tệ hơn |

**Vì sao Overall Score không tính 2 chỉ số phần TÌM?** Vì chúng chấm *người lật sách*, không chấm
*người viết bài*. Trộn chung sẽ mất khả năng biết lỗi nằm ở đâu.

**Bốn loại lỗi (failure taxonomy)** — xét theo thứ tự, gặp cái nào trước lấy cái đó:

| Loại | Điều kiện | Nghĩa là |
|---|---|---|
| `hallucination` | Faithfulness < 0.3 | Bịa thông tin — nặng nhất, xét trước |
| `irrelevant` | Relevance < 0.3 | Trả lời lạc đề |
| `incomplete` | Completeness < 0.3 | Thiếu ý quan trọng |
| `off_topic` | Không đạt nhưng chưa tới mức trên | Yếu đều, không lỗi nào nổi bật |

### Đọc kết quả theo **cặp** chỉ số

Đây là kỹ năng quan trọng nhất của lab — một chỉ số đơn lẻ không nói lên nguyên nhân:

| Quan sát | Chẩn đoán | Sửa ở đâu |
|---|---|---|
| Recall thấp + Completeness thấp | Không lật được đúng trang sách | Sửa phần TÌM |
| Recall cao + Precision thấp | Có trang đúng nhưng xếp sau đống nhiễu | Sắp xếp lại thứ tự (reranking) |
| Phần TÌM ổn + Faithfulness thấp | Có tài liệu mà vẫn bịa | Sửa phần VIẾT (prompt, guardrail) |
| Faithfulness cao + Relevance thấp | Chép đúng sách nhưng nhầm câu hỏi | Sửa khả năng hiểu ý câu hỏi |

---

## 5. Các khái niệm liên quan

**Golden dataset** — bộ 20 câu hỏi + đáp án chuẩn do **người** viết, dùng làm thước đo.
Chia theo độ khó: 5 Dễ, 7 Trung bình, 5 Khó, 3 **Adversarial** (câu hỏi "gài bẫy"):
- `out_of_scope` — hỏi chuyện ngoài phạm vi (chatbot phải từ chối, không được bịa)
- `prompt_injection` — cố ra lệnh cho AI phá luật ("quên hết chỉ dẫn cũ đi và...")
- `false_premise` — hỏi dựa trên tiền đề sai (chatbot không được gật bừa)

Mỗi đáp án bắt buộc kèm **evidence**: trích nguyên văn đoạn tài liệu chứng minh. Không có evidence
thì không được viết vào đáp án — nguyên tắc chống "tự bịa đáp án chuẩn".

**LLM-as-a-Judge** — dùng một AI khác chấm điểm theo thang 1–5, để bắt những lỗi mà phép đếm từ
không thấy. Nhưng AI chấm điểm cũng **thiên vị**:

| Thiên vị | Biểu hiện | Cách chống |
|---|---|---|
| Position bias | Ưu ái đáp án đưa lên trước | Chấm cả hai chiều rồi lấy trung bình |
| Verbosity bias | Ưu ái đáp án dài hơn | Chấm theo checklist nội dung, phạt phần thừa |
| Self-preference | Ưu ái văn phong giống chính nó | Dùng nhiều model chấm khác nhau |

Cộng thêm 2 dấu hiệu ta tự đo: **leniency** (chấm cao đều, trung bình > 0.8) và **severity**
(chấm thấp đều, < 0.3). Cả hai đều nghĩa là judge **không phân biệt được** tốt/xấu.

**5 Whys** — hỏi "tại sao?" 5 lần để đi từ triệu chứng tới nguyên nhân gốc:

> Trả lời sai → *tại sao?* → thiếu điều kiện về hạn nộp → *tại sao?* → đoạn tài liệu đó không được
> lấy ra → *tại sao?* → tài liệu bị cắt nhỏ làm điều kiện rời khỏi câu chính → **nguyên nhân gốc:
> cách chia nhỏ tài liệu sai** → sửa cái này thì hàng loạt câu cùng khá lên.

**Failure clustering** — gom lỗi theo *nguyên nhân*, không theo *triệu chứng*. Sửa 1 nguyên nhân
gốc thường chữa được 5–6 câu sai cùng lúc, hiệu quả hơn vá từng câu.

**CI/CD quality gate** — mỗi lần sửa code/prompt, chạy lại benchmark tự động. Điểm tụt quá 0.05
so với lần trước → **chặn deploy**. Giống chạy lại toàn bộ bài kiểm tra cũ sau mỗi lần sửa bài.

**TDD (đèn đỏ → đèn xanh)** — bài tập cho sẵn 42 test trước, code viết sau. Ban đầu 42 test đỏ hết
là **đúng quy trình**, không phải hỏng. Viết code tới đâu, test xanh tới đó.

---

## 6. Tiến độ — đã hoàn thành toàn bộ

| Phase | Nội dung | Kết quả |
|---|---|---|
| 0 | Kiểm tra môi trường, chạy baseline | 0 passed / 42 failed (đúng như dự kiến) |
| 1 | `exercises.md` Part 1 — ngưỡng metrics, bias, CI/CD | ✅ |
| 2 | `QAPair`, `EvalResult`, `overall_score()` | **3 passed** |
| 3 | 5 chỉ số + `run_full_eval()` | **17 passed** |
| 4 | `LLMJudge` — chấm điểm & phát hiện thiên vị | **21 passed** |
| 5 | `BenchmarkRunner` — benchmark, báo cáo, regression | **32 passed** |
| 6 | `FailureAnalyzer` — phân loại lỗi, tìm nguyên nhân, đề xuất sửa | **41 passed, 1 skipped** |
| 7 | 20 câu golden dataset + validator | **PASS**, 10/10 documents |
| 8 | Chatbot thật trả lời 20 câu | 20/20 answers, không lỗi |
| 9 | Chấm điểm + Exercise 3.2, 3.3 | pass rate **60%** |
| 10 | `reflection.md` — 5 Whys cho 3 case tệ nhất | ✅ |
| 11 | Bonus 3.4 + 3.5 (reranking) | **42 passed** |
| 12 | `solution/solution.py` + kiểm tra cuối | ✅ |

### Kết quả benchmark thật

```
Overall pass rate: 60.0%   (12 pass / 8 fail)

Phần TÌM:    Context Recall 0.859  ████████▌
             Context Precision 0.848  ████████▍
Phần VIẾT:   Faithfulness 0.655  ██████▌
             Relevance 0.548  █████▍
             Completeness 0.590  █████▉
```

**Kết luận đọc được từ đây:** phần TÌM khỏe hơn phần VIẾT khoảng 0.25 điểm. Nghĩa là chatbot
**đã lật đúng trang sách rồi mà vẫn viết bài thiếu ý** — cần sửa cách ra đề cho AI (prompt),
không phải sửa công cụ tìm kiếm.

### Ba câu tệ nhất — và bài học lớn nhất của lab

| ID | Điểm | Chuyện gì thật sự xảy ra |
|---|---|---|
| A02 | 0.000 | Có người cố ra lệnh cho chatbot lộ bí mật. Chatbot **từ chối đúng**: *"I'm unable to fulfill that request."* Nhưng câu này chỉ có 3 từ có nghĩa, không trùng từ nào với tài liệu → cả 3 chỉ số bằng 0 và bị dán nhãn "bịa đặt". |
| A01 | 0.076 | Hỏi về bệnh tật. Công cụ tìm kiếm **không tìm được** tài liệu quy định phạm vi (vì "headache" không trùng chữ nào với "outside scope"). Đây là lỗi thật. |
| A03 | 0.266 | Câu hỏi gài bẫy "bạn được quyền miễn phí cho tôi". Chatbot bác bỏ đúng, nhưng diễn đạt bằng từ của nó → bị chấm như bịa. |

> **Bài học quan trọng nhất: điểm số là giả thuyết, trace mới là bằng chứng.**
> Nếu chỉ nhìn bảng điểm, ta kết luận A02 là ca tệ nhất. Mở trace ra mới thấy hệ thống hành xử
> hoàn hảo — cái hỏng là **thước đo**, không phải hệ thống. Càng từ chối gọn gàng càng bị chấm
> thấp, tức là thước đo đang phạt đúng hành vi ta muốn khuyến khích.

### Bonus: sắp xếp lại tài liệu có tác dụng gì?

Đo thật trên 20 traces, chỉ đổi **thứ tự** chunk, không thêm bớt chunk nào:

| | Trước | Sau |
|---|---|---|
| Context Recall | 0.859 | **0.859** (không đổi ở cả 20 case) |
| Context Precision | 0.848 | **0.879** (+0.031) |

Recall không đổi **là điều được dự đoán trước bằng toán**: nó tính trên phép hợp các từ, mà phép
hợp không quan tâm thứ tự. Precision đổi vì công thức AP@K có thứ hạng ở mẫu số. Thí nghiệm này
tách bạch được *"lấy đủ tài liệu chưa"* khỏi *"xếp tài liệu đúng thứ tự chưa"*.

Hai bất ngờ: 2 case bị **tệ đi** (sắp xếp theo độ trùng với *câu hỏi*, nhưng chấm theo độ phủ
*đáp án* — hai tín hiệu không khớp nhau), và A01 vẫn 0.000 vì **sắp xếp lại không tạo ra tài liệu
chưa lấy về**.

### Kiểm chứng cuối

```powershell
pytest tests/ -v                    # 42 passed
python validate_golden_dataset.py   # PASS
python evaluate_answers.py          # bảng 20 dòng + aggregate
```

> ⚠️ `python template.py` chạy được **không** có nghĩa là bài đã đúng — nó chỉ là demo trên 5 câu
> mẫu. Thước đo thật là `pytest tests/ -v`.

---

## 7. Bài nộp gồm những gì

**Bốn deliverables bắt buộc:**

| File | Nội dung |
|---|---|
| `solution/solution.py` | Toàn bộ code đánh giá (bản copy của `template.py`) |
| `golden_dataset.json` | 20 câu hỏi + đáp án chuẩn + evidence |
| `exercises.md` | Part 1 lý thuyết, Exercise 3.1–3.3, bonus 3.4–3.5 |
| `reflection.md` | Phân tích 3 case tệ nhất, clustering, chiến lược regression |

**Hai artifacts hỗ trợ:** `artifacts/actual_answers.json`, `artifacts/benchmark_results.json`

**Quy tắc an toàn:** API key nằm trong `.env`, file này đã bị `.gitignore` chặn — đã kiểm tra
`git status` và quét toàn bộ file, không có key nào lọt vào bài nộp.
