# Chương 11: Thiết kế Hệ thống Bảng tin (News Feed System)

## Giới thiệu
**Hệ thống bảng tin (news feed)** hiển thị danh sách các bài đăng cập nhật liên tục (cập nhật trạng thái, ảnh, video và liên kết) từ các kết nối của người dùng. Các ví dụ bao gồm bảng tin của Facebook, Instagram và dòng thời gian của Twitter. Chương này khám phá việc thiết kế một hệ thống bảng tin có khả năng mở rộng.

---

## Bước 1: Hiểu rõ vấn đề

### Yêu cầu
1. **Nền tảng:** Hệ thống hỗ trợ cả ứng dụng web và di động.
2. **Tính năng:**
   - Người dùng có thể xuất bản bài viết.
   - Người dùng có thể xem bài viết từ bạn bè trong bảng tin của họ.
3. **Sắp xếp:** Bảng tin được sắp xếp theo **thứ tự thời gian đảo ngược** (mới nhất trước) cho đơn giản.
4. **Quy mô:**
   - Người dùng có thể có tối đa 5.000 bạn bè.
   - 10 triệu người dùng hoạt động hàng ngày (DAU).
   - Bảng tin có thể bao gồm văn bản, hình ảnh và video.

---

## Bước 2: Thiết kế tổng quan

### Tổng quan
Thiết kế bao gồm hai luồng chính:
1. **Xuất bản Bảng tin (Feed Publishing):** Một người dùng đăng một bài viết, bài viết này được ghi vào cơ sở dữ liệu và truyền đi đến bảng tin của bạn bè họ.
2. **Xây dựng Bảng tin (News Feed Building):** Một người dùng lấy bảng tin của họ bằng cách tổng hợp các bài viết từ bạn bè theo thứ tự thời gian đảo ngược.

---

### API Bảng tin
1. **API Xuất bản Bảng tin:**
   - **Endpoint:** `POST /v1/me/feed`
   - **Tham số:** `content` (nội dung bài đăng) và `auth_token` (xác thực).

2. **API Lấy Bảng tin:**
   - **Endpoint:** `GET /v1/me/feed`
   - **Tham số:** `auth_token` (xác thực).

---

### Xuất bản Bảng tin

   <div style="margin-left:3rem">
      <img src="./images/feed-publishing.png" alt="Feed Publishing" width="400">
   </div>

1. **Tương tác của người dùng:** Người dùng đăng bài viết thông qua API xuất bản bảng tin.
2. **Bộ cân bằng tải (Load Balancer):** Phân phối lưu lượng truy cập đến các máy chủ web.
3. **Máy chủ Web (Web Servers):** Xác thực các yêu cầu và chuyển hướng đến các dịch vụ.
4. **Dịch vụ Bài viết (Post Service):** Lưu trữ bài viết trong cơ sở dữ liệu và bộ nhớ đệm (cache).
5. **Dịch vụ Truyền phát (Fanout Service):** Truyền bài viết tới bảng tin của bạn bè trong bộ nhớ đệm.
6. **Dịch vụ Thông báo (Notification Service):** Gửi thông báo cho bạn bè.

---

### Xây dựng Bảng tin

   <div style="margin-left:3rem">
      <img src="./images/news-feed-building.png" alt="News Feed Building" width="400">
   </div>

1. **Tương tác của người dùng:** Người dùng yêu cầu bảng tin của họ thông qua API lấy bảng tin.
2. **Bộ cân bằng tải (Load Balancer):** Phân phối lưu lượng truy cập đến các máy chủ web.
3. **Máy chủ Web (Web Servers):** Chuyển tiếp các yêu cầu đến dịch vụ bảng tin.
4. **Dịch vụ Bảng tin (News Feed Service):** Lấy danh sách ID bài viết từ bộ nhớ đệm bảng tin và truy xuất chi tiết bài viết đầy đủ từ cơ sở dữ liệu hoặc bộ nhớ đệm.

---

## Bước 3: Đi sâu vào thiết kế

### Đi sâu vào Xuất bản Bảng tin
1. **Máy chủ Web:**
   - Xác thực người dùng bằng `auth_token`.
   - Thực thi giới hạn tốc độ (rate limit) để ngăn chặn thư rác.

2. **Dịch vụ Truyền phát (Fanout Service):**
   - **Truyền phát khi Ghi (Fanout on Write):** Đẩy bài viết đến bảng tin của bạn bè tại thời điểm viết bài.
     - **Ưu điểm:** Cập nhật theo thời gian thực, tải bảng tin nhanh chóng.
     - **Nhược điểm:** Tiêu tốn nhiều tài nguyên đối với người dùng có nhiều bạn bè.
   - **Truyền phát khi Đọc (Fanout on Read):** Kéo bài viết tại thời điểm đọc.
     - **Ưu điểm:** Hiệu quả đối với người dùng không hoạt động.
     - **Nhược điểm:** Tải bảng tin chậm hơn.
   - **Phương pháp Kết hợp (Hybrid Approach):** Sử dụng mô hình push cho hầu hết người dùng và mô hình pull cho người dùng có kết nối cao (ví dụ: người nổi tiếng).

        <img src="./images/feed-publishing-deep-dive.png" alt="Feed Publishing Deep Dive" width="500">

    **Dịch vụ truyền phát** hoạt động như sau:

    1. **Lấy ID Bạn bè:** Lấy danh sách bạn bè từ cơ sở dữ liệu đồ thị (graph database).
    2. **Lọc Bạn bè từ Cache:** Truy cập cài đặt người dùng trong cache để loại trừ một số bạn bè nhất định (ví dụ: bạn bè đã bị tắt tiếng hoặc tùy chọn chia sẻ có chọn lọc).
    3. **Gửi tới Hàng đợi Tin nhắn:** Gửi danh sách bạn bè đã được lọc cùng với ID bài đăng mới tới hàng đợi tin nhắn (message queue) để xử lý.
    4. **Công nhân Truyền phát (Fanout Workers):** Các worker lấy dữ liệu từ message queue và cập nhật cache của bảng tin. Cache lưu trữ các ánh xạ `<post_id, user_id>` thay vì toàn bộ đối tượng người dùng và bài viết để tiết kiệm bộ nhớ.
    5. **Lưu trữ trong Cache Bảng tin:** Nối các ID bài đăng mới vào cache bảng tin của bạn bè. Một giới hạn cấu hình đảm bảo rằng chỉ các bài đăng gần đây mới được lưu trữ, vì hầu hết người dùng tập trung vào nội dung mới nhất, giúp việc tiêu thụ bộ nhớ cache được kiểm soát.

        <img src="./images/fanout-service.png" alt="Fanout Service" width="500">

## Đi sâu vào Lấy Bảng tin

### Kiến trúc Cache
Cache được chia thành năm lớp:
1. **Cache Bảng tin:** Lưu trữ các ID bài viết để truy xuất nhanh chóng.
2. **Cache Nội dung:** Lưu trữ chi tiết bài viết (bài viết phổ biến trong hot cache).
3. **Cache Đồ thị Xã hội:** Lưu trữ dữ liệu mối quan hệ người dùng.
4. **Cache Hành động:** Theo dõi hành động của người dùng (thích, trả lời, chia sẻ).
5. **Cache Bộ đếm:** Duy trì số đếm cho lượt thích, lượt trả lời, người theo dõi, v.v.

    <img src="./images/cache-architecture.png" alt="Cache Architecture" width="500">
---

## Tối ưu hóa Chính

### Mở rộng Quy mô (Scaling)
1. **Mở rộng Cơ sở dữ liệu:**
   - Mở rộng theo chiều ngang (horizontal scaling) và phân mảnh (sharding).
   - Sử dụng các bản sao chỉ đọc (read replicas) cho các truy vấn lưu lượng cao.
2. **Tầng Web Phi trạng thái (Stateless Web Tier):** Giữ cho các máy chủ web không có trạng thái để có thể mở rộng theo chiều ngang.

### Bộ nhớ đệm (Caching)
1. Lưu trữ dữ liệu thường xuyên truy cập trong bộ nhớ.
2. Sử dụng các lớp cache để giảm độ trễ và tải cơ sở dữ liệu.

### Độ tin cậy (Reliability)
1. **Băm Nhất quán (Consistent Hashing):** Phân phối yêu cầu đồng đều qua các máy chủ.
2. **Hàng đợi Tin nhắn (Message Queues):** Tách rời các thành phần hệ thống và tạo bộ đệm cho lưu lượng truy cập.

### Giám sát (Monitoring)
1. Theo dõi các số liệu chính như QPS (số truy vấn mỗi giây) và độ trễ.
2. Theo dõi tỷ lệ hit cache và điều chỉnh cấu hình phù hợp.