# Chương 7: Thiết kế Bộ Tạo ID Duy nhất trong Hệ thống Phân tán

## Giới thiệu
Chương này giải quyết thách thức trong việc thiết kế một **bộ tạo ID duy nhất** (unique ID generator) cho các hệ thống phân tán. Các khóa tự động tăng (auto-increment keys) truyền thống không phù hợp trong môi trường phân tán do các thách thức về khả năng mở rộng và đồng bộ hóa. Trọng tâm là tạo ra các ID số 64-bit, có thể sắp xếp được và duy nhất, đáp ứng các yêu cầu sau:
- Các ID phải **duy nhất** và **được sắp xếp theo ngày**.
- Các ID phải vừa vặn trong **64 bit**.
- Hệ thống nên tạo ra **hơn 10.000 ID mỗi giây**.

---

## Bước 1: Hiểu Vấn đề
### Các Yêu cầu Cơ bản
- Các ID phải duy nhất, bằng số và phải vừa trong 64 bit.
- Các ID tăng theo thời gian nhưng không nhất thiết phải tuân ngặt nghèo theo `+1`.
- Các ID phải có thể sắp xếp theo ngày.
- Hệ thống phải xử lý lưu lượng cao (10.000 ID/giây).

---

## Bước 2: Các Lựa chọn Thiết kế Cấp cao
### 1. Sao chép Đa Master (Multi-Master Replication)
- **Phương pháp:** Sử dụng `auto_increment` của cơ sở dữ liệu với các bước tăng (ví dụ: `+k` cho k máy chủ).

    <p align="left">
    <img src="./images/multi-master.png"  alt="Multi Master" width="400">
    </p>

- **Nhược điểm:**
  - Khó mở rộng trên nhiều trung tâm dữ liệu.
  - Các ID không phải lúc nào cũng tăng theo thời gian.
  - Vấn đề mở rộng khi các máy chủ được thêm/loại bỏ.

### 2. UUID (Universally Unique Identifier - Định danh Duy nhất Toàn cầu)
- **Phương pháp:** 
    - Tạo các định danh duy nhất 128-bit một cách độc lập trên mỗi máy chủ bằng UUID.
    - Các UUID có thể được tạo độc lập mà không cần phối hợp giữa các máy chủ.

        <p align="left">
        <img src="./images/uuid.png"  alt="UUID generator" width="600">
        </p>

- **Ưu điểm:**
  - Không cần phối hợp giữa các máy chủ.
  - Mở rộng dễ dàng cùng với các máy chủ web.
- **Nhược điểm:**
  - Vượt quá yêu cầu 64-bit.
  - Các ID không thể sắp xếp theo thời gian và có thể không phải là số.

### 3. Máy chủ Cấp vé (Ticket Server)
- **Phương pháp:** Sử dụng một máy chủ cơ sở dữ liệu tập trung để tăng và gán ID.

    <p align="left">
    <img src="./images/ticket-server.png"  alt="UUID generator" width="500">
    </p>

- **Ưu điểm:**
  - Đơn giản để triển khai cho các hệ thống quy mô nhỏ.
  - Tạo ra các ID bằng số.
- **Nhược điểm:**
  - Điểm lỗi đơn lẻ (Single point of failure).
  - Thách thức về đồng bộ hóa trong các thiết lập nhiều máy chủ.

### 4. Cách tiếp cận Twitter Snowflake
- **Phương pháp:** 

    <div style="margin-left:3rem">
      <img src="./images/twitter-snowflake.png"  alt="Snowflake approach" width="500">
    </div>
    <div style="margin-left:3rem">
      <img src="./images/snowflake-id-breakdown.png"  alt="Snowflake ID breakdow" width="500">
    </div>

    - Chia các ID thành các phần để đảm bảo tính duy nhất và khả năng mở rộng.
    - **Bit dấu (Sign Bit - 1 bit):** Luôn là `0`, có khả năng phân biệt số có dấu và không dấu.
    - **Dấu thời gian (Timestamp - 41 bits):** Số mili-giây kể từ một kỷ nguyên (epoch) tùy chỉnh (Mặc định của Twitter là `1288834974657`, tương đương với ngày 04 tháng 11 năm 2010, 01:42:54 UTC). Đảm bảo các ID được sắp xếp theo thời gian.
    - **ID Trung tâm Dữ liệu (Datacenter ID - 5 bits):** Xác định tối đa `2^5 = 32` trung tâm dữ liệu.
    - **ID Máy (Machine ID - 5 bits):** Xác định tối đa `2^5 = 32` máy trong mỗi trung tâm dữ liệu.
    - **Số thứ tự (Sequence Number - 12 bits):** Theo dõi các ID được tạo ra trên một máy trong cùng một mili-giây, hỗ trợ tối đa `2^12 = 4096` ID mỗi mili-giây. Chuỗi này thiết lập lại về `0` sau mỗi mili-giây.

- **Ưu điểm:**
    - **Khả năng mở rộng:** Xử lý hơn 10.000 ID mỗi giây trên nhiều máy chủ.
    - **Thứ tự thời gian:** Đảm bảo các ID có thể sắp xếp theo thời gian.
    - **Phi tập trung:** Không có điểm lỗi đơn lẻ.

## Bước 4: Các Xem xét Bổ sung
### 1. Đồng bộ hóa Đồng hồ (Clock Synchronization)
- **Thách thức:** Việc tạo ID giả định rằng đồng hồ được đồng bộ hóa trên các máy chủ.
- **Giải pháp:** Sử dụng **Giao thức Thời gian Mạng (NTP)** để giảm thiểu sự sai lệch.

### 2. Điều chỉnh Chiều dài Phần (Section Length Tuning)
- Điều chỉnh kích thước các phần (ví dụ: ít bit số thứ tự hơn, nhiều bit dấu thời gian hơn) dựa trên trường hợp sử dụng.

### 3. Tính Sẵn sàng Cao (High Availability)
- Bộ tạo ID là thiết yếu và phải có khả năng chịu lỗi.
- Xem xét các cơ chế dự phòng và chuyển đổi dự phòng (failover).
