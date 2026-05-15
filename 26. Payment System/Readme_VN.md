# Chương 26: Hệ thống Thanh toán

## Giới thiệu
Trong chương này, chúng ta sẽ thiết kế một **hệ thống thanh toán**, nền tảng của mọi hoạt động **thương mại điện tử** hiện đại.

Một **hệ thống thanh toán** được sử dụng để giải quyết các giao dịch tài chính, chuyển giao giá trị tiền tệ.

---

## Bước 1: Hiểu vấn đề và thiết lập phạm vi thiết kế
 * C: Chúng ta đang xây dựng loại hệ thống thanh toán nào?
 * I: Một backend thanh toán cho hệ thống thương mại điện tử, tương tự như Amazon.com. Nó xử lý mọi thứ liên quan đến luân chuyển tiền tệ.
 * C: Những tùy chọn thanh toán nào được hỗ trợ - Thẻ tín dụng, PayPal, thẻ ngân hàng, v.v.?
 * I: Hệ thống nên hỗ trợ tất cả các tùy chọn này trong thực tế. Cho mục đích của buổi phỏng vấn, chúng ta có thể sử dụng thanh toán qua thẻ tín dụng.
 * C: Chúng ta có tự xử lý thẻ tín dụng không?
 * I: Không, chúng ta sử dụng nhà cung cấp bên thứ ba như Stripe, Braintree, Square, v.v.
 * C: Chúng ta có lưu trữ dữ liệu thẻ tín dụng trong hệ thống của mình không?
 * I: Vì lý do tuân thủ (compliance), chúng ta không lưu trữ trực tiếp dữ liệu thẻ tín dụng trong hệ thống của mình. Chúng ta dựa vào các bộ xử lý thanh toán bên thứ ba.
 * C: Ứng dụng có mang tính toàn cầu không? Chúng ta có cần hỗ trợ các loại tiền tệ khác nhau và thanh toán quốc tế không?
 * I: Ứng dụng mang tính toàn cầu, nhưng chúng ta giả định chỉ sử dụng một loại tiền tệ cho mục đích phỏng vấn.
 * C: Chúng ta cần hỗ trợ bao nhiêu giao dịch thanh toán mỗi ngày?
 * I: 1 triệu giao dịch mỗi ngày.
 * C: Chúng ta có cần hỗ trợ luồng chi trả (payout) cho người bán mỗi tháng không?
 * I: Có, chúng ta cần hỗ trợ việc đó.
 * C: Có điều gì khác tôi cần chú ý không?
 * I: Chúng ta cần hỗ trợ đối soát (reconciliation) để khắc phục bất kỳ sự không nhất quán nào trong việc giao tiếp với các hệ thống nội bộ và bên ngoài.

### **Yêu cầu chức năng**
 * Luồng nạp tiền (Pay-in flow) - hệ thống thanh toán nhận tiền từ khách hàng thay mặt cho người bán.
 * Luồng chi trả (Pay-out flow) - hệ thống thanh toán gửi tiền cho người bán trên toàn thế giới.

### **Yêu cầu phi chức năng**
 * Độ tin cậy và khả năng chịu lỗi. Các thanh toán thất bại cần được xử lý cẩn thận.
 * Cần thiết lập quy trình đối soát giữa hệ thống nội bộ và bên ngoài.

### **Ước tính nhanh (Back-of-the-envelope estimation)**
Hệ thống cần xử lý 1 triệu giao dịch mỗi ngày, tương đương khoảng 10 giao dịch mỗi giây.

Đây không phải là mức thông lượng cao đối với bất kỳ hệ thống cơ sở dữ liệu nào, vì vậy nó không phải là trọng tâm của buổi phỏng vấn này.

---

## Bước 2: Đề xuất thiết kế cấp cao và đạt được sự đồng thuận
Ở cấp độ cao, chúng ta có ba tác nhân tham gia vào việc luân chuyển tiền:

<div style="margin-left:3rem">
    <img src="./images/high-level-flow.png" alt="high-level-flow" width="500" />
</div>

### **Luồng nạp tiền (Pay-in flow)**
Dưới đây là tổng quan cấp cao của luồng nạp tiền:

<div style="margin-left:3rem">
    <img src="./images/payin-flow-high-level.png" alt="pay-in-flow-high-level" width="500" />
</div>

 * Dịch vụ thanh toán (Payment service) - chấp nhận các sự kiện thanh toán và điều phối quy trình thanh toán. Nó cũng thường thực hiện kiểm tra rủi ro bằng cách sử dụng nhà cung cấp bên thứ ba để kiểm tra các vi phạm AML (chống rửa tiền) hoặc hoạt động tội phạm.
 * Bộ thực thi thanh toán (Payment executor) - thực thi một lệnh thanh toán đơn lẻ thông qua Nhà cung cấp dịch vụ thanh toán (PSP). Các sự kiện thanh toán có thể chứa nhiều lệnh thanh toán.
 * Nhà cung cấp dịch vụ thanh toán (PSP) - chuyển tiền từ tài khoản này sang tài khoản khác, ví dụ: từ tài khoản thẻ tín dụng của người mua sang tài khoản ngân hàng của trang thương mại điện tử.
 * Hệ thống thẻ (Card schemes) - các tổ chức xử lý hoạt động thẻ tín dụng, ví dụ: Visa, MasterCard, v.v.
 * Sổ cái (Ledger) - lưu giữ hồ sơ tài chính của tất cả các giao dịch thanh toán.
 * Ví (Wallet) - giữ số dư tài khoản cho tất cả người bán.

Ví dụ về luồng nạp tiền:
 * người dùng nhấp vào "đặt hàng" và một sự kiện thanh toán được gửi đến dịch vụ thanh toán.
 * dịch vụ thanh toán lưu trữ sự kiện trong cơ sở dữ liệu của nó.
 * dịch vụ thanh toán gọi bộ thực thi thanh toán cho tất cả các lệnh thanh toán thuộc sự kiện đó.
 * bộ thực thi thanh toán lưu trữ lệnh thanh toán trong cơ sở dữ liệu của nó.
 * bộ thực thi thanh toán gọi PSP bên ngoài để xử lý thanh toán thẻ tín dụng.
 * Sau khi bộ thực thi thanh toán xử lý thanh toán, dịch vụ thanh toán cập nhật ví để ghi nhận số tiền người bán có.
 * dịch vụ ví lưu trữ thông tin số dư đã cập nhật trong cơ sở dữ liệu của nó.
 * dịch vụ thanh toán gọi sổ cái để ghi lại tất cả các chuyển động của tiền.

### **API cho dịch vụ thanh toán**
```
POST /v1/payments
{
  "buyer_info": {...},
  "checkout_id": "some_id",
  "credit_card_info": {...},
  "payment_orders": [{...}, {...}, {...}]
}
```

Ví dụ về `payment_order`:
```
{
  "seller_account": "SELLER_IBAN",
  "amount": "3.15",
  "currency": "USD",
  "payment_order_id": "globally_unique_payment_id"
}
```

Lưu ý:
 * `payment_order_id` được chuyển tiếp đến PSP để khử trùng lặp thanh toán, tức là nó là khóa idempotent.
 * Trường amount là `string` vì kiểu `double` không phù hợp để biểu diễn các giá trị tiền tệ.

```
GET /v1/payments/{:id}
```

Endpoint này trả về trạng thái thực thi của một thanh toán đơn lẻ, dựa trên `payment_order_id`.

### **Mô hình dữ liệu dịch vụ thanh toán**
Chúng ta cần duy trì hai bảng - `payment_events` và `payment_orders`.

Đối với thanh toán, hiệu năng thường không phải là yếu tố quan trọng nhất. Tuy nhiên, tính nhất quán mạnh (strong consistency) thì có.

Các cân nhắc khác khi chọn cơ sở dữ liệu:
 * Thị trường DBA dồi dào để thuê quản trị cơ sở dữ liệu.
 * Hồ sơ theo dõi đã được chứng minh khi cơ sở dữ liệu được sử dụng bởi các tổ chức tài chính lớn khác.
 * Sự phong phú của các công cụ hỗ trợ.
 * Ưu tiên SQL truyền thống hơn NoSQL/NewSQL vì các đảm bảo ACID của nó.

Dưới đây là những gì bảng `payment_events` chứa:
 * `checkout_id` - string, khóa chính.
 * `buyer_info` - string (ghi chú cá nhân - có lẽ một khóa ngoại đến bảng khác sẽ phù hợp hơn).
 * `seller_info` - string (ghi chú cá nhân - nhận xét tương tự như trên).
 * `credit_card_info` - tùy thuộc vào nhà cung cấp thẻ.
 * `is_payment_done` - boolean.

Dưới đây là những gì bảng `payment_orders` chứa:
 * `payment_order_id` - string, khóa chính.
 * `buyer_account` - string.
 * `amount` - string.
 * `currency` - string.
 * `checkout_id` - string, khóa ngoại.
 * `payment_order_status` - enum (`NOT_STARTED`, `EXECUTING`, `SUCCESS`, `FAILED`).
 * `ledger_updated` - boolean.
 * `wallet_updated` - boolean.

Lưu ý:
 * có nhiều lệnh thanh toán liên kết với một sự kiện thanh toán nhất định.
 * chúng ta không cần `seller_info` cho luồng nạp tiền. Điều đó chỉ bắt buộc đối với luồng chi trả.
 * `ledger_updated` và `wallet_updated` được cập nhật khi dịch vụ tương ứng được gọi để ghi lại kết quả thanh toán.
 * việc chuyển đổi trạng thái thanh toán được quản lý bởi một background job, công việc này kiểm tra các cập nhật của các thanh toán đang xử lý và kích hoạt cảnh báo nếu một thanh toán không được xử lý trong một khoảng thời gian hợp lý.

### **Hệ thống sổ cái ghi kép (Double-entry ledger system)**
Cơ chế kế toán ghi kép là chìa khóa của bất kỳ hệ thống thanh toán nào. Đó là một cơ chế theo dõi các chuyển động của tiền bằng cách luôn áp dụng các hoạt động tiền tệ vào hai tài khoản, trong đó số dư của một tài khoản tăng lên (có - credit) và tài khoản kia giảm xuống (nợ - debit):

| Tài khoản | Nợ (Debit) | Có (Credit) |
|-----------|------------|-------------|
| người mua | $1         |             |
| người bán |            | $1          |

Tổng của tất cả các bút toán giao dịch luôn bằng không. Cơ chế này cung cấp khả năng truy xuất nguồn gốc từ đầu đến cuối của tất cả các chuyển động tiền tệ trong hệ thống.

### **Trang thanh toán được lưu trữ (Hosted payment page)**
Để tránh lưu trữ thông tin thẻ tín dụng và phải tuân thủ các quy định khắt khe khác nhau, hầu hết các công ty thích sử dụng một widget do các PSP cung cấp, widget này sẽ lưu trữ và xử lý thanh toán thẻ tín dụng cho bạn:

<div style="margin-left:3rem">
    <img src="./images/hosted-payment-page.png" alt="hosted-payment-page" width="500" />
</div>

### **Luồng chi trả (Pay-out flow)**
Các thành phần của luồng chi trả rất giống với luồng nạp tiền.

Sự khác biệt chính:
 * tiền được chuyển từ tài khoản ngân hàng của trang thương mại điện tử sang tài khoản ngân hàng của người bán.
 * chúng ta có thể sử dụng nhà cung cấp dịch vụ chi trả bên thứ ba như Tipalti.
 * Có rất nhiều yêu cầu về sổ sách và quy định cần xử lý liên quan đến việc chi trả.

---

## Bước 3: Thiết kế chi tiết
Phần này tập trung vào việc làm cho hệ thống nhanh hơn, mạnh mẽ hơn và an toàn hơn.

### **Tích hợp PSP**
Nếu hệ thống của chúng ta có thể kết nối trực tiếp với các ngân hàng hoặc hệ thống thẻ, việc thanh toán có thể được thực hiện mà không cần PSP.
Những loại kết nối này rất hiếm và không phổ biến, thường chỉ được thực hiện tại các công ty lớn có thể chứng minh được hiệu quả đầu tư.

Nếu chúng ta đi theo con đường truyền thống, một PSP có thể được tích hợp theo một trong hai cách:
 * Thông qua API, nếu hệ thống thanh toán của chúng ta có thể thu thập thông tin thanh toán.
 * Thông qua một trang thanh toán được lưu trữ (hosted payment page) để tránh xử lý các quy định về thông tin thanh toán.

Dưới đây là cách hoạt động của luồng trang thanh toán được lưu trữ:

<div style="margin-left:3rem">
    <img src="./images/hosted-payment-page-workflow.png" alt="hosted-payment-page-workflow" width="500" />
</div>

 * Người dùng nhấp vào nút "thanh toán" trong trình duyệt.
 * Client gọi dịch vụ thanh toán với thông tin lệnh thanh toán.
 * Sau khi nhận được thông tin lệnh thanh toán, dịch vụ thanh toán gửi yêu cầu đăng ký thanh toán tới PSP.
 * PSP nhận thông tin thanh toán như loại tiền, số tiền, ngày hết hạn, v.v., cũng như một UUID cho mục đích idempotency. Thường là UUID của lệnh thanh toán.
 * PSP trả về một token xác định duy nhất việc đăng ký thanh toán. Token này được lưu trữ trong cơ sở dữ liệu dịch vụ thanh toán.
 * Sau khi token được lưu trữ, người dùng được cung cấp một trang thanh toán do PSP lưu trữ. Trang này được khởi tạo bằng token cũng như một URL chuyển hướng khi thành công/thất bại.
 * Người dùng điền chi tiết thanh toán trên trang của PSP, PSP xử lý thanh toán và trả về trạng thái thanh toán.
 * Người dùng hiện được chuyển hướng trở lại redirectURL. Ví dụ về redirect URL - `https://your-company.com/?tokenID=JIOUIQ123NSF&payResult=X324FSa`
 * Một cách không đồng bộ, PSP gọi dịch vụ thanh toán của chúng ta thông qua một webhook để thông báo cho backend của chúng ta về kết quả thanh toán.
 * Dịch vụ thanh toán ghi lại kết quả thanh toán dựa trên webhook nhận được.

### **Đối soát (Reconciliation)**
Phần trước giải thích luồng hoạt động bình thường (happy path) của một thanh toán. Các luồng không bình thường được phát hiện và đối soát bằng quy trình đối soát chạy ngầm.

Mỗi đêm, PSP gửi một tệp quyết toán (settlement file) mà hệ thống của chúng ta sử dụng để so sánh trạng thái của hệ thống bên ngoài với trạng thái hệ thống nội bộ của chúng ta.

<div style="margin-left:3rem">
    <img src="./images/settlement-report.png" alt="settlement-report" width="500" />
</div>

Quy trình này cũng có thể được sử dụng để phát hiện sự không nhất quán nội bộ giữa các dịch vụ như sổ cái và ví.

Các trường hợp không khớp được đội ngũ tài chính xử lý thủ công. Các trường hợp không khớp được xử lý như sau:
 * có thể phân loại, do đó, đây là một sự không khớp đã biết và có thể được điều chỉnh bằng một quy trình tiêu chuẩn.
 * có thể phân loại, nhưng không thể tự động hóa. Được điều chỉnh thủ công bởi đội ngũ tài chính.
 * không thể phân loại. Được điều tra và điều chỉnh thủ công bởi đội ngũ tài chính.

### **Xử lý sự chậm trễ trong xử lý thanh toán**
Có những trường hợp một thanh toán có thể mất hàng giờ để hoàn thành, mặc dù thông thường nó chỉ mất vài giây.

Điều này có thể xảy ra do:
 * một thanh toán bị gắn cờ là rủi ro cao và ai đó phải xem xét thủ công.
 * thẻ tín dụng yêu cầu bảo vệ bổ sung, ví dụ: Xác thực 3D Secure, yêu cầu thêm chi tiết từ chủ thẻ để hoàn tất.

Những tình huống này được xử lý bằng cách:
 * đợi PSP gửi cho chúng ta một webhook khi thanh toán hoàn tất hoặc thăm dò (polling) API của nó nếu PSP không cung cấp webhook.
 * hiển thị trạng thái "đang chờ xử lý" cho người dùng và cung cấp cho họ một trang nơi họ có thể kiểm tra các cập nhật thanh toán. Chúng ta cũng có thể gửi email cho họ khi quá trình thanh toán hoàn tất.

### **Giao tiếp giữa các dịch vụ nội bộ**
Có hai loại mô hình giao tiếp mà các dịch vụ sử dụng để giao tiếp với nhau - đồng bộ và không đồng bộ.

Giao tiếp đồng bộ (ví dụ: HTTP) hoạt động tốt cho các hệ thống quy mô nhỏ, nhưng gặp khó khăn khi quy mô tăng lên:
 * hiệu năng thấp - chu kỳ yêu cầu-phản hồi dài khi có nhiều dịch vụ tham gia vào chuỗi gọi.
 * cách ly lỗi kém - nếu PSP hoặc bất kỳ dịch vụ nào khác bị lỗi, người dùng sẽ không nhận được phản hồi.
 * khớp nối chặt chẽ - người gửi cần biết người nhận.
 * khó mở rộng - không dễ dàng hỗ trợ việc lưu lượng truy cập tăng đột ngột do không có bộ đệm.

Giao tiếp không đồng bộ có thể được chia thành hai loại.

Một người nhận (Single receiver) - nhiều người nhận đăng ký cùng một chủ đề (topic) và tin nhắn chỉ được xử lý một lần:

<div style="margin-left:3rem">
    <img src="./images/single-receiver.png" alt="single-receiver" width="500" />
</div>

Nhiều người nhận (Multiple receivers) - nhiều người nhận đăng ký cùng một chủ đề, nhưng tin nhắn được chuyển tiếp đến tất cả họ:

<div style="margin-left:3rem">
    <img src="./images/multiple-receiver.png" alt="multiple-receiver" width="500" />
</div>

Mô hình thứ hai hoạt động tốt cho hệ thống thanh toán của chúng ta vì một thanh toán có thể kích hoạt nhiều tác dụng phụ, được xử lý bởi các dịch vụ khác nhau.

Tóm lại, giao tiếp đồng bộ đơn giản hơn nhưng không cho phép các dịch vụ hoạt động tự chủ.
Giao tiếp không đồng bộ đánh đổi sự đơn giản và tính nhất quán để lấy khả năng mở rộng và khả năng phục hồi.

### **Xử lý thanh toán thất bại**
Mọi hệ thống thanh toán đều cần giải quyết các thanh toán thất bại. Dưới đây là một số cơ chế chúng ta sẽ sử dụng để đạt được điều đó:
 * Theo dõi trạng thái thanh toán - bất cứ khi nào thanh toán thất bại, chúng ta có thể xác định xem nên thử lại/hoàn tiền dựa trên trạng thái thanh toán.
 * Hàng đợi thử lại (Retry queue) - các thanh toán mà chúng ta sẽ thử lại được đẩy vào hàng đợi thử lại.
 * Hàng đợi thư chết (Dead-letter queue) - các thanh toán bị thất bại hoàn toàn được đẩy vào hàng đợi thư chết, nơi các thanh toán thất bại có thể được gỡ lỗi và kiểm tra.

<div style="margin-left:3rem">
    <img src="./images/failed-payments.png" alt="failed-payments" width="500" />
</div>

### **Truyền tải chính xác một lần (Exactly-once delivery)**
Chúng ta cần đảm bảo một thanh toán được xử lý chính xác một lần để tránh tính phí khách hàng hai lần.

Một hoạt động được thực thi chính xác một lần nếu nó được thực thi ít nhất một lần và đồng thời tối đa một lần.

Để đạt được sự đảm bảo ít nhất một lần, chúng ta sẽ sử dụng cơ chế thử lại (retry):

<div style="margin-left:3rem">
    <img src="./images/retry-mechanism.png" alt="retry-mechanism" width="500" />
</div>

Dưới đây là một số chiến lược phổ biến để quyết định khoảng thời gian thử lại:
 * thử lại ngay lập tức - client gửi ngay một yêu cầu khác sau khi thất bại.
 * khoảng thời gian cố định - đợi một khoảng thời gian cố định trước khi thử lại thanh toán.
 * khoảng thời gian tăng dần - tăng dần khoảng thời gian thử lại giữa mỗi lần thử lại.
 * lùi dần theo hàm mũ (exponential back-off) - gấp đôi khoảng thời gian thử lại giữa các lần thử lại tiếp theo.
 * hủy - client hủy yêu cầu. Điều này xảy ra khi lỗi là vĩnh viễn hoặc đã đạt đến ngưỡng thử lại.

Theo nguyên tắc chung, hãy để mặc định là chiến lược thử lại lùi dần theo hàm mũ. Một cách thực hành tốt là máy chủ chỉ định một khoảng thời gian thử lại bằng cách sử dụng tiêu đề `Retry-After`.

Một vấn đề với việc thử lại là máy chủ có khả năng xử lý một thanh toán hai lần:
 * người dùng nhấp vào "nút thanh toán" hai lần, do đó, họ bị tính phí hai lần.
 * thanh toán được PSP xử lý thành công, nhưng không được các dịch vụ hạ nguồn (sổ cái, ví) xử lý. Việc thử lại khiến thanh toán được PSP xử lý lại một lần nữa.

Để giải quyết vấn đề thanh toán gấp đôi, chúng ta cần sử dụng cơ chế idempotency - một thuộc tính mà một hoạt động được áp dụng nhiều lần chỉ được xử lý một lần.

Từ góc độ API, client có thể thực hiện nhiều lệnh gọi tạo ra cùng một kết quả.
Idempotency được quản lý bởi một tiêu đề đặc biệt trong yêu cầu (ví dụ: `idempotency-key`), thường là một UUID.

<div style="margin-left:3rem">
    <img src="./images/idempotency-example.png" alt="idempotency-example" width="500" />
</div>

Idempotency có thể đạt được bằng cách sử dụng cơ chế của cơ sở dữ liệu là thêm các ràng buộc khóa duy nhất:
 * máy chủ cố gắng chèn một hàng mới vào cơ sở dữ liệu.
 * việc chèn thất bại do vi phạm ràng buộc khóa duy nhất.
 * máy chủ phát hiện lỗi đó và thay vào đó trả về đối tượng hiện có cho client.

Idempotency cũng được áp dụng ở phía PSP, sử dụng nonce, đã được thảo luận trước đó. Các PSP sẽ lưu ý để không xử lý các khoản thanh toán có cùng nonce hai lần.

### **Tính nhất quán**
Có một số dịch vụ có trạng thái (stateful services) được gọi trong suốt vòng đời của một khoản thanh toán - PSP, sổ cái, ví, dịch vụ thanh toán.

Giao tiếp giữa bất kỳ hai dịch vụ nào đều có thể thất bại.
Chúng ta có thể đảm bảo tính nhất quán dữ liệu cuối cùng (eventual consistency) giữa tất cả các dịch vụ bằng cách triển khai xử lý chính xác một lần và đối soát.

Nếu chúng ta sử dụng nhân bản (replication), chúng ta sẽ phải đối mặt với độ trễ nhân bản, điều này có thể dẫn đến việc người dùng quan sát thấy dữ liệu không nhất quán giữa cơ sở dữ liệu chính và bản sao.

Để giảm thiểu điều đó, chúng ta có thể phục vụ tất cả các lần đọc và ghi từ cơ sở dữ liệu chính và chỉ sử dụng các bản sao cho mục đích dự phòng và chuyển đổi dự phòng.
Ngoài ra, chúng ta có thể đảm bảo các bản sao luôn đồng bộ bằng cách sử dụng thuật toán đồng thuận như Paxos hoặc Raft.
Chúng ta cũng có thể sử dụng cơ sở dữ liệu phân tán dựa trên sự đồng thuận như YugabyteDB hoặc CockroachDB.

### **Bảo mật thanh toán**
Dưới đây là một số cơ chế chúng ta có thể sử dụng để đảm bảo bảo mật thanh toán:
 * Nghe lén yêu cầu/phản hồi - chúng ta có thể sử dụng HTTPS để bảo mật mọi giao tiếp.
 * Làm xáo trộn dữ liệu (Data tampering) - thực thi mã hóa và giám sát tính toàn vẹn.
 * Tấn công Man-in-the-middle - sử dụng SSL với certificate pinning.
 * Mất dữ liệu - nhân bản dữ liệu qua nhiều vùng và sao chụp dữ liệu (data snapshots).
 * Tấn công DDoS - triển khai giới hạn tốc độ (rate limiting) và tường lửa.
 * Trộm thẻ - sử dụng token thay vì lưu trữ thông tin thẻ thực trong hệ thống của chúng ta.
 * Tuân thủ PCI - một tiêu chuẩn bảo mật cho các tổ chức xử lý thẻ tín dụng có thương hiệu.
 * Gian lận - xác minh địa chỉ, giá trị xác minh thẻ (CVV), phân tích hành vi người dùng, v.v.

---

## Bước 4: Tổng kết
Các điểm thảo luận khác:
 * Giám sát và cảnh báo.
 * Công cụ gỡ lỗi - chúng ta cần các công cụ giúp dễ dàng hiểu lý do tại sao một thanh toán thất bại.
 * Đổi ngoại tệ - quan trọng khi thiết kế hệ thống thanh toán cho mục đích quốc tế.
 * Địa lý - các khu vực khác nhau có thể có các phương thức thanh toán khác nhau.
 * Thanh toán bằng tiền mặt - rất phổ biến ở những nơi như Ấn Độ và Brazil.
 * Tích hợp Google/Apple Pay.
