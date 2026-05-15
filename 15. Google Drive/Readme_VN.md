# Chương 15: Thiết kế Google Drive

## Giới thiệu
Google Drive là một dịch vụ lưu trữ và đồng bộ hóa tệp tin dựa trên đám mây, cho phép người dùng lưu trữ, truy cập và chia sẻ tệp từ nhiều thiết bị khác nhau. Chương này thảo luận về việc thiết kế một hệ thống có khả năng mở rộng với các tính năng sau:
- **Tải lên và Tải xuống tệp**
- **Đồng bộ hóa tệp trên các thiết bị**
- **Chia sẻ tệp**
- **Lịch sử phiên bản tệp**
- **Thông báo về các thay đổi (chỉnh sửa, xóa và chia sẻ)**

---

## Bước 1: Hiểu vấn đề

### Các yêu cầu chính
#### Yêu cầu chức năng:
- Tải lên và tải xuống tệp.
- Đồng bộ hóa tệp trên nhiều thiết bị.
- Duy trì các phiên bản của tệp.
- Cho phép chia sẻ tệp với các quyền hạn cụ thể.
- Gửi thông báo khi tệp được chỉnh sửa, xóa hoặc chia sẻ.

#### Yêu cầu phi chức năng:
- **Độ tin cậy:** Không chấp nhận mất mát dữ liệu.
- **Tốc độ đồng bộ nhanh:** Tránh để người dùng phải chờ đợi lâu khi đồng bộ.
- **Hiệu quả băng thông:** Giảm thiểu việc sử dụng dữ liệu không cần thiết.
- **Khả năng mở rộng:** Xử lý 10 triệu người dùng hoạt động hàng ngày (DAU).
- **Tính khả dụng cao:** Hoạt động liền mạch ngay cả khi máy chủ gặp sự cố hoặc có vấn đề về mạng.

### Các ràng buộc và giả định
- Người dùng nhận được **10 GB dung lượng miễn phí**.
- Kích thước tệp tối đa: **10 GB**.
- Kích thước tải lên trung bình: **500 KB**.
- Tần suất tải lên: **2 tệp mỗi ngày trên mỗi người dùng**.
- Tổng dung lượng lưu trữ yêu cầu: **500 PB**.

---

## Bước 2: Thiết kế ở mức cao (High-Level Design)
### Thiết lập máy chủ đơn (Single-Server Setup)
Một thiết lập cơ bản bao gồm:
1. **Web Server:** Xử lý việc tải lên và tải xuống.
2. **Metadata Database:** Theo dõi metadata như dữ liệu người dùng, thông tin đăng nhập, thông tin tệp.
3. **Storage Directory:** Chứa các tệp được sắp xếp theo các không gian tên (namespaces).


<div style="margin-left:3rem">
    <img src="./images/namespaces.png" alt="Namespaces" width="400" />
</div>

- Một máy chủ web và một thư mục gọi là drive/ được thiết lập làm thư mục gốc để lưu trữ các tệp được tải lên. 
- Dưới thư mục drive/, có một danh sách các thư mục được gọi là namespaces. 
- Mỗi namespace chứa tất cả các tệp đã tải lên của người dùng đó. 
- Mỗi tệp hoặc thư mục có thể được xác định duy nhất bằng cách kết hợp namespace và đường dẫn tương đối.


Thiết kế này là một điểm bắt đầu nhưng không đủ để mở rộng quy mô.

#### Các API
1. **Tải một tệp lên Google Drive:** Hỗ trợ hai loại tải lên:
    - Tải lên đơn giản: Được sử dụng khi kích thước tệp nhỏ.
    - Tải lên có thể tiếp tục (Resumable upload): 
        - Điểm cuối (Endpoint): https://api.example.com/files/upload?uploadType=resumable
        - Gửi yêu cầu ban đầu để lấy URL có thể tiếp tục.
        - Tải dữ liệu lên và theo dõi trạng thái tải lên.
        - Nếu việc tải lên bị gián đoạn, hãy tiếp tục lại.
2. **Tải một tệp từ Google Drive:** Để tải xuống tệp:
    - Điểm cuối: https://api.example.com/files/download
3. **Lấy danh sách các phiên bản tệp:**
    - Điểm cuối: https://api.example.com/files/list_revisions

### Chuyển sang hệ thống phân tán

#### Các cải tiến:
1. **Sharding (Phân mảnh):** Chia nhỏ bộ lưu trữ trên các máy chủ dựa trên `user_id`.
2. **Amazon S3:** Sử dụng S3 để lưu trữ tệp có khả năng mở rộng và dư thừa với tính năng sao chép xuyên vùng (cross-region replication).

    <img src="./images/replication.png" alt="Replication" width="600" />
     
3. **Load Balancer (Bộ cân bằng tải):** Phân phối lưu lượng truy cập trên nhiều máy chủ web.
4. **Metadata Database Replication:** Đảm bảo tính khả dụng thông qua việc phân mảnh và sao chép cơ sở dữ liệu.


#### Xung đột đồng bộ (Sync Conflicts):
Đối với một hệ thống lưu trữ lớn như Google Drive, các xung đột đồng bộ thỉnh thoảng sẽ xảy ra.
Khi hai người dùng sửa đổi cùng một tệp hoặc thư mục cùng một lúc, một xung đột sẽ phát sinh.

<div style="margin-left:5rem">
<img src="./images/sync-conflicts.png" alt="Sync Conflicts" width="600" />
</div>

- Trong ví dụ, người dùng 1 và người dùng 2 cố gắng cập nhật cùng một tệp cùng một lúc, nhưng tệp của người dùng 1 được hệ thống xử lý trước.
- Hoạt động cập nhật của người dùng 1 được thực hiện, nhưng người dùng 2 nhận được một xung đột đồng bộ. 
- Hệ thống trình bày cả hai bản sao của cùng một tệp: bản sao cục bộ của người dùng 2 và phiên bản mới nhất từ máy chủ.
- Người dùng 2 có tùy chọn gộp cả hai tệp hoặc ghi đè một phiên bản lên phiên bản kia.

### Thiết kế cải tiến
<div style="margin-left:5rem">
<img src="./images/high-level-design.png" alt="High Level Design" width="500" />
</div>

1. **Tương tác người dùng:** Người dùng truy cập ứng dụng qua trình duyệt hoặc ứng dụng di động.

2. **Block Servers (Máy chủ khối):**
   - Các tệp được chia thành các **khối 4 MB** (kích thước tối đa) và được gán các giá trị hash duy nhất.
   - Các khối được lưu trữ độc lập trong bộ lưu trữ đám mây (ví dụ: Amazon S3).
   - Việc tái tạo tệp bao gồm việc kết hợp các khối theo một thứ tự cụ thể.

3. **Cloud Storage:** Các khối được lưu trữ trong bộ lưu trữ đám mây để đảm bảo khả năng mở rộng và dư thừa.

4. **Cold Storage (Lưu trữ lạnh):** Các tệp không hoạt động được chuyển đến lưu trữ lạnh để giảm chi phí.

5. **Load Balancer:** Phân phối các yêu cầu đồng đều giữa các máy chủ API để đảm bảo hoạt động hiệu quả.

6. **API Servers:**
   - Xử lý xác thực người dùng, quản lý hồ sơ và cập nhật metadata của tệp.
   - Quản lý tất cả các quy trình công việc không liên quan đến tải lên.

7. **Metadata Database and Cache:**
   - Lưu trữ metadata cho người dùng, tệp, khối và các phiên bản.
   - Metadata được truy cập thường xuyên sẽ được lưu vào cache để truy xuất nhanh hơn.

8. **Notification Service (Dịch vụ thông báo):**
   - Một **hệ thống publisher/subscriber** thông báo cho các client về các thay đổi của tệp (thêm, sửa, xóa).
   - Đảm bảo các client có thể lấy được các bản cập nhật mới nhất.

9. **Offline Backup Queue (Hàng đợi sao lưu ngoại tuyến):** Lưu trữ tạm thời thông tin thay đổi tệp cho các client đang ngoại tuyến để đồng bộ khi họ trực tuyến trở lại.

---

## Bước 3: Thiết kế chi tiết

### Metadata Database
Dưới đây là một phiên bản rút gọn hiển thị các bảng và trường quan trọng nhất.
#### Thiết kế Schema:
- **Bảng User:** Lưu trữ hồ sơ và tùy chọn của người dùng.
- **Bảng File:** Duy trì metadata của tệp (ví dụ: kích thước, tên, đường dẫn).
- **Bảng Block:** Theo dõi các khối tệp để tái tạo lại tệp.
- **Bảng File Version:** Lưu trữ lịch sử phiên bản của tệp.

<div style="margin-left:5rem">
<img src="./images/metadata-database.png" alt="Metadata Database " width="500" />
</div>

---

### Luồng tải lên tệp

1. **Tải tệp lên:**
   - Tệp được chia thành các khối, nén và mã hóa bởi máy chủ khối.
   - Các khối được tải lên các máy chủ khối và được lưu trữ trong S3.
2. **Tải metadata lên:**
   - Client gửi metadata đến máy chủ API.
   - Metadata được lưu trữ trong cơ sở dữ liệu với trạng thái `pending` (đang chờ).
3. **Hoàn tất:**
   - S3 kích hoạt một callback để cập nhật trạng thái tệp thành `uploaded` (đã tải lên).
   - Dịch vụ thông báo thông tin cho những người dùng liên quan.


<div style="margin-left:5rem">
<img src="./images/upload-flow.png" alt="Upload Flow " width="500" />
</div>


---

### Đồng bộ hóa tệp (File Sync)
1. **Delta Sync:** Chỉ chuyển các khối đã bị sửa đổi thay vì toàn bộ tệp.

    <div style="margin-left:2rem">
    <img src="./images/delta-sync.png" alt="Delta Sync" width="400" />
    </div>

2. **Nén:** Các khối được nén bằng thuật toán nén tùy thuộc vào loại tệp. 
3. **Giải quyết xung đột:**
   - Phiên bản được xử lý đầu tiên sẽ thắng.
   - Các phiên bản xung đột được lưu riêng biệt để người dùng tự giải quyết.

<div style="margin-left:5rem">
<img src="./images/file-sync.png" alt="File Synce " width="400" />
</div>

---

### Luồng tải xuống tệp
Luồng tải xuống được kích hoạt khi một tệp được thêm hoặc chỉnh sửa ở một nơi khác. Có hai cách để client biết được:
- Nếu client A đang trực tuyến trong khi tệp được thay đổi bởi một client khác, dịch vụ thông báo sẽ thông báo cho client A.
- Nếu client A đang ngoại tuyến trong khi tệp được thay đổi, dữ liệu sẽ được lưu vào cache. Khi client ngoại tuyến trực tuyến trở lại, nó sẽ lấy các thay đổi mới nhất.

Khi client biết tệp đã thay đổi, trước tiên nó yêu cầu metadata qua máy chủ API, sau đó tải xuống các khối để tái tạo tệp.

1. **Kích hoạt:** Dịch vụ thông báo thông tin cho client về các cập nhật tệp.
2. **Lấy Metadata:** Client truy xuất metadata đã cập nhật qua API.
3. **Tải xuống khối:** Client tải xuống các khối đã cập nhật từ máy chủ khối và tái tạo lại tệp.


<div style="margin-left:3rem">
<img src="./images/download-flow.png" alt="Upload Flow " width="600" />
</div>


---

### Dịch vụ thông báo (Notification Service)
1. **Mục đích:** Giữ cho client luôn cập nhật về các thay đổi tệp.
2. **Cơ chế:** Sử dụng **long polling** cho các thông báo không đồng bộ.
3. **Ví dụ:** Khi một tệp được thêm, sửa hoặc xóa, các thông báo sẽ được đẩy đến tất cả các client liên quan.


---

### Tối ưu hóa lưu trữ
1. **De-duplication (Loại bỏ trùng lặp):** Loại bỏ các khối trùng lặp ở cấp độ tài khoản bằng cách so sánh dựa trên mã hash.
2. **Chiến lược quản lý phiên bản:**
   - Giới hạn số lượng các phiên bản được lưu.
   - Ưu tiên các phiên bản gần đây cho các tệp được chỉnh sửa thường xuyên.
3. **Cold Storage (Lưu trữ lạnh):** Chuyển các tệp hiếm khi được truy cập sang các giải pháp lưu trữ rẻ hơn (ví dụ: Amazon S3 Glacier).

---

### Xử lý lỗi
1. **Lỗi bộ cân bằng tải:** Bộ cân bằng tải phụ sẽ hoạt động.
2. **Lỗi máy chủ khối:** Các tác vụ đang chờ xử lý sẽ được gán lại cho các máy chủ khác.
3. **Lỗi cơ sở dữ liệu Metadata:**
   - Nâng cấp một nút phụ (slave) thành nút chính (master).
   - Chuyển hướng lưu lượng truy cập đến các bản sao còn lại.
4. **Lỗi bộ lưu trữ đám mây:** Sử dụng sao chép xuyên vùng để lấy các tệp không khả dụng.
5. **Lỗi dịch vụ thông báo:** Các client kết nối lại với các máy chủ thay thế.
