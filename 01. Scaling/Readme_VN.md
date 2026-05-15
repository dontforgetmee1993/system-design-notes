# Chương 1: Quy mô từ số 0 đến hàng triệu người dùng

## Giới thiệu
Mở rộng quy mô một hệ thống để hỗ trợ hàng triệu người dùng là một hành trình phức tạp, lặp đi lặp lại, đòi hỏi sự tinh chỉnh và tối ưu hóa liên tục. Chương này phác thảo cách bắt đầu với thiết lập máy chủ đơn lẻ và mở rộng kiến trúc từng bước để xử lý hàng triệu người dùng.

---

## Phần 1: Thiết lập máy chủ đơn lẻ (Single Server Setup)
Ban đầu, tất cả các thành phần (ứng dụng web, cơ sở dữ liệu, bộ nhớ đệm) đều chạy trên một máy chủ duy nhất.

### Luồng yêu cầu (Request Flow)
1. Người dùng truy cập ứng dụng thông qua tên miền (ví dụ: `api.mysite.com`), được phân giải thành địa chỉ IP bằng DNS.
2. Địa chỉ IP của máy chủ web được trả về trình duyệt hoặc ứng dụng di động.
3. Các yêu cầu HTTP được gửi đến máy chủ web, máy chủ này sẽ trả về phản hồi HTML hoặc JSON.

### Nguồn lưu lượng (Traffic Sources)
1. **Ứng dụng Web:** Sử dụng các ngôn ngữ phía máy chủ (ví dụ: Python, Java) cho logic nghiệp vụ và các ngôn ngữ phía máy khách (ví dụ: JavaScript, HTML) để hiển thị.
2. **Ứng dụng di động:** Giao tiếp với máy chủ web bằng HTTP và JSON để trao đổi dữ liệu nhẹ nhàng.

---

## Phần 2: Tách biệt cơ sở dữ liệu (Database Separation)
Khi lượng người dùng tăng lên, cơ sở dữ liệu được chuyển sang một máy chủ chuyên dụng để cho phép mở rộng độc lập các tầng web và cơ sở dữ liệu.

### Lựa chọn cơ sở dữ liệu (Database Choices)

1. **Cơ sở dữ liệu quan hệ (SQL):** Dữ liệu có cấu trúc được lưu trữ trong các bảng. Ví dụ: MySQL, PostgreSQL.
2. **Cơ sở dữ liệu phi quan hệ (NoSQL):** Phù hợp với dữ liệu không cấu trúc hoặc yêu cầu độ trễ thấp. Các danh mục bao gồm:
   - Kho lưu trữ Key-Value
   - Cơ sở dữ liệu đồ thị (Graph Databases)
   - Kho lưu trữ cột (Column Stores)
   - Kho lưu trữ tài liệu (Document Stores)

- NoSQL có thể là lựa chọn đúng nếu:
   - Ứng dụng yêu cầu độ trễ cực thấp.
   - Dữ liệu không có cấu trúc hoặc không có dữ liệu quan hệ.
   - Chỉ cần tuần tự hóa và giải tuần tự hóa dữ liệu (JSON, XML, YAML, v.v.).
   - Cần lưu trữ một lượng dữ liệu khổng lồ.

---

## Phần 3: Mở rộng dọc vs Mở rộng ngang (Vertical vs Horizontal Scaling)
### Mở rộng dọc (Vertical Scaling)
- Thêm nhiều tài nguyên hơn (CPU, RAM) vào các máy chủ hiện có.
- Bị hạn chế bởi giới hạn phần cứng và thiếu tính dự phòng.

### Mở rộng ngang (Horizontal Scaling)
- Thêm nhiều máy chủ hơn vào nhóm, giúp nó phù hợp hơn cho các hệ thống quy mô lớn.
- Một bộ cân bằng tải (Load Balancer) được sử dụng để xử lý việc định tuyến yêu cầu giữa các máy chủ.

---

## Phần 4: Bộ cân bằng tải (Load Balancer)
Bộ cân bằng tải phân phối lưu lượng truy cập giữa nhiều máy chủ. Lợi ích bao gồm:
1. **Tính dự phòng (Redundancy):** Nếu một máy chủ ngoại tuyến, lưu lượng truy cập sẽ được chuyển hướng sang máy chủ khác.
2. **Khả năng mở rộng (Scalability):** Dễ dàng thêm máy chủ để xử lý các đợt lưu lượng truy cập tăng đột biến.

---

## Phần 5: Sao chép cơ sở dữ liệu (Database Replication)
### Mô hình Master-Slave
- **Master Database:** Xử lý các thao tác ghi (write). Tất cả các lệnh sửa đổi dữ liệu như insert, delete hoặc update phải được gửi đến master.
- **Slave Databases:** Xử lý các thao tác đọc (read), cải thiện hiệu suất và độ tin cậy. Thông thường số lượng slave sẽ nhiều hơn master vì thao tác đọc thường chiếm đa số.

### Lợi ích
1. Cải thiện hiệu suất thông qua các thao tác đọc song song.
2. Tính sẵn sàng cao và độ tin cậy của dữ liệu thông qua tính dự phòng.

### Xử lý lỗi
- Nếu một slave ngoại tuyến, các yêu cầu đọc sẽ được chuyển sang slave khác hoặc master.
- Nếu master ngoại tuyến, một slave sẽ được thăng cấp thành master mới.

---

## Phần 6: Bộ nhớ đệm (Caching)
Bộ nhớ đệm lưu trữ dữ liệu được truy cập thường xuyên trong bộ nhớ để giảm tải cho cơ sở dữ liệu. Tầng cache nhanh hơn nhiều so với cơ sở dữ liệu.

### Các cân nhắc khi sử dụng Cache
1. **Trường hợp sử dụng:** Sử dụng khi dữ liệu được đọc thường xuyên nhưng ít khi thay đổi.
2. **Chính sách hết hạn (Expiration):** Dữ liệu hết hạn sẽ bị xóa khỏi cache.
3. **Tính nhất quán (Consistency):** Giữ cho dữ liệu trong DB và cache luôn đồng bộ.
4. **Giảm thiểu lỗi:** Nên sử dụng nhiều máy chủ cache để tránh điểm lỗi duy nhất (SPOF).
5. **Chính sách loại bỏ (Eviction):** Khi cache đầy, các mục cũ cần bị loại bỏ (ví dụ: thuật toán LRU - Least Recently Used).

---

## Phần 7: Mạng phân phối nội dung (CDN)
CDN cải thiện thời gian tải bằng cách lưu trữ nội dung tĩnh (hình ảnh, CSS, JS) trên các máy chủ phân tán theo địa lý.

### Quy trình làm việc
1. Người dùng yêu cầu nội dung từ máy chủ CDN gần nhất.
2. Nếu không có sẵn, nội dung sẽ được lấy từ máy chủ gốc (origin) và được lưu vào cache của CDN.

---

## Phần 8: Tầng Web không trạng thái (Stateless Web Tier)
Bằng cách di chuyển dữ liệu phiên (session) sang một kho lưu trữ dữ liệu dùng chung (như Redis), các máy chủ web trở nên không trạng thái (stateless). Điều này cho phép:
1. Mở rộng ngang dễ dàng hơn.
2. Tự động mở rộng (auto-scaling) dựa trên lưu lượng truy cập.

---

## Phần 9: Thiết lập đa trung tâm dữ liệu (Multi-Data Center Setup)
Triển khai trên nhiều trung tâm dữ liệu giúp cải thiện tính sẵn sàng và giảm độ trễ.
1. **Định tuyến GeoDNS:** Hướng người dùng đến trung tâm dữ liệu gần nhất.
2. **Sao chép dữ liệu:** Đồng bộ hóa dữ liệu giữa các trung tâm để ngăn chặn sự không nhất quán.

---

## Phần 10: Hàng đợi tin nhắn (Message Queue)
Hàng đợi tin nhắn hỗ trợ giao tiếp bất đồng bộ, đóng vai trò như một bộ đệm và phân phối các yêu cầu bất đồng bộ.
- **Producers (Người sản xuất):** Tạo và gửi tin nhắn vào hàng đợi.
- **Consumers (Người tiêu dùng):** Kết nối với hàng đợi và thực hiện các hành động được định nghĩa trong tin nhắn.

---

## Phần 11: Nhật ký (Logging), Chỉ số (Metrics) và Tự động hóa (Automation)
1. **Logging:** Theo dõi lỗi và sức khỏe hệ thống.
2. **Metrics:** Cung cấp thông tin chi tiết về hiệu suất và hoạt động của người dùng.
3. **Automation:** Hợp lý hóa việc kiểm thử, triển khai và mở rộng quy mô.

---

## Phần 12: Mở rộng cơ sở dữ liệu (Database Scaling)
### Mở rộng dọc (Vertical Scaling)
- Thêm tài nguyên phần cứng nhưng có giới hạn về vật lý và chi phí, dễ gặp rủi ro điểm lỗi duy nhất.

### Mở rộng ngang (Sharding)
- Chia nhỏ cơ sở dữ liệu lớn thành các phần nhỏ hơn gọi là shards. Mỗi shard có cùng lược đồ (schema) nhưng dữ liệu là duy nhất.
- **Sharding key** là cực kỳ quan trọng để phân phối dữ liệu đồng đều.

#### Thách thức
1. **Tái phân vùng dữ liệu (Resharding):** Cần thiết khi một shard không còn sức chứa hoặc dữ liệu phân phối không đều.
2. **Vấn đề người nổi tiếng (Celebrity problem):** Truy cập quá mức vào một shard cụ thể gây quá tải.
3. **Join và de-normalization:** Khó thực hiện các phép Join giữa các shards, thường phải khử chuẩn hóa (de-normalize) dữ liệu.

---

## Kết luận
### Các bài học chính
1. Giữ cho tầng web không trạng thái (**Stateless**).
2. Xây dựng tính dự phòng (**Redundancy**) ở mọi tầng.
3. Sử dụng **Cache** và **CDN** để tối ưu hiệu suất.
4. Mở rộng tầng dữ liệu bằng **Sharding**.
5. Tách biệt (**Decouple**) các thành phần để linh hoạt hơn.

Chương này cung cấp nền tảng vững chắc để xây dựng các hệ thống có khả năng xử lý hàng triệu người dùng.
