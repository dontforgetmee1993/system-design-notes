# Chương 21: Tổng hợp sự kiện lượt nhấp quảng cáo

## Giới thiệu
**Quảng cáo kỹ thuật số** là một ngành công nghiệp lớn với sự trỗi dậy của Facebook, YouTube, TikTok, v.v.

Do đó, việc theo dõi các sự kiện lượt nhấp quảng cáo (ad click events) là rất quan trọng. Trong chương này, chúng ta khám phá cách thiết kế một hệ thống **tổng hợp sự kiện lượt nhấp quảng cáo** ở quy mô của Facebook/Google.

Quảng cáo kỹ thuật số có một quy trình gọi là **đấu thầu thời gian thực (real-time bidding - RTB)**, nơi không gian quảng cáo kỹ thuật số được mua và bán:

<div style="margin-left:3rem">
    <img src="./images/digital-advertising-example.png" alt="digital-advertising-example" width="500" />
</div>

Tốc độ của RTB rất quan trọng vì nó thường diễn ra trong vòng một giây.
Độ chính xác của dữ liệu cũng rất quan trọng vì nó ảnh hưởng đến số tiền mà các nhà quảng cáo phải trả.

Dựa trên việc tổng hợp các sự kiện lượt nhấp quảng cáo, các nhà quảng cáo có thể đưa ra các quyết định như điều chỉnh đối tượng mục tiêu và từ khóa.

---

## Bước 1: Hiểu vấn đề và Thiết lập Phạm vi Thiết kế
 - C: Định dạng của dữ liệu đầu vào là gì?
 - I: 1 tỷ lượt nhấp quảng cáo mỗi ngày và tổng cộng 2 triệu quảng cáo. Số lượng sự kiện lượt nhấp quảng cáo tăng trưởng 30% mỗi năm.
 - C: Một số truy vấn quan trọng nhất mà hệ thống của chúng ta cần hỗ trợ là gì?
 - I: Các truy vấn hàng đầu cần được xem xét:
   - Trả về số lượng sự kiện lượt nhấp cho quảng cáo X trong Y phút qua.
   - Trả về top 100 quảng cáo được nhấp nhiều nhất trong 1 phút qua. Cả hai tham số đều có thể cấu hình được. Việc tổng hợp diễn ra mỗi phút.
   - Hỗ trợ lọc dữ liệu theo `ip`, `user_id`, `country` cho các truy vấn trên.
 - C: Chúng ta có cần lo lắng về các trường hợp biên không? Một số trường hợp tôi có thể nghĩ ra:
   - Có thể có các sự kiện đến muộn hơn dự kiến.
   - Có thể có các sự kiện bị trùng lặp.
   - Các phần khác nhau của hệ thống có thể bị sập, vì vậy chúng ta cần xem xét việc phục hồi hệ thống.
 - I: Đó là một danh sách tốt, hãy cân nhắc những điều đó.
 - C: Yêu cầu về độ trễ là gì?
 - I: Độ trễ e2e vài phút cho việc tổng hợp lượt nhấp quảng cáo. Đối với RTB, nó ít hơn một giây. Có thể chấp nhận độ trễ đó cho việc tổng hợp lượt nhấp quảng cáo vì chúng thường được sử dụng cho việc thanh toán và báo cáo.

### **Yêu cầu chức năng**
 - Tổng hợp số lượng lượt nhấp của `ad_id` trong Y phút qua.
 - Trả về top 100 `ad_id` được nhấp nhiều nhất mỗi phút.
 - Hỗ trợ lọc tổng hợp theo các thuộc tính khác nhau.
 - Khối lượng tập dữ liệu ở quy mô của Facebook hoặc Google.

### **Yêu cầu phi chức năng**
 - Tính chính xác của kết quả tổng hợp là quan trọng vì nó được sử dụng cho RTB và thanh toán quảng cáo.
 - Xử lý đúng cách các sự kiện bị chậm hoặc bị trùng lặp.
 - Tính mạnh mẽ (Robustness) - hệ thống phải có khả năng phục hồi sau các lỗi cục bộ.
 - Độ trễ - tối đa vài phút độ trễ e2e.

### **Ước tính sơ bộ**
 - 1 tỷ DAU.
 - Giả sử người dùng nhấp vào 1 quảng cáo mỗi ngày -> 1 tỷ lượt nhấp quảng cáo mỗi ngày.
 - QPS lượt nhấp quảng cáo = 10,000.
 - QPS đỉnh cao gấp 5 lần con số đó = 50,000.
 - Một lượt nhấp quảng cáo duy nhất chiếm 0.1KB lưu trữ. Yêu cầu lưu trữ hàng ngày là 100GB.
 - Lưu trữ hàng tháng = 3TB.

---

## Bước 2: Đề xuất Thiết kế Mức cao và Đạt được sự Thống nhất
Trong phần này, chúng ta thảo luận về thiết kế query API, mô hình dữ liệu và thiết kế mức cao.

### **Thiết kế Query API**
API là một hợp đồng giữa client và server. Trong trường hợp của chúng ta, client là người dùng dashboard - nhà khoa học dữ liệu/nhà phân tích, nhà quảng cáo, v.v.

Đây là các yêu cầu chức năng của chúng ta:
 - Tổng hợp số lượng lượt nhấp của `ad_id` trong M phút qua.
 - Trả về top N quảng cáo được nhấp nhiều nhất trong M phút qua.
 - Hỗ trợ lọc tổng hợp theo các thuộc tính khác nhau.

Chúng ta cần hai endpoint để đạt được các yêu cầu đó. Việc lọc có thể được thực hiện thông qua các tham số truy vấn trên một trong số chúng.

**Tổng hợp số lượng lượt nhấp của ad_id trong M phút qua**:

```
GET /v1/ads/{:ad_id}/aggregated_count
```

Tham số truy vấn:
 - from - phút bắt đầu. Mặc định là hiện tại - 1 phút.
 - to - phút kết thúc. Mặc định là hiện tại.
 - filter - mã định danh cho các chiến lược lọc khác nhau. Ví dụ: 001 có nghĩa là "lượt nhấp ngoài Hoa Kỳ".

Phản hồi:
 - ad_id - mã định danh quảng cáo.
 - count - số lượng tổng hợp giữa phút bắt đầu và kết thúc.

**Trả về top N ad_id được nhấp nhiều nhất trong M phút qua**

```
GET /v1/ads/popular_ads
```

Tham số truy vấn:
 - count - top N quảng cáo được nhấp nhiều nhất.
 - window - kích thước cửa sổ tổng hợp tính bằng phút.
 - filter - mã định danh cho các chiến lược lọc khác nhau.

Phản hồi:
 - danh sách các ad_id.

### **Mô hình dữ liệu**
Trong hệ thống của chúng ta, chúng ta có dữ liệu thô và dữ liệu đã tổng hợp.

Dữ liệu thô trông như thế nào:

```
[AdClickEvent] ad001, 2021-01-01 00:00:01, user 1, 207.148.22.22, USA
```

Đây là một ví dụ ở định dạng có cấu trúc:
| ad_id | click_timestamp     | user  | ip            | country |
|-------|---------------------|-------|---------------|---------|
| ad001 | 2021-01-01 00:00:01 | user1 | 207.148.22.22 | USA     |
| ad001 | 2021-01-01 00:00:02 | user1 | 207.148.22.22 | USA     |
| ad002 | 2021-01-01 00:00:02 | user2 | 209.153.56.11 | USA     |

Đây là phiên bản đã tổng hợp:
| ad_id | click_minute | filter_id | count |
|-------|--------------|-----------|-------|
| ad001 | 202101010000 | 0012      | 2     |
| ad001 | 202101010000 | 0023      | 3     |
| ad001 | 202101010001 | 0012      | 1     |
| ad001 | 202101010001 | 0023      | 6     |

`filter_id` giúp chúng ta đạt được các yêu cầu lọc.
| filter_id | region | IP        | user_id |
|-----------|--------|-----------|---------|
| 0012      | US     | *         | *       |
| 0013      | *      | 123.1.2.3 | *       |

Để hỗ trợ việc trả về nhanh chóng top N quảng cáo được nhấp nhiều nhất trong M phút qua, chúng ta cũng sẽ duy trì cấu trúc này:
| most_clicked_ads   |           |                                                  |
|--------------------|-----------|--------------------------------------------------|
| window_size        | integer   | Kích thước cửa sổ tổng hợp (M) tính bằng phút |
| update_time_minute | timestamp | Mốc thời gian cập nhật lần cuối (độ phân giải 1 phút) |
| most_clicked_ads   | array     | Danh sách các ad ID ở định dạng JSON. |

Một số ưu và nhược điểm giữa việc lưu trữ dữ liệu thô và lưu trữ dữ liệu đã tổng hợp là gì?
 - Dữ liệu thô cho phép sử dụng toàn bộ tập dữ liệu và hỗ trợ lọc dữ liệu và tính toán lại.
 - Mặt khác, dữ liệu đã tổng hợp cho phép chúng ta có một tập dữ liệu nhỏ hơn và truy vấn nhanh hơn.
 - Dữ liệu thô đồng nghĩa với việc có một kho lưu trữ dữ liệu lớn hơn và truy vấn chậm hơn.
 - Dữ liệu đã tổng hợp, tuy nhiên, là dữ liệu phái sinh, do đó có sự mất mát dữ liệu.

Trong thiết kế của chúng ta, chúng ta sẽ sử dụng kết hợp cả hai cách tiếp cận:
 - Giữ lại dữ liệu thô để phục vụ mục đích gỡ lỗi là một ý tưởng hay. Nếu có lỗi trong quá trình tổng hợp, chúng ta có thể phát hiện lỗi và tính toán lại (backfill).
 - Dữ liệu đã tổng hợp cũng nên được lưu trữ để có hiệu suất truy vấn nhanh hơn.
 - Dữ liệu thô có thể được lưu trữ trong kho lạnh (cold storage) để tránh thêm chi phí lưu trữ.

Khi nói đến cơ sở dữ liệu, có một số yếu tố cần xem xét:
 - Dữ liệu trông như thế nào? Nó là quan hệ (relational), tài liệu (document) hay blob?
 - Khối lượng công việc là đọc nhiều (read-heavy), ghi nhiều (write-heavy) hay cả hai?
 - Có cần các giao dịch (transactions) không?
 - Các truy vấn có dựa trên các hàm OLAP như SUM và COUNT không?

Đối với dữ liệu thô, chúng ta có thể thấy rằng QPS trung bình là 10k và QPS đỉnh cao là 50k, vì vậy hệ thống là ghi nhiều.
Mặt khác, lưu lượng đọc thấp vì dữ liệu thô chủ yếu được sử dụng làm bản sao dự phòng nếu có sự cố xảy ra.

Các cơ sở dữ liệu quan hệ có thể thực hiện công việc này, nhưng việc mở rộng quy mô ghi có thể là một thách thức.
Ngoài ra, chúng ta có thể sử dụng Cassandra hoặc InfluxDB, những hệ thống này có sự hỗ trợ bản địa tốt hơn cho khối lượng ghi lớn.

Một lựa chọn khác là sử dụng Amazon S3 với định dạng dữ liệu dạng cột (columnar) như ORC, Parquet hoặc AVRO. Vì thiết lập này không quen thuộc, chúng ta sẽ chọn Cassandra.

Đối với dữ liệu đã tổng hợp, khối lượng công việc là cả đọc và ghi nhiều vì dữ liệu đã tổng hợp liên tục được truy vấn cho các dashboard và cảnh báo.
Nó cũng ghi nhiều vì dữ liệu được tổng hợp và ghi lại mỗi phút bởi dịch vụ tổng hợp.
Do đó, chúng ta cũng sẽ sử dụng cùng một kho lưu trữ dữ liệu (Cassandra) ở đây.

### **Thiết kế mức cao**
Đây là cách hệ thống của chúng ta trông như thế nào:

<div style="margin-left:3rem">
    <img src="./images/high-level-design-1.png" alt="high-level-design-1" width="500" />
</div>

Dữ liệu chảy như một luồng dữ liệu không giới hạn trên cả đầu vào và đầu ra.

Để tránh việc có một bồn chứa đồng bộ (synchronous sink), nơi một consumer bị sập có thể khiến toàn bộ hệ thống bị đình trệ,
chúng ta sẽ tận dụng xử lý không đồng bộ bằng cách sử dụng hàng đợi tin nhắn (Kafka) để tách biệt các consumer và producer.

<div style="margin-left:3rem">
    <img src="./images/high-level-design-2.png" alt="high-level-design-2" width="500" />
</div>

Hàng đợi tin nhắn đầu tiên lưu trữ dữ liệu sự kiện lượt nhấp quảng cáo:
| ad_id | click_timestamp | user_id | ip | country |

Hàng đợi tin nhắn thứ hai chứa số lượng lượt nhấp quảng cáo, được tổng hợp mỗi phút:
| ad_id | click_minute | count |

Cũng như top N quảng cáo được nhấp nhiều nhất được tổng hợp mỗi phút:
| update_time_minute | most_clicked_ads |

Hàng đợi tin nhắn thứ hai có ở đó để đạt được ngữ nghĩa atomic commit e2e chính xác một lần (exactly-once):

<div style="margin-left:3rem">
    <img src="./images/atomic-commit.png" alt="atomic-commit" width="500" />
</div>

Đối với dịch vụ tổng hợp, sử dụng framework MapReduce là một lựa chọn tốt:

<div style="margin-left:3rem">
    <img src="./images/ad-count-map-reduce.png" alt="ad-count-map-reduce" width="500" />
</div>

<div style="margin-left:3rem">
    <img src="./images/top-100-map-reduce.png" alt="top-100-map-reduce" width="500" />
</div>

Mỗi node chịu trách nhiệm cho một nhiệm vụ duy nhất và nó gửi kết quả xử lý cho node hạ nguồn.

Node map chịu trách nhiệm đọc từ nguồn dữ liệu, sau đó lọc và chuyển đổi dữ liệu.

Ví dụ, node map có thể phân bổ dữ liệu qua các node tổng hợp khác nhau dựa trên `ad_id`:

<div style="margin-left:3rem">
    <img src="./images/map-node.png" alt="map-node" width="500" />
</div>

Ngoài ra, chúng ta có thể phân phối quảng cáo qua các phân vùng Kafka và để các node tổng hợp đăng ký trực tiếp trong một nhóm consumer.
Tuy nhiên, node mapping cho phép chúng ta làm sạch hoặc chuyển đổi dữ liệu trước khi xử lý tiếp theo.

Một lý do khác có thể là chúng ta không có quyền kiểm soát cách dữ liệu được tạo ra,
vì vậy các sự kiện liên quan đến cùng một `ad_id` có thể đi vào các phân vùng khác nhau.

Node tổng hợp (aggregate node) đếm các sự kiện lượt nhấp quảng cáo theo `ad_id` trong bộ nhớ mỗi phút.

Node reduce thu thập kết quả tổng hợp từ node aggregate và tạo ra kết quả cuối cùng:

<div style="margin-left:3rem">
    <img src="./images/reduce-node.png" alt="reduce-node" width="500" />
</div>

Mô hình DAG này sử dụng mô hình MapReduce. Nó lấy dữ liệu lớn và tận dụng tính toán phân tán song song để biến nó thành dữ liệu có kích thước bình thường.

Trong mô hình DAG, dữ liệu trung gian được lưu trữ trong bộ nhớ và các node khác nhau giao tiếp với nhau bằng TCP hoặc bộ nhớ dùng chung.

Hãy khám phá cách mô hình này có thể giúp chúng ta đạt được các trường hợp sử dụng khác nhau.

**Trường hợp sử dụng 1 - tổng hợp số lượng lượt nhấp**:

<div style="margin-left:3rem">
    <img src="./images/use-case-1.png" alt="use-case-1" width="500" />
</div>

 - Các quảng cáo được phân vùng bằng cách sử dụng `ad_id % 3`.

**Trường hợp sử dụng 2 - trả về top N quảng cáo được nhấp nhiều nhất**:

<div style="margin-left:3rem">
    <img src="./images/use-case-2.png" alt="use-case-2" width="500" />
</div>

 - Trong trường hợp này, chúng ta đang tổng hợp top 3 quảng cáo, nhưng điều này có thể dễ dàng mở rộng thành top N quảng cáo.
 - Mỗi node duy trì một cấu trúc dữ liệu heap để truy xuất nhanh chóng top N quảng cáo.

**Trường hợp sử dụng 3 - lọc dữ liệu**:
Để hỗ trợ lọc dữ liệu nhanh chóng, chúng ta có thể xác định trước các tiêu chí lọc và tổng hợp trước dựa trên đó:
| ad_id | click_minute | country | count |
|-------|--------------|---------|-------|
| ad001 | 202101010001 | USA     | 100   |
| ad001 | 202101010001 | GPB     | 200   |
| ad001 | 202101010001 | others  | 3000  |
| ad002 | 202101010001 | USA     | 10    |
| ad002 | 202101010001 | GPB     | 25    |
| ad002 | 202101010001 | others  | 12    |

Kỹ thuật này được gọi là **star schema** (mô hình sao) và được sử dụng rộng rãi trong các kho dữ liệu.
Các trường lọc được gọi là các **chiều (dimensions)**.

Cách tiếp cận này có các lợi ích sau:
 - Đơn giản để hiểu và xây dựng.
 - Dịch vụ tổng hợp hiện tại có thể được tái sử dụng để tạo thêm nhiều chiều trong star schema.
 - Việc truy cập dữ liệu dựa trên tiêu chí lọc rất nhanh vì các kết quả đã được tính toán trước.

Một hạn chế của cách tiếp cận này là nó tạo ra nhiều bucket và bản ghi hơn, đặc biệt là khi chúng ta có nhiều tiêu chí lọc.

---

## Bước 3: Thiết kế Chi tiết
Hãy đi sâu hơn vào một số chủ đề thú vị hơn.

### **Streaming vs. Batching**
Kiến trúc mức cao mà chúng ta đã đề xuất là một loại hệ thống xử lý luồng (stream processing).
Dưới đây là so sánh giữa ba loại hệ thống:
| | Dịch vụ (Hệ thống trực tuyến) | Hệ thống lô (Hệ thống ngoại tuyến) | Hệ thống luồng (Hệ thống gần thời gian thực) |
|-------------------------|-------------------------------|--------------------------------------------------------|----------------------------------------------|
| Khả năng phản hồi | Phản hồi client nhanh chóng | Không cần phản hồi client | Không cần phản hồi client |
| Đầu vào | Các yêu cầu của người dùng | Đầu vào có giới hạn với kích thước hữu hạn. Một lượng lớn dữ liệu | Đầu vào không có ranh giới (luồng vô hạn) |
| Đầu ra | Phản hồi cho client | Materialized views, chỉ số tổng hợp, v.v. | Materialized views, chỉ số tổng hợp, v.v. |
| Đo lường hiệu suất | Tính sẵn sàng, độ trễ | Thông lượng (Throughput) | Thông lượng, độ trễ |
| Ví dụ | Mua sắm trực tuyến | MapReduce | Flink [13] |

Trong thiết kế của mình, chúng ta đã sử dụng sự kết hợp giữa xử lý lô và xử lý luồng.

Chúng ta sử dụng xử lý luồng để xử lý dữ liệu khi nó đến và tạo ra các kết quả tổng hợp trong thời gian gần thực.
Mặt khác, chúng ta sử dụng xử lý lô để sao lưu dữ liệu lịch sử.

Một hệ thống chứa đồng thời hai lộ trình xử lý - lô và luồng, kiến trúc này được gọi là lambda.
Một nhược điểm là bạn có hai lộ trình xử lý với hai codebase khác nhau để duy trì.

Kappa là một kiến trúc thay thế, kết hợp xử lý lô và luồng trong một lộ trình xử lý duy nhất.
Ý tưởng chính là sử dụng một công cụ xử lý luồng duy nhất.

Kiến trúc Lambda:

<div style="margin-left:3rem">
    <img src="./images/lambda-architecture.png" alt="lambda-architecture" width="500" />
</div>

Kiến trúc Kappa:

<div style="margin-left:3rem">
    <img src="./images/kappa-architecture.png" alt="kappa-architecture" width="500" />
</div>

Thiết kế mức cao của chúng ta sử dụng kiến trúc Kappa vì việc xử lý lại dữ liệu lịch sử cũng đi qua dịch vụ tổng hợp.

Bất cứ khi nào chúng ta phải tính toán lại dữ liệu tổng hợp do ví dụ một lỗi nghiêm trọng trong logic tổng hợp, chúng ta có thể tính toán lại việc tổng hợp từ dữ liệu thô mà chúng ta lưu trữ.
 - Dịch vụ tính toán lại truy xuất dữ liệu từ kho lưu trữ thô. Đây là một công việc theo lô.
 - Dữ liệu được truy xuất được gửi đến một dịch vụ tổng hợp chuyên dụng, để dịch vụ tổng hợp xử lý thời gian thực không bị ảnh hưởng.
 - Kết quả tổng hợp được gửi đến hàng đợi tin nhắn thứ hai, sau đó chúng ta cập nhật kết quả trong cơ sở dữ liệu tổng hợp.

<div style="margin-left:3rem">
    <img src="./images/recalculation-example.png" alt="recalculation-example" width="500" />
</div>

### **Thời gian**
Chúng ta cần một mốc thời gian (timestamp) để thực hiện việc tổng hợp. Nó có thể được tạo ra ở hai nơi:
 - Thời gian sự kiện (event time) - khi lượt nhấp quảng cáo xảy ra.
 - Thời gian xử lý (processing time) - thời gian hệ thống khi máy chủ xử lý sự kiện.

Do việc sử dụng xử lý không đồng bộ (hàng đợi tin nhắn) và sự chậm trễ của mạng, có thể có sự khác biệt đáng kể giữa thời gian sự kiện và thời gian xử lý.
 - Nếu chúng ta sử dụng thời gian xử lý, kết quả tổng hợp có thể không chính xác.
 - Nếu chúng ta sử dụng thời gian sự kiện, chúng ta phải đối mặt với các sự kiện bị chậm.

Không có giải pháp hoàn hảo, chúng ta cần xem xét các sự đánh đổi:
| | Ưu điểm | Nhược điểm |
|-----------------|---------------------------------------|--------------------------------------------------------------------------------------|
| Thời gian sự kiện | Kết quả tổng hợp chính xác hơn | Client có thể có thời gian sai hoặc mốc thời gian có thể được tạo bởi người dùng xấu |
| Thời gian xử lý | Mốc thời gian của máy chủ đáng tin cậy hơn | Mốc thời gian không chính xác nếu sự kiện bị chậm |

Vì độ chính xác của dữ liệu là quan trọng, chúng ta sẽ sử dụng thời gian sự kiện để tổng hợp.

Để giảm thiểu vấn đề các sự kiện bị chậm, một kỹ thuật gọi là "watermark" (hình mờ) có thể được tận dụng.

Trong ví dụ bên dưới, sự kiện 2 bỏ lỡ cửa sổ mà nó cần được tổng hợp:

<div style="margin-left:3rem">
    <img src="./images/watermark-technique.png" alt="watermark-technique" width="500" />
</div>

Tuy nhiên, nếu chúng ta cố ý mở rộng cửa sổ tổng hợp, chúng ta có thể giảm khả năng bị bỏ lỡ các sự kiện.
Phần mở rộng của một cửa sổ được gọi là "watermark":

<div style="margin-left:3rem">
    <img src="./images/watermark-2.png" alt="watermark-2" width="500" />
</div>

 - Watermark ngắn làm tăng khả năng bỏ lỡ sự kiện, nhưng làm giảm độ trễ.
 - Watermark dài hơn làm giảm khả năng bỏ lỡ sự kiện, nhưng làm tăng độ trễ.

Luôn có khả năng xảy ra các sự kiện bị bỏ lỡ, bất kể kích thước của watermark. Nhưng không có ích gì khi tối ưu hóa cho các sự kiện có xác suất thấp như vậy.

Thay vào đó, chúng ta có thể giải quyết sự mâu thuẫn đó bằng cách thực hiện đối soát (reconciliation) vào cuối ngày.

### **Cửa sổ tổng hợp**
Có bốn loại hàm cửa sổ:
 - Cửa sổ Tumbling (cố định)
 - Cửa sổ Hopping
 - Cửa sổ Sliding
 - Cửa sổ Session

Trong thiết kế của mình, chúng ta tận dụng cửa sổ tumbling cho việc tổng hợp lượt nhấp quảng cáo:

<div style="margin-left:3rem">
    <img src="./images/tumbling-window.png" alt="tumbling-window" width="500" />
</div>

Cũng như một cửa sổ sliding cho việc tổng hợp top N quảng cáo được nhấp nhiều nhất trong M phút:

<div style="margin-left:3rem">
    <img src="./images/sliding-window.png" alt="sliding-window" width="500" />
</div>

### **Cam kết phân phối**
Vì dữ liệu chúng ta đang tổng hợp sẽ được sử dụng để thanh toán, độ chính xác của dữ liệu là ưu tiên hàng đầu.

Do đó, chúng ta cần thảo luận về:
 - Cách tránh xử lý các sự kiện bị trùng lặp.
 - Cách đảm bảo tất cả các sự kiện đều được xử lý.

Có ba cam kết phân phối mà chúng ta có thể sử dụng - tối đa một lần (at-most-once), ít nhất một lần (at-least-once) và chính xác một lần (exactly-once).

Trong hầu hết các trường hợp, at-least-once là đủ khi một lượng nhỏ trùng lặp có thể chấp nhận được.
Tuy nhiên, điều này không đúng với hệ thống của chúng ta, vì một sự khác biệt nhỏ về phần trăm có thể dẫn đến sự sai lệch hàng triệu đô la.
Do đó, chúng ta sẽ cần sử dụng ngữ nghĩa phân phối exactly-once.

### **Loại bỏ dữ liệu trùng lặp**
Một trong những vấn đề chất lượng dữ liệu phổ biến nhất là dữ liệu bị trùng lặp.

Nó có thể đến từ nhiều nguồn khác nhau:
 - Phía client - một client có thể gửi lại cùng một sự kiện nhiều lần. Các sự kiện trùng lặp được gửi với ý đồ xấu nên được xử lý bởi một công cụ rủi ro (risk engine).
 - Sự cố máy chủ - Một node dịch vụ tổng hợp bị sập giữa chừng khi đang tổng hợp và dịch vụ thượng nguồn chưa nhận được xác nhận nên sự kiện được gửi lại.

Đây là một ví dụ về việc trùng lặp dữ liệu xảy ra do không xác nhận được một sự kiện ở bước cuối cùng:

<div style="margin-left:3rem">
    <img src="./images/data-duplication-example.png" alt="data-duplication-example" width="500" />
</div>

Trong ví dụ này, offset 100 sẽ được xử lý và gửi xuống hạ nguồn nhiều lần.

Một tùy chọn để cố gắng giảm thiểu điều này là lưu trữ offset đã thấy lần cuối trong HDFS/S3, nhưng điều này có rủi ro là kết quả không bao giờ đến được hạ nguồn:

<div style="margin-left:3rem">
    <img src="./images/data-duplication-example-2.png" alt="data-duplication-example-2" width="500" />
</div>

Cuối cùng, chúng ta có thể lưu trữ offset trong khi tương tác với hạ nguồn một cách atomic. Để đạt được điều này, chúng ta cần triển khai một giao dịch phân tán (distributed transaction):

<div style="margin-left:3rem">
    <img src="./images/data-duplication-example-3.png" alt="data-duplication-example-3" width="500" />
</div>

**Lưu ý cá nhân**: Ngoài ra, nếu hệ thống hạ nguồn xử lý kết quả tổng hợp một cách có tính lũy đẳng (idempotently), thì không cần giao dịch phân tán.

### **Mở rộng quy mô hệ thống**
Hãy thảo luận về cách chúng ta mở rộng quy mô hệ thống khi nó phát triển.

Chúng ta có ba thành phần độc lập - hàng đợi tin nhắn, dịch vụ tổng hợp và cơ sở dữ liệu.
Vì chúng được tách biệt, chúng ta có thể mở rộng quy mô của chúng một cách độc lập.

Cách chúng ta mở rộng quy mô hàng đợi tin nhắn:
 - Chúng ta không đặt giới hạn cho các producer, vì vậy chúng có thể được mở rộng dễ dàng.
 - Các consumer có thể được mở rộng bằng cách gán chúng vào các nhóm consumer và tăng số lượng consumer.
 - Để điều này hoạt động, chúng ta cũng cần đảm bảo có đủ các phân vùng được tạo sẵn.
 - Ngoài ra, việc tái cân bằng consumer (consumer rebalancing) có thể mất một thời gian khi có hàng nghìn consumer, vì vậy khuyến nghị nên thực hiện vào những giờ thấp điểm.
 - Chúng ta cũng có thể xem xét phân vùng topic theo địa lý, ví dụ `topic_na`, `topic_eu`, v.v.

<div style="margin-left:3rem">
    <img src="./images/scale-consumers.png" alt="scale-consumers" width="500" />
</div>

Cách chúng ta mở rộng dịch vụ tổng hợp:

<div style="margin-left:3rem">
    <img src="./images/aggregation-service-scaling.png" alt="aggregation-service-scaling" width="500" />
</div>

 - Các node map-reduce có thể dễ dàng được mở rộng bằng cách thêm nhiều node hơn.
 - Thông lượng của dịch vụ tổng hợp có thể được mở rộng bằng cách sử dụng đa luồng (multi-threading).
 - Ngoài ra, chúng ta có thể tận dụng các nhà cung cấp tài nguyên như Apache YARN để sử dụng đa tiến trình (multi-processing).
 - Tùy chọn 1 dễ dàng hơn, nhưng tùy chọn 2 được sử dụng rộng rãi hơn trong thực tế vì nó có khả năng mở rộng cao hơn.
 - Đây là ví dụ về đa luồng:

<div style="margin-left:3rem">
    <img src="./images/multi-threading-example.png" alt="multi-threading-example" width="500" />
</div>

Cách chúng ta mở rộng cơ sở dữ liệu:
 - Nếu chúng ta sử dụng Cassandra, nó hỗ trợ bản địa việc mở rộng theo chiều ngang sử dụng băm nhất quán.
 - Nếu một node mới được thêm vào cluster, dữ liệu sẽ tự động được tái cân bằng qua tất cả các (virtual) node.
 - Với cách tiếp cận này, không yêu cầu phân mảnh (re-sharding) thủ công.

<div style="margin-left:3rem">
    <img src="./images/cassandra-scalability.png" alt="cassandra-scalability" width="500" />
</div>

Một vấn đề khả năng mở rộng khác cần xem xét là vấn đề hotspot (điểm nóng) - điều gì sẽ xảy ra nếu một quảng cáo phổ biến hơn và nhận được nhiều sự chú ý hơn những quảng cáo khác?

<div style="margin-left:3rem">
    <img src="./images/hotspot-issue.png" alt="hotspot-issue" width="500" />
</div>

 - Trong ví dụ trên, các node của dịch vụ tổng hợp có thể yêu cầu thêm tài nguyên thông qua trình quản lý tài nguyên (resource manager).
 - Trình quản lý tài nguyên phân bổ thêm tài nguyên, để node ban đầu không bị quá tải.
 - Node ban đầu chia nhỏ các sự kiện thành 3 nhóm và mỗi node tổng hợp xử lý 100 sự kiện.
 - Kết quả được ghi lại về node tổng hợp ban đầu.

Các cách thay thế tinh vi hơn để xử lý vấn đề hotspot:
 - Tổng hợp Global-Local (Global-Local Aggregation)
 - Tổng hợp Split Distinct (Split Distinct Aggregation)

### **Khả năng chịu lỗi**
Bên trong các node tổng hợp, chúng ta đang xử lý dữ liệu trong bộ nhớ. Nếu một node bị sập, dữ liệu đang xử lý sẽ bị mất.

Chúng ta có thể tận dụng consumer offsets trong Kafka để tiếp tục từ nơi chúng ta đã dừng lại khi một node khác tiếp nhận công việc.
Tuy nhiên, có trạng thái trung gian bổ sung mà chúng ta cần duy trì, vì chúng ta đang tổng hợp top N quảng cáo trong M phút.

Chúng ta có thể tạo các bản sao nhanh (snapshots) tại một phút cụ thể cho việc tổng hợp đang diễn ra:

<div style="margin-left:3rem">
    <img src="./images/fault-tolerance-example.png" alt="fault-tolerance-example" width="500" />
</div>

Nếu một node bị sập, node mới có thể đọc consumer offset đã commit mới nhất, cũng như snapshot mới nhất để tiếp tục công việc:

<div style="margin-left:3rem">
    <img src="./images/fault-tolerance-recovery-example.png" alt="fault-tolerance-recovery-example" width="500" />
</div>

### **Giám sát dữ liệu và tính chính xác**
Vì dữ liệu chúng ta đang tổng hợp là rất quan trọng vì nó được sử dụng để thanh toán, điều cực kỳ quan trọng là phải có sự giám sát nghiêm ngặt tại chỗ để đảm bảo tính chính xác.

Một số chỉ số chúng ta có thể muốn giám sát:
 - **Độ trễ (Latency)**: Các mốc thời gian của các sự kiện khác nhau có thể được theo dõi để hiểu độ trễ e2e của hệ thống.
 - **Kích thước hàng đợi tin nhắn**: Nếu có sự gia tăng đột ngột về kích thước hàng đợi, chúng ta cần thêm các node tổng hợp. Vì Kafka được triển khai thông qua một log commit phân tán, chúng ta cần theo dõi các chỉ số records-lag thay thế.
 - **Tài nguyên hệ thống trên các node tổng hợp**: CPU, đĩa, JVM, v.v.

Chúng ta cũng cần triển khai một luồng đối soát (reconciliation flow), đây là một công việc theo lô, chạy vào cuối ngày.
Nó tính toán các kết quả tổng hợp từ dữ liệu thô và so sánh chúng với dữ liệu thực tế được lưu trữ trong cơ sở dữ liệu tổng hợp:

<div style="margin-left:3rem">
    <img src="./images/reconciliation-flow.png" alt="reconciliation-flow" width="500" />
</div>

### **Thiết kế thay thế**
Trong một cuộc phỏng vấn thiết kế hệ thống tổng quát, bạn không được kỳ vọng sẽ biết các chi tiết bên trong của các phần mềm chuyên dụng được sử dụng trong xử lý dữ liệu lớn.

Giải thích quá trình suy nghĩ và thảo luận về các sự đánh đổi quan trọng hơn là việc biết các công cụ cụ thể, đó là lý do tại sao chương này bao gồm một giải pháp tổng quát.

Một thiết kế thay thế, tận dụng các công cụ có sẵn, là lưu trữ dữ liệu lượt nhấp quảng cáo trong Hive với một lớp ElasticSearch bên trên được xây dựng để truy vấn nhanh hơn.

Việc tổng hợp thường được thực hiện trong các cơ sở dữ liệu OLAP như ClickHouse hoặc Druid.

<div style="margin-left:3rem">
    <img src="./images/alternative-design.png" alt="alternative-design" width="500" />
</div>

---

## Bước 4: Tổng kết
Những điều chúng ta đã đề cập:
 - Mô hình dữ liệu và Thiết kế API.
 - Sử dụng MapReduce để tổng hợp các sự kiện lượt nhấp quảng cáo.
 - Mở rộng quy mô hàng đợi tin nhắn, dịch vụ tổng hợp và cơ sở dữ liệu.
 - Giảm thiểu vấn đề hotspot.
 - Giám sát hệ thống liên tục.
 - Sử dụng đối soát để đảm bảo tính chính xác.
 - Khả năng chịu lỗi.

Việc tổng hợp sự kiện lượt nhấp quảng cáo là một hệ thống xử lý dữ liệu lớn điển hình.

Sẽ dễ dàng hơn để hiểu và thiết kế nó nếu bạn có kiến thức trước về các công nghệ liên quan:
 - Apache Kafka
 - Apache Spark
 - Apache Flink
