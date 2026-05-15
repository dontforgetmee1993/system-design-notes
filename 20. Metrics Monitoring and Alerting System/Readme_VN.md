# Chương 20: Hệ thống Giám sát Chỉ số và Cảnh báo

## Giới thiệu
Chương này tập trung vào việc thiết kế một **hệ thống giám sát chỉ số và cảnh báo** (metrics monitoring and alerting system) có khả năng mở rộng cao, điều này rất quan trọng để đảm bảo tính sẵn sàng và độ tin cậy cao của hệ thống.

---

## Bước 1: Hiểu vấn đề và Thiết lập Phạm vi Thiết kế
Một hệ thống giám sát chỉ số có thể mang nhiều ý nghĩa khác nhau - ví dụ: bạn không muốn thiết kế một hệ thống thu thập log (logs aggregation) khi người phỏng vấn chỉ quan tâm đến các chỉ số hạ tầng (infra metrics).

Hãy cố gắng hiểu vấn đề trước:
 - C: Chúng ta đang xây dựng hệ thống cho ai? Một hệ thống giám sát nội bộ cho một công ty công nghệ lớn hay một dịch vụ SaaS như DataDog?
 - I: Chúng ta đang xây dựng để sử dụng nội bộ.
 - C: Những chỉ số nào chúng ta muốn thu thập?
 - I: Các chỉ số vận hành hệ thống - tải CPU, Bộ nhớ, dung lượng đĩa dữ liệu. Ngoài ra còn có các chỉ số mức cao như số lượng yêu cầu mỗi giây (requests per second). Các chỉ số kinh doanh không nằm trong phạm vi.
 - C: Quy mô của hạ tầng chúng ta đang giám sát là bao nhiêu?
 - I: 100 triệu người dùng hoạt động hàng ngày, 1000 nhóm máy chủ (server pools), 100 máy mỗi nhóm.
 - C: Chúng ta nên giữ dữ liệu trong bao lâu?
 - I: Giả sử thời gian lưu trữ là 1 năm.
 - C: Chúng ta có thể giảm độ phân giải của dữ liệu chỉ số để lưu trữ dài hạn không?
 - I: Giữ các chỉ số mới nhận được trong 7 ngày. Tổng hợp chúng thành độ phân giải 1 phút cho 30 ngày tiếp theo. Tiếp tục tổng hợp thành độ phân giải 1 giờ sau 30 ngày.
 - C: Các kênh cảnh báo được hỗ trợ là gì?
 - I: Email, điện thoại, PagerDuty hoặc webhooks.
 - C: Chúng ta có cần thu thập log như log lỗi hoặc log truy cập không?
 - I: Không.
 - C: Chúng ta có cần hỗ trợ truy vết hệ thống phân tán (distributed system tracing) không?
 - I: Không.

### **Các yêu cầu và giả định mức cao**
Hạ tầng được giám sát có quy mô lớn:
 - 100 triệu DAU
 - 1000 nhóm máy chủ * 100 máy * ~100 chỉ số mỗi máy -> ~10 triệu chỉ số
 - Lưu trữ dữ liệu trong 1 năm
 - Chính sách lưu trữ dữ liệu - dữ liệu thô trong 7 ngày, độ phân giải 1 phút trong 30 ngày, độ phân giải 1 giờ trong 1 năm.

Nhiều loại chỉ số có thể được giám sát:
 - Tải CPU
 - Số lượng yêu cầu (Request count)
 - Sử dụng bộ nhớ (Memory usage)
 - Số lượng tin nhắn trong hàng đợi tin nhắn (Message count in message queues)

### **Yêu cầu phi chức năng**
 - **Tính khả năng mở rộng (Scalability)**: Hệ thống phải có khả năng mở rộng để đáp ứng thêm nhiều chỉ số và cảnh báo.
 - **Độ trễ thấp (Low latency)**: Hệ thống cần có độ trễ truy vấn thấp cho các dashboard và cảnh báo.
 - **Độ tin cậy (Reliability)**: Hệ thống phải có độ tin cậy cao để tránh bỏ lỡ các cảnh báo quan trọng.
 - **Tính linh hoạt (Flexibility)**: Hệ thống có thể dễ dàng tích hợp các công nghệ mới trong tương lai.

Những yêu cầu nào nằm ngoài phạm vi?
 - **Giám sát log (Log monitoring)**: stack ELK rất phổ biến cho trường hợp sử dụng này.
 - **Truy vết hệ thống phân tán (Distributed system tracing)**: điều này đề cập đến việc thu thập dữ liệu về vòng đời của một yêu cầu khi nó chảy qua nhiều dịch vụ trong hệ thống.

---

## Bước 2: Đề xuất Thiết kế Mức cao và Đạt được sự Thống nhất

### **Các nguyên tắc cơ bản**
Có năm thành phần cốt lõi trong một hệ thống giám sát chỉ số và cảnh báo:

<div style="margin-left:3rem">
    <img src="./images/metrics-monitoring-core-components.png" alt="metrics-monitoring-core-components" width="500" />
</div>

 - **Thu thập dữ liệu (Data collection)**: thu thập dữ liệu chỉ số từ các nguồn khác nhau.
 - **Truyền tải dữ liệu (Data transmission)**: chuyển dữ liệu từ các nguồn đến hệ thống giám sát.
 - **Lưu trữ dữ liệu (Data storage)**: tổ chức và lưu trữ dữ liệu đi vào.
 - **Cảnh báo (Alerting)**: Phân tích dữ liệu đi vào, phát hiện bất thường và tạo cảnh báo.
 - **Trực quan hóa (Visualization)**: Trình bày dữ liệu dưới dạng đồ thị, biểu đồ, v.v.

### **Mô hình dữ liệu**
Dữ liệu chỉ số thường được ghi lại dưới dạng chuỗi thời gian (time-series), bao gồm một tập hợp các giá trị đi kèm với mốc thời gian (timestamps).
Chuỗi này có thể được định danh bằng tên và một tập hợp các thẻ (tags) tùy chọn.

Ví dụ 1 - Tải CPU trên phiên bản máy chủ production i631 lúc 20:00 là bao nhiêu?

<div style="margin-left:3rem">
    <img src="./images/metrics-example-1.png" alt="metrics-example-1" width="500" />
</div>

Dữ liệu có thể được xác định bằng bảng sau:

<div style="margin-left:3rem">
    <img src="./images/metrics-example-1-data.png" alt="metrics-example-1-data" width="500" />
</div>

Chuỗi thời gian được xác định bởi tên chỉ số, nhãn (labels) và một điểm dữ liệu duy nhất tại một thời điểm cụ thể.

Ví dụ 2 - Tải CPU trung bình trên tất cả các máy chủ web ở vùng us-west trong 10 phút qua là bao nhiêu?

```
CPU.load host=webserver01,region=us-west 1613707265 50

CPU.load host=webserver01,region=us-west 1613707265 62

CPU.load host=webserver02,region=us-west 1613707265 43

CPU.load host=webserver02,region=us-west 1613707265 53

...

CPU.load host=webserver01,region=us-west 1613707265 76

CPU.load host=webserver01,region=us-west 1613707265 83
```

Đây là ví dụ về dữ liệu chúng ta có thể lấy từ bộ lưu trữ để trả lời câu hỏi đó.
Tải CPU trung bình có thể được tính bằng cách lấy trung bình các giá trị ở cột cuối cùng của các hàng.

Định dạng hiển thị ở trên được gọi là giao thức dòng (line protocol) và được sử dụng bởi nhiều phần mềm giám sát phổ biến trên thị trường - ví dụ: Prometheus, OpenTSDB.

Mọi chuỗi thời gian bao gồm:

<div style="margin-left:3rem">
    <img src="./images/time-series-data-example.png" alt="time-series-data-example" width="500" />
</div>

Một cách tốt để trực quan hóa dữ liệu trông như thế nào:

<div style="margin-left:3rem">
    <img src="./images/time-series-data-viz.png" alt="time-series-data-viz" width="500" />
</div>

 - Trục x là thời gian.
 - Trục y là chiều (dimension) bạn đang truy vấn - ví dụ: tên chỉ số, thẻ, v.v.

Mô hình truy cập dữ liệu là ghi nhiều (write-heavy) và đọc đột biến (spiky reads) vì chúng ta thu thập rất nhiều chỉ số, nhưng chúng ít khi được truy cập, mặc dù sẽ truy cập dồn dập khi có sự cố đang xảy ra.

Hệ thống lưu trữ dữ liệu là trái tim của thiết kế này.
 - Không nên sử dụng cơ sở dữ liệu đa mục đích (general-purpose database) cho vấn đề này, mặc dù bạn có thể đạt được quy mô tốt nếu tinh chỉnh ở mức độ chuyên gia.
 - Sử dụng cơ sở dữ liệu NoSQL về lý thuyết có thể hoạt động, nhưng rất khó để thiết kế một schema có khả năng mở rộng để lưu trữ và truy vấn dữ liệu chuỗi thời gian một cách hiệu quả.

Có nhiều cơ sở dữ liệu được thiết kế riêng để lưu trữ dữ liệu chuỗi thời gian. Nhiều trong số đó hỗ trợ các giao diện truy vấn tùy chỉnh cho phép truy vấn hiệu quả dữ liệu chuỗi thời gian.
 - OpenTSDB là một cơ sở dữ liệu chuỗi thời gian phân tán, nhưng nó dựa trên Hadoop và HBase. Nếu bạn không có sẵn hạ tầng đó, sẽ rất khó để sử dụng công nghệ này.
 - Twitter sử dụng MetricsDB, trong khi Amazon cung cấp Timestream.
 - Hai cơ sở dữ liệu chuỗi thời gian phổ biến nhất là InfluxDB và Prometheus.
 - Chúng được thiết kế để lưu trữ khối lượng lớn dữ liệu chuỗi thời gian. Cả hai đều dựa trên bộ nhớ đệm trong RAM + lưu trữ trên đĩa.

Ví dụ về quy mô của InfluxDB - hơn 250 nghìn lượt ghi mỗi giây khi được trang bị 8 lõi và 32GB RAM:

<div style="margin-left:3rem">
    <img src="./images/influxdb-scale.png" alt="influxdb-scale" width="500" />
</div>

Bạn không nhất thiết phải hiểu nội bộ của một cơ sở dữ liệu chỉ số vì đó là kiến thức chuyên sâu. Bạn chỉ có thể bị hỏi nếu bạn đã đề cập đến nó trong CV của mình.

Đối với mục đích của cuộc phỏng vấn, chỉ cần hiểu rằng các chỉ số là dữ liệu chuỗi thời gian và biết về các cơ sở dữ liệu chuỗi thời gian phổ biến như InfluxDB là đủ.

Một tính năng hay của cơ sở dữ liệu chuỗi thời gian là việc tổng hợp và phân tích hiệu quả một lượng lớn dữ liệu chuỗi thời gian theo nhãn.
Ví dụ, InfluxDB xây dựng các chỉ mục (indexes) cho mỗi nhãn.

Tuy nhiên, điều quan trọng là phải giữ cho lực lượng (cardinality) của các nhãn ở mức thấp - tức là không sử dụng quá nhiều nhãn duy nhất.

### **Thiết kế mức cao**

<div style="margin-left:3rem">
    <img src="./images/high-level-design.png" alt="high-level-design" width="500" />
</div>

 - **Nguồn chỉ số (Metrics source)**: có thể là các máy chủ ứng dụng, cơ sở dữ liệu SQL, hàng đợi tin nhắn, v.v.
 - **Bộ thu thập chỉ số (Metrics collector)**: Thu thập dữ liệu chỉ số và ghi vào cơ sở dữ liệu chuỗi thời gian.
 - **Cơ sở dữ liệu chuỗi thời gian (Time-series database)**: lưu trữ các chỉ số dưới dạng chuỗi thời gian. Cung cấp một giao diện truy vấn tùy chỉnh để phân tích lượng lớn các chỉ số.
 - **Dịch vụ truy vấn (Query service)**: Giúp việc truy vấn và lấy dữ liệu từ DB chuỗi thời gian trở nên dễ dàng. Có thể được thay thế hoàn toàn bằng giao diện của DB nếu nó đủ mạnh.
 - **Hệ thống cảnh báo (Alerting system)**: Gửi thông báo cảnh báo đến các điểm đến cảnh báo khác nhau.
 - **Hệ thống trực quan hóa (Visualization system)**: Hiển thị các chỉ số dưới dạng đồ thị/biểu đồ.

---

## Bước 3: Thiết kế Chi tiết
Hãy đi sâu vào một số phần thú vị hơn của hệ thống.

### **Thu thập chỉ số**
Đối với việc thu thập chỉ số, thỉnh thoảng mất dữ liệu không phải là vấn đề quá nghiêm trọng. Việc các client gửi và quên (fire and forget) là có thể chấp nhận được.

<div style="margin-left:3rem">
    <img src="./images/metrics-collection.png" alt="metrics-collection" width="500" />
</div>

Có hai cách để triển khai việc thu thập chỉ số - kéo (pull) hoặc đẩy (push).

Đây là cách mô hình kéo có thể trông như thế nào:

<div style="margin-left:3rem">
    <img src="./images/pull-model-example.png" alt="pull-model-example" width="500" />
</div>

Đối với giải pháp này, bộ thu thập chỉ số cần duy trì một danh sách cập nhật của các dịch vụ và các điểm cuối (endpoints) của chỉ số.
Chúng ta có thể sử dụng Zookeeper hoặc etcd cho mục đích đó - phát hiện dịch vụ (service discovery).

Phát hiện dịch vụ chứa các quy tắc cấu hình về thời điểm và nơi thu thập các chỉ số:

<div style="margin-left:3rem">
    <img src="./images/service-discovery-example.png" alt="service-discovery-example" width="500" />
</div>

Dưới đây là giải thích chi tiết về luồng thu thập chỉ số:

<div style="margin-left:3rem">
    <img src="./images/metrics-collection-flow.png" alt="metrics-collection-flow" width="500" />
</div>

 - Bộ thu thập chỉ số lấy siêu dữ liệu cấu hình từ service discovery. Điều này bao gồm khoảng thời gian kéo, địa chỉ IP, các tham số timeout và retry.
 - Bộ thu thập kéo dữ liệu chỉ số thông qua một điểm cuối http được xác định trước (ví dụ `/metrics`). Điều này thường được thực hiện bởi một thư viện máy khách.
 - Ngoài ra, bộ thu thập chỉ số có thể đăng ký thông báo sự kiện thay đổi với service discovery để được thông báo khi điểm cuối dịch vụ thay đổi.
 - Một lựa chọn khác là bộ thu thập chỉ số định kỳ thăm dò (poll) các thay đổi cấu hình điểm cuối chỉ số.

Ở quy mô của chúng ta, một bộ thu thập chỉ số duy nhất là không đủ. Phải có nhiều phiên bản.
Tuy nhiên, cũng phải có một loại cơ chế đồng bộ hóa giữa chúng để hai bộ thu thập không thu thập cùng một chỉ số hai lần.

Một giải pháp cho vấn đề này là định vị các bộ thu thập và các máy chủ trên một vòng băm nhất quán (consistent hash ring) và liên kết một nhóm máy chủ với duy nhất một bộ thu thập:

<div style="margin-left:3rem">
    <img src="./images/consistent-hash-ring.png" alt="consistent-hash-ring" width="500" />
</div>

Ngược lại, với mô hình đẩy, các dịch vụ chủ động đẩy các chỉ số của chúng đến bộ thu thập chỉ số:

<div style="margin-left:3rem">
    <img src="./images/push-model-example.png" alt="push-model-example" width="500" />
</div>

Trong cách tiếp cận này, thông thường một tác nhân thu thập (collection agent) được cài đặt cùng với các phiên bản dịch vụ.
Tác nhân thu thập các chỉ số từ máy chủ và đẩy chúng đến bộ thu thập chỉ số.

<div style="margin-left:3rem">
    <img src="./images/metrics-collector-agent.png" alt="metrics-collector-agent" width="500" />
</div>

Với mô hình này, chúng ta có thể tổng hợp các chỉ số trước khi gửi chúng đến bộ thu thập, giúp giảm khối lượng dữ liệu được xử lý bởi bộ thu thập.

Mặt khác, bộ thu thập chỉ số có thể từ chối các yêu cầu đẩy nếu nó không thể xử lý tải.
Do đó, việc thêm bộ thu thập vào một nhóm tự động mở rộng (auto-scaling group) nằm sau một bộ cân bằng tải là rất quan trọng.

Vậy cái nào tốt hơn? Có sự đánh đổi giữa cả hai cách tiếp cận và các hệ thống khác nhau sử dụng các cách tiếp cận khác nhau:
 - Prometheus sử dụng kiến trúc kéo (pull).
 - Amazon Cloud Watch và Graphite sử dụng kiến trúc đẩy (push).

Dưới đây là một số khác biệt chính giữa đẩy và kéo:
| | Pull | Push |
|----------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Dễ dàng gỡ lỗi (Easy debugging) | Điểm cuối /metrics trên các máy chủ ứng dụng được sử dụng để kéo chỉ số có thể được dùng để xem các chỉ số bất cứ lúc nào. Bạn thậm chí có thể thực hiện việc này trên laptop của mình. Pull thắng. | Nếu bộ thu thập chỉ số không nhận được các chỉ số, vấn đề có thể do các sự cố mạng. |
| Kiểm tra sức khỏe (Health check) | Nếu một máy chủ ứng dụng không phản hồi yêu cầu kéo, bạn có thể nhanh chóng biết được máy chủ đó có bị sập hay không. Pull thắng. | Nếu bộ thu thập chỉ số không nhận được các chỉ số, vấn đề có thể do các sự cố mạng. |
| Các công việc ngắn hạn (Short-lived jobs) | | Một số công việc chạy theo lô (batch jobs) có thể diễn ra ngắn hạn và không tồn tại đủ lâu để được kéo. Push thắng. Điều này có thể được khắc phục bằng cách giới thiệu các push gateway cho mô hình kéo [22]. |
| Tường lửa hoặc thiết lập mạng phức tạp | Việc có các máy chủ kéo chỉ số yêu cầu tất cả các điểm cuối chỉ số phải có thể truy cập được. Điều này có thể gây vấn đề trong các thiết lập đa trung tâm dữ liệu. Nó có thể yêu cầu một hạ tầng mạng phức tạp hơn. | Nếu bộ thu thập chỉ số được thiết lập với bộ cân bằng tải và nhóm tự động mở rộng, nó có thể nhận dữ liệu từ bất kỳ đâu. Push thắng. |
| Hiệu năng (Performance) | Các phương pháp kéo thường sử dụng TCP. | Các phương pháp đẩy thường sử dụng UDP. Điều này có nghĩa là phương pháp đẩy cung cấp việc truyền tải các chỉ số với độ trễ thấp hơn. Lập luận ngược lại ở đây là nỗ lực thiết lập kết nối TCP là nhỏ so với việc gửi tải trọng chỉ số. |
| Tính xác thực của dữ liệu (Data authenticity) | Các máy chủ ứng dụng để thu thập chỉ số được xác định trước trong các file cấu hình. Các chỉ số thu thập được từ các máy chủ đó được đảm bảo là xác thực. | Bất kỳ loại client nào cũng có thể đẩy các chỉ số đến bộ thu thập chỉ số. Điều này có thể được khắc phục bằng cách đưa các máy chủ được chấp nhận vào danh sách trắng (whitelisting), hoặc bằng cách yêu cầu xác thực. |

Không có người chiến thắng rõ ràng. Một tổ chức lớn có lẽ cần hỗ trợ cả hai. Có thể ban đầu không có cách nào để cài đặt một push agent.

### **Mở rộng quy mô đường ống truyền tải chỉ số**

<div style="margin-left:3rem">
    <img src="./images/metrics-transmission-pipeline.png" alt="metrics-transmission-pipeline" width="500" />
</div>

Bộ thu thập chỉ số được cung cấp trong một nhóm tự động mở rộng, bất kể chúng ta sử dụng mô hình đẩy hay kéo.

Tuy nhiên, có khả năng mất dữ liệu nếu DB chuỗi thời gian bị sập. Để giảm thiểu điều này, chúng ta sẽ cung cấp một cơ chế hàng đợi (queuing):

<div style="margin-left:3rem">
    <img src="./images/queuing-mechanism.png" alt="queuing-mechanism" width="500" />
</div>

 - Các bộ thu thập chỉ số đẩy dữ liệu chỉ số vào Kafka.
 - Các consumer hoặc các dịch vụ xử lý luồng (stream processing) như Apache Storm, Flink hoặc Spark xử lý dữ liệu và đẩy nó vào DB chuỗi thời gian.

Cách tiếp cận này có một số lợi thế:
 - Kafka được sử dụng như một nền tảng tin nhắn phân tán có độ tin cậy cao và khả năng mở rộng tốt.
 - Nó tách biệt việc thu thập dữ liệu và xử lý dữ liệu với nhau.
 - Nó có thể ngăn chặn mất dữ liệu bằng cách giữ lại dữ liệu trong Kafka.

Kafka có thể được cấu hình với một phân vùng (partition) cho mỗi tên chỉ số, để các consumer có thể tổng hợp dữ liệu theo tên chỉ số.
Để mở rộng quy mô, chúng ta có thể phân vùng thêm theo thẻ/nhãn và phân loại/ưu tiên các chỉ số cần được thu thập trước.

<div style="margin-left:3rem">
    <img src="./images/metrics-collection-kafka.png" alt="metrics-collection-kafka" width="500" />
</div>

Nhược điểm chính của việc sử dụng Kafka cho vấn đề này là chi phí bảo trì/vận hành.
Một giải pháp thay thế là sử dụng một hệ thống nạp dữ liệu (ingestion system) quy mô lớn như [Gorilla](https://www.vldb.org/pvldb/vol8/p1816-teller.pdf).
Có thể lập luận rằng việc sử dụng nó sẽ có khả năng mở rộng tương đương với việc sử dụng Kafka cho hàng đợi.

### **Nơi thực hiện các phép tổng hợp**
Các chỉ số có thể được tổng hợp tại nhiều nơi. Có sự đánh đổi giữa các lựa chọn khác nhau:
 - **Tác nhân thu thập (Collection agent)**: tác nhân thu thập phía client chỉ hỗ trợ logic tổng hợp đơn giản. Ví dụ: thu thập một bộ đếm trong 1 phút và gửi nó đến bộ thu thập chỉ số.
 - **Đường ống nạp dữ liệu (Ingestion pipeline)**: Để tổng hợp dữ liệu trước khi ghi vào DB, chúng ta cần một công cụ xử lý luồng như Flink. Điều này làm giảm khối lượng ghi, nhưng chúng ta mất đi độ chính xác của dữ liệu vì không lưu trữ dữ liệu thô.
 - **Phía truy vấn (Query side)**: Chúng ta có thể tổng hợp dữ liệu khi chạy các truy vấn thông qua hệ thống trực quan hóa. Không có sự mất mát dữ liệu, nhưng các truy vấn có thể chậm do phải xử lý nhiều dữ liệu.

### **Dịch vụ Truy vấn**
Việc có một dịch vụ truy vấn riêng biệt với DB chuỗi thời gian giúp tách biệt hệ thống trực quan hóa và cảnh báo khỏi cơ sở dữ liệu, cho phép chúng ta tách biệt DB khỏi các client và thay đổi nó theo ý muốn.

Chúng ta có thể thêm một lớp Cache ở đây để giảm tải cho cơ sở dữ liệu chuỗi thời gian:

<div style="margin-left:3rem">
    <img src="./images/cache-layer-query-service.png" alt="cache-layer-query-service" width="500" />
</div>

Chúng ta cũng có thể tránh hoàn toàn việc thêm dịch vụ truy vấn vì hầu hết các hệ thống trực quan hóa và cảnh báo đều có các plugin mạnh mẽ để tích hợp với hầu hết các cơ sở dữ liệu chuỗi thời gian.
Với một DB chuỗi thời gian được chọn tốt, chúng ta cũng có thể không cần giới thiệu lớp caching của riêng mình.

Hầu hết các DB chuỗi thời gian không hỗ trợ SQL đơn giản vì nó không hiệu quả để truy vấn dữ liệu chuỗi thời gian. Dưới đây là một ví dụ về truy vấn SQL để tính toán trung bình động lũy thừa (exponential moving average):

```
select id,
       temp,
       avg(temp) over (partition by group_nr order by time_read) as rolling_avg
from (
  select id,
         temp,
         time_read,
         interval_group,
         id - row_number() over (partition by interval_group order by time_read) as group_nr
  from (
    select id,
    time_read,
    "epoch"::timestamp + "900 seconds"::interval * (extract(epoch from time_read)::int4 / 900) as interval_group,
    temp
    from readings
  ) t1
) t2
order by time_read;
```

Đây là cùng một truy vấn trong Flux - ngôn ngữ truy vấn được sử dụng trong InfluxDB:

```
from(db:"telegraf")
  |> range(start:-1h)
  |> filter(fn: (r) => r._measurement == "foo")
  |> exponentialMovingAverage(size:-10s)
```

### **Lớp lưu trữ**
Điều quan trọng là phải chọn cơ sở dữ liệu chuỗi thời gian một cách cẩn thận.

Theo nghiên cứu được công bố bởi Facebook, ~85% các truy vấn đến kho lưu trữ vận hành là dành cho dữ liệu từ 26 giờ qua.

Nếu chúng ta chọn một cơ sở dữ liệu tận dụng được đặc tính này, nó có thể có tác động đáng kể đến hiệu năng hệ thống. InfluxDB là một lựa chọn như vậy.

Bất kể cơ sở dữ liệu chúng ta chọn là gì, có một số tối ưu hóa mà chúng ta có thể áp dụng.

Mã hóa và nén dữ liệu có thể làm giảm đáng kể kích thước dữ liệu. Các tính năng đó thường được tích hợp sẵn trong một cơ sở dữ liệu chuỗi thời gian tốt.

<div style="margin-left:3rem">
    <img src="./images/double-delta-encoding.png" alt="double-delta-encoding" width="500" />
</div>

Trong ví dụ trên, thay vì lưu trữ các mốc thời gian đầy đủ, chúng ta có thể lưu trữ các hiệu số mốc thời gian (timestamp deltas).

Một kỹ thuật khác mà chúng ta có thể áp dụng là giảm mẫu (down-sampling) - chuyển đổi dữ liệu độ phân giải cao thành độ phân giải thấp để giảm mức sử dụng đĩa.

Chúng ta có thể sử dụng kỹ thuật đó cho dữ liệu cũ và làm cho các quy tắc có thể cấu hình được bởi các nhà khoa học dữ liệu, ví dụ:
 - 7 ngày - không giảm mẫu.
 - 30 ngày - giảm mẫu xuống độ phân giải 1 phút.
 - 1 năm - giảm mẫu xuống độ phân giải 1 giờ.

Ví dụ, đây là bảng chỉ số với độ phân giải 10 giây:
| metric | timestamp            | hostname | Metric_value |
|--------|----------------------|----------|--------------|
| cpu    | 2021-10-24T19:00:00Z | host-a   | 10           |
| cpu    | 2021-10-24T19:00:10Z | host-a   | 16           |
| cpu    | 2021-10-24T19:00:20Z | host-a   | 20           |
| cpu    | 2021-10-24T19:00:30Z | host-a   | 30           |
| cpu    | 2021-10-24T19:00:40Z | host-a   | 20           |
| cpu    | 2021-10-24T19:00:50Z | host-a   | 30           |

giảm mẫu xuống độ phân giải 30 giây:
| metric | timestamp            | hostname | Metric_value (avg) |
|--------|----------------------|----------|--------------------|
| cpu    | 2021-10-24T19:00:00Z | host-a   | 19                 |
| cpu    | 2021-10-24T19:00:30Z | host-a   | 25                 |

Cuối cùng, chúng ta cũng có thể sử dụng lưu trữ lạnh (cold storage) cho dữ liệu cũ không còn được sử dụng thường xuyên. Chi phí tài chính cho lưu trữ lạnh thấp hơn nhiều.

### **Hệ thống cảnh báo**

<div style="margin-left:3rem">
    <img src="./images/alerting-system.png" alt="alerting-system" width="500" />
</div>

Cấu hình được tải lên các máy chủ cache. Các quy tắc thường được định nghĩa ở định dạng YAML. Đây là một ví dụ:

```
- name: instance_down
  rules:

  # Alert for any instance that is unreachable for >5 minutes.
  - alert: instance_down
    expr: up == 0
    for: 5m
    labels:
      severity: page
```

Trình quản lý cảnh báo (alert manager) lấy các cấu hình cảnh báo từ bộ nhớ đệm. Dựa trên các quy tắc cấu hình, nó cũng gọi dịch vụ truy vấn theo một khoảng thời gian xác định trước.
Nếu một quy tắc được thỏa mãn, một sự kiện cảnh báo sẽ được tạo ra.

Các trách nhiệm khác của trình quản lý cảnh báo là:
 - Lọc, gộp và loại bỏ các cảnh báo trùng lặp. Ví dụ: nếu cảnh báo của một phiên bản duy nhất được kích hoạt nhiều lần, chỉ một sự kiện cảnh báo được tạo ra.
 - Kiểm soát truy cập - điều quan trọng là hạn chế các hoạt động quản lý cảnh báo chỉ cho một số cá nhân nhất định.
 - Thử lại (Retry) - trình quản lý đảm bảo rằng cảnh báo được truyền đi ít nhất một lần.

Kho lưu trữ cảnh báo (alert store) là một cơ sở dữ liệu key-value, như Cassandra, giữ trạng thái của tất cả các cảnh báo. Nó đảm bảo một thông báo được gửi đi ít nhất một lần.
Khi một cảnh báo được kích hoạt, nó được xuất bản lên Kafka.

Cuối cùng, các consumer cảnh báo sẽ lấy dữ liệu cảnh báo từ Kafka và gửi thông báo qua các kênh khác nhau - Email, tin nhắn văn bản, PagerDuty, webhooks.

Trong thực tế, có nhiều giải pháp có sẵn (off-the-shelf) cho các hệ thống cảnh báo. Rất khó để biện minh cho việc tự xây dựng hệ thống nội bộ của riêng bạn.

### **Hệ thống trực quan hóa**
Hệ thống trực quan hóa hiển thị các chỉ số và cảnh báo qua một khoảng thời gian. Đây là một dashboard được xây dựng bằng Grafana:

<div style="margin-left:3rem">
    <img src="./images/grafana-dashboard.png" alt="grafana-dashboard" width="500" />
</div>

Một hệ thống trực quan hóa chất lượng cao rất khó để xây dựng. Thật khó để biện minh cho việc không sử dụng một giải pháp có sẵn như Grafana.

---

## Bước 4: Tổng kết
Đây là thiết kế cuối cùng của chúng ta:

<div style="margin-left:3rem">
    <img src="./images/final-design.png" alt="final-design" width="500" />
</div>
