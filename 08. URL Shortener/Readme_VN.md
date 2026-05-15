# Chương 8: Thiết kế Trình Rút gọn URL

## Giới thiệu
Chương này thảo luận về việc thiết kế một dịch vụ rút gọn URL như TinyURL. Mục tiêu chính của hệ thống bao gồm **rút gọn URL**, **chuyển hướng (redirecting)** và **khả năng mở rộng cao** để xử lý khối lượng truy cập lớn.

### Yêu cầu
- Các URL được rút gọn phải **duy nhất** và **ngắn nhất có thể**.
- Xử lý **100 triệu lần tạo URL mỗi ngày** với khả năng hỗ trợ trong 10 năm.
- Hỗ trợ **hoạt động đọc hiệu quả** với tỷ lệ đọc-ghi là 10:1.
- Lưu trữ 365 tỷ bản ghi, yêu cầu khoảng **365 TB** dung lượng lưu trữ trong hơn 10 năm.

---

## Bước 1: Thiết kế Cấp cao

### Các Endpoint API
1. **Rút gọn URL:**  
   - Endpoint: `POST api/v1/data/shorten`  
   - Parameters: `{longUrl: longURLString}`  
   - Returns: `shortURL`

2. **Chuyển hướng URL:**  
   - Endpoint: `GET api/v1/shortUrl`  
   - Returns: `longURL` để chuyển hướng.

    <p align="center">
    <img src="./images/url-redirection.png" alt="URL Redirection" width="600">
    </p>

### Chuyển hướng URL
- **Chuyển hướng 301 (301 Redirect):** Chuyển hướng 301 cho thấy URL được yêu cầu được di chuyển "vĩnh viễn" đến URL dài. Trình duyệt sẽ lưu bộ đệm phản hồi và các yêu cầu tiếp theo đối với cùng một URL sẽ không được gửi đến dịch vụ rút gọn URL.
- **Chuyển hướng 302 (302 Redirect):** Tạm thời; hữu ích cho việc phân tích như theo dõi các lần nhấp chuột.

### Rút gọn URL
<p align="center">
    <img src="./images/url-shortening.png" alt="URL Shortening" width="400">
</p>

- Sử dụng một **hàm băm (hash function)** để tạo URL ngắn, ánh xạ URL dài thành các phiên bản rút gọn duy nhất.
- Hàm băm phải đáp ứng các yêu cầu sau:
    - Mỗi longURL phải được băm thành một hashValue.
    - Mỗi hashValue có thể được ánh xạ lại thành longURL.

---

## Bước 2: Đi sâu vào Thiết kế Chi tiết

### Mô hình Dữ liệu
Lưu trữ ánh xạ `<shortURL, longURL>` trong cơ sở dữ liệu quan hệ để tối ưu hóa việc sử dụng bộ nhớ. Lược đồ bảng bao gồm:
- `id` (khóa chính),
- `shortURL`,
- `longURL`.

    <img src="./images/table-schema.png" alt="Table Schema" width="300">

### Hàm băm
#### 1. Chuyển đổi Cơ số 62 (Base 62 Conversion):
- Mã hóa các số bằng cách sử dụng các ký tự `[0-9, a-z, A-Z]`, cung cấp **62 ký tự có thể có**.
- Chuyển đổi cơ số là một cách tiếp cận khác thường được sử dụng cho các trình rút gọn URL.
- Một id duy nhất có thể được gán cho url ngắn và ID có thể được chuyển đổi theo cơ số 62 để có được URL ngắn.
- Một chuỗi băm 7 ký tự hỗ trợ tới **3,5 nghìn tỷ URL duy nhất**, đủ cho 365 tỷ URL.

**Ví dụ:**  
Chuyển đổi ID `2009215674938` sang Cơ số 62:
- `2009215674938` → `zn9edcu`.

#### 2. Băm + Giải quyết Xung đột (Hash + Collision Resolution):
- Sử dụng các hàm băm như CRC32, MD5, hoặc SHA-1.

    <img src="./images/hash-function.png" alt="Hash Function" width="500">

- Một cách tiếp cận là thu thập 7 ký tự đầu tiên của giá trị băm; tuy nhiên, phương pháp này có thể dẫn đến xung đột băm.
- Để giải quyết xung đột, nối thêm một chuỗi được xác định trước một cách đệ quy cho đến khi không còn xung đột nhưng điều này có thể tốn kém.
- Giải quyết xung đột với **Bộ lọc Bloom (Bloom Filters)** để tra cứu hiệu quả.

    <p align="center">
    <img src="./images/url-lookup.png" alt="URL Lookup" width="500">
    </p>

### So sánh

-  **Băm + Giải quyết Xung đột:**
    - Chiều dài URL ngắn cố định.
    - Không cần bộ tạo ID duy nhất.
    - Xung đột là có thể và cần giải quyết.
    - Không thể tìm thấy URL ngắn khả dụng tiếp theo vì nó không phụ thuộc vào ID.

- **Chuyển đổi Cơ số 62:**
    - Chiều dài không cố định và tăng theo ID.
    - Nó cần một bộ tạo ID duy nhất.
    - Không thể xảy ra xung đột.
    - Dễ dàng tìm thấy URL ngắn tiếp theo nếu ID tăng thêm 1 (Có thể là một vấn đề bảo mật).

---

### Luồng Rút gọn URL

<p align="center">
    <img src="./images/url-shortening-flow.png" alt="URL Shortening" width="500">
</p>

1. Kiểm tra xem `longURL` có tồn tại trong cơ sở dữ liệu không.
2. Nếu tìm thấy, trả về `shortURL` hiện có.
3. Nếu không:
   - Tạo một ID duy nhất bằng cách sử dụng **bộ tạo ID phân tán**.
   - Chuyển đổi ID thành `shortURL` bằng Cơ số 62.
   - Lưu trữ ánh xạ `<id, shortURL, longURL>` trong cơ sở dữ liệu.

---

### Luồng Chuyển hướng URL
<p align="center">
    <img src="./images/url-redirecting-flow.png" alt="URL Shortening" width="600">
</p>

1. Người dùng nhấp vào một `shortURL`.
2. Truy vấn ánh xạ `<shortURL, longURL>`:
   - Kiểm tra **bộ nhớ đệm (cache)** trước để truy cập nhanh hơn.
   - Nếu không có trong bộ nhớ đệm, truy vấn cơ sở dữ liệu.
3. Chuyển hướng người dùng đến `longURL`.

---

## Các Xem xét Bổ sung
### Bộ Hạn chế Tốc độ (Rate Limiter)
- Ngăn chặn việc lạm dụng bằng cách thiết lập giới hạn số lượng yêu cầu trên mỗi IP.

### Khả năng Mở rộng
1. **Tầng Web (Web Tier):** Stateless (không trạng thái), có thể mở rộng bằng cách thêm/loại bỏ các máy chủ web.
2. **Tầng Cơ sở Dữ liệu (Database Tier):** Sử dụng nhân bản (replication) và phân mảnh (sharding).

### Phân tích (Analytics)
- Thu thập dữ liệu như tỷ lệ nhấp chuột, nguồn và dấu thời gian cho thông tin chi tiết về kinh doanh.

### Tính Sẵn sàng Cao và Độ Tin cậy
- Đảm bảo các dịch vụ nhất quán và đáng tin cậy bằng cách sử dụng nhân bản cơ sở dữ liệu và thiết kế chịu lỗi.
