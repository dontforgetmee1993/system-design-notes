# Chương 4: Thiết kế Bộ Hạn chế Tốc độ (Rate Limiter)

## Giới thiệu
Chương này khám phá việc thiết kế và triển khai một bộ hạn chế tốc độ (rate limiter)—một thành phần hệ thống được sử dụng để kiểm soát tốc độ lưu lượng được gửi bởi các máy khách hoặc dịch vụ. Bộ hạn chế tốc độ là cực kỳ quan trọng để ngăn chặn việc lạm dụng, giảm chi phí và đảm bảo tính ổn định của tài nguyên máy chủ. Ví dụ về việc sử dụng chúng bao gồm hạn chế bài đăng, tạo tài khoản và nhận phần thưởng.

## Lợi ích của việc Hạn chế Tốc độ
- **Ngăn chặn các cuộc tấn công DoS:** Chặn các cuộc gọi dư thừa để tránh cạn kiệt tài nguyên.
- **Giảm chi phí:** Hạn chế các yêu cầu không cần thiết để giảm chi phí máy chủ.
- **Ngăn chặn quá tải:** Lọc bỏ các yêu cầu quá mức để ổn định hiệu suất máy chủ.

## Bước 1: Hiểu Vấn đề
### Các tính năng chính
- Bộ hạn chế tốc độ API phía máy chủ.
- Hỗ trợ nhiều quy tắc điều tiết (throttle rules).
- Xử lý các hệ thống quy mô lớn trong môi trường phân tán.
- Tùy chọn cho một dịch vụ độc lập hoặc mã cấp ứng dụng.
- Thông báo cho người dùng khi bị hạn chế.

### Yêu cầu
- Điều tiết yêu cầu chính xác.
- Độ trễ tối thiểu.
- Sử dụng ít bộ nhớ.
- Khả năng phân tán.
- Xử lý ngoại lệ rõ ràng.
- Khả năng chịu lỗi cao.

## Bước 2: Thiết kế Cấp cao
### Các tùy chọn đặt vị trí
<div style="margin-left:2rem">
    <img src="./images/rate_limiter_architecture.png"  alt="Rate Limiting Middleware Architecture" width="550">
</div>

1. **Triển khai phía Máy khách:** Không đáng tin cậy do khả năng bị lạm dụng.
2. **Triển khai phía Máy chủ:** Được ưu tiên vì khả năng kiểm soát và độ tin cậy.
3. **Middleware (API Gateway):** Một tùy chọn linh hoạt để tích hợp hạn chế tốc độ.

### Hướng dẫn đặt vị trí
- Đánh giá ngăn xếp công nghệ hiện tại và chọn các tùy chọn hiệu quả.
- Chọn thuật toán phù hợp dựa trên nhu cầu kinh doanh.
- Sử dụng API gateway nếu áp dụng microservices.
- Lựa chọn các giải pháp thương mại nếu tài nguyên có hạn.

## Bước 3: Các thuật toán Hạn chế Tốc độ
### 1. Thùng Token (Token Bucket)
<div style="margin-left:2rem">
  <img src="./images/token-bucket.png"  alt="Token Bucket Algorithm" width="550">
</div>

- **Mô tả:** Các token được thêm vào một cái thùng với tốc độ cố định; mỗi yêu cầu tiêu thụ một token.
- **Tham số:** Kích thước thùng và tốc độ nạp lại.
- **Ưu điểm:** Dễ triển khai, tiết kiệm bộ nhớ, hỗ trợ lưu lượng tăng đột biến (burst traffic).
- **Nhược điểm:** Cần điều chỉnh tham số cẩn thận.

### 2. Thùng Rò rỉ (Leaking Bucket)
<div style="margin-left:2rem">
  <img src="./images/leaking-bucket.png"  alt="Leaking Bucket Algorithm" width="550">
</div>

- **Mô tả:** Xử lý các yêu cầu với tốc độ cố định bằng cách sử dụng hàng đợi FIFO.
- **Ưu điểm:** Tiết kiệm bộ nhớ, tốc độ đầu ra ổn định.
- **Nhược điểm:** Lưu lượng tăng đột biến có thể làm chậm các yêu cầu gần đây.

  Ví dụ: https://github.com/uber-go/ratelimit

### 3. Bộ đếm Cửa sổ Cố định (Fixed Window Counter)
<div style="margin-left:2rem">
  <img src="./images/fixed-window-counter.png"  alt="Fixed Window Counter" width="550">
</div>

- **Mô tả:** Chia thời gian thành các khoảng cố định và sử dụng bộ đếm để hạn chế yêu cầu.
- **Ưu điểm:** Đơn giản, hiệu quả cho các trường hợp sử dụng cụ thể.
- **Nhược điểm:** Lưu lượng tăng vọt ở các cạnh cửa sổ có thể vượt quá giới hạn.

- Lưu lượng tăng đột ngột ở các cạnh của cửa sổ thời gian có thể khiến nhiều yêu cầu hơn hạn mức cho phép đi qua.

  <img src="./images/fixed-window-issue.png"  alt="Fixed Window Issue" width="550">

### 4. Nhật ký Cửa sổ Trượt (Sliding Window Log)
<div style="margin-left:2rem">
  <img src="./images/sliding-window-log.png"  alt="Sliding Window Log" width="550">
</div>

- **Mô tả:** Theo dõi dấu thời gian (timestamps) để cho phép một cửa sổ thời gian cuốn chiếu.
- **Ưu điểm:** Hạn chế tốc độ chính xác.
- **Nhược điểm:** Tiêu tốn nhiều bộ nhớ.

### 5. Bộ đếm Cửa sổ Trượt (Sliding Window Counter)
<div style="margin-left:2rem">
  <img src="./images/sliding-window-counter.png"  alt="Fixed Window Counter" width="550">
</div>

- **Mô tả:** Kết hợp phương pháp cửa sổ cố định và nhật ký trượt để làm mượt các đỉnh lưu lượng.
- **Ưu điểm:** Tiết kiệm bộ nhớ, xử lý tốt lưu lượng tăng đột biến.
- **Nhược điểm:** Ước tính có thể không hoàn toàn chính xác tuyệt đối.

## Kiến trúc Cấp cao
<div style="margin-left:2rem">
  <img src="./images/architecture.png" style="margin-left: 40px; margin-top: 40px; margin-bottom: 20px;" alt="Architecture" width="550">
</div>

- **Lưu trữ dữ liệu:** Sử dụng bộ nhớ đệm trong bộ nhớ (ví dụ: Redis) để thực hiện các thao tác bộ đếm nhanh chóng.
- **Các bước:**
  1. Máy khách gửi yêu cầu đến middleware.
  2. Middleware kiểm tra các bộ đếm trong Redis.
  3. Yêu cầu được xử lý hoặc bị từ chối dựa trên các giới hạn.

## Các xem xét nâng cao
### Môi trường Phân tán
- **Thách thức:** Điều kiện đua (race conditions), vấn đề đồng bộ hóa.
- **Giải pháp:** Sử dụng khóa (locks), script Lua, hoặc sorted sets trong Redis. Sử dụng các kho lưu trữ dữ liệu tập trung để đồng bộ hóa.

### Tối ưu hóa Hiệu suất
- Thiết lập đa trung tâm dữ liệu để giảm độ trễ.
- Mô hình nhất quán cuối cùng (eventual consistency) để đồng bộ hóa.

### Giám sát (Monitoring)
- Phân tích thường xuyên để đảm bảo hiệu quả của thuật toán và điều chỉnh các quy tắc khi cần thiết.
