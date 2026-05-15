# Chương 22: Hệ thống Đặt phòng Khách sạn

## Giới thiệu
Trong chương này, chúng ta sẽ thiết kế một **hệ thống đặt phòng khách sạn**, tương tự như Marriott International.

Thiết kế này cũng có thể áp dụng cho các loại hệ thống khác - Airbnb, đặt vé máy bay, đặt vé xem phim.

---

## Bước 1: Hiểu vấn đề và Thiết lập Phạm vi Thiết kế
Trước khi bắt tay vào thiết kế hệ thống, chúng ta nên đặt câu hỏi cho người phỏng vấn để làm rõ phạm vi:
 - C: Quy mô của hệ thống là bao nhiêu?
 - I: Chúng ta đang xây dựng một website cho một chuỗi khách sạn với 5000 khách sạn và 1 triệu phòng.
 - C: Khách hàng thanh toán khi họ đặt phòng hay khi họ đến khách sạn?
 - I: Họ thanh toán đầy đủ khi đặt phòng.
 - C: Khách hàng chỉ đặt phòng khách sạn qua website thôi sao? Chúng ta có cần hỗ trợ các hình thức đặt phòng khác như gọi điện thoại không?
 - I: Họ chỉ đặt phòng qua website hoặc ứng dụng.
 - C: Khách hàng có thể hủy đặt phòng không?
 - I: Có.
 - C: Những điều khác cần cân nhắc?
 - I: Có, chúng ta cho phép đặt phòng vượt mức (overbooking) 10%. Khách sạn sẽ bán nhiều phòng hơn số lượng thực tế có. Các khách sạn làm điều này vì dự đoán rằng khách hàng sẽ hủy đặt phòng.
 - C: Vì không có nhiều thời gian, chúng ta sẽ tập trung vào - hiển thị trang liên quan đến khách sạn, trang chi tiết phòng khách sạn, đặt phòng, bảng điều khiển quản trị (admin panel), hỗ trợ overbooking.
 - I: Nghe ổn đấy.
 - I: Thêm một điều nữa - giá phòng khách sạn thay đổi liên tục. Giả sử giá phòng khách sạn thay đổi hàng ngày.
 - C: Được rồi.

### **Yêu cầu phi chức năng**
 - Hỗ trợ tính đồng thời cao (high concurrency) - có thể có rất nhiều khách hàng cố gắng đặt cùng một khách sạn trong mùa cao điểm.
 - Độ trễ vừa phải - lý tưởng nhất là có độ trễ thấp khi người dùng thực hiện đặt phòng, nhưng có thể chấp nhận được nếu hệ thống mất vài giây để xử lý.

### **Ước tính sơ bộ**
 - Tổng cộng 5000 khách sạn và 1 triệu phòng.
 - Giả sử 70% số phòng được lấp đầy và thời gian lưu trú trung bình là 3 ngày.
 - Ước tính số lượng đặt phòng hàng ngày - 1 triệu * 0.7 / 3 = ~240 nghìn lượt đặt phòng mỗi ngày.
 - Số lượng đặt phòng mỗi giây - 240 nghìn / 10^5 giây trong một ngày = ~3. TPS đặt phòng trung bình là thấp.

Hãy ước tính QPS. Nếu chúng ta giả định rằng có ba bước để đi đến trang đặt phòng và có tỷ lệ chuyển đổi 10% cho mỗi trang,
chúng ta có thể ước tính rằng nếu có 3 lượt đặt phòng, thì phải có 30 lượt xem trang đặt phòng và 300 lượt xem trang chi tiết phòng khách sạn.

<div style="margin-left:3rem">
    <img src="./images/qps-estimation.png" alt="qps-estimation" width="500" />
</div>

---

## Bước 2: Đề xuất Thiết kế Mức cao và Đạt được sự Thống nhất
Chúng ta sẽ khám phá - Thiết kế API, Mô hình dữ liệu, thiết kế mức cao.

### **Thiết kế API**
Thiết kế API này tập trung vào các endpoint cốt lõi (sử dụng các thực hành RESTful), chúng ta sẽ cần để hỗ trợ một hệ thống đặt phòng khách sạn.

 một hệ thống hoàn chỉnh sẽ yêu cầu một API mở rộng hơn với sự hỗ trợ cho việc tìm kiếm phòng dựa trên nhiều tiêu chí, nhưng chúng ta sẽ không tập trung vào phần đó trong mục này.
Lý do là chúng không khó về mặt kỹ thuật, nên chúng nằm ngoài phạm vi.

**API liên quan đến Khách sạn**
 - `GET /v1/hotels/{id}` - lấy thông tin chi tiết về một khách sạn.
 - `POST /v1/hotels` - thêm một khách sạn mới. Chỉ dành cho nhân viên vận hành (ops).
 - `PUT /v1/hotels/{id}` - cập nhật thông tin khách sạn. Chỉ dành cho ops.
 - `DELETE /v1/hotels/{id}` - xóa một khách sạn. API chỉ dành cho ops.

**API liên quan đến Phòng**
 - `GET /v1/hotels/{id}/rooms/{id}` - lấy thông tin chi tiết về một phòng.
 - `POST /v1/hotels/{id}/rooms` - Thêm một phòng. Chỉ dành cho ops.
 - `PUT /v1/hotels/{id}/rooms/{id}` - Cập nhật thông tin phòng. Chỉ dành cho ops.
 - `DELETE /v1/hotels/{id}/rooms/{id}` - Xóa một phòng. Chỉ dành cho ops.

**API liên quan đến Đặt phòng**
 - `GET /v1/reservations` - lấy lịch sử đặt phòng của người dùng hiện tại.
 - `GET /v1/reservations/{id}` - lấy thông tin chi tiết về một lượt đặt phòng.
 - `POST /v1/reservations` - thực hiện một lượt đặt phòng mới.
 - `DELETE /v1/reservations/{id}` - hủy một lượt đặt phòng.

Dưới đây là một ví dụ về yêu cầu thực hiện đặt phòng:

```
{
  "startDate":"2021-04-28",
  "endDate":"2021-04-30",
  "hotelID":"245",
  "roomID":"U12354673389",
  "reservationID":"13422445"
}
```

Lưu ý rằng `reservationID` là một khóa lũy đẳng (idempotency key) để tránh việc đặt phòng trùng lặp. Chi tiết được giải thích trong [phần đồng thời](#cac-van-de-dong-thoi).

### **Mô hình dữ liệu**
Trước khi chọn cơ sở dữ liệu nào để sử dụng, hãy xem xét các mô hình truy cập (access patterns) của chúng ta.

Chúng ta cần hỗ trợ các truy vấn sau:
 - Xem thông tin chi tiết về một khách sạn.
 - Tìm các loại phòng còn trống trong một khoảng ngày nhất định.
 - Ghi lại một lượt đặt phòng.
 - Tra cứu một lượt đặt phòng hoặc lịch sử đặt phòng cũ.

Từ các ước tính của mình, chúng ta biết quy mô của hệ thống không lớn, nhưng chúng ta cần chuẩn bị cho các đợt tăng vọt lưu lượng truy cập.

Với những kiến thức này, chúng ta sẽ chọn một cơ sở dữ liệu quan hệ (relational database) vì:
 - DB quan hệ hoạt động tốt với các hệ thống đọc nhiều và ghi ít.
 - Cơ sở dữ liệu NoSQL thường được tối ưu hóa cho việc ghi, nhưng chúng ta biết mình sẽ không có nhiều lượt ghi vì chỉ một phần nhỏ người dùng truy cập trang web thực hiện đặt phòng.
 - DB quan hệ cung cấp các đảm bảo ACID. Những điều này rất quan trọng đối với một hệ thống như vậy vì nếu không có chúng, chúng ta sẽ không thể ngăn chặn các vấn đề như số dư âm, sạc phí hai lần, v.v.
 - DB quan hệ có thể dễ dàng mô hình hóa dữ liệu vì cấu trúc rất rõ ràng.

Dưới đây là thiết kế schema của chúng ta:

<div style="margin-left:3rem">
    <img src="./images/schema-design.png" alt="schema-design" width="500" />
</div>

Hầu hết các trường đều dễ hiểu. Trường duy nhất đáng đề cập là trường `status` đại diện cho máy trạng thái (state machine) của một phòng nhất định:

<div style="margin-left:3rem">
    <img src="./images/status-state-machine.png" alt="status-state-machine" width="500" />
</div>

Mô hình dữ liệu này hoạt động tốt cho một hệ thống như Airbnb, nhưng không phù hợp cho các khách sạn nơi người dùng không đặt một phòng cụ thể mà đặt một loại phòng (room type).
Họ đặt một loại phòng và số phòng sẽ được chọn tại thời điểm đặt phòng.

Thiếu sót này sẽ được giải quyết trong phần [Mô hình dữ liệu Cải tiến](#mo-hinh-du-lieu-cai-tien).

### **Thiết kế mức cao**
Chúng ta đã chọn kiến trúc microservice cho thiết kế này. Nó đã trở nên rất phổ biến trong những năm gần đây:

<div style="margin-left:3rem">
    <img src="./images/high-level-design.png" alt="high-level-design" width="500" />
</div>

 - **Người dùng (Users)**: đặt phòng khách sạn trên điện thoại hoặc máy tính của họ.
 - **Quản trị viên (Admin)**: thực hiện các chức năng quản trị như hoàn tiền/hủy thanh toán, v.v.
 - **CDN**: lưu trữ các tài nguyên tĩnh như file JS, hình ảnh, video, v.v.
 - **Public API Gateway**: dịch vụ được quản lý hoàn toàn hỗ trợ giới hạn tốc độ (rate limiting), xác thực, v.v.
 - **Internal APIs**: chỉ hiển thị cho nhân viên được ủy quyền. Thường được bảo vệ bởi VPN.
 - **Dịch vụ khách sạn (Hotel service)**: cung cấp thông tin chi tiết về khách sạn và phòng. Dữ liệu khách sạn và phòng là tĩnh, vì vậy nó có thể được cache mạnh mẽ.
 - **Dịch vụ giá (Rate service)**: cung cấp giá phòng cho các ngày khác nhau trong tương lai. Một lưu ý thú vị về lĩnh vực này là giá cả phụ thuộc vào mức độ lấp đầy của một khách sạn vào một ngày nhất định.
 - **Dịch vụ đặt phòng (Reservation service)**: nhận các yêu cầu đặt phòng và thực hiện đặt phòng. Đồng thời theo dõi kho phòng (inventory) khi các lượt đặt phòng được thực hiện/hủy bỏ.
 - **Dịch vụ thanh toán (Payment service)**: xử lý thanh toán và cập nhật trạng thái đặt phòng khi thành công.
 - **Dịch vụ quản lý khách sạn (Hotel management service)**: chỉ dành cho nhân viên được ủy quyền. Cho phép các chức năng quản trị nhất định để quản lý và xem các lượt đặt phòng, khách sạn, v.v.

Giao tiếp giữa các dịch vụ có thể được hỗ trợ thông qua một RPC framework, chẳng hạn như gRPC.

---

## Bước 3: Thiết kế Chi tiết
Hãy đi sâu vào:
 - Mô hình dữ liệu cải tiến
 - Các vấn đề đồng thời
 - Khả năng mở rộng
 - Giải quyết sự không nhất quán dữ liệu trong microservices

### **Mô hình dữ liệu cải tiến**
Như đã đề cập trong phần trước, chúng ta cần sửa đổi API và schema của mình để cho phép đặt một loại phòng thay vì một phòng cụ thể.

Đối với API đặt phòng, chúng ta không còn đặt `roomID` nữa, mà đặt một `roomTypeID`:

```
POST /v1/reservations
{
  "startDate":"2021-04-28",
  "endDate":"2021-04-30",
  "hotelID":"245",
  "roomTypeID":"12354673389",
  "roomCount":"3",
  "reservationID":"13422445"
}
```

Dưới đây là schema đã được cập nhật:

<div style="margin-left:3rem">
    <img src="./images/updated-schema.png" alt="updated-schema" width="500" />
</div>

 - **room**: chứa thông tin về một phòng.
 - **room_type_rate**: chứa thông tin về giá cho một loại phòng nhất định.
 - **reservation**: ghi lại dữ liệu đặt phòng của khách.
 - **room_type_inventory**: lưu trữ dữ liệu kho (inventory) về các phòng khách sạn.

Hãy xem qua các cột của `room_type_inventory` vì bảng đó thú vị hơn:
 - **hotel_id**: id của khách sạn.
 - **room_type_id**: id của một loại phòng.
 - **date**: một ngày duy nhất.
 - **total_inventory**: tổng số lượng phòng trừ đi những phòng tạm thời bị loại khỏi kho.
 - **total_reserved**: tổng số lượng phòng đã được đặt cho (hotel_id, room_type_id, date) cụ thể.

Có các cách thay thế để thiết kế bảng này, nhưng việc có một hàng cho mỗi (hotel_id, room_type_id, date) giúp việc quản lý đặt phòng và các truy vấn trở nên dễ dàng hơn.

Các hàng trong bảng được điền sẵn bằng một công việc CRON hàng ngày.

Dữ liệu mẫu:
| hotel_id | room_type_id | date       | total_inventory | total_reserved |
|----------|--------------|------------|-----------------|----------------|
| 211      | 1001         | 2021-06-01 | 100             | 80             |
| 211      | 1001         | 2021-06-02 | 100             | 82             |
| 211      | 1001         | 2021-06-03 | 100             | 86             |
| 211      | 1001         | ...        | ...             |                |
| 211      | 1001         | 2023-05-31 | 100             | 0              |
| 211      | 1002         | 2021-06-01 | 200             | 16             |
| 2210     | 101          | 2021-06-01 | 30              | 23             |
| 2210     | 101          | 2021-06-02 | 30              | 25             |

Truy vấn SQL mẫu để kiểm tra tính sẵn có của một loại phòng:

```
SELECT date, total_inventory, total_reserved
FROM room_type_inventory
WHERE room_type_id = ${roomTypeId} AND hotel_id = ${hotelId}
AND date between ${startDate} and ${endDate}
```

Cách kiểm tra tính sẵn có cho một số lượng phòng cụ thể sử dụng dữ liệu đó (lưu ý rằng chúng ta hỗ trợ overbooking):

```
if (total_reserved + ${numberOfRoomsToReserve}) <= 110% * total_inventory
```

Bây giờ hãy thực hiện một số ước tính về khối lượng lưu trữ.
 - Chúng ta có 5000 khách sạn.
 - Mỗi khách sạn có 20 loại phòng.
 - 5000 * 20 * 2 (năm) * 365 (ngày) = 73 triệu hàng.

73 triệu hàng không phải là nhiều dữ liệu và một máy chủ cơ sở dữ liệu duy nhất có thể xử lý được.
Tuy nhiên, việc thiết lập bản sao đọc (read replication - có khả năng trên các vùng khác nhau) là hợp lý để đảm bảo tính sẵn sàng cao.

Câu hỏi tiếp theo - nếu dữ liệu đặt phòng quá lớn cho một cơ sở dữ liệu duy nhất, bạn sẽ làm gì?
 - Chỉ lưu trữ dữ liệu đặt phòng hiện tại và tương lai. Lịch sử đặt phòng có thể được chuyển sang kho lạnh.
 - Phân mảnh cơ sở dữ liệu (Database sharding) - chúng ta có thể phân mảnh dữ liệu của mình theo `hash(hotel_id) % servers_cnt` vì chúng ta luôn chọn `hotel_id` trong các truy vấn của mình.

### **Các vấn đề đồng thời**
Một vấn đề quan trọng khác cần giải quyết là đặt phòng trùng lặp (double booking).

Có hai vấn đề cần giải quyết:
 - Cùng một người dùng nhấp vào nút "đặt" hai lần.
 - Nhiều người dùng cố gắng đặt cùng một phòng tại cùng một thời điểm.

Dưới đây là hình ảnh minh họa cho vấn đề đầu tiên:

<div style="margin-left:3rem">
    <img src="./images/double-booking-single-user.png" alt="double-booking-single-user" width="500" />
</div>

Có hai cách tiếp cận để giải quyết vấn đề này:
 - Xử lý phía client - front-end có thể vô hiệu hóa nút đặt phòng ngay khi được nhấp vào. Tuy nhiên, nếu người dùng vô hiệu hóa javascript, họ sẽ không thấy nút bị mờ đi.
 - API lũy đẳng (Idempotent API) - Thêm một khóa lũy đẳng vào API, cho phép người dùng thực hiện một hành động một lần duy nhất, bất kể endpoint được gọi bao nhiêu lần:

<div style="margin-left:3rem">
    <img src="./images/idempotency.png" alt="idempotency" width="500" />
</div>

Đây là cách luồng này hoạt động:
 - Một đơn hàng đặt phòng (reservation order) được tạo ra ngay khi bạn đang trong quá trình điền thông tin chi tiết và thực hiện đặt phòng. Đơn hàng đặt phòng được tạo bằng một mã định danh duy nhất toàn cầu (globally unique identifier).
 - Gửi yêu cầu đặt phòng 1 sử dụng `reservation_id` được tạo ở bước trước.
 - Nếu nút "hoàn tất đặt phòng" được nhấp lần thứ hai, cùng một `reservation_id` sẽ được gửi đi và backend sẽ phát hiện đây là một lượt đặt phòng trùng lặp.
 - Việc trùng lặp được tránh bằng cách đặt cột `reservation_id` có ràng buộc duy nhất (unique constraint), ngăn chặn nhiều bản ghi với cùng id đó được lưu trữ trong DB.

<div style="margin-left:3rem">
    <img src="./images/unique-constraint-violation.png" alt="unique-constraint-violation" width="500" />
</div>

Điều gì xảy ra nếu có nhiều người dùng thực hiện cùng một lượt đặt phòng?

<div style="margin-left:3rem">
    <img src="./images/double-booking-multiple-users.png" alt="double-booking-multiple-users" width="500" />
</div>

 - Giả sử mức độ cô lập giao dịch (transaction isolation level) không phải là serializable.
 - Người dùng 1 và 2 cố gắng đặt cùng một phòng tại cùng một thời điểm.
 - Giao dịch 1 kiểm tra xem có đủ phòng không - có.
 - Giao dịch 2 kiểm tra xem có đủ phòng không - có.
 - Giao dịch 2 đặt phòng và cập nhật kho phòng.
 - Giao dịch 1 cũng đặt phòng vì nó vẫn thấy có 99 phòng `total_reserved` trên tổng số 100 phòng.
 - Cả hai giao dịch đều commit thay đổi thành công.

Vấn đề này có thể được giải quyết bằng một số hình thức cơ chế khóa (locking mechanism):
 - Khóa bi quan (Pessimistic locking)
 - Khóa lạc quan (Optimistic locking)
 - Ràng buộc cơ sở dữ liệu (Database constraints)

Dưới đây là SQL chúng ta sử dụng để đặt một phòng:

```sql
# bước 1: kiểm tra kho phòng
SELECT date, total_inventory, total_reserved
FROM room_type_inventory
WHERE room_type_id = ${roomTypeId} AND hotel_id = ${hotelId}
AND date between ${startDate} and ${endDate}

# Đối với mỗi mục được trả về từ bước 1
if((total_reserved + ${numberOfRoomsToReserve}) > 110% * total_inventory) {
  Rollback
}

# bước 2: đặt phòng
UPDATE room_type_inventory
SET total_reserved = total_reserved + ${numberOfRoomsToReserve}
WHERE room_type_id = ${roomTypeId}
AND date between ${startDate} and ${endDate}

Commit
```

#### Lựa chọn 1: Khóa bi quan (Pessimistic locking)
Khóa bi quan ngăn chặn các cập nhật đồng thời bằng cách đặt một khóa lên một bản ghi trong khi nó đang được cập nhật.

Điều này có thể thực hiện trong MySQL bằng cách sử dụng truy vấn `SELECT... FOR UPDATE`, truy vấn này sẽ khóa các hàng được chọn cho đến khi giao dịch được commit.

<div style="margin-left:3rem">
    <img src="./images/pessimistic-locking.png" alt="pessimistic-locking" width="500" />
</div>

Ưu điểm:
 - Ngăn ứng dụng cập nhật dữ liệu đang bị thay đổi.
 - Dễ triển khai và tránh xung đột bằng cách tuần tự hóa các cập nhật. Hữu ích khi có sự tranh chấp dữ liệu (data contention) lớn.

Nhược điểm:
 - Deadlock có thể xảy ra khi nhiều tài nguyên bị khóa.
 - Cách tiếp cận này không có khả năng mở rộng tốt - nếu một giao dịch bị khóa quá lâu, nó sẽ ảnh hưởng đến tất cả các giao dịch khác đang cố gắng truy cập tài nguyên đó.
 - Tác động sẽ nghiêm trọng khi truy vấn chọn nhiều tài nguyên và giao dịch tồn tại lâu.

Tác giả không khuyến nghị cách tiếp cận này do các vấn đề về khả năng mở rộng.

#### Lựa chọn 2: Khóa lạc quan (Optimistic locking)
Khóa lạc quan cho phép nhiều người dùng cố gắng cập nhật một bản ghi cùng một lúc.

Có hai cách phổ biến để triển khai nó - số phiên bản (version numbers) và mốc thời gian (timestamps). Số phiên bản được khuyến nghị vì đồng hồ máy chủ có thể không chính xác.

<div style="margin-left:3rem">
    <img src="./images/optimistic-locking.png" alt="optimistic-locking" width="500" />
</div>

 - Một cột `version` mới được thêm vào bảng cơ sở dữ liệu.
 - Trước khi người dùng sửa đổi một hàng trong cơ sở dữ liệu, số phiên bản sẽ được đọc.
 - Khi người dùng cập nhật hàng đó, số phiên bản sẽ tăng thêm 1 và được ghi lại vào cơ sở dữ liệu.
 - Việc xác thực cơ sở dữ liệu sẽ ngăn chặn lệnh chèn nếu số phiên bản mới không lớn hơn số phiên bản trước đó.

Khóa lạc quan thường nhanh hơn khóa bi quan vì chúng ta không khóa cơ sở dữ liệu.
Tuy nhiên, hiệu suất của nó có xu hướng giảm khi tính đồng thời cao, vì điều đó dẫn đến nhiều lượt rollback.

Ưu điểm:
 - Ngăn ứng dụng chỉnh sửa dữ liệu cũ (stale data).
 - Chúng ta không cần lấy một khóa trong cơ sở dữ liệu.
 - Lựa chọn ưu tiên khi tranh chấp dữ liệu thấp, tức là hiếm khi có xung đột cập nhật.

Nhược điểm:
 - Hiệu suất kém khi tranh chấp dữ liệu cao.

Khóa lạc quan là một lựa chọn tốt cho hệ thống của chúng ta vì QPS đặt phòng không quá cao.

#### Lựa chọn 3: Ràng buộc cơ sở dữ liệu (Database constraints)
Cách tiếp cận này rất giống với khóa lạc quan, nhưng các hàng rào bảo vệ được triển khai bằng một ràng buộc của cơ sở dữ liệu:

```
CONSTRAINT `check_room_count` CHECK((`total_inventory - total_reserved` >= 0))
```

<div style="margin-left:3rem">
    <img src="./images/database-constraint.png" alt="database-constraint" width="500" />
</div>

Ưu điểm:
 - Dễ triển khai.
 - Hoạt động tốt khi tranh chấp dữ liệu nhỏ.

Nhược điểm:
 - Tương tự như khóa lạc quan, hiệu suất kém khi tranh chấp dữ liệu cao.
 - Các ràng buộc cơ sở dữ liệu không thể dễ dàng được quản lý phiên bản (version-controlled) như code ứng dụng.
 - Không phải tất cả các cơ sở dữ liệu đều hỗ trợ các ràng buộc.

Đây là một lựa chọn tốt khác cho một hệ thống đặt phòng khách sạn nhờ tính dễ triển khai của nó.

### **Khả năng mở rộng**
Thông thường, tải của một hệ thống đặt phòng khách sạn không cao.

Tuy nhiên, người phỏng vấn có thể hỏi bạn cách xử lý một tình huống mà hệ thống được áp dụng cho một trang web du lịch lớn và phổ biến như booking.com.
Trong trường hợp đó, QPS có thể lớn gấp 1000 lần.

Khi gặp tình huống như vậy, điều quan trọng là phải hiểu các nút thắt cổ chai (bottlenecks) của chúng ta nằm ở đâu. Tất cả các dịch vụ đều không trạng thái (stateless), vì vậy chúng có thể dễ dàng được mở rộng thông qua bản sao (replication).

Tuy nhiên, cơ sở dữ liệu là có trạng thái (stateful) và không dễ thấy cách để mở rộng nó.

Một cách để mở rộng nó là triển khai phân mảnh cơ sở dữ liệu (database sharding) - chúng ta có thể chia nhỏ dữ liệu trên nhiều cơ sở dữ liệu, nơi mỗi cơ sở dữ liệu chứa một phần dữ liệu.

Chúng ta có thể phân mảnh dựa trên `hotel_id` vì tất cả các truy vấn đều lọc dựa trên nó.
Giả sử QPS là 30,000, sau khi phân mảnh cơ sở dữ liệu thành 16 phân mảnh (shards), mỗi phân mảnh sẽ xử lý 1875 QPS, nằm trong khả năng chịu tải của một MySQL cluster duy nhất.

<div style="margin-left:3rem">
    <img src="./images/database-sharding.png" alt="database-sharding" width="500" />
</div>

Chúng ta cũng có thể sử dụng caching cho kho phòng và lượt đặt phòng thông qua Redis. Chúng ta có thể đặt TTL để dữ liệu cũ có thể hết hạn đối với những ngày đã qua.

<div style="margin-left:3rem">
    <img src="./images/inventory-cache.png" alt="inventory-cache" width="500" />
</div>

Cách chúng ta lưu trữ kho phòng dựa trên `hotel_id`, `room_type_id` và `date`:

```
key: hotelID_roomTypeID_{date}
value: số lượng phòng còn trống cho hotel ID, room type ID và date nhất định.
```

Tính nhất quán dữ liệu xảy ra không đồng bộ và được quản lý bằng cách sử dụng cơ chế CDC streaming - các thay đổi của cơ sở dữ liệu được đọc và áp dụng cho một hệ thống riêng biệt.
Debezium là một lựa chọn phổ biến để đồng bộ hóa các thay đổi của cơ sở dữ liệu với Redis.

Sử dụng cơ chế như vậy, có khả năng bộ nhớ đệm và cơ sở dữ liệu không nhất quán trong một thời gian.
Điều này ổn trong trường hợp của chúng ta vì cơ sở dữ liệu sẽ ngăn chúng ta thực hiện một lượt đặt phòng không hợp lệ.

Điều này sẽ gây ra một số vấn đề trên giao diện người dùng vì người dùng sẽ phải làm mới trang để thấy rằng "không còn phòng nào nữa",
nhưng đó là điều có thể xảy ra bất kể vấn đề này nếu một người chần chừ quá lâu trước khi thực hiện đặt phòng.

Ưu điểm của Caching:
 - Giảm tải cho cơ sở dữ liệu.
 - Hiệu suất cao, vì Redis quản lý dữ liệu trong bộ nhớ.

Nhược điểm của Caching:
 - Duy trì tính nhất quán dữ liệu giữa cache và DB là khó. Chúng ta cần xem xét sự không nhất quán ảnh hưởng đến trải nghiệm người dùng như thế nào.

### **Tính nhất quán dữ liệu giữa các dịch vụ**
Một ứng dụng monolithic cho phép chúng ta sử dụng một cơ sở dữ liệu quan hệ dùng chung để đảm bảo tính nhất quán của dữ liệu.

Trong thiết kế microservice của mình, chúng ta đã chọn một cách tiếp cận lai (hybrid) nơi một số dịch vụ được tách riêng,
nhưng các API đặt phòng và kho phòng được xử lý bởi cùng một dịch vụ.

Điều này được thực hiện vì chúng ta muốn tận dụng các đảm bảo ACID của cơ sở dữ liệu quan hệ để đảm bảo tính nhất quán.

Tuy nhiên, người phỏng vấn có thể thách thức cách tiếp cận này vì nó không phải là một kiến trúc microservice thuần túy, nơi mỗi dịch vụ có một cơ sở dữ liệu riêng:

<div style="margin-left:3rem">
    <img src="./images/microservices-vs-monolith.png" alt="microservices-vs-monolith" width="500" />
</div>

Điều này có thể dẫn đến các vấn đề về tính nhất quán. Trong một máy chủ monolithic, chúng ta có thể tận dụng các khả năng giao dịch của DB quan hệ để triển khai các hoạt động atomic:

<div style="margin-left:3rem">
    <img src="./images/atomicity-monolith.png" alt="atomicity-monolith" width="500" />
</div>

Tuy nhiên, việc đảm bảo tính atomic này sẽ khó khăn hơn khi hoạt động trải dài trên nhiều dịch vụ:

<div style="margin-left:3rem">
    <img src="./images/microservice-non-atomic-operation.png" alt="microservice-non-atomic-operation" width="500" />
</div>

Có một số kỹ thuật nổi tiếng để xử lý những sự không nhất quán dữ liệu này:
 - **Two-phase commit (2PC)**: một giao thức cơ sở dữ liệu đảm bảo commit giao dịch atomic trên nhiều node.
   Tuy nhiên, nó không hiệu quả về hiệu suất, vì sự chậm trễ của một node duy nhất sẽ khiến tất cả các node khác bị chặn.
 - **Saga**: một chuỗi các giao dịch cục bộ, nơi các giao dịch bù đắp (compensating transactions) được kích hoạt nếu bất kỳ bước nào trong quy trình làm việc thất bại. Đây là một cách tiếp cận nhất quán cuối cùng (eventually consistent).

Cần lưu ý rằng việc giải quyết sự không nhất quán dữ liệu qua các microservice là một vấn đề đầy thách thức, làm tăng độ phức tạp của hệ thống.
Tốt hơn là nên xem xét liệu cái giá phải trả có xứng đáng hay không, so với cách tiếp cận thực dụng hơn của chúng ta là đóng gói các hoạt động phụ thuộc trong cùng một cơ sở dữ liệu quan hệ.

---

## Bước 4: Tổng kết
Chúng ta đã trình bày một thiết kế cho một hệ thống đặt phòng khách sạn.

Dưới đây là các bước chúng ta đã trải qua:
 - Thu thập các yêu cầu và thực hiện các tính toán sơ bộ để hiểu quy mô của hệ thống.
 - Chúng ta đã trình bày Thiết kế API, Mô hình dữ liệu và kiến trúc hệ thống trong thiết kế mức cao.
 - Trong phần đi sâu, chúng ta đã khám phá các thiết kế schema cơ sở dữ liệu thay thế khi các yêu cầu thay đổi.
 - Chúng ta đã thảo luận về các tình trạng tranh đua (race conditions) và đề xuất các giải pháp - khóa bi quan/lạc quan, ràng buộc cơ sở dữ liệu.
 - Các cách để mở rộng hệ thống thông qua phân mảnh cơ sở dữ liệu và caching.
 - Cuối cùng, chúng ta đã giải quyết cách xử lý các vấn đề nhất quán dữ liệu qua nhiều microservice.
