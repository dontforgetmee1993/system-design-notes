# Chương 2: Ước tính nhanh (Back-of-the-Envelope Estimation)

## Giới thiệu
Ước tính nhanh (Back-of-the-envelope estimation) là một kỹ năng quan trọng trong các buổi phỏng vấn thiết kế hệ thống. Nó bao gồm việc thực hiện các tính toán nhanh, thô để đánh giá dung lượng hoặc hiệu suất của hệ thống. Theo Jeff Dean, Google Senior Fellow, những ước tính này giúp đánh giá liệu các thiết kế có đáp ứng yêu cầu hay không thông qua các thí nghiệm tư duy và các mốc hiệu suất phổ biến.

Chương này bao gồm các khái niệm chính, phương pháp luận và các ví dụ để xây dựng sự thành thạo trong việc mở rộng quy mô và ước tính.

---

## Phần 1: Các khái niệm chính

### Sức mạnh của lũy thừa cơ số 2 (Power of Two)
Hiểu khối lượng dữ liệu dưới dạng lũy thừa của 2 là điều cơ bản:

<div style="margin-left:3rem">
   <img src="./images/power-of-two.png" alt="power-of-two" width="500" />
</div>

Kiến thức này giúp thực hiện các tính toán lưu trữ và băng thông chính xác.

---

### Các con số về độ trễ mà mọi lập trình viên nên biết (Latency Numbers)
Các con số về độ trễ đại diện cho thời gian thực hiện các thao tác khác nhau trong hệ thống máy tính. Chúng cung cấp cái nhìn sâu sắc về hiệu suất tương đối:

| Thao tác (Operation) | Độ trễ (Latency 2020) |
|----------------------|-----------------------|
| Truy cập L1 Cache    | 0.5 ns                |
| Truy cập L2 Cache    | 7 ns                  |
| Truy cập Bộ nhớ chính (RAM) | 100 ns         |
| Đọc ngẫu nhiên SSD   | 150 µs                |
| Tìm kiếm ngẫu nhiên HDD | 10 ms               |
| Round-Trip trong Data Center | 500 µs       |
| Giữa các Data Center khác vùng | 150 ms     |

**Thông tin chính:**
- Bộ nhớ (Memory) thì nhanh, đĩa (Disk) thì chậm.
- Tránh tìm kiếm trên đĩa (disk seeks) bất cứ khi nào có thể.
- Nén dữ liệu trước khi truyền qua internet để tiết kiệm băng thông.

---

### Các con số về tính sẵn sàng (Availability)
Tính sẵn sàng cao (High availability - HA) đảm bảo thời gian ngừng hoạt động (downtime) là tối thiểu. Tính sẵn sàng được thể hiện bằng các **con số 9**:
- **99% (Hai con số 9):** ~3.65 ngày/năm ngừng hoạt động
- **99.9% (Ba con số 9):** ~8.8 giờ/năm ngừng hoạt động
- **99.99% (Bốn con số 9):** ~52 phút/năm ngừng hoạt động
- **99.999% (Năm con số 9):** ~5.3 phút/năm ngừng hoạt động
- **99.9999% (Sáu con số 9):** ~31.56 giây/năm ngừng hoạt động

Các nhà cung cấp đám mây như Amazon, Google và Microsoft đặt mục tiêu SLA (Service Level Agreements) từ **99.9% trở lên**.

---

## Phần 2: Ví dụ ước tính - QPS của Twitter và yêu cầu lưu trữ

### Các giả định
- **300 triệu người dùng hoạt động hàng tháng (MAU).**
- **50% người dùng hoạt động hàng ngày (DAU).**
- **Trung bình mỗi người dùng đăng 2 tweet/ngày.**
- **10% tweet có chứa phương tiện truyền thông (ảnh, video).**
- **Dữ liệu được lưu giữ trong 5 năm.**

### Ước tính
1. **Số truy vấn mỗi giây (Query Per Second - QPS):**
   - DAU = \( 300M x 50\% = 150M \)
   - Tweets QPS = \( 150M x 2 \text{ tweets} / 24 \text{ giờ} / 3600 \text{ giây} \approx 3500 \)
   - Peak QPS (Đỉnh) = \( 2 x 3500 = 7000 \)

2. **Lưu trữ phương tiện truyền thông (Media Storage):**
   - **Thành phần kích thước Tweet:**
     - `tweet_id`: 64 bytes
     - `text`: 140 bytes
     - `media`: 1 MB
   - **Lưu trữ phương tiện hàng ngày:** \( 150M x 2 x 10\% x 1MB = 30TB \text{ mỗi ngày} \)
   - **Lưu trữ trong 5 năm:** \( 30TB x 365 x 5 \approx 55PB \)

---

## Phần 3: Mẹo để ước tính hiệu quả

### 1. Làm tròn và Xấp xỉ
Độ chính xác tuyệt đối không quan trọng; hãy tập trung vào quy trình. Đơn giản hóa các tính toán phức tạp bằng cách sử dụng các con số tròn. Ví dụ:
- \( 99987 / 9.1 \) có thể được xấp xỉ thành \( 100,000 / 10 = 10,000 \).

### 2. Viết ra các giả định
Ghi lại các giả định một cách rõ ràng để tham khảo sau này.

### 3. Ghi nhãn đơn vị
Tránh sự mơ hồ bằng cách ghi rõ đơn vị (ví dụ: `5 MB` thay vì chỉ ghi `5`).

### 4. Các kịch bản ước tính phổ biến
- **QPS (Queries Per Second):** Đo lường cường độ lưu lượng truy cập.
- **Peak QPS:** Tính đến các đợt lưu lượng truy cập tăng đột biến.
- **Yêu cầu lưu trữ (Storage Requirements):** Ước tính tổng nhu cầu dữ liệu.
- **Yêu cầu bộ nhớ đệm (Cache Requirements):** Đánh giá nhu cầu bộ nhớ cho việc lưu trữ đệm.
- **Số lượng máy chủ:** Tính toán nhu cầu phần cứng dựa trên khối lượng công việc.
