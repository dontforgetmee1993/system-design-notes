# Chương 25: Bảng xếp hạng Trò chơi Thời gian thực

## Giới thiệu

Chúng ta sẽ thiết kế một **bảng xếp hạng** (leaderboard) cho một trò chơi di động trực tuyến:

<div style="margin-left:3rem">
    <img src="./images/leaderboard.png" alt="leaderboard" width="500" />
</div>

---

## Bước 1: Hiểu vấn đề và Thiết lập Phạm vi Thiết kế

- C: Điểm số cho bảng xếp hạng được tính như thế nào?
- I: Người dùng nhận được một điểm bất cứ khi nào họ thắng một trận đấu.
- C: Tất cả người chơi có được đưa vào bảng xếp hạng không?
- I: Có.
- C: Có phân đoạn thời gian nào liên quan đến bảng xếp hạng không?
- I: Mỗi tháng, một giải đấu mới bắt đầu và một bảng xếp hạng mới sẽ được khởi tạo.
- C: Chúng ta có thể giả định rằng chúng ta chỉ quan tâm đến top 10 người dùng không?
- I: Chúng ta muốn hiển thị top 10 người dùng, cùng với vị trí của một người dùng cụ thể. Nếu thời gian cho phép, chúng ta có thể thảo luận về việc hiển thị những người dùng xung quanh một người dùng cụ thể trên bảng xếp hạng.
- C: Có bao nhiêu người dùng trong một giải đấu?
- I: 5 triệu DAU và 25 triệu MAU.
- C: Trung bình có bao nhiêu trận đấu được chơi trong một giải đấu?
- I: Mỗi người chơi chơi trung bình 10 trận mỗi ngày.
- C: Làm thế nào để xác định thứ hạng nếu hai người chơi có cùng điểm số?
- I: Thứ hạng của họ sẽ giống nhau trong trường hợp đó. Nếu thời gian cho phép, chúng ta có thể thảo luận về việc phân định thứ hạng khi bằng điểm.
- C: Bảng xếp hạng có cần theo thời gian thực không?
- I: Có, chúng ta muốn trình bày kết quả thời gian thực hoặc càng gần thời gian thực càng tốt. Việc trình bày kết quả được xử lý theo lô (batched) là không ổn.

### **Yêu cầu chức năng**

- Hiển thị top 10 người chơi trên bảng xếp hạng.
- Hiển thị thứ hạng cụ thể của một người dùng.
- Hiển thị những người dùng đứng trên và đứng dưới người dùng hiện tại 4 bậc (bonus).

### **Yêu cầu phi chức năng**

- Cập nhật điểm số theo thời gian thực.
- Việc cập nhật điểm số được phản ánh trên bảng xếp hạng trong thời gian thực.
- Khả năng mở rộng, tính sẵn sàng và độ tin cậy nói chung.

### **Ước tính sơ bộ**

Với 50 triệu DAU, nếu trò chơi có sự phân bổ người chơi đồng đều trong khoảng thời gian 24 giờ, chúng ta sẽ có trung bình 50 người dùng mỗi giây.
Tuy nhiên, vì sự phân bổ thường không đều, chúng ta có thể ước tính rằng số lượng người dùng trực tuyến cao điểm sẽ là 250 người dùng mỗi giây.

QPS cho việc người dùng ghi điểm - với trung bình 10 trận đấu mỗi ngày, 50 người dùng/giây * 10 = 500 QPS. QPS đỉnh = 2500.

QPS cho việc lấy top 10 của bảng xếp hạng - giả sử người dùng mở xem trung bình một lần mỗi ngày, QPS là 50.

---

## Bước 2: Đề xuất Thiết kế Mức cao và Đạt được sự Thống nhất

### **Thiết kế API**

API đầu tiên chúng ta cần là API để cập nhật điểm số của người dùng:

```
POST /v1/scores
```

API này nhận hai tham số - `user_id` và số điểm (`points`) đạt được khi thắng một trò chơi.

API này chỉ nên được truy cập bởi các máy chủ trò chơi (game servers), không phải các client cuối.

Tiếp theo là API để lấy top 10 người chơi của bảng xếp hạng:

```
GET /v1/scores
```

Ví dụ phản hồi:

```
{
  "data": [
    {
      "user_id": "user_id1",
      "user_name": "alice",
      "rank": 1,
      "score": 12543
    },
    {
      "user_id": "user_id2",
      "user_name": "bob",
      "rank": 2,
      "score": 11500
    }
  ],
  ...
  "total": 10
}
```

Bạn cũng có thể lấy điểm số của một người dùng cụ thể:

```
GET /v1/scores/{:user_id}
```

Ví dụ phản hồi:

```
{
    "user_info": {
        "user_id": "user5",
        "score": 1000,
        "rank": 6,
    }
}
```

### **Kiến trúc mức cao**

<div style="margin-left:3rem">
    <img src="./images/high-level-architecture.png" alt="high-level-architecture" width="500" />
</div>

- Khi một người chơi thắng một trò chơi, client gửi một yêu cầu đến dịch vụ trò chơi (game service).
- Dịch vụ trò chơi xác thực xem trận thắng có hợp lệ không và gọi dịch vụ bảng xếp hạng (leaderboard service) để cập nhật điểm số của người chơi.
- Dịch vụ bảng xếp hạng cập nhật điểm số của người dùng trong kho lưu trữ bảng xếp hạng.
- Người chơi thực hiện một cuộc gọi đến dịch vụ bảng xếp hạng để lấy dữ liệu bảng xếp hạng, ví dụ top 10 người chơi và thứ hạng của chính người chơi đó.

Một thiết kế thay thế đã được xem xét là client cập nhật điểm số trực tiếp trong dịch vụ bảng xếp hạng:

<div style="margin-left:3rem">
    <img src="./images/alternative-design.png" alt="alternative-design" width="500" />
</div>

Lựa chọn này không an toàn vì nó dễ bị tấn công man-in-the-middle. Người chơi có thể sử dụng proxy và thay đổi điểm số của họ theo ý muốn.

Một lưu ý bổ sung là đối với các trò chơi mà logic trò chơi được quản lý bởi máy chủ, các client không cần gọi máy chủ một cách rõ ràng để ghi lại trận thắng của họ.
Các máy chủ sẽ tự động thực hiện việc đó dựa trên logic trò chơi.

Một cân nhắc bổ sung là liệu chúng ta có nên đặt một hàng đợi tin nhắn giữa máy chủ trò chơi và dịch vụ bảng xếp hạng hay không. Điều này sẽ hữu ích nếu các dịch vụ khác cũng quan tâm đến kết quả trò chơi, nhưng đó không phải là yêu cầu rõ ràng trong cuộc phỏng vấn cho đến nay, vì vậy nó không được đưa vào thiết kế:

<div style="margin-left:3rem">
    <img src="./images/message-queue-based-comm.png" alt="message-queue-based-comm" width="500" />
</div>

### **Mô hình dữ liệu**

Hãy thảo luận về các lựa chọn chúng ta có để lưu trữ dữ liệu bảng xếp hạng - DB quan hệ, Redis, NoSQL.

Giải pháp NoSQL sẽ được thảo luận trong phần đi sâu.

#### Giải pháp cơ sở dữ liệu quan hệ

Nếu quy mô không thành vấn đề và chúng ta không có quá nhiều người dùng, một DB quan hệ phục vụ chúng ta khá tốt.

Chúng ta có thể bắt đầu từ một bảng bảng xếp hạng đơn giản, mỗi tháng một bảng (Lưu ý cá nhân - điều này không hợp lý lắm. Bạn chỉ cần thêm một cột `month` và tránh sự phiền phức khi duy trì các bảng mới mỗi tháng):

<div style="margin-left:3rem">
    <img src="./images/leaderboard-table.png" alt="leaderboard-table" width="500" />
</div>

Có dữ liệu bổ sung cần bao gồm ở đó, nhưng điều đó không liên quan đến các truy vấn chúng ta sẽ chạy, vì vậy nó được bỏ qua.

Điều gì xảy ra khi một người dùng thắng một điểm?

<div style="margin-left:3rem">
    <img src="./images/user-wins-point.png" alt="user-wins-point" width="500" />
</div>

Nếu người dùng chưa tồn tại trong bảng, chúng ta cần chèn họ trước:

```
INSERT INTO leaderboard (user_id, score) VALUES ('mary1934', 1);
```

Trong các cuộc gọi tiếp theo, chúng ta chỉ cần cập nhật điểm số của họ:

```
UPDATE leaderboard set score=score + 1 where user_id='mary1934';
```

Làm thế nào để tìm những người chơi đứng đầu của một bảng xếp hạng?

<div style="margin-left:3rem">
    <img src="./images/find-leaderboard-position.png" alt="find-leaderboard-position" width="500" />
</div>

Chúng ta có thể chạy truy vấn sau:

```
SELECT (@rownum := @rownum + 1) AS rank, user_id, score
FROM leaderboard
ORDER BY score DESC;
```

Tuy nhiên, điều này không hiệu quả vì nó thực hiện quét bảng (table scan) để sắp xếp tất cả các bản ghi trong bảng cơ sở dữ liệu.

Chúng ta có thể tối ưu hóa nó bằng cách thêm một chỉ mục (index) vào `score` và sử dụng hoạt động `LIMIT` để tránh quét mọi thứ:

```
SELECT (@rownum := @rownum + 1) AS rank, user_id, score
FROM leaderboard
ORDER BY score DESC
LIMIT 10;
```

Tuy nhiên, cách tiếp cận này không mở rộng tốt nếu người dùng không ở đầu bảng xếp hạng và bạn muốn xác định thứ hạng của họ.

#### Giải pháp Redis

Chúng ta muốn tìm một giải pháp hoạt động tốt ngay cả với hàng triệu người chơi mà không cần phải nhờ đến các truy vấn cơ sở dữ liệu phức tạp.

Redis là một kho lưu trữ dữ liệu trong bộ nhớ (in-memory), nhanh vì nó hoạt động trong RAM và có một cấu trúc dữ liệu phù hợp để phục vụ nhu cầu của chúng ta - sorted set (tập hợp được sắp xếp).

Một sorted set là một cấu trúc dữ liệu tương tự như các tập hợp (sets) trong các ngôn ngữ lập trình, cho phép bạn giữ một cấu trúc dữ liệu được sắp xếp theo một tiêu chí nhất định.
Bên trong, nó được triển khai bằng cách sử dụng một bảng băm (hash-map) để duy trì ánh xạ giữa khóa (`user_id`) và giá trị (`score`) và một skip list (danh sách nhảy) ánh xạ điểm số đến người dùng theo thứ tự được sắp xếp:

<div style="margin-left:3rem">
    <img src="./images/sorted-set.png" alt="sorted-set" width="500" />
</div>

Skip list hoạt động như thế nào?
- Nó là một danh sách liên kết cho phép tìm kiếm nhanh.
- Nó bao gồm một danh sách liên kết được sắp xếp và các chỉ mục đa cấp.

<div style="margin-left:3rem">
    <img src="./images/skip-list.png" alt="skip-list" width="500" />
</div>

Cấu trúc này cho phép chúng ta nhanh chóng tìm kiếm các giá trị cụ thể khi tập dữ liệu đủ lớn.
Trong ví dụ dưới đây (64 node), việc tìm kiếm yêu cầu duyệt qua 62 node trong danh sách liên kết cơ bản và 11 node trong trường hợp dùng skip-list:

<div style="margin-left:3rem">
    <img src="./images/skip-list-performance.png" alt="skip-list-performance" width="500" />
</div>

Sorted sets hiệu quả hơn các cơ sở dữ liệu quan hệ vì dữ liệu luôn được giữ ở trạng thái sắp xếp với chi phí O(logN) cho thao tác thêm và tìm kiếm.

Ngược lại, đây là một ví dụ về truy vấn lồng nhau mà chúng ta cần chạy để tìm thứ hạng của một người dùng cụ thể trong một DB quan hệ:

```
SELECT *,(SELECT COUNT(*) FROM leaderboard lb2
WHERE lb2.score >= lb1.score) RANK
FROM leaderboard lb1
WHERE lb1.user_id = {:user_id};
```

Chúng ta cần những thao tác nào để vận hành bảng xếp hạng của mình trong Redis?
- **ZADD** - chèn người dùng vào tập hợp nếu họ chưa tồn tại. Nếu đã có, cập nhật điểm số. Độ phức tạp thời gian O(logN).
- **ZINCRBY** - tăng điểm số của người dùng thêm một lượng nhất định. Nếu người dùng chưa tồn tại, điểm bắt đầu từ 0. Độ phức tạp thời gian O(logN).
- **ZRANGE/ZREVRANGE** - lấy một phạm vi người dùng, được sắp xếp theo điểm số. Chúng ta có thể chỉ định thứ tự (ASC/DESC), độ lệch (offset) và kích thước kết quả. Độ phức tạp thời gian O(logN+M) với M là kích thước kết quả.
- **ZRANK/ZREVRANK** - Lấy vị trí (thứ hạng) của người dùng cụ thể theo thứ tự ASC/DESC. Độ phức tạp thời gian O(logN).

Điều gì xảy ra khi một người dùng thắng một điểm?

```
ZINCRBY leaderboard_feb_2021 1 'mary1934'
```

Một bảng xếp hạng mới được tạo ra mỗi tháng trong khi những bảng cũ được chuyển sang kho lưu trữ lịch sử.

Điều gì xảy ra khi một người dùng lấy top 10 người chơi?

```
ZREVRANGE leaderboard_feb_2021 0 9 WITHSCORES
```

Kết quả ví dụ:

```
[(user2,score2),(user1,score1),(user5,score5)...]
```

Còn việc người dùng lấy vị trí trên bảng xếp hạng của họ thì sao?

<div style="margin-left:3rem">
    <img src="./images/leaderboard-position-of-user.png" alt="leaderboard-position-of-user" width="500" />
</div>

Điều này có thể dễ dàng đạt được bằng truy vấn sau, giả sử chúng ta biết vị trí của người dùng trên bảng xếp hạng:

```
ZREVRANGE leaderboard_feb_2021 357 365
```

Vị trí của một người dùng có thể được lấy bằng `ZREVRANK <user-id>`.

Hãy khám phá các yêu cầu về lưu trữ của chúng ta:
- Giả sử kịch bản tệ nhất là tất cả 25 triệu MAU tham gia trò chơi trong một tháng nhất định.
- ID là chuỗi 24 ký tự và điểm số là số nguyên 16-bit, chúng ta cần 26 byte * 25 triệu = ~650MB dung lượng lưu trữ.
- Ngay cả khi chúng ta gấp đôi chi phí lưu trữ do overhead của skip list, con số này vẫn dễ dàng nằm gọn trong một cụm Redis hiện đại.

Một yêu cầu phi chức năng khác cần xem xét là hỗ trợ 2500 lần cập nhật mỗi giây. Điều này nằm trong khả năng của một máy chủ Redis duy nhất.

Lưu ý bổ sung:
- Một thực hành tốt là cấp phát bộ nhớ gấp đôi lượng yêu cầu cho các node Redis ghi nhiều để đáp ứng các bản sao nhanh (snapshots) nếu cần.
- Chúng ta có thể sử dụng một công cụ gọi là Redis-benchmark để theo dõi hiệu suất của thiết lập Redis và đưa ra các quyết định dựa trên dữ liệu.

---

## Bước 3: Thiết kế Chi tiết

### **Sử dụng nhà cung cấp đám mây hay không?**

Chúng ta có thể chọn tự triển khai và quản lý các dịch vụ của mình hoặc sử dụng một nhà cung cấp đám mây để quản lý chúng giúp chúng ta.

Nếu chúng ta chọn tự quản lý các dịch vụ, chúng ta sẽ sử dụng Redis cho dữ liệu bảng xếp hạng, MySQL cho hồ sơ người dùng và có thể là một bộ nhớ đệm cho hồ sơ người dùng nếu chúng ta muốn mở rộng quy mô cơ sở dữ liệu:

<div style="margin-left:3rem">
    <img src="./images/manage-services-ourselves.png" alt="manage-services-ourselves" width="500" />
</div>

Ngoài ra, chúng ta có thể sử dụng các sản phẩm đám mây để quản lý nhiều dịch vụ giúp chúng ta. Ví dụ, chúng ta có thể sử dụng AWS API Gateway để điều hướng các cuộc gọi API đến các hàm AWS Lambda:

<div style="margin-left:3rem">
    <img src="./images/api-gateway-mapping.png" alt="api-gateway-mapping" width="500" />
</div>

AWS Lambda cho phép chúng ta chạy mã mà không cần tự quản lý hoặc cung cấp máy chủ. Nó chỉ chạy khi cần thiết và tự động mở rộng quy mô.

Ví dụ người dùng ghi một điểm:

<div style="margin-left:3rem">
    <img src="./images/user-scoring-point-lambda.png" alt="user-scoring-point-lambda" width="500" />
</div>

Ví dụ người dùng lấy bảng xếp hạng:

<div style="margin-left:3rem">
    <img src="./images/user-retrieve-leaderboard.png" alt="user-retrieve-leaderboard" width="500" />
</div>

Lambda là một triển khai của kiến trúc không máy chủ (serverless). Chúng ta không cần quản lý việc mở rộng quy mô và thiết lập môi trường.

Tác giả khuyến nghị đi theo cách tiếp cận này nếu chúng ta xây dựng trò chơi từ đầu.

### **Mở rộng quy mô Redis**

Với 5 triệu DAU, chúng ta có thể xử lý được chỉ với một phiên bản Redis duy nhất xét về cả khía cạnh lưu trữ và QPS.

Tuy nhiên, nếu chúng ta tưởng tượng cơ sở người dùng tăng gấp 10 lần lên 500 triệu DAU, thì chúng ta sẽ cần 65GB dung lượng lưu trữ và QPS lên tới 250 nghìn.

Quy mô như vậy sẽ yêu cầu phân mảnh (sharding).

Một cách để đạt được điều đó là phân vùng dữ liệu theo phạm vi (range-partitioning):

<div style="margin-left:3rem">
    <img src="./images/range-partition.png" alt="range-partition" width="500" />
</div>

Trong ví dụ này, chúng ta sẽ phân mảnh dựa trên điểm số của người dùng. Chúng ta sẽ duy trì ánh xạ giữa `user_id` và shard trong mã ứng dụng.
Chúng ta có thể thực hiện việc đó qua MySQL hoặc một bộ nhớ đệm khác cho chính ánh xạ đó.

Để lấy top 10 người chơi, chúng ta sẽ truy vấn shard có điểm số cao nhất (`[900-1000]`).

Để lấy thứ hạng của một người dùng, chúng ta sẽ cần tính toán thứ hạng trong shard của người dùng đó và cộng tất cả những người dùng có điểm số cao hơn ở các shard khác.
Hoạt động sau cùng là O(1) vì tổng số bản ghi trên mỗi shard có thể được truy cập nhanh chóng qua lệnh `info keyspace`.

Ngoài ra, chúng ta có thể sử dụng phân vùng băm (hash partitioning) qua Redis Cluster. Nó là một proxy phân phối dữ liệu qua các node Redis dựa trên việc phân vùng tương tự như băm nhất quán (consistent hashing), nhưng không hoàn toàn giống nhau:

<div style="margin-left:3rem">
    <img src="./images/hash-partition.png" alt="hash-partition" width="500" />
</div>

Việc tính toán top 10 người chơi là một thách thức với thiết lập này. Chúng ta sẽ cần lấy top 10 người chơi của mỗi shard và hợp nhất kết quả trong ứng dụng:

<div style="margin-left:3rem">
    <img src="./images/top-10-players-calculation.png" alt="top-10-players-calculation" width="500" />
</div>

Có một số hạn chế với phân vùng băm:
- Nếu chúng ta cần lấy top K người dùng với K lớn, độ trễ có thể tăng lên vì chúng ta sẽ cần lấy rất nhiều dữ liệu từ tất cả các shard.
- Độ trễ tăng lên khi số lượng phân vùng tăng.
- Không có cách tiếp cận đơn giản để xác định thứ hạng của một người dùng.

Do tất cả những điều này, tác giả nghiêng về việc sử dụng các phân vùng cố định (fixed partitions) cho vấn đề này.

Lưu ý khác:
- Một thực hành tốt là cấp phát bộ nhớ gấp đôi lượng yêu cầu cho các node Redis ghi nhiều để đáp ứng các bản sao nhanh (snapshots) nếu cần.
- Chúng ta có thể sử dụng công cụ Redis-benchmark để theo dõi hiệu suất của thiết lập Redis và đưa ra các quyết định dựa trên dữ liệu.

### **Giải pháp thay thế: NoSQL**

Một giải pháp thay thế cần xem xét là sử dụng một cơ sở dữ liệu NoSQL phù hợp được tối ưu hóa cho:
- Lượng ghi lớn.
- Sắp xếp hiệu quả các mục trong cùng một phân vùng theo điểm số.

DynamoDB, Cassandra hoặc MongoDB đều là những lựa chọn phù hợp.

Trong chương này, tác giả đã quyết định sử dụng DynamoDB. Nó là một cơ sở dữ liệu NoSQL được quản lý hoàn toàn, cung cấp hiệu suất đáng tin cậy và khả năng mở rộng tuyệt vời.
Nó cũng cho phép sử dụng các chỉ mục thứ cấp toàn cục (global secondary indexes) khi chúng ta cần truy vấn các trường không thuộc khóa chính.

<div style="margin-left:3rem">
    <img src="./images/dynamo-db.png" alt="dynamo-db" width="500" />
</div>

Hãy bắt đầu từ một bảng để lưu trữ bảng xếp hạng cho một trò chơi cờ vua:

<div style="margin-left:3rem">
    <img src="./images/chess-game-leaderboard-table-1.png" alt="chess-game-leaderboard-table-1" width="500" />
</div>

Điều này hoạt động tốt, nhưng không mở rộng tốt nếu chúng ta cần truy vấn bất cứ thứ gì theo điểm số. Do đó, chúng ta có thể đưa điểm số vào làm khóa sắp xếp (sort key):

<div style="margin-left:3rem">
    <img src="./images/chess-game-leaderboard-table-2.png" alt="chess-game-leaderboard-table-2" width="500" />
</div>

Một vấn đề khác với thiết kế này là chúng ta đang phân vùng theo tháng. Điều này dẫn đến phân vùng điểm nóng (hotspot partition) vì tháng mới nhất sẽ được truy cập không đồng đều so với những tháng khác.

Chúng ta có thể sử dụng kỹ thuật gọi là ghi phân mảnh (write sharding), nơi chúng ta thêm một số phân vùng cho mỗi khóa, được tính toán qua `user_id % num_partitions`:

<div style="margin-left:3rem">
    <img src="./images/chess-game-leaderboard-table-3.png" alt="chess-game-leaderboard-table-3" width="500" />
</div>

Một sự đánh đổi quan trọng cần xem xét là chúng ta nên sử dụng bao nhiêu phân vùng:
- Càng nhiều phân vùng, khả năng mở rộng ghi càng cao.
- Tuy nhiên, khả năng mở rộng đọc bị ảnh hưởng vì chúng ta cần truy vấn nhiều phân vùng hơn để thu thập kết quả tổng hợp.

Sử dụng cách tiếp cận này yêu cầu chúng ta sử dụng kỹ thuật "scatter-gather" mà chúng ta đã thấy trước đó, độ phức tạp thời gian của nó tăng lên khi chúng ta thêm nhiều phân vùng hơn:

<div style="margin-left:3rem">
    <img src="./images/scatter-gather-2.png" alt="scatter-gather-2" width="500" />
</div>

Để đưa ra đánh giá tốt về số lượng phân vùng, chúng ta cần thực hiện một số bước đo điểm chuẩn (benchmarking).

Cách tiếp cận NoSQL này vẫn có một nhược điểm lớn - rất khó để tính toán thứ hạng cụ thể của một người dùng.

Nếu chúng ta có quy mô đủ lớn để yêu cầu phân mảnh, thì có lẽ chúng ta có thể cho người dùng biết họ nằm trong "phần trăm" (percentile) điểm số nào.

Một công việc định kỳ (cron job) có thể chạy định kỳ để phân tích sự phân bổ điểm số, dựa trên đó xác định phần trăm của người dùng, ví dụ:

```
Phần trăm thứ 10 = điểm số < 100
Phần trăm thứ 20 = điểm số < 500
...
Phần trăm thứ 90 = điểm số < 6500
```

---

## Bước 4: Tổng kết

Những điều khác cần thảo luận nếu thời gian cho phép:
- **Truy xuất nhanh hơn** - Chúng ta có thể lưu trữ đối tượng người dùng qua Redis hash với ánh xạ `user_id -> user object`. Điều này cho phép truy xuất nhanh hơn so với việc truy vấn cơ sở dữ liệu.
- **Phân định thứ hạng khi bằng điểm (Breaking ties)** - Khi hai người chơi có cùng điểm số, chúng ta có thể phân định bằng cách sắp xếp họ dựa trên trận đấu được chơi gần nhất.
- **Phục hồi sau lỗi hệ thống** - Trong trường hợp xảy ra sự cố Redis quy mô lớn, chúng ta có thể tạo lại bảng xếp hạng bằng cách xem qua các mục nhập MySQL WAL và tạo lại nó qua một tập lệnh ad-hoc.
