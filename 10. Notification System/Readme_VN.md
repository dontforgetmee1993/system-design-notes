# Chương 10: Thiết kế Hệ thống Thông báo

## Giới thiệu
Một **hệ thống thông báo** là thiết yếu cho các ứng dụng hiện đại, cung cấp các cập nhật kịp thời như thông báo sản phẩm, sự kiện, ưu đãi và cảnh báo. Thông báo có thể được gửi thông qua:
1. **Thông báo đẩy (Push notifications)** (di động hoặc máy tính để bàn),
2. **Tin nhắn SMS**, và
3. **Email**.

Chương này tập trung vào việc thiết kế một hệ thống có khả năng mở rộng, có thể gửi hàng triệu thông báo mỗi ngày.

---

## Bước 1: Hiểu Vấn đề
### Yêu cầu
- **Các loại thông báo:** Thông báo đẩy, SMS và Email.
- **Phân phối:** Hệ thống thời gian thực mềm (soft real-time) với độ trễ tối thiểu.
- **Nền tảng:** iOS, Android và máy tính để bàn.
- **Kích hoạt (Triggers):** Thông báo có thể được kích hoạt bởi các ứng dụng khách hoặc được lên lịch trên máy chủ.
- **Quy mô:**
  - **Thông báo Đẩy:** 10 triệu/ngày,
  - **SMS:** 1 triệu/ngày,
  - **Email:** 5 triệu/ngày.
- **Hỗ trợ Chọn không tham gia (Opt-out):** Người dùng có thể vô hiệu hóa các loại thông báo cụ thể.

---

## Bước 2: Thiết kế Cấp cao

### Các Thành phần

1. **Các Loại Thông báo:**
   - **Thông báo đẩy iOS:** Sử dụng **Dịch vụ Thông báo Đẩy của Apple (APNS)**.
   - **Thông báo đẩy Android:** Sử dụng **Firebase Cloud Messaging (FCM)**.
   - **Tin nhắn SMS:** Các dịch vụ của bên thứ ba như Twilio hoặc Nexmo.
   - **Email:** Các dịch vụ email thương mại như SendGrid hoặc Mailchimp.

2. **Thu thập Thông tin Liên hệ:**
   <div style="margin-left:3rem">
      <img src="./images/contact-info-gathering.png" alt="Contact Info Gathering" width="500">
   </div>

   - Thu thập mã thông báo thiết bị (device tokens), số điện thoại hoặc địa chỉ email trong quá trình cài đặt ứng dụng hoặc đăng ký.
   - Lưu trữ thông tin liên hệ trong cơ sở dữ liệu:
     - **Bảng Device Tokens:** Dành cho thông báo đẩy.
     - **Bảng Người dùng (User Table):** Dành cho email và số điện thoại.

3. **Luồng Gửi Thông báo:**

   <div style="margin-left:3rem">
      <img src="./images/high-level-design.png" alt="High Level Design" width="500">
   </div>

   - **Các Dịch vụ Kích hoạt (Trigger Services):**
      - Tạo các sự kiện để bắt đầu thông báo (ví dụ: nhắc nhở thanh toán, cập nhật giao hàng).
      - Một dịch vụ có thể là một micro-service, một cron job, hoặc một hệ thống phân phân tán kích hoạt các sự kiện gửi thông báo.
   - **Máy chủ Thông báo (Notification Server):** 
      - Cung cấp các API cho các dịch vụ để gửi thông báo. 
      - Thực hiện các xác thực cơ bản để xác minh email, số điện thoại.
      - Truy vấn cơ sở dữ liệu hoặc bộ nhớ đệm để lấy dữ liệu cần thiết để hiển thị một thông báo.
   - **Dịch vụ của Bên Thứ ba:** Phân phối thông báo tới người dùng.

### Thách thức trong Thiết kế Ban đầu
- **Điểm Lỗi Đơn lẻ (SPOF):** Một máy chủ thông báo có thể làm sập toàn bộ hệ thống.
- **Vấn đề Khả năng Mở rộng:** Khó mở rộng cơ sở dữ liệu, bộ nhớ đệm và các thành phần xử lý một cách độc lập.
- **Nút thắt Hiệu suất:** Yêu cầu tài nguyên cao cho việc gửi thông báo.

### Thiết kế Cải tiến

   <div style="margin-left:3rem">
      <img src="./images/improved-design.png" alt="Improved Design" width="500">
   </div>

- Chuyển cơ sở dữ liệu và bộ nhớ đệm ra khỏi máy chủ thông báo.
- Giới thiệu **mở rộng theo chiều ngang (horizontal scaling)** với nhiều máy chủ thông báo.
- Sử dụng **hàng đợi tin nhắn (message queues)** để tách rời (decouple) các thành phần hệ thống.
   - Hàng đợi tin nhắn hoạt động như bộ đệm khi khối lượng thông báo lớn cần được gửi đi.
- Thêm các worker để lấy các sự kiện thông báo từ hàng đợi tin nhắn và gửi chúng đến các dịch vụ bên thứ ba tương ứng.

---

## Bước 3: Đi sâu vào Thiết kế Chi tiết

### Độ Tin cậy (Reliability)
1. **Ngăn chặn Mất Dữ liệu:** 
   <div style="margin-left:3rem">
   <img src="./images/data-loss.png" alt="Data Loss" width="400">
   </div>

   - Lưu trữ dữ liệu thông báo trong cơ sở dữ liệu và triển khai cơ chế thử lại (retry).
   - Cơ sở dữ liệu nhật ký thông báo (Notification log database) được bao gồm cho việc lưu trữ dữ liệu bền vững.

2. **Khử trùng lặp (Deduplication):** 
   - Kiểm tra ID sự kiện để tránh gửi thông báo trùng lặp.
   - Khi một sự kiện thông báo đến lần đầu, hãy kiểm tra xem nó đã được nhìn thấy trước đó chưa bằng cách kiểm tra ID sự kiện.
Nếu đã thấy trước đó, hãy loại bỏ nó, nếu không thì gửi thông báo.

### Các Thành phần Bổ sung
   <div style="margin-left:3rem">
   <img src="./images/events-tracking.png" alt="Events Tracking" width="400">
   </div>

1. **Mẫu Thông báo (Notification Templates):** Các mẫu được định dạng sẵn cho các thông báo nhất quán và hiệu quả.
2. **Cài đặt Thông báo:**
   - Người dùng có thể chọn tham gia hoặc không tham gia đối với các kênh cụ thể (đẩy, SMS hoặc email).
   - Được lưu trữ trong một bảng cài đặt thông báo chuyên dụng.
3. **Giới hạn Tốc độ (Rate Limiting):** Giới hạn tần suất gửi thông báo cho người dùng.
4. **Cơ chế Thử lại (Retry Mechanism):** Thử gửi lại thông báo nếu các dịch vụ bên thứ ba gặp lỗi.
5. **Giám sát Hàng đợi:** Theo dõi các thông báo trong hàng đợi để mở rộng các worker một cách động.
6. **Theo dõi Sự kiện:** Thu thập các số liệu như tỷ lệ mở, tỷ lệ nhấp và mức độ tương tác.

### Bảo mật
- Sử dụng **AppKey** và **AppSecret** để xác thực và bảo mật các API cho thông báo đẩy.

### Luồng Thông báo

   <div style="margin-left:3rem">
   <img src="./images/updated-design.png" alt="Updated Design" width="500">
   </div>

1. Các dịch vụ kích hoạt gọi API để gửi thông báo.
2. Máy chủ thông báo xác thực yêu cầu và lấy siêu dữ liệu (metadata) từ bộ nhớ đệm hoặc cơ sở dữ liệu.
3. Các sự kiện thông báo được gửi đến hàng đợi tin nhắn.
4. Các worker xử lý sự kiện và tương tác với các dịch vụ bên thứ ba.
5. Các dịch vụ bên thứ ba phân phối thông báo tới người dùng.

---

## Các Tối ưu hóa Chính
1. **Mở rộng theo Chiều ngang:** Thêm nhiều máy chủ thông báo hơn để phân phối tải.
2. **Hàng đợi Tin nhắn:** Tách rời quá trình xử lý để xử lý khối lượng lớn.
3. **Bộ nhớ Đệm (Caching):** Giảm độ trễ bằng cách lưu vào bộ nhớ đệm dữ liệu thường xuyên truy cập.
4. **Thu thập Phân tán:** Tối ưu hóa phân phối tin nhắn theo địa lý để có hiệu suất tốt hơn.