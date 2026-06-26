# Case 1 - Support Ticket Triage

## Mục tiêu

Case này giúp học viên luyện 4 câu hỏi nên bật ra ngay khi gặp một AI task:

- Cái gì deterministic và nên chấm bằng code?
- Cái gì cần semantic judgment và nên giao cho LLM judge hoặc human?
- Cái gì high-risk nên cần gate chặt hơn?
- Sai ở đâu thì cần escalation sang người thật?

Chỉ cần thiết kế eval ban đầu, không cần code full system.

Case này nối trực tiếp từ track **AI Customer Support Agent** ở Day 18/19, nhưng đổi góc nhìn từ **thiết kế trải nghiệm** sang **thiết kế eval**.

---

## 1. Bối cảnh

Một công ty SaaS B2B dùng AI để đọc ticket support mới và tạo output triage cho hệ thống nội bộ.

Output này không gửi trực tiếp cho khách hàng, nhưng nó được dùng để:

- phân loại ticket,
- đánh dấu mức độ gấp,
- route đến đúng team,
- quyết định có cần người thật nhảy vào hay không.

Nếu AI route sai, ticket có thể bị trễ, bỏ sót escalation, hoặc đẩy sai sang team không xử lý được.

---

## 2. Workflow logic (ASCII)

```text
Khách hàng gửi ticket hỗ trợ
    ↓
AI đọc:
- tiêu đề
- nội dung ticket
- loại khách hàng
    ↓
Hệ thống phải quyết định:
- đây là loại vấn đề gì?
- mức độ khẩn cấp ra sao?
- có cần người thật xử lý ngay không?
- ticket nên vào hàng của team nào?
    ↓
UI inbox nội bộ hiển thị:
- nhãn loại yêu cầu
- mức độ khẩn
- team phụ trách
- cờ "cần xử lý ngay"
- lý do tóm tắt
    ↓
Nếu khách doanh nghiệp + có dấu hiệu chặn công việc
    ↓
Đẩy lên hàng ưu tiên cao / escalation
```

---

## 3. UI hiển thị dự kiến (ASCII)

```text
+----------------------------------------------------------------+
| Hộp thư hỗ trợ nội bộ                                           |
+----------------------------------------------------------------+
| Ticket: T-002                                                   |
| Khách hàng: Công ty ABC (Enterprise)                            |
| Tiêu đề: Thanh toán lỗi, tài khoản bị khóa                      |
|----------------------------------------------------------------|
| AI gợi ý                                                        |
| - Loại yêu cầu: [ ? ]                                           |
| - Mức độ khẩn: [ ? ]                                            |
| - Team phụ trách: [ ? ]                                         |
| - Cần người xử lý ngay: [ ? ]                                   |
| - Lý do tóm tắt: [ .......................................... ] |
|----------------------------------------------------------------|
| Hàng đợi hiện tại: [ Bình thường ] hoặc [ Ưu tiên cao ]         |
+----------------------------------------------------------------+
```

Học viên cần tự đề xuất output contract tối thiểu phía sau để màn hình này hiển thị được.

---

## 4. Input mẫu

```json
{
  "ticket_id": "T-001",
  "subject": "Cannot login after password reset",
  "message": "I reset my password twice but still cannot log in. This is blocking my work.",
  "customer_tier": "enterprise"
}
```

Một input khác:

```json
{
  "ticket_id": "T-002",
  "subject": "URGENT: payment failed and account disabled",
  "message": "Our team is locked out because your billing system failed. Fix this now.",
  "customer_tier": "enterprise"
}
```

---

## 5. Business rules / operational rules

- Output phải đúng schema và đúng allowed enums.
- `confidence` phải nằm trong khoảng `0-1`.
- Nếu `customer_tier = enterprise` và `urgency` là `high` hoặc `critical`, `requires_human` phải bằng `true`.
- Ticket billing không được route sang `product_team`.
- Ticket có dấu hiệu “blocking work”, “locked out”, hoặc “account disabled” không nên bị đánh `low`.
- `reason_codes` phải phản ánh được nội dung ticket, không được bốc thêm sự thật không có trong input.

---

## 6. Ví dụ full luồng để hình dung nhanh

### Tình huống

Khách hàng doanh nghiệp nhắn vào kênh hỗ trợ:

```text
Chị ơi bên em reset mật khẩu 2 lần rồi mà tài khoản admin vẫn không vào được.
Bên em đang bị chặn công việc từ sáng.
```

### Data mẫu

- `customer_tier`: `enterprise`
- `account_name`: `Công ty Minh Phát Logistics`
- `previous_tickets_7d`: `0`
- `channel`: `Zalo OA`

### Workflow ASCII

```text
Khách nhắn vấn đề đăng nhập
    ↓
AI đọc nội dung + loại khách hàng
    ↓
AI phát hiện tín hiệu:
- login issue
- blocked work
- enterprise customer
    ↓
Hệ thống gợi ý:
- category = technical
- urgency = high hoặc critical
- requires_human = true
- route_to = technical_support
    ↓
UI nội bộ đẩy ticket lên hàng ưu tiên
    ↓
Nhân viên hỗ trợ xem lại rồi tiếp nhận
```

### UI trước khi AI xử lý (ASCII)

```text
+----------------------------------------------------------------+
| Hộp thư hỗ trợ nội bộ                                           |
+----------------------------------------------------------------+
| Ticket: T-115                                                   |
| Kênh: Zalo OA                                                   |
| Khách hàng: Minh Phát Logistics                                 |
|----------------------------------------------------------------|
| Nội dung khách nhắn:                                            |
| "Reset mật khẩu 2 lần rồi mà tài khoản admin vẫn không vào..."  |
|----------------------------------------------------------------|
| AI gợi ý: Chưa có                                               |
+----------------------------------------------------------------+
```

### UI sau khi AI xử lý (ASCII)

```text
+----------------------------------------------------------------+
| Hộp thư hỗ trợ nội bộ                                           |
+----------------------------------------------------------------+
| Ticket: T-115                                                   |
| Khách hàng: Minh Phát Logistics (Enterprise)                    |
|----------------------------------------------------------------|
| AI gợi ý                                                        |
| - Loại yêu cầu: Technical                                       |
| - Mức độ khẩn: High                                             |
| - Team phụ trách: Technical Support                             |
| - Cần người xử lý ngay: Có                                      |
| - Lý do tóm tắt: Lỗi đăng nhập đang chặn công việc              |
| - Hàng đợi: Ưu tiên cao                                         |
+----------------------------------------------------------------+
```

Ví dụ này giúp người đọc hình dung ngay:

- AI đang quyết định gì,
- quyết định nào hiển thị ra UI,
- và sai ở đâu thì ảnh hưởng vận hành.

---

## 7. Seed cases

Đây không phải full dataset. Đây chỉ là 3 seed cases để học viên hình dung phạm vi và failure modes.

### Seed A - Happy path

- `subject`: `Cannot login after password reset`
- Kỳ vọng: `category = technical`, `requires_human = true` nếu urgency đủ cao, route về `technical_support`

### Seed B - Ambiguous / low-info

- `subject`: `Help`
- `message`: `Please help asap`
- Kỳ vọng: AI không nên tự tin gán category quá mạnh; cần `unknown` hoặc route theo hướng cần review

### Seed C - High-risk / escalation

- `subject`: `URGENT: payment failed and account disabled`
- Kỳ vọng: `category = billing`, `urgency = critical`, `requires_human = true`, route về `billing_ops` hoặc `human_escalation`

---

## 8. Bạn phải đề xuất thêm 5 Dataset Edge Cases

Sau khi đọc seed cases ở trên, hãy đề xuất thêm 5 case cần đưa vào reference dataset version đầu.

Không cần nộp một bảng coverage riêng. Hãy chọn 5 case đại diện cho các lát cắt khác nhau, ví dụ: match rõ, thiếu tín hiệu, ambiguity, escalation, và regression.

1. Happy path: Khách hàng yêu cầu hỗ trợ đổi mật khẩu thông thường.
   - case này dùng để bắt failure gì? Đảm bảo AI phân loại đúng nhóm `technical` và định tuyến chính xác đến `support_l1` khi thông tin rõ ràng và không khẩn cấp.
2. Ambiguous input: "Tôi cần xem lại gói cước đang dùng và hỏi thêm về lỗi đăng nhập."
   - case này dùng để bắt failure gì? Bắt lỗi AI bị bối rối giữa `billing` và `technical`; đảm bảo AI biết gán nhãn trung lập hoặc cảnh báo độ tự tin thấp nếu có sự tranh chấp nhãn rõ ràng.
3. Missing information: "Alo, xem hộ mình cái này với."
   - case này dùng để bắt failure gì? Đảm bảo AI không tự bịa ra category/urgency (hallucination) mà gán `unknown` hoặc đề xuất định tuyến cần làm rõ thông tin.
4. High-risk / escalation: Khách hàng Enterprise nhắn: "Hệ thống thanh toán bị lỗi, toàn bộ nhân viên công ty tôi bị khóa tài khoản."
   - case này dùng để bắt failure gì? Kiểm tra xem AI có tuân thủ quy tắc an toàn bắt buộc: tự động đẩy `urgency = critical` và kích hoạt `requires_human = true` cho khách hàng Enterprise.
5. Regression case: Khách hàng nhắn: "Tôi không muốn hủy dịch vụ nữa, tôi đã gia hạn thành công rồi."
   - case này dùng để bắt failure gì? Đảm bảo AI không bị nhầm lẫn từ khóa "hủy dịch vụ" để phân vào nhóm hủy/khiếu nại, mà phải hiểu ngữ nghĩa là "đã gia hạn thành công".

---

## 9. Mock outcome để soi

Giả sử trên UI nội bộ, hệ thống hiển thị kết quả gợi ý như sau cho `T-002`:

```text
+----------------------------------------------------------------+
| Ticket: T-002                                                   |
| Khách hàng: Enterprise                                          |
|----------------------------------------------------------------|
| AI gợi ý                                                        |
| - Loại yêu cầu: Product question                                |
| - Mức độ khẩn: Medium                                           |
| - Team phụ trách: Support L1                                    |
| - Cần người xử lý ngay: Không                                   |
| - Lý do tóm tắt: Có vấn đề thanh toán                           |
| - Độ tin cậy: 0.91                                              |
+----------------------------------------------------------------+
```

Kết quả này trông có vẻ “ổn” nếu chỉ nhìn bề mặt, nhưng khả năng cao là sai về judgment vận hành.

---

## 10. Nhiệm vụ học viên

Hãy điền workbook bên dưới cho case này.

Không cần:

- viết eval runner,
- viết prompt judge thật,
- làm lại `User Input Grid` đầy đủ như bài test inputs hôm trước,
- tạo full dataset lớn,
- code full system.

Cần làm:

- chọn đúng nguồn chấm cho từng thành phần,
- viết các rule kiểm tra đủ cụ thể để có thể implement sau,
- đặt release gate có ý nghĩa vận hành,
- đề xuất 5 edge cases cần đưa vào reference dataset,
- và lập một pilot plan có thời gian + chi phí sơ bộ.

---

## 11. Bạn nên làm gì ở case 1?

Đây là case scaffold cao, nên cách làm tốt nhất là:

1. Đọc ví dụ full luồng trước để hiểu “một output tốt trông như thế nào”.
2. So mock outcome với ví dụ full luồng để thấy lỗi đang nằm ở đâu.
3. Nhìn từ UI để suy ra các field tối thiểu hệ thống phải có.
4. Điền `Eval Decision Map` trước, rồi mới quay lại viết các kiểm tra tự động và gate.

Case này thường **không bắt buộc phải có domain expert chuyên sâu**. Nếu chọn không cần expert, bạn vẫn phải giải thích vì sao human review vận hành là đủ.

---

## 12. Workbook

Lưu ý chung cho toàn bộ câu trả lời:

- Không chỉ điền đáp án ngắn.
- Với mỗi phần, hãy nêu cả **quyết định** và **lý do**.
- Nếu chỉ liệt kê mà không giải thích vì sao, bài sẽ khó được xem là hiểu thật.

### 1. Unit of Work

- AI đang thực hiện công việc gì?
- Output cuối cùng được dùng bởi ai?
- Nếu sai, hậu quả vận hành là gì?

Gợi ý từ bài hôm trước:

- Đừng chọn “toàn bộ hệ thống hỗ trợ khách hàng”.
- Ở case này, một `Unit of Work` tốt thường là: **một ticket đi vào -> AI gán nhãn, đánh mức ưu tiên, đề xuất route và cờ escalation**.

**Trả lời của bạn:**

Tôi lựa chọn lát cắt là: Một ticket hỗ trợ mới đi vào hệ thống -> AI phân loại (`category`), đánh giá độ khẩn cấp (`urgency`), xác định nhu cầu có cần người thật xử lý hay không (`requires_human`), đề xuất team phụ trách (`route_to`) và tóm tắt lý do (`reason_summary`).
Đây là đơn vị công việc (Unit of Work) nhỏ nhất, độc lập và có ý nghĩa nghiệp vụ hoàn chỉnh. Việc cô lập tác vụ ở đây giúp kiểm tra trực tiếp khả năng hiểu ngữ nghĩa và phân loại của model mà không bị ảnh hưởng bởi các yếu tố bên ngoài như độ trễ của database hoặc lỗi hiển thị của giao diện UI.

### 2. Quality Question

Viết một câu hỏi chất lượng đủ cụ thể cho lát cắt này.

Gợi ý:

- Đừng hỏi kiểu quá rộng như: “AI có triage tốt không?”
- Nếu AI làm sai ở đây, điều gì sẽ khiến khách hàng mất trust hoặc không hoàn thành mục tiêu?
- Behavior nào là bắt buộc?
- Behavior nào là bị cấm?
- Viết theo dạng: **AI có gắn đúng route và escalation để ticket không bị đi sai hàng xử lý không?**

**Trả lời của bạn:**

AI có phân loại đúng loại yêu cầu, xác định chính xác mức độ khẩn cấp và kích hoạt cờ escalation cho khách hàng Enterprise gặp sự cố nghiêm trọng, để tránh trường hợp ticket bị chuyển sai bộ phận hoặc bị trễ hạn xử lý hay không?
Nếu AI trả lời sai câu hỏi này, hậu quả vận hành là các ticket khẩn cấp của khách hàng Enterprise có thể bị bỏ quên trong hàng đợi ưu tiên thấp, hoặc ticket Billing bị chuyển nhầm sang Product Team, làm tăng thời gian giải quyết sự cố, gây vỡ cam kết dịch vụ (SLA) và làm mất lòng tin của các khách hàng lớn.

### 3. Output Contract tối thiểu

Không cần đoán full JSON hoàn chỉnh. Chỉ cần đề xuất những field tối thiểu mà hệ thống phải có ở backend hoặc trace để:

- render UI ở trên,
- route đúng hàng xử lý,
- trigger escalation nếu cần,
- và chạy eval sau này.

Mẹo lấy từ ví dụ full luồng:

- Hãy nhìn ngược từ UI và mock outcome.
- Field nào không làm thay đổi màn hình, routing hoặc gate thì chưa cần đưa vào.

**Trả lời của bạn:**

Dưới đây là các field tối thiểu trong output contract và lý do:
- `ticket_id` (string): Để khớp kết quả với ticket đầu vào phục vụ cho việc logging, hiển thị trên UI và chạy eval sau này.
- `category` (enum: `technical`, `billing`, `product_question`, `general_inquiry`, `unknown`): Nhãn danh mục chính để lọc và hiển thị trên UI gợi ý.
- `urgency` (enum: `low`, `medium`, `high`, `critical`): Mức độ khẩn cấp để xếp thứ tự ưu tiên trong hàng đợi.
- `route_to` (enum: `technical_support`, `billing_ops`, `product_team`, `support_l1`, `human_escalation`): Chỉ định chính xác hàng đợi đích để hệ thống tự động định tuyến.
- `requires_human` (boolean): Cờ cảnh báo chuyển thẳng cho nhân viên trực để can thiệp lập tức (hiển thị dưới dạng "Cần người xử lý ngay" trên UI).
- `reason_summary` (string): Tóm tắt ngắn gọn lý do phân loại để nhân viên trực có thể đọc nhanh trong 3 giây mà không cần đọc hết email dài.
- `confidence` (float [0.0 - 1.0]): Độ tự tin của AI để hỗ trợ đưa ra quyết định tự động chuyển tuyến hoặc nâng cấp khi AI không chắc chắn.

### 4. Eval Decision Map

Ở phần này, bạn phải **tự quyết định** đâu là các thành phần thật sự cần chấm.

Đừng chép lại toàn bộ business rules hay toàn bộ UI. Hãy chọn ra những thành phần quan trọng nhất, bám vào:

- `Output Contract` bạn đã đề xuất
- quyết định nào thật sự làm thay đổi route, escalation, hoặc safety
- chỗ nào nếu sai sẽ gây hậu quả vận hành rõ ràng

| Thành phần cần chấm | Code | LLM | Human | Expert | Lý do |
| --- | ---: | ---: | ---: | ---: | --- |
| Schema & Enum validation | Yes | No | No | No | Codebase tự động kiểm tra tính hợp lệ của cấu trúc dữ liệu JSON và các giá trị enum một cách nhanh chóng, rẻ và chính xác 100% |
| Rule: Enterprise + High/Critical Urgency -> Requires Human | Yes | No | No | No | Đây là một quy tắc nghiệp vụ mang tính xác định (deterministic rule) có thể dễ dàng viết bằng biểu thức logic trong code |
| Rule: Billing ticket không route sang Product Team | Yes | No | No | No | Mối quan hệ loại trừ này hoàn toàn kiểm tra được bằng code thông qua câu lệnh logic đơn giản |
| Phân loại `category` và `urgency` theo ngữ nghĩa | No | Yes | Yes | No | Yêu cầu đọc hiểu ngữ cảnh ngôn ngữ tự nhiên phức tạp mà code thông thường không làm được. LLM-as-judge sẽ chấm tự động dựa trên rubric, kết hợp Human review định kỳ để hiệu chuẩn |
| Tính chính xác và trung thực của `reason_summary` | No | Yes | Yes | No | Tránh lỗi AI bịa đặt thông tin (hallucination). LLM-as-judge sẽ kiểm tra sự tương đồng ngữ nghĩa giữa summary và email gốc |

### 5. Kiểm tra tự động bằng code

Liệt kê **đầy đủ** các rule kiểm tra tự động mà bạn cho rằng case này cần có.

Không giới hạn số lượng. Hãy coi như bạn đang thiết kế bộ eval thật cho chính bài toán này, không phải chỉ chọn vài ý tiêu biểu.

Ưu tiên các kiểm tra mà nếu fail thì ticket sẽ đi sai hàng, thiếu escalation, hoặc vỡ schema.

Mỗi ý nên viết theo dạng:

- Kiểm tra: Cấu trúc JSON trùng khớp hoàn toàn với JSON Schema quy định và các giá trị enum của `category`, `urgency`, `route_to` nằm trong danh sách cho phép.
  Vì sao nên giao cho code: Đây là tác vụ cú pháp (syntax validation) thuần túy, code chạy tức thời trong mili-giây, chính xác tuyệt đối và hoàn toàn miễn phí.
- Kiểm tra: Quy tắc bắt buộc chuyển xử lý người thật cho khách hàng Enterprise. Nếu `customer_tier == "enterprise"` và `urgency` thuộc `high` hoặc `critical`, thì `requires_human` bắt buộc phải là `true`.
  Vì sao nên giao cho code: Đây là logic boolean xác định (deterministic), lập trình bằng code giúp bảo đảm 100% không bao giờ bị lọt lưới do model suy luận không ổn định.
- Kiểm tra: Quy tắc ngăn định tuyến sai cho bộ phận Billing. Nếu `category == "billing"`, thì `route_to` bắt buộc phải khác `product_team`.
  Vì sao nên giao cho code: Ràng buộc định tuyến mang tính loại trừ tuyệt đối, dễ dàng kiểm tra bằng một câu lệnh assert đơn giản trong unit test.
- Kiểm tra: Giá trị độ tin cậy hợp lệ. Trường `confidence` phải là kiểu số thực nằm trong đoạn `[0.0, 1.0]`.
  Vì sao nên giao cho code: Các phép so sánh số học và kiểu dữ liệu là thế mạnh của code, không cần tốn token cho LLM chấm điểm.
- Kiểm tra: Bắt buộc nâng mức độ khẩn cấp dựa trên từ khóa nhạy cảm. Nếu nội dung ticket chứa các chuỗi con như "locked out", "blocking work", "account disabled" (không phân biệt chữ hoa thường), thì `urgency` không được gán nhãn là `low`.
  Vì sao nên giao cho code: Một kiểm tra chuỗi (string scan) đơn giản bằng code giúp lọc nhanh các lỗi phân loại thiếu trách nhiệm của AI trước khi đưa qua các bước kiểm tra phức tạp hơn.

### 6. Tiêu chí chấm bằng LLM

Liệt kê **đầy đủ** các tiêu chí semantic mà case này cần có và code không chấm tốt.

Không giới hạn số lượng. Hãy coi như đây là bộ tiêu chí bạn thật sự sẽ dùng để chấm case này.

Chỉ giữ những tiêu chí mà cần đọc hiểu nghĩa của ticket hoặc mức độ hợp lý của lý do tóm tắt.

Mỗi ý nên viết theo dạng:

- Tiêu chí: **Category Semantic Accuracy** (AI phân loại đúng bản chất vấn đề của khách hàng dựa trên nội dung mô tả).
  Vì sao code không bắt tốt: Khách hàng mô tả lỗi bằng ngôn ngữ tự nhiên vô cùng đa dạng (ví dụ: "lỗi thanh toán", "không trừ tiền", "đóng phí thất bại" đều thuộc nhóm billing). Code thông thường hay regex không thể bao bao quát hết ngữ nghĩa phong phú này mà cần khả năng suy luận của LLM.
- Tiêu chí: **Urgency Realism** (Mức độ khẩn cấp được gán tương xứng với mức độ ảnh hưởng đến hoạt động vận hành của khách hàng được mô tả trong ticket).
  Vì sao code không bắt tốt: Cần đánh giá mức độ nghiêm trọng dựa trên sắc thái ngôn từ (sự giận dữ, mức độ khẩn thiết) và tác động thực tế của sự cố lên hoạt động của doanh nghiệp, đòi hỏi khả năng phán đoán ngữ cảnh của LLM.
- Tiêu chí: **Reason Summary Faithfulness & Conciseness** (Bản tóm tắt lý do phải trung thực với thông tin gốc, không được tự suy diễn hoặc bịa đặt sự thật bên ngoài, đồng thời độ dài tối đa là 1 câu ngắn gọn).
  Vì sao code không bắt tốt: Việc phát hiện lỗi bịa đặt thông tin (hallucination) yêu cầu đối chiếu và phân tích mối quan hệ ngữ nghĩa giữa hai đoạn văn bản tự do, điều này vượt quá khả năng của code thông thường.

### 7. Human / Expert Review

- Ai cần review?
- Review những case nào?
- Có cần domain expert không? Nếu không, vì sao?

**Trả lời của bạn:**

- **Ai cần review**: Trưởng nhóm CSKH (Support Ops Lead).
- **Review những case nào**: 
  - Các case AI gán độ tự tin thấp (`confidence < 0.7`).
  - Các case mà LLM-as-judge phát hiện nghi ngờ lỗi hoặc cho điểm chất lượng thấp dưới ngưỡng an toàn.
  - Audit ngẫu nhiên 5-10% tổng lượng ticket hàng ngày để hiệu chuẩn (calibrate) bộ prompt chấm điểm của LLM-as-judge.
- **Có cần domain expert không**: **Không**. Nghiệp vụ phân loại ticket của một công ty SaaS B2B hoàn toàn tuân theo các quy tắc vận hành nội bộ và hướng dẫn chăm sóc khách hàng chuẩn mực (SOP). Support Ops Lead là người nắm rõ nhất và thực thi các quy trình này hàng ngày, do đó họ có đầy đủ năng lực để đưa ra phán quyết chính xác mà không cần các chuyên gia đặc thù có chứng chỉ y khoa, pháp lý hay tài chính chuyên sâu.

Nếu chọn **có domain expert**, bạn phải làm thêm 2 phần dưới đây. Nếu **không cần domain expert**, hãy ghi `Không áp dụng` và giải thích 1 câu.

#### 7A. Màn hình cho Domain Expert (ASCII)

Không áp dụng (Vì nghiệp vụ phân loại ticket hỗ trợ khách hàng không yêu cầu chuyên gia chuyên ngành chuyên sâu thẩm định mà chỉ cần Trưởng nhóm CSKH trực tiếp kiểm tra).

#### 7B. Tiêu chí review của Domain Expert

Không áp dụng.

### 8. Release Gate

Đề xuất release gate phù hợp cho case này. Nêu rõ điều kiện chặn, ngưỡng chất lượng tối thiểu, và trường hợp cần human review.

**Trả lời của bạn:**

Bộ lọc duyệt phát hành (Release Gate) bắt buộc trước khi triển khai cấu hình prompt hoặc model mới lên production bao gồm:
1. **Safety Gate (Code Assertion - Yêu cầu đạt 100%)**:
   - Tỷ lệ vỡ schema và sai enum = 0%.
   - Tỷ lệ vi phạm rule nghiệp vụ Enterprise (Khách enterprise gặp lỗi khẩn cấp nhưng cờ `requires_human` bằng `false`) = 0%.
   - Tỷ lệ vi phạm định tuyến Billing sang Product Team = 0%.
2. **Quality Gate (LLM Judge - Yêu cầu đạt >= 95%)**:
   - Tỷ lệ gán nhãn đúng `category` (so với tập golden dataset) >= 95%.
   - Tỷ lệ gán nhãn đúng `urgency` >= 95%.
   - Điểm trung thực của `reason_summary` (đo bằng LLM-as-judge) đạt 100% (không có hallucination).
3. **Performance Gate (Yêu cầu đạt >= 95%)**:
   - Average Latency < 1.5 giây.
   - P95 Latency < 2.5 giây.
Nếu bất kỳ chỉ số nào rơi vào khoảng nghi ngờ (ví dụ: Quality Gate đạt từ 90% - 94%), hệ thống sẽ chặn tự động triển khai và chuyển toàn bộ tập test lỗi sang cho Support Ops Lead đánh giá thủ công (Human Review) trước khi đưa ra quyết định cuối cùng.

### 9. Kế hoạch chạy thử và dự toán chi phí

Làm phần này với giả định team của bạn vừa nhận đề bài triage này từ công ty.

Bạn là PM phụ trách đề xuất cách xây bộ eval, cách chạy thử, và chi phí cần xin để trả lời câu hỏi:

- hướng làm này hiện chính xác tới đâu
- cần thêm những checkpoint nào trước khi đề xuất triển khai tiếp
- và với một khoản budget thử nghiệm nhỏ, team có thể chứng minh được gì

README của folder này chỉ cho khung tính. Hãy giữ cách tính đơn giản: phần người tính bằng `giờ công`, phần máy tính bằng `chi phí API key` tính từ **giá thật** của model / dịch vụ bạn chọn.

Để làm phần này, bạn cần tự tính và nêu rõ:

- giá API thật bạn dùng để tính
- tổng số cases pilot dự kiến
- tổng số lần chạy / lặp lại dự kiến
- tổng giờ PM / thiết kế eval
- tổng giờ vận hành / kỹ thuật
- tổng giờ human review
- nếu có `domain expert`, tổng số giờ expert
- tổng chi phí API key
- tổng chi phí pilot
- tổng thời gian dự kiến

Có thể lấy mốc tham khảo để nhẩm nhanh:

- khoảng `50-100 cases`
- khoảng `30-50 lần chạy / lặp lại`

Không cần trình bày thành bảng. Hãy tự chọn cách trình bày miễn là người đọc nhìn vào hiểu được bạn đã tính gì và chi phí tổng rơi vào đâu.

Sau phần này, viết thêm 2-4 câu ngắn:

- bạn dùng giá API thật từ đâu để tính,
- với quy mô này chi phí tổng rơi vào khoảng nào,
- và vì sao plan này đủ để chứng minh case có thể pilot được.

**Trả lời của bạn:**

**Chi tiết kế hoạch pilot và dự toán ngân sách:**
- **Mô hình & Giá API thực tế**:
  - Mô hình chạy Triage chính: **Gemini 1.5 Flash** (Giá: $0.075 / 1M input tokens và $0.30 / 1M output tokens).
  - Mô hình chấm điểm LLM Judge: **Gemini 1.5 Pro** (Giá: $1.25 / 1M input tokens và $5.00 / 1M output tokens).
- **Tham số quy mô thử nghiệm**:
  - Tổng số cases trong tập tham chiếu (Reference Dataset): 100 cases.
  - Tổng số lần chạy thử nghiệm lặp lại để tối ưu hóa prompt (Iterations): 30 lần.
  - Tổng số lượt gọi API cho mô hình chính: 100 * 30 = 3,000 lượt.
  - Tổng số lượt gọi API cho LLM Judge: 3,000 lượt.
  - Kích thước trung bình mỗi case: Input 1,000 tokens (bao gồm system prompt, dữ liệu thô và vài ví dụ mẫu), Output 150 tokens.
- **Tính toán chi phí**:
  - *Chi phí API Gemini 1.5 Flash*: 3,000 lượt * ((1,000 * $0.075/1M) + (150 * $0.30/1M)) = 3,000 * ($0.000075 + $0.000045) = $0.36.
  - *Chi phí API Gemini 1.5 Pro (LLM Judge)*: 3,000 lượt * ((1,500 * $1.25/1M) + (200 * $5.00/1M)) = 3,000 * ($0.001875 + $0.001) = $8.63.
  - *Tổng chi phí API*: ~$9 (làm tròn lên để dự phòng token phát sinh).
  - *Chi phí nhân sự (tính theo giờ công, trung bình $20/giờ)*:
    - PM / Thiết kế Eval (Soạn thảo rubric, kịch bản edge case): 15 giờ = $300.
    - Kỹ sư vận hành / Kỹ thuật (Setup pipeline ghi log, build tool chạy test tự động): 10 giờ = $200.
    - Support Ops Lead (Dán nhãn thủ công tập mẫu ban đầu và audit kết quả lỗi): 5 giờ = $100.
    - Chuyên gia ngành (Domain Expert): 0 giờ (Không áp dụng).
  - *Tổng chi phí Pilot*: Chi phí API ($9) + Chi phí nhân sự ($600) = $609.
  - *Tổng thời gian triển khai dự kiến*: 1 tuần.

Tôi sử dụng giá API chính thức của Google Gemini từ website của Google Cloud Developer Platform. Với quy mô này, tổng chi phí dự án thử nghiệm chỉ rơi vào khoảng hơn $600 (khoảng 15 triệu VND), trong đó chi phí cho máy tính (API) là cực kỳ rẻ (chỉ khoảng $9) và phần lớn chi phí nằm ở công sức tối ưu hóa của con người. Kế hoạch này hoàn toàn khả thi để trình ban giám đốc vì nó cho phép chứng minh độ chính xác thực tế của AI trên tập 100 tình huống sát sườn nhất trước khi đầu tư hạ tầng lớn.

---
