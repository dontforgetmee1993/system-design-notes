# Chương 19: Hàng đợi tin nhắn phân tán (Distributed Message Queue)

## Giới thiệu

Chúng ta sẽ thiết kế một **hàng đợi tin nhắn phân tán** trong chương này.

Lợi ích của hàng đợi tin nhắn:
- **Tách biệt (Decoupling)**: Loại bỏ sự ràng buộc chặt chẽ giữa các thành phần. Cho phép chúng cập nhật độc lập.
- **Cải thiện khả năng mở rộng**: Người sản xuất (producer) và người tiêu dùng (consumer) có thể được mở rộng độc lập dựa trên lưu lượng truy cập.
- **Tăng tính khả dụng**: Nếu một phần của hệ thống bị lỗi, các phần khác vẫn tiếp tục tương tác với hàng đợi.
- **Hiệu suất tốt hơn**: Người sản xuất có thể gửi tin nhắn mà không cần đợi xác nhận từ người tiêu dùng.

Một số triển khai hàng đợi tin nhắn phổ biến - Kafka, RabbitMQ, RocketMQ, Apache Pulsar, ActiveMQ, ZeroMQ.

Nói một cách chính xác, Kafka và Pulsar không phải là hàng đợi tin nhắn. Chúng là các nền tảng truyền phát sự kiện (event streaming platforms). Tuy nhiên, có một sự hội tụ các tính năng làm mờ đi ranh giới giữa hàng đợi tin nhắn và nền tảng truyền phát sự kiện.

Trong chương này, chúng ta sẽ xây dựng một hàng đợi tin nhắn hỗ trợ các tính năng nâng cao như lưu trữ dữ liệu lâu dài, tiêu thụ tin nhắn lặp lại, v.v.

---

## Bước 1: Hiểu vấn đề và xác định phạm vi thiết kế

Hàng đợi tin nhắn cần hỗ trợ một vài tính năng cơ bản - người sản xuất tạo ra tin nhắn và người tiêu dùng tiêu thụ chúng. Tuy nhiên, có những cân nhắc khác nhau liên quan đến hiệu suất, phân phối tin nhắn, lưu trữ dữ liệu, v.v.

Dưới đây là một bộ câu hỏi tiềm năng giữa Ứng viên và Người phỏng vấn:
 * C: Định dạng và kích thước trung bình của tin nhắn là bao nhiêu? Chỉ có văn bản thôi phải không?
 * I: Tin nhắn chỉ là văn bản và thường là vài KB.
 * C: Tin nhắn có thể được tiêu thụ lặp lại không?
 * I: Có, tin nhắn có thể được tiêu thụ lặp lại bởi các người tiêu dùng khác nhau. Đây là một yêu cầu bổ sung mà các hàng đợi tin nhắn truyền thống không hỗ trợ.
 * C: Tin nhắn có được tiêu thụ theo đúng thứ tự mà chúng được tạo ra không?
 * I: Có, đảm bảo thứ tự phải được duy trì. Đây là một yêu cầu bổ sung, các hàng đợi tin nhắn truyền thống không hỗ trợ điều này.
 * C: Yêu cầu về lưu trữ dữ liệu (retention) là gì?
 * I: Tin nhắn cần được lưu trữ trong hai tuần. Đây là một yêu cầu bổ sung.
 * C: Chúng ta muốn hỗ trợ bao nhiêu người sản xuất và người tiêu dùng?
 * I: Càng nhiều càng tốt.
 * C: Ngữ nghĩa phân phối dữ liệu (data delivery semantic) nào chúng ta muốn hỗ trợ? At-most-once, at-least-once, exactly-once?
 * I: Chúng ta chắc chắn muốn hỗ trợ ít nhất một lần (at-least-once). Lý tưởng nhất là chúng ta hỗ trợ tất cả và có thể cấu hình được.
 * C: Thông lượng mục tiêu cho độ trễ đầu-cuối (end-to-end latency) là bao nhiêu?
 * I: Nó nên hỗ trợ thông lượng cao cho các trường hợp như tổng hợp log và thông lượng thấp cho các trường hợp truyền thống hơn.

### **Yêu cầu chức năng**

 * Người sản xuất gửi tin nhắn đến một hàng đợi tin nhắn.
 * Người tiêu dùng tiêu thụ tin nhắn từ hàng đợi.
 * Tin nhắn có thể được tiêu thụ một lần hoặc lặp lại.
 * Dữ liệu lịch sử có thể được cắt bớt (truncated).
 * Kích thước tin nhắn nằm trong khoảng vài KB.
 * Thứ tự tin nhắn cần được duy trì.
 * Ngữ nghĩa phân phối dữ liệu có thể cấu hình được - at-most-once/at-least-once/exactly-once.

### **Yêu cầu phi chức năng**

- **Thông lượng cao hoặc độ trễ thấp**: Có thể cấu hình dựa trên trường hợp sử dụng.
- **Có khả năng mở rộng**: Hệ thống phải được phân tán và hỗ trợ sự gia tăng đột biến của khối lượng tin nhắn.
- **Bền vững và lâu dài**: Dữ liệu phải được lưu trữ trên đĩa và được sao chép giữa các nút.

Các hàng đợi tin nhắn truyền thống thường không hỗ trợ lưu trữ dữ liệu lâu dài và không cung cấp đảm bảo thứ tự. Điều này giúp đơn giản hóa thiết kế đáng kể và chúng ta sẽ thảo luận về nó.

---

## Bước 2: Đề xuất thiết kế ở mức cao và đạt được sự đồng thuận

Các thành phần chính của một hàng đợi tin nhắn:

<div style="margin-left:3rem">
    <img src="./images/message-queue-components.png" alt="message-queue-components" width="500" />
</div>

 * Producer gửi tin nhắn đến một hàng đợi.
 * Consumer đăng ký vào một hàng đợi và tiêu thụ các tin nhắn đã đăng ký.
 * Message queue là một dịch vụ ở giữa giúp tách biệt người sản xuất khỏi người tiêu dùng, cho phép chúng mở rộng độc lập.
 * Producer và consumer đều là các client, trong khi message queue là server.

### **Các mô hình nhắn tin**

Loại mô hình nhắn tin đầu tiên là điểm-đến-điểm (point-to-point) và nó thường thấy trong các hàng đợi tin nhắn truyền thống:

<div style="margin-left:3rem">
    <img src="./images/point-to-point-model.png" alt="point-to-point-model" width="500" />
</div>

 * Một tin nhắn được gửi đến hàng đợi và nó được tiêu thụ bởi chính xác một người tiêu dùng.
 * Có thể có nhiều người tiêu dùng, nhưng một tin nhắn chỉ được tiêu thụ một lần.
 * Khi tin nhắn được xác nhận là đã tiêu thụ, nó sẽ được xóa khỏi hàng đợi.
 * Không có lưu trữ dữ liệu lâu dài trong mô hình điểm-đến-điểm, nhưng thiết kế của chúng ta thì có.

Mặt khác, mô hình xuất bản-đăng ký (publish-subscribe) phổ biến hơn cho các nền tảng truyền phát sự kiện:

<div style="margin-left:3rem">
    <img src="./images/publish-subscribe-model.png" alt="publish-subscribe-model" width="500" />
</div>

 * Trong mô hình này, các tin nhắn được liên kết với một chủ đề (topic).
 * Các người tiêu dùng đăng ký vào một chủ đề và họ nhận được tất cả các tin nhắn được gửi đến chủ đề này.

### **Chủ đề, phân mảnh (partitions) và broker**

Nếu khối lượng dữ liệu cho một chủ đề quá lớn thì sao? Một cách để mở rộng là chia nhỏ một chủ đề thành các phân mảnh (partitions - hay còn gọi là sharding):

<div style="margin-left:3rem">
    <img src="./images/partitions.png" alt="partitions" width="500" />
</div>

 * Tin nhắn gửi đến một chủ đề được phân phối đều trên các phân mảnh.
 * Các máy chủ lưu trữ các phân mảnh được gọi là các broker.
 * Mỗi phân mảnh hoạt động giống như một hàng đợi sử dụng FIFO để xử lý tin nhắn. Thứ tự tin nhắn được duy trì trong một phân mảnh.
 * Vị trí của một tin nhắn trong phân mảnh được gọi là một **offset**.
 * Mỗi tin nhắn được tạo ra sẽ được gửi đến một phân mảnh cụ thể. Một khóa phân mảnh (partition key) xác định tin nhắn nên nằm ở phân mảnh nào.
   * Ví dụ: `user_id` có thể được dùng làm khóa phân mảnh để đảm bảo thứ tự tin nhắn cho cùng một người dùng.
 * Mỗi người tiêu dùng đăng ký vào một hoặc nhiều phân mảnh. Khi có nhiều người tiêu dùng cho cùng một loại tin nhắn, họ tạo thành một nhóm người tiêu dùng (consumer group).

### **Nhóm người tiêu dùng (Consumer groups)**

Consumer groups là một tập hợp các người tiêu dùng làm việc cùng nhau để tiêu thụ tin nhắn từ một chủ đề:

<div style="margin-left:3rem">
    <img src="./images/consumer-groups.png" alt="consumer-groups" width="500" />
</div>

 * Tin nhắn được sao chép theo từng nhóm người tiêu dùng (không phải từng người tiêu dùng).
 * Mỗi nhóm người tiêu dùng duy trì offset riêng của mình.
 * Việc đọc tin nhắn song song bởi một nhóm người tiêu dùng giúp cải thiện thông lượng nhưng làm ảnh hưởng đến việc đảm bảo thứ tự.
 * Điều này có thể được giảm thiểu bằng cách chỉ cho phép một người tiêu dùng trong một nhóm đăng ký vào một phân mảnh.
 * Điều này có nghĩa là chúng ta không thể có nhiều người tiêu dùng trong một nhóm hơn số lượng phân mảnh.

### **Kiến trúc ở mức cao**

<div style="margin-left:3rem">
    <img src="./images/high-level-architecture.png" alt="high-level-architecture" width="500" />
</div>

- **Clients**: producer và consumer. Producer đẩy tin nhắn đến một chủ đề được chỉ định. Consumer group đăng ký nhận tin nhắn từ một chủ đề.
- **Brokers**: chứa nhiều phân mảnh. Một phân mảnh chứa một tập hợp con các tin nhắn của một chủ đề.
- **Data storage**: lưu trữ tin nhắn trong các phân mảnh.
- **State storage**: lưu giữ trạng thái của người tiêu dùng.
- **Metadata storage**: lưu trữ cấu hình và thuộc tính của chủ đề.
- **Coordination service**: chịu trách nhiệm khám phá dịch vụ (broker nào đang hoạt động) và bầu chọn leader (broker nào là leader, chịu trách nhiệm phân bổ các phân mảnh).

---

## Bước 3: Thiết kế chi tiết

Để đạt được thông lượng cao và duy trì yêu cầu lưu trữ dữ liệu lâu dài, chúng ta đã thực hiện một số lựa chọn thiết kế quan trọng:
 * Chúng ta chọn một cấu trúc dữ liệu trên đĩa tận dụng các đặc tính của ổ cứng HDD hiện đại và các chiến lược lưu cache đĩa của các hệ điều hành hiện đại.
 * Cấu trúc dữ liệu tin nhắn là bất biến (immutable) để tránh việc sao chép thêm, điều mà chúng ta muốn tránh trong một hệ thống có khối lượng/lưu lượng cao.
 * Chúng ta thiết kế việc ghi dựa trên gộp lô (batching) vì các hoạt động I/O nhỏ là kẻ thù của thông lượng cao.

### **Lưu trữ dữ liệu**

Để tìm ra kho lưu trữ dữ liệu tốt nhất cho tin nhắn, chúng ta phải xem xét các đặc tính của một tin nhắn:
 * Ghi nặng, đọc nặng.
 * Không có hoạt động cập nhật/xóa. Trong các hàng đợi tin nhắn truyền thống, có một hoạt động "xóa" vì tin nhắn không được lưu giữ lâu.
 * Chủ yếu là mô hình truy cập đọc/ghi tuần tự.

Các lựa chọn của chúng ta:
- **Cơ sở dữ liệu**: không lý tưởng vì các cơ sở dữ liệu điển hình không hỗ trợ tốt cả hệ thống ghi và đọc nặng.
- **Write-ahead log (WAL)**: một tệp văn bản thuần túy chỉ hỗ trợ ghi thêm vào cuối và rất thân thiện với ổ cứng HDD. 
  * Chúng ta chia nhỏ các phân mảnh thành các phân đoạn (segments) để tránh việc phải duy trì một tệp quá lớn.
  * Các phân đoạn cũ là chỉ đọc. Các lượt ghi chỉ được chấp nhận bởi phân đoạn mới nhất.

<div style="margin-left:3rem">
    <img src="./images/wal-example.png" alt="wal-example" width="500" />
</div>

Các tệp WAL cực kỳ hiệu quả khi được sử dụng với các ổ cứng HDD truyền thống. 

Có một quan niệm sai lầm rằng truy cập HDD là chậm, nhưng điều đó phụ thuộc rất nhiều vào mô hình truy cập. Khi mô hình truy cập là tuần tự (như trong trường hợp của chúng ta), ổ cứng HDD có thể đạt được tốc độ đọc/ghi vài MB/s, đủ cho nhu cầu của chúng ta. Chúng ta cũng tận dụng thực tế là hệ điều hành lưu cache dữ liệu đĩa vào bộ nhớ một cách mạnh mẽ.

### **Cấu trúc dữ liệu tin nhắn**

Điều quan trọng là schema tin nhắn phải tương thích giữa producer, hàng đợi và consumer để tránh việc sao chép thêm. Điều này cho phép xử lý hiệu quả hơn nhiều.

Ví dụ về cấu trúc tin nhắn:

<div style="margin-left:3rem">
    <img src="./images/message-structure.png" alt="message-structure" width="500" />
</div>

Khóa (key) của tin nhắn xác định tin nhắn đó thuộc về phân mảnh nào. Một ánh xạ ví dụ là `hash(key) % numPartitions`. Để linh hoạt hơn, producer có thể ghi đè các khóa mặc định để kiểm soát việc phân phối tin nhắn đến các phân mảnh nào.

Giá trị (value) của tin nhắn là nội dung của tin nhắn. Nó có thể là văn bản thuần túy hoặc một khối nhị phân đã nén.

**Lưu ý:** Các khóa tin nhắn, không giống như các kho lưu trữ KV truyền thống, không cần phải là duy nhất. Việc có các khóa trùng lặp hoặc thậm chí thiếu khóa là hoàn toàn chấp nhận được.

Các trường khác của tin nhắn:
- **Topic**: chủ đề mà tin nhắn thuộc về.
- **Partition**: ID của phân mảnh mà tin nhắn thuộc về.
- **Offset**: Vị trí của tin nhắn trong một phân mảnh. Một tin nhắn có thể được định vị qua `topic`, `partition`, `offset`.
- **Timestamp**: Thời điểm tin nhắn được lưu trữ.
- **Size**: kích thước của tin nhắn này.
- **CRC**: mã kiểm tra để đảm bảo tính toàn vẹn của tin nhắn.

Các tính năng bổ sung như lọc có thể được hỗ trợ bằng cách thêm các trường bổ sung.

### **Gộp lô (Batching)**

Gộp lô là yếu tố quan trọng đối với hiệu suất của hệ thống. Chúng ta áp dụng nó ở producer, consumer và message queue.

Nó quan trọng vì:
 * Nó cho phép hệ điều hành nhóm các tin nhắn lại với nhau, giúp dàn trải chi phí của các vòng lặp mạng (network round trips) đắt đỏ.
 * Các tin nhắn được ghi vào WAL theo nhóm tuần tự, dẫn đến rất nhiều hoạt động ghi tuần tự và tận dụng tốt bộ nhớ cache đĩa.

Có một sự đánh đổi giữa độ trễ và thông lượng:
 * Gộp lô lớn dẫn đến thông lượng cao hơn nhưng độ trễ cũng cao hơn. 
 * Gộp lô ít dẫn đến thông lượng thấp hơn nhưng độ trễ thấp hơn.

Nếu cần hỗ trợ độ trễ thấp hơn khi hệ thống được triển khai như một hàng đợi tin nhắn truyền thống, hệ thống có thể được điều chỉnh để sử dụng kích thước lô nhỏ hơn.

Nếu được điều chỉnh cho thông lượng, chúng ta có thể cần nhiều phân mảnh hơn cho mỗi chủ đề để bù đắp cho tốc độ ghi đĩa tuần tự chậm hơn.

### **Luồng của người sản xuất (Producer flow)**

Nếu một producer muốn gửi một tin nhắn đến một phân mảnh, nó nên kết nối với broker nào?

Một tùy chọn là giới thiệu một lớp định tuyến (routing layer), lớp này sẽ định tuyến tin nhắn đến đúng broker. Nếu tính năng sao chép (replication) được bật, broker đúng là broker leader của bản sao:

<div style="margin-left:3rem">
    <img src="./images/routing-layer.png" alt="routing-layer" width="500" />
</div>

 * Lớp định tuyến đọc kế hoạch sao chép từ kho lưu trữ siêu dữ liệu và lưu cache cục bộ.
 * Producer gửi một tin nhắn đến lớp định tuyến.
 * Tin nhắn được chuyển tiếp đến broker 1, là leader của phân mảnh đó.
 * Các bản sao follower kéo tin nhắn mới từ leader. Khi nhận được đủ xác nhận, leader sẽ commit dữ liệu và phản hồi lại cho producer.

Lý do có các bản sao là để cho phép khả năng chịu lỗi.

Cách tiếp cận này hoạt động nhưng có một số nhược điểm:
 * Thêm các chặng mạng (network hops) do có thêm thành phần mới.
 * Thiết kế này không cho phép gộp lô tin nhắn.

Để giảm thiểu các vấn đề này, chúng ta có thể nhúng lớp định tuyến vào chính producer:

<div style="margin-left:3rem">
    <img src="./images/routing-layer-producer.png" alt="routing-layer-producer" width="500" />
</div>

 * Ít chặng mạng hơn dẫn đến độ trễ thấp hơn.
 * Producer có thể kiểm soát tin nhắn được định tuyến đến phân mảnh nào.
 * Bộ đệm (buffer) cho phép chúng ta gộp lô tin nhắn trong bộ nhớ và gửi đi các lô lớn hơn trong một yêu cầu duy nhất, giúp tăng thông lượng.

Việc chọn kích thước lô là một sự đánh đổi điển hình giữa thông lượng và độ trễ. 

<div style="margin-left:3rem">
    <img src="./images/batch-size-throughput-vs-latency.png" alt="batch-size-throughput-vs-latency" width="500" />
</div>

 * Kích thước lô lớn hơn dẫn đến thời gian chờ đợi lâu hơn trước khi lô được commit. 
 * Kích thước lô nhỏ hơn dẫn đến yêu cầu được gửi đi sớm hơn và có độ trễ thấp hơn nhưng thông lượng thấp hơn.

### **Luồng của người tiêu dùng (Consumer flow)**

Consumer chỉ định offset của nó trong một phân mảnh và nhận về một đoạn các tin nhắn, bắt đầu từ offset đó:

<div style="margin-left:3rem">
    <img src="./images/consumer-example.png" alt="consumer-example" width="500" />
</div>

Một cân nhắc quan trọng khi thiết kế consumer là nên sử dụng mô hình đẩy (push) hay kéo (pull):
- **Mô hình đẩy (Push model)**: dẫn đến độ trễ thấp hơn vì broker đẩy tin nhắn đến consumer ngay khi nhận được.
  * Tuy nhiên, nếu tốc độ tiêu thụ chậm hơn tốc độ sản xuất, consumer có thể bị quá tải.
  * Rất khó để xử lý các consumer có năng lực xử lý khác nhau vì broker kiểm soát tốc độ tiêu thụ.
- **Mô hình kéo (Pull model)**: dẫn đến việc consumer tự kiểm soát tốc độ tiêu thụ. 
  * Nếu tốc độ tiêu thụ chậm, consumer sẽ không bị quá tải và chúng ta có thể mở rộng quy mô để bắt kịp.
  * Mô hình kéo phù hợp hơn cho việc xử lý theo lô, vì với mô hình đẩy, broker không thể biết consumer có thể xử lý bao nhiêu tin nhắn. 
  * Ngược lại, với mô hình kéo, consumer có thể chủ động lấy về các lô tin nhắn lớn.
  * Nhược điểm là độ trễ cao hơn và các cuộc gọi mạng thừa khi không có tin nhắn mới. Vấn đề sau có thể được giảm thiểu bằng cách sử dụng long polling.

Do đó, hầu hết các hàng đợi tin nhắn (và cả chúng ta) chọn mô hình kéo.

<div style="margin-left:3rem">
    <img src="./images/consumer-flow.png" alt="consumer-flow" width="500" />
</div>

 * Một consumer mới đăng ký vào chủ đề A và gia nhập nhóm 1.
 * Nút broker chính xác được tìm thấy bằng cách băm (hashing) tên nhóm. Bằng cách này, tất cả các consumer trong một nhóm sẽ kết nối với cùng một broker.
 * Lưu ý rằng bộ điều phối nhóm consumer này khác với dịch vụ điều phối (ZooKeeper).
 * Bộ điều phối xác nhận consumer đã gia nhập nhóm và chỉ định phân mảnh 2 cho consumer đó.
 * Có các chiến lược phân bổ phân mảnh khác nhau - round-robin, range, v.v.
 * Consumer lấy về các tin nhắn mới nhất từ offset cuối cùng. Kho lưu trữ trạng thái giữ các offset của consumer.
 * Consumer xử lý tin nhắn và commit offset cho broker. Thứ tự của các hoạt động này ảnh hưởng đến ngữ nghĩa phân phối tin nhắn.

### **Tái cân bằng người tiêu dùng (Consumer rebalancing)**

Tái cân bằng người tiêu dùng chịu trách nhiệm quyết định consumer nào chịu trách nhiệm cho phân mảnh nào. Quá trình này xảy ra khi một consumer gia nhập/rời khỏi hoặc một phân mảnh được thêm/xóa.

Broker, đóng vai trò là bộ điều phối, đóng một vai trò lớn trong việc điều phối luồng công việc tái cân bằng.

<div style="margin-left:3rem">
    <img src="./images/consumer-rebalancing.png" alt="consumer-rebalancing" width="500" />
</div>

 * Tất cả các consumer từ cùng một nhóm được kết nối với cùng một bộ điều phối. Bộ điều phối được tìm thấy bằng cách băm tên nhóm.
 * Khi danh sách consumer thay đổi, bộ điều phối sẽ chọn một leader mới của nhóm.
 * Leader của nhóm tính toán một kế hoạch phân bổ phân mảnh mới và báo cáo lại cho bộ điều phối, bộ điều phối này sẽ phát sóng nó đến các consumer khác.

Khi bộ điều phối ngừng nhận được nhịp tim (heartbeats) từ các consumer trong một nhóm, một cuộc tái cân bằng sẽ được kích hoạt:

<div style="margin-left:3rem">
    <img src="./images/consumer-rebalance-example.png" alt="consumer-rebalance-example" width="500" />
</div>

Hãy khám phá điều gì xảy ra khi một consumer gia nhập một nhóm:

<div style="margin-left:3rem">
    <img src="./images/consumer-join-group-usecase.png" alt="consumer-join-group-usecase" width="500" />
</div>

 * Ban đầu, chỉ có consumer A trong nhóm và nó tiêu thụ tất cả các phân mảnh.
 * Consumer B gửi yêu cầu gia nhập nhóm.
 * Bộ điều phối thông báo cho tất cả các thành viên trong nhóm rằng đã đến lúc tái cân bằng một cách thụ động - như một phản hồi cho nhịp tim.
 * Khi tất cả các consumer gia nhập lại nhóm, bộ điều phối chọn một leader và thông báo cho những người còn lại về kết quả bầu chọn.
 * Leader tạo ra kế hoạch phân bổ phân mảnh và gửi nó cho bộ điều phối. Những người khác đợi kế hoạch phân bổ.
 * Các consumer bắt đầu tiêu thụ từ các phân mảnh mới được chỉ định.

Đây là những gì xảy ra khi một consumer rời khỏi nhóm:

<div style="margin-left:3rem">
    <img src="./images/consumer-leaves-group-usecase.png" alt="consumer-leaves-group-usecase" width="500" />
</div>

 * Consumer A và B ở trong cùng một nhóm.
 * Consumer B yêu cầu rời khỏi nhóm.
 * Khi bộ điều phối nhận được nhịp tim của A, nó thông báo cho A rằng đã đến lúc tái cân bằng.
 * Các bước còn lại đều giống nhau.

Quá trình này cũng tương tự khi một consumer không gửi nhịp tim trong một thời gian dài:

<div style="margin-left:3rem">
    <img src="./images/consumer-no-heartbeat-usecase.png" alt="consumer-no-heartbeat-usecase" width="500" />
</div>

### **Lưu trữ trạng thái (State storage)**

Kho lưu trữ trạng thái lưu trữ ánh xạ giữa các phân mảnh và người tiêu dùng, cũng như các offset tiêu thụ cuối cùng cho một phân mảnh.

<div style="margin-left:3rem">
    <img src="./images/state-storage.png" alt="state-storage" width="500" />
</div>

Offset của Nhóm 1 đang ở mức 6, nghĩa là tất cả các tin nhắn trước đó đã được tiêu thụ. Nếu một consumer bị lỗi, consumer mới sẽ tiếp tục từ tin nhắn đó trở đi.
 
Mô hình truy cập dữ liệu cho trạng thái người tiêu dùng:
 * Các hoạt động đọc/ghi thường xuyên, nhưng khối lượng thấp.
 * Dữ liệu được cập nhật thường xuyên, nhưng hiếm khi bị xóa.
 * Đọc/ghi ngẫu nhiên.
 * Tính nhất quán của dữ liệu là quan trọng.

Với các yêu cầu này, một kho lưu trữ KV nhanh như Zookeeper là lý tưởng.

### **Lưu trữ siêu dữ liệu (Metadata storage)**

Kho lưu trữ siêu dữ liệu lưu trữ cấu hình và các thuộc tính của chủ đề - số lượng phân mảnh, thời gian lưu trữ, phân phối bản sao. Siêu dữ liệu không thay đổi thường xuyên và khối lượng nhỏ, nhưng có yêu cầu tính nhất quán cao. Zookeeper là một lựa chọn tốt cho kho lưu trữ này.

### **ZooKeeper**

Zookeeper là yếu tố thiết yếu để xây dựng các hàng đợi tin nhắn phân tán. Nó là một kho lưu trữ khóa-giá trị phân cấp, thường được sử dụng cho cấu hình phân tán, dịch vụ đồng bộ hóa và đăng ký đặt tên (tức là khám phá dịch vụ).

<div style="margin-left:3rem">
    <img src="./images/zookeeper.png" alt="zookeeper" width="500" />
</div>

Với sự thay đổi này, broker chỉ cần duy trì dữ liệu cho các tin nhắn. Lưu trữ siêu dữ liệu và trạng thái nằm trong Zookeeper. Zookeeper cũng giúp bầu chọn leader của các bản sao broker.

### **Sao chép (Replication)**

Trong các hệ thống phân tán, các sự cố phần cứng là không thể tránh khỏi. Chúng ta có thể giải quyết vấn đề này thông qua sao chép để đạt được tính khả dụng cao.

<div style="margin-left:3rem">
    <img src="./images/replication-example.png" alt="replication-example" width="500" />
</div>

 * Mỗi phân mảnh được sao chép trên nhiều broker, nhưng chỉ có một bản sao leader.
 * Producer gửi tin nhắn đến các bản sao leader.
 * Follower kéo các tin nhắn được sao chép từ leader.
 * Khi đủ số lượng bản sao được đồng bộ hóa, leader sẽ trả về xác nhận cho producer.
 * Việc phân phối các bản sao cho mỗi phân mảnh được gọi là kế hoạch phân phối bản sao (replica distribution plan).
 * Leader của một phân mảnh cụ thể tạo ra kế hoạch phân phối bản sao và lưu nó vào Zookeeper.

### **Các bản sao đồng bộ (In-sync replicas - ISR)**

Một vấn đề chúng ta cần giải quyết là giữ cho tin nhắn được đồng bộ giữa leader và các follower cho một phân mảnh cụ thể. In-sync replicas (ISR) là các bản sao của một phân mảnh luôn giữ trạng thái đồng bộ với leader.

Thông số `replica.lag.max.messages` xác định một bản sao có thể bị tụt lại bao nhiêu tin nhắn so với leader để vẫn được coi là đồng bộ.

<div style="margin-left:3rem">
    <img src="./images/in-sync-replicas-example.png" alt="in-sync-replicas-example" width="500" />
</div>

 * Offset đã commit là 13.
 * Hai tin nhắn mới được ghi vào leader, nhưng chưa được commit.
 * Một tin nhắn được commit khi tất cả các bản sao trong ISR đã đồng bộ hóa tin nhắn đó.
 * Bản sao 2 và 3 đã bắt kịp hoàn toàn với leader, do đó, chúng nằm trong ISR.
 * Bản sao 4 bị tụt lại phía sau, do đó bị loại khỏi ISR tạm thời.

ISR phản ánh sự đánh đổi giữa hiệu suất và độ bền vững.
 * Để producer không bị mất tin nhắn, tất cả các bản sao nên được đồng bộ trước khi gửi xác nhận.
 * Nhưng một bản sao chậm sẽ khiến toàn bộ phân mảnh trở nên không khả dụng.

Việc xử lý xác nhận có thể cấu hình được.

`ACK=all` có nghĩa là tất cả các bản sao trong ISR phải đồng bộ tin nhắn. Việc gửi tin nhắn sẽ chậm hơn, nhưng độ bền vững của tin nhắn là cao nhất.

<div style="margin-left:3rem">
    <img src="./images/ack-all.png" alt="ack-all" width="500" />
</div>

`ACK=1` có nghĩa là producer nhận được xác nhận ngay khi leader nhận được tin nhắn. Việc gửi tin nhắn nhanh, nhưng độ bền vững của tin nhắn thấp.

<div style="margin-left:3rem">
    <img src="./images/ack-1.png" alt="ack-1" width="500" />
</div>

`ACK=0` có nghĩa là producer gửi tin nhắn mà không đợi bất kỳ xác nhận nào từ leader. Việc gửi tin nhắn nhanh nhất, độ bền vững thấp nhất.

<div style="margin-left:3rem">
    <img src="./images/ack-0.png" alt="ack-0" width="500" />
</div>

Về phía consumer, chúng ta có thể kết nối tất cả các consumer với leader của một phân mảnh và để chúng đọc tin nhắn từ đó:
 * Điều này tạo nên thiết kế đơn giản nhất và vận hành dễ dàng nhất.
 * Tin nhắn trong một phân mảnh chỉ được gửi đến một consumer trong một nhóm, giúp giới hạn số lượng kết nối đến bản sao leader.
 * Số lượng kết nối đến bản sao leader thường không cao miễn là chủ đề không quá "hot".
 * Chúng ta có thể mở rộng quy mô một chủ đề hot bằng cách tăng số lượng phân mảnh và người tiêu dùng.
 * Trong một số tình huống nhất định, có thể hợp lý khi để một consumer đọc từ một bản sao ISR, ví dụ: nếu chúng nằm ở một trung tâm dữ liệu (DC) riêng biệt.

Danh sách ISR được duy trì bởi leader, người theo dõi độ trễ giữa mình và từng bản sao.

### **Khả năng mở rộng (Scalability)**

Hãy đánh giá cách chúng ta có thể mở rộng các phần khác nhau của hệ thống.

#### Producer

Producer nhẹ hơn nhiều so với consumer. Khả năng mở rộng của nó có thể dễ dàng đạt được bằng cách thêm/xóa các phiên bản producer mới.

#### Consumer

Các nhóm consumer cô lập với nhau. Rất dễ dàng thêm/xóa các nhóm consumer tùy ý. Tái cân bằng giúp xử lý trường hợp consumer được thêm/xóa khỏi một nhóm một cách duyên dáng. Nhóm consumer và tái cân bằng giúp chúng ta đạt được khả năng mở rộng và khả năng chịu lỗi.

#### Broker

Làm thế nào các broker xử lý sự cố?

<div style="margin-left:3rem">
    <img src="./images/broker-failure-recovery.png" alt="broker-failure-recovery" width="500" />
</div>

 * Khi một broker gặp sự cố, vẫn còn đủ các bản sao để tránh mất dữ liệu phân mảnh.
 * Một leader mới được bầu chọn và bộ điều phối broker sẽ phân phối lại các phân mảnh nằm ở broker bị lỗi sang các bản sao hiện có.
 * Các bản sao hiện có tiếp nhận các phân mảnh mới và đóng vai trò là follower cho đến khi chúng bắt kịp với leader và trở thành ISR.

Các cân nhắc bổ sung để làm cho broker có khả năng chịu lỗi:
 * Số lượng ISR tối thiểu giúp cân bằng giữa độ trễ và sự an toàn. Bạn có thể tinh chỉnh nó để đáp ứng nhu cầu của mình.
 * Nếu tất cả các bản sao của một phân mảnh nằm trong cùng một nút, thì đó là một sự lãng phí tài nguyên. Các bản sao nên nằm trên các broker khác nhau.
 * Nếu tất cả các bản sao của một phân mảnh đều gặp sự cố, dữ liệu sẽ bị mất vĩnh viễn. Việc rải các bản sao qua các trung tâm dữ liệu có thể giúp ích, nhưng nó làm tăng thêm rất nhiều độ trễ. Một tùy chọn là sử dụng [data mirroring](https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=27846330) như một giải pháp thay thế.

Làm thế nào để chúng ta xử lý việc phân phối lại các bản sao khi một broker mới được thêm vào?

<div style="margin-left:3rem">
    <img src="./images/broker-replica-redistribution.png" alt="broker-replica-redistribution" width="500" />
</div>

 * Chúng ta có thể tạm thời cho phép nhiều bản sao hơn mức cấu hình, cho đến khi broker mới bắt kịp.
 * Khi nó đã bắt kịp, chúng ta có thể xóa bản sao phân mảnh không còn cần thiết nữa.

#### Phân mảnh (Partition)

Bất cứ khi nào một phân mảnh mới được thêm vào, producer sẽ được thông báo và việc tái cân bằng consumer sẽ được kích hoạt.

Về mặt lưu trữ dữ liệu, chúng ta chỉ có thể lưu trữ các tin nhắn mới vào phân mảnh mới thay vì cố gắng sao chép tất cả các tin nhắn cũ:

<div style="margin-left:3rem">
    <img src="./images/partition-exmaple.png" alt="partition-example" width="500" />
</div>

Việc giảm số lượng phân mảnh phức tạp hơn:

<div style="margin-left:3rem">
    <img src="./images/partition-decrease.png" alt="partition-decrease" width="500" />
</div>

 * Khi một phân mảnh bị ngừng hoạt động, các tin nhắn mới chỉ được nhận bởi các phân mảnh còn lại.
 * Phân mảnh bị ngừng hoạt động không bị xóa ngay lập tức vì các tin nhắn vẫn có thể được tiêu thụ từ nó.
 * Khi thời gian lưu trữ đã cấu hình trước trôi qua, dữ liệu mới bị cắt bớt và không gian lưu trữ được giải phóng.
 * Trong giai đoạn chuyển tiếp, producer chỉ gửi tin nhắn đến các phân mảnh đang hoạt động, nhưng consumer đọc từ tất cả.
 * Khi thời gian lưu trữ hết hạn, consumer được tái cân bằng.

### **Ngữ nghĩa phân phối dữ liệu (Data delivery semantics)**

Hãy thảo luận về các ngữ nghĩa phân phối khác nhau.

#### Tối đa một lần (At-most once)

Với đảm bảo này, tin nhắn được phân phối không quá một lần và có thể không được phân phối chút nào.

<div style="margin-left:3rem">
    <img src="./images/at-most-once.png" alt="at-most-once" width="500" />
</div>

 * Producer gửi một tin nhắn không đồng bộ đến một chủ đề. Nếu việc phân phối tin nhắn thất bại, không có việc thử lại.
 * Consumer lấy tin nhắn và commit offset ngay lập tức. Nếu consumer gặp sự cố trước khi xử lý tin nhắn, tin nhắn đó sẽ không được xử lý.

#### Ít nhất một lần (At-least once)

Một tin nhắn có thể được gửi nhiều hơn một lần và không có tin nhắn nào bị bỏ sót không được xử lý.

<div style="margin-left:3rem">
    <img src="./images/at-least-once.png" alt="at-least-once" width="500" />
</div>

 * Producer gửi tin nhắn với `ack=1` hoặc `ack=all`. Nếu có bất kỳ vấn đề gì, nó sẽ tiếp tục thử lại.
 * Consumer lấy tin nhắn và chỉ commit offset sau khi đã xử lý xong.
 * Có khả năng một tin nhắn được phân phối nhiều hơn một lần nếu ví dụ consumer gặp sự cố sau khi xử lý xong tin nhắn nhưng trước khi commit offset.
 * Đây là lý do tại sao phương thức này phù hợp cho các trường hợp sử dụng chấp nhận dữ liệu trùng lặp hoặc có thể thực hiện loại bỏ trùng lặp (deduplication).

#### Chính xác một lần (Exactly once)

Cực kỳ tốn kém để triển khai cho hệ thống, mặc dù đây là sự đảm bảo thân thiện nhất với người dùng:

<div style="margin-left:3rem">
    <img src="./images/exactly-once.png" alt="exactly-once" width="500" />
</div>

---

### **Các tính năng nâng cao**

Hãy thảo luận về một số tính năng nâng cao mà chúng ta có thể đề cập trong buổi phỏng vấn.

#### Lọc tin nhắn (Message filtering)

Một số consumer có thể chỉ muốn tiêu thụ các tin nhắn thuộc một loại nhất định trong một phân mảnh. Điều này có thể được thực hiện bằng cách xây dựng các chủ đề riêng biệt cho từng tập hợp con các tin nhắn, nhưng điều này có thể tốn kém nếu hệ thống có quá nhiều trường hợp sử dụng khác nhau.
 * Sẽ lãng phí tài nguyên khi lưu trữ cùng một tin nhắn trên các chủ đề khác nhau.
 * Producer giờ đây bị ràng buộc chặt chẽ với các consumer vì nó thay đổi theo mỗi yêu cầu mới của consumer.

Chúng ta có thể giải quyết vấn đề này bằng cách lọc tin nhắn.
 * Một cách tiếp cận thô sơ là thực hiện lọc ở phía consumer, nhưng điều đó gây ra lưu lượng truy cập consumer không cần thiết.
 * Ngoài ra, tin nhắn có thể được gắn thẻ (tags) và consumer có thể chỉ định thẻ nào họ muốn đăng ký.
 * Việc lọc cũng có thể được thực hiện qua nội dung tin nhắn nhưng điều đó có thể khó khăn và không an toàn cho các tin nhắn được mã hóa/tuần tự hóa.
 * Đối với các công thức toán học phức tạp hơn, broker có thể triển khai một bộ phân tích ngữ pháp hoặc bộ thực thi kịch bản, nhưng điều đó có thể gây nặng nề cho hàng đợi tin nhắn.

<div style="margin-left:3rem">
    <img src="./images/message-filtering.png" alt="message-filtering" width="500" />
</div>

#### Tin nhắn bị trì hoãn & tin nhắn được lên lịch (Delayed messages & scheduled messages)

Đối với một số trường hợp sử dụng, chúng ta có thể muốn trì hoãn hoặc lên lịch phân phối tin nhắn. Ví dụ, chúng ta có thể gửi một yêu cầu kiểm tra xác minh thanh toán sau 30 phút kể từ bây giờ, yêu cầu này sẽ kích hoạt consumer kiểm tra xem việc thanh toán có thành công hay không.

Điều này có thể được thực hiện bằng cách gửi tin nhắn đến kho lưu trữ tạm thời trong broker và di chuyển tin nhắn đó đến phân mảnh vào thời điểm thích hợp:

<div style="margin-left:3rem">
    <img src="./images/delayed-message-implementation.png" alt="delayed-message-implementation" width="500" />
</div>

 * Kho lưu trữ tạm thời có thể là một hoặc nhiều chủ đề tin nhắn đặc biệt.
 * Chức năng thời gian có thể được thực hiện bằng cách sử dụng các hàng đợi trì hoãn chuyên dụng hoặc một [bánh xe thời gian phân cấp (hierarchical time wheel)](http://www.cs.columbia.edu/~nahum/w6998/papers/sosp87-timing-wheels.pdf).

---

## Bước 4: Tổng kết

Các điểm thảo luận bổ sung:
- **Giao thức truyền thông**: Các cân nhắc quan trọng - hỗ trợ tất cả các trường hợp sử dụng và khối lượng dữ liệu lớn, cũng như xác minh tính toàn vẹn của tin nhắn. Các giao thức phổ biến - AMQP và giao thức Kafka.
- **Tiêu thụ lại (Retry consumption)**: nếu chúng ta không thể xử lý một tin nhắn ngay lập tức, chúng ta có thể gửi nó đến một chủ đề thử lại chuyên dụng để thử lại sau.
- **Lưu trữ dữ liệu lịch sử**: các tin nhắn cũ có thể được sao lưu trong các kho lưu trữ dung lượng cao như HDFS hoặc lưu trữ đối tượng (ví dụ: S3).
