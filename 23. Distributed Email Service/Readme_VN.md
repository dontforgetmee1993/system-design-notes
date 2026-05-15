# Chương 23: Dịch vụ Email Phân tán

## Giới thiệu

Chúng ta sẽ thiết kế một **dịch vụ email phân tán**, tương tự như **Gmail** trong chương này.

Vào năm 2020, **Gmail** có 1,8 tỷ người dùng hoạt động, trong khi **Outlook** có 400 triệu người dùng trên toàn thế giới.

---

## Bước 1: Hiểu vấn đề và Thiết lập Phạm vi Thiết kế

- C: Có bao nhiêu người dùng sử dụng hệ thống?
- I: 1 tỷ người dùng.
- C: Tôi nghĩ các tính năng sau đây là quan trọng - xác thực, gửi/nhận email, lấy email, lọc email, tìm kiếm email, bảo vệ chống spam.
- I: Danh sách tốt đấy. Hiện tại đừng lo lắng về xác thực.
- C: Người dùng kết nối với máy chủ email như thế nào?
- I: Thông thường, các ứng dụng email kết nối qua SMTP, POP, IMAP, nhưng chúng ta sẽ sử dụng HTTP cho vấn đề này.
- C: Email có thể có tệp đính kèm không?
- I: Có.

### **Yêu cầu phi chức năng**

- **Độ tin cậy (Reliability)** - chúng ta không được làm mất dữ liệu.
- **Tính sẵn sàng (Availability)** - Chúng ta nên sử dụng bản sao (replication) để ngăn chặn các điểm lỗi đơn lẻ (single points of failure). Chúng ta cũng nên chấp nhận các lỗi hệ thống cục bộ.
- **Khả năng mở rộng (Scalability)** - Khi cơ sở người dùng tăng lên, hệ thống của chúng ta phải có khả năng xử lý chúng.
- **Tính linh hoạt và khả năng mở rộng (Flexibility and extensibility)** - hệ thống phải linh hoạt và dễ dàng mở rộng với các tính năng mới. Đó là một trong những lý do chúng ta chọn HTTP thay vì SMTP/các giao thức mail khác.

### **Ước tính sơ bộ**

- **1 tỷ người dùng**.
- Giả sử một người gửi 10 email mỗi ngày -> **100 nghìn email mỗi giây**.
- Giả sử một người nhận 40 email mỗi ngày và mỗi email trung bình có 50KB siêu dữ liệu (metadata) -> **730PB lưu trữ mỗi năm**.
- Giả sử 20% số email có tệp đính kèm và kích thước trung bình là 500KB -> **1,460PB mỗi năm**.

---

## Bước 2: Đề xuất Thiết kế Mức cao và Đạt được sự Thống nhất

### **Kiến thức cơ bản về Email**

Có nhiều giao thức khác nhau được sử dụng để gửi và nhận email:
- **SMTP** - giao thức tiêu chuẩn để gửi email từ máy chủ này sang máy chủ khác.
- **POP** - giao thức tiêu chuẩn để nhận và tải email từ máy chủ mail từ xa xuống client cục bộ. Sau khi lấy về, các email sẽ bị xóa khỏi máy chủ từ xa.
- **IMAP** - tương tự như POP, nó được sử dụng để nhận và tải email từ máy chủ từ xa, nhưng nó giữ lại các email trên phía máy chủ.
- **HTTPS** - về kỹ thuật không phải là một giao thức email, nhưng nó có thể được sử dụng cho các ứng dụng email dựa trên web.

Ngoài giao thức gửi thư, có một số bản ghi DNS chúng ta cần cấu hình cho máy chủ email của mình - bản ghi MX:

<div style="margin-left:3rem">
    <img src="./images/dns-lookup.png" alt="dns-lookup" width="500" />
</div>

Các tệp đính kèm email được gửi dưới dạng mã hóa base64 và thường có giới hạn kích thước là 25MB trên hầu hết các dịch vụ mail.
Điều này có thể cấu hình được và thay đổi tùy theo tài khoản cá nhân hoặc doanh nghiệp.

### **Máy chủ mail truyền thống**

Các máy chủ mail truyền thống hoạt động tốt khi có số lượng người dùng hạn chế, kết nối với một máy chủ duy nhất.

<div style="margin-left:3rem">
    <img src="./images/traditional-mail-server.png" alt="traditional-mail-server" width="500" />
</div>

- Alice đăng nhập vào email Outlook của mình và nhấn "gửi". Email được gửi đến máy chủ mail Outlook. Giao tiếp qua SMTP.
- Máy chủ Outlook truy vấn DNS để tìm bản ghi MX cho gmail.com và chuyển email đến máy chủ của họ. Giao tiếp qua SMTP.
- Bob lấy email từ máy chủ gmail của mình qua IMAP/POP.

Trong các máy chủ mail truyền thống, email được lưu trữ trên hệ thống tệp cục bộ. Mỗi email là một tệp riêng biệt.

<div style="margin-left:3rem">
    <img src="./images/local-dir-storage.png" alt="local-dir-storage" width="500" />
</div>

Khi quy mô tăng lên, I/O của đĩa trở thành nút thắt cổ chai. Ngoài ra, nó không thỏa mãn các yêu cầu về tính sẵn sàng cao và độ tin cậy của chúng ta.
Đĩa có thể bị hỏng và máy chủ có thể bị sập.

### **Máy chủ mail phân tán**

Máy chủ mail phân tán được thiết kế để hỗ trợ các trường hợp sử dụng hiện đại và giải quyết các vấn đề về khả năng mở rộng hiện đại.

Các máy chủ này vẫn có thể hỗ trợ IMAP/POP cho các ứng dụng email gốc và SMTP để trao đổi thư qua các máy chủ.

Nhưng đối với các ứng dụng mail dựa trên web phong phú, một RESTful API qua HTTP thường được sử dụng.

Các API ví dụ:
- `POST /v1/messages` - gửi một tin nhắn đến những người nhận trong các header To, Cc, Bcc.
- `GET /v1/folders` - trả về tất cả các thư mục của một tài khoản email.

Ví dụ phản hồi:

```
[{id: string        Định danh thư mục duy nhất.
  name: string      Tên của thư mục.
                    Theo RFC6154 [9], các thư mục mặc định có thể là một trong
                    các thư mục sau: All, Archive, Drafts, Flagged, Junk, Sent,
                    và Trash.
  user_id: string   Tham chiếu đến chủ sở hữu tài khoản.
}]
```

- `GET /v1/folders/{:folder_id}/messages` - trả về tất cả các tin nhắn trong một thư mục kèm theo phân trang.
- `GET /v1/messages/{:message_id}` - lấy tất cả thông tin về một tin nhắn cụ thể.

Ví dụ phản hồi:

```
{
  user_id: string                      // Tham chiếu đến chủ sở hữu tài khoản.
  from: {name: string, email: string}  // Cặp <tên, email> của người gửi.
  to: [{name: string, email: string}]  // Một danh sách các cặp <tên, email> người nhận.
  subject: string                      // Tiêu đề của email.
  body: string                         // Nội dung tin nhắn.
  is_read: boolean                     // Cho biết tin nhắn đã đọc hay chưa.
}
```

Dưới đây là thiết kế mức cao của máy chủ mail phân tán:

<div style="margin-left:3rem">
    <img src="./images/high-level-architecture.png" alt="high-level-architecture" width="500" />
</div>

- **Webmail** - người dùng sử dụng trình duyệt web để gửi/nhận email.
- **Máy chủ web (Web servers)** - các dịch vụ yêu cầu/phản hồi hướng ra công chúng được sử dụng để quản lý đăng nhập, đăng ký, hồ sơ người dùng, v.v.
- **Máy chủ thời gian thực (Real-time servers)** - Được sử dụng để đẩy các cập nhật email mới cho client trong thời gian thực. Chúng ta sử dụng websockets cho giao tiếp thời gian thực nhưng dự phòng sang long-polling cho các trình duyệt cũ không hỗ trợ chúng.
- **Cơ sở dữ liệu siêu dữ liệu (Metadata db)** - lưu trữ siêu dữ liệu email như tiêu đề, nội dung, người gửi, người nhận, v.v.
- **Kho lưu trữ tệp đính kèm (Attachment store)** - Kho lưu trữ đối tượng (ví dụ Amazon S3), phù hợp để lưu trữ các tệp lớn.
- **Bộ nhớ đệm phân tán (Distributed cache)** - Chúng ta có thể lưu các email gần đây trong Redis để cải thiện trải nghiệm người dùng.
- **Kho lưu trữ tìm kiếm (Search store)** - kho lưu trữ tài liệu phân tán, được sử dụng để hỗ trợ tìm kiếm toàn văn (full-text search).

Dưới đây là luồng gửi email trông như thế nào:

<div style="margin-left:3rem">
    <img src="./images/email-sending-flow.png" alt="email-sending-flow" width="500" />
</div>

- Người dùng viết một email và nhấn "gửi". Email được gửi đến bộ cân bằng tải.
- Bộ cân bằng tải giới hạn tốc độ gửi thư quá mức và điều hướng đến một trong các máy chủ web.
- Máy chủ web thực hiện xác thực email cơ bản (ví dụ kích thước email) và xử lý nhanh nếu tên miền giống với người gửi. Nhưng cần thực hiện kiểm tra spam trước.
- Nếu xác thực cơ bản vượt qua, email được gửi đến hàng đợi tin nhắn (tệp đính kèm được tham chiếu từ kho lưu trữ đối tượng).
- Nếu xác thực cơ bản thất bại, email được gửi đến hàng đợi lỗi.
- Các worker SMTP gửi thư (outgoing workers) lấy tin nhắn từ hàng đợi gửi, thực hiện kiểm tra spam/virus và điều hướng đến máy chủ mail đích.
- Email được lưu trữ trong thư mục "Sent Emails" (Thư đã gửi).

Chúng ta cũng cần theo dõi kích thước của hàng đợi tin nhắn gửi đi. Việc hàng đợi quá lớn có thể cho thấy một vấn đề:
- Máy chủ mail của người nhận không khả dụng. Chúng ta có thể thử gửi lại email sau đó bằng cách sử dụng thuật toán exponential backoff.
- Không đủ consumer để xử lý tải, chúng ta có thể phải mở rộng quy mô consumer.

Dưới đây là luồng nhận email:

<div style="margin-left:3rem">
    <img src="./images/email-receiving-flkow.png" alt="email-receiving-flow" width="500" />
</div>

- Các email đến tại bộ cân bằng tải SMTP. Các thư được phân phối đến các máy chủ SMTP, nơi thực hiện chính sách chấp nhận thư (ví dụ các email không hợp lệ sẽ bị loại bỏ trực tiếp).
- Nếu tệp đính kèm của email quá lớn, chúng ta có thể đưa nó vào kho lưu trữ đối tượng (s3).
- Các worker xử lý thư thực hiện các kiểm tra sơ bộ, sau đó thư được chuyển tiếp đến kho lưu trữ, bộ nhớ đệm, kho lưu trữ đối tượng và các máy chủ thời gian thực.
- Người dùng ngoại tuyến sẽ nhận được email mới của họ khi họ trực tuyến trở lại thông qua HTTP API.

---

## Bước 3: Thiết kế Chi tiết

Bây giờ hãy đi sâu vào một số thành phần.

### **Cơ sở dữ liệu Siêu dữ liệu (Metadata)**

Dưới đây là một số đặc điểm của siêu dữ liệu email:
- Các header thường nhỏ và được truy cập thường xuyên.
- Kích thước nội dung (Body) từ nhỏ đến lớn, nhưng thường được đọc một lần.
- Hầu hết các hoạt động mail được cô lập cho một người dùng duy nhất - ví dụ lấy email, đánh dấu là đã đọc, tìm kiếm.
- Tính mới của dữ liệu ảnh hưởng đến việc sử dụng dữ liệu. Người dùng thường chỉ đọc các email gần đây.
- Dữ liệu có yêu cầu độ tin cậy cao. Việc mất dữ liệu là không thể chấp nhận được.

Ở quy mô gmail/outlook, cơ sở dữ liệu thường được tùy chỉnh để giảm số lượng hoạt động nhập/xuất mỗi giây (IOPS).

Hãy xem xét các tùy chọn cơ sở dữ liệu chúng ta có:
- **Cơ sở dữ liệu quan hệ (Relational database)** - chúng ta có thể xây dựng các chỉ mục cho header và nội dung, nhưng các DB này thường được tối ưu hóa cho các khối dữ liệu nhỏ.
- **Kho lưu trữ đối tượng phân tán (Distributed object store)** - đây có thể là một lựa chọn tốt cho việc lưu trữ dự phòng, nhưng không thể hỗ trợ hiệu quả việc tìm kiếm/đánh dấu là đã đọc/v.v.
- **NoSQL** - Google BigTable được gmail sử dụng, nhưng nó không phải là mã nguồn mở.

Dựa trên phân tích trên, rất ít giải pháp hiện có dường như phù hợp hoàn hảo với nhu cầu của chúng ta.
Trong môi trường phỏng vấn, việc thiết kế một giải pháp cơ sở dữ liệu phân tán mới là không khả thi, nhưng quan trọng là phải đề cập đến các đặc điểm:
- Cột đơn có thể có kích thước vài MB.
- Tính nhất quán dữ liệu mạnh.
- Được thiết kế để giảm I/O đĩa.
- Tính sẵn sàng cao và khả năng chịu lỗi.
- Dễ dàng tạo các bản sao lưu tăng dần (incremental backups).

Để phân vùng dữ liệu, chúng ta có thể sử dụng `user_id` làm khóa phân vùng (partition key), để dữ liệu của một người dùng được lưu trữ trên một shard duy nhất.
Điều này ngăn cản chúng ta chia sẻ một email với nhiều người dùng, nhưng đây không phải là yêu cầu cho cuộc phỏng vấn này.

Hãy định nghĩa các bảng:
- Khóa chính bao gồm khóa phân vùng (phân phối dữ liệu) và khóa phân cụm (clustering key - sắp xếp dữ liệu).
- Các truy vấn chúng ta cần hỗ trợ - lấy tất cả các thư mục của một người dùng, hiển thị tất cả các email cho một thư mục, tạo/lấy/xóa một email, lấy email đã đọc/chưa đọc, lấy các luồng hội thoại (bonus).

Chú giải cho các bảng tiếp theo:

<div style="margin-left:3rem">
    <img src="./images/legend.png" alt="legend" width="500" />
</div>

Đây là bảng thư mục (folders):

<div style="margin-left:3rem">
    <img src="./images/folders-table.png" alt="folders-table" width="500" />
</div>

Bảng email:

<div style="margin-left:3rem">
    <img src="./images/emails-table.png" alt="emails-table" width="500" />
</div>

- `email_id` là timeuuid cho phép sắp xếp dựa trên mốc thời gian khi email được tạo.

Các tệp đính kèm được lưu trữ trong một bảng riêng biệt, được xác định bằng tên tệp:

<div style="margin-left:3rem">
    <img src="./images/attachments.png" alt="attachments" width="500" />
</div>

Hỗ trợ lấy các email đã đọc/chưa đọc là dễ dàng trong một cơ sở dữ liệu quan hệ truyền thống, nhưng không phải trong Cassandra, vì việc lọc trên khóa không phải là phân vùng/phân cụm bị cấm.
Một cách giải quyết là lấy tất cả email trong một thư mục và lọc trong bộ nhớ, nhưng điều đó không hoạt động tốt cho một ứng dụng đủ lớn.

Những gì chúng ta có thể làm là phi bình thường hóa (denormalize) bảng email thành các bảng email đã đọc/chưa đọc:

<div style="margin-left:3rem">
    <img src="./images/read-unread-emails.png" alt="read-unread-emails" width="500" />
</div>

Để hỗ trợ các luồng hội thoại (conversation threads), chúng ta có thể bao gồm một số header, mà các client mail giải mã và sử dụng để tái cấu trúc một luồng hội thoại:

```
{
  "headers" {
     "Message-Id": "<7BA04B2A-430C-4D12-8B57-862103C34501@gmail.com>",
     "In-Reply-To": "<CAEWTXuPfN=LzECjDJtgY9Vu03kgFvJnJUSHTt6TW@gmail.com>",
     "References": ["<7BA04B2A-430C-4D12-8B57-862103C34501@gmail.com>"]
  }
}
```

Cuối cùng, chúng ta sẽ đánh đổi tính sẵn sàng lấy tính nhất quán cho cơ sở dữ liệu phân tán của mình, vì đó là một yêu cầu bắt buộc cho vấn đề này.

Do đó, trong trường hợp xảy ra lỗi failover hoặc phân mảnh mạng (network partition), các hành động đồng bộ hóa/cập nhật sẽ tạm thời không khả dụng cho những người dùng bị ảnh hưởng.

### **Khả năng phân phối Email**

Thiết lập một máy chủ để gửi email thì dễ, nhưng đưa email vào hộp thư đến của người nhận thì khó, do các thuật toán chống spam.

Nếu chúng ta chỉ thiết lập một máy chủ mail mới và bắt đầu gửi mail qua nó, các email của chúng ta có thể sẽ kết thúc trong thư mục spam.

Dưới đây là những gì chúng ta có thể làm để ngăn chặn điều đó:
- **IP chuyên dụng (Dedicated IPs)** - sử dụng các IP chuyên dụng để gửi email, nếu không, các máy chủ nhận sẽ không tin tưởng bạn.
- **Phân loại email** - tránh gửi email marketing từ cùng một máy chủ để ngăn chặn các email quan trọng hơn bị phân loại là spam.
- **Làm ấm địa chỉ IP (Warm up your IP address)** từ từ để xây dựng uy tín tốt với các nhà cung cấp email lớn. Mất từ 2 đến 6 tuần để làm ấm một IP mới.
- **Cấm những kẻ phát tán spam (spammers)** nhanh chóng để không làm giảm uy tín của bạn.
- **Xử lý phản hồi (Feedback processing)** - thiết lập một vòng lặp phản hồi với các ISP để theo dõi tỷ lệ khiếu nại và cấm các tài khoản spam nhanh chóng.
- **Xác thực email** - sử dụng các kỹ thuật phổ biến để chống lừa đảo như Sender Policy Framework (SPF), DomainKeys Identified Mail (DKIM), v.v.

Bạn không cần phải nhớ tất cả những điều này. Chỉ cần biết rằng việc xây dựng một máy chủ mail tốt đòi hỏi rất nhiều kiến thức chuyên môn.

### **Tìm kiếm**

Tìm kiếm bao gồm việc thực hiện tìm kiếm toàn văn dựa trên nội dung email hoặc các truy vấn nâng cao hơn dựa trên các bộ lọc từ, đến, tiêu đề, chưa đọc, v.v.

Một đặc điểm của tìm kiếm email là nó mang tính cục bộ cho người dùng và nó có nhiều lượt ghi hơn lượt đọc, bởi vì chúng ta cần đánh chỉ mục lại trên mỗi hoạt động, nhưng người dùng hiếm khi sử dụng tab tìm kiếm.

Hãy so sánh tìm kiếm google với tìm kiếm email:

| | Phạm vi | Sắp xếp | Độ chính xác |
|---------------|----------------------|---------------------------------------|---------------------------------------------------|
| Google search | Toàn bộ internet | Sắp xếp theo mức độ liên quan | Việc đánh chỉ mục mất một thời gian, vì vậy kết quả không tức thì. |
| Email search | Hộp thư riêng của người dùng | Sắp xếp theo các thuộc tính ví dụ thời gian, ngày tháng, v.v. | Việc đánh chỉ mục phải nhanh chóng và kết quả phải chính xác. |

Để đạt được tính năng tìm kiếm này, một lựa chọn là sử dụng một cụm Elasticsearch. Chúng ta có thể sử dụng `user_id` làm khóa phân vùng để nhóm dữ liệu dưới cùng một node:

<div style="margin-left:3rem">
    <img src="./images/elasticsearch.png" alt="elasticsearch" width="500" />
</div>

Các hoạt động thay đổi (mutating operations) là không đồng bộ qua Kafka để tách biệt các dịch vụ khỏi luồng đánh chỉ mục lại.
Việc tìm kiếm dữ liệu thực tế diễn ra đồng bộ.

Elasticsearch là một trong những cơ sở dữ liệu công cụ tìm kiếm phổ biến nhất và hỗ trợ tìm kiếm toàn văn cho email rất tốt.

Ngoài ra, chúng ta có thể cố gắng phát triển giải pháp tìm kiếm tùy chỉnh của riêng mình để đáp ứng các yêu cầu cụ thể.

Thiết kế một hệ thống như vậy nằm ngoài phạm vi. Một trong những thách thức cốt lõi khi xây dựng nó là tối ưu hóa cho khối lượng công việc ghi nhiều.

Để đạt được điều đó, chúng ta có thể sử dụng Log-Structured Merge-Trees (LSM) để cấu trúc dữ liệu chỉ mục trên đĩa. Đường dẫn ghi được tối ưu hóa chỉ cho các lượt ghi tuần tự.
Kỹ thuật này được sử dụng trong Cassandra, BigTable và RocksDB.

Ý tưởng cốt lõi của nó là lưu trữ dữ liệu trong bộ nhớ cho đến khi đạt đến một ngưỡng xác định trước, sau đó nó được hợp nhất vào lớp tiếp theo (đĩa):

<div style="margin-left:3rem">
    <img src="./images/lsm-tree.png" alt="lsm-tree" width="500" />
</div>

Các sự đánh đổi chính giữa hai cách tiếp cận:
- Elasticsearch có thể mở rộng đến một mức độ nào đó, trong khi một công cụ tìm kiếm tùy chỉnh có thể được tinh chỉnh cho trường hợp sử dụng email, cho phép nó mở rộng hơn nữa.
- Elasticsearch là một dịch vụ riêng biệt mà chúng ta cần duy trì, bên cạnh kho siêu dữ liệu. Một giải pháp tùy chỉnh có thể chính là kho dữ liệu đó.
- Elasticsearch là một giải pháp có sẵn, trong khi công cụ tìm kiếm tùy chỉnh sẽ yêu cầu nỗ lực kỹ thuật đáng kể để xây dựng.

### **Khả năng mở rộng và tính sẵn sàng**

Vì các hoạt động của từng người dùng không va chạm với những người dùng khác, hầu hết các thành phần có thể được mở rộng độc lập.

Để đảm bảo tính sẵn sàng cao, chúng ta cũng có thể sử dụng thiết lập đa trung tâm dữ liệu (multi-DC) với cơ chế failover leader-follower trong trường hợp xảy ra sự cố:

<div style="margin-left:3rem">
    <img src="./images/multi-dc-example.png" alt="multi-dc-example" width="500" />
</div>

---

## Bước 4: Tổng kết

Các điểm bổ sung có thể thảo luận:
- **Khả năng chịu lỗi (Fault tolerance)** - Nhiều phần của hệ thống có thể bị lỗi. Rất đáng để xem xét cách chúng ta xử lý các lỗi node.
- **Tuân thủ (Compliance)** - PII (Thông tin định danh cá nhân) cần được lưu trữ một cách hợp lý, tuân theo luật GDPR của Châu Âu.
- **Bảo mật** - mã hóa email, bảo vệ chống lừa đảo, duyệt web an toàn, v.v.
- **Tối ưu hóa** - ví dụ ngăn chặn việc trùng lặp các tệp đính kèm giống nhau, được gửi nhiều lần bởi những người dùng khác nhau.
