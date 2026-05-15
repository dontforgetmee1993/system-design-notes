# Chương 28: Sàn giao dịch chứng khoán

## Giới thiệu
Trong chương này, chúng ta sẽ thiết kế một **sàn giao dịch chứng khoán điện tử**.

Chức năng cơ bản của nó là khớp lệnh giữa người mua và người bán một cách hiệu quả.

Các sàn giao dịch chứng khoán lớn bao gồm **NYSE**, **NASDAQ**, và nhiều sàn khác.

<div style="margin-left:3rem">
    <img src="./images/world-stock-exchanges.png" alt="world-stock-exchanges" width="500" />
</div>

---

## Bước 1: Hiểu vấn đề và thiết lập phạm vi thiết kế
 * C: Chúng ta sẽ giao dịch loại chứng khoán nào? Cổ phiếu, quyền chọn hay hợp đồng tương lai?
 * I: Chỉ cổ phiếu để đơn giản hóa.
 * C: Những loại lệnh nào được hỗ trợ - đặt, hủy, thay thế? Còn lệnh giới hạn (limit), lệnh thị trường (market), lệnh điều kiện thì sao?
 * I: Chúng ta cần hỗ trợ đặt và hủy lệnh. Đối với loại lệnh, chúng ta chỉ cần xem xét lệnh giới hạn (limit order).
 * C: Hệ thống có cần hỗ trợ giao dịch ngoài giờ không?
 * I: Không, chỉ trong giờ giao dịch bình thường.
 * C: Bạn có thể mô tả các chức năng cơ bản của sàn giao dịch không?
 * I: Khách hàng có thể đặt hoặc hủy lệnh giới hạn và nhận kết quả khớp lệnh theo thời gian thực. Họ nên xem được sổ lệnh (order book) theo thời gian thực.
 * C: Quy mô của sàn giao dịch là bao nhiêu?
 * I: Hàng chục nghìn người dùng giao dịch cùng lúc và khoảng 100 mã chứng khoán (symbols). Hàng tỷ lệnh mỗi ngày. Chúng ta cũng cần hỗ trợ kiểm tra rủi ro để tuân thủ quy định.
 * C: Loại kiểm tra rủi ro nào?
 * I: Hãy thực hiện các kiểm tra rủi ro đơn giản - ví dụ: giới hạn một người dùng chỉ được giao dịch tối đa 1 triệu cổ phiếu Apple trong một ngày.
 * C: Còn việc kết nối với ví của người dùng thì sao?
 * I: Chúng ta cần đảm bảo khách hàng có đủ tiền trước khi đặt lệnh. Số tiền dành cho các lệnh đang chờ xử lý cần được giữ lại cho đến khi lệnh được hoàn tất.

### **Yêu cầu phi chức năng**
Quy mô mà người phỏng vấn đề cập ám chỉ rằng chúng ta sẽ thiết kế một sàn giao dịch quy mô vừa và nhỏ.
Chúng ta cũng cần đảm bảo tính linh hoạt để hỗ trợ nhiều mã chứng khoán và người dùng hơn trong tương lai.

Các yêu cầu phi chức năng khác:
 * Độ sẵn sàng - Ít nhất 99.99%. Thời gian ngừng hoạt động có thể gây hại cho uy tín.
 * Khả năng chịu lỗi - Cần có khả năng chịu lỗi và cơ chế khôi phục nhanh để hạn chế tác động của sự cố sản xuất.
 * Độ trễ - Độ trễ vòng lặp (round-trip latency) nên ở mức mili giây (ms) với trọng tâm là phân vị thứ 99 (99th percentile). Độ trễ 99p cao kéo dài gây ra trải nghiệm tệ cho một nhóm người dùng.
 * Bảo mật - Chúng ta nên có hệ thống quản lý tài khoản. Để tuân thủ pháp luật, chúng ta cần hỗ trợ KYC để xác minh danh tính người dùng. Chúng ta cũng nên bảo vệ chống lại DDoS cho các tài nguyên công cộng.

### **Ước tính nhanh (Back-of-the-envelope estimation)**
 * 100 mã chứng khoán, 1 tỷ lệnh mỗi ngày.
 * Giờ giao dịch bình thường từ 09:30 đến 16:00 (6.5 giờ).
 * QPS trung bình = 1 tỷ / 6.5 / 3600 = 43,000.
 * QPS đỉnh = 5 * QPS trung bình = 215,000.
 * Khối lượng giao dịch thường cao hơn đáng kể khi thị trường mở cửa.

---

## Bước 2: Đề xuất thiết kế cấp cao và đạt được sự đồng thuận

### **Kiến thức nghiệp vụ cơ bản (Business Knowledge 101)**
Hãy thảo luận về một số khái niệm cơ bản liên quan đến sàn giao dịch.

Một nhà môi giới (broker) đóng vai trò trung gian giữa sàn giao dịch và người dùng cuối - ví dụ: Robinhood, Fidelity, v.v.

Các khách hàng tổ chức giao dịch với số lượng lớn bằng phần mềm chuyên dụng. Họ cần được đối xử đặc biệt.
Ví dụ: chia nhỏ lệnh khi giao dịch khối lượng lớn để tránh tác động đến thị trường.

Các loại lệnh:
 * Lệnh giới hạn (Limit) - mua hoặc bán ở một mức giá cố định. Nó có thể không khớp ngay lập tức hoặc có thể được khớp một phần.
 * Lệnh thị trường (Market) - không chỉ định giá. Được thực hiện ngay lập tức ở mức giá thị trường hiện tại.

Giá:
 * Bid (Giá mua) - mức giá cao nhất mà người mua sẵn sàng trả để mua một cổ phiếu.
 * Ask (Giá bán) - mức giá thấp nhất mà người bán sẵn sàng chấp nhận để bán một cổ phiếu.

Thị trường Hoa Kỳ có ba cấp độ báo giá - L1, L2, L3.

Dữ liệu thị trường L1 chứa giá mua/bán tốt nhất và khối lượng tương ứng:

<div style="margin-left:3rem">
    <img src="./images/l1-price.png" alt="l1-price" width="500" />
</div>

L2 bao gồm nhiều mức giá hơn:

<div style="margin-left:3rem">
    <img src="./images/l2-price.png" alt="l2-price" width="500" />
</div>

L3 hiển thị các mức giá và khối lượng đang chờ ở mỗi mức:

<div style="margin-left:3rem">
    <img src="./images/l3-price.png" alt="l3-price" width="500" />
</div>

Biểu đồ nến (candlestick) hiển thị giá mở cửa và đóng cửa, cũng như giá cao nhất và thấp nhất trong một khoảng thời gian nhất định:

<div style="margin-left:3rem">
    <img src="./images/candlestick.png" alt="candlestick" width="500" />
</div>

FIX là một giao thức trao đổi thông tin giao dịch chứng khoán được hầu hết các nhà cung cấp sử dụng. Ví dụ về một giao dịch chứng khoán:
```
8=FIX.4.2 | 9=176 | 35=8 | 49=PHLX | 56=PERS | 52=20071123-05:30:00.000 | 11=ATOMNOCCC9990900 | 20=3 | 150=E | 39=E | 55=MSFT | 167=CS | 54=1 | 38=15 | 40=2 | 44=15 | 58=PHLX EQUITY TESTING | 59=0 | 47=C | 32=0 | 31=0 | 151=15 | 14=0 | 6=0 | 10=128 |
```

### **Thiết kế cấp cao**

<div style="margin-left:3rem">
    <img src="./images/high-level-design.png" alt="high-level-design" width="500" />
</div>

Luồng giao dịch (Trade flow):
 * Khách hàng đặt lệnh qua giao diện giao dịch.
 * Nhà môi giới gửi lệnh đến sàn giao dịch.
 * Lệnh vào sàn qua cổng khách hàng (client gateway), nơi thực hiện xác thực, giới hạn tốc độ, kiểm tra quyền, v.v. Lệnh được chuyển tiếp đến bộ quản lý lệnh (order manager).
 * Bộ quản lý lệnh thực hiện kiểm tra rủi ro dựa trên các quy tắc do bộ quản lý rủi ro thiết lập.
 * Sau khi vượt qua kiểm tra rủi ro, bộ quản lý lệnh xác minh xem có đủ tiền trong ví cho lệnh đó hay không.
 * Lệnh được gửi đến bộ máy khớp lệnh (matching engine). Khi tìm thấy kết quả khớp, bộ máy khớp lệnh sẽ phát ra hai kết quả thực hiện (gọi là fills) cho bên mua và bên bán. Cả hai lệnh đều được đánh số thứ tự để đảm bảo tính xác định.
 * Kết quả thực hiện được trả về cho khách hàng.

Luồng dữ liệu thị trường (Market data flow - M1-M3):
 * bộ máy khớp lệnh tạo ra một luồng các kết quả thực hiện, gửi đến bộ xuất bản dữ liệu thị trường (market data publisher).
 * Bộ xuất bản dữ liệu thị trường xây dựng các biểu đồ nến và gửi chúng đến dịch vụ dữ liệu.
 * Dữ liệu thị trường được lưu trữ trong bộ lưu trữ chuyên dụng để phân tích thời gian thực. Các nhà môi giới kết nối với dịch vụ dữ liệu để nhận dữ liệu thị trường kịp thời.

Luồng báo cáo (Reporter flow - R1-R2):
 * bộ báo cáo thu thập tất cả các trường báo cáo cần thiết từ các lệnh và kết quả thực hiện và ghi chúng vào DB.
 * các trường báo cáo - client_id, giá, khối lượng, loại lệnh, khối lượng đã khớp, khối lượng còn lại.

Luồng giao dịch nằm trên con đường quan trọng (critical path), trong khi các luồng còn lại thì không, do đó các yêu cầu về độ trễ giữa chúng là khác nhau.

#### Luồng giao dịch
Luồng giao dịch nằm trên con đường quan trọng, do đó nó nên được tối ưu hóa cao để có độ trễ thấp.

Bộ máy khớp lệnh (matching engine) là trái tim của sàn giao dịch, còn được gọi là cross engine. Các trách nhiệm chính:
 * Duy trì sổ lệnh (order book) cho mỗi mã chứng khoán - danh sách các lệnh mua/bán cho một mã.
 * Khớp các lệnh mua và bán - một kết quả khớp tạo ra hai kết quả thực hiện (fills), mỗi kết quả cho bên mua và bên bán. Chức năng này phải nhanh và chính xác.
 * Phân phối luồng thực hiện dưới dạng dữ liệu thị trường.
 * Các kết quả khớp lệnh phải được tạo ra theo một thứ tự xác định. Đây là nền tảng cho tính sẵn sàng cao.

Tiếp theo là bộ đánh số thứ tự (sequencer) - đây là thành phần chính làm cho bộ máy khớp lệnh mang tính xác định bằng cách đánh dấu mỗi lệnh đến và mỗi kết quả thực hiện đi bằng một ID trình tự (sequence ID).

<div style="margin-left:3rem">
    <img src="./images/sequencer.png" alt="sequencer" width="500" />
</div>

Chúng ta đánh dấu các lệnh đến và các kết quả đi vì một số lý do:
 * tính kịp thời và công bằng.
 * khôi phục/phát lại (replay) nhanh chóng.
 * đảm bảo chính xác một lần (exactly-once).

Về mặt khái niệm, chúng ta có thể sử dụng Kafka làm sequencer vì nó thực chất là một hàng đợi tin nhắn đến và đi. Tuy nhiên, chúng ta sẽ tự triển khai nó để đạt được độ trễ thấp hơn.

Bộ quản lý lệnh (order manager) quản lý trạng thái của các lệnh. Nó cũng tương tác với bộ máy khớp lệnh - gửi lệnh và nhận các kết quả khớp (fills).

Trách nhiệm của bộ quản lý lệnh:
 * Gửi lệnh để kiểm tra rủi ro - ví dụ: xác minh khối lượng giao dịch của người dùng dưới 1 triệu.
 * Kiểm tra lệnh đối với ví người dùng và xác minh xem có đủ tiền để thực hiện lệnh đó hay không.
 * Gửi lệnh đến sequencer và sau đó đến matching engine. Để giảm băng thông, chỉ thông tin lệnh cần thiết mới được chuyển đến matching engine.
 * Các kết quả thực hiện (fills) được nhận lại từ sequencer, sau đó chúng được gửi đến các nhà môi giới thông qua client gateway.

Thách thức chính khi triển khai bộ quản lý lệnh là quản lý chuyển đổi trạng thái. Event sourcing là một giải pháp khả thi (sẽ thảo luận trong phần thiết kế chi tiết).

Cuối cùng, client gateway nhận các lệnh từ người dùng và gửi chúng đến bộ quản lý lệnh. Trách nhiệm của nó:

<div style="margin-left:3rem">
    <img src="./images/client-gateway.png" alt="client-gateway" width="500" />
</div>

Vì client gateway nằm trên con đường quan trọng, nó nên được giữ ở mức nhẹ (lightweight).

Có thể có nhiều client gateway cho các khách hàng khác nhau. Ví dụ: một colo engine là một máy chủ công cụ giao dịch, được nhà môi giới thuê trong trung tâm dữ liệu của sàn giao dịch.

<div style="margin-left:3rem">
    <img src="./images/client-gateways.png" alt="client-gateways" width="500" />
</div>

#### Luồng dữ liệu thị trường
Bộ xuất bản dữ liệu thị trường nhận các kết quả thực hiện từ bộ máy khớp lệnh và xây dựng sổ lệnh/biểu đồ nến từ luồng thực hiện đó.

Dữ liệu đó được gửi đến dịch vụ dữ liệu, dịch vụ này chịu trách nhiệm hiển thị dữ liệu tổng hợp cho những người đăng ký:

<div style="margin-left:3rem">
    <img src="./images/market-data.png" alt="market-data" width="500" />
</div>

#### Luồng báo cáo
Bộ báo cáo không nằm trên con đường quan trọng, nhưng nó vẫn là một thành phần quan trọng.

<div style="margin-left:3rem">
    <img src="./images/reporting-flow.png" alt="reporting-flow" width="500" />
</div>

Nó chịu trách nhiệm về lịch sử giao dịch, báo cáo thuế, báo cáo tuân thủ, quyết toán, v.v.
Độ trễ không phải là yêu cầu quan trọng đối với luồng báo cáo. Tính chính xác và tuân thủ quan trọng hơn.

### **Thiết kế API**
Khách hàng tương tác với sàn chứng khoán thông qua các nhà môi giới để đặt lệnh, xem kết quả thực hiện, dữ liệu thị trường, tải xuống dữ liệu lịch sử để phân tích, v.v.

Chúng ta sử dụng RESTful API để giao tiếp giữa client gateway và các nhà môi giới.

Đối với khách hàng tổ chức, một giao thức độc quyền (proprietary protocol) được sử dụng để đáp ứng các yêu cầu về độ trễ thấp của họ.

Tạo lệnh:
```
POST /v1/order
```

Các tham số:
 * symbol - mã chứng khoán. Kiểu String.
 * side - mua hoặc bán. Kiểu String.
 * price - giá của lệnh giới hạn. Kiểu Long.
 * orderType - giới hạn hoặc thị trường (chúng ta chỉ hỗ trợ lệnh giới hạn trong thiết kế này). Kiểu String.
 * quantity - khối lượng của lệnh. Kiểu Long.

Phản hồi:
 * id - ID của lệnh. Kiểu Long.
 * creationTime - thời gian tạo lệnh trong hệ thống. Kiểu Long.
 * filledQuantity - khối lượng đã khớp thành công. Kiểu Long.
 * remainingQuantity - khối lượng còn lại chờ khớp. Kiểu Long.
 * status - mới/đã hủy/đã khớp. Kiểu String.
 * các thuộc tính còn lại giống như các tham số đầu vào.

Lấy kết quả thực hiện:
```
GET /execution?symbol={:symbol}&orderId={:orderId}&startTime={:startTime}&endTime={:endTime}
```

Các tham số:
 * symbol - mã chứng khoán. Kiểu String.
 * orderId - ID của lệnh. Tùy chọn. Kiểu String.
 * startTime - thời gian bắt đầu truy vấn tính theo epoch. Kiểu Long.
 * endTime - thời gian kết thúc truy vấn tính theo epoch. Kiểu Long.

Phản hồi:
 * executions - mảng chứa mỗi kết quả thực hiện trong phạm vi (xem các thuộc tính bên dưới). Kiểu Array.
 * id - ID của kết quả thực hiện. Kiểu Long.
 * orderId - ID của lệnh. Kiểu Long.
 * symbol - mã chứng khoán. Kiểu String.
 * side - mua hoặc bán. Kiểu String.
 * price - giá thực hiện. Kiểu Long.
 * orderType - giới hạn hoặc thị trường. Kiểu String.
 * quantity - khối lượng đã khớp. Kiểu Long.

Lấy sổ lệnh (order book):
```
GET /marketdata/orderBook/L2?symbol={:symbol}&depth={:depth}
```

Các tham số:
 * symbol - mã chứng khoán. Kiểu String.
 * depth - độ sâu của sổ lệnh cho mỗi bên. Kiểu Int.

Phản hồi:
 * bids - mảng chứa giá và khối lượng mua. Kiểu Array.
 * asks - mảng chứa giá và khối lượng bán. Kiểu Array.

Lấy biểu đồ nến:
```
GET /marketdata/candles?symbol={:symbol}&resolution={:resolution}&startTime={:startTime}&endTime={:endTime}
```

Các tham số:
 * symbol - mã chứng khoán. Kiểu String.
 * resolution - độ dài cửa sổ thời gian của biểu đồ nến tính bằng giây. Kiểu Long.
 * startTime - thời gian bắt đầu của cửa sổ tính theo epoch. Kiểu Long.
 * endTime - thời gian kết thúc của cửa sổ tính theo epoch. Kiểu Long.

Phản hồi:
 * candles - mảng chứa dữ liệu mỗi cây nến (các thuộc tính liệt kê bên dưới). Kiểu Array.
 * open - giá mở cửa của mỗi cây nến. Kiểu Double.
 * close - giá đóng cửa của mỗi cây nến. Kiểu Double.
 * high - giá cao nhất của mỗi cây nến. Kiểu Double.
 * low - giá thấp nhất của mỗi cây nến. Kiểu Double.

### **Mô hình dữ liệu**
Có ba loại dữ liệu chính trong sàn giao dịch của chúng ta:
 * Sản phẩm, lệnh, kết quả thực hiện.
 * Sổ lệnh (order book).
 * Biểu đồ nến.

#### Sản phẩm, lệnh, kết quả thực hiện
Sản phẩm mô tả các thuộc tính của một mã được giao dịch - loại sản phẩm, mã giao dịch, mã hiển thị giao diện người dùng, v.v.

Dữ liệu này không thay đổi thường xuyên, nó chủ yếu được sử dụng để hiển thị trên giao diện người dùng.

Một lệnh đại diện cho một chỉ dẫn mua/bán. Các kết quả thực hiện là kết quả khớp lệnh được đưa ra.

Dưới đây là mô hình dữ liệu:

<div style="margin-left:3rem">
    <img src="./images/product-order-execution-data-model.png" alt="product-order-execution-data-model" width="500" />
</div>

Chúng ta gặp các lệnh và kết quả thực hiện trong cả ba luồng:
 * trong con đường quan trọng, chúng được xử lý trong bộ nhớ để đạt hiệu năng cao. Chúng được lưu trữ và khôi phục từ sequencer.
 * Bộ báo cáo ghi các lệnh và kết quả thực hiện vào cơ sở dữ liệu cho các mục đích báo cáo.
 * Các kết quả thực hiện được chuyển tiếp đến bộ phận dữ liệu thị trường để tái cấu trúc sổ lệnh và biểu đồ nến.

#### Sổ lệnh (Order book)
Sổ lệnh là danh sách các lệnh mua/bán cho một công cụ tài chính, được sắp xếp theo mức giá.

Một cấu trúc dữ liệu hiệu quả cho mô hình này cần thỏa mãn:
 * thời gian tra cứu không đổi - lấy khối lượng tại một mức giá hoặc giữa các mức giá.
 * các hoạt động thêm/thực thi/hủy lệnh nhanh chóng.
 * truy vấn giá mua/bán tốt nhất.
 * duyệt qua các mức giá.

Ví dụ về thực thi sổ lệnh:

<div style="margin-left:3rem">
    <img src="./images/order-book-execution.png" alt="order-book-execution" width="500" />
</div>

Sau khi thực hiện xong lệnh lớn này, giá tăng lên khi khoảng cách giá mua/bán (spread) rộng ra.

Ví dụ triển khai sổ lệnh bằng mã giả:
```
class PriceLevel{
    private Price limitPrice;
    private long totalVolume;
    private List<Order> orders;
}

class Book<Side> {
    private Side side;
    private Map<Price, PriceLevel> limitMap;
}

class OrderBook {
    private Book<Buy> buyBook;
    private Book<Sell> sellBook;
    private PriceLevel bestBid;
    private PriceLevel bestOffer;
    private Map<OrderID, Order> orderMap;
}
```

Để triển khai hiệu quả hơn, chúng ta có thể sử dụng danh sách liên kết đôi (doubly-linked list) thay vì danh sách tiêu chuẩn:
 * Đặt một lệnh mới là O(1), vì chúng ta thêm một lệnh vào cuối danh sách.
 * Khớp một lệnh là O(1), vì chúng ta xóa một lệnh ở đầu danh sách.
 * Hủy một lệnh có nghĩa là xóa một lệnh khỏi sổ lệnh. Chúng ta sử dụng `orderMap` để tra cứu O(1) và xóa O(1) (do `Order` có tham chiếu đến phần tử trước đó trong danh sách).

<div style="margin-left:3rem">
    <img src="./images/order-book-impl.png" alt="order-book-impl" width="500" />
</div>

Cấu trúc dữ liệu này cũng được sử dụng trong các dịch vụ dữ liệu thị trường để tái cấu trúc sổ lệnh.

#### Biểu đồ nến
Dữ liệu nến được tính toán trong các dịch vụ dữ liệu thị trường dựa trên việc xử lý các lệnh trong một khoảng thời gian:
```
class Candlestick {
    private long openPrice;
    private long closePrice;
    private long highPrice;
    private long lowPrice;
    private long volume;
    private long timestamp;
    private int interval;
}

class CandlestickChart {
    private LinkedList<Candlestick> sticks;
}
```

Một số tối ưu hóa để tránh tiêu thụ quá nhiều bộ nhớ:
 * Sử dụng các bộ đệm vòng (ring buffers) được phân bổ trước để giữ các nến nhằm giảm số lượng phân bổ bộ nhớ.
 * Giới hạn số lượng nến trong bộ nhớ và lưu phần còn lại vào ổ đĩa.

Chúng ta sẽ sử dụng cơ sở dữ liệu hướng cột trong bộ nhớ (ví dụ: KDB) để phân tích thời gian thực. Sau khi thị trường đóng cửa, dữ liệu được lưu vào cơ sở dữ liệu lịch sử.

---

## Bước 3: Thiết kế chi tiết
Một điều thú vị cần lưu ý về các sàn giao dịch hiện đại là không giống như hầu hết các phần mềm khác, chúng thường chạy mọi thứ trên một máy chủ khổng lồ duy nhất.

Hãy cùng khám phá các chi tiết.

### **Hiệu năng**
Đối với một sàn giao dịch, việc có độ trễ tổng thể tốt cho tất cả các phân vị là rất quan trọng.

Làm thế nào chúng ta có thể giảm độ trễ?
 * Giảm số lượng tác vụ trên con đường quan trọng (critical path).
 * Rút ngắn thời gian dành cho mỗi tác vụ bằng cách giảm sử dụng mạng/ổ đĩa và/hoặc giảm thời gian thực thi tác vụ.

Để đạt được mục tiêu đầu tiên, chúng ta đã tước bỏ mọi trách nhiệm không liên quan khỏi con đường quan trọng, ngay cả việc ghi nhật ký (logging) cũng được loại bỏ để đạt được độ trễ tối ưu.

 Nếu chúng ta tuân theo thiết kế ban đầu, có một số điểm nghẽn - độ trễ mạng giữa các dịch vụ và việc sử dụng đĩa của sequencer.

Với thiết kế như vậy, chúng ta có thể đạt được độ trễ đầu cuối ở mức hàng chục mili giây. Chúng ta muốn đạt được hàng chục micro giây.

Do đó, chúng ta sẽ đặt mọi thứ trên một máy chủ duy nhất và các tiến trình sẽ giao tiếp thông qua mmap như một kho lưu trữ sự kiện (event store):

<div style="margin-left:3rem">
    <img src="./images/mmap-bus.png" alt="mmap-bus" width="500" />
</div>

Một tối ưu hóa khác là sử dụng vòng lặp ứng dụng (application loop - một vòng lặp while thực thi các tác vụ quan trọng), được gắn (pinned) vào cùng một CPU để tránh chuyển đổi ngữ cảnh (context switching).

<div style="margin-left:3rem">
    <img src="./images/application-loop.png" alt="application-loop" width="500" />
</div>

Một tác dụng phụ khác của việc sử dụng vòng lặp ứng dụng là không có sự tranh chấp khóa (lock contention) - khi nhiều luồng chiến đấu cho cùng một tài nguyên.

Bây giờ hãy khám phá cách mmap hoạt động - nó là một syscall của UNIX, ánh xạ một tệp trên đĩa vào bộ nhớ của ứng dụng.

Một mẹo chúng ta có thể sử dụng là tạo tệp trong `/dev/shm`, viết tắt của "shared memory" (bộ nhớ dùng chung). Do đó, chúng ta hoàn toàn không truy cập vào đĩa.

### **Event sourcing (Nguồn sự kiện)**
Event sourcing được thảo luận sâu trong [chương ví điện tử](../chapter27). Tham khảo chương đó để biết mọi chi tiết.

Tóm lại, thay vì lưu trữ các trạng thái hiện tại, chúng ta lưu trữ các bước chuyển đổi trạng thái bất biến:

<div style="margin-left:3rem">
    <img src="./images/event-sourcing.png" alt="event-sourcing" width="500" />
</div>

 * Bên trái - lược đồ truyền thống.
 * Bên phải - lược đồ event sourcing.

Dưới đây là thiết kế của chúng ta cho đến nay:

<div style="margin-left:3rem">
    <img src="./images/design-so-far.png" alt="design-so-far" width="500" />
</div>

 * miền bên ngoài tương tác với client gateway của chúng ta bằng giao thức FIX.
 * Bộ quản lý lệnh nhận sự kiện lệnh mới, xác thực nó và thêm nó vào trạng thái nội bộ. Lệnh sau đó được gửi đến matching core.
 * Nếu lệnh được khớp, `OrderFilledEvent` được tạo ra và gửi qua mmap.
 * Các thành phần khác đăng ký vào kho lưu trữ sự kiện và thực hiện phần xử lý của mình.

Một tối ưu hóa bổ sung - tất cả các thành phần đều giữ một bản sao của bộ quản lý lệnh, được đóng gói dưới dạng một thư viện để tránh các lệnh gọi bổ sung nhằm quản lý lệnh.

Sequencer trong thiết kế này thay đổi để không phải là một kho lưu trữ sự kiện, mà là một trình ghi duy nhất (single writer), sắp xếp thứ tự các sự kiện trước khi chuyển tiếp chúng đến kho lưu trữ sự kiện:

<div style="margin-left:3rem">
    <img src="./images/sequencer-deep-dive.png" alt="sequencer-deep-dive" width="500" />
</div>

### **Độ sẵn sàng cao**
Chúng ta hướng tới độ sẵn sàng 99.99% - chỉ 8.64 giây ngừng hoạt động mỗi ngày.

Để đạt được điều đó, chúng ta phải xác định các điểm lỗi duy nhất (single-point-of-failures) trong kiến trúc sàn giao dịch:
 * thiết lập các phiên bản dự phòng cho các dịch vụ quan trọng (ví dụ: matching engine) đang ở trạng thái chờ (stand-by).
 * tự động hóa mạnh mẽ việc phát hiện lỗi và chuyển đổi dự phòng (failover) sang phiên bản dự phòng.

Các dịch vụ không trạng thái như client gateway có thể dễ dàng mở rộng theo chiều ngang bằng cách thêm nhiều máy chủ.

Đối với các thành phần có trạng thái, chúng ta có thể xử lý các sự kiện đến, nhưng không xuất bản các sự kiện đi nếu chúng ta không phải là leader:

<div style="margin-left:3rem">
    <img src="./images/leader-election.png" alt="leader-election" width="500" />
</div>

Để phát hiện bản sao chính bị hỏng, chúng ta có thể gửi các nhịp tim (heartbeats) để nhận biết nó không còn hoạt động.

Cơ chế này chỉ hoạt động trong phạm vi của một máy chủ duy nhất.
Nếu chúng ta muốn mở rộng nó, chúng ta có thể thiết lập toàn bộ máy chủ dưới dạng bản sao nóng/ấm (hot/warm replica) và chuyển đổi dự phòng trong trường hợp xảy ra lỗi.

Để nhân bản kho lưu trữ sự kiện trên các bản sao, chúng ta có thể sử dụng UDP tin cậy để giao tiếp nhanh hơn.

### **Khả năng chịu lỗi**
Điều gì xảy ra nếu ngay cả các phiên bản dự phòng cũng bị hỏng? Đây là một sự kiện có xác suất thấp nhưng chúng ta nên sẵn sàng cho nó.

Các công ty công nghệ lớn giải quyết vấn đề này bằng cách nhân bản dữ liệu cốt lõi đến các trung tâm dữ liệu ở nhiều thành phố để giảm thiểu tác động của ví dụ như thiên tai.

Các câu hỏi cần xem xét:
 * Nếu phiên bản chính bị hỏng, khi nào và làm thế nào để chúng ta chuyển đổi dự phòng sang phiên bản dự phòng?
 * Làm thế nào để chúng ta chọn leader trong số các phiên bản dự phòng?
 * Thời gian khôi phục cần thiết (RTO - recovery time objective) là bao nhiêu?
 * Những chức năng nào cần được khôi phục? Hệ thống của chúng ta có thể hoạt động trong điều kiện bị suy giảm (degraded conditions) không?

Cách giải quyết:
 * Hệ thống có thể bị hỏng do lỗi mã nguồn (ảnh hưởng đến cả bản chính và bản sao), chúng ta có thể sử dụng chaos engineering để phát hiện các trường hợp biên và kết quả thảm khốc như thế này.
 * Ban đầu, chúng ta có thể thực hiện chuyển đổi dự phòng thủ công cho đến khi thu thập đủ kiến thức về các chế độ lỗi của hệ thống.
 * bầu chọn leader có thể được sử dụng (ví dụ: Raft) để xác định bản sao nào trở thành leader trong trường hợp bản chính bị hỏng.

Ví dụ về cách nhân bản hoạt động trên các máy chủ khác nhau:

<div style="margin-left:3rem">
    <img src="./images/replication-across-servers.png" alt="replication-across-servers" width="500" />
</div>

Ví dụ về các nhiệm kỳ bầu chọn leader:

<div style="margin-left:3rem">
    <img src="./images/leader-election-terms.png" alt="leader-election-terms" width="500" />
</div>

Để biết chi tiết về cách hoạt động của Raft, [hãy xem tại đây](https://thesecretlivesofdata.com/raft/).

Cuối cùng, chúng ta cũng cần xem xét khả năng chịu mất mát (loss tolerance) - chúng ta có thể mất bao nhiêu dữ liệu trước khi mọi thứ trở nên quan trọng?
Điều này sẽ xác định tần suất chúng ta sao lưu dữ liệu.

Đối với một sàn giao dịch chứng khoán, mất mát dữ liệu là không thể chấp nhận được, vì vậy chúng ta phải sao lưu dữ liệu thường xuyên và dựa vào sự nhân bản của Raft để giảm xác suất mất dữ liệu.

### **Thuật toán khớp lệnh**
Một chút lạc đề về cách hoạt động của việc khớp lệnh qua mã giả:
```
Context handleOrder(OrderBook orderBook, OrderEvent orderEvent) {
    if (orderEvent.getSequenceId() != nextSequence) {
        return Error(OUT_OF_ORDER, nextSequence);
    }

    if (!validateOrder(symbol, price, quantity)) {
        return ERROR(INVALID_ORDER, orderEvent);
    }

    Order order = createOrderFromEvent(orderEvent);
    switch (msgType):
        case NEW:
            return handleNew(orderBook, order);
        case CANCEL:
            return handleCancel(orderBook, order);
        default:
            return ERROR(INVALID_MSG_TYPE, msgType);

}

Context handleNew(OrderBook orderBook, Order order) {
    if (BUY.equals(order.side)) {
        return match(orderBook.sellBook, order);
    } else {
        return match(orderBook.buyBook, order);
    }
}

Context handleCancel(OrderBook orderBook, Order order) {
    if (!orderBook.orderMap.contains(order.orderId)) {
        return ERROR(CANNOT_CANCEL_ALREADY_MATCHED, order);
    }

    removeOrder(order);
    setOrderStatus(order, CANCELED);
    return SUCCESS(CANCEL_SUCCESS, order);
}

Context match(OrderBook book, Order order) {
    Quantity leavesQuantity = order.quantity - order.matchedQuantity;
    Iterator<Order> limitIter = book.limitMap.get(order.price).orders;
    while (limitIter.hasNext() && leavesQuantity > 0) {
        Quantity matched = min(limitIter.next.quantity, order.quantity);
        order.matchedQuantity += matched;
        leavesQuantity = order.quantity - order.matchedQuantity;
        remove(limitIter.next);
        generateMatchedFill();
    }
    return SUCCESS(MATCH_SUCCESS, order);
}
```

Thuật toán khớp lệnh này sử dụng thuật toán FIFO để xác định lệnh nào tại một mức giá sẽ được khớp trước.

### **Tính xác định (Determinism)**
Tính xác định về chức năng được đảm bảo thông qua kỹ thuật sequencer mà chúng ta đã sử dụng.

Thời điểm thực tế khi sự kiện xảy ra không quan trọng:

<div style="margin-left:3rem">
    <img src="./images/determinism.png" alt="determinism" width="500" />
</div>

Tính xác định về độ trễ là điều chúng ta phải theo dõi. Chúng ta có thể tính toán nó dựa trên việc giám sát độ trễ ở phân vị thứ 99 hoặc 99.99.

Những thứ có thể gây ra hiện tượng tăng vọt độ trễ là các sự kiện thu gom rác (garbage collector) trong các ngôn ngữ như Java.

### **Tối ưu hóa bộ xuất bản dữ liệu thị trường**
Bộ xuất bản dữ liệu thị trường nhận các kết quả khớp từ bộ máy khớp lệnh và xây dựng lại sổ lệnh và biểu đồ nến dựa trên chúng.

Chúng ta chỉ giữ một phần dữ liệu nến vì chúng ta không có bộ nhớ vô hạn. Khách hàng có thể chọn mức độ chi tiết của thông tin họ muốn. Thông tin chi tiết hơn có thể yêu cầu mức giá cao hơn:

<div style="margin-left:3rem">
    <img src="./images/market-data-publisher.png" alt="market-data-publisher" width="500" />
</div>

Một bộ đệm vòng (ring buffer hay circular buffer) là một hàng đợi có kích thước cố định với đầu kết nối với cuối. Không gian được phân bổ trước để tránh việc phân bổ động. Cấu trúc dữ liệu này cũng không sử dụng khóa (lock-free).

Một kỹ thuật khác để tối ưu hóa bộ đệm vòng là đệm (padding), đảm bảo số thứ tự không bao giờ nằm trong cùng một dòng cache (cache line) với bất kỳ thứ gì khác.

### **Sự công bằng trong phân phối dữ liệu thị trường và Multicast**
Chúng ta cần đảm bảo những người đăng ký nhận được dữ liệu cùng một lúc vì nếu một người nhận được dữ liệu trước người khác, điều đó mang lại cho họ cái nhìn sâu sắc quan trọng về thị trường, họ có thể sử dụng thông tin đó để thao túng thị trường.

Để đạt được điều này, chúng ta có thể sử dụng multicast bằng cách dùng UDP tin cậy khi xuất bản dữ liệu cho những người đăng ký.

Dữ liệu có thể được vận chuyển qua internet theo ba cách:
 * Unicast - một nguồn, một đích.
 * Broadcast - một nguồn đến toàn bộ mạng con.
 * Multicast - một nguồn đến một tập hợp các máy chủ trên các mạng con khác nhau.

Về lý thuyết, bằng cách sử dụng multicast, tất cả những người đăng ký sẽ nhận được dữ liệu cùng một lúc.

Tuy nhiên, UDP không đáng tin cậy và dữ liệu có thể không đến được với tất cả mọi người. Mặc dù vậy, nó có thể được cải thiện bằng cách truyền lại.

### **Colocation (Đặt máy chủ cùng vị trí)**
Các sàn giao dịch cung cấp cho các nhà môi giới khả năng đặt máy chủ của họ trong cùng một trung tâm dữ liệu với sàn giao dịch.

Điều này giúp giảm độ trễ một cách đáng kể và có thể được coi là một dịch vụ VIP.

### **Bảo mật mạng**
DDoS là một thách thức đối với các sàn giao dịch vì có một số dịch vụ hướng ra internet. Dưới đây là các lựa chọn của chúng ta:
 * Cách ly các dịch vụ và dữ liệu công cộng khỏi các dịch vụ riêng tư, để các cuộc tấn công DDoS không ảnh hưởng đến các khách hàng quan trọng nhất.
 * Sử dụng một lớp lưu đệm (caching) để lưu trữ dữ liệu không thường xuyên thay đổi.
 * Tăng cường bảo mật cho URL chống lại DDoS, ví dụ: ưu tiên `https://my.website.com/data/recent` thay vì `https://my.website.com/data?from=123&to=456`, vì URL trước có khả năng lưu đệm tốt hơn.
 * Cần có cơ chế danh sách cho phép (allowlist)/danh sách chặn (blocklist) hiệu quả.
 * Giới hạn tốc độ (Rate limiting) có thể được sử dụng để giảm thiểu DDoS.

---

## Bước 4: Tổng kết
Các lưu ý thú vị khác:
 * không phải tất cả các sàn giao dịch đều dựa trên việc đặt mọi thứ trên một máy chủ lớn, nhưng một số vẫn làm vậy.
 * các sàn giao dịch hiện đại dựa nhiều hơn vào cơ sở hạ tầng đám mây và cũng dựa vào các nhà tạo lập thị trường tự động (AMM) để tránh việc duy trì sổ lệnh.
