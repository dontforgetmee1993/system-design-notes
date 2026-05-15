# Chương 6: Thiết kế Kho Lưu trữ Key-Value

## Giới thiệu
Một **kho lưu trữ key-value** là một loại cơ sở dữ liệu phi quan hệ, nơi dữ liệu được lưu trữ dưới dạng các cặp khóa-giá trị (key-value pairs). Mỗi khóa là duy nhất và các giá trị được truy cập thông qua các khóa này. Chương này chi tiết cách thiết kế một kho lưu trữ key-value phân tán, có khả năng mở rộng, tính sẵn sàng cao, hỗ trợ các thao tác như:
- `put(key, value)` để chèn dữ liệu.
- `get(key)` để truy xuất dữ liệu.

### Đặc điểm của Thiết kế
- Các cặp key-value nhỏ (<10 KB).
- Hỗ trợ dữ liệu lớn với tính sẵn sàng và khả năng mở rộng cao.
- Tự động mở rộng và tính nhất quán có thể điều chỉnh.
- Độ trễ thấp.

---

## Kho Lưu trữ Key-Value Đơn Máy chủ
### Triển khai
- Sử dụng một **bảng băm (hash table)** để lưu trữ các cặp key-value trong bộ nhớ.
- Tối ưu hóa:
  - Nén dữ liệu.
  - Lưu trữ dữ liệu ít được truy cập hơn trên đĩa.

### Hạn chế
Bộ nhớ của một máy chủ đơn lẻ là có hạn, đòi hỏi một **cách tiếp cận phân tán** để có khả năng mở rộng.

---

## Kho Lưu trữ Key-Value Phân tán
Một **kho lưu trữ key-value phân tán** phân vùng dữ liệu trên nhiều máy chủ và phải giải quyết các đánh đổi được nêu ra bởi **định lý CAP**.

### Định lý CAP
1. **Tính Nhất quán (Consistency):** Tất cả các máy khách thấy cùng một dữ liệu tại cùng một thời điểm.
2. **Tính Sẵn sàng (Availability):** Hệ thống phản hồi mọi yêu cầu, ngay cả khi một số nút bị hỏng.
3. **Khả năng Chịu lỗi Phân vùng (Partition Tolerance):** Hệ thống tiếp tục hoạt động mặc dù có các phân vùng mạng.

**Đánh đổi:** Theo định lý CAP, chỉ có thể đạt được hai trong ba đảm bảo.

<p align="center">
  <img src="./images/cap.png" alt="CAP" width="400">
</p>

#### Các loại hệ thống:
- **Hệ thống CP:** Tính nhất quán và khả năng chịu lỗi phân vùng, hy sinh tính sẵn sàng (ví dụ: hệ thống ngân hàng).
- **Hệ thống AP:** Tính sẵn sàng và khả năng chịu lỗi phân vùng, hy sinh tính nhất quán (ví dụ: nhất quán cuối cùng).
- **Hệ thống CA:** Tính nhất quán và tính sẵn sàng, hy sinh khả năng chịu lỗi phân vùng.

    **Vì lỗi mạng là không thể tránh khỏi, một hệ thống phân tán phải chịu được phân vùng mạng. Do đó, một hệ thống CA không thể tồn tại trong các ứng dụng thực tế.**

    Trong một hệ thống phân tán, các phân vùng là không thể tránh khỏi. Khi một phân vùng xảy ra, chúng ta phải chọn giữa tính nhất quán và tính sẵn sàng. Ví dụ, nếu nút n3 bị hỏng, bất kỳ dữ liệu nào được ghi vào các nút n1 hoặc n2 đều không thể truyền đến n3. Ngược lại, nếu dữ liệu được ghi vào n3 nhưng chưa được truyền đến n1 và n2, các nút n1 và n2 sẽ có dữ liệu cũ (stale data).

    <p align="center">
    <img src="./images/server-down.png"  alt="Server down" width="400">
    </p>
    
- Nếu chúng ta chọn hệ thống CP, chúng ta phải chặn tất cả các hoạt động ghi vào n1 và n2 để tránh sự không nhất quán của dữ liệu.
- Nếu chúng ta chọn hệ thống AP, hệ thống tiếp tục chấp nhận các yêu cầu đọc, mặc dù nó có thể trả về dữ liệu cũ. Đối với các hoạt động ghi, n1 và n2 tiếp tục chấp nhận các bản ghi, và dữ liệu sẽ được đồng bộ hóa với n3 khi phân vùng mạng được giải quyết.

---

## Các thành phần Hệ thống
### 1. Phân vùng Dữ liệu (Data Partitioning)
- **Kỹ thuật:** Băm nhất quán (Consistent Hashing) được sử dụng để phân phối dữ liệu đồng đều trên nhiều máy chủ.
- **Ưu điểm:**
  - Tự động mở rộng khi thêm/loại bỏ máy chủ.
  - Tính không đồng nhất thông qua các nút ảo. Số lượng nút ảo cho một máy chủ tỷ lệ thuận với công suất của máy chủ đó.

### 2. Bản sao Dữ liệu (Data Replication)
- Sao chép dữ liệu trên `N` máy chủ để đảm bảo tính sẵn sàng cao.
- `N` máy chủ được chọn bằng cách đi theo chiều kim đồng hồ từ vị trí máy chủ và chọn N máy chủ đầu tiên trên vòng tròn để lưu trữ các bản sao dữ liệu. Đặt các bản sao ở các trung tâm dữ liệu khác nhau để cải thiện độ tin cậy trong trường hợp sử dụng các nút ảo.

    <p align="center">
    <img src="./images/data-replication.png" alt="Data replication" width="300">
    </p>

### 3. Tính Nhất quán (Consistency)
Vì dữ liệu được sao chép tại nhiều nút, nó phải được đồng bộ hóa giữa các bản sao.
- **Đồng thuận Đa số (Quorum Consensus):**
  - `N`: Tổng số bản sao.
  - `W`: Kích thước đa số ghi (write quorum). Để một bản ghi được coi là thành công, bản ghi đó phải được xác nhận từ W bản sao.
  - `R`: Kích thước đa số đọc (read quorum). Để một bản đọc được coi là thành công, bản đọc đó phải chờ phản hồi từ ít nhất R bản sao.
  - **Quy tắc:** `W + R > N` đảm bảo tính nhất quán mạnh.
  - Cấu hình của W, R và N là một sự đánh đổi điển hình giữa độ trễ và tính nhất quán.

    <p align="center">
    <img src="./images/quorum-consensus.png"   alt="Quorum consensus" width="400">
    </p>
    
    - Nếu R = 1 và W = N, hệ thống được tối ưu hóa cho việc đọc nhanh.
    - Nếu W = 1 và R = N, hệ thống được tối ưu hóa cho việc ghi nhanh.
    - Nếu W + R > N, tính nhất quán mạnh được đảm bảo (Thường là N = 3, W = R = 2).
    - Nếu W + R <= N, tính nhất quán mạnh không được đảm bảo.

- **Mô hình**:
  - **Nhất quán Mạnh (Strong Consistency):** Một hoạt động đọc trả về một giá trị tương ứng với kết quả của mục dữ liệu ghi được cập nhật mới nhất.
  - **Nhất quán Yếu (Weak Consistency):** Các hoạt động đọc tiếp theo có thể không thấy giá trị được cập nhật mới nhất.
  - **Nhất quán Cuối cùng (Eventual Consistency):** Sau một khoảng thời gian đủ lâu, tất cả các bản cập nhật được lan truyền và tất cả các bản sao đều nhất quán.

### 4. Giải quyết Sự không nhất quán
Sao chép mang lại tính sẵn sàng cao nhưng gây ra sự không nhất quán giữa các bản sao. Quản lý phiên bản (Versioning) và khóa vector (vector clocks) được sử dụng để giải quyết các vấn đề không nhất quán.
- **Quản lý phiên bản (Versioning):**
    - Sử dụng **vector clocks** để theo dõi các phiên bản dữ liệu và giải quyết xung đột.
    - Quản lý phiên bản có nghĩa là coi mỗi sửa đổi dữ liệu như một phiên bản bất biến mới của dữ liệu.
        <div>
        <img src="./images/consistent-server.png"   alt="Consisten hashing" width="400">
        <img src="./images/inconsistent-server.png"   alt="Inconsistent server" height="230">
        </div>
    
    - Máy chủ 1 thay đổi tên, và máy chủ 2 cũng thay đổi tên. Hai thay đổi này được thực hiện đồng thời. Bây giờ, chúng ta có các giá trị xung đột, được gọi là các phiên bản v1 và v2.

- **Khóa Vector (Vector Clock)**
    1. **Thiết lập**: Một khóa vector là một cặp [server, version] gắn liền với một mục dữ liệu. Nó có thể được sử dụng để kiểm tra xem một phiên bản có trước, có sau, hay xung đột với các phiên bản khác.
        - Giả sử một khóa vector được đại diện bởi D([S1, v1], [S2, v2], …, [Sn, vn]). Nếu mục dữ liệu D được ghi vào máy chủ Si, hệ thống phải thực hiện một trong các tác vụ sau.
        - Trong đó: `D` là mục dữ liệu. `Si` là định danh máy chủ. `vi` là bộ đếm phiên bản cho dữ liệu tại máy chủ `Si`.

    2. **Cập nhật Khóa Vector:** Khi một mục dữ liệu được sửa đổi tại một máy chủ:
        - Nếu máy chủ đã tồn tại trong khóa vector, bộ đếm phiên bản của nó sẽ tăng lên.
        - Nếu không, một mục mới sẽ được thêm vào khóa vector.

    3. **Phát hiện Xung đột:**
        - **Không có Xung đột:** Phiên bản X là tổ tiên của phiên bản Y nếu tất cả các bộ đếm trong X nhỏ hơn hoặc bằng các bộ đếm tương ứng trong Y.
        - **Có Xung đột:** Hai phiên bản là anh em (siblings) nếu có ít nhất một bộ đếm trong Y nhỏ hơn bộ đếm tương ứng của nó trong X.

    4. **Giải quyết Xung đột:** Khi các xung đột được phát hiện (các phiên bản anh em), hệ thống dựa vào logic cụ thể của ứng dụng hoặc sự can thiệp của máy khách để điều hòa dữ liệu.

        <p align="center">
        <img src="./images/vector-clock.png"  alt="Server hashing" width="500">
        </p>

- **Thách thức:**
  - Tăng độ phức tạp cho máy khách.
  - Kích thước khóa vector có thể tăng lên với nhiều bản cập nhật, đòi hỏi các chiến lược cắt tỉa (trimming) để giới hạn kích thước của nó.

### 5. Xử lý Lỗi

#### a. Phát hiện Lỗi
Không đủ để tin rằng một máy chủ bị hỏng chỉ vì một máy chủ khác nói như vậy. Thông thường, nó yêu cầu ít nhất hai nguồn thông tin độc lập để đánh dấu một máy chủ là hỏng.
- **Giao thức Gossip (Gossip Protocol):**
    <div style="margin-left:3rem">
        <img src="./images/gossip-protocol.png"  alt="Gossip protocol" width="600">
    </div>

    - Mỗi nút duy trì một danh sách các ID thành viên và bộ đếm nhịp tim (heartbeat counters).
    - Mỗi nút định kỳ tăng bộ đếm nhịp tim của nó.
    - Mỗi nút định kỳ gửi nhịp tim đến một tập hợp các nút ngẫu nhiên.
    - Nếu nhịp tim không tăng trong một khoảng thời gian được xác định trước, thành viên đó được coi là ngoại tuyến (offline).

#### b. Lỗi Tạm thời
- **Sloppy Quorum:** Sử dụng các nút khỏe mạnh để duy trì hoạt động tạm thời.
        <p align="center">
        <img src="./images/sloppy-quorum.png"   alt="Sloppy Quorum" width="400">
        </p>

    - Sau khi phát hiện lỗi, hệ thống cần triển khai một số cơ chế nhất định để đảm bảo tính sẵn sàng.
    - Thay vì thực thi yêu cầu quorum nghiêm ngặt, hệ thống chọn W máy chủ khỏe mạnh đầu tiên cho các hoạt động ghi và R máy chủ khỏe mạnh đầu tiên cho các hoạt động đọc trên vòng tròn băm.
    - Các máy chủ ngoại tuyến bị bỏ qua. Nếu một máy chủ không khả dụng, một máy chủ khác sẽ tạm thời xử lý các yêu cầu.

- **Hinted Handoff:** Các máy chủ ngoại tuyến bắt kịp các thay đổi sau khi phục hồi.
    - Khi máy chủ bị hỏng hoạt động trở lại, các thay đổi sẽ được đẩy ngược lại để đạt được tính nhất quán của dữ liệu.

#### c. Lỗi Vĩnh viễn
- Sử dụng **Cây Merkle (Merkle Trees)** để đồng bộ hóa hiệu quả giữa các bản sao.
    Một **Cây Merkle** (hoặc cây băm) là một cấu trúc dữ liệu để phát hiện và giải quyết hiệu quả sự không nhất quán giữa các bản sao trong các lỗi vĩnh viễn.

- Cách hoạt động:
    1. **Cấu trúc:**
        - **Các Nút Lá (Leaf Nodes)** lưu trữ mã băm của các khối dữ liệu riêng lẻ.
        - **Các Nút Không phải Lá (Non-Leaf Nodes)** lưu trữ mã băm của các nút con của chúng.
        - **Mã băm gốc (Root hash)** đại diện cho trạng thái kết hợp của tất cả dữ liệu trong cây.

    2. **Xây dựng Cây Merkle:**
        - **Bước 1:** Chia không gian khóa thành các thùng (buckets).
            
            <img src="./images/key-bucket.png"   alt="Key Bucket" width="500">

        - **Bước 2:** Băm mỗi khóa trong một thùng bằng cách sử dụng băm đồng nhất.

            <img src="./images/hash-key-bucket.png"   alt="Hash Key Bucket" width="500">

        - **Bước 3:** Tạo một mã băm duy nhất cho mỗi thùng.
        
            <img src="./images/hash-bucket.png"   alt="Hash Bucket" width="500">

        - **Bước 4:** Kết hợp các mã băm của các thùng để tính toán các mã băm ở cấp cao hơn, kết thúc tại mã băm gốc.

            <img src="./images/merkel-tree.png"   alt="Merkel Tree" width="500">

    3. **Đồng bộ hóa:**
        - Để đồng bộ hóa hai bản sao:
            - So sánh các mã băm gốc của chúng.
            - Nếu các mã băm gốc khớp nhau, các bản sao là nhất quán.
            - Nếu các mã băm gốc khác nhau, so sánh mã băm của các nút con một cách đệ quy để xác định các thùng không nhất quán.
        - Chỉ dữ liệu không nhất quán mới được đồng bộ hóa.

- Ưu điểm:
    - **Hiệu quả:** Chỉ dữ liệu không nhất quán mới được đồng bộ hóa, giảm truyền tải dữ liệu.
    - **Khả năng mở rộng:** Hiệu quả cho các tập dữ liệu lớn với chi phí đồng bộ hóa tối thiểu.
    - **Độ tin cậy:** Đảm bảo tính nhất quán dữ liệu giữa các bản sao.

### 6. Xử lý Sự cố Trung tâm Dữ liệu
- Sao chép dữ liệu trên nhiều trung tâm dữ liệu để đảm bảo tính sẵn sàng trong các lần mất điện hoặc sự cố.

---

## Luồng Ghi và Đọc
### 1. Luồng Ghi (Dựa trên kiến trúc Cassandra)

<div style="margin-left:3rem">
    <img src="./images/write-path.png"   alt="Hash Bucket" width="500">
</div>

- Lưu lại bản ghi trong một **nhật ký cam kết (commit log)**.
- Lưu dữ liệu vào một **bộ nhớ đệm (memory cache)**.
- Đẩy dữ liệu vào **SSTable** (Sorted String Table) trên đĩa khi bộ nhớ đệm đầy.

### 2. Luồng Đọc
<div style="margin-left:3rem">
    <img src="./images/read-path.png"   alt="Hash Bucket" width="500">
    <img src="./images/read-path-without-cache.png"   alt="Hash Bucket" width="500">
</div>

- Kiểm tra **bộ nhớ đệm (memory cache)** để tìm dữ liệu.
- Nếu không có, sử dụng một **Bộ lọc Bloom (Bloom Filter)** để định vị dữ liệu trong các SSTable.
- Truy xuất và trả về dữ liệu.

---

## Kiến trúc Cuối cùng

<p align="center">
<img src="./images/final-architecture.png"   alt="Hash Bucket" width="500">
</p>

- Các máy khách giao tiếp với kho lưu trữ key-value thông qua các API đơn giản: get(key) và put(key, value).
- Một điều phối viên (coordinator) là một nút hoạt động như một proxy giữa máy khách và kho lưu trữ key-value.
- Các nút được phân phối trên một vòng tròn bằng cách sử dụng băm nhất quán.
- Hệ thống hoàn toàn phi tập trung nên việc thêm và di chuyển các nút có thể diễn ra tự động.
- Dữ liệu được sao chép tại nhiều nút.
- Không có điểm lỗi đơn lẻ (single point of failure) vì mỗi nút đều có cùng một tập hợp các trách nhiệm.
