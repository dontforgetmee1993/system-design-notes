# Chương 13: Thiết kế Hệ thống Tự động Hoàn thành Tìm kiếm (Search Autocomplete System)

## Giới thiệu
Tự động hoàn thành (Autocomplete), còn được gọi là typeahead hoặc tìm kiếm tăng dần, cung cấp các đề xuất theo thời gian thực cho người dùng khi họ gõ vào hộp tìm kiếm. Hệ thống phải phân phối hiệu quả top-k đề xuất phù hợp và phổ biến nhất dựa trên dữ liệu truy vấn lịch sử.

### Các Tính năng Chính
- Đề xuất tối đa **5 kết quả tự động hoàn thành**.
- Dựa trên **độ phổ biến của truy vấn** (tần suất).
- Chỉ hỗ trợ **các ký tự tiếng Anh viết thường**.
- Thời gian phản hồi nhanh (<100 ms) và có khả năng mở rộng.

---

## Bước 1: Hiểu rõ vấn đề

### Yêu cầu
1. **Đề xuất Theo Thời gian Thực:** Hiển thị các kết quả phù hợp khi người dùng đang gõ.
2. **Top-k Kết quả:** Trả về tối đa 5 kết quả được sắp xếp theo độ phổ biến.
3. **Khả năng Mở rộng:** Xử lý **10 triệu DAU** với QPS đỉnh điểm là **48.000**.
4. **Tính Sẵn sàng Cao:** Xử lý các lỗi mà hệ thống không bị ngừng hoạt động.
5. **Tăng trưởng Dữ liệu:** Hỗ trợ tăng trưởng lưu trữ hàng ngày **0,4 GB** cho dữ liệu truy vấn mới.

---

## Bước 2: Thiết kế Tổng quan
Ở mức tổng quan, hệ thống được chia thành hai dịch vụ:
1. **Dịch vụ Thu thập Dữ liệu (Data Gathering Service):** 
    - Thu thập các truy vấn của người dùng và tổng hợp chúng để phân tích tần suất theo thời gian thực.
    - Xử lý theo thời gian thực không thực tế đối với các tập dữ liệu lớn; tuy nhiên, nó là một điểm khởi đầu tốt.


2. **Dịch vụ Truy vấn (Query Service):** Cung cấp các đề xuất top-k dựa trên dữ liệu đầu vào của người dùng.

---

### Dịch vụ Thu thập Dữ liệu
<div style="margin-left:3rem">
    <img src="./images/data-gathering.png" alt="Data Gathering" width="600">
</div>

- Tổng hợp dữ liệu truy vấn từ các nhật ký phân tích (analytics logs) và cập nhật bảng tần suất.
- Xử lý dữ liệu lịch sử hàng tuần để xây dựng một **trie** (cây tiền tố).

### Dịch vụ Truy vấn
<div style="margin-left:3rem">
    <img src="./images/frequency-table.png" alt="Frequency Table" width="400">
    <img src="./images/basic-search-suggestions.png" alt="Search Suggestions" width="360">
</div>

- Sử dụng bảng tần suất từ dịch vụ thu thập dữ liệu.
- Xử lý đầu vào của người dùng và lấy các đề xuất top-k từ bảng tần suất sử dụng Trie.
- Tối ưu hóa cho tra cứu nhanh bằng cách sử dụng bộ nhớ đệm (caching) và cấu trúc dữ liệu hiệu quả.
- Ví dụ, khi người dùng gõ "tw" trong hộp tìm kiếm, 5 truy vấn được tìm kiếm nhiều nhất sau đây sẽ được hiển thị.


---

## Bước 3: Đi sâu vào Thiết kế

### Cấu trúc Dữ liệu Trie
**Trie** là một cấu trúc dữ liệu dạng cây được sử dụng để lưu trữ và truy xuất các chuỗi truy vấn một cách hiệu quả.

#### Các Tính năng Chính
1. **Lưu trữ Nhỏ gọn:** Biểu diễn các tiền tố theo hệ phân cấp để giảm thiểu sự dư thừa.
2. **Thông tin Tần suất:** Lưu trữ độ phổ biến của các truy vấn tại mỗi nút.

4. **Các bước để lấy top k truy vấn được tìm kiếm nhiều nhất**
   <div style="margin-left:3rem">
      <img src="./images/trie-structure.png" alt="Trie Structure" width="500">
   </div>

    - Tìm tiền tố
    - Duyệt qua cây con từ nút tiền tố để lấy tất cả các nút con hợp lệ
    - Sắp xếp các nút con và lấy top k 


3. **Tối ưu hóa:**
   - Lưu trữ tạm thời (cache) các truy vấn top-k ở mỗi nút để tăng tốc độ truy xuất và tránh duyệt qua toàn bộ trie.

        <img src="./images/cached-trie.png" alt="Cached Trie" width="600">

   - Giới hạn độ dài tiền tố để giảm không gian tìm kiếm vì người dùng hiếm khi gõ một truy vấn tìm kiếm quá dài (ví dụ: 50 ký tự).

#### Các Thao tác trên Trie
1. **Tạo (Create):** 
    - Được xây dựng hàng tuần sử dụng dữ liệu truy vấn đã được tổng hợp.
    - Nguồn dữ liệu từ Nhật ký Phân tích / Cơ sở dữ liệu.
2. **Cập nhật (Update):** Hiếm khi được cập nhật trong thời gian thực; các bản cập nhật hàng tuần sẽ thay thế dữ liệu cũ.
3. **Xóa (Delete):** 
      <div style="margin-left:3rem">
         <img src="./images/delete-kv.png" alt="Delete KV" width="500">
      </div>

    - Các bộ lọc loại bỏ các đề xuất không mong muốn hoặc có hại (ví dụ: ngôn từ kích động thù địch).
    - Có một lớp lọc cho phép chúng ta linh hoạt loại bỏ các kết quả dựa trên các quy tắc lọc khác nhau.
    - Các đề xuất không mong muốn được xóa vật lý khỏi cơ sở dữ liệu một cách bất đồng bộ.
    

---

### Luồng Xử lý Truy vấn
1. **Tìm kiếm Tiền tố (Prefix Search):**
   - Xác định nút tiền tố tương ứng với đầu vào của người dùng.
   - Duyệt qua cây con để thu thập các đề xuất hợp lệ.
2. **Sắp xếp Top-k:**
   - Lưu trữ các đề xuất top-k ở mỗi nút để giảm thiểu chi phí sắp xếp.
3. **Xây dựng Phản hồi:**
   - Xây dựng kết quả sử dụng dữ liệu được lưu trong bộ đệm để có thời gian phản hồi nhanh.

---

### Tối ưu hóa
1. **Bộ đệm tại Mỗi Nút:**
   - Lưu trữ các truy vấn top-k để tránh các thao tác duyệt dư thừa.
2. **Giới hạn Độ dài Tiền tố:**
   - Giới hạn độ dài tiền tố ở một giá trị nhỏ (ví dụ: 50 ký tự) để tra cứu nhanh hơn.
3. **Yêu cầu AJAX:**
   - Sử dụng các yêu cầu bất đồng bộ nhẹ (lightweight) cho các phản hồi theo thời gian thực.
4. **Bộ đệm Trình duyệt (Browser Caching):**
   - Lưu các kết quả tự động hoàn thành trong bộ đệm của trình duyệt cho các thuật ngữ thường xuyên được tìm kiếm.

---

### Đường ống Thu thập Dữ liệu (Data Gathering Pipeline)
Trong thiết kế tổng quan, bất cứ khi nào người dùng gõ một truy vấn tìm kiếm, dữ liệu sẽ được cập nhật theo thời gian thực. Cách tiếp cận này không thực tế.
- Người dùng có thể nhập hàng tỷ truy vấn mỗi ngày. Việc cập nhật trie cho mọi truy vấn là không khả thi.
- Các đề xuất hàng đầu có thể không thay đổi nhiều khi trie đã được xây dựng.


#### Thiết kế Cập nhật

<div style="margin-left:3rem">
   <img src="./images/data-gathering-flow.png" alt="Updated Data Gathering Flow" width="600">
</div>

1. **Nhật ký Phân tích (Analytics Logs):**
   - Lưu trữ dữ liệu truy vấn thô dưới dạng nhật ký để tổng hợp hàng tuần.
   - Nhật ký là dạng chỉ ghi thêm (append-only) và không được lập chỉ mục.
2. **Bộ tổng hợp (Aggregators):**
   - Xử lý các nhật ký thành các bảng tần suất, phù hợp cho việc xây dựng trie.
   - Đối với các ứng dụng thời gian thực như Twitter, tổng hợp dữ liệu trong một khoảng thời gian ngắn hơn.
   - Đối với các trường hợp khác, việc tổng hợp dữ liệu ít thường xuyên hơn, ví dụ một lần mỗi tuần là đủ tốt.
3. **Công nhân (Workers):**
   - Các máy chủ bất đồng bộ xây dựng lại trie và lưu trữ nó trong kho lưu trữ bền vững.
4. **Tùy chọn Lưu trữ:**
    - **Trie Cache**: Trie Cache là một hệ thống bộ đệm phân tán giữ trie trong bộ nhớ để đọc nhanh.
    - **Trie DB** 
        1. **Kho lưu trữ Tài liệu (Document Store - ví dụ: MongoDB)**: Vì một trie mới được xây dựng hàng tuần, chúng ta có thể chụp nhanh (snapshot) định kỳ, tuần tự hóa nó và lưu trữ dữ liệu đã được tuần tự hóa trong cơ sở dữ liệu như MongoDB.
        2. **Kho lưu trữ Khóa-Giá trị (Key-Value Store):** 
            - Ánh xạ các tiền tố đến dữ liệu nút để truy cập nhanh.
            - Mọi tiền tố trong trie được ánh xạ tới một khóa trong bảng băm.
            - Dữ liệu trên mỗi nút trie được ánh xạ tới một giá trị trong bảng băm.

                <img src="./images/trie-db.png" alt="Trie DB" width="600">
---

### Khả năng Mở rộng
1. **Phân mảnh (Sharding):**
   - Phân phối các nút trie qua các máy chủ dựa trên phạm vi tiền tố (ví dụ: `a-m`, `n-z`).
   - Phân mảnh sâu hơn bên trong các tiền tố để cân bằng các phân phối không đồng đều (ví dụ: `aa-ag`, `ah-an`).
2. **Cân bằng Tải (Load Balancing):**
   <div style="margin-left:3rem">
      <img src="./images/sharding.png" alt="Sharding" width="400">
   </div>

   - Sử dụng một trình quản lý bản đồ phân mảnh (shard map manager) để định tuyến các yêu cầu đến máy chủ thích hợp.


---

## Bước 4: Các Tính năng Nâng cao

### Hỗ trợ Đa Ngôn ngữ
1. **Ký tự Unicode:** Sử dụng Unicode để hỗ trợ các ngôn ngữ không phải tiếng Anh.
2. **Trie Đặc thù theo Quốc gia:** Xây dựng các trie riêng biệt cho các quốc gia hoặc khu vực khác nhau.

### Các Truy vấn Thịnh hành (Trending Queries)
- Xử lý các sự kiện thời gian thực bằng cách cập nhật động các nút trie hoặc gán trọng số lớn hơn cho các truy vấn gần đây.