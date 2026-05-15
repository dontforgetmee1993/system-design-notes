# Chương 5: Thiết kế Băm Nhất quán (Consistent Hashing)

## Giới thiệu
Chương này khám phá băm nhất quán (consistent hashing), một kỹ thuật thiết yếu để đạt được khả năng mở rộng theo chiều ngang bằng cách phân phối hiệu quả các yêu cầu và dữ liệu trên các máy chủ. Nó giảm thiểu việc phân phối lại dữ liệu khi các máy chủ được thêm vào hoặc loại bỏ và đảm bảo phân phối dữ liệu đồng đều để giảm bớt các vấn đề như điểm nóng máy chủ (server hotspots).

## Vấn đề Cấp lại Băm (Rehashing)
### Giải thích
Trong các phương pháp băm truyền thống, chẳng hạn như `serverIndex = hash(key) % N`, việc phân phối lại dữ liệu trở nên có vấn đề khi số lượng máy chủ thay đổi. Ví dụ:
- Loại bỏ một máy chủ khiến hầu hết các khóa bị gán lại, dẫn đến bỏ lỡ bộ nhớ đệm (cache misses).
- Thêm một máy chủ dẫn đến việc phân phối lại khóa không cần thiết.

  <img src="./images/server-hashing.png"  alt="Server hashing" width="450">

- Cách tiếp cận này hoạt động tốt khi quy mô của nhóm máy chủ là cố định. Tuy nhiên, các vấn đề phát sinh khi các máy chủ mới được thêm vào, hoặc các máy chủ hiện có bị loại bỏ.

  <img src="./images/server-hashing-miss.png"  alt="Server hashing Miss" width="450">

### Vấn đề chính
Việc phân phối lại hầu hết các khóa khi số lượng máy chủ thay đổi gây ra sự kém hiệu quả và quá tải.

## Băm Nhất quán (Consistent Hashing)
### Định nghĩa
Băm nhất quán đảm bảo rằng chỉ một phần nhỏ các khóa bị ánh xạ lại khi các máy chủ được thêm vào hoặc loại bỏ. Điều này giảm thiểu sự gián đoạn và tăng cường khả năng mở rộng.

### Các Khái niệm Chính
1. **Không gian băm và Vòng băm (Hash Space and Ring):** Không gian băm tạo thành một vòng liên tục, với các giá trị băm được phân phối từ `0` đến `2^160-1` (ví dụ: sử dụng hàm băm như SHA-1). Bằng cách kết nối hai đầu, chúng ta có một vòng tròn.
    <p align="center">
    <img src="./images/hash-ring.png"  alt="Hash Ring" width="450">
    </p>

- Sử dụng cùng một hàm băm f, chúng ta ánh xạ các máy chủ dựa trên IP hoặc tên máy chủ lên vòng tròn.

    <p align="center">
    <img src="./images/server-ring.png"  alt="Server Ring" width="450">
    </p>

1. **Tra cứu Máy chủ (Server Lookup)**
- Máy chủ của một khóa được xác định bằng cách duyệt theo chiều kim đồng hồ trên vòng tròn cho đến khi tìm thấy một máy chủ.

  <p align="center">
  <img src="./images/server-lookup.png"  alt="Server Lookup" width="450">
  </p>

2. **Thêm và Loại bỏ Máy chủ**
- Thêm một máy chủ chỉ phân phối lại các khóa lân cận. Chỉ một phần nhỏ các khóa được phân phối lại cho máy chủ mới.
  
  <p align="center">
  <img src="./images/adding-server.png"  alt="Adding Server" width="450">
  </p>

- Loại bỏ một máy chủ chỉ ảnh hưởng đến các khóa trong phạm vi của nó. Chỉ các khóa từ máy chủ bị loại bỏ mới được gán lại cho máy chủ tiếp theo theo chiều kim đồng hồ.

  <p align="center">
  <img src="./images/removing-server.png"  alt="Removing Server" width="450">
  </p>

## Thách thức và Giải pháp
### Hai vấn đề trong cách tiếp cận cơ bản
1. **Kích thước Phân vùng Không đều:** Các máy chủ có thể có các phân vùng dữ liệu không bằng nhau.
2. **Phân phối Khóa Không đồng nhất:** Một số máy chủ có thể nhận được nhiều khóa hơn đáng kể so với những máy chủ khác.

### Giải pháp: Nút Ảo (Virtual Nodes)
- Mỗi máy chủ được đại diện bởi nhiều nút ảo trên vòng tròn, được phân phối đồng đều trên vòng tròn.
- Các nút ảo cải thiện việc phân phối khóa và cân bằng tải. Khi số lượng nút ảo tăng lên, việc phân phối các khóa trở nên cân bằng hơn. Điều này là do độ lệch chuẩn trở nên nhỏ hơn với nhiều nút ảo hơn, dẫn đến phân phối dữ liệu cân bằng.
   
  <p align="center">
  <img src="./images/virtual-nodes.png"   alt="Virtual Nodes" width="450">
  </p>

## Các khóa bị ảnh hưởng
Khi các máy chủ được thêm vào hoặc loại bỏ:
- **Thêm Máy chủ:** Các khóa bị ảnh hưởng là những khóa nằm giữa máy chủ mới và máy chủ đứng trước nó.
  Trong ví dụ sau, máy chủ 4 (s4) được thêm vào vòng tròn. Phạm vi bị ảnh hưởng bắt đầu từ s4 (nút mới được thêm vào) và di chuyển ngược chiều kim đồng hồ quanh vòng tròn cho đến khi tìm thấy một máy chủ (s3). Do đó, các khóa nằm giữa s3 và s4 cần được phân phối lại cho s4.

  <p align="center">
  <img src="./images/server-addition.png"   alt="Server Addition" width="450">
  </p>

- **Loại bỏ Máy chủ:** Các khóa bị ảnh hưởng là những khóa nằm giữa máy chủ bị loại bỏ và máy chủ đứng trước nó. Trong ví dụ sau, khi một máy chủ (s1) bị loại bỏ, phạm vi bị ảnh hưởng bắt đầu từ s1 (nút bị loại bỏ) và di chuyển ngược chiều kim đồng hồ quanh vòng tròn cho đến khi tìm thấy một máy chủ (s0). Do đó, các khóa nằm giữa s0 và s1 phải được phân phối lại cho s2.
   
  <p align="center">
  <img src="./images/server-removed.png"   alt="Server Removed" width="450">
  </p>

## Lợi ích của Băm Nhất quán
- **Giảm thiểu việc Phân phối lại:** Chỉ một phần nhỏ các khóa bị gán lại.
- **Khả năng mở rộng:** Cho phép mở rộng theo chiều ngang.
- **Giảm bớt Điểm nóng:** Cân bằng phân phối dữ liệu để tránh quá tải máy chủ.

## Ứng dụng Thực tế
- Amazon Dynamo DB
- Apache Cassandra
- Discord
- Akamai CDN
- Maglev Load Balancer
