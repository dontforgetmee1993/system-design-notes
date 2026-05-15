# Chương 9: Thiết kế Trình Thu thập Dữ liệu Web (Web Crawler)

## Giới thiệu
Một **trình thu thập dữ liệu web** (web crawler), còn được gọi là spider hoặc robot, được sử dụng để khám phá và thu thập nội dung web, chẳng hạn như các trang web, hình ảnh và video. Chương này tập trung vào việc thiết kế một trình thu thập dữ liệu web có thể mở rộng cho việc **lập chỉ mục công cụ tìm kiếm** (search engine indexing).

### Các Ứng dụng của Trình thu thập Dữ liệu Web
1. **Lập Chỉ mục Công cụ Tìm kiếm:** Thu thập các trang web để tạo ra các chỉ mục có thể tìm kiếm được (ví dụ: Googlebot).
2. **Lưu trữ Web (Web Archiving):** Bảo tồn dữ liệu web để sử dụng trong tương lai (ví dụ: Thư viện Quốc hội Hoa Kỳ).
3. **Khai phá Web (Web Mining):** Trích xuất kiến thức từ dữ liệu web (ví dụ: phân tích tài chính từ các báo cáo cổ đông).
4. **Giám sát Web:** Phát hiện các vi phạm bản quyền hoặc nhãn hiệu.

### Các Thách thức trong Thiết kế
Một trình thu thập dữ liệu web tốt phải giải quyết:
- **Khả năng Mở rộng:** Xử lý hàng tỷ trang web bằng cách sử dụng song song hóa.
- **Tính Mạnh mẽ (Robustness):** Quản lý HTML bị hỏng, sự cố máy chủ và các liên kết độc hại.
- **Sự Lịch sự (Politeness):** Tránh làm quá tải các máy chủ bằng quá nhiều yêu cầu.
- **Khả năng Mở rộng chức năng (Extensibility):** Hỗ trợ các loại nội dung mới với những thay đổi tối thiểu.

---

## Bước 1: Hiểu Vấn đề

### Yêu cầu
1. Thu thập **1 tỷ trang web mỗi tháng** (400 trang/giây, cao điểm 800 QPS).
2. Chỉ thu thập **nội dung HTML**.
3. Theo dõi các trang mới và được cập nhật.
4. Bỏ qua nội dung trùng lặp.
5. Lưu trữ dữ liệu được thu thập trong **5 năm**, yêu cầu khoảng ~30 PB dung lượng lưu trữ.

---

## Bước 2: Thiết kế Cấp cao

### Các Thành phần
<p align="center">
<img src="./images/web-crawler-architecture.png" alt="Web Crawler Architecture" width="700">
</p>

1. **Các URL Hạt giống (Seed URLs):** Các điểm bắt đầu cho trình thu thập dữ liệu.
    - Cần chọn lọc để làm điểm bắt đầu tốt mà trình thu thập dữ liệu có thể sử dụng để duyệt qua càng nhiều liên kết càng tốt.
    - Có thể dựa trên vị trí, trên các trang web phổ biến khác nhau hoặc dựa trên chủ đề.
    - Chiến lược: Phân loại theo vị trí hoặc chủ đề (ví dụ: thể thao, chăm sóc sức khỏe).

2. **Tiền tuyến URL (URL Frontier):** Lưu trữ các URL sẽ được tải xuống.
   - Được triển khai như một **hàng đợi FIFO**.

3. **Trình tải xuống HTML (HTML Downloader):** Tải xuống các trang web từ các URL do URL Frontier cung cấp.

4. **Trình phân giải DNS (DNS Resolver):** Chuyển đổi các URL thành các địa chỉ IP.

5. **Trình Phân tích Nội dung (Content Parser):** Xác thực và phân tích các trang web.
   - Loại bỏ các trang bị lỗi định dạng.

6. **Đã thấy Nội dung chưa? (Content Seen?):** Kiểm tra nội dung trùng lặp bằng cách sử dụng so sánh mã băm (so sánh các giá trị băm của hai trang web).

7. **Lưu trữ Nội dung (Content Storage):** Lưu trữ các trang HTML trên đĩa (nội dung phổ biến trong bộ nhớ đệm để giảm độ trễ).

8. **Trình Trích xuất URL (URL Extractor):** Trích xuất các liên kết mới từ các trang đã được phân tích.

9. **Bộ lọc URL (URL Filter):** Loại trừ các URL nằm trong danh sách đen (blacklist) hoặc có lỗi.

10. **Đã thấy URL chưa? (URL Seen?):** Theo dõi các URL đã truy cập để tránh trùng lặp.

11. **Lưu trữ URL (URL Storage):** Lưu trữ các URL đã truy cập.

---

### Luồng Hoạt động (Workflow)
1. Thêm **Các URL Hạt giống** vào URL Frontier.
2. **Trình tải xuống HTML** lấy các URL và phân giải IP của chúng qua Trình phân giải DNS.
3. **Trình Phân tích Nội dung** xác thực và chuyển nội dung đến thành phần "Đã thấy Nội dung chưa?".
4. Nếu nội dung là mới, trích xuất các liên kết qua **Trình Trích xuất URL**.
5. Lọc và thêm các liên kết duy nhất vào URL Frontier.

---

## Bước 3: Đi sâu vào Các Thành phần Chính
### DFS/BFS
- Web có thể được coi là một đồ thị có hướng trong đó các trang web là các nút và siêu liên kết (URLs) là các cạnh.
- BFS (Tìm kiếm theo chiều rộng) thường được sử dụng cho việc duyệt đồ thị vì độ sâu có thể rất lớn nên DFS (Tìm kiếm theo chiều sâu) không lý tưởng.
- BFS tiêu chuẩn không xem xét mức độ ưu tiên của một URL, không phải mọi trang đều có cùng mức chất lượng và tầm quan trọng.

### Tiền tuyến URL (URL Frontier)
- **Sự Lịch sự (Politeness):** 
    - Đảm bảo chỉ có một yêu cầu cho mỗi máy chủ tại một thời điểm. Thêm một khoảng trễ giữa hai tác vụ tải xuống.
    - Sử dụng một ánh xạ từ tên máy chủ đến các hàng đợi và các luồng làm việc (download).
    - Mỗi luồng tải xuống có một hàng đợi FIFO riêng và chỉ tải xuống các URL từ hàng đợi đó.

        <img src="./images/politeness.png" alt="Politeness" width="500">

    - **Bộ định tuyến hàng đợi (Queue router):** Đảm bảo rằng mỗi hàng đợi (b1, b2, … bn) chỉ chứa các URL từ cùng một máy chủ.
    - **Bảng ánh xạ (Mapping table):** Ánh xạ mỗi máy chủ tới một hàng đợi.
    - **Bộ chọn hàng đợi (Queue selector):** Mỗi luồng làm việc được ánh xạ tới một hàng đợi FIFO, và nó chỉ tải xuống các URL từ hàng đợi đó. Logic chọn hàng đợi được thực hiện bởi Bộ chọn hàng đợi.
    - **Luồng làm việc 1 đến N (Worker thread 1 to N):** Một luồng làm việc tải xuống các trang web một cách tuần tự từ cùng một máy chủ. Một khoảng trễ có thể được thêm vào giữa hai tác vụ tải xuống.

- **Độ Ưu tiên (Priority):** 
    - Gán độ ưu tiên cao hơn cho các trang quan trọng (ví dụ: bằng PageRank hoặc tần suất cập nhật).

        <img src="./images/prioritizer.png" alt="Politeness" width="500">
    
    - **Bộ ưu tiên (Prioritizer):** Lấy các URL làm đầu vào và tính toán các mức độ ưu tiên.
    - **Hàng đợi f1 đến fn:** Mỗi hàng đợi có một độ ưu tiên được gán. Các hàng đợi có mức độ ưu tiên cao được chọn với xác suất cao hơn.
    - **Bộ chọn hàng đợi (Queue selector):** Chọn ngẫu nhiên một hàng đợi với xu hướng thiên về các hàng đợi có mức độ ưu tiên cao hơn.
    - **Các hàng đợi phía trước (Front queues):** quản lý độ ưu tiên.
    - **Các hàng đợi phía sau (Back queues):** quản lý sự lịch sự.

- **Độ mới (Freshness):** Thu thập lại dựa trên lịch sử cập nhật hoặc tầm quan trọng.

### Trình tải xuống HTML
- **Tuân thủ Robots.txt:** Tôn trọng các quy tắc trong các tệp robots.txt.
- **Tối ưu hóa Hiệu suất:**
  1. Thu thập dữ liệu phân tán bằng cách sử dụng nhiều máy chủ.
  2. Sử dụng **bộ nhớ đệm DNS** để tránh các tra cứu lặp lại.
  3. Phân tán các máy chủ thu thập dữ liệu theo địa lý để tải xuống nhanh hơn.
  4. Sử dụng thời gian chờ (timeout) ngắn để tránh các máy chủ chậm hoặc không phản hồi.

### Tính Mạnh mẽ (Robustness)
1. **Băm Nhất quán:** Phân phối tải giữa các máy chủ một cách hiệu quả.
2. **Xử lý Lỗi:** Ngăn chặn hệ thống gặp sự cố do các ngoại lệ (exceptions).
3. **Xác thực Dữ liệu:** Đảm bảo tính toàn vẹn của nội dung.

### Khả năng Mở rộng chức năng (Extensibility)
- Thêm các mô-đun cho các loại nội dung mới (ví dụ: trình tải xuống PNG, trình giám sát web).
- Ví dụ: Cắm thêm một mô-đun để giám sát nội dung web đối với các vi phạm bản quyền.

    <img src="./images/extensibility.png" alt="Politeness" width="600">
---

### Tránh Nội dung Có vấn đề
1. **Nội dung Trùng lặp:** Phát hiện bằng cách sử dụng so sánh băm.
2. **Bẫy Spider (Spider Traps):** Tránh các vòng lặp vô hạn bằng các kỹ thuật như giới hạn chiều dài URL.
3. **Dữ liệu Nhiễu:** Lọc nội dung không liên quan như quảng cáo hoặc thư rác.

---

## Bước 4: Tổng kết
### Các Bài học Chính
1. Các trình thu thập dữ liệu web phải cân bằng giữa khả năng mở rộng, tính mạnh mẽ, sự lịch sự và khả năng mở rộng chức năng.
2. **Sự lịch sự** ngăn chặn việc làm quá tải các máy chủ, trong khi **độ ưu tiên** đảm bảo các trang quan trọng được thu thập trước.
3. Lưu trữ hiệu quả và xử lý lỗi là cực kỳ quan trọng để xử lý thu thập dữ liệu quy mô lớn.

### Các Xem xét Bổ sung
- **Render phía Máy chủ (Server-Side Rendering):** Xử lý nội dung động được tạo ra bởi JavaScript hoặc AJAX.
- **Các Biện pháp Chống Spam:** Loại trừ các trang chất lượng thấp hoặc không liên quan.
- **Phân mảnh Cơ sở dữ liệu (Database Sharding):** Mở rộng tầng dữ liệu bằng cách sử dụng nhân bản và phân mảnh.
- **Mở rộng theo chiều ngang (Horizontal Scaling):** Sử dụng các máy chủ không trạng thái (stateless) để mở rộng các công việc thu thập dữ liệu một cách hiệu quả.
- **Phân tích (Analytics):** Thu thập và phân tích dữ liệu để có thông tin chi tiết.
