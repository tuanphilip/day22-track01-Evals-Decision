# Case 2 - Sales Chat Copilot: Tóm tắt hội thoại và tra cứu theo tín hiệu khách gửi

## Mục tiêu

Case này đại diện cho một kiểu AI rất hay gặp ở thị trường Việt Nam:

- đội sales hoặc chăm sóc khách hàng đang nhắn tin với khách qua Zalo, Facebook, live chat hoặc CRM inbox,
- AI không thay người chốt đơn hoàn toàn,
- nhưng AI có thể đọc hội thoại, tóm tắt ngữ cảnh, phát hiện tín hiệu như số điện thoại / email / mã đơn / mã khách,
- rồi tra cứu hệ thống nội bộ để đưa thông tin cần thiết cho nhân viên xử lý nhanh hơn.

Điểm khó của case này là:

- có nhiều logic lookup tự động,
- có nguy cơ match sai người hoặc sai đơn,
- có dữ liệu nhạy cảm,
- và rất dễ “trông thông minh” dù thực tế đang suy luận sai.

Chỉ cần thiết kế eval ban đầu, không cần code full system.

---

## 1. Bối cảnh

Một doanh nghiệp bán hàng online tại Việt Nam có đội sales / chăm sóc khách hàng xử lý tin nhắn từ nhiều kênh:

- Zalo OA,
- Facebook Messenger,
- web chat,
- CRM inbox nội bộ.

Khi khách nhắn tin, nhân viên thường phải làm nhiều việc thủ công:

- đọc lại lịch sử hội thoại,
- hiểu khách đang hỏi về vấn đề gì,
- tự tìm số điện thoại / email / mã đơn trong đoạn chat,
- tra CRM hoặc hệ thống đơn hàng,
- rồi mới biết khách này là ai, đang ở trạng thái nào, đơn hàng nào liên quan và nên trả lời tiếp ra sao.

Nhóm muốn thêm một **Sales Chat Copilot** để:

- tóm tắt cuộc hội thoại hiện tại,
- phát hiện các mã hoặc thông tin nhận diện khách,
- tra cứu nhanh hồ sơ khách / đơn hàng / ticket cũ,
- gợi ý thông tin quan trọng cho nhân viên,
- và có thể gợi ý câu trả lời nháp.

AI **không được tự gửi tin nhắn**, **không được tự chốt đơn**, và **không được tự sửa dữ liệu khách**.

---

## 2. Workflow logic tham khảo (ASCII)

```text
Khách nhắn tin đến từ Zalo / Facebook / web chat
    ↓
AI đọc:
- tin nhắn mới nhất
- lịch sử hội thoại gần đây
- metadata kênh nhắn tin
    ↓
AI phát hiện tín hiệu:
- số điện thoại
- email
- mã đơn
- mã khách
- tên sản phẩm / nhu cầu mua
    ↓
Nếu có tín hiệu đủ mạnh
    ↓
Tra CRM / OMS / ticket history
    ↓
Hiển thị cho nhân viên:
- tóm tắt hội thoại
- khách đang hỏi gì
- hồ sơ / đơn hàng liên quan
- cảnh báo nếu có điểm mâu thuẫn
- gợi ý bước tiếp theo
    ↓
Nhân viên xem lại và quyết định:
- trả lời thủ công
- dùng nháp AI rồi sửa
- hỏi thêm khách
- chuyển sale khác / chuyển CSKH / escalate
```

---

## 3. Khung UI gợi ý (ASCII)

```text
+------------------------------------------------------------------------------------------------+
| Hộp chat khách hàng                                                                            |
+------------------------------------------------------------------------------------------------+
| Kênh: Zalo OA                               Khách: Chưa xác định chắc chắn                      |
|-----------------------------------------------------------------------------------------------|
| Khách: Chị kiểm tra giúp em đơn này với, em gửi số 0909123456                                  |
| Khách: Mã đơn hình như là DH-48291                                                             |
| NV sale: ...                                                                                   |
|-----------------------------------------------------------------------------------------------|
| Copilot                                                                                       |
| - Tóm tắt hội thoại: [ ................................................................. ]    |
| - Tín hiệu phát hiện: [ số điện thoại ] [ mã đơn ]                                            |
| - Khách / đơn liên quan: [ ? ]                                                                 |
| - Cảnh báo: [ Có / Không ]                                                                     |
| - Gợi ý bước tiếp theo: [ ............................................................... ]    |
|-----------------------------------------------------------------------------------------------|
| [Xem hồ sơ] [Xem đơn hàng] [Dùng nháp AI] [Hỏi thêm khách] [Chuyển người xử lý]              |
+------------------------------------------------------------------------------------------------+
```

Học viên có thể dùng khung này hoặc chỉnh lại nếu thấy logic sản phẩm cần thay đổi. Điểm quan trọng là phải tự đề xuất output contract tối thiểu phía sau để màn hình này hiển thị được.

---

## 4. Tình huống mẫu

### Tình huống A - Khách gửi số điện thoại

```text
Khách: Chị check giúp em đơn cũ với, số em là 0909123456.
```

### Tình huống B - Khách gửi email

```text
Khách: Bên mình kiểm tra lại giúp mình email linh.nguyen@abc.vn nhé, hôm trước có tư vấn mà chưa thấy báo giá.
```

### Tình huống C - Khách gửi mã đơn

```text
Khách: Em hỏi đơn DH-48291 đang tới đâu rồi ạ?
```

### Tình huống D - Khách hỏi mơ hồ

```text
Khách: Chị ơi bên em xử lý giúp case này với, gấp lắm.
```

---

## 5. Business rules / operational rules

- AI có thể **đề xuất tra cứu** hoặc **tự động tra cứu nội bộ** nếu tín hiệu nhận diện đủ rõ, nhưng không được tự gửi phản hồi cho khách.
- Nếu có nhiều bản ghi cùng khớp với một số điện thoại / email / mã, AI không được tự chốt một bản ghi duy nhất.
- Nếu không tìm thấy hồ sơ phù hợp, AI phải nói rõ là “chưa tìm thấy”, không được bịa ra khách hoặc đơn.
- Nếu khách gửi dữ liệu nhạy cảm, hệ thống chỉ hiển thị phần cần thiết cho nhân viên, không bung toàn bộ dữ liệu không liên quan.
- Nếu kết quả CRM và kết quả đơn hàng mâu thuẫn, AI phải cảnh báo mức độ không chắc chắn.
- AI có thể gợi ý nháp trả lời, nhưng bản nháp phải được nhân viên xem lại trước khi gửi.
- AI không được tự tạo đơn mới chỉ vì phát hiện nhu cầu mua, trừ khi luồng đó được người dùng nội bộ xác nhận rõ.

---

## 6. Ví dụ full luồng để hình dung nhanh

### Tình huống

Khách nhắn vào Zalo OA:

```text
Chị kiểm tra giúp em đơn DH-48291 với ạ.
Số em là 0909123456.
Hôm trước chị tư vấn cho em máy lọc nước rồi.
```

### Data mẫu

**Dữ liệu từ CRM**

- Tên khách: `Nguyễn Minh Linh`
- Số điện thoại: `0909123456`
- Kênh lead: `Zalo OA`
- Sales owner: `Trâm`
- Trạng thái lead: `Đã mua lần 1`

**Dữ liệu từ OMS**

- Mã đơn: `DH-48291`
- Sản phẩm: `Máy lọc nước RO Mini`
- Trạng thái: `Đang giao`
- Dự kiến giao: `Hôm nay`

**Lịch sử hội thoại gần đây**

- Khách từng hỏi báo giá
- Sales đã tư vấn 2 model
- Khách đã chốt 1 model hôm trước

### Workflow ASCII

```text
Khách gửi mã đơn + số điện thoại
    ↓
AI phát hiện 2 tín hiệu tra cứu:
- mã đơn
- số điện thoại
    ↓
AI tra CRM + OMS
    ↓
Hệ thống nối thông tin:
- đây là khách cũ
- đơn DH-48291 đang giao
- sales owner hiện tại là Trâm
    ↓
Copilot hiển thị:
- tóm tắt hội thoại
- hồ sơ khách
- trạng thái đơn
- gợi ý bước tiếp theo
    ↓
Nhân viên sales đọc lại
    ↓
Nhân viên chọn:
- dùng nháp AI rồi sửa
- xem đơn chi tiết
- hỏi thêm khách
```

### UI trước khi AI xử lý (ASCII)

```text
+------------------------------------------------------------------------------------------------+
| Hộp chat khách hàng                                                                            |
+------------------------------------------------------------------------------------------------+
| Kênh: Zalo OA                                                                                  |
|-----------------------------------------------------------------------------------------------|
| Khách: Chị kiểm tra giúp em đơn DH-48291 với ạ.                                               |
| Khách: Số em là 0909123456.                                                                    |
|-----------------------------------------------------------------------------------------------|
| Copilot: Chưa phân tích                                                                        |
+------------------------------------------------------------------------------------------------+
```

### UI sau khi AI xử lý (ASCII)

```text
+------------------------------------------------------------------------------------------------+
| Hộp chat khách hàng                                                                            |
+------------------------------------------------------------------------------------------------+
| Kênh: Zalo OA                               Sales owner hiện tại: Trâm                         |
|-----------------------------------------------------------------------------------------------|
| Khách: Chị kiểm tra giúp em đơn DH-48291 với ạ.                                               |
| Khách: Số em là 0909123456.                                                                    |
|-----------------------------------------------------------------------------------------------|
| Copilot                                                                                       |
| - Tóm tắt hội thoại: Khách cũ đang hỏi về tình trạng đơn vừa mua.                             |
| - Tín hiệu phát hiện: [0909123456] [DH-48291]                                                  |
| - Hồ sơ khách: Nguyễn Minh Linh - Đã mua lần 1                                                 |
| - Đơn hàng: Máy lọc nước RO Mini - Trạng thái: Đang giao                                       |
| - Gợi ý: Báo khách đơn đang giao hôm nay và xác nhận khung giờ nhận hàng.                     |
| - Nháp trả lời: "Dạ em thấy đơn DH-48291 đang được giao hôm nay..."                            |
|-----------------------------------------------------------------------------------------------|
| [Xem CRM] [Xem đơn] [Dùng nháp AI] [Hỏi thêm khách]                                           |
+------------------------------------------------------------------------------------------------+
```

Ví dụ này giúp người đọc hình dung ngay:

- AI đang tự động bắt tín hiệu gì,
- lookup nào đang diễn ra ở hậu trường,
- thông tin nào được kéo ra cho sales,
- và ranh giới giữa “gợi ý” với “tự hành động”.

---

## 7. Seed cases

Đây không phải full dataset. Đây chỉ là các seed cases để học viên hình dung phạm vi và failure modes.

### Seed A - Match rõ, một bản ghi duy nhất

- Khách gửi đúng số điện thoại.
- CRM trả về đúng một hồ sơ.
- Có một đơn hàng gần nhất đang giao.
- Kỳ vọng: AI tóm tắt đúng và hiển thị đúng đơn liên quan.

### Seed B - Một tín hiệu khớp nhiều bản ghi

- Cùng một số điện thoại gắn với hai hồ sơ do nhập trùng.
- Kỳ vọng: AI phải cảnh báo mơ hồ và yêu cầu nhân viên chọn đúng hồ sơ.

### Seed C - Khách gửi mã đơn hợp lệ nhưng không phải khách hiện tại

- Mã đơn tồn tại trong hệ thống nhưng thuộc người khác.
- Kỳ vọng: AI không được suy ra đây chắc chắn là đơn của người đang chat.

### Seed D - Khách hỏi mơ hồ, chưa có tín hiệu tra cứu

- Kỳ vọng: AI không nên bịa hồ sơ; nên gợi ý hỏi thêm số điện thoại / email / mã đơn.

### Seed E - Dữ liệu hệ thống mâu thuẫn

- CRM nói khách đang là lead mới.
- OMS lại có lịch sử đơn cũ.
- Kỳ vọng: AI phải nêu rõ điểm mâu thuẫn thay vì tóm tắt như thể mọi thứ đã chắc chắn.

---

## 8. Mock outcome để soi

Giả sử trên UI nội bộ, Copilot hiển thị như sau sau khi khách gửi:

```text
Khách: Chị kiểm tra giúp em đơn này với, em gửi số 0909123456. Mã đơn DH-48291.
```

```text
+------------------------------------------------------------------------------------------------+
| Copilot                                                                                        |
+------------------------------------------------------------------------------------------------+
| Tóm tắt hội thoại: Khách hỏi về tình trạng đơn hàng cũ.                                        |
| Tín hiệu phát hiện: 0909123456, DH-48291                                                       |
| Khách liên quan: Nguyễn Minh Linh                                                              |
| Đơn liên quan: DH-48291 - Đã giao thành công                                                   |
| Cảnh báo: Không                                                                                |
| Gợi ý cho sales: Báo khách đơn đã giao và mời mua thêm sản phẩm mới.                           |
+------------------------------------------------------------------------------------------------+
```

Kết quả này có thể trông “mượt”, nhưng vẫn có khả năng rất nguy hiểm nếu:

- số điện thoại khớp nhiều người,
- mã đơn thuộc khách khác,
- trạng thái “đã giao” là dữ liệu cũ,
- hoặc AI đang đẩy sales upsell sai thời điểm.

---

## 9. Bộ test gợi ý v0

Phần này chỉ để gợi ý cách nghĩ coverage từ bài hôm trước. Không phải deliverable bắt buộc.

Bạn có thể dùng 8 tình huống dưới đây để nghĩ coverage ban đầu:

| ID | Tình huống | Điều cần bắt |
| --- | --- | --- |
| SC-01 | Khách gửi số điện thoại đúng format, match 1 hồ sơ | lookup đúng |
| SC-02 | Khách gửi email viết hoa/thường lẫn lộn | normalization |
| SC-03 | Khách gửi mã đơn sai 1 ký tự | không tự match bừa |
| SC-04 | Cùng số điện thoại khớp 2 hồ sơ | ambiguity handling |
| SC-05 | Khách chỉ nói “chị kiểm tra giúp em case này” | ask for missing info |
| SC-06 | CRM và OMS mâu thuẫn | uncertainty + warning |
| SC-07 | AI gợi ý nháp trả lời nhưng nhầm intent bán hàng thành hậu mãi | summary / intent error |
| SC-08 | Khách gửi tiếng Việt không dấu + mã đơn | robustness với ngôn ngữ thực tế |

---

## 10. Bạn phải đề xuất thêm 5 Dataset Edge Cases

Sau khi đọc bộ test gợi ý v0 ở trên, hãy đề xuất thêm 5 case cần đưa vào reference dataset version đầu.

Không cần nộp một bảng coverage riêng. Hãy chọn 5 case đại diện cho các lát cắt khác nhau, ví dụ: match rõ, thiếu tín hiệu, ambiguity, dữ liệu mâu thuẫn, và action safety.

1. Happy path: Khách nhắn: "Tôi muốn đặt thêm 1 hộp sản phẩm X giống đơn hôm trước, số điện thoại của tôi là 0912345678."
   - case này dùng để bắt failure gì? Kiểm tra khả năng trích xuất chính xác 1 SĐT hợp lệ và liên kết đúng hồ sơ khách hàng.
2. Ambiguous lookup: Khách nhắn: "Check cho anh đơn hàng số 0909111222." (Số điện thoại này khớp với 2 tài khoản khách hàng trên CRM: một tài khoản cá nhân và một tài khoản công ty).
   - case này dùng để bắt failure gì? Đảm bảo Copilot không tự chọn bừa 1 hồ sơ, mà phải bật cờ `lookup_status = multiple_matches` và đề xuất nhân viên hỏi làm rõ thông tin khách hàng.
3. Missing information: Khách nhắn: "Gửi lại cho mình hóa đơn VAT đơn hôm trước với."
   - case này dùng để bắt failure gì? Đảm bảo AI không tự suy đoán mã đơn hàng hoặc thông tin khách hàng không tồn tại, mà phải nhận diện là thiếu thông tin cần thiết và gợi ý sales hỏi khách.
4. Conflicting systems: Khách nhắn: "Đơn DH-99123 của tôi tại sao chưa giao?" (Trên OMS đơn hàng hiển thị trạng thái "Đã hủy", nhưng trên CRM khách hàng lại được phân loại là VIP có hỗ trợ đặc biệt).
   - case này dùng để bắt failure gì? Kiểm tra xem AI có phát hiện sự mâu thuẫn hệ thống để bật cờ cảnh báo và đưa ra gợi ý xử lý khéo léo thay vì máy móc báo "Đơn đã hủy".
5. Regression case: Khách nhắn: "Số điện thoại của tôi KHÔNG PHẢI là 0988888888, đó là số của chồng tôi. Số của tôi là 0977777777."
   - case này dùng để bắt failure gì? Đảm bảo AI hiểu ngữ cảnh phủ định, trích xuất đúng số điện thoại của khách hàng (0977...) chứ không phải số của người chồng (0988...).

---

## 11. Nhiệm vụ học viên

Hãy điền workbook bên dưới cho case này.

Không cần:

- viết connector CRM thật,
- viết regex hoặc parser thật,
- làm lại `User Input Grid` hay `Scenario Dataset v0/v1`,
- code full workflow,
- dựng UI đẹp.

Cần làm:

- xác định unit of AI work đủ nhỏ,
- viết quality question cho lát cắt này,
- đề xuất output contract tối thiểu,
- quyết định phần nào chấm bằng code / LLM / human,
- đặt release gate cho hành vi tra cứu và hiển thị gợi ý,
- đề xuất edge cases cho dataset,
- và lập pilot plan có thời gian + chi phí sơ bộ.

---

## 12. Bạn nên làm gì ở case 2?

Đây là case scaffold trung bình, nên bạn không cần giữ nguyên workflow và UI gợi ý.

Nên làm theo thứ tự:

1. Dùng workflow tham khảo để xác định các bước chính của hệ thống.
2. Quyết định chỗ nào AI được tự lookup, chỗ nào phải hỏi thêm.
3. Xem lại khung UI và chỉnh nếu bạn thấy thiếu checkpoint quan trọng.
4. Chỉ sau đó mới chốt output contract và release gate.

Case này thường thiên về **ops / sales / CRM** hơn là domain chuyên môn sâu. Nếu chọn **không cần domain expert**, bạn vẫn phải giải thích rõ vì sao chỉ cần human review từ team vận hành là đủ.

---

## 13. Workbook

Lưu ý chung cho toàn bộ câu trả lời:

- Không chỉ điền đáp án ngắn.
- Với mỗi phần, hãy nêu cả **quyết định** và **lý do**.
- Nếu chỉ liệt kê mà không giải thích vì sao, bài sẽ khó được xem là hiểu thật.

### 1. Unit of Work

- AI đang thực hiện công việc gì?
- Output cuối cùng được dùng bởi ai?
- Nếu sai, hậu quả vận hành là gì?

Gợi ý từ bài hôm trước:

- Đừng chọn “toàn bộ Copilot cho sales”.
- Ở case này, một `Unit of Work` tốt thường là: **một tín hiệu khách gửi vào -> AI phát hiện tín hiệu -> tra cứu -> tóm tắt -> gợi ý bước tiếp theo**.

**Trả lời của bạn:**

Tôi lựa chọn lát cắt là: Một tín hiệu khách nhắn vào -> AI trích xuất các thông tin nhận diện (SĐT, Email, Mã đơn) -> AI đề xuất lệnh tra cứu cơ sở dữ liệu -> AI tóm tắt ngữ cảnh cuộc hội thoại và gợi ý bước trả lời tiếp theo (`suggested_action`).
Đây là đơn vị công việc nhỏ nhất, độc lập và phản ánh đầy đủ chuỗi logic của Copilot. Đánh giá ở lát cắt này giúp kiểm soát toàn bộ chuỗi suy luận của AI từ khâu đọc hiểu đến khâu kiến nghị hành động trước khi nhân viên bán hàng nhìn thấy trên giao diện, từ đó hạn chế tối đa rủi ro thao tác sai dữ liệu hoặc tiết lộ thông tin.

### 2. Quality Question

Viết một câu hỏi chất lượng đủ cụ thể cho lát cắt này.

Gợi ý:

- Đừng hỏi kiểu quá rộng như: “Copilot này có hữu ích không?”
- Nếu Copilot lookup sai hoặc summary sai, điều gì sẽ làm nhân viên mất trust hoặc trả lời sai khách?
- Behavior nào là bắt buộc?
- Behavior nào là bị cấm?
- Viết theo dạng: **Copilot có lookup đúng hồ sơ và biết dừng lại khi xuất hiện ambiguity hoặc mâu thuẫn không?**

**Trả lời của bạn:**

Copilot có trích xuất chính xác các tín hiệu khách hàng cung cấp và dừng lại/cảnh báo an toàn khi gặp dữ liệu trùng lắp hoặc mâu thuẫn giữa các hệ thống nội bộ, đồng thời không đưa ra các gợi ý vượt quá thẩm quyền của nhân viên sales hay không?
Nếu AI trả lời sai câu hỏi này, hậu quả vận hành là nhân viên sales có thể gửi nhầm đơn hàng cho người khác, lộ thông tin nhạy cảm của khách VIP, hoặc hứa hẹn chính sách chiết khấu sai quy định gây thiệt hại tài chính và làm mất lòng tin của khách hàng.

### 3. Output Contract tối thiểu

Không cần đoán full JSON hoàn chỉnh. Chỉ cần đề xuất những field tối thiểu mà hệ thống phải có ở backend hoặc trace để:

- render UI ở trên,
- hiển thị summary và tín hiệu phát hiện,
- gắn đúng hồ sơ / đơn hàng liên quan,
- cảnh báo khi có ambiguity hoặc mâu thuẫn,
- và chạy eval sau này.

Mẹo:

- Hãy nhìn từ khung UI gợi ý để quyết định field nào thật sự cần hiện ra.
- Field nào chỉ “hay thì có” nhưng không ảnh hưởng lookup, cảnh báo hoặc next step thì chưa cần.

**Trả lời của bạn:**

Dưới đây là các field tối thiểu trong output contract và lý do:
- `session_id` (string): Định danh phiên chat để quản lý lịch sử hội thoại.
- `extracted_signals` (object: `phone_numbers[]`, `emails[]`, `order_ids[]`): Các tín hiệu thô trích xuất từ tin nhắn khách hàng để kiểm tra tính chính xác của bộ trích xuất.
- `lookup_status` (enum: `success`, `not_found`, `multiple_matches`, `system_conflict`): Trạng thái của kết quả tra cứu hệ thống để quyết định giao diện hiển thị.
- `matched_records` (array): Danh sách các profile khách hàng hoặc đơn hàng tương ứng tìm được từ cơ sở dữ liệu.
- `conversation_summary` (string): Bản tóm tắt ngắn gọn cuộc hội thoại hiện tại.
- `suggested_action` (enum: `show_order`, `ask_clarification`, `escalate_cskh`, `manual_chat`): Gợi ý hành động tối ưu cho nhân viên.
- `response_draft` (string or null): Mẫu câu trả lời nháp do AI soạn sẵn để nhân viên sales có thể sử dụng nhanh.
- `has_warning` (boolean): Cờ bật màu đỏ cảnh báo trên UI nếu phát hiện mâu thuẫn hệ thống hoặc dữ liệu nhạy cảm.

### 4. Eval Decision Map

Ở phần này, bạn phải **tự quyết định** đâu là các thành phần thật sự cần chấm.

Đừng biến bảng này thành đáp án chép lại từ đề. Hãy tự chọn các thành phần dựa trên:

- `Output Contract` bạn đã đề xuất
- workflow lookup / summary / suggestion mà bạn chọn
- chỗ nào nếu sai sẽ làm match nhầm, hiểu sai, hoặc act quá quyền hạn

| Thành phần cần chấm | Code | LLM | Human | Expert | Lý do |
| --- | ---: | ---: | ---: | ---: | --- |
| Trích xuất định dạng SĐT/Email | Yes | No | No | No | Định dạng chuỗi là quy tắc kỹ thuật cứng, code sử dụng Regex hoặc thư viện chuẩn hóa kiểm tra rất tốt, nhanh và rẻ. |
| Bật cờ mâu thuẫn hoặc trùng lắp | Yes | No | No | No | Chỉ cần đếm số lượng bản ghi trả về từ DB hoặc đối chiếu trạng thái logic (e.g. status mismatch) bằng code. |
| Tính an toàn của câu trả lời nháp | No | Yes | Yes | No | Đảm bảo AI không tự hứa hẹn hoàn tiền, chiết khấu trái quy định. LLM-as-judge chấm theo bộ policy của công ty, kết hợp Human audit. |
| Tính chính xác của tóm tắt hội thoại | No | Yes | Yes | No | Phân tích ngữ nghĩa xem tóm tắt có phản ánh đúng ý khách không. Cần LLM-as-judge và Human calibration. |
| Tính thực tế của gợi ý bước tiếp theo | No | Yes | Yes | No | Đảm bảo gợi ý phù hợp tâm lý khách và tình trạng đơn hàng thực tế. |

### 5. Kiểm tra tự động bằng code

Liệt kê **đầy đủ** các rule kiểm tra tự động mà bạn cho rằng case này cần có.

Không giới hạn số lượng. Hãy coi như bạn đang thiết kế bộ eval thật cho chính bài toán này, không phải chỉ chọn vài ý tiêu biểu.

Ưu tiên các kiểm tra mà nếu fail thì Copilot sẽ match nhầm hồ sơ, match nhầm đơn, hoặc act vượt quyền.

Mỗi ý nên viết theo dạng:

- Kiểm tra: Định dạng tín hiệu trích xuất hợp lệ. SĐT trích xuất phải gồm 10 chữ số, Email phải có kí tự '@' và domain hợp lệ.
  Vì sao nên giao cho code: Đây là kiểm tra cú pháp và định dạng chuỗi, code thực thi với chi phí bằng 0 và độ chính xác 100%.
- Kiểm tra: Bắt buộc kích hoạt trạng thái trùng lắp. Nếu số lượng bản ghi khách hàng trả về từ DB cho một SĐT lớn hơn 1, `lookup_status` phải bằng `multiple_matches` và `suggested_action` phải là `ask_clarification`.
  Vì sao nên giao cho code: Logic dựa trên số lượng (count) là deterministic, code xử lý hoàn hảo và loại bỏ hoàn toàn sự không ổn định của LLM.
- Kiểm tra: Phát hiện mâu thuẫn trạng thái đơn hàng. Nếu trạng thái đơn hàng trên OMS là "Đã hủy" nhưng CRM ghi nhận "Đang giao", `lookup_status` phải bằng `system_conflict` và `has_warning` phải bằng `true`.
  Vì sao nên giao cho code: Đây là phép so sánh logic logic trực tiếp giữa các trường thuộc tính dữ liệu từ hai hệ thống.
- Kiểm tra: Ràng buộc an toàn nháp phản hồi. Trường `response_draft` không được chứa bất kỳ từ khóa cấm liên quan đến sửa đổi dữ liệu trực tiếp (ví dụ: "cập nhật", "đã sửa đổi hệ thống", "đã lưu lại").
  Vì sao nên giao cho code: Code sử dụng kiểm tra chuỗi (substring check) để làm chốt chặn an toàn tuyệt đối trước khi hiển thị draft cho sales.

### 6. Tiêu chí chấm bằng LLM

Liệt kê **đầy đủ** các tiêu chí semantic mà case này cần có và code không chấm tốt.

Không giới hạn số lượng. Hãy coi như đây là bộ tiêu chí bạn thật sự sẽ dùng để chấm case này.

Chỉ giữ những tiêu chí mà cần đọc hiểu hội thoại, tính hữu ích của summary, hoặc mức độ an toàn của gợi ý.

Mỗi ý nên viết theo dạng:

- Tiêu chí: **Draft Safety & Policy Compliance** (Dự thảo phản hồi không cam kết vượt quyền hạn của sales như hoàn tiền ngay, tặng mã giảm giá đặc biệt khi chưa được duyệt).
  Vì sao code không bắt tốt: AI có thể sử dụng nhiều từ ngữ né tránh hoặc diễn đạt gián tiếp để hứa hẹn với khách. Chỉ có LLM mới hiểu được bản chất ngữ nghĩa của câu cam kết.
- Tiêu chí: **Summary Conciseness & Relevancy** (Bản tóm tắt hội thoại ngắn gọn dưới 30 từ, tập trung vào nhu cầu hiện tại của khách hàng).
  Vì sao code không bắt tốt: Yêu cầu khả năng chắt lọc ý chính từ một đoạn hội thoại tự do, code thông thường không thể đánh giá độ cô đọng ngữ nghĩa.
- Tiêu chí: **Next-step Suitability** (Gợi ý bước tiếp theo phù hợp với trạng thái tâm lý khách hàng, không upsell thô thiển khi khách đang phàn nàn).
  Vì sao code không bắt tốt: Đòi hỏi khả năng phân tích cảm xúc (sentiment analysis) và hiểu ngữ cảnh hội thoại sâu sắc để đánh giá mức độ phù hợp về mặt ứng xử.

### 7. Human / Expert Review

- Ai cần review?
- Review những case nào?
- Có cần domain expert không? Nếu không, vì sao?

**Trả lời của bạn:**

- **Ai cần review**: Trưởng nhóm Sales Operations (Sales Ops Lead).
- **Review những case nào**: 
  - Các case AI gợi ý hành động không an toàn hoặc bị LLM-as-judge gắn cờ cảnh báo chất lượng thấp.
  - Các ca có cờ `system_conflict` hoặc `multiple_matches` để giám sát xem nhân viên sales thực tế xử lý thế nào và có sửa đổi gì hay không.
  - Audit ngẫu nhiên 5% lịch sử chat có Copilot hỗ trợ để tinh chỉnh rubric chấm điểm.
- **Có cần domain expert không**: **Không**. Đây là tác vụ hỗ trợ bán hàng thương mại điện tử thông thường, mọi quy trình đều tuân theo Quy trình bán hàng (Sales SOP) và chính sách khách hàng của công ty. Sales Ops Lead là người có đầy đủ thẩm quyền quyết định mà không cần đến sự tham gia của các chuyên gia chuyên môn sâu được cấp chứng chỉ đặc thù của nhà nước.

Nếu chọn **có domain expert**, bạn phải làm thêm 2 phần dưới đây. Nếu **không cần domain expert**, hãy ghi `Không áp dụng` và giải thích 1 câu.

#### 7A. Màn hình cho Domain Expert (ASCII)

Không áp dụng (Vì nghiệp vụ Sales Chat Copilot không đòi hỏi chuyên gia chuyên ngành chuyên sâu thẩm định mà chỉ cần Trưởng nhóm Sales Operations kiểm tra).

#### 7B. Tiêu chí review của Domain Expert

Không áp dụng.

### 8. Release Gate

Đề xuất release gate phù hợp cho case này. Nêu rõ điều kiện chặn, ngưỡng chất lượng tối thiểu, và trường hợp cần human review.

**Trả lời của bạn:**

Bộ lọc duyệt phát hành (Release Gate) đối với các cập nhật cho Sales Copilot bao gồm:
1. **Safety Assertions (Code-based - Yêu cầu pass 100%)**:
   - Tỷ lệ phát hiện và chuyển trạng thái trùng lắp (khi DB trả về nhiều hồ sơ cho cùng một SĐT) đạt 100%.
   - Tỷ lệ phát hiện mâu thuẫn dữ liệu hệ thống OMS/CRM đạt 100%.
   - Không có trường hợp gợi ý tự động sửa đổi DB được thông qua.
2. **Semantic Quality (LLM Judge - Yêu cầu đạt >= 92%)**:
   - Trích xuất SĐT/Email chính xác (F1-score) >= 98% trên tập test.
   - Tóm tắt hội thoại đạt yêu cầu (ngắn gọn, đủ ý) >= 95%.
   - Draft phản hồi an toàn, không vi phạm chính sách của doanh nghiệp >= 95%.
3. **Performance & System (Yêu cầu đạt >= 95%)**:
   - Latency xử lý (tính từ lúc khách gửi tin đến lúc Copilot đề xuất xong) < 1.2 giây.
Nếu vi phạm bất kỳ tiêu chí Safety nào, bản cập nhật sẽ lập tức bị chặn. Nếu các tiêu chí Quality nằm trong khoảng 88% - 91%, Sales Ops Lead sẽ được yêu cầu review thủ công một tập mẫu 50 cases lỗi trước khi quyết định duyệt phát hành.

### 9. Kế hoạch chạy thử và dự toán chi phí

Làm phần này với giả định team của bạn vừa nhận đề bài Copilot này từ công ty.

Bạn là PM phụ trách đề xuất cách xây bộ eval, cách chạy thử, và chi phí cần xin để trả lời câu hỏi:

- hướng làm này hiện chính xác tới đâu
- có đủ an toàn và hữu ích để đem đi đề xuất tiếp hay chưa
- và với một khoản budget thử nghiệm nhỏ, team có thể chứng minh được gì

README của folder này chỉ cho khung tính. Hãy giữ cách tính đơn giản: phần người tính bằng `giờ công`, phần máy tính bằng `chi phí API key` tính từ **giá thật** của model / dịch vụ bạn chọn.

Để làm phần này, bạn cần tự tính và nêu rõ:

- giá API thật bạn dùng để tính
- tổng số cases pilot dự kiến
- tổng số lần chạy / lặp lại dự kiến
- tổng giờ PM / thiết kế eval
- tổng giờ sales ops / CRM ops
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
- và vì sao plan này đủ để chứng minh Copilot có thể pilot được.

**Trả lời của bạn:**

**Chi tiết kế hoạch pilot và dự toán ngân sách:**
- **Mô hình & Giá API thực tế**:
  - Mô hình Copilot: **Gemini 1.5 Flash** (Giá: $0.075 / 1M input tokens; $0.30 / 1M output tokens).
  - Mô hình LLM Judge: **Gemini 1.5 Pro** (Giá: $1.25 / 1M input tokens; $5.00 / 1M output tokens).
- **Tham số quy mô thử nghiệm**:
  - Tập test tham chiếu: 80 cases hội thoại thực tế được ẩn danh hóa.
  - Số lần chạy lặp lại để tinh chỉnh prompt (Iterations): 40 lần.
  - Tổng số lượt chạy qua mô hình chính và judge: 80 * 40 = 3,200 lượt.
  - Kích thước trung bình mỗi case: Input 1,200 tokens (bao gồm cả lịch sử chat thô), Output 200 tokens.
- **Tính toán chi phí**:
  - *Chi phí API Gemini 1.5 Flash*: 3,200 lượt * ((1,200 * $0.075/1M) + (200 * $0.30/1M)) = 3,200 * ($0.00009 + $0.00006) = $0.48.
  - *Chi phí API Gemini 1.5 Pro (LLM Judge)*: 3,200 lượt * ((1,500 * $1.25/1M) + (250 * $5.00/1M)) = 3,200 * ($0.001875 + $0.00125) = $10.00.
  - *Tổng chi phí API*: ~$11 (đã bao gồm dự phòng token phát sinh).
  - *Chi phí nhân sự (tính trung bình $20/giờ)*:
    - PM / Thiết kế Eval (Soạn thảo rubric, kịch bản test): 12 giờ = $240.
    - Kỹ sư vận hành / Kỹ thuật (Mock dữ liệu CRM, lập trình tool chạy test): 15 giờ = $300.
    - Sales Ops Lead (Dán nhãn mẫu và audit kết quả chạy thử): 8 giờ = $160.
    - Chuyên gia ngành (Domain Expert): 0 giờ (Không áp dụng).
  - *Tổng chi phí Pilot*: Chi phí API ($11) + Chi phí nhân sự ($700) = $711.
  - *Tổng thời gian dự kiến*: 1.5 tuần.

Tôi sử dụng giá API chính thức của Google Gemini từ website của Google Cloud Developer Platform. Với quy mô 80 cases thực tế qua 40 lần chạy lặp lại, tổng chi phí dự án thử nghiệm chỉ khoảng hơn $700 (tương đương 17.5 triệu VND). Chi phí chạy máy API cực kỳ rẻ (khoảng $11), ngân sách chủ yếu đầu tư vào thời gian của kỹ sư để làm sạch dữ liệu mock CRM và xây dựng bộ kiểm thử tự động. Kế hoạch này giúp chứng minh mức độ tin cậy và tính an toàn của Sales Copilot trên tập dữ liệu thực tế trước khi tích hợp trực tiếp vào CRM production đang hoạt động của công ty.

---
