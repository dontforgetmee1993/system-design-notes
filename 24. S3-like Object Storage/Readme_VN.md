# Chương 24: Lưu trữ Đối tượng kiểu S3

## Giới thiệu

Trong chương này, chúng ta sẽ thiết kế một dịch vụ **lưu trữ đối tượng (object storage)**, tương tự như **Amazon S3**.

Các hệ thống lưu trữ được chia thành ba loại chính:
- **Lưu trữ khối (Block storage)**
- **Lưu trữ tệp (File storage)**
- **Lưu trữ đối tượng (Object storage)**

**Lưu trữ khối** là các thiết bị xuất hiện từ những năm 1960. HDD và SSD là những ví dụ như vậy.
Các thiết bị này thường được gắn vật lý vào máy chủ, mặc dù chúng cũng có thể được gắn qua mạng thông qua các giao thức mạng tốc độ cao.
Máy chủ có thể định dạng các khối thô và sử dụng chúng như một hệ thống tệp hoặc nó có thể trao quyền kiểm soát trực tiếp cho các máy chủ.

**Lưu trữ tệp** được xây dựng trên nền tảng lưu trữ khối. Nó cung cấp một mức độ trừu tượng cao hơn, giúp việc quản lý các thư mục và tệp trở nên dễ dàng hơn.

**Lưu trữ đối tượng** đánh đổi hiệu suất để lấy độ bền cao, quy mô lớn và chi phí thấp.
Nó nhắm mục tiêu vào dữ liệu "lạnh" và chủ yếu được sử dụng để lưu trữ lưu trữ và sao lưu.
Không có cấu trúc thư mục phân cấp, tất cả dữ liệu được lưu trữ dưới dạng các đối tượng trong một cấu trúc phẳng.
Nó tương đối chậm so với các loại lưu trữ khác. Hầu hết các nhà cung cấp đám mây đều có dịch vụ lưu trữ đối tượng - Amazon S3, Google GCS, v.v.

<div style="margin-left:3rem">
    <img src="./images/storage-comparison.png" alt="storage-comparison" width="500" />
</div>

| | Lưu trữ Khối | Lưu trữ Tệp | Lưu trữ Đối tượng |
|-----------------|----------------------------------|-----------------------------------------|--------------------------------|
| Nội dung có thể thay đổi | Y | Y | N (có quản lý phiên bản đối tượng) |
| Chi phí | Cao | Trung bình đến cao | Thấp |
| Hiệu suất | Trung bình đến cao, rất cao | Trung bình đến cao | Thấp đến trung bình |
| Tính nhất quán | Nhất quán mạnh | Nhất quán mạnh | Nhất quán mạnh [5] |
| Truy cập dữ liệu | SAS/iSCSI/FC | Truy cập tệp tiêu chuẩn, CIFS/SMB, và NFS | RESTful API |
| Khả năng mở rộng | Khả năng mở rộng trung bình | Khả năng mở rộng cao | Khả năng mở rộng cực lớn |
| Phù hợp cho | Máy ảo (VM), cơ sở dữ liệu | Truy cập hệ thống tệp đa mục đích | Dữ liệu nhị phân, dữ liệu không cấu trúc |

Một số thuật ngữ liên quan đến lưu trữ đối tượng:
- **Bucket** - vùng chứa logic cho các đối tượng. Tên là duy nhất trên toàn cầu.
- **Object** - Một mẩu dữ liệu riêng lẻ, được lưu trữ trong một bucket. Chứa dữ liệu đối tượng và siêu dữ liệu (metadata).
- **Versioning (Quản lý phiên bản)** - Một tính năng giữ nhiều biến thể của một đối tượng trong cùng một bucket.
- **Uniform Resource Identifier (URI)** - mỗi tài nguyên được định danh duy nhất bởi một URI.
- **Service-level Agreement (SLA)** - hợp đồng giữa nhà cung cấp dịch vụ và khách hàng.

SLA của lớp lưu trữ Amazon S3 Standard-Infrequent Access:
- Độ bền (Durability) đạt 99.999999999% trên nhiều Availability Zones.
- Dữ liệu có khả năng phục hồi trong trường hợp toàn bộ Availability Zone bị phá hủy.
- Được thiết kế cho tính sẵn sàng (availability) đạt 99.9%.

---

## Bước 1: Hiểu vấn đề và Thiết lập Phạm vi Thiết kế

- C: Những tính năng nào nên được bao gồm?
- I: Tạo bucket, Tải lên/Tải xuống đối tượng, quản lý phiên bản, Liệt kê các đối tượng trong một bucket.
- C: Kích thước dữ liệu điển hình là bao nhiêu?
- I: Chúng ta cần lưu trữ hiệu quả cả các đối tượng lớn và các đối tượng nhỏ.
- C: Chúng ta lưu trữ bao nhiêu dữ liệu trong một năm?
- I: 100 petabytes.
- C: Chúng ta có thể giả định độ bền dữ liệu là 6 con số 9 (99.9999%) và tính sẵn sàng của dịch vụ là 4 con số 9 (99.99%) không?
- I: Có, điều đó nghe có vẻ hợp lý.

### **Yêu cầu phi chức năng**

- **100 PB dữ liệu**.
- **Độ bền dữ liệu 6 con số 9**.
- **Tính sẵn sàng của dịch vụ 4 con số 9**.
- Hiệu quả lưu trữ. Giảm chi phí lưu trữ trong khi vẫn duy trì độ tin cậy và hiệu suất cao.

### **Ước tính sơ bộ**

Lưu trữ đối tượng có khả năng gặp nút thắt cổ chai ở dung lượng đĩa hoặc số lượng IO mỗi giây (IOPS).

Giả định:
- Chúng ta có 20% đối tượng nhỏ (nhỏ hơn 1MB), 60% đối tượng kích thước trung bình (1-64MB) và 20% đối tượng lớn (lớn hơn 64MB).
- Một ổ cứng (SATA, 7200rpm) có khả năng thực hiện 100-150 lần tìm kiếm ngẫu nhiên mỗi giây (100-150 IOPS).

Dựa trên các giả định, chúng ta có thể ước tính tổng số đối tượng mà hệ thống có thể lưu trữ.
- Hãy sử dụng kích thước trung vị cho mỗi loại đối tượng để đơn giản hóa việc tính toán - 0.5MB cho đối tượng nhỏ, 32MB cho đối tượng trung bình, 200MB cho đối tượng lớn.
- Với 100PB lưu trữ (10^11 MB) và 40% dung lượng lưu trữ được sử dụng sẽ dẫn đến 0.68 tỷ đối tượng.
- Nếu chúng ta giả định siêu dữ liệu là 1KB, thì chúng ta cần 0.68TB không gian để lưu trữ thông tin siêu dữ liệu.

---

## Bước 2: Đề xuất Thiết kế Mức cao và Đạt được sự Thống nhất

Hãy khám phá một số tính chất thú vị của lưu trữ đối tượng trước khi đi sâu vào thiết kế:
- **Tính bất biến của đối tượng (Object immutability)** - các đối tượng trong lưu trữ đối tượng là bất biến (không giống như các hệ thống lưu trữ khác). Chúng ta có thể xóa hoặc thay thế chúng, nhưng không cập nhật.
- **Kho lưu trữ Key-value** - một URI của đối tượng chính là khóa (key) của nó và chúng ta có thể lấy nội dung của nó bằng cách thực hiện một cuộc gọi HTTP.
- **Ghi một lần, đọc nhiều lần (Write once, read many times)** - mô hình truy cập dữ liệu là ghi một lần và đọc nhiều lần. Theo một số nghiên cứu của Linkedin, 95% các hoạt động là đọc.
- Hỗ trợ cả các đối tượng nhỏ và lớn.

Triết lý thiết kế của lưu trữ đối tượng tương tự như UNIX - khi chúng ta lưu một tệp, nó tạo ra tên tệp trong một cấu trúc dữ liệu gọi là inode và dữ liệu tệp được lưu trữ ở các vị trí khác nhau trên đĩa.
Inode chứa một danh sách các con trỏ khối tệp, trỏ đến các vị trí khác nhau trên đĩa.

Khi truy cập một tệp, trước tiên chúng ta lấy siêu dữ liệu của nó từ inode, trước khi lấy nội dung tệp.

Lưu trữ đối tượng hoạt động tương tự - kho siêu dữ liệu được sử dụng cho thông tin tệp, nhưng nội dung được lưu trữ trên đĩa:

<div style="margin-left:3rem">
    <img src="./images/object-store-vs-unix.png" alt="object-store-vs-unix" width="500" />
</div>

Bằng cách tách biệt siêu dữ liệu khỏi nội dung tệp, chúng ta có thể mở rộng các kho lưu trữ khác nhau một cách độc lập:

<div style="margin-left:3rem">
    <img src="./images/bucket-and-object.png" alt="bucket-and-object" width="500" />
</div>

### **Thiết kế mức cao**

<div style="margin-left:3rem">
    <img src="./images/high-level-design.png" alt="high-level-design" width="500" />
</div>

- **Bộ cân bằng tải (Load balancer)** - phân phối các yêu cầu API qua các bản sao dịch vụ.
- **Dịch vụ API (API service)** - Máy chủ không trạng thái (Stateless), điều phối các cuộc gọi đến kho siêu dữ liệu và kho đối tượng, cũng như dịch vụ IAM.
- **Quản lý danh tính và quyền truy cập (IAM)** - nơi tập trung để xác thực (auth), phân quyền (authz) và kiểm soát truy cập.
- **Kho dữ liệu (Data store)** - lưu trữ và truy xuất dữ liệu thực tế. Các hoạt động dựa trên ID đối tượng (UUID).
- **Kho siêu dữ liệu (Metadata store)** - lưu trữ siêu dữ liệu của đối tượng.

### **Tải lên một đối tượng**

<div style="margin-left:3rem">
    <img src="./images/uploading-object.png" alt="uploading-object" width="500" />
</div>

- Tạo một bucket tên là "bucket-to-share" thông qua yêu cầu HTTP PUT.
- Dịch vụ API gọi IAM để đảm bảo người dùng được ủy quyền và có quyền ghi.
- Dịch vụ API gọi kho siêu dữ liệu để tạo một mục nhập bucket. Sau khi tạo xong, phản hồi thành công được trả về.
- Sau khi bucket được tạo, HTTP PUT được gửi để tạo một đối tượng tên là "script.txt".
- Dịch vụ API xác minh danh tính người dùng và đảm bảo người dùng có quyền ghi.
- Sau khi xác thực vượt qua, nội dung đối tượng được gửi qua HTTP PUT đến kho dữ liệu. Kho dữ liệu lưu trữ nó và trả về một UUID.
- Dịch vụ API gọi kho siêu dữ liệu để tạo một mục nhập mới với object_id, bucket_id và bucket_name, cùng các siêu dữ liệu khác.

Ví dụ yêu cầu tải lên đối tượng:

```
PUT /bucket-to-share/script.txt HTTP/1.1
Host: foo.s3example.org
Date: Sun, 12 Sept 2021 17:51:00 GMT
Authorization: authorization string
Content-Type: text/plain
Content-Length: 4567
x-amz-meta-author: Alex

[4567 bytes of object data]
```

### **Tải xuống một đối tượng**

Các bucket không có phân cấp thư mục, nhưng chúng ta có thể tạo một phân cấp logic bằng cách nối tên bucket và tên đối tượng để mô phỏng cấu trúc thư mục.

Ví dụ yêu cầu GET để lấy một đối tượng:

```
GET /bucket-to-share/script.txt HTTP/1.1
Host: foo.s3example.org
Date: Sun, 12 Sept 2021 18:30:01 GMT
Authorization: authorization string
```

<div style="margin-left:3rem">
    <img src="./images/download-object.png" alt="download-object" width="500" />
</div>

- Client gửi một yêu cầu HTTP GET đến bộ cân bằng tải, ví dụ `GET /bucket-to-share/script.txt`.
- Dịch vụ API truy vấn IAM để xác minh người dùng có quyền hợp lệ để đọc bucket.
- Sau khi được xác thực, UUID của đối tượng được lấy từ kho siêu dữ liệu.
- Nội dung đối tượng được lấy từ kho dữ liệu dựa trên UUID và trả về cho client.

---

## Bước 3: Thiết kế Chi tiết

### **Kho lưu trữ dữ liệu**

Dưới đây là cách dịch vụ API tương tác với kho dữ liệu:

<div style="margin-left:3rem">
    <img src="./images/data-store-interactions.png" alt="data-store-interactions" width="500" />
</div>

Các thành phần chính của kho dữ liệu:

<div style="margin-left:3rem">
    <img src="./images/data-store-main-components.png" alt="data-store-main-components" width="500" />
</div>

Dịch vụ điều phối dữ liệu (data routing service) cung cấp một RESTful hoặc gRPC API để truy cập cụm node dữ liệu.
Nó là một dịch vụ không trạng thái, mở rộng bằng cách thêm nhiều máy chủ hơn.

Trách nhiệm chính của nó là:
- truy vấn dịch vụ vị trí (placement service) để tìm node dữ liệu tốt nhất để lưu trữ dữ liệu.
- đọc dữ liệu từ các node dữ liệu và trả về cho dịch vụ API.
- Ghi dữ liệu vào các node dữ liệu.

Dịch vụ vị trí (placement service) xác định các node dữ liệu nào nên lưu trữ một đối tượng.
Nó duy trì một bản đồ cụm ảo (virtual cluster map), xác định cấu trúc liên kết vật lý của một cụm.

<div style="margin-left:3rem">
    <img src="./images/virtual-cluster-map.png" alt="virtual-cluster-map" width="500" />
</div>

Dịch vụ cũng gửi nhịp tim (heartbeats) đến tất cả các node dữ liệu để xác định xem chúng có nên bị loại bỏ khỏi cụm ảo hay không.

Vì đây là một dịch vụ quan trọng, khuyến nghị nên duy trì một cụm gồm 5 hoặc 7 bản sao, được đồng bộ hóa thông qua các thuật toán đồng thuận Paxos hoặc Raft.
Ví dụ: một cụm 7 node có thể chịu được lỗi 3 node.

Các node dữ liệu lưu trữ dữ liệu đối tượng thực tế.
Độ tin cậy và độ bền được đảm bảo bằng cách sao chép dữ liệu sang nhiều node dữ liệu.

Mỗi node dữ liệu có một tiến trình daemon đang chạy, gửi nhịp tim đến dịch vụ vị trí.

Nhịp tim bao gồm:
- Node dữ liệu quản lý bao nhiêu ổ đĩa (HDD hoặc SSD)?
- Bao nhiêu dữ liệu được lưu trữ trên mỗi ổ đĩa?

#### Luồng lưu trữ dữ liệu

<div style="margin-left:3rem">
    <img src="./images/data-persistence-flow.png" alt="data-persistence-flow" width="500" />
</div>

- Dịch vụ API chuyển tiếp dữ liệu đối tượng đến kho dữ liệu.
- Dịch vụ điều phối dữ liệu gửi dữ liệu đến node dữ liệu chính (primary data node).
- Node dữ liệu chính lưu dữ liệu cục bộ và sao chép nó sang hai node dữ liệu phụ (secondary data nodes). Phản hồi được gửi sau khi sao chép thành công.
- UUID của đối tượng được trả về cho dịch vụ API.

Lưu ý:
- Với một UUID đối tượng nhất định, nhóm sao chép của nó được chọn một cách xác định bằng cách sử dụng băm nhất quán (consistent hashing).
- Ở bước 4, node dữ liệu chính sao chép dữ liệu đối tượng trước khi trả về phản hồi. Điều này ưu tiên tính nhất quán mạnh hơn là độ trễ cao hơn.

<div style="margin-left:3rem">
    <img src="./images/consistency-vs-latency.png" alt="consistency-vs-latency" width="500" />
</div>

#### Cách tổ chức dữ liệu

Một cách tiếp cận đơn giản để quản lý dữ liệu là lưu trữ mỗi đối tượng trong một tệp riêng biệt.

Điều này có thể hoạt động, nhưng không hiệu quả với nhiều tệp nhỏ trong một hệ thống tệp:
- Các khối dữ liệu trên HDD bị lãng phí, vì mọi tệp đều sử dụng toàn bộ kích thước khối. Kích thước khối điển hình là 4KB.
- Nhiều tệp có nghĩa là nhiều inode. Hệ điều hành không xử lý tốt với quá nhiều inode và cũng có giới hạn inode tối đa.

Các vấn đề này có thể được giải quyết bằng cách hợp nhất nhiều tệp nhỏ thành những tệp lớn hơn thông qua một log ghi trước (write-ahead log - WAL). Khi tệp đạt đến dung lượng tối đa (thường là vài GB), một tệp mới sẽ được tạo:

<div style="margin-left:3rem">
    <img src="./images/wal-optimization.png" alt="wal-optimization" width="500" />
</div>

Nhược điểm của cách tiếp cận này là quyền truy cập ghi vào tệp cần được tuần tự hóa. Nhiều lõi truy cập cùng một tệp phải đợi nhau.
Để khắc phục điều này, chúng ta có thể giới hạn các tệp cho các lõi cụ thể để tránh tranh chấp khóa (lock contention).

#### Tra cứu đối tượng

Để hỗ trợ lưu trữ nhiều đối tượng trong cùng một tệp, chúng ta cần duy trì một bảng để thông báo cho node dữ liệu biết:
- `object_id`
- `filename` nơi đối tượng được lưu trữ.
- `file_offset` nơi đối tượng bắt đầu.
- `object_size`

Chúng ta có thể triển khai bảng này trong một DB dạng tệp như RocksDB hoặc một cơ sở dữ liệu quan hệ truyền thống.
Vì mô hình truy cập là ghi ít + đọc nhiều, một cơ sở dữ liệu quan hệ hoạt động tốt hơn.

Chúng ta nên triển khai nó như thế nào?
Chúng ta có thể triển khai DB và mở rộng nó riêng biệt trong một cụm, được truy cập bởi tất cả các node dữ liệu.

Nhược điểm:
- chúng ta cần mở rộng cụm một cách mạnh mẽ để phục vụ tất cả các yêu cầu.
- có thêm độ trễ mạng giữa node dữ liệu và cụm DB.

Một lựa chọn khác là tận dụng thực tế rằng các node dữ liệu chỉ quan tâm đến dữ liệu liên quan đến chúng,
vì vậy chúng ta có thể triển khai DB quan hệ ngay bên trong chính node dữ liệu đó.

SQLite là một lựa chọn tốt vì nó là một cơ sở dữ liệu quan hệ dạng tệp nhẹ.

#### Luồng lưu trữ dữ liệu đã cập nhật

<div style="margin-left:3rem">
    <img src="./images/updated-data-persistence-flow.png" alt="updated-data-persistence-flow" width="500" />
</div>

- Dịch vụ API gửi một yêu cầu để lưu một đối tượng mới.
- Dịch vụ node dữ liệu nối thêm đối tượng mới vào cuối một tệp, tên là "/data/c".
- Một bản ghi mới cho đối tượng được chèn vào bảng ánh xạ đối tượng (object mapping table).

#### Độ bền dữ liệu (Durability)

Độ bền dữ liệu là một yêu cầu quan trọng trong thiết kế của chúng ta. Để đạt được 6 con số 9 về độ bền, mọi trường hợp lỗi cần được kiểm tra kỹ lưỡng.

Vấn đề đầu tiên cần giải quyết là lỗi phần cứng. Chúng ta có thể đạt được điều đó bằng cách sao chép các node dữ liệu để giảm thiểu xác suất lỗi.
Nhưng ngoài ra, chúng ta cũng nên sao chép qua các miền lỗi khác nhau (cross-rack, cross-dc, các mạng riêng biệt, v.v.).
Một sự kiện nghiêm trọng có thể gây ra nhiều lỗi phần cứng trong cùng một miền lỗi:

<div style="margin-left:3rem">
    <img src="./images/failure-domain-isolation.png" alt="failure-domain-isolation" width="500" />
</div>

Giả sử tỷ lệ lỗi hàng năm của một HDD điển hình là 0.81%, việc tạo ba bản sao sẽ mang lại cho chúng ta 6 con số 9 về độ bền.

Sao chép các node dữ liệu như vậy mang lại cho chúng ta độ bền mong muốn, nhưng chúng ta cũng có thể tận dụng mã hóa xóa (erasure coding) để giảm chi phí lưu trữ.

Mã hóa xóa cho phép chúng ta sử dụng các bit chẵn lẻ (parity bits), cho phép chúng ta tái cấu trúc các bit bị mất trong trường hợp xảy ra lỗi:

<div style="margin-left:3rem">
    <img src="./images/erasure-coding.png" alt="erasure-coding" width="500" />
</div>

Hãy tưởng tượng những bit đó là các node dữ liệu. Nếu hai trong số chúng bị hỏng, chúng có thể được phục hồi bằng bốn node còn lại.

Có các sơ đồ mã hóa xóa khác nhau. Trong trường hợp của chúng ta, chúng ta có thể sử dụng mã hóa xóa 8+4, chia nhỏ qua các miền lỗi khác nhau để tối đa hóa độ tin cậy:

<div style="margin-left:3rem">
    <img src="./images/erasure-coding-across-failure-domains.png" alt="erasure-coding-across-failure-domains" width="500" />
</div>

Mã hóa xóa cho phép chúng ta đạt được chi phí lưu trữ thấp hơn nhiều (cải thiện 50%) với cái giá là tốc độ truy cập do dịch vụ điều phối dữ liệu phải thu thập dữ liệu từ nhiều vị trí:

<div style="margin-left:3rem">
    <img src="./images/erasure-coding-vs-replication.png" alt="erasure-coding-vs-replication" width="500" />
</div>

Các lưu ý khác:
- Sao chép yêu cầu chi phí lưu trữ 200% (trong trường hợp có 3 bản sao) so với 50% thông qua mã hóa xóa.
- Mã hóa xóa [mang lại cho chúng ta 11 con số 9 về độ bền](https://github.com/Backblaze/erasure-coding-durability) so với 6 con số 9 qua sao chép.
- Mã hóa xóa đòi hỏi nhiều tính toán hơn để tính toán và lưu trữ các bit chẵn lẻ.

Tóm lại, sao chép hữu ích hơn cho các ứng dụng nhạy cảm với độ trễ, trong khi mã hóa xóa hấp dẫn về hiệu quả chi phí lưu trữ và độ bền.
Mã hóa xóa cũng khó triển khai hơn nhiều.

#### Xác minh tính chính xác

Nếu một đĩa bị hỏng hoàn toàn, thì lỗi rất dễ phát hiện. Điều này khó khăn hơn trong trường hợp một phần bộ nhớ của đĩa bị hỏng.

Để phát hiện điều này, chúng ta có thể sử dụng mã kiểm tra (checksums) - một mã băm của nội dung tệp, có thể được sử dụng để xác minh tính toàn vẹn của tệp.

Trong trường hợp của chúng ta, chúng ta sẽ lưu trữ checksum cho mỗi tệp và mỗi đối tượng:

<div style="margin-left:3rem">
    <img src="./images/checksums-for-correctness.png" alt="checksums-for-correctness" width="500" />
</div>

Trong trường hợp mã hóa xóa (8+4), chúng ta sẽ cần lấy riêng từng phần trong số 8 phần dữ liệu và xác minh checksum của từng phần đó.

### **Mô hình dữ liệu Siêu dữ liệu (Metadata)**

Schemas bảng:

<div style="margin-left:3rem">
    <img src="./images/metadata-data-model.png" alt="metadata-data-model" width="500" />
</div>

Các truy vấn chúng ta cần hỗ trợ:
- Tìm một ID đối tượng theo tên.
- Chèn/xóa đối tượng dựa trên tên.
- Liệt kê các đối tượng trong một bucket có cùng tiền tố.

Thường có giới hạn về số lượng bucket mà một người dùng có thể tạo, do đó, kích thước của bảng bucket nhỏ và có thể chứa gọn trong một máy chủ DB duy nhất.
Nhưng chúng ta vẫn cần mở rộng máy chủ cho thông lượng đọc.

Tuy nhiên, bảng đối tượng có lẽ sẽ không vừa với một máy chủ cơ sở dữ liệu duy nhất. Do đó, chúng ta có thể mở rộng bảng thông qua phân mảnh (sharding):
- Phân mảnh theo `bucket_id` sẽ dẫn đến vấn đề hotspot (điểm nóng) vì một bucket có thể có hàng tỷ đối tượng.
- Phân mảnh theo `user_id` làm cho tải được phân bổ đều hơn, nhưng các truy vấn của chúng ta sẽ chậm.
- Chúng ta chọn phân mảnh theo `hash(bucket_name, object_name)` vì hầu hết các truy vấn đều dựa trên tên đối tượng/bucket.

Mặc dù với sơ đồ phân mảnh này, việc liệt kê các đối tượng trong một bucket sẽ vẫn chậm.

### **Liệt kê các đối tượng trong một bucket**

Trong một cơ sở dữ liệu duy nhất, việc liệt kê một đối tượng dựa trên tiền tố của nó (trông giống như một thư mục) hoạt động như sau:

```
SELECT * FROM object WHERE bucket_id = "123" AND object_name LIKE `abc/%`
```

Điều này là một thách thức để thực hiện khi cơ sở dữ liệu được phân mảnh. Để đạt được điều đó, chúng ta có thể chạy truy vấn trên mọi mảnh (shard) và tổng hợp các kết quả trong bộ nhớ.
Tuy nhiên, điều này làm cho việc phân trang trở nên khó khăn, vì các mảnh khác nhau chứa kích thước kết quả khác nhau và chúng ta cần duy trì giới hạn/độ lệch (limit/offset) riêng cho từng mảnh.

Chúng ta có thể tận dụng thực tế là thông thường các kho lưu trữ đối tượng không được tối ưu hóa cho việc liệt kê đối tượng, vì vậy chúng ta có thể đánh đổi hiệu suất liệt kê.
Chúng ta cũng có thể tạo một bảng phi bình thường hóa để liệt kê các đối tượng, được phân mảnh theo ID bucket.
Điều đó sẽ làm cho truy vấn liệt kê của chúng ta đủ nhanh vì nó bị cô lập trong một phiên bản cơ sở dữ liệu duy nhất.

### **Quản lý phiên bản đối tượng (Object versioning)**

Quản lý phiên bản hoạt động bằng cách có thêm một cột `object_version` thuộc loại TIMEUUID, cho phép chúng ta sắp xếp các bản ghi dựa trên nó.

Mỗi phiên bản mới tạo ra một `object_id` mới:

<div style="margin-left:3rem">
    <img src="./images/object-versioning.png" alt="object-versioning" width="500" />
</div>

Xóa một đối tượng sẽ tạo ra một phiên bản mới với một `object_id` đặc biệt cho biết đối tượng đã bị xóa. Các truy vấn cho nó sẽ trả về 404:

<div style="margin-left:3rem">
    <img src="./images/deleting-versioned-object.png" alt="deleting-versioned-object" width="500" />
</div>

### **Tối ưu hóa tải lên các tệp lớn**

Tải lên các tệp lớn có thể được tối ưu hóa bằng cách sử dụng tải lên nhiều phần (multipart uploads) - chia một tệp lớn thành nhiều phần nhỏ, được tải lên độc lập:

<div style="margin-left:3rem">
    <img src="./images/multipart-upload.png" alt="multipart-upload" width="500" />
</div>

- Client gọi dịch vụ để bắt đầu tải lên nhiều phần.
- Kho dữ liệu trả về một ID tải lên (upload ID) định danh duy nhất cho việc tải lên đó.
- Client chia tệp lớn thành nhiều phần, tải lên độc lập bằng upload id.
- Khi một phần được tải lên, kho dữ liệu trả về một etag, là một mã băm md5, xác định phần tải lên đó.
- Sau khi tất cả các phần được tải lên, client gửi một yêu cầu hoàn tất tải lên nhiều phần, bao gồm upload_id, số thứ tự các phần và tất cả các etag.
- Kho dữ liệu lắp ráp lại đối tượng từ các phần của nó. Quá trình này có thể mất vài phút. Sau đó, phản hồi thành công được trả về cho client.

Các phần cũ không còn hữu ích có thể được xóa tại thời điểm này. Chúng ta có thể giới thiệu một bộ thu gom rác (garbage collector) để xử lý việc đó.

### **Thu gom rác (Garbage collection)**

Thu gom rác là quá trình thu hồi không gian lưu trữ không còn được sử dụng. Có một vài cách khiến dữ liệu trở thành rác:
- **Xóa đối tượng lười (lazy object deletion)** - đối tượng được đánh dấu là đã xóa mà không thực sự bị xóa ngay.
- **Dữ liệu mồ côi (orphan data)** - ví dụ: một lần tải lên bị thất bại giữa chừng và các phần cũ cần được xóa.
- **Dữ liệu bị hỏng (corrupted data)** - dữ liệu không vượt qua được xác minh checksum.

Bộ thu gom rác cũng chịu trách nhiệm thu hồi không gian không sử dụng trong các bản sao.
Với sao chép, dữ liệu được xóa khỏi cả bản sao chính và bản sao phụ. Với mã hóa xóa (8+4), dữ liệu được xóa khỏi tất cả 12 node.

Để tạo điều kiện cho việc xóa, chúng ta sẽ sử dụng một quy trình gọi là nén (compaction):
- Bộ thu gom rác sao chép các đối tượng không bị xóa từ "data/b" sang "data/d".
- Bảng `object_mapping` được cập nhật sau khi sao chép hoàn tất bằng một giao dịch cơ sở dữ liệu.
- Để tránh tạo ra quá nhiều tệp nhỏ, việc nén được thực hiện trên các tệp vượt quá một ngưỡng nhất định.

<div style="margin-left:3rem">
    <img src="./images/compaction.png" alt="compaction" width="500" />
</div>

---

## Bước 4: Tổng kết

Những điều chúng ta đã đề cập:
- Thiết kế một kho lưu trữ đối tượng kiểu S3.
- So sánh sự khác biệt giữa lưu trữ đối tượng, khối và tệp.
- Đề cập đến việc tải lên, tải xuống, liệt kê, quản lý phiên bản các đối tượng trong một bucket.
- Đi sâu vào thiết kế - kho dữ liệu và kho siêu dữ liệu, sao chép và mã hóa xóa, tải lên nhiều phần, phân mảnh (sharding).
